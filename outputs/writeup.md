# BÁO CÁO PHÂN TÍCH TỐI ƯU HÓA CHI PHÍ GPU (GPU FINOPS WRITE-UP)

**Dự án:** NimbusAI GPU FinOps Cost Optimization  
**Học viên:** Nguyễn Mai Hoàng Thiện (MSSV: 2A202602912)  
**Khóa học:** AICB - Phase 2 - Track 2 (Infrastructure & FinOps) - Day 25  
**Kiểm thử tự động:** `verify.py` 11/11 PASS | `pytest` 15/15 PASS  

---

## 1. BASELINE VS. OPTIMIZED: SO SÁNH CHI PHÍ VÀ ĐƠN GIÁ TOKEN

Kết quả kiểm toán tài chính trước và sau khi triển khai các biện pháp tối ưu FinOps tại NimbusAI:

| Tiêu chí đo lường | Baseline (Trước tối ưu) | Optimized (Sau tối ưu) | Mức cắt giảm | Tỷ lệ giảm |
| :--- | :---: | :---: | :---: | :---: |
| **Tổng chi phí hàng tháng (Total Spend)** | **$27,133** | **$14,626** | **-$12,507** | **-46.1%** |
| **Đơn giá suy luận ($/1M-token)** | **$6.488 / 1M** | **$1.126 / 1M** | **-$5.362 / 1M** | **-82.6%** |
| **Chi phí suy luận hàng ngày** | $48.87 / ngày | $8.48 / ngày | -$40.39 / ngày | -82.6% |
| **Chi phí Workload máy chủ hàng tháng** | $25,667 / tháng | $15,627 / tháng | -$10,040 / tháng | -39.1% |
| **Lãng phí không tải (Idle Waste)** | $600 / tháng | $0 / tháng | -$600 / tháng | -100.0% |
| **Lãng phí Over-provisioning (Util-Lies)**| $655 / tháng | $0 / tháng | -$655 / tháng | -100.0% |

> **Ý nghĩa mấu chốt:** Việc chuẩn hóa chi phí theo đơn vị **`$/1M-token`** giúp tổ chức đo lường chính xác hiệu quả sinh lợi nghiệp vụ. NimbusAI phục vụ toàn bộ 7,533,027 tokens trong ngày kiểm thử với mức chi phí chỉ bằng **17.4%** so với trước đây.

---

## 2. PHÂN TÍCH TỪNG ĐÒN BẨY TỐI ƯU HÓA CHI PHÍ

Tổng số tiền tiết kiệm được là **$12,507/tháng**, phân bổ qua 4 đòn bẩy:

| Thứ hạng | Đòn bẩy tối ưu | Tiết kiệm hàng tháng | Tỷ trọng |
| :---: | :--- | :---: | :---: |
| **1** | **Purchasing Strategy (Spot & Reserved Commitments)** | **$10,040** | **80.3%** |
| **2** | **Inference Levers (Cascade, Prompt Cache, Batch API)** | **$1,212** | **9.7%** |
| **3** | **Right-sizing Util-Lies (Hạ cấp phần cứng over-provisioned)** | **$655** | **5.2%** |
| **4** | **Kill Idle GPUs (Thu hồi GPU không tải)** | **$600** | **4.8%** |

### Đòn bẩy đóng góp nhiều nhất: Chiến lược mua sắm (Purchasing Strategy - 80.3%)
* **Tại sao đóng góp lớn nhất?** 
  Chi phí thuê GPU huấn luyện và xử lý nền chiếm tỷ trọng áp đảo (>94%) trong tổng hóa đơn của NimbusAI ($25,667 / $27,133). Việc loại bỏ 100% hình thức thuê On-Demand truyền thống và tái cấu trúc danh mục cam kết tạo ra bước nhảy vọt tài chính ngay lập tức:
  - **Spot Instances:** Áp dụng cho các tác vụ training và batch eval có thể chịu lỗi (`interruptible=1`), kết hợp tự động lưu checkpoint định kỳ. Giá Spot mang lại mức chiết khấu 60–65% so với On-demand (ví dụ: `job-train-llm` tiết kiệm $4,404/tháng; `job-train-embed` tiết kiệm $755/tháng).
  - **Reserved Instances (1–3 năm):** Áp dụng cho các workload inference ổn định 24/7 (`job-infer-chat`, `job-infer-rag`, `job-infer-search`). Với duty cycle 100% (vượt xa điểm hòa vốn **Break-even = 55%** tại mức discount 45%), việc cam kết Reserved giúp tiết kiệm $4,242/tháng mà không gặp rủi ro bị thu hồi tài nguyên.

