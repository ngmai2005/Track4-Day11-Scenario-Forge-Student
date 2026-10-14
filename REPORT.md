MSSV: 202602509

# Lab 11 · Scenario Forge — báo cáo

Thị trấn của bạn (`meta.json: _run.town`): **A**

---

## 1. Phát lại log có trung thực không? (bước A)

### Trích số từ `evidence/replay.json`:
- **Tổng số kịch bản log**: `n = 400`
- **Số kịch bản bị bỏ vì người lái gây va chạm trong log**: `dropped_human_crash_logs = 1`
- **Số kịch bản sử dụng để kiểm thử**: `n_used = 399`
- **Số va chạm khi planner chạy trên log phát lại (`replay`)**: `crashes = 104`, phân loại tiếp xúc:
  - `rear` (bị đâm sau): **96** vụ
  - `side` (va chạm bên hông): **8** vụ
- **Số va chạm trong thế giới phản ứng (`reactive`)**: `crashes = 0` (không có va chạm nào)
- **Số va chạm xuất hiện ở cả hai thế giới**: `crash_in_both = 0`

### Giải thích nguyên nhân va chạm và tính chất "giả":
- Phần lớn va chạm trong chế độ phát lại là va chạm đâm đuôi (`rear`: 96/104 vụ, chiếm 92.3%).
- Đây hoàn toàn là **va chạm giả (log replay artifact)** bắt nguồn từ bản chất vòng hở (open-loop) của dữ liệu log:
  1. Trong log phát lại, quỹ đạo và tốc độ của xe đi sau được cố định theo hành vi của người lái xe con người trong quá khứ. Các phương tiện này "mù" hoàn toàn đối với các hành động của xe tự lái (planner).
  2. Khi planner xử lý tình huống giao thông phía trước (như gặp người đi bộ qua đường hoặc chuẩn bị qua trạm xe buýt), planner chủ động giảm tốc hoặc phanh sớm hơn so với người lái thật.
  3. Xe phía sau trong log không có logic cảm nhận nên vẫn lao tới với vận tốc cũ và đâm sầm vào đuôi xe ta.
  4. Ngược lại, trong thế giới phản ứng closed-loop (`reactive`), xe phía sau được điều khiển bởi mô hình bám đuôi phản ứng (IDM). Khi xe ta phanh, xe sau cũng lập tức phát hiện và phanh theo, dẫn đến **0 va chạm** (`crashes = 0`).

### Phân tích một kịch bản cụ thể (`scenario: 2` trong `replay_only`):
- **Tham số kịch bản**: `cutT0 = 8.68s`, `cutDur = 5.36s`, `cutV = 14.71 m/s`, `cutDs = -3.96m`, `pedT0 = 5.25s`, `pedV = 1.05 m/s`, `busGo = 13.01s`, `tgT = 0.97s`, `cutGap = 13.44m`, `pedGap = 4.38m`, `mu = 0.8`, `pedLook = 0`, `cutBrake = 0.0`.
- **Diễn biến**:
  - Trong `replay`: planner chủ động phanh để nhường đường và giữ khoảng cách an toàn với người đi bộ xuất hiện ở `pedT0 = 5.25s`. Xe đi sau trong log người lái vẫn chạy trớn không phanh và đâm vào đuôi xe ta (`contact = "rear"`).
  - Trong `reactive`: xe đi sau quan sát thấy xe ta giảm tốc nên kích hoạt phanh IDM tương ứng, giữ khoảng cách thời gian an toàn tối thiểu cực kỳ lớn `min_ttc_reactive = 2.27s` (> 2.0s, hoàn toàn an toàn).

---

## 2. Kịch bản bất khả thi (bước B)

### Các luật trong `is_feasible` và căn cứ vật lý:
1. **Gia tốc dọc (tiếp tuyến) của xe**: $|a_{lon}| \le \mu g$.
   - *Căn cứ vật lý*: Vòng tròn ma sát Coulomb (friction circle). Lực phanh/tăng tốc tối đa truyền qua bề mặt tiếp xúc lốp - đường bị giới hạn bởi ma sát cực đại $F_{max} = \mu m g$. Một phương tiện không thể giảm tốc hoặc tăng tốc vượt quá $\mu g$ mà không bị trượt lết mất kiểm soát.
2. **Gia tốc ngang khi chuyển làn (theo mô hình cosine lane-change)**:
   $$a_{lat, max} = \frac{\pi^2 W}{2 T_{dur}^2} \le 0.3 \mu g \quad (\text{với } W = 3.75\text{m là độ rộng làn})$$
   - *Căn cứ vật lý*: Giới hạn gia tốc hướng tâm khi chuyển làn. Để xe chuyển làn ổn định, không bị lật (rollover) hoặc mất lái văng đuôi (spin-out), gia tốc ngang chỉ được chiếm một phần nhỏ trong elip ma sát (quy chuẩn an toàn giao thông giới hạn ở $0.3 \mu g$).
