MSSV: 2A202602757

# Lab 11 · Scenario Forge — báo cáo

Thị trấn (`meta.json: _run.town`): **C** (`cutGap` trung vị 10 m, `pedLook` 0,80). Mọi số trích từ `evidence/` của lần chạy đầy đủ (không `--quick`).

## 1. Phát lại log có trung thực không?

| `replay.json` | Số |
|---|---|
| `n` / `dropped_human_crash_logs` / `n_used` | 400 / 0 / 400 |
| Phát lại (`replay.crashes`) | **111** (`rear` 93, `side` 18) |
| Phản ứng (`reactive.crashes`) | **0** |
| `crash_in_both` | 0 |

Cả 111 vụ là va chạm **giả**. Planner thận trọng hơn người lái (giảm tốc sớm trước vạch bị xe buýt che, `T` = 1,6 s so với 1,1 s), nhưng xe bám đuôi TG chạy đúng quỹ đạo trong log, tức là bám theo người lái. Log không biết planner phanh sớm hơn, nên TG không phanh và tông vào đuôi.

Ví dụ `replay_only[0]` (kịch bản 2, `tgT` = 0,70 s): từ t ≈ 9,5 s planner chậm hơn người lái hơn 1 m/s. Lúc t = 17,1 s, xe ta còn 1,18 m/s thì TG (2,23 m/s) tông vào đuôi. Trong thế giới phản ứng, TTC nhỏ nhất là 2,37 s, không va chạm.

## 2. Kịch bản bất khả thi

| Luật trong `is_feasible` | Căn cứ vật lý |
|---|---|
| Gia tốc ngang khi chuyển làn (π/cutDur)²·3,5/2 > μ·g | Lốp không truyền được lực ngang quá μ·m·g, xe sẽ trượt. |
| `cutBrake` > μ·g | Không phanh mạnh hơn ma sát lốp–đường (μ 0,8 khô, 0,5 ướt). |
| `pedV` > 2,5 m/s | Người đi bộ băng đường ~1,2–1,5 m/s; trên 2,5 m/s là chạy. |
| `cutV` ngoài (0, 22] m/s | Xe phải đang chạy; 22 m/s (~80 km/h) vượt xa tốc độ phố nội thành. |

`hunt.json: infeasible`: `bus_bypass` 2, `cut_in` 4, `crosswalk_block` 8, **`rear_ended` 13**. `mc.json: infeasible` = 19/3 000.

Cuộc săn `rear_ended` bị kéo về vùng bất khả thi nhiều nhất. Muốn xe ta bị tông đuôi thì xe ta phải phanh gấp, và cách "nguy hiểm" nhất theo điểm số thường là cắt làn quá gắt hoặc phanh quá μ·g. Điểm nguy hiểm không biết vật lý. Nếu đếm các kịch bản này là va chạm, ước lượng bị thổi phồng; với IS chúng còn nằm ở đuôi p nên mang trọng số lớn và có thể chi phối cả con số.

## 3. Monte Carlo

`mc.json`: k = **12**, n = **3 000**, p̂ = **4,00·10⁻³**, Wilson 95 % **[2,29·10⁻³; 6,98·10⁻³]**, toàn bộ là `bus_bypass`.

Để khoảng tin cậy còn ±20 % của p: n ≈ 96/p ≈ **24 000** lần chạy.

`cut_in`, `crosswalk_block`, `rear_ended`, `other` chưa xuất hiện (k = 0). Không báo 0 cho các kiểu này mà báo cận trên Wilson: P < **1,28·10⁻³**.

## 4. Importance sampling

`estimate.json`, n = 600 mẫu mỗi kiểu mỗi seed. `cut_in` và `rear_ended` có 0 hits ở cả 3 seed (ess 115–138), nên báo cận trên `upper_zero_hits` = **2,5·10⁻²**, không báo 0.

| Seed | Kiểu | p | hits | ess | top_share |
|---|---|---|---|---|---|
| 0 | bus_bypass | 1,07·10⁻³ | 31 | 125 | 0,211 |
| 0 | crosswalk_block | 6,65·10⁻⁷ | 53 | 140 | 0,497 |
| 1 | bus_bypass | 4,03·10⁻⁴ | 22 | 138 | 0,461 |
| 1 | crosswalk_block | 2,01·10⁻⁷ | 62 | 127 | 0,231 |
| 2 | bus_bypass | 5,86·10⁻⁴ | 37 | 155 | 0,136 |
| 2 | crosswalk_block | 1,82·10⁻⁷ | 52 | 138 | 0,238 |

