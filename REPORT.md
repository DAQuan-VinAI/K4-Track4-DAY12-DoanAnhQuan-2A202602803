# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Đoàn Anh Quân |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602803 |
| GitHub | DAQuan-VinAI |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 8030 / 9164 (5 seed: 9164–9168) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (mặc định) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,248 | 20,2 | BLOCK (R2, R3, R4, R6) |
| IDM | 1,527 | 0,07 (2/2883) | (baseline) |
| EgoMLP | 0,763 ± 0,003 | 15,7 – 43,2 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0,619 ± 0,003 | 0,4 – 1,9 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP+ (sau khi sửa) | 0,807 ± 0,016 | 0,2 – 0,3 | RELEASE (qua cả 7 quy tắc) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

3 vấn đề. 
1) **Cột `Acc_LV` không dùng được:** tương quan của nó với đạo hàm của `Speed_LV` chỉ **0,045**, trong khi cột tương ứng của xe theo sau đạt **0,929**. Nếu khai thác ca phanh bằng `Acc_LV` thì sẽ bắt toàn nhiễu, nên tôi dùng gia tốc tính lại từ tốc độ xe dẫn đã làm mượt (`a_lead`). 
2) **Quỹ đạo trùng hệt nhau:** có **383** quỹ đạo trùng (9993 → 9610), nằm ở các id cách xa nhau. Chia ngẫu nhiên thì một bản có thể rơi vào train và bản kia vào test, làm điểm test bị thổi phồng vì mô hình đã "thấy" đúng quỹ đạo đó. 
3) **Phải chia theo quỹ đạo, không theo khung:** ở 10 Hz, hai khung liền nhau gần như giống hệt. Chia theo khung thì khung test luôn có "hàng xóm" trong train, nên đó là rò rỉ chứ không phải tổng quát hoá. Cột `Spatial_Gap` thì đáng tin: nó đúng bằng hiệu vị trí trừ một độ lệch hằng 4,5 m (chiều dài xe).

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã cho 0/600 đoạn khác và 0 đoạn khác bit, nên công cụ phát lại tái lập được. Bản refactor float32 (cùng phép toán, đổi thứ tự) khác bit ở **600/600** đoạn nhưng vượt dung sai (atol = rtol = 1e-3 trên lệnh gia tốc) ở **0** đoạn, sai khác lớn nhất 4,0e-4 m/s². Lý do là phép cộng số thực không có tính kết hợp: (0,1+0,2)+0,3 = 0,6000000000000001 còn 0,1+(0,2+0,3) = 0,6. Vì vậy so từng bit là phép thử sai: nó báo "khác" cho một thay đổi vô hại. Dung sai hợp lý phải lớn hơn nhiễu số học (~1e-4) nhưng nhỏ hơn mọi thay đổi hành vi có ý nghĩa.

Thêm AEB làm lệch **6/600** đoạn (55, 297, 448, 468, 523, 577), sai khác lớn nhất 1,15 m/s². Mở đoạn 55: chỉ một chuỗi khung liền nhau ở cuối đoạn bị lệch, đúng lúc khoảng cách còn vài mét và TTC tụt dưới ngưỡng `aeb_ttc` = 1,5 s. Ở đó AEB thay lệnh của IDM bằng phanh tối đa. Ngoài vùng TTC thấp, AEB không can thiệp, nên các đoạn và khung còn lại giữ nguyên.