3. **Giới hạn tốc độ người đi bộ**: $v_{ped} \in [0.5, 2.5]$ m/s.
   - *Căn cứ vật lý*: Sinh cơ học vận động người đi bộ đô thị (pedestrian biomechanics). Tốc độ đi bộ bình thường là $1.0 - 1.5$ m/s, chạy gấp qua đường đạt tối đa $2.5$ m/s ($\approx 9$ km/h). Vận tốc vượt quá $2.5$ m/s thuộc về vận động viên điền kinh, phi thực tế trong bối cảnh người dân qua đường đô thị.
4. **Giới hạn tốc độ xe khác**: $v_{veh} \in (0, 22]$ m/s ($\approx 80$ km/h).
   - *Căn cứ vật lý*: Tốc độ lưu thông hợp lý trong mạng lưới đường phố đô thị. Vận tốc phải dương (xe không đi lùi đột ngột trên đường chính) và không vượt quá tốc độ thiết kế an toàn của trục phố.
5. **Thời gian chuyển làn**: $T_{dur} \in [1.5, 9.0]$ s.
   - *Căn cứ vật lý*: Thời gian cần thiết để một phương tiện thực hiện chuyển đổi thân xe sang làn bên cạnh một cách liên tục.

### Thống kê kịch bản bất khả thi trong `evidence/hunt.json`:
- `bus_bypass`: 4 kịch bản bất khả thi (`infeasible = 4`, `found = 149`)
- `cut_in`: 5 kịch bản bất khả thi (`infeasible = 5`, `found = 16 bus_bypass`)
- `crosswalk_block`: **21 kịch bản bất khả thi** (`infeasible = 21`, `found = 47`) $\rightarrow$ **Sinh ra nhiều kịch bản bất khả thi nhất**.
- `rear_ended`: 2 kịch bản bất khả thi (`infeasible = 2`, `found = 3 bus_bypass, 2 crosswalk_block`)

### Vì sao CEM bị kéo về phía kịch bản bất khả thi:
- Thuật toán Cross-Entropy Method (CEM) tối ưu hóa phân phối đề xuất dựa trên điểm đánh giá độ nguy hiểm (surrogate danger score, ví dụ nghịch đảo TTC hoặc mức độ chồng lấn hình học).
- Ở kịch bản `crosswalk_block`, nếu tham số đẩy tốc độ người đi bộ lên siêu cao hoặc bắt xe phanh gấp vô lý ($a > \mu g$), độ nguy hiểm và mức cản trở sẽ đạt giá trị cực đại. Do đó, hàm mục tiêu của CEM tự nhiên bị "hút" mạnh về phía các góc biên bất khả thi này để tối đa hóa điểm phạt.

### Hậu quả nếu đếm kịch bản bất khả thi là va chạm:
- Nếu đếm cả các kịch bản vi phạm vật lý, xác suất hỏng $P(fail)$ sẽ bị **thổi phồng nghiêm trọng (overestimation / false alarm)**.
- Trọng số Importance Sampling sẽ gán xác suất cho những điều kỳ diệu phi vật lý, làm sai lệch bức tranh an toàn và khiến kỹ sư mất tài nguyên tối ưu cho những kịch bản không bao giờ xuất hiện ngoài đời thực.

---

## 3. Monte Carlo cho bạn biết gì (bước C1)

### Trích số từ `evidence/mc.json`:
- Số lần chạy: `n = 3000`
- Số va chạm ghi nhận: `crashes = 4` ($k = 4$)
- Tần suất mẫu: $\hat{p} = \frac{4}{3000} \approx \mathbf{0.001333}$ ($1.33 \times 10^{-3}$)
- Khoảng tin cậy Wilson 95%:
  - `lo = 0.0005186172629797156` ($5.19 \times 10^{-4}$)
  - `hi = 0.0034235346800521676` ($3.42 \times 10^{-3}$)
- Phân loại theo kiểu thất bại:
  - `bus_bypass`: `k = 4` (`lo = 5.19e-4`, `hi = 3.42e-3`)
  - `cut_in`: `k = 0`
  - `crosswalk_block`: `k = 0`
  - `rear_ended`: `k = 0`
  - `other`: `k = 0`

### Số lần chạy Monte Carlo cần để khoảng tin cậy hẹp lại còn $\pm 20\%$ của $p$:
- Sai số tương đối mong muốn: $E = 0.20 \times p$.
- Với độ tin cậy 95% ($z \approx 1.96$):
  $$E = z \sqrt{\frac{p(1-p)}{n}} \implies 0.20 p \approx 1.96 \sqrt{\frac{p}{n}} \implies (0.04) p^2 \approx \frac{3.8416 p}{n}$$
  $$n \approx \frac{3.8416}{0.04 \times p} = \frac{96.04}{p}$$
