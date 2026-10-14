MSSV: 202602760

# Lab 11 · Scenario Forge — báo cáo

Thị trấn C (`evidence/meta.json: _run.town`), seed MC/search/test lần lượt `405050949`, `4152366860`, `4241070839` (`meta.json: seeds`). Pipeline A–G dùng ngân sách đầy đủ; lịch sử bước cuối ở `meta.json: steps` đều `quick:false`.

## 1. Phát lại log có trung thực không? (bước A)

Trong 400 log, không log nào bị bỏ vì người lái gây va chạm (`replay.json: dropped_human_crash_logs=0`). Planner trên log phát lại có 129 va chạm: 114 `rear`, 1 `front`, 14 `side`; trong giao thông phản ứng có 0 (`replay.json: replay`, `reactive`, `crash_in_both=0`). Phần lớn là tông đuôi giả: log ghi nguyên quỹ đạo xe sau, không phản ứng khi planner phanh sớm hơn người lái thật; giao thông phản ứng sẽ phanh theo xe ta. Ví dụ `replay_only[0]`, scenario 1: với `cutT0=8.72 s`, replay kết thúc bằng `ego|TG` rear lúc 22.4 s; khi traffic phản ứng, TG phanh theo xe ta và không va chạm (`replay.json: replay_only[0]`, đối chiếu rollout cùng params).

## 2. Kịch bản bất khả thi (bước B)

`is_feasible` loại (1) chuyển làn 3.5 m cần gia tốc ngang cực đại `(π/cutDur)²·3.5/2` lớn hơn `μg`; (2) phanh xe vừa nhập làn `cutBrake > μg`; (3) người đi bộ chạy nhanh hơn 2.5 m/s; (4) xe chuyển làn có `cutV` ngoài `(0,22]` m/s (`lab11/feasible.py`). Giới hạn ma sát là vòng tròn ma sát lốp/đường, còn hai giới hạn tốc độ loại hành vi nằm ngoài phạm vi người/xe trong mô hình. Hunt `crosswalk_block` ghi nhiều infeasible nhất: 5, cùng rear-ended cũng 5; bus bypass 1 và cut-in 2 (`hunt.json: infeasible`). CEM tối ưu điểm nguy hiểm, nên các mẫu vi phạm giới hạn nhưng có score cao vẫn gây áp lực kéo proposal về vùng bất khả thi; pipeline loại chúng trước khi mô phỏng/tính lỗi. Nếu gán chúng là crash, tử số lỗi sẽ cộng thêm các sự kiện không thể xảy ra trên đường; xác suất bị thổi phồng và không còn đo lỗi trong miền khả thi.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: `k=4`, `n=3000`, `p̂=0.0013333`, Wilson 95% `[0.0005186, 0.0034235]`. Xấp xỉ số lần chạy cho sai số tương đối 20% ở mức 95% là `n≈1.96²(1-p)/(0.2²p)≈96/p≈72,000`. Không thấy cut-in, crosswalk_block, rear_ended hay other; không kết luận xác suất bằng 0: cận trên Wilson cho từng mode không thấy là `0.0012789` (`mc.json: per_mode.*.hi`).

## 4. Importance sampling, ESS và độ suy biến (bước C3)

`estimate.json` ghi `p / hits / ess / top_share`:

| Mode | Seed 0 | Seed 1 | Seed 2 |
|---|---|---|---|
| bus_bypass | 1.1477e-2 / 19 / 144.4 / 0.715 | 6.8905e-4 / 29 / 129.6 / 0.347 | 3.5276e-3 / 28 / 142.7 / 0.739 |
| cut_in | 0 / 0 / 133.2 / 0 | 0 / 0 / 122.1 / 0 | 0 / 0 / 137.3 / 0 |
| crosswalk_block | 3.9091e-6 / 88 / 119.5 / 0.468 | 6.0336e-6 / 101 / 104.3 / 0.328 | 1.2122e-5 / 97 / 120.2 / 0.451 |
| rear_ended | 0 / 0 / 104.7 / 0 | 0 / 0 / 114.6 / 0 | 0 / 0 / 122.0 / 0 |