### Đòn bẩy mang lại hiệu quả sâu sắc nhất: Tối ưu hóa Inference (Inference Levers)
* Cắt giảm đến **82.6%** đơn giá token thông qua việc "chồng chiết khấu" (discount stack):
  1. **Cascade Routing:** Chuyển các prompt đơn giản sang model nhỏ rẻ hơn 15 lần.
  2. **Prompt Caching:** Chiết khấu 90% cho phần prefix lặp lại (System prompt / Context).
  3. **Batch API:** Giảm 50% chi phí cho các tác vụ không yêu cầu thời gian thực.
  * Hiệu ứng tích lũy: Khi kết hợp Batch API với 100% Cache Hit, chi phí thực tế chỉ còn **5.0%** so với gọi API On-Demand thông thường.

---

## 3. HIỆN TƯỢNG "GPU-UTIL LIE" VÀ TÁC ĐỘNG TÀI CHÍNH

### Danh sách GPU bị "Lie" phát hiện trong kiểm toán:
1. **`gpu-h100-4`**: `gpu_util_pct = 98.2%`, nhưng `MFU = 0.194` (19.4%) và `MBU = 0.207` (20.7%).
2. **`gpu-a10g-1`**: `gpu_util_pct = 96.9%`, nhưng `MFU = 0.268` (26.8%) và `MBU = 0.302` (30.2%).

### Cơ chế kỹ thuật và nguyên nhân gốc rễ:
* **`nvidia-smi` chỉ đo Clock Activity:** Đồng hồ đo của driver NVIDIA tính toán tỷ lệ thời gian trong cửa sổ lấy mẫu mà có *ít nhất một kernel đang chiếm dụng GPU*. Nó **không đo lường mức độ bão hòa tính toán** của các lõi Tensor Core.
* **Nghẽn băng thông bộ nhớ (Memory Stall) trong pha LLM Decode:**
  Trong tác vụ sinh văn bản tự hồi quy (Autoregressive Decode), mỗi token mới sinh ra đòi hỏi phải đọc toàn bộ ma trận trọng số hàng chục GB từ HBM vào chip (Arithmetic Intensity cực thấp ~1–2 FLOP/byte $\rightarrow$ *Memory-bound*).
  Các Streaming Multiprocessors (SMs) trên H100 liên tục bị nghẽn (stall) chờ dữ liệu từ HBM3, không thể thực hiện tính toán ma trận. Dù GPU đang "bận chờ đợi", `nvidia-smi` vẫn báo 98% utilization!

### Tác động tài chính:
Doanh nghiệp đang trả tiền On-demand cho siêu GPU H100 ($2.49/giờ) nhưng chỉ thu về năng lực tính toán tương đương một GPU phân khúc thấp hơn. Hạ cấp (`right-sizing`) `gpu-h100-4` xuống A100 và `gpu-a10g-1` xuống L4 giúp tiết kiệm ngay **$655/tháng** mà không làm suy giảm tốc độ phục vụ người dùng.

---

## 4. ĐÁNH GIÁ CÁC ĐỊNH HƯỚNG MỞ RỘNG (EXTENSIONS ANALYSIS)

Theo phạm vi thực thi cốt lõi của đề tài, toàn bộ 4 đòn bẩy FinOps nền tảng đã được triển khai và kiểm định chặt chẽ (11/11 checks PASS, 15/15 tests PASS). Dưới đây là phân tích định lượng và đánh giá kiến trúc cho 2 phần mở rộng trọng điểm:

### Extension 1 — Tinh chỉnh chính sách mua sắm (`recommend_tier` đa biến)
* **Vấn đề của logic cơ bản:** Chỉ dựa vào 2 yếu tố thô (`hours_per_day` và cờ `interruptible`).
* **Định hướng tối ưu:**
  1. Cân nhắc thêm **Tỷ lệ thu hồi Spot (Interruption Rate)** theo kiến trúc chip (H100 có tỷ lệ thu hồi cao hơn và biến động mạnh hơn L4/A10G). Nếu job có chi phí khởi động lại (checkpoint resume overhead) lớn hơn chênh lệch giá, không nên ép sang Spot.
  2. So sánh **Reserved 1-năm vs. 3-năm**: Hợp đồng 3 năm đem lại chiết khấu lớn nhất (~45%), nhưng nếu vòng đời sản phẩm LLM thay đổi nhanh (< 12 tháng), cam kết 1 năm giúp giảm rủi ro thanh khoản tài chính.

### Extension 3 — Kinh tế học của Prompt Caching (`cache_is_worth_it`)
* **Vấn đề:** Không phải lúc nào Prompt Caching cũng tiết kiệm tiền, vì nhà cung cấp đám mây thu phí ghi/lưu trữ cache (Cache Write / Storage Fee).
* **Quy tắc hòa vốn (Break-even Cache Reads):**
  $$\text{Số lần đọc lại tối thiểu } N = \frac{\text{Chi phí ghi cache / 1M token}}{\text{Tiết kiệm từ mỗi lần đọc / 1M token}}$$
  Với tỷ lệ chiết khấu đọc 90% (tiết kiệm 90% giá token đầu vào), chỉ khi một System Prompt hoặc đoạn RAG context được đọc lại trung bình $\ge 2-3$ lần thì việc bật cache mới bắt đầu sinh lời ròng.

