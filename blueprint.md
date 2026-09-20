# KẾ HOẠCH TRIỂN KHAI CHI TIẾT (IMPLEMENTATION BLUEPRINT)
**Dự án:** `netrafs` (Network Traffic Shaping System)

---

## GIAI ĐOẠN 0: KHỞI TẠO DỰ ÁN VÀ MÔI TRƯỜNG
1.  **Khởi tạo Module:**
    *   Chạy `go mod init netrafs`.
    *   Cài đặt các dependencies lõi:
        *   `go get github.com/cilium/ebpf/cmd/bpf2go`
        *   `go get github.com/vishvananda/netlink`
        *   `go get github.com/spf13/cobra@latest`
2.  **Tạo cấu trúc thư mục:** Tạo chính xác các thư mục `bpf/headers`, `cmd/netrafs`, `internal/ai`, `internal/bpf`, `internal/daemon`, `internal/netlink`, `internal/api`, `internal/types`, `web`.
3.  **Tạo file Model mặc định:** Tạo file `default_model.json` tại thư mục gốc chứa mảng 3 object Centroids (Realtime, Normal, Bulk) với các vector giả định.

---

## GIAI ĐOẠN 1: TẦNG KERNEL (eBPF C CODE)
**Mục tiêu:** Viết mã C, biên dịch thành bytecode và tạo Go bindings.

1.  **Chuẩn bị Headers (`bpf/headers/`):**
    *   Tạo/Tải `vmlinux.h` (dành cho kiến trúc x86_64/arm64).
    *   Tạo `bpf_helpers.h` và `bpf_endian.h`.
2.  **Định nghĩa Cấu trúc dùng chung (`bpf/headers/common.h`):**
    *   `struct flow_key`: 5-tuple (src_ip, dst_ip, src_port, dst_port, protocol).
    *   `struct flow_metrics`: Chứa `packet_count`, `total_size`, `last_time`, `iat_sum`, `iat_sq_sum` (để tính phương sai).
3.  **Viết `bpf/dns_snoop.c`:**
    *   Hook: `SEC("socket")` hoặc `SEC("xdp")` lọc UDP Port 53.
    *   Logic: Parse DNS header, tìm bản ghi A/AAAA response.
    *   Map: Khai báo `BPF_MAP_TYPE_LRU_HASH` tên `dns_map` (Key: `__u32` IP, Value: `char[256]` Domain). Ghi kết quả vào map.
4.  **Viết `bpf/tc_flow.c`:**
    *   Hook: `SEC("tc")` cho Ingress và Egress.
    *   Map 1: `BPF_MAP_TYPE_HASH` tên `flow_state` (Key: `flow_key`, Value: `flow_metrics`).
    *   Map 2: `BPF_MAP_TYPE_HASH` tên `flow_class` (Key: `flow_key`, Value: `__u32` ClassID).
    *   Map 3: `BPF_MAP_TYPE_RINGBUF` tên `flow_events`.
    *   Logic:
        *   Bóc tách packet lấy 5-tuple.
        *   Tra `flow_class`. Nếu có ClassID -> Gán `skb->mark = ClassID` -> `return TC_ACT_OK`.
        *   Nếu chưa có: Cập nhật `flow_metrics` trong `flow_state`.
        *   Nếu `packet_count == 10`: Đẩy `flow_metrics` qua `flow_events` (Ringbuf) lên User-space. Gán tạm `skb->mark = 2` (Normal).
5.  **Tạo Go Bindings (`bpf/gen.go`):**
    *   Viết directive: `//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -type flow_key -type flow_metrics bpf tc_flow.c -- -I./headers`
    *   Viết directive tương tự cho `dns_snoop.c`.
    *   Chạy `go generate ./bpf/...`.

---

## GIAI ĐOẠN 2: TẦNG TRÍ TUỆ NHÂN TẠO (PURE GO AI)
**Mục tiêu:** Xây dựng engine suy luận K-Means không phụ thuộc thư viện ngoài.

