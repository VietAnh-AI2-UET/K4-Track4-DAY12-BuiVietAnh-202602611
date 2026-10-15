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

- Theo `scenarios.json`, lưới có cruise 12 m/s, gap 2.760–59.312 m, chênh lệch tốc độ 0–12 m/s và 2.700 ô.
- Khi trễ 0 s, 2.222% ô không thể tránh va chạm dù phanh tối đa; khi trễ 0.6 s, tỷ lệ này là 7.444%.
- Vùng đâm của IDM cũng tăng từ 2.222% lên 7.444%: tăng 5.222 điểm phần trăm, khoảng 3.35 lần; không có va chạm ở phần lưới còn tránh được.
- Log có 614 sự kiện phanh, nhưng chưa phủ đủ vùng khó: mức phanh mạnh tham chiếu của người lái (p99.9) là 3.625 m/s², còn 8.556% lưới đòi hỏi phanh mạnh hơn mức này. Đây là mức theo phân vị, không phải cực đại tuyệt đối.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Theo `openloop.json` và mục `ranking` trong `closedloop.json`, lấy trung bình qua seed; hạng 1 là tốt nhất:

| Bộ điều khiển | L2 (m) | Hạng L2 | Va chạm trên log (%) | Hạng va chạm |
|---|---|---|---|---|
| constant-velocity | 1.255 | 3 | 20.985 | 3 |
| IDM | 1.542 | 4 | 0.104 | 1 |
| EgoMLP | 0.758 | 2 | 21.498 | 4 |
| PercMLP | 0.618 | 1 | 1.485 | 2 |

- Các cặp đổi thứ tự: IDM–constant-velocity, IDM–EgoMLP, IDM–PercMLP và EgoMLP–constant-velocity.
- IDM có L2 cao vì không cố bắt chước chính xác quỹ đạo người lái; nó dùng gap và tốc độ tương đối để giữ khoảng cách, phanh khi cần.
- EgoMLP chỉ nhìn lịch sử tốc độ của chính xe nên dễ bỏ lỡ xe trước phanh. L2 trung bình thấp không bảo đảm an toàn khi sai số tích lũy trong vòng kín.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

- Trong `closedloop.json`, EgoMLP dao động từ 15.470% đến 30.108% va chạm qua 5 seed: chênh 14.638 điểm phần trăm, gần gấp đôi; L2 chỉ dao động 0.755–0.766 m.
- Seed 5798 tốt nhất: 446/2.883 va chạm, Wilson CI95 là 14.196–16.836%; seed 5799 tệ nhất: 868/2.883, CI95 là 28.460–31.808%.
- Wilson CI thể hiện độ bất định do số quỹ đạo kiểm thử hữu hạn ở từng seed; nó không bao gồm toàn bộ dao động do training hay thay đổi môi trường.
- Nếu chỉ chạy seed 5798, tôi có thể báo 15.470%, thấp hơn constant-velocity (20.985%); chạy seed 5799 lại cho 30.108%, cao hơn. Chỉ một seed dễ dẫn đến kết luận chọn model sai.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Theo `agents.json`, bảng dưới là % ô có va chạm phía trước hoặc phía sau; hai cột trễ là độ trễ của xe ego:

| Bộ điều khiển | Xe sau phát lại, trễ 0 s | Xe sau phản ứng, trễ 0 s | Xe sau phát lại, trễ 0.6 s | Xe sau phản ứng, trễ 0.6 s |
|---|---|---|---|---|
| constant-velocity | 77.630 | 77.630 | 77.630 | 77.630 |
| IDM | 75.259 | 2.222 | 75.741 | 7.444 |
| EgoMLP | 84.111 | 84.111 | 81.815 | 81.815 |
| PercMLP | 79.963 | 18.259 | 76.815 | 29.593 |
| PercMLP+ | 75.148 | 3.148 | 75.444 | 8.296 |