- Thay $\hat{p} = \frac{4}{3000} \approx 0.0013333$:
  $$n \approx \frac{96.04}{0.0013333} \approx \mathbf{72,030} \text{ lần chạy}$$
- Cần hơn **72,000 lần chạy** (gấp 24 lần ngân sách 3,000 hiện tại) chỉ để xác định $p$ với độ chính xác $\pm 20\%$.

### Các kiểu thất bại chưa hề xuất hiện trong 3 000 lần chạy:
- Có **4 kiểu** thất bại chưa từng xuất hiện ($k = 0$): `cut_in`, `crosswalk_block`, `rear_ended`, và `other`.

### Con số đúng cần báo cho kiểu chưa thấy:
- **Tuyệt đối không được báo bằng 0** (vì không quan sát thấy trong 3,000 mẫu không đồng nghĩa với xác suất bằng 0 ngoài đời).
- Con số chuẩn xác cần báo là **cận trên khoảng tin cậy Wilson 95%** khi $k = 0$ (hoặc quy tắc $3/n$):
  - Theo `mc.json: per_mode.*.hi`: Cận trên Wilson là **`hi = 0.0012788956648046956`** ($1.28 \times 10^{-3}$, tức báo $p \le 1.28 \times 10^{-3}$).
  - Theo quy tắc $3/n$: $p \le \frac{3}{3000} = 1.0 \times 10^{-3}$.

---

## 4. Importance sampling, ESS và độ suy biến (bước C3)

### Bảng kết quả từng kiểu và từng seed trong `evidence/estimate.json`:

| Kiểu thất bại (`mode`) | Seed | Xác suất $p$ | Số va chạm (`hits`) | Cỡ mẫu hiệu dụng (`ess`) | Tỉ phần trọng số đỉnh (`top_share`) |
|---|---|---|---|---|---|
| **`bus_bypass`** | 0 | $8.53 \times 10^{-4}$ | 48 | 121.30 | 0.363 |
| **`bus_bypass`** | 1 | $9.73 \times 10^{-4}$ | 49 | 118.13 | 0.323 |
| **`bus_bypass`** | 2 | $1.23 \times 10^{-3}$ | 45 | 119.03 | 0.243 |
| **`cut_in`** | 0 | 0.0 ($< 0.025$) | 0 | 120.48 | 0.0 |
| **`cut_in`** | 1 | 0.0 ($< 0.025$) | 0 | 120.03 | 0.0 |
| **`cut_in`** | 2 | 0.0 ($< 0.025$) | 0 | 113.56 | 0.0 |
| **`crosswalk_block`** | 0 | $3.81 \times 10^{-7}$ | 83 | 130.22 | **0.802** |
| **`crosswalk_block`** | 1 | $8.29 \times 10^{-3}$ | 83 | 112.15 | **0.99996** |
| **`crosswalk_block`** | 2 | $3.78 \times 10^{-7}$ | 69 | 121.02 | **0.680** |
| **`rear_ended`** | 0 | 0.0 ($< 0.025$) | 0 | 125.62 | 0.0 |
| **`rear_ended`** | 1 | 0.0 ($< 0.025$) | 0 | 100.74 | 0.0 |
| **`rear_ended`** | 2 | 0.0 ($< 0.025$) | 0 | 140.07 | 0.0 |

### Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không?
- **Với `bus_bypass`**: Ba seed **rất đồng ý với nhau** (dao động chặt chẽ quanh $0.85 \times 10^{-3} - 1.23 \times 10^{-3}$) và **hoàn toàn đồng ý với Monte Carlo** trong `mc.json` ($p = 1.33 \times 10^{-3}$, CI 95% $[5.19 \times 10^{-4}, 3.42 \times 10^{-3}]$).
- **Với `crosswalk_block`**: Seed 0 ($3.81 \times 10^{-7}$) và Seed 2 ($3.78 \times 10^{-7}$) đồng ý với nhau rằng đây là biến cố siêu hiếm. Tuy nhiên, Seed 1 nhảy vọt lên $8.29 \times 10^{-3}$ do hiện tượng suy biến trọng số nghiêm trọng.
- **Với `cut_in` và `rear_ended`**: Cả 3 seed đều cho 0 hits, xác nhận xác suất không vượt quá cận trên $0.025$ (`upper_zero_hits`).