---

## 5. KHUYẾN NGHỊ HÀNH ĐỘNG CHO FINOPS LEAD TẠI NIMBUSAI

3 hành động chiến lược ưu tiên thực hiện trong 30 ngày đầu:

1. **Hành động 1: Tái cơ cấu cam kết mua sắm phần cứng (Commitment Restructuring)**
   * Ký hợp đồng Reserved Instances cho 3 cụm GPU inference chạy 24/7 (`job-infer-chat`, `job-infer-rag`, `job-infer-search`), lập tức khóa mức tiết kiệm $5,189/tháng.
   * Cấu hình cơ chế lưu checkpoint tự động và chuyển 100% job training có thể gián đoạn sang Spot Instances, tiết kiệm thêm $4,851/tháng.
2. **Hành động 2: Kích hoạt bộ đòn bẩy Inference trên API Gateway**
   * Triển khai bộ định tuyến **Cascade Routing** để tự động đẩy 60–70% truy vấn đơn giản về model nhỏ giá rẻ (tier `small`).
   * Tối ưu hóa cấu trúc prompt để tận dụng tối đa **Prompt Caching** cho các tài liệu RAG lặp lại và gom batch cho các tác vụ đánh giá mô hình.
3. **Hành động 3: Thiết lập ranh giới quản trị tự động (Policy Guardrails & Auto-Shutdown)**
   * Kích hoạt script tự động phát hiện và tắt các GPU rơi vào trạng thái nhàn rỗi (< 10% util) quá 30 phút, triệt tiêu $600/tháng lãng phí không tải.
   * Áp dụng quy tắc "No Tag - No Compute" để duy trì Tag Coverage $\ge 90\%$, kích hoạt cổng thanh toán nội bộ Chargeback minh bạch giữa các phòng ban.

---

## 6. PHỤ LỤC BỔ SUNG: PHÂN BỔ CHI PHÍ (FOCUS), PHÁT THẢI CARBON & ORAL CHECK

### Phân bổ chi phí & Chuẩn FOCUS:
* **Tag Coverage:** Đạt **92%** (vượt ngưỡng 80% $\rightarrow$ Cổng Chargeback sẵn sàng hoạt động).
* **Chi phí theo team:** `assistant` ($2.59/ngày), `search` ($2.49/ngày), `eval` ($1.79/ngày), `rag` ($1.60/ngày).
* **Xuất bản:** File `outputs/focus_export.csv` (50 dòng) tuân thủ tiêu chuẩn FinOps FOCUS.

### Tính bền vững (Sustainability):
* Năng lượng trung bình: **0.24 Wh / query**; Carbon tại `us-east-1`: **0.091 gCO2e / query**.
* Chuyển các job training/eval sang vùng **`europe-north1`** (sử dụng thủy điện, 28 gCO2/kWh) giúp giảm **>90%** lượng phát thải carbon so với các vùng điện than.

### Giải đáp 5 câu hỏi phỏng vấn Oral Check:
1. *GPU-Util 98% có hiệu quả không?* $\rightarrow$ Không, đó là "GPU-Util Lie" do `nvidia-smi` chỉ đo clock activity; GPU có thể đang stall chờ dữ liệu từ bộ nhớ HBM.
2. *Tại sao cần $\ge 80\%$ tag coverage mới dám chargeback?* $\rightarrow$ Để tránh phân bổ bất công chi phí vô chủ (>20%), gây mâu thuẫn tài chính giữa các phòng ban.
3. *Tối ưu thế nào nếu 70% workload interruptible?* $\rightarrow$ Thiết lập auto-checkpointing và đưa toàn bộ 70% workload này lên Spot/Preemptible Instances (giảm 60–70% chi phí).
4. *Khi nào $/GPU-hr và $/1M-token trái ngược nhau?* $\rightarrow$ Khi GPU đắt hơn nhưng throughput token cao hơn vượt trội, hoặc ngược lại (GPU rẻ nhưng throughput cực thấp).
5. *Tại sao decode là memory-bound còn prefill là compute-bound?* $\rightarrow$ Prefill xử lý đồng thời $N$ tokens với cường độ tính toán cao ($O(N \cdot d^2)$ FLOPs trên $O(d^2)$ bytes); còn decode sinh từng token một, phải nạp lại toàn bộ ma trận trọng số cho mỗi token nên cường độ tính toán cực thấp (~1–2 FLOP/byte).