Phát nhanh nhất cho 0 khung trễ và 0 khung rớt trên 59 303 khung, vì nó luôn chờ bộ điều khiển tính xong: không có hạn chót. Phát theo thời gian thật có **580 khung trễ và 576 khung rớt**, làm **377/600** đoạn đổi lệnh, vì khung nào tính quá 100 ms thì lệnh cũ bị giữ lại. Phát nhanh nhất vì thế che mất lỗi trễ hạn, loại lỗi chỉ xuất hiện trên xe thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cắt vào có cruise **10 m/s**, khoảng cách **2,8–59,3 m** (p2–p98 của log), tốc độ tiến gần 0–10 m/s, tổng **60 × 45 = 2700** ô. Biên bất khả thi là `gap < dv·t_react + dv²/(2·b_max)`, với b_max = 9 m/s². Biên này chiếm **1,0 %** lưới khi độ trễ 0 s và **5,1 %** khi độ trễ 0,6 s. Kiểm chứng bằng FullBrake cho **0** ô lệch xa biên ở cả hai độ trễ. Vùng đâm của IDM tăng đúng bằng phần đó, **1,0 % → 5,1 %**, và phần đâm do lỗi của chính IDM (attributable) là **0 %** ở cả hai mức. Nghĩa là mọi va chạm của IDM đều thuộc loại "ai cũng đâm", không phải "bộ điều khiển đâm". Ngược lại, const-vel có 68,2 % lưới là lỗi của nó (độ trễ 0,6 s). Log **không phủ vùng khó**: người lái trong log phanh mạnh nhất **3,6 m/s²** (p99,9), trong khi xe phanh được 9 m/s², và **5,2 %** lưới đòi phanh mạnh hơn mức đó. 614 ca khai thác được (6,4 % số quỹ đạo, xe dẫn phanh mạnh nhất có trung vị 2,5 m/s²) tập trung ở góc dễ của lưới, tức tiến gần chậm và khoảng cách vừa. Góc khoảng cách nhỏ, tiến gần nhanh gần như trống, nên mô hình học từ log chưa từng thấy vùng này.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở (m) | Hạng | L2 lát cắt phanh (m) | Va chạm vòng kín (TB qua seed) | Hạng |
|---|---|---|---|---|---|
| const-vel | 1,248 | 3 | 1,886 | 20,2 % | 3 |
| IDM | 1,527 | 4 | 1,562 | 0,07 % | 1 |
| EgoMLP | 0,763 | 2 | 1,073 | 24,3 % | 4 |
| PercMLP | 0,619 | 1 | 0,859 | 1,3 % | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). **IDM có L2 tệ nhất mà đâm ít nhất**, vì L2 đo mức *giống người lái* chứ không đo mức *an toàn*. IDM là một luật có thời khoảng cách và mức giảm tốc riêng, nên quỹ đạo của nó lệch khỏi người. Nhưng lệnh của nó là hàm của khoảng cách và tốc độ tiến gần: hễ khoảng cách co lại là nó phanh, nên trong vòng kín nó luôn tự kéo về trạng thái an toàn. **EgoMLP có L2 tốt hơn const-vel mà vẫn đâm** vì nó chỉ nhìn tốc độ của chính mình. Phần lớn cửa sổ là chạy đều, nên đoán "giữ quán tính như người" cho L2 thấp (lát cắt phanh chỉ chiếm 4,3 % cửa sổ). Nhưng nó không có thông tin về khoảng cách và xe dẫn, nên không thể biết khi nào phải phanh: trên các quỹ đạo có xe dẫn phanh, seed xấu nhất đâm **66,1 %**. Trong vòng kín, sai số không được log "kéo về" nữa mà tích luỹ dần thành va chạm.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Qua 5 seed huấn luyện, EgoMLP đâm **15,7 % / 16,6 % / 20,9 % / 25,1 % / 43,2 %**, tức dải rộng gần 28 điểm, dù L2 vòng hở gần như không đổi (0,763 ± 0,003). CI95 Wilson của seed tốt nhất (9168) là **[14,4; 17,1] %**, của seed xấu nhất (9166) là **[41,4; 45,0] %**: hai khoảng **không chồng nhau**. Wilson chỉ đo sai số do *số quỹ đạo test có hạn* (n = 2883) trong **một** mô hình đã huấn luyện. Nó không đo sai số do *quá trình huấn luyện*, và với mô hình học thì sai số huấn luyện lớn hơn nhiều. Chạy một seed thì có thể báo "16 %" hoặc "43 %" tuỳ may rủi, kèm một CI hẹp trông rất chắc chắn. PercMLP cho thấy rủi ro này còn cụ thể hơn: seed tốt nhất đâm 0,4 %, CI [0,2; 0,7] %, cận trên dưới 0,07 % + 1 điểm nên **qua** R2. Seed xấu nhất đâm 1,9 %, cận trên 2,4 %, nên **rớt** R2. Báo một seed thì có thể phát hành nhầm, nên cổng lấy seed xấu nhất và R5 đòi dải ≤ 1 điểm.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Tỷ lệ lưới cắt vào có va chạm bất kỳ (trước hoặc sau), độ trễ 0,6 s:

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm phía sau (phát lại / phản ứng) |
|---|---|---|---|
| const-vel | 73,4 % | 73,4 % | 0,0 % / 0,0 % |
| IDM | 70,1 % | 5,1 % | 64,9 % / 0,0 % |
| EgoMLP | 78,4 % | 78,4 % | 0,0 % / 0,0 % |
| PercMLP | 73,4 % | 21,5 % | 53,8 % / 0,0 % |