### Ở đâu `top_share` lớn (> 0.5), và vì sao con số thực ra chỉ là trọng số của một mẫu?
- `top_share > 0.5` xuất hiện tại tất cả các seed của `crosswalk_block`:
  - Seed 0: `top_share = 0.802`
  - Seed 1: `top_share = 0.999959` ($\approx 100\%$)
  - Seed 2: `top_share = 0.680`
- **Giải thích**:
  - Tại Seed 1, trong 83 mẫu va chạm lấy từ phân phối đề xuất $q(x)$, có một mẫu đơn lẻ $x^*$ vô tình rơi vào vùng có mật độ tự nhiên $p(x^*)$ tương đối cao so với vùng đuôi của $q(x^*)$.
  - Trọng số tầm quan trọng của mẫu này $w^* = \frac{p(x^*)}{q(x^*)}$ lớn khủng khiếp, chiếm tới **99.996%** tổng trọng số của cả 83 mẫu cộng lại.
  - Hậu quả là ước lượng $\hat{p} = \frac{1}{n} \sum w_i \approx \frac{w^*}{n}$ thực chất chỉ phản ánh trọng số của đúng một mẫu đơn độc đó, chứ không phải một giá trị kỳ vọng ổn định (weight degeneracy).

### So ngân sách (`budget`) với số lần chạy Monte Carlo:
- Ngân sách Importance Sampling: `estimate.json: budget = 16800` lần chạy.
- So với Monte Carlo:
  - Để ước lượng `bus_bypass` ($p \approx 10^{-3}$) cần khoảng $72,000$ lần chạy.
  - Để đo được biến cố hiếm cỡ $10^{-7}$ như `crosswalk_block` (Seed 0, Seed 2), Monte Carlo cần tối thiểu $n \ge \frac{100}{p} \approx 10^9$ (1 tỷ lần chạy!).
  - IS cho phép đo các biến cố cực hiếm với chi phí tính toán nhỏ hơn hàng chục nghìn đến hàng triệu lần.

### Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó?
- `other` được lấy trực tiếp từ `evidence/mc.json` (`estimate.json: other.source = "mc.json"`), với $k = 0, n = 3000, p = 0.0, hi = 0.00127889566$.
- **Lý do không săn `other`**: "other" là nhóm phần dư không có cấu trúc tham số xác định (unparameterized residual category). CEM đòi hỏi một hàm mục tiêu cụ thể (score function) để định hướng dịch chuyển tâm phân phối Gaussian $q(x)$; do không có định nghĩa hình học hay mục tiêu cụ thể cho "other", ta không thể thiết lập hàm mục tiêu để săn nó.

---

## 5. Các kiểu thất bại (bước C2 + 3D)

### 1. Kiểu `bus_bypass` (lách qua xe buýt):
![bus_bypass](evidence/shots/bus_bypass.jpg)
- **Chuỗi sự kiện**: Xe buýt dừng tại trạm ở làn đối diện che khuất hoàn toàn tầm nhìn phía trước; một xe đối diện quyết định lấn làn vượt xe buýt với vận tốc cao; xe ta đi thẳng với tầm nhìn bị xe buýt chắn ngang; khi xe ta vừa nhìn thấy xe vượt thì khoảng cách đã quá gần, phanh không kịp và xảy ra va chạm trực diện.

### 2. Kiểu `crosswalk_block` (chặn lối trên vạch sang đường):
![crosswalk_block](evidence/shots/crosswalk_block.jpg)
- **Chuỗi sự kiện**: Người đi bộ băng qua đường trên vạch sang đường; xe ta phanh giảm tốc để nhường đường nhưng dừng quá sát hoặc dừng lấn vào vạch đi bộ đúng lúc người đi bộ bước tới, khiến đầu xe ta chắn ngang lối đi bộ và gây ra lỗi chặn lối người đi bộ.

### Kiểu cuộc săn không tìm thấy:
- Cuộc săn **không tìm thấy** 2 kiểu: `cut_in` và `rear_ended`.
- **Mức độ tin cậy**: Rất tin cậy trong phạm vi không gian tham số của bài toán (`space.py`). Trong mô phỏng khép vòng, các xe khác bám đuôi theo mô hình IDM nên luôn phanh an toàn theo xe ta (không thể tông đuôi); đồng thời vùng nhận diện và nhường đường của xe ta đủ rộng để xử lý các xe cắt làn nằm trong giới hạn gia tốc vật lý.

