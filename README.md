# BẢN THIẾT KẾ KỸ THUẬT VÀ TRIỂN KHAI (TECHNICAL DESIGN DOCUMENT)
**Dự án:** `netrafs` (Network Traffic Shaping System)</br>
**Mục tiêu:** Hệ thống phân loại và định hình lưu lượng mạng tự động dựa trên AI, hoạt động zero-latency thông qua eBPF. Phù hợp chạy cục bộ trên máy cá nhân (Endpoint) hoặc làm gateway/router điều phối mạng LAN. Tôn trọng triết lý Suckless, sử dụng công nghệ tối giản, nguyên bản và hiệu năng cao.

---

## 1. KIẾN TRÚC TỔNG THỂ (SYSTEM ARCHITECTURE)

Hệ thống `netrafs` được chia thành 3 phân hệ độc lập, tuân thủ nguyên tắc "Separation of Concerns":

1.  **Tầng Kernel (eBPF & Linux `tc`):** Xử lý ở cấp độ Ring 0. Chịu trách nhiệm bắt gói tin, trích xuất đặc trưng (Feature Extraction), phân giải DNS nội bộ, và thực thi luật QoS thông qua thuật toán Token Bucket (HTB) kết hợp `CAKE/fq_codel`. Độ trễ xử lý tiệm cận 0.
2.  **Tầng User-space Daemon (Go Backend):** Tiến trình ngầm `netrafs-daemon` chạy quyền Root. Giao tiếp với Kernel qua `eBPF Ring Buffer` và `BPF Maps`. Thực thi mô hình Toán học/AI thuần Go để ra quyết định và cấu hình Network Namespace qua API `netlink`.
3.  **Tầng Điều khiển (CLI & Web GUI CSR):** Điểm giao tiếp duy nhất cho người dùng. CLI client sử dụng HTTP Request qua Unix Domain Socket (`/var/run/netrafs.sock`) để ra lệnh cho Daemon, không dùng gRPC để đảm bảo sự tối giản.

### Sơ đồ Luồng dữ liệu Xử lý (Packet Pipeline)
1. Packet đi vào/ra NIC -> Bị chặn bởi `eBPF TC Hook`.
2. Nếu là Flow mới (Socket 5-tuple mới): eBPF đo đạc 10 gói tin đầu tiên -> Gửi Metadata (Features) qua Ring Buffer lên Go Daemon. Packet tạm đi vào class `Normal`.
3. Go Daemon ráp metadata -> Đưa vào K-Means Inference (Go thuần) -> Xuất ra `ClassID` (1, 2, hoặc 3).
4. Go Daemon ghi `ClassID` vào `eBPF Hash Map`.
5. Từ gói thứ 11 trở đi, eBPF TC Hook chỉ cần tra cứu Map -> Gắn `skb->mark` = `ClassID`.
6. Hạt nhân Linux (`tc HTB`) đọc `skb->mark` và định tuyến packet vào đúng Hàng đợi vật lý theo giới hạn băng thông và độ trễ đã thiết lập.

---

## 2. QUY HOẠCH CLASS LƯU LƯỢNG (QoS CLASSES) VÀ VẬT LÝ HÀNG ĐỢI

Hệ thống loại bỏ hoàn toàn việc bắt người dùng phân loại từng IP/Tiến trình. Thuật toán AI sẽ tự động ánh xạ mọi luồng mạng vào 3 Class vật lý dựa trên **Băng thông (Rate/Ceil)**, **Độ trễ (Latency Target)** và **Ưu tiên xử lý (Priority)**.