Người lái trong log giữ thời khoảng cách trung vị **1,86 s**. Tôi hiệu chỉnh xe sau là một IDM với **T = 1,4 s**: khoảng cách W1 giữa phân bố thời khoảng cách của nó và của người là **0,35**, thấp nhất trong các giá trị T đã thử. Một xe sau không bao giờ phản ứng có W1 **0,88**, tức kém chân thực hơn rõ rệt. Khi xe sau chỉ phát lại, nó chạy theo tốc độ trong log và không biết xe thử vừa phanh. Vì vậy chính hành động đúng (IDM phanh để tránh xe cắt vào) lại bị tính là **64,9 %** va chạm phía sau. Đó là va chạm giả do thế giới không phản ứng, không phải lỗi của IDM. Với xe sau phản ứng, va chạm phía sau về **0 %**. Kết luận đổi hẳn: ở thế giới phát lại, IDM (70,1 %) trông gần như tệ bằng const-vel (73,4 %), vì const-vel không bao giờ phanh nên không bao giờ bị tông đuôi. Ở thế giới phản ứng, IDM chỉ còn 5,1 %, đúng bằng phần bất khả thi, còn const-vel vẫn 73,4 %. Kiểm thử bằng phát lại có thể *thưởng* cho bộ điều khiển không phanh.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi R2–R6, với hai nguyên nhân gốc. **(a) Kịch bản mà log không chứa:**
- R4: diện tích lỗi trên lưới cắt vào **16,4 %** so với 0,0 % + 2 điểm;
- R6: **21,5 %** so với 5,1 % + 2 điểm.

Như câu 3 cho thấy, log không có ca cắt vào gần và tiến gần nhanh, nên mô hình không biết phải phanh mạnh ở đó. **(b) Sai số tích luỹ khi tự lái** (covariate shift):
- R2: seed xấu nhất đâm 1,9 %, cận trên **2,4 %** so với 0,07 % + 1 điểm;
- R3: 6,0 %, cận trên 10,4 %;
- R5: dải **0,4..1,9 %** qua 5 seed.

Mô hình chỉ học ở các trạng thái người lái đã đi qua. Khi tự lái, nó trôi dần sang những trạng thái chưa từng thấy và không biết cách kéo về. Cách sửa gồm:
- thêm **197 060** cửa sổ từ **20 000** ca cắt vào mô phỏng do IDM lái, chữa (a);
- thêm **2 vòng DAgger**: cho mô hình tự lái trên log train, gắn nhãn IDM ngay tại các trạng thái nó tự đi vào, chữa (b).

Sau khi sửa, PercMLP+ **qua cả 7 quy tắc**: R2 cận trên 0,5 %, R3 0,5 %, R4 **1,6 %**, R5 0,2..0,3 %, R6 6,7 %.