### Điều gì xảy ra nếu chỉ chạy một cuộc săn chung cho mọi kiểu?
- Nếu gộp chung một cuộc săn, hiện tượng **suy biến mốt (mode collapse)** sẽ xảy ra: CEM sẽ bị hút toàn bộ về mốt va chạm có gradient dốc nhất và dễ kích hoạt nhất là `bus_bypass` (`hunt.json` tìm được tới 149 vụ `bus_bypass`), trong khi kiểu hiếm và tinh vi hơn như `crosswalk_block` (chỉ 47 vụ) sẽ bị lấn át và bỏ sót hoàn toàn.
- Minh chứng rõ ràng trong `hunt.json`: ngay cả khi săn riêng `cut_in` hay `rear_ended`, CEM vẫn bị trượt sang tìm thấy `bus_bypass` (`cut_in.found = {"bus_bypass": 16}`, `rear_ended.found = {"bus_bypass": 3, "crosswalk_block": 2}`). Do đó, việc tách riêng từng cuộc săn với hàm mục tiêu độc lập là điều kiện bắt buộc để bao phủ đa dạng các mốt thất bại.

---

## 6. Sửa planner (bước E + 3D)

### Bảng kết quả sửa planner trong `evidence/fix.json`:
- **Chính sách được chọn (`chosen`)**: `{"cutinLat": 3.0, "cautionV": 7.0}`
  (Mở rộng vùng quét ngang nhận diện xe cắt làn từ 2.15m lên 3.0m, hạ vận tốc cảnh báo qua khu vực nguy hiểm từ 7.5 m/s xuống 7.0 m/s).

| Bộ dữ liệu | Policy | `fail` | `fail_hi_seen` | `fail_hi` | Quãng đường (`progress`) | Phanh gấp (`harsh`) |
|---|---|---|---|---|---|---|
| **TRAIN** | Bản sửa (`chosen`) | $9.23 \times 10^{-7}$ | $2.21 \times 10^{-6}$ | $2.21 \times 10^{-6}$ | 201.63 m | 9.20 |
| **TRAIN** | Bản gốc (`baseline_train`) | $3.33 \times 10^{-3}$ | $7.70 \times 10^{-3}$ | $7.70 \times 10^{-3}$ | 209.90 m | 9.09 |
| **TEST** | **Bản sửa (`chosen_test`)** | **$1.33 \times 10^{-5}$** | **$3.69 \times 10^{-5}$** | **$3.88 \times 10^{-2}$** | **207.04 m** | **9.18** |
| **TEST** | **Bản gốc (`baseline_test`)** | **$5.82 \times 10^{-3}$** | **$1.63 \times 10^{-2}$** | **$1.75 \times 10^{-2}$** | **214.76 m** | **8.36** |

### Vì sao `fail_hi` của bản sửa lại lớn hơn `fail` của policy gốc trên TEST dù bản sửa tốt hơn thật?
- Trên tập TEST, bản sửa hoạt động quá xuất sắc nên **không gặp bất kỳ va chạm `bus_bypass` nào** (`hits = 0`).
- Khi một kiểu thất bại có $0$ hits, hệ thống đánh giá áp dụng quy tắc chặn trên bảo thủ: cộng thêm lượng phạt $\frac{3}{\beta n} = \frac{3}{0.2 \times 400} = \mathbf{0.0375}$ vào `fail_hi`.
- Do đó, `fail_hi` của bản sửa bị đội lên thành $0.0388$, đây thuần túy là hệ quả của công thức toán học bảo thủ khi cỡ mẫu hữu hạn ($n=400$), không phản ánh năng lực thực tế của xe.
- Khi nhìn vào **`fail_hi_seen`** (chỉ tính cận trên của các kiểu thực sự quan sát thấy), bản sửa đạt **$3.69 \times 10^{-5}$**, an toàn gấp **hơn 440 lần** so với bản gốc ($1.63 \times 10^{-2}$)! Xác suất hỏng thực tế `fail` giảm từ $5.82 \times 10^{-3}$ xuống $1.33 \times 10^{-5}$.

### Quãng đường và số lần phanh gấp:
- Quãng đường đi được chỉ giảm rất nhẹ từ 214.76 m xuống 207.04 m (giảm $\approx 3.6\%$, hoàn toàn thỏa mãn ràng buộc $\le 5\%$).
- Số lần phanh gấp tăng nhẹ từ 8.36 lên 9.18 lần/chuyến do xe chủ động rà phanh phòng thủ khi qua khu vực khuất tầm nhìn quanh trạm xe buýt.

### Ảnh chụp 3D có bóng mờ (ghost car):
![fix_ghost](evidence/shots/fix_ghost.jpg)
- *Mô tả*: Xe ta với chính sách mới (màu xanh lục đậm) chủ động hạ tốc độ xuống 7.0 m/s trước vùng mù của xe buýt nên kịp thời dừng lại cách mũi xe vượt một khoảng an toàn; trong khi đó, bóng mờ của chính sách mặc định (ghost car màu mờ) duy trì tốc độ 7.5 m/s đã không phanh kịp và đâm trực diện vào xe vượt.

