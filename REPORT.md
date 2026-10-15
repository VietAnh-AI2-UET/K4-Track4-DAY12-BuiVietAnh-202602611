# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Bùi Việt Anh` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `202602611` |
| GitHub | `VietAnh-AI2-UET` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `4450 / 5796, 5797, 5798, 5799, 5800` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.255 | 20.985 | BLOCK |
| IDM | 1.542 | 0.104 | (baseline) |
| EgoMLP | 0.755–0.766 | 15.470–30.108 | BLOCK |
| PercMLP | 0.612–0.622 | 0.382–2.844 | BLOCK |
| PercMLP+ (sau khi sửa) | 0.820 ± 0.017 (mean ± std; evidence không lưu min–max) | 0.243–0.277 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

- Dữ liệu có quỹ đạo trùng hoàn toàn và cột `Acc_LV` không khớp với gia tốc suy ra từ vận tốc.
- Tương quan giữa `Acc_LV` và đạo hàm vận tốc chỉ khoảng 0.0454, nên bài dùng `a_lead` tính bằng đạo hàm của `Speed_LV` đã làm mượt.
- Đã loại 383 quỹ đạo trùng, còn 9.610 quỹ đạo từ 9.993 quỹ đạo ban đầu.
- Loại trùng trước rồi chia theo quỹ đạo giúp tránh cùng một hành trình, hoặc các cửa sổ chồng lấn của nó, xuất hiện ở cả train và test, làm kết quả tốt giả tạo.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

- Refactor float32 khác bit ở 600/600 đoạn nhưng không đoạn nào vượt dung sai; sai lệch lớn nhất chỉ khoảng 9.32e-5 m/s².
- AEB làm 9 đoạn khác chính xác, trong đó 8 đoạn vượt dung sai: 46, 64, 164, 226, 252, 300, 512 và 534.
- Kiểm tra đoạn 46, bước 98: TTC khoảng 1.474 s, dưới ngưỡng 1.5 s, nên AEB đổi lệnh từ khoảng -5.436 xuống -6 m/s². Lệnh chỉ thay đổi khi xe đang tiến gần, TTC dưới ngưỡng và lệnh cũ chưa phanh mạnh bằng -6 m/s².
- Replay thời gian thật có 584 khung trễ, 575 khung bị bỏ trên 59.849 khung, làm 374/600 đoạn vượt dung sai vì phải giữ lệnh trước. Chạy nhanh nhất không xét deadline nên báo 0 trễ, 0 bỏ, che khuất ảnh hưởng của xử lý chậm.

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
