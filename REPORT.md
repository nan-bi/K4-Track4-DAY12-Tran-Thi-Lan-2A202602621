# REPORT — Lab Day 12

> Kết quả chạy đầy đủ với MSSV 2A202602621, 5 seed, không dùng --quick. Các số liệu và giải thích đối chiếu với evidence/; môi trường chạy và SHA-256 của dữ liệu nằm trong evidence/runtime.txt.

|                                                                                  |              |
| -------------------------------------------------------------------------------- | ------------ |
| Họ và tên                                                                        | Trần Thị Lan |
| MSSV (đúng bằng `--student-id` bạn đã chạy)                                      | 2A202602621  |
| GitHub                                                                           | nan-bi       |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 3588 / 9235, 9236, 9237, 9238, 9239 |
| Số seed huấn luyện bạn dùng (`--seeds`)                                          | 5 |

## Kết quả cuối

Nguồn: `evidence/openloop.json`, `closedloop.json`, `gate.json`. Mô hình học ghi dải nhỏ nhất–lớn nhất qua 5 seed; L2 từng seed của PercMLP+ được tính lại từ model đã lưu và ghi trong `evidence/l2_per_seed.txt`.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---:|---:|---|
| constant-velocity | 1,2715 | 20,742 | BLOCK |
| IDM | 1,4997 | 0,104 | baseline |
| EgoMLP | 0,7621–0,7712 | 7,249–33,680 | BLOCK |
| PercMLP | 0,6183–0,6282 | 0,451–3,052 | BLOCK |
| PercMLP+ (sau khi sửa) | 0,7926–0,8401 | 0,173–0,243 | RELEASE |

## Câu 1 · Dữ liệu

Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Dữ liệu có 992.679 dòng, 9.993 quỹ đạo ban đầu; loại 383 bản trùng còn 9.610 quỹ đạo (evidence/data_audit.json).
Tương quan Acc_LV với đạo hàm Speed_LV chỉ 0,04535, trong khi phía xe theo sau là 0,92909; vì vậy cột Acc_LV không đáng tin để nhận diện phanh.
Tôi dùng a_lead: đạo hàm trung tâm của Speed_LV sau trung bình trượt 5 mẫu, thay vì ngưỡng trên gia tốc thô.
Nếu chia trước khi loại trùng, cùng một tình huống có thể xuất hiện ở cả train và test dù mang ID khác, làm điểm kiểm thử tốt giả tạo.
Ngay cả khi đã loại trùng, chia theo khung hình vẫn làm rò rỉ các cửa sổ liền kề gần như giống nhau; phải giữ toàn bộ một quỹ đạo trong cùng một tập.
Seed chia 3588 tạo tập kiểm thử 2.883 quỹ đạo không giao ID với tập huấn luyện; log vẫn chỉ là các đoạn ngắn 70–109 mẫu nên không đủ đại diện mọi tình huống.

## Câu 2 · Phát lại

Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Trên 600 đoạn, chạy lại cùng mã khác 0 đoạn; float32 khác bit ở cả 600 đoạn nhưng 0 đoạn vượt atol=rtol=0,001; sai khác lớn nhất chỉ 0,00001825 m/s² (evidence/replay.json).
Đổi thứ tự phép tính và độ chính xác làm thay đổi làm tròn số thực, nên yêu cầu bằng bit quá chặt; dung sai tuyệt đối cộng tương đối theo lệnh chuẩn giữ được thay đổi hành vi đáng kể.
AEB làm 8 đoạn vượt dung sai. Tôi mở đoạn 171: các khung 65, 66, 67 có TTC lần lượt 1,4656; 1,4167; 1,4088 s, đều dưới ngưỡng 1,5 s.
Lệnh tương ứng đổi từ khoảng -5,706; -5,628; -5,591 sang -6 m/s²; chỉ khi đang tiến gần và lệnh IDM chưa phanh mạnh bằng AEB thì đầu ra mới thay đổi (evidence/replay_aeb_inspection.txt).
Phát nhanh nhất có 0 khung trễ và 0 khung rớt; phát theo thời gian log có 582 khung trễ, 578 khung rớt trên 59.788 khung, làm 377 đoạn vượt dung sai.
Đây là chi phí xử lý được mô phỏng: phát nhanh nhất không áp hạn chót, còn phát theo thời gian giữ lệnh cũ khi xử lý quá chậm, nên làm lộ ảnh hưởng của trễ hạn.

