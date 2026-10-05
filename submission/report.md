# BÁO CÁO BÀI TẬP VỀ NHÀ 1: TẬP VAL CÓ NÓI THẬT KHÔNG?
**Chủ đề:** Phân tích thực nghiệm thiên lệch tập val, lỗi `flip_idx`, và ảnh hưởng của `kpt_oks_sigmas` tự ước lượng trên dataset Tiger-Pose.

---
Link notebook đã chạy:https://drive.google.com/file/d/1WDa39jxtDq61z6K-QWvaxjauxoeGs8wu/view?usp=sharing

## 1. Bảng số liệu thực nghiệm tổng hợp

| Cấu hình Model | Val gốc (σ cào bằng 1/12) | Val lật gương (σ cào bằng 1/12) | Val gốc (σ tự ước lượng) | Val lật gương (σ tự ước lượng) |
|---|:---:|:---:|:---:|:---:|
| **Model 1: `flip_idx` giải phẫu** | 0.457 (mAP50: 0.995) | 0.439 (mAP50: 0.995) | 0.449 (mAP50: 0.995) | 0.431 (mAP50: 0.995) |
| **Model 2: `flip_idx` đồng nhất** | 0.417 (mAP50: 0.995) | 0.298 (mAP50: 0.878) | 0.408 (mAP50: 0.995) | 0.283 (mAP50: 0.865) |
| **Chênh lệch (Tụt hiệu năng)** | -0.040 (-8.8%) | **-0.141 (-32.1%)** | -0.041 (-9.1%) | **-0.148 (-34.3%)** |

*(Ghi chú: Box mAP50 trên cả hai tập val và cả hai model đều đạt ~0.995).*

---

## 2. Metric nào đã che giấu lỗi `flip_idx`?

Qua bảng kết quả thực nghiệm trên, ta thấy rõ 3 "tấm màn che" đã giấu nhẹm lỗi `flip_idx` nguy hiểm này:

### A. Tấm màn che 1: Box mAP (Detection Metric)
- **Hiện tượng:** Cả hai model (`flip_idx` giải phẫu và đồng nhất) đều đạt **Box mAP50 = 0.995** và Box mAP50-95 đạt 0.90–0.93 trên cả tập val gốc lẫn val lật gương.
- **Nguyên nhân:** Box detector chỉ dự đoán bounding box bao quanh toàn thân con hổ `[x, y, w, h]`. Bounding box hoàn toàn bất biến đối với thứ tự hoặc sự tráo đổi các keypoint bên trong cơ thể con hổ. Nếu kỹ sư chỉ nhìn vào Box mAP để kết luận model hoạt động tốt thì đã hoàn toàn bỏ qua việc cấu trúc xương bên trong bị đảo lộn.

### B. Tấm màn che 2: Pose mAP50 trên tập val gốc (Data Bias + Metric Lỏng lẻo)
- **Hiện tượng:** Trên tập val gốc, cả hai model đều cho điểm tuyệt đối **Pose mAP50 = 0.995**, và Pose mAP50-95 cũng rất sát nhau (0.457 so với 0.417).
- **Nguyên nhân:**
  1. *Thiên lệch dữ liệu (Dataset Orientation Bias):* Thống kê ở Mục 4A cho thấy 100% dữ liệu gốc (210 ảnh train và 53 ảnh val) đều là hổ quay sang phải. Model với `flip_idx` đồng nhất học vẹt rằng: "chân ở gần camera luôn là right_* (chân phải)". Khi đánh giá trên val gốc, model không bao giờ gặp hổ quay trái, nên quy luật học vẹt này hoạt động trơn tru và đạt điểm tối đa!
  2. *Ngưỡng OKS = 0.5 quá dễ dãi:* Ở ngưỡng OKS = 0.5, sai số vị trí keypoint được phép lệch tới bán kính khá lớn mà vẫn tính là True Positive.

### C. Tấm màn che 3: Ngay trên tập val lật gương, Pose mAP50 vẫn đạt 0.878!
- **Hiện tượng:** Khi đưa sang tập val lật gương (giả lập 100% hổ quay sang trái), model `flip_idx` đồng nhất dù bị tráo nhầm toàn bộ chân trái/phải nhưng **Pose mAP50 vẫn đạt 0.878**!
- **Nguyên nhân:** Con số 0.878 trông có vẻ rất cao và dễ đánh lừa người kiểm thử rằng "model chỉ suy giảm nhẹ". Nhưng thực tế, 4 điểm trục giữa (`nose`, `head`, `withers`, `tail_base`) không bị ảnh hưởng bởi phép lật, cộng với việc ngưỡng OKS = 0.5 quá lỏng, đã giữ cho mAP50 ở mức cao giả tạo.