- Hiệu chỉnh được T = 1.4 s; khoảng cách Wasserstein-1 giữa phân bố headway mô phỏng và người lái giảm từ 0.860 xuống 0.344, khoảng 60%. Xe sau phản ứng giống log hơn theo thước đo này, chưa chứng minh giống mọi hành vi người lái.
- Xe sau phát lại cứ giữ tốc độ dù ego phanh, nên có thể đâm vào IDM đang tránh xe trước. Với ego trễ 0 s, va chạm phía sau của IDM là 73.037% khi phát lại và 0% khi phản ứng.
- Vì vậy, phát lại khiến IDM bị đánh giá xấu do giả định xe sau không phản ứng; dùng xe sau phản ứng đưa IDM từ hạng nhì lên tốt nhất trong bảng này.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

- Theo `gate.json`, PercMLP bị chặn ở R2, R3, R4, R5 và R6: cận trên CI95 va chạm log 3.517% > 1.104%; cận trên ở nhóm xe trước phanh 19.703% > 5.575%; vùng đâm còn tránh được 22.148% > 2%; dao động seed 2.463 > 1 điểm phần trăm; va chạm với xe sau phản ứng 29.593% > 9.444%.
- Nguyên nhân chính là thiếu tình huống khó và các trạng thái do model tự lái tạo ra: nhóm sắp phanh chỉ chiếm 4.287% cửa sổ, còn 8.556% lưới đòi hỏi phanh mạnh hơn mức p99.9 của người lái trong log.
- Bản sửa thêm 20.000 kịch bản tổng hợp do IDM hướng dẫn và 2 vòng DAgger: cho model chạy trên train, lấy trạng thái nó gặp rồi gán nhãn bằng IDM. Cách này bổ sung tình huống thiếu và dạy model sửa sai khi lệch khỏi trạng thái của người lái.
- PercMLP+ qua cả 7 quy tắc: cận trên CI95 trên log còn 0.547%, vùng đâm còn tránh được 0.852%, va chạm với xe sau phản ứng 8.296% và dao động seed chỉ 0.035 điểm phần trăm.
- Chưa thể nói tốt hơn IDM: L2 thấp hơn (0.820 so với 1.542 m), nhưng va chạm log cao hơn (0.243–0.277% so với 0.104%) và vẫn có vùng đâm còn tránh được trong khi IDM là 0%.
- Tôi chưa đề xuất phát hành thực tế. `fix.json` cho thấy ở cruise chưa dùng để sửa là 20 m/s, vùng đâm còn tránh được vẫn 0.185–2.630% tùy seed; cần kiểm thử độc lập ở nhiều tốc độ, tình huống khó và trên đường thử trước khi quyết định.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

- Mô phỏng chỉ xét chuyển động dọc một chiều, nên không chứng minh an toàn khi chuyển làn, vào cua hoặc gặp người đi bộ. Cần mở rộng tình huống và kiểm thử trên đường thử.
- Xe trước vẫn phát lại log, xe sau chỉ là IDM hiệu chỉnh theo headway; tương tác giao thông thật có thể khác. Cần nhiều kiểu tác nhân và đối chiếu mô phỏng với dữ liệu tương tác thực.
- Chỉ có một cách chia dữ liệu, 5 seed và 174 quỹ đạo xe trước phanh trong test; kết quả ở tình huống hiếm còn bất định. Cần thêm dữ liệu độc lập, nhiều lần chia và nhiều sự kiện nguy hiểm.
- Lưới chính dùng cruise 12 m/s; thử ở 20 m/s vẫn còn tới 2.630% vùng đâm tránh được. Cần kiểm tra nhiều tốc độ, mức phanh và điều kiện ngoài tập training để đánh giá khả năng tổng quát.
- Nhiễu quan sát và độ trễ mô phỏng chưa đại diện đầy đủ cho lỗi cảm biến, giới hạn phanh và thời gian xử lý trên xe thật. Cần thử trên phần cứng, đường thử và chạy shadow trước khi cân nhắc phát hành.