## Câu 3 · Kịch bản

Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới gồm 60 khoảng cách từ 2,7601 đến 59,3123 m và 45 tốc độ tiến gần từ 0 đến 10 m/s: 2.700 ô, tốc độ cruise 10 m/s (evidence/scenarios.json).
Với độ trễ τ, xe mất dv·τ mét trước khi phanh, rồi cần dv²/(2·b_max) mét để triệt tiêu tốc độ tiến gần; gap nhỏ hơn tổng này là bất khả thi.
Ở τ=0, vùng bất khả thi và va chạm IDM đều chiếm 0,963%; ở τ=0,6 s cùng tăng lên 5,148%, tức tăng 4,185 điểm phần trăm.
FullBrake lệch biên vật lý ở 0 ô xa biên tại cả hai độ trễ; IDM cũng có diện tích va chạm có thể quy trách nhiệm bằng 0, nên không được gọi các ô bất khả thi là lỗi của IDM.
Khai thác được 614 ca phanh, chiếm 6,389% quỹ đạo; các chấm log tập trung ở tốc độ tiến gần thấp, ít phủ góc khoảng cách nhỏ và tiến gần nhanh trên hình scenarios_idm.png.
Giới hạn phanh người lái theo phân vị 99,9 là 3,625 m/s², không phải cực đại tuyệt đối; 5,222% lưới cần mạnh hơn mức đó, cho thấy khoảng trống bao phủ của log.

## Câu 4 · Vòng hở và vòng kín

Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng dưới lấy L2 trung bình và tỷ lệ va chạm trung bình qua seed từ evidence/closedloop.json, mục ranking; hạng 1 là tốt nhất.

| Bộ điều khiển | L2 trung bình (m) | Hạng L2 | Va chạm trung bình (%) | Hạng vòng kín |
|---|---:|---:|---:|---:|
| constant-velocity | 1,2715 | 3 | 20,742 | 3 |
| IDM | 1,4997 | 4 | 0,104 | 1 |
| EgoMLP | 0,7661 | 2 | 21,332 | 4 |
| PercMLP | 0,6235 | 1 | 1,353 | 2 |

Có 4 cặp đảo thứ tự: constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP.
IDM không tối ưu việc bắt chước vị trí người lái nên L2 lớn, nhưng dùng khoảng cách và tốc độ tương đối để phanh khi tự lái; vì vậy nó chỉ va chạm 3/2.883 log.
EgoMLP dự đoán được quán tính từ lịch sử tốc độ nhưng không nhìn xe dẫn hay khoảng cách, nên không phân biệt được hai cảnh có cùng lịch sử xe mình mà rủi ro khác nhau; sai số còn tích luỹ trong vòng kín.
Lát cắt xe dẫn sắp phanh chỉ chiếm 4,698% cửa sổ; L2 EgoMLP tăng từ 0,766 lên 1,124 m và PercMLP từ 0,624 lên 0,885 m, cho thấy điểm trung bình che phần khó (evidence/openloop.json).

## Câu 5 · Một lần chạy chưa đủ

EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Trên cùng 2.883 log, EgoMLP ở seed 9235–9239 lần lượt va chạm 29,899%; 33,680%; 16,198%; 19,632%; 7,249% (evidence/closedloop.json, logs.EgoMLP.per_seed).
Dải 7,249–33,680% rộng 26,431 điểm phần trăm; trung bình 21,332%, nên kết luận từ một seed rất nhạy với khởi tạo và huấn luyện.
Seed tốt nhất 9239 có Wilson 95% [6,359%; 8,254%], còn seed xấu nhất 9236 có [31,978%; 35,426%]; hai khoảng không chồng nhau.
Wilson mô tả bất định của tỷ lệ va chạm với số log đã thử, không thay thế việc đo độ biến động giữa các seed và không chứng minh an toàn ngoài phân phối này.
Nếu chỉ chọn seed 9239, tôi có thể báo 7,249% và cho rằng EgoMLP tốt hơn constant-velocity 20,742%; seed 9236 lại cho 33,680%, tức kém hơn rõ rệt.
Vì thế phải công bố mọi seed, dải và khoảng tin cậy, đồng thời dùng seed xấu nhất khi kiểm tra cổng theo quy định.

## Câu 6 · Tác nhân phản ứng

Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Từ evidence/agents.json, bảng dưới dùng độ trễ xe thử 0,6 s; các tỷ lệ tính trên toàn bộ 2.700 ô, mô hình học lấy trường hợp xấu nhất theo cách tổng hợp của bước agents.

| Bộ điều khiển | Xe sau phát lại: mọi va chạm (%) | Xe sau phát lại: va chạm sau (%) | Xe sau phản ứng: mọi va chạm (%) | Xe sau phản ứng: va chạm sau (%) |
|---|---:|---:|---:|---:|
| constant-velocity | 73,370 | 0 | 73,370 | 0 |
| IDM | 70,074 | 64,926 | 5,148 | 0 |
| EgoMLP | 76,296 | 0 | 76,296 | 0 |
| PercMLP | 69,037 | 34,852 | 34,185 | 0 |