### D. Metric nào THỰC SỰ vạch trần lỗi?
1. **Pose mAP50-95:** Model đồng nhất tụt dốc không phanh từ **0.417 xuống 0.298 (tụt 28.5% với σ mặc định và tụt 34.3% với σ ước lượng)**. Trong khi model giải phẫu chuẩn chỉ giảm nhẹ từ 0.457 xuống 0.439 (giảm < 4%, sự biến thiên bình thường do nội suy lật ảnh).
2. **Pose mAP ở ngưỡng khắt khe (mAP75, mAP90):** Ở các ngưỡng OKS cao, khoảng cách dự đoán sai giữa chân trái và chân phải không thể nào đáp ứng tiêu chuẩn OKS, khiến mAP75 sụp đổ.
3. **Ảnh hưởng của `kpt_oks_sigmas` tự ước lượng:**
   - Khi áp dụng bộ $\sigma$ tự ước lượng (thắt chặt $\sigma_{nose}=0.025$, các khớp xương $\sigma=0.06$, nới lỏng u vai và đuôi), mAP50-95 của model lỗi trên tập val lật gương tụt sâu hơn nữa (từ 0.298 xuống **0.283**).
   - Lý do: $\sigma$ tự ước lượng đánh giá khắt khe hơn ở các khớp cổ chân/cổ tay (`wrist`, `hock`), nơi lỗi hoán đổi trái/phải xảy ra nghiêm trọng nhất.

---

## 3. Bạn sẽ thiết kế tập val thế nào để phản ánh đúng thực tế?

Để tập validation thực sự "nói thật" và đóng vai trò như một chốt chặn kiểm thử tin cậy (reliability gate) trước khi triển khai hệ thống Computer Vision ra môi trường thực tế, ta cần tuân thủ 4 nguyên tắc thiết kế sau:

### 1. Phá vỡ thiên lệch phân phối (Stratified & Balanced Validation Slices)
- **Cân bằng hướng di chuyển (Orientation Balance):** Tập val bắt buộc phải phân bổ 50% đối tượng quay trái và 50% đối tượng quay phải. Nếu nguồn dữ liệu thu thập bị lệch một chiều (như camera một chiều ở sở thú), phải chủ động bổ sung dữ liệu thu thập mới hoặc dùng synthetic mirroring có kiểm định nhãn để cân bằng tập val.
- Không bao giờ cho phép tập val có cùng điểm mù (blind spot) với tập train!

### 2. Thiết kế các kịch bản kiểm thử mở rộng (Stress-test Benchmarks / Slices)
Chia tập val thành các "slice" chuyên biệt đại diện cho các điều kiện thực tế khắc nghiệt:
- `val_canonical`: Điều kiện tiêu chuẩn (ảnh rõ nét, tư thế chuẩn).
- `val_occluded`: Đối tượng bị che khuất một phần (hổ đi trong bụi cỏ cao, bị thân cây che nửa thân, chân ngập dưới nước).
- `val_unusual_poses`: Các tư thế hiếm gặp trong tự nhiên (hổ cuộn tròn ngủ, vồ mồi trên không, nằm ngửa phơi bụng).
- `val_crowded`: Nhiều cá thể đan xen nhau (chồng chéo chìa chân sang nhau để thử thách khả năng phân biệt cá thể).

### 3. Bộ Metric chẩn đoán đa chiều (Multi-metric Diagnostic Protocol)
- **Tuyệt đối không nghiệm thu hệ thống chỉ bằng một con số mAP50:** Phải báo cáo song song `mAP50-95`, `mAP75`, và phân rã mAP theo từng kích thước object (`mAP_s`, `mAP_m`, `mAP_l`).
- **Bổ sung Symmetry Consistency Test (Kiểm tra đối xứng giải phẫu):**
  Tự động đo metric `OKS_swapped` cho mọi đối tượng. Nếu tỷ lệ ảnh có $OKS_{swapped} > OKS_{original}$ vượt quá một ngưỡng dung sai (ví dụ 3%), hệ thống CI/CD phải lập tức cảnh báo đỏ (trigger alert) về lỗi gán nhãn hoặc sai `flip_idx`.
- **Đánh giá sai số chuẩn hóa theo từng keypoint riêng biệt ($err_i / \sqrt{area}$):** Bóc tách chính xác keypoint nào đang là "mắt xích yếu nhất" của mô hình.

### 4. Chuẩn hóa `kpt_oks_sigmas` theo quy trình khoa học (COCO Methodology)
- Không dùng hằng số cào bằng $\sigma = 1/K$ cho các bộ dữ liệu custom.
- Thực hiện đo độ lệch chuẩn sai số giữa nhiều chuyên gia gán nhãn độc lập trên tập mẫu $100$ ảnh: $\sigma_i = \text{std}(d_{i,k} / s_k)$. Bộ $\sigma$ này phải được đưa vào file cấu hình YAML (`kpt_oks_sigmas`) để đảm bảo điểm số OKS phản ánh đúng tính chất vật lý và sinh học của đối tượng.