`total`: 1,07·10⁻³ / 4,03·10⁻⁴ / 5,87·10⁻⁴ (trung vị ≈ 5,9·10⁻⁴).

- **Giữa các seed:** cùng bậc nhưng chênh 2,7 lần (`bus_bypass`) và 3,7 lần (`crosswalk_block`).
- **So với MC:** `bus_bypass` theo IS thấp hơn MC 4–10 lần và nằm ngoài khoảng Wilson. Cuộc săn co quá hẹp (`stopped_tightening = true`), q bỏ sót một phần vùng va chạm, nên IS bị chệch thấp, đúng cái bẫy README đã nêu.
- **`top_share`:** cao nhất 0,497 và 0,461, tức gần nửa ước lượng nằm ở **một** mẫu. Khi đó con số thực chất là trọng số p/q của mẫu đó, và ±1,96·se có thể hẹp sai.
- **Ngân sách:** `budget` = **16 800**. Để đạt cùng sai số tương đối, MC cần ~9 000–29 000 lần cho `bus_bypass` (IS lợi ít), nhưng ~4·10⁶–6·10⁷ lần cho `crosswalk_block` (IS rẻ hơn 2–3 bậc).
- **`other`:** lấy từ MC (k 0/3 000, cận trên 1,28·10⁻³). Kiểu này không được săn vì không có điểm nguy hiểm liên tục để CEM leo dốc.

## 5. Các kiểu thất bại

`hunt.json: found`: săn `bus_bypass` → 120 `bus_bypass`; săn `crosswalk_block` → 32 `crosswalk_block`; săn `cut_in` → 28 `bus_bypass`; săn `rear_ended` → 31 `bus_bypass` + 9 `crosswalk_block`.

**`bus_bypass`** (B-01, t = 12,6 s) ![bus_bypass](evidence/shots/bus_bypass.png)
C2 kẹt sau xe buýt đỗ ở trạm, chấp nhận khe 7,35 m trên đường ướt để lách sang làn xe ta, rồi gần như đứng lại khi còn chéo giữa hai làn đúng lúc xe buýt rời trạm. Xe ta (3,8 m/s) không kịp coi C2 là xe phía trước nên đâm vào (`front`).

**`crosswalk_block`** (C-02, t = 22,2 s) ![crosswalk_block](evidence/shots/crosswalk_block.png)
P1 đứng sau xe buýt và không ngó qua (`pedLook` = 0). Planner chỉ thấy P1 khi người đã bước ra, phanh muộn trên đường ướt, và dừng với đầu xe nằm trên vạch, chặn lối P1 (`kind: block`).

**Không tìm thấy:** `cut_in` và `rear_ended`. Cả săn lẫn MC đều không ra vụ nào. Ta chỉ nói được là chúng hiếm hơn ~10⁻³, không nói được là không có.

**Nếu chỉ một cuộc săn chung:** cuộc săn sẽ co về kiểu dễ nhất là `bus_bypass`, như chính các cuộc săn `cut_in` và `rear_ended` đã trôi về đó. `crosswalk_block` (~10⁻⁷) sẽ bị bỏ sót.

## 6. Sửa planner

Policy được chọn: **`cutinLat` = 2,6** (gốc 2,15), tức coi xe đang lấn sang là xe phía trước sớm hơn.

| `fix.json` | fail | fail_hi_seen | fail_hi | Quãng đường (m) | Phanh gấp |
|---|---|---|---|---|---|
| Bản sửa, TRAIN | 1,12·10⁻⁶ | 2,55·10⁻⁶ | 2,55·10⁻⁶ (`unseen`: `bus_bypass`) | 210,32 | 7,93 |
| Gốc, TRAIN | 3,29·10⁻⁴ | 6,00·10⁻⁴ | 6,00·10⁻⁴ | 210,26 | 8,98 |
| Bản sửa, **TEST** | **3,44·10⁻⁴** | **8,18·10⁻⁴** | **2,10·10⁻³** | 209,19 | 8,66 |
| Gốc, **TEST** | 9,76·10⁻⁴ | 1,87·10⁻³ | 3,15·10⁻³ | 209,24 | 8,99 |