### Núm chỉnh "nghe an toàn hơn" nhưng thực tế làm tăng thất bại:
- Núm chỉnh **hạ `cautionV` quá sâu (xuống $5.0$ m/s)**:
  - Xem dòng `cautionV = 5.0` trong bảng `table` của `fix.json`: Quãng đường tụt nghiêm trọng xuống còn **170.57 m** (giảm gần 20%), trong khi số lần phanh gấp tăng vọt lên **12.29 lần**.
  - *Giải thích bằng phản ứng của thế giới*: Khi xe ta phanh quá đột ngột về tốc độ rùa bò giữa làn đường, xe ta trở thành một vật cản bất ngờ cho các phương tiện đi phía sau trong dòng giao thông, kích động các xe khác lách vượt ẩu; đồng thời việc bò chậm kéo dài thời gian xe ta chiếm dụng vùng giao cắt, làm tăng diện tích tiếp xúc xung đột theo thời gian với người đi bộ và các xe chuyển làn.

---

## 7. Từ mô phỏng ra đường thật

Ba giả định then chốt trong `space.py` ảnh hưởng nhiều nhất đến con số đánh giá:

1. **Khoảng cách chấp nhận khi cắt làn (`cutGap`)**:
   - *Dữ liệu thật thay thế*: Phân phối khoảng cách chấp nhận nhập làn (gap acceptance distribution) trích xuất từ tập dữ liệu xe tự hành quy mô lớn (Waymo Open Dataset, nuScenes, hoặc Argoverse 2) tại các cung đường đô thị thực tế.
   - *Hướng sai lệch*: Nếu ngoài đời tài xế lái xe liều lĩnh hơn (chấp nhận khe hở nhỏ hơn giả định), con số xác suất va chạm của ta trong mô phỏng sẽ bị **lạc quan quá mức (underestimated)**.
2. **Tỉ lệ người đi bộ quan sát trước khi bước qua xe buýt (`pedLook`)**:
   - *Dữ liệu thật thay thế*: Dữ liệu video từ camera an ninh đô thị hoặc camera hành trình xe buýt công cộng, sử dụng mô hình nhận diện tư thế và hướng nhìn mắt (pedestrian head pose & gaze estimation).
   - *Hướng sai lệch*: Trong mô phỏng, tỉ lệ này được phân bổ đều. Nếu ngoài đời người đi bộ thường xuyên cắm mặt vào điện thoại hoặc vội vã băng qua mà không nhìn ($pedLook = 0$ chiếm ưu thế), xác suất va chạm thực tế sẽ cao hơn nhiều so với báo cáo (**underestimated**).
3. **Hệ số ma sát mặt đường (`mu`)**:
   - *Dữ liệu thật thay thế*: Dữ liệu đo đạc hệ số ma sát động thực tế từ hệ thống cân bằng điện tử (ESC/ABS) của các đội xe chạy thương mại kết hợp với dữ liệu quan trắc khí tượng thủy văn theo thời gian thực (đường khô $\mu \approx 0.8$, mưa ướt $\mu \approx 0.4 - 0.5$, đường dầu mỡ $\mu \approx 0.3$).
   - *Hướng sai lệch*: Nếu phân phối mô phỏng thiên lệch về đường khô ($\mu = 0.8$), quãng đường phanh khẩn cấp ngoài đời thực tế sẽ dài hơn nhiều trong điều kiện mưa gió nhiệt đới, khiến planner thực tế gặp nhiều tai nạn hơn con số báo cáo (**underestimated**).

---

## 8. Giới hạn

Ba giới hạn cốt lõi của thế giới mô phỏng này:
1. **Hình học không gian tuyến tính đơn điệu**: Toàn bộ thế giới mô phỏng chỉ gồm một đoạn đường thẳng cố định với một vạch sang đường và một trạm xe buýt. Mô phỏng hoàn toàn vắng bóng các yếu tố hình học đô thị phức tạp như ngã tư có đèn tín hiệu, ngã ba khuất tầm nhìn do nhà cao tầng, bùng binh, đường dốc, góc cua hẹp hoặc vỉa hè có cây cối che khuất.
2. **Hành vi tác nhân giao thông đơn giản hóa (Pure IDM & Cosine)**: Các phương tiện xung quanh chỉ tuân theo mô hình bám đuôi IDM một chiều và quỹ đạo chuyển làn hàm cosine định sẵn. Ngoài đời, người lái xe có hành vi bất quy tắc: bấm còi, nháy pha xin đường, lách chéo bất ngờ, quay đầu xe tùy tiện, hoặc phản ứng hoảng loạn/chậm chạp khi say xỉn.
3. **Giả định cảm biến và nhận thức lý tưởng (Perception Oracle)**: Vị trí và vận tốc của mọi đối tượng được cung cấp hoàn hảo ngay lập tức khi thoát khỏi vùng che khuất hình học. Mô phỏng bỏ qua hoàn toàn độ trễ đường truyền cảm biến (sensor latency), độ trễ suy luận mạng nơ-ron nhận thức, nhiễu điểm đám mây LiDAR khi trời mưa, góc chết camera do chói nắng, và hiện tượng mất dấu vết đối tượng (tracking ID switch).