| Class ID | Tên Class | Loại lưu lượng (AI nhận diện) | Băng thông (B/W) | Độ trễ (Ping Target) | Priority | Xử lý vật lý (Queue) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | `Realtime` | Game, VoIP, SSH, DNS | Nhỏ (Giới hạn Rate/Ceil thấp) | **Cực thấp** (< 5ms) | **Cao nhất (1)** | Không đệm (No-buffer), đi qua lập tức. Vượt ngưỡng bandwidth -> Drop. |
| **2** | `Normal` | Lướt Web, Netflix, Social | Trung bình | **Trung bình** (~20ms) | **Mức 2** | Đệm nhẹ, đảm bảo luồng không bị rớt khung hình nhưng không chiếm sóng. |
| **3** | `Bulk` | Tải file, Torrent, Update | Lớn (Cho phép quét sạch BW) | **Rất cao** (>100ms) | **Thấp nhất (3)** | Hàng đợi cực sâu (Deep buffer). Cho phép dồn TCP để max băng thông nhưng phải nhường đường 100% khi Class 1/2 có gói tin. |

---

## 3. THAO TÁC TRẠNG THÁI (STATE/MODE) & "GOD MODE"

Người dùng thay đổi chất lượng mạng thông qua lệnh CLI: `netrafs class set <class> <state>`. Bản chất của lệnh này là việc Go Daemon gọi API `netlink` để tái cấu trúc lại thông số vật lý của HTB (Hierarchical Token Bucket) dưới Kernel theo thời gian thực (Zero-downtime).

Dưới đây là mô tả kỹ thuật của 3 trạng thái khi thao tác trên một Class (Ví dụ: Class `bulk`):

1.  **Trạng thái `default` (Mặc định):**
    *   Class hoạt động đúng theo Bảng quy hoạch ở Phần 2.
    *   Ví dụ: `bulk` bị giới hạn trần (Ceil) ở mức thấp/vừa phải để đảm bảo nhiệt độ máy và tránh nghẽn I/O.
2.  **Trạng thái `max` (Polite Max - Tối đa lịch sự):**
    *   Chỉ can thiệp **Băng thông**, giữ nguyên **Priority** và **Latency**.
    *   Kỹ thuật: Mở thông số `Ceil` của `bulk` lên 100% card mạng. `bulk` được tải tốc độ cực đại, NHƯNG vì Priority vẫn là 3, nó sẽ bị Kernel chặn lại ngay lập tức nếu xuất hiện gói tin `Realtime` (Priority 1). Mạng đảm bảo không lag.
3.  **Trạng thái `god` (God Mode - Chế độ Độc tài / Đảo ngược ưu tiên):**
    *   Phá vỡ mọi giới hạn, ép hệ thống phục vụ duy nhất 1 mục đích bằng kỹ thuật **Priority Inversion (Đảo ngược ưu tiên)**.
    *   Kỹ thuật khi gọi `netrafs class set bulk god`:
        *   Sửa `bulk`: Priority = 1 (Cao nhất), Ceil = 100%, Latency Target = 5ms.
        *   Sửa `realtime`: Priority = 2 hoặc 3.
        *   Sửa `normal`: Priority = 3.
    *   Kết quả: Gói tin Torrent/Download đi thẳng lên đầu hàng. Nếu người dùng chơi Game lúc này, Ping game sẽ tăng vọt. Quyền kiểm soát sinh/sát hoàn toàn thuộc về Root User.

---

## 4. MÔ HÌNH TRÍ TUỆ NHÂN TẠO (AI) VÀ TÍCH HỢP

### 4.1. Feature Engineering (Trích xuất đặc trưng eBPF)
Hệ thống sử dụng Học không giám sát (Unsupervised Learning). AI phân tích hình thái vật lý thay vì Payload, giúp chống lại sự thay đổi giao thức như QUIC (HTTP/3) hoặc mã hóa TLS 1.3/ECH.

Mỗi Flow mới, eBPF gom 10 gói tin đầu tiên vào một cửa sổ tính toán để sinh ra 4 chiều không gian (Features):
1.  **`avg_pkt_size` (Kích thước trung bình):** Biến số phân biệt Game (< 200 Bytes) và Download (~1500 Bytes).
2.  **`iat_mean` (Trung bình khoảng cách thời gian):** Nhận diện hành vi gửi liên tục hay ngắt quãng.
3.  **`iat_variance` (Phương sai thời gian tới):** Trọng số cốt lõi phân biệt *Cloud Gaming / Realtime Streaming* (Phương sai thấp, dòng chảy đều đặn) với *Buffered Video / Netflix* (Phương sai cao, Burst-then-sleep).
4.  **`pkt_rate` (Tần suất):** Tính trên 50ms đầu tiên.