- **TRAIN chỉ dùng để sàng.** Bản sửa có 0 vụ `bus_bypass` trên TRAIN chỉ vì mẫu TRAIN được rút ở nơi policy gốc thất bại. Trên TEST, `bus_bypass` xuất hiện lại (4 hits), nhưng thất bại vẫn giảm ~2,8 lần mà quãng đường gần như không đổi.
- **`fail_hi` của bản sửa (2,10·10⁻³) lớn hơn `fail` của bản gốc (9,76·10⁻⁴):** `fail_hi` là cận trên. Nó cộng ±1,96·se của IS (se lớn vì `top_share` cao) và cận Wilson 1,28·10⁻³ của các va chạm khác chưa thấy. Phải so cận với cận (2,10·10⁻³ < 3,15·10⁻³).

![fix ghost](evidence/shots/fix_ghost.png)
B-02, t = 12,8 s: xe đặc chạy policy gốc và đâm C2; bóng mờ là bản sửa trên cùng kịch bản, dừng lại phía sau.

**Núm "an toàn" phản tác dụng** (TRAIN):
- **`T` = 2,0:** `crosswalk_block` tăng 1,12·10⁻⁶ → **4,89·10⁻⁶**. Khoảng trống lớn hơn phía trước khiến người đi bộ và xe nhập làn cho rằng đủ an toàn để đi vào, nên xe ta phải dừng gấp, thân xe nằm trên vạch.
- **`cautionV` = 6,0:** `bus_bypass` tăng 3,28·10⁻⁴ → **6,56·10⁻⁴**. Xe ta bò chậm cạnh xe buýt lâu hơn, nên C2 có thêm thời gian và khe để chen vào trước mũi xe.

## 7. Từ mô phỏng ra đường thật

| Giả định | Dữ liệu thật thay thế | Nếu sai |
|---|---|---|
| `pedLook` = 0,80 | Video tại trạm xe buýt có vạch qua đường (camera giao lộ, inD / nuScenes): tỉ lệ người nhìn trước khi bước ra | Người thật ngó ít hơn → `crosswalk_block` bị **ước lượng thấp** |
| `cutGap` trung vị 10 m | Quỹ đạo nhập làn có nhãn (highD, exiD, NGSIM) ở tốc độ nội thành | Khe thật nhỏ hơn (giao thông Việt Nam) → `cut_in`, `bus_bypass` bị **ước lượng thấp** |
| `wet` = 0,20; μ 0,5 / 0,8 | Số giờ mưa từ trạm khí tượng; μ đo thực tế hoặc từ dữ liệu ABS của đội xe | Mưa nhiều hoặc đường trơn hơn → các kiểu cần phanh bị **ước lượng thấp** |

## 8. Giới hạn

1. **Một con phố, một trạm xe buýt, một vạch qua đường.** p là mỗi lần qua đoạn phố này, không phải mỗi km. Mô hình không có giao lộ đèn và không có xe máy.
2. **Mọi tác nhân là IDM có gap acceptance.** Không có ai vượt đèn hay mất tập trung, nên các kiểu thất bại ngoài mô hình này có xác suất bằng 0.
3. **Cảm biến hoàn hảo trừ phần bị xe buýt che.** Không nhiễu, không trễ. Các con số là cận dưới của rủi ro thật.

Thêm một tuần: chạy MC ~25 000 lần cho `bus_bypass` để kiểm độ chệch của IS; thêm nhiễu và trễ cảm biến; săn từ nhiều điểm khởi đầu khác nhau.

## 9. Rắc rối của bạn

- **Rắc rối:** hành khách (H0, `ped`, `side: N`) chạy băng ngang ngoài vạch, ngay quanh đầu xe buýt đang đỗ (x 100–118 m; vạch ở x = 80 m), để kịp xe buýt.
- **Căn cứ:** planner chỉ thận trọng quanh vạch qua đường (`cautionV`, `pedBuffer` cho P1), còn người băng ngoài vạch nằm ngoài các tình huống đó.
- **Giả định** (`hazard.json: why`):
  - `h0_v` 1,2–2,5 m/s: từ đi bình thường đến chạy chậm.
  - `h0_trig` 12–35 m: dưới 12 m đã ngang quãng phanh ≈ 9 m nên không ai tránh nổi; trên 35 m thì ai cũng kịp.
  - `exposure` 0,02/chuyến: ước lượng ~4 trạm có xe buýt đỗ mỗi chuyến × 0,5 % mỗi trạm. Cần thay bằng số đếm từ camera tại trạm.