Nhưng PercMLP+ **không tốt hơn IDM**, nó chỉ *không kém hơn trong biên cho phép*:
- trên log nó đâm 0,2–0,3 % so với 0,07 % của IDM;
- lỗi trên lưới 1,6 % so với 0,0 %; lưới phản ứng 6,7 % so với 5,1 %;
- ở cruise **20 m/s**, tốc độ chưa dùng khi đo các quy tắc, nó vẫn đâm ở **0,6–2,0 %** ô tránh được, còn IDM là **0 %**.

Thêm vào đó, nó học từ chính IDM (nhãn mô phỏng và DAgger đều do IDM gán), nên khó có thể vượt thầy. L2 vòng hở của nó còn **tăng** từ 0,619 lên 0,807 m, vì nó bớt giống người và giống IDM hơn, trong khi vòng kín lại tốt hơn hẳn. Đây thêm một lần cho thấy điểm vòng hở không nói gì về an toàn.

Đề xuất: **chưa thay IDM**. Trước khi quyết định, nên:
- chạy PercMLP+ ở chế độ shadow (chạy song song, chỉ ghi lại, không điều khiển xe) để so lệnh với IDM trên dữ liệu thật;
- mở rộng lưới sang nhiều tốc độ cruise và nhiều họ kịch bản khác (xe dẫn phanh gấp, nhiễu cảm biến lớn hơn);
- trả lời câu hỏi *nó mang lại lợi ích gì so với IDM* (êm hơn, giống người hơn). Nếu không có lợi ích rõ ràng thì không có lý do nhận thêm rủi ro.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Thế giới mô phỏng là giả định của chúng ta, và giả định này vòng tròn.** IDM vừa là baseline, vừa là "chuyên gia" gán nhãn cho PercMLP+, vừa là mô hình của xe sau phản ứng. Cổng do vậy chỉ chứng minh PercMLP+ *giống IDM trong thế giới của IDM*. Mọi lỗi mô hình hoá chung (động lực học xe, độ trễ 0,6 s, b_max = 9 m/s² trên mặt đường lý tưởng) sẽ không bị phát hiện.
2. **Một chiều dọc, không cảm biến.** Thực tế có rẽ, đổi làn, người đi bộ và nhận dạng sai. Bài lab chỉ thử nhiễu Gauss (khoảng cách ±1 m, tốc độ xe dẫn ±0,5 m/s): PercMLP+ vẫn 0,2–0,3 %. Nhưng nhiễu thật có đuôi dày (mất track, false positive, nhận nhầm làn), và đó mới là nơi mô hình học dễ thất bại nhất.
3. **Phân bố dữ liệu hẹp.** Log Argoverse 2 là phố đô thị, tốc độ xe dẫn p99 chỉ 15,6 m/s, người lái phanh tối đa 3,6 m/s². Ngay ở 20 m/s, PercMLP+ đã đâm 0,6–2,0 % lưới so với 0 % của IDM, tức ngoài vùng dữ liệu kết luận không còn đúng.
4. **Thống kê không đủ cho sự kiện hiếm.** Có 2883 quỹ đạo test, nên CI95 của IDM là [0,02; 0,25] %. Bài lab chỉ phân biệt được khác biệt cỡ 0,1–1 điểm, còn mục tiêu an toàn thật là va chạm trên hàng triệu km.
5. **Xe dẫn trong log không phản ứng với xe thử.** Bài lab chỉ thay xe *sau* bằng tác nhân phản ứng. Câu 6 cho thấy riêng điều này đã đổi hẳn kết luận, nên một xe dẫn hay xe cắt vào có phản ứng cũng có thể làm vậy.

Việc cần làm tiếp:
- thêm dữ liệu từ đường cao tốc và các ca phanh gấp thật;
- dùng mô hình cảm biến có lỗi thực tế;
- dùng tác nhân phản ứng học từ dữ liệu cho mọi xe xung quanh;
- kiểm thử trên đường đóng, rồi chạy shadow mode trên đội xe trước khi cho mô hình điều khiển.