### 4.2. Tích hợp AI (Decoupled Pure Go Inference)
Không sử dụng CGO, không nhúng TensorFlow/ONNX.
*   **Model:** K-Means Clustering (Train offline sẵn, lưu tọa độ trọng tâm Centroids ra JSON).
*   **Struct & Load:** Model là file `model.json` được nhúng trực tiếp vào Binary bằng `//go:embed`. Hỗ trợ CLI lệnh `netrafs model reload <path>` để nạp model đã Fine-tune.
*   **Inference:** Logic suy luận thuần toán học trong Go (Euclidean distance function).
    ```go
    // Pseudo-logic In-memory Inference
    func Predict(flowFeatures Vector, centroids []Centroid) ClassID {
        minDist := math.MaxFloat64
        class := Normal
        for _, c := range centroids {
            dist := calculateEuclidean(flowFeatures, c.Vector)
            if dist < minDist { minDist = dist; class = c.ClassID }
        }
        return class
    }
    ```

---

## 5. ĐỊNH DANH ĐÍCH ĐẾN (DESTINATION RESOLUTION) VÀ HIỂN THỊ

Hiển thị Destination cho người dùng theo trình tự Ưu tiên (Fallback). Chấp nhận bỏ qua các thư viện GeoIP cồng kềnh.

1.  **Local DNS Snooping (eBPF):**
    *   Chương trình `dns_snoop.c` đính vào UDP Port 53.
    *   Bắt gói tin phản hồi DNS (DNS Response), đọc Record A/AAAA.
    *   Ghi vào `BPF_MAP_TYPE_LRU_HASH`: `Key = IP`, `Value = Domain String`. Map LRU tự động đẩy IP cũ ra khi đầy để tiết kiệm RAM.
2.  **Hiển thị CLI (Luồng Fallback):**
    *   Khi CLI gọi API yêu cầu log, Go Daemon lấy IP của Flow.
    *   B1: Tra vào `DNS LRU Map`. Có kết quả -> Gửi Domain (`github.com`).
    *   B2: Nếu không có (Game dùng Direct IP, DNS Cache miss) -> Gửi trực tiếp IP (`103.25.14.9`). Không sử dụng ASN/GeoIP.

---

## 6. LỰA CHỌN CÔNG NGHỆ VÀ TIÊU CHUẨN CODE (TECH STACK)

Tuân thủ nghiêm ngặt chuẩn hệ thống Unix, Suckless, cấm sử dụng các bộ UI nặng nề.

*   **Ngôn ngữ lõi:** `Go >= 1.22`. Tận dụng `net/http` router mới. Không CGO ở phần User-space.
*   **eBPF Toolchain:** `github.com/cilium/ebpf` + `bpf2go`. Compile mã C sang bytecode trực tiếp tại build-time. Không yêu cầu Clang trên máy End-user.
*   **Linux Networking API:** `github.com/vishvananda/netlink`. Không gọi `os.Exec("tc ...")`. **Lưu ý quan trọng:** Bắt buộc dùng `runtime.LockOSThread()` cho mọi Goroutine thao tác với netlink để tránh lỗi Network Namespace.
*   **CLI Framework:** `github.com/spf13/cobra` + `pflag`.
*   **CLI Realtime Log (Thay thế TUI):** Standard lib `text/tabwriter`. Sử dụng ANSI Escape Codes: `fmt.Print("\033[2J\033[H")` (Xóa màn hình, reset con trỏ) vòng lặp 1s/lần.
*   **Giao tiếp IPC:** HTTP Server listen trên `net.Listen("unix", "/var/run/netrafs.sock")`.
*   **Web GUI:** Client-Side Rendering bằng `Svelte` hoặc `Preact` + `TailwindCSS` (CDN hoặc compiled). Static files được gói qua `//go:embed`.