- **Kết quả** (`mine.json`): `accepted` **99,7 %** (1 495/1 500). Cảnh gốc (`exact`): planner đâm H0 lúc 15,75 s.

| `mine.json` | Gốc | Bản sửa `cutinLat` 2,6 |
|---|---|---|
| Va chạm | 24 | 24 |
| `avoidable` / `unavoidable` / `unattributed` | **21** / 0 / 3 | **23** / 0 / 1 |
| `low_speed` | 2 | 1 |
| P(lỗi \| rắc rối), Wilson 95 % | **1,40·10⁻²** [9,21·10⁻³; 2,14·10⁻²] | 1,54·10⁻² [1,03·10⁻²; 2,30·10⁻²] |
| Mỗi chuyến (× 0,02) | **2,81·10⁻⁴** (cận trên 4,28·10⁻⁴) | 3,08·10⁻⁴ |

- **So với lách xe buýt:** `bus_bypass` theo MC là 4,0·10⁻³ mỗi lần qua phố, gấp ~14 lần rắc rối này; theo IS (~5,9·10⁻⁴) thì cùng bậc.
- **Bản sửa không cứu được:** 23 vụ so với 21. `cutinLat` không liên quan đến người đi bộ.
- **Ba loại va chạm:**
  - Chỉ `avoidable` là lỗi planner.
  - `unattributed` là va chạm thường xảy ra trước khi H0 bắt đầu băng.
  - `avoidable_foreknown` = `avoidable` (21 và 23), nên bị che tầm nhìn không phải nguyên nhân: người đến từ phía bên kia, xe buýt không che. Planner có giảm tốc nhưng vẫn đâm ở 1–5 m/s.
- **"Tránh được" là chuẩn lạc quan:** người phanh giả định biết ngay và phanh μ·g từ khung hình đầu tiên, không có thời gian phản xạ, nên `avoidable` **đếm nhiều hơn** số vụ planner thật tránh được. Ngược lại, chỉ thử phanh mà không thử đánh lái nên nó **đếm ít đi** những vụ có thể lách tránh.
- **Hình:** ![mine](evidence/shots/mine_hazard.png) GB-01, t = 16,0 s.
- **Bạn tự lái thì sao:** tôi không lái bằng bàn phím trong viewer. Thay vào đó tôi chạy cảnh gốc với `ego="script"`, cùng cơ chế nút "Bạn", bằng một chuỗi chân ga/phanh mô phỏng người lái. Đây là lần chạy thêm ngoài `evidence/`. H0 bắt đầu băng lúc 10,1 s và lọt vào tầm nhìn xe ta lúc 10,45 s.
  - Phản xạ 1,0 s, phanh 6 m/s²: dừng lúc 13,45 s ở x = 109,1 m, cách H0 khoảng 6 m, **không va chạm**.
  - Phản xạ 1,5 s (phanh 6 m/s²) hoặc phanh nhẹ 4 m/s² (phản xạ 1,0 s): dừng sát người, vẫn **không va chạm**.
  - Giữ nguyên 12 m/s, không phanh: xe ta qua trước khi người tới làn, cũng không va chạm.

  Planner đâm lúc 15,75 s vì nó giảm tốc **dở dang**: chậm đủ để đến đúng lúc người bước vào làn, nhưng không dừng hẳn. Người lái thì quyết định dứt khoát: hoặc phanh dừng hẳn ngay khi thấy người, hoặc đi qua trước. Lưu ý là tôi đã biết trước cảnh này, còn planner thì không.
- **Cách sửa:** số dưới đây từ một lần chạy thêm ngoài `evidence/` (600 lần rút), là số vụ `avoidable`. Gốc 11; `pedBuffer` 3,5 hoặc 5,0 vẫn 11; `T` 2,0 → 8; `cautionRange` 120 m → 7; `v0` 10 m/s → 6. Không núm nào sửa tận gốc, và giảm tốc thì mất quãng đường ở mọi nơi. Cần **logic mới**: theo dõi mọi người đi bộ trên lòng đường (không chỉ ở vạch), dự đoán quỹ đạo, và dừng hẳn nhường nếu quỹ đạo cắt làn xe ta; kèm vùng giảm tốc quanh trạm khi có xe buýt đỗ.
