# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Đoàn Anh Quân |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602803 |
| GitHub | `<điền: tên tài khoản>` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `<điền>` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `<điền>` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | | | |
| IDM | | | (baseline) |
| EgoMLP | | | |
| PercMLP | | | |
| PercMLP+ (sau khi sửa) | | | |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

(trả lời ở đây)

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

<!-- NHÁP: đọc lại và viết lại bằng lời của bạn -->
Chạy lại cùng mã cho 0/600 đoạn khác và 0 đoạn khác bit, nên công cụ phát lại tái lập được. Bản refactor float32 (cùng phép toán, đổi thứ tự) khác bit ở **600/600** đoạn nhưng vượt dung sai (atol = rtol = 1e-3 trên lệnh gia tốc) ở **0** đoạn, sai khác lớn nhất 4,0e-4 m/s². Lý do là phép cộng số thực không có tính kết hợp: (0,1+0,2)+0,3 = 0,6000000000000001 còn 0,1+(0,2+0,3) = 0,6. Vì vậy so từng bit là phép thử sai: nó báo "khác" cho một thay đổi vô hại. Dung sai hợp lý phải lớn hơn nhiễu số học (~1e-4) nhưng nhỏ hơn mọi thay đổi hành vi có ý nghĩa.

Thêm AEB làm lệch **6/600** đoạn (55, 297, 448, 468, 523, 577), sai khác lớn nhất 1,15 m/s². Mở đoạn 55: chỉ một chuỗi khung liền nhau ở cuối đoạn bị lệch, đúng lúc khoảng cách còn vài mét và TTC tụt dưới ngưỡng `aeb_ttc` = 1,5 s. Ở đó AEB thay lệnh của IDM bằng phanh tối đa. Ngoài vùng TTC thấp, AEB không can thiệp, nên các đoạn và khung còn lại giữ nguyên.

Phát nhanh nhất cho 0 khung trễ và 0 khung rớt trên 59 303 khung, vì nó luôn chờ bộ điều khiển tính xong: không có hạn chót. Phát theo thời gian thật có **580 khung trễ và 576 khung rớt**, làm **377/600** đoạn đổi lệnh, vì khung nào tính quá 100 ms thì lệnh cũ bị giữ lại. Phát nhanh nhất vì thế che mất lỗi trễ hạn, loại lỗi chỉ xuất hiện trên xe thật.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

(trả lời ở đây)

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

(trả lời ở đây)

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

(trả lời ở đây)

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

(trả lời ở đây)

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

(trả lời ở đây)

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(trả lời ở đây)