---

## 7. GIAO DIỆN DÒNG LỆNH (CLI COMMANDS)

Sử dụng Socket IPC để Client ra lệnh cho Daemon. Kiến trúc Client rất mỏng, chỉ làm nhiệm vụ parse args và call HTTP.

*   `netrafs daemon [start|stop|restart] [--mode endpoint|router] [--iface eth0]`
*   `netrafs watch`:
    *   Tự động clear screen mỗi giây.
    *   Định dạng cột (Tabwriter): `PID/COMM | FLOW ID | CLASS | RATE | DESTINATION`
*   `netrafs class set <realtime|normal|bulk> <default|max|god>`:
    *   Đổi trạng thái vật lý (Bandwidth/Priority/Latency). Sinh ra payload JSON gửi xuống Socket.
*   `netrafs class show`: In ra cấu trúc HTB/CAKE vật lý hiện tại của Kernel.
*   `netrafs model reload <file.json>`: Hot-reload trọng số K-Means.

---

## 8. TRIỂN KHAI VÀ GIAO DIỆN WEB (DEPLOYMENT & GUI)

### 8.1. Các chế độ hoạt động (Modes)
*   **Endpoint Mode (Mặc định):** Chạy trên PC/Laptop Linux cá nhân. eBPF hook vào Ingress/Egress của Card vật lý chính. API `/api/flows` truy xuất thông tin PID/Tiến trình để hiển thị log.
*   **Router/LAN Mode:** Kích hoạt qua cờ `--mode router`.
    *   Chạy trên Gateway Linux (Homelab/OpenWrt x86).
    *   eBPF hook vào Bridge Interface (`br-lan`).
    *   API `/api/flows` tự động bỏ qua cột PID (vì tiến trình không tồn tại trên Router, gói tin chỉ đi xuyên qua). Tập trung điều tiết IP nguồn của các máy con trong LAN.

### 8.2. Web GUI Architecture (Khi bật cờ `--web`)
*   Khởi chạy một Webserver trên Port (vd: `8080`). Chỉ phục vụ file `index.html` tĩnh (từ `//go:embed`).
*   Browser của người dùng tải HTML/JS về, tự động fetch data từ các endpoint `/api/v1/flows` và `/api/v1/classes` thông qua AJAX/Fetch.
*   DOM Rendering, tính toán Chart (Băng thông) diễn ra 100% tại Client (CSR), đảm bảo Router/Server không tốn CPU để gen HTML.
*   Cung cấp các Toggle Switch tương đương các cờ `default | max | god` trên GUI.

---

## 9. CẤU TRÚC THƯ MỤC CHUẨN

```text
netrafs/
├── bpf/                    # C Code (Zero dependencies ngoài headers)
│   ├── headers/            # vmlinux.h, bpf_helpers.h
│   ├── tc_flow.c           # TC Ingress/Egress hooking, feature extraction
│   ├── dns_snoop.c         # UDP 53 intercept
│   └── gen.go              # Lệnh //go:generate bpf2go (Cilium)
├── cmd/
│   └── netrafs/            
│       └── main.go         # Cobra CLI setup & entrypoint
├── internal/
│   ├── ai/                 # kmeans.go (Pure mathematical inference)
│   ├── bpf/                # Go wrapper for eBPF maps & ringbufs
│   ├── daemon/             # ipc.go (Unix socket init), daemon loop
│   ├── netlink/            # qdisc.go, htb.go (Locked OS thread setup)
│   ├── api/                # handlers.go (REST API cho CLI & Web)
│   └── types/              # DTOs dùng chung cho toàn bộ module
├── web/
│   ├── src/                # Svelte/Preact Source code
│   └── embed.go            # //go:embed dist/ (Chứa HTML build sẵn)
├── default_model.json      # File trọng số K-Means
├── Makefile                # Build scripts (bpf, web, go build)
└── go.mod
```