Các seed không đồng ý tốt về bus bypass (hơn một bậc độ lớn), dù cùng tìm ra crash; crosswalk cùng bậc độ lớn. Tổng lần lượt `0.01148`, `0.000695`, `0.00354`, trung vị `0.00354`; cả ba đều lớn hơn MC `0.00133` nhưng CI rộng và các mode khác nhau không được hiểu là xác suất 0 (`estimate.json: seeds`, `mc.json: p`). `upper_zero_hits=0.025` cho cut-in và rear-ended mỗi seed; đây là cận zero-hit bảo thủ từ defensive mixture, không phải điểm ước lượng. Khi `top_share>0.5` (bus bypass seed 0/2), một mẫu chiếm phần lớn tổng trọng số sự kiện nên ước lượng nhạy với một kịch bản đơn lẻ. Ngân sách IS là `16,800` mẫu (4 mode × (2,400 hunt + 1,800 estimate), `estimate.json: budget`) so với khoảng 72,000 MC cho cùng sai số tương đối lý tưởng; song IS thực tế suy biến do top_share. `other` lấy từ MC: 0/3000, ước lượng 0 nhưng cận Wilson `0.0012789` (`estimate.json: other`). Không săn `other` vì đây là phần tổng hợp nhiều kiểu không có một score/biến nguy hiểm đồng nhất để CEM hướng tới.

## 5. Các kiểu thất bại (bước C2 + 3D)

Hunt tìm được bus_bypass 153 và crosswalk_block 52 lần; không tìm được đúng cut_in hay rear_ended: các lần crash trong hai hunt đó đều bị gán sang bus_bypass/crosswalk_block (`hunt.json: found`). Việc không thấy không chứng minh các mode hiếm hoặc không tồn tại; CEM của mode kia mode collapse vào crash dễ hơn. Nếu gom một hunt chung, mode nguy hiểm dễ tìm sẽ chiếm elite và các kiểu còn lại mất khỏi proposal, vì thế cần hunt riêng theo score từng mode.

Ảnh đúng frame va chạm: [q5_bus_bypass.png](evidence/shots/q5_bus_bypass.png) và [q5_crosswalk_block.png](evidence/shots/q5_crosswalk_block.png). Ở bus_bypass, xe C2 lách qua xe buýt dừng rồi đi vào hành lang của ego; ego tiếp tục tiến và va vào C2. Ở crosswalk_block, người đi bộ bị xe buýt che khuất; ego phanh muộn, dừng đè lên vạch và chặn P1 tại 18.9 s (nhãn sự kiện trong ảnh). Săn riêng giữ elite tập trung vào tín hiệu nguy hiểm của từng mode.

## 6. Sửa planner (bước E + 3D)

`fix.json: chosen.policy` là `cutinLat=2.6`, được chọn trên TRAIN; TRAIN `fail=1.608e-6`, `fail_hi=3.312e-6`, progress `210.751 m`, harsh `8.070` frame/trial. TEST policy sửa: `fail=6.508e-5`, `fail_hi_seen=1.767e-4`, `fail_hi=1.456e-3`, progress `210.473 m`, harsh `8.793`; baseline TEST: `fail=9.288e-4`, `fail_hi_seen=1.502e-3`, `fail_hi=2.781e-3`, progress `210.611 m`, harsh `9.004` (`fix.json: chosen_test`, `baseline_test`). Policy được chọn giảm fail điểm khoảng 93% và progress gần như không đổi. `fail_hi` chosen vẫn cao hơn `fail` baseline vì cận TEST còn cộng bất định/cận zero-hit cho các va chạm ngoài hai mode được hunt; baseline `fail` chỉ là điểm ước lượng. Một núm nghe an toàn hơn nhưng làm tăng lỗi trong mẫu TRAIN là tăng headway `T` từ 1.6 lên 2.0 s: `fail` tăng từ 1.485e-3 lên 2.918e-3, chủ yếu bus_bypass (1.272e-3 lên 2.916e-3), trong khi progress giảm từ 210.75 xuống 200.18 m (`fix.json: baseline_train`, `table` policy `T=2.0`). Gap theo thời gian lớn hơn làm đổi thời điểm/ vị trí tương đối của ego khi C2 nhập và lách xe buýt trong thế giới phản ứng, nên tương tác khác có thể tạo thêm lỗi; kết quả này là ước lượng hữu hạn TRAIN, không khẳng định T luôn làm rủi ro tăng. Ảnh baseline va chạm cùng bóng mờ policy đã chọn: [q6_policy_ghost.png](evidence/shots/q6_policy_ghost.png).