1.  **Định nghĩa Types (`internal/types/ai.go`):**
    *   `type ClassID uint32` (1: Realtime, 2: Normal, 3: Bulk).
    *   `type FeatureVector struct { AvgPktSize, IatMean, IatVar, PktRate float64 }`.
    *   `type Centroid struct { Class ClassID; Vector FeatureVector }`.
2.  **Viết Logic Suy luận (`internal/ai/kmeans.go`):**
    *   Hàm `LoadModel(jsonData []byte) error`: Parse `default_model.json` vào slice các `Centroid`.
    *   Hàm `Predict(features FeatureVector) ClassID`:
        *   Dùng `math.Pow` tính khoảng cách Euclidean giữa `features` và từng `Centroid`.
        *   Trả về `ClassID` của Centroid có khoảng cách nhỏ nhất.

---

## GIAI ĐOẠN 3: TẦNG USER-SPACE DAEMON & DATA PIPELINE
**Mục tiêu:** Đọc Ringbuf, tính toán Features, gọi AI và ghi Map.

1.  **Quản lý eBPF Lifecycle (`internal/bpf/loader.go`):**
    *   Hàm `LoadAndAttach(ifaceName string)`: Load compiled objects, attach TC hook vào interface chỉ định qua `netlink`.
2.  **Xử lý Ring Buffer (`internal/bpf/ringbuf.go`):**
    *   Khởi tạo `ringbuf.NewReader(objs.FlowEvents)`.
    *   Goroutine vòng lặp `Read()`:
        *   Nhận `flow_metrics` thô từ C.
        *   Tính toán: `AvgPktSize = total_size / 10`, `IatMean = iat_sum / 9`, `IatVar = (iat_sq_sum / 9) - (IatMean * IatMean)`.
        *   Gọi `ai.Predict(features)`.
        *   Ghi kết quả xuống Kernel: `objs.FlowClass.Update(key, classID, ebpf.UpdateAny)`.

---

## GIAI ĐOẠN 4: TẦNG ĐIỀU KHIỂN VẬT LÝ (NETLINK & TC)
**Mục tiêu:** Cấu hình HTB, CAKE và xử lý các trạng thái `default`, `max`, `god`.

1.  **Khởi tạo Qdisc (`internal/netlink/qdisc.go`):**
    *   **QUAN TRỌNG:** Thêm `runtime.LockOSThread()` ở đầu mọi hàm thao tác netlink.
    *   Hàm `InitTrafficControl(iface string, maxBandwidth uint64)`:
        *   Xóa toàn bộ qdisc cũ (`netlink.QdiscDel`).
        *   Tạo Root HTB qdisc (Handle 1:0).
        *   Tạo 3 HTB Classes (1:1 Realtime, 1:2 Normal, 1:3 Bulk) với Rate/Ceil/Prio mặc định theo TDD v2.0.
        *   Tạo 3 CAKE qdiscs gắn vào 3 HTB Classes.
        *   Tạo 3 U32/Fwmark Filters: Mark 1 -> Class 1:1, Mark 2 -> Class 1:2, Mark 3 -> Class 1:3.
2.  **Xử lý Trạng thái (`internal/netlink/state.go`):**
    *   Hàm `SetClassState(targetClass ClassID, state string)`:
        *   Nếu `state == "default"`: Reset Rate, Ceil, Prio của cả 3 class về chuẩn.
        *   Nếu `state == "max"` (Polite Max): Chỉ gọi `netlink.ClassReplace` để sửa `Ceil` của `targetClass` thành 100% maxBandwidth. Giữ nguyên Prio.
        *   Nếu `state == "god"` (Priority Inversion):
            *   Sửa `targetClass`: Prio = 1, Ceil = 100%.
            *   Sửa các class còn lại: Prio = 3, Ceil = giới hạn thấp.
            *   Gọi `netlink.ClassReplace` cho cả 3 class để áp dụng ngay lập tức.

---