T được chọn là 1,4 s vì W1 giữa phân bố thời khoảng cách và người lái nhỏ nhất: 0,34851, so với 0,85524 của tác nhân không phản ứng; thời khoảng cách trung vị người lái là 1,86483 s.
Xe sau được đặt cách 16 m và phản ứng trễ 0,6 s. Khi chỉ phát lại tốc độ, nó không biết IDM vừa phanh nên tạo 64,926% va chạm sau; với tác nhân phản ứng, tỷ lệ đó về 0.
Nếu chấm bằng thế giới phát lại, IDM trông gần constant-velocity (70,074% so với 73,370% mọi va chạm); thế giới phản ứng cho thấy IDM tốt hơn nhiều (5,148% so với 73,370%).
Thậm chí phát lại xếp PercMLP 69,037% tốt hơn IDM 70,074%, trong khi phản ứng đảo lại 34,185% so với 5,148%; tác nhân sai có thể đổi kết luận, không chỉ làm lệch một con số.
W1 thấp hơn chỉ cho thấy khớp phân bố này tốt hơn, không bảo đảm tác nhân IDM đại diện mọi người lái.

## Câu 7 · Cổng phát hành và sửa

Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Constant-velocity bị chặn bởi R2, R3, R4, R6; EgoMLP và PercMLP đều bị chặn bởi R2–R6, trong khi R1 và R7 đạt (evidence/gate.json).
Với PercMLP, R2 có cận trên va chạm 3,745% vượt ngưỡng 1,104%; R3 có cận trên lát cắt 16,989% vượt 6,478%; R5 có dải 0,451–3,052%, rộng 2,601 điểm phần trăm vượt mức 1.
R4 có diện tích tránh được nhưng vẫn va chạm 29,037%, vượt baseline 0% cộng 2 điểm; R6 có 34,185% mọi va chạm trong thế giới phản ứng, vượt 5,148% cộng 2 điểm. Đây là bằng chứng về họ cắt vào khó mà log chưa phủ; R2, R3 và độ nhạy seed cho thấy thêm vấn đề sai số khi mô hình tự lái.
Tôi bổ sung 20.000 kịch bản cắt vào với nhãn IDM và 2 vòng DAgger lấy trạng thái do chính mô hình đi vào trên log huấn luyện, rồi gắn dự báo IDM 3 giây tại các trạng thái ấy; không dùng log test để huấn luyện hay đổi ngưỡng.
Sau sửa, R2–R6 đều chuyển sang đạt và PercMLP+ được RELEASE: va chạm trên log 0,173–0,243%, cận trên seed xấu nhất 0,500%; R4 còn 0,148%, R6 còn 5,296%. L2 trung bình đổi từ 0,6235 lên 0,8126 m, cho thấy cải thiện an toàn vòng kín không đồng nghĩa với điểm bắt chước vòng hở đẹp hơn.
Ở phép thử bổ sung cruise 20 m/s, PercMLP+ vẫn có 0,111–4,222% lưới tránh được nhưng va chạm, còn IDM là 0% (evidence/fix.json). PercMLP+ học từ IDM, nên qua cổng chỉ là đạt tiêu chí không kém hơn trong bộ thử này, chưa chứng minh tốt hơn IDM; tôi chưa đề nghị thay baseline ngay, mà cần tập tình huống độc lập, kiểm thử ngoài phân phối, cảm biến và phần cứng trước khi quyết định.

## Câu 8 · Bài lab này không chứng minh được gì

Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Bài lab chỉ mô phỏng chuyển động dọc một chiều, nên chưa kiểm tra đánh lái, đổi làn hay xung đột ở giao lộ; cần mở rộng tình huống và kiểm thử tương tác ngang trước khi quyết định phát hành.
Mô hình nhận trực tiếp khoảng cách và tốc độ, không qua cảm biến thật; sai số, che khuất và mất dấu có thể làm TTC sai và phanh muộn, nên cần kiểm thử cả chuỗi cảm nhận cùng độ trễ và lỗi tín hiệu thực tế.
Xe dẫn phát lại log không phản ứng với xe thử; dù có xe sau IDM phản ứng, hành vi đó vẫn do giả định mô phỏng quyết định, nên cần nhiều mô hình tác nhân và dữ liệu tương tác thật để kiểm tra độ nhạy của kết luận.
Tần số 10 Hz, giới hạn phanh 9 m/s² và các tham số mô phỏng không đại diện cho mọi mặt đường, thời tiết hay cơ cấu chấp hành; cần hiệu chỉnh bằng đo đạc và kiểm tra trên nhiều điều kiện vận hành.
Wilson 95% và dải qua seed chỉ mô tả bất định trong tập thử đã chọn, không bảo đảm an toàn trước tình huống hiếm chưa xuất hiện. Qua cổng chỉ đủ để tiếp tục kiểm thử trên tập độc lập, phần cứng trong vòng lặp và đường thử có giám sát, chưa đủ căn cứ phát hành thực tế.