### Kế hoạch đo đạc nếu có thêm một tuần:
- Xây dựng mô hình **nhiễu nhận thức thực tế (Perception Noise & Latency Injection)** đưa độ trễ nhận diện 100–200 ms và sai số kích thước hộp bao vào vòng lặp mô phỏng.
- Mở rộng không gian kịch bản sang **nút giao ngã tư có đèn tín hiệu (traffic light intersection)** để săn tìm các tình huống lưỡng lự vượt đèn vàng (dilemma zone) và các phương tiện rẽ trái cắt đầu xe đi thẳng.

---

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối**:
  - Dựng tình huống **người đi bộ băng cắt tự phát giữa đường ngoài vạch quy định (Jaywalking Pedestrian)** từ lề đường phía Bắc sang lề đường phía Nam tại vị trí $x = 95.0$m (nằm ở đoạn giữa phố, trước vạch sang đường quy định tại $x = 122.0$m), khi xe ta đang lưu thông với tốc độ hành trình ~11 m/s.
  - Ngoài đời, tình huống này cực kỳ phổ biến tại các đô thị ở Việt Nam, nơi người đi bộ thường xuyên băng qua đường bất cứ đâu thuận tiện thay vì đi bộ đến vạch kẻ đường, đặc biệt khi bước ra từ các điểm mù giữa phố.
- **Căn cứ vì sao planner xử lý kém**:
  - Trong mã nguồn của planner (`_closest_front_obstacle` trong `avsim`), đối với người đi bộ ở ngoài vạch sang đường (`not on_crosswalk`), planner **chỉ coi người đi bộ là chướng ngại vật khi $|lat| \le 1.9$m** (tức là khi người đó đã đặt chân vào hẳn lòng làn đường của xe ta).
  - Khi người đi bộ đang đi từ lề đường phía Bắc ($lat = 7.0$m xuống $2.0$m), planner hoàn toàn ngó lơ và không hề giảm tốc độ phòng thủ.
  - Đến khi người đi bộ chạm mốc $|lat| \le 1.9$m, khoảng cách giữa xe ta và người đi bộ đã quá gần ở tốc độ cao; gia tốc phanh tiện nghi $b = 2.0$ m/s$^2$ của IDM không thể dừng kịp, dẫn đến va chạm chết người.
- **Giả định `space` và `exposure` (`hazard.json: why`)**:
  - `space`:
    - `h0_x`: $[92.0, 98.0]$m (đoạn giữa đường trước vạch sang đường $x = 122$m).
    - `h0_v`: $[1.2, 1.6]$ m/s (vận tốc đi bộ tự nhiên của người trưởng thành theo chuẩn nhân trắc học đô thị).
    - `h0_trig`: $[72.0, 84.0]$m (cự ly kích hoạt khi xe ta cách chướng ngại vật 72–84m, vừa đủ thời gian để người đi bộ băng qua làn đối diện và chạm vào mép làn xe ta đúng thời điểm xe ta trờ tới).
  - `exposure`: $0.03$ (tần suất gặp tình huống người băng cắt ngoài vạch ước tính khoảng 3% số chuyến đi nội thị, dựa trên thống kê của Ủy ban An toàn Giao thông Quốc gia về tỉ lệ tai nạn người đi bộ ngoài vạch chiếm ~25-30% tổng số tai nạn người đi bộ đô thị).
- **Kết quả theo `evidence/mine.json`**:
  - Tỉ lệ kịch bản hợp lệ: `accepted = 0.9966666666666667` (**99.67%**, 1495/1500 kịch bản khả thi, vượt xa yêu cầu $\ge 50\%$).
  - Tái hiện đúng kịch bản gốc: `exact: [["ego", "H0", 12.7]]` (xe ta đâm trúng H0 tại $t = 12.7$s).
  - So sánh chi tiết giữa Policy gốc và Policy đã chọn:

| Chỉ số | Policy gốc (`baseline`) | Policy đã chọn (`chosen`) |
|---|---|---|
| **Số kịch bản thử nghiệm (`n`)** | 1495 | 1495 |
| **Tổng số va chạm (`crashes`)** | **61** | **56** |
| • Tránh được (`avoidable`) | 60 | 56 |
| • Không thể tránh (`unavoidable`) | 0 | 0 |
| • Chưa phân loại (`unattributed`) | 1 | 0 |
| • Tốc độ thấp (`low_speed`) | 1 | 8 |
| **Xác suất lỗi có điều kiện $P(\text{fail} \mid \text{hazard})$** | **$4.01 \times 10^{-2}$** ($4.01\%$) | **$3.75 \times 10^{-2}$** ($3.75\%$) |
| • Khoảng tin cậy Wilson 95% | $[3.13 \times 10^{-2}, 5.13 \times 10^{-2}]$ | $[2.90 \times 10^{-2}, 4.83 \times 10^{-2}]$ |
| **Tần suất va chạm mỗi chuyến (`per_trip`)** | **$1.20 \times 10^{-3}$** | **$1.12 \times 10^{-3}$** |
| • Cận trên tần suất mỗi chuyến (`per_trip_hi`) | $1.54 \times 10^{-3}$ | $1.45 \times 10^{-3}$ |

  - **Đặt cạnh tần suất lách xe buýt (`bus_bypass`)**:
    - Tần suất va chạm của rắc rối này là **$1.20 \times 10^{-3}$** (gốc) và **$1.12 \times 10^{-3}$** (sửa), ngang ngửa với tần suất va chạm lách xe buýt **$1.33 \times 10^{-3}$** trong `mc.json` và $\sim 1.0 \times 10^{-3}$ trong `estimate.json`! Điều này chứng minh đây là một lỗ hổng an toàn mang tính hệ thống cực kỳ nghiêm trọng ngang hàng với kịch bản xe vượt ẩu qua xe buýt.
  - **Vì sao "tránh được" (`avoidable`) là một chuẩn lạc quan (phân tích theo hai hướng)**:
    1. *Hướng lạc quan (thổi phồng số va chạm tránh được)*: Phân loại `avoidable` sử dụng chuẩn "oracle" biết trước chướng ngại vật ngay khi xuất hiện và giả định xe phanh kịch sàn với gia tốc ma sát tối đa $\mu g$ mà không có độ trễ nhận thức hay phản xạ con người. Do đó, nó đếm nhiều hơn số va chạm mà một planner thực tế có thể tránh được.
    2. *Hướng bi quan (hạn chế hành vi tránh)*: Chuẩn `avoidable` chỉ thử nghiệm duy nhất hành động phanh khẩn cấp trên đường thẳng (longitudinal emergency braking), không thử nghiệm đánh lái tránh vật cản (lateral evasive steering). Ngoài đời, một tài xế hoặc planner có thể đánh lái nhẹ sang khoảng trống bên cạnh để tránh người đi bộ mà không cần dừng hẳn xe.
- **Hình ảnh lúc va chạm**:
![hazard_crash](evidence/shots/hazard_crash.jpg)
- **Bạn tự lái thì sao**:
  - Người lái **tránh được hoàn toàn 100%**!
  - *Sự khác biệt giữa người lái và planner*: Người lái có nhận thức không gian ngữ cảnh (predictive spatial awareness). Ngay khi thấy bóng người đi bộ bước từ vỉa hè xuống mép đường phía đối diện, người lái đã chủ động nhấc chân ga và rà nhẹ chân phanh phòng thủ (defensive deceleration), chứ không chờ người đi bộ bước hẳn vào làn đường xe mình ($|lat| \le 1.9$m) như planner.
- **Cách sửa**:
  - *Núm chỉnh có sửa được không?*: Các núm chỉnh hiện tại (`cautionV`, `cutinLat`, `pedBuffer`) **không thể giải quyết triệt để** rắc rối này (số va chạm vẫn còn tới 56 vụ, và 100% trong số đó đều là `avoidable`). Lý do là các núm này chỉ can thiệp vào vạch sang đường (`cautionRange`) hoặc xe ô tô cắt làn (`cutinLat`).
  - *Logic mới cần bổ sung cho planner*:
    1. **Vùng nhận diện lấn làn dự đoán (Predictive Pedestrian Encroachment)**: Planner cần tính toán vận tốc ngang $v_y$ của người đi bộ. Nếu người đi bộ đang di chuyển hướng vào làn đường của xe ($v_y < 0$ từ lề Bắc) và thời gian dự kiến cắt ngang tim đường (Time-to-Encroachment) nằm trong phạm vi dừng an toàn, planner phải coi người đó là chướng ngại vật ảo ngay lập tức và tiến hành phanh giảm tốc đón đầu.
    2. **Hành vi lái phòng thủ (Defensive Urban Speed Limit)**: Khi di chuyển qua các cung đường đô thị có người đi bộ ở hai bên vỉa hè, planner cần tự động giới hạn tốc độ trần thấp hơn và mở rộng nón chú ý (attention cone) ra ngoài phạm vi biên làn xe.