## GIAI ĐOẠN 5: GIAO TIẾP IPC & HTTP API
**Mục tiêu:** Tạo cầu nối giữa CLI/Web và Daemon.

1.  **Khởi tạo Unix Socket (`internal/daemon/ipc.go`):**
    *   Xóa file socket cũ nếu tồn tại.
    *   `listener, err := net.Listen("unix", "/var/run/netrafs.sock")`.
    *   `os.Chmod("/var/run/netrafs.sock", 0660)`.
2.  **Định nghĩa API Handlers (`internal/api/handlers.go`):**
    *   Sử dụng `http.NewServeMux()`.
    *   `GET /api/flows`:
        *   Duyệt `objs.FlowState` map (eBPF).
        *   Với mỗi IP đích, tra cứu `objs.DnsMap`. Nếu có, gán Domain; nếu không, giữ nguyên IP.
        *   Trả về mảng JSON chứa danh sách luồng.
    *   `POST /api/class/set`:
        *   Nhận JSON `{ "class": 3, "state": "god" }`.
        *   Gọi `netlink.SetClassState()`. Trả về 200 OK.
    *   `POST /api/model/reload`: Nhận file JSON, gọi `ai.LoadModel()`.

---

## GIAI ĐOẠN 6: GIAO DIỆN DÒNG LỆNH (CLI CLIENT)
**Mục tiêu:** Xây dựng công cụ `netrafs` bằng Cobra.

1.  **Setup Root Command (`cmd/netrafs/main.go`):** Khởi tạo Cobra root cmd.
2.  **Lệnh `daemon`:**
    *   `netrafs daemon start --mode [endpoint|router] --iface [name] --web`
    *   Khởi chạy các module ở Giai đoạn 3, 4, 5. Block main thread bằng `chan os.Signal`.
3.  **Lệnh `watch`:**
    *   Vòng lặp `for { ... time.Sleep(1 * time.Second) }`.
    *   Gửi HTTP GET tới Unix Socket `/api/flows`.
    *   In `\033[2J\033[H`.
    *   Khởi tạo `tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', 0)`.
    *   In Header và các dòng dữ liệu. Gọi `w.Flush()`.
4.  **Lệnh `class set`:**
    *   Cú pháp: `netrafs class set <realtime|normal|bulk> <default|max|god>`.
    *   Parse args, map string thành ClassID.
    *   Gửi HTTP POST tới Unix Socket `/api/class/set`. In kết quả thành công/thất bại.

---

## GIAI ĐOẠN 7: GIAO DIỆN WEB (CSR) VÀ TÍCH HỢP CUỐI
**Mục tiêu:** Hoàn thiện Web GUI và đóng gói Binary duy nhất.

1.  **Phát triển Frontend (`web/`):**
    *   Dùng Vite khởi tạo project Svelte/Preact.
    *   Viết service fetch data từ `/api/flows` (dùng relative path để tương thích cả Unix Socket proxy và TCP Port).
    *   Tạo UI Dashboard: Bảng danh sách luồng, 3 nút Toggle (Default, Max, God) cho từng Class.
    *   Chạy `npm run build` xuất ra thư mục `web/dist`.
2.  **Nhúng Frontend vào Go (`web/embed.go`):**
    *   Khai báo `//go:embed dist/*` vào biến `embed.FS`.
    *   Trong `internal/daemon/ipc.go`, nếu cờ `--web` bật, khởi tạo thêm một `http.Server` listen trên TCP Port 8080, serve `embed.FS` tại route `/`.
3.  **Hoàn thiện Makefile:**
    *   Tạo target `bpf`: Chạy `go generate ./bpf/...`.
    *   Tạo target `web`: Chạy `cd web && npm run build`.
    *   Tạo target `build`: Phụ thuộc vào `bpf` và `web`, chạy `go build -ldflags="-s -w" -o netrafs ./cmd/netrafs`.
    *   Tạo target `clean`: Xóa binary, xóa `web/dist`, xóa các file `.o` sinh ra bởi bpf2go.