## 7. Từ mô phỏng ra đường thật

1. `cutGap` là log-normal với median 10 m ở town C. Có thể lấy vehicle trajectory 10 Hz và vị trí làn từ video của bộ [FHWA NGSIM Lankershim Boulevard](https://www.fhwa.dot.gov/publications/research/operations/07029/index.cfm) để định nghĩa gap tại thời điểm nhập làn thành công/thất bại; vì bộ này ở Los Angeles năm 2005, nó chỉ là nguồn khởi đầu cho phương pháp, không đại diện xe máy và giao thông Việt Nam. Cần fit lại bằng camera/radar tại phố mục tiêu. Nếu tài xế ở phố mục tiêu nhận khe nhỏ hơn median giả định, mô hình bỏ sót cut-in sát và đánh giá thấp rủi ro; nếu khe lớn hơn, có thể đánh giá cao.
2. `pedLook` có xác suất nhìn qua xe buýt 0.8 ở town C. [SHRP2 Naturalistic Driving Study (NDS)](https://www.fhwa.dot.gov/goshrp2/Solutions/Safety/NDS/Naturalistic_Driving_Study) có video trước đường, video mặt/tay người lái và GPS/radar, hữu ích để hiệu chuẩn phản ứng của người lái khi tiếp cận che khuất; nhưng không đo trực tiếp ánh mắt người đi bộ. Vì vậy phải bổ sung camera cố định đồng bộ tại các điểm dừng xe buýt và gán nhãn liệu người đi bộ nhìn/nhô ra trước khi băng qua; dùng nghiên cứu Johari Bazar trong `hazard.json: why` làm tham khảo hành vi và lưu lượng, không thay cho tần suất town C. Nếu tỷ lệ nhìn qua xe buýt thực thấp hơn 0.8, bước ra bất ngờ thường xuyên hơn và mô hình đánh giá thấp crosswalk_block; nếu cao hơn thì ngược lại.
3. `wet` có xác suất 0.2, ma sát được rút gọn thành μ=0.8 khô và 0.5 ướt. Thay bằng dữ liệu ma sát/skid resistance đo theo đoạn và thời gian cùng nhật ký mưa của sở giao thông địa phương; phương pháp đo liên tục và dữ liệu mặt đường được mô tả trong [FHWA Pavement Friction Management](https://highways.fhwa.dot.gov/safety/proven-safety-countermeasures/pavement-friction-management) và có thể đối chiếu [FHWA CPFM dataset](https://infomaterials.fhwa.dot.gov/data). Nếu thời lượng/độ trơn thực tế lớn hơn giả định, quãng phanh dài hơn và mô hình đánh giá thấp va chạm; nếu ít hơn thì có thể đánh giá cao.

Các phân phối trong `space.py` là giả định giảng dạy, chưa được fit từ các nguồn trên. Dữ liệu Mỹ chỉ giúp chọn phương pháp/biến đo; ước lượng cuối phải dùng quan sát tại tuyến triển khai, đồng bộ trajectory giao thông, hành vi người đi bộ, thời tiết và ma sát mặt đường.

## 8. Giới hạn

Thí nghiệm cho thấy planner sửa giảm đáng kể xác suất lỗi ước lượng trên TEST mà gần như giữ nguyên quãng đường; đây là tín hiệu tích cực trong phạm vi mô phỏng, chưa phải bảo đảm ngoài đường. Ba giới hạn cụ thể của thế giới này là: (1) kết quả chỉ áp dụng cho một tuyến và các phân phối giả định của town C, nên chưa đại diện hình học, lưu lượng và cách đi xe ở các tuyến khác; (2) xe khác dùng IDM cùng vài luật gap acceptance, chưa bao quát đầy đủ phản ứng xã hội và độ đa dạng hành vi; (3) cảm biến gần như hoàn hảo ngoài vùng xe buýt che khuất, chưa có nhiễu, trễ hay lỗi phân loại chung. Nếu có thêm một tuần, tôi sẽ thu video/trajectory đồng bộ xe–người đi bộ–xe nhập làn tại các nút giao có xe buýt che khuất, hiệu chuẩn phân phối hành vi và kiểm định dự báo trên một tuyến giữ lại.

## 9. Rắc rối của bạn (bước G + chương 10)

Rắc rối là người chạy băng ngang giữa phố ngoài vạch: hazard `ped` từ lề nam, vận tốc trung tâm 3 m/s, tại x=120 m, xuất hiện khi xe cách 20 m (`hazard.json`). Cảnh trung tâm khả thi và `mine.json: exact.planner` xác nhận va chạm `ego|H0` tại 16.2 s. Nghiên cứu video ở Johari Bazar ghi 510 lượt băng ngang/giờ khi lưu lượng 600–2400 PCU/giờ; tỷ số bảo thủ `510/2400=0.2125`, làm tròn xuống exposure proxy 0.2 mỗi chuyến. Đây không phải tần suất đã đo cho town C. Tốc độ 2.8–3.2 m/s được chọn cho người băng nhanh; tọa độ và trigger là cửa sổ quét quanh cảnh mô phỏng, cần thay bằng đo tại tuyến triển khai (`hazard.json: why`, `space`).

| Policy | accepted | avoidable | unavoidable | unattributed | low_speed | P(failure|hazard), Wilson 95% | mỗi chuyến (exposure × P) |
|---|---:|---:|---:|---:|---:|---:|---:|
| baseline | 99.4% | 229 | 0 | 2 | 4 | 0.1536 [0.1362, 0.1728] | 0.03072 [upper 0.03456] |
| chosen (`cutinLat=2.6`) | 99.4% | 230 | 0 | 1 | 8 | 0.1543 [0.1368, 0.1735] | 0.03085 [upper 0.03470] |

Số trên lấy từ `mine.json: accepted`, `baseline`, `chosen`; 1491/1500 mẫu khả thi. Trong hazard này, policy sửa giữ tỷ lệ chấp nhận 99.4% và xác suất va chạm có điều kiện gần như ngang baseline; kết quả xác nhận `cutinLat=2.6` không làm xấu đáng kể tình huống người băng ngang, nhưng cũng cho thấy núm này không nhắm tới hazard ấy. Rủi ro proxy mỗi chuyến là 3.07%, so với baseline town-wide bus-bypass MC 4/3000 = 0.133% (`mc.json: per_mode.bus_bypass.p`); hai con số có điều kiện và không gian khác nhau, nên không phải so sánh trực tiếp hiệu quả policy. Oracle “avoidable” lạc quan vì biết trước hazard và phanh tối đa ngay khi thấy, nên có thể đếm nhiều hơn planner thật tránh được; nhưng chỉ thử phanh, không thử đánh lái, nên bỏ sót cách tránh bằng chuyển hướng.

Khi tự lái thử scene trong viewer, tôi thấy xe phản ứng tốt và xử lý tình huống ổn. Trải nghiệm này là quan sát định tính của một lượt tương tác, không thay thế kết quả 1500 mẫu hay được quy đổi thành tỷ lệ an toàn. Một người lái có thể cân nhắc cả khoảng trống làn bên và tầm nhìn để đánh lái, trong khi oracle ở đây chỉ thử phanh. Để xử lý có hệ thống hơn, planner có thể dự báo vùng che khuất, giảm tốc sớm theo stopping distance, rồi chọn phanh hoặc đánh lái khi làn bên đã được xác nhận an toàn và trong giới hạn ma sát. Ảnh hazard lúc va chạm với người băng ngang (GB-01): [q9_midblock_ped.png](evidence/shots/q9_midblock_ped.png).
