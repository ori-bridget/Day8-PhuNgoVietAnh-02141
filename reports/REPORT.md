# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Phu Ngo Viet Anh

Công cụ gán nhãn đã dùng: CVAT/YOLO Detection 1.0, sau đó kiểm tra và đóng gói các file nhãn YOLO.

Các số liệu dưới đây được lấy từ `day8_round0_out/outputs/metrics_round0.json`,
`day8_round1_out/outputs/metrics_round1.json`, `day8_round2_out/outputs/metrics_round2.json`,
các `rounds_table.md`, `round*_diff.md` và `reports/SELECTION.md`.

## 1. Dữ liệu và cách chia tập

Video được lấy mẫu 2.5 ảnh/giây, thu được 400 frame ở 1280×720. Dữ liệu được chia theo thời gian:
20 ảnh test, 112 ảnh buffer và 268 ảnh pool. Test gồm bốn đoạn quanh các mốc 20, 60, 100 và 140
giây; ảnh pool gần test nhất vẫn cách 4.4 giây.

Camera cố định và mỗi xe xuất hiện trong nhiều frame liên tiếp. Nếu chia ngẫu nhiên, các frame gần
như giống nhau hoặc cùng một chiếc xe có thể rơi vào cả train và test. Khi đó mô hình được chấm trên
cảnh/xe rất giống dữ liệu đã thấy, gây rò rỉ thời gian và làm AP, precision hoặc recall cao giả tạo.
Vùng đệm giúp phép đo gần hơn với khả năng tổng quát sang thời điểm khác.

## 2. Mô hình khởi đầu lạnh (cold start)

Dòng vòng 0 trong `rounds_table.md` là:

| vòng | model | ảnh train | box train | AP50 | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

`compare_round0.jpg` cho thấy cold start bắt được nhiều xe lớn/trung bình có thân rõ, nhưng bỏ sót
nhiều xe nhỏ ở xa, xe tối và xe bị che một phần; một số box cũng bám vào vùng đèn/đèn hậu nên cần
đối chiếu guideline. Recall small chỉ 0.182, thấp hơn nhiều so với medium 0.547 và large 0.561,
cho thấy điểm yếu chính là xe nhỏ/xa chứ không chỉ là thiếu precision.

Trước khi kết luận mô hình sai, cần rà các box rất nhỏ ở gần đường chân trời, ví dụ các cụm đèn trong
frame `frame_0250.jpg`. Nhãn tham chiếu cũng do model tạo và chưa được người kiểm từng box; hơn nữa
14 box cao dưới 16 pixel bị bỏ qua khi chấm. Vì vậy một khác biệt ở vùng chỉ còn hai chấm đèn có thể là
khác biệt của nhãn tham chiếu, không phải lỗi detector.

## 3. Chiến lược chọn mẫu

`U` là trung bình độ bất định của tối đa năm box khó nhất, với mỗi box có
`u(c)=1-|2c-1|`; confidence gần 0.5 thì bất định cao. `A` là số box mập mờ
(`0.15 <= confidence < 0.50`) chuẩn hóa theo mức lớn nhất trong pool. `D` là khoảng cách thời gian
đến frame đã chọn gần nhất, bị chặn ở 10 giây. Với trọng số mặc định,
`score = 0.5*U + 0.3*A + 0.2*D`.

`MIN_GAP_S=2.0` loại các frame cách nhau dưới 2 giây vì camera cố định khiến chúng gần như trùng
cảnh; nếu chưa đủ 12 frame, thuật toán mới nới khoảng cách dần. Điều này giảm công gán nhãn trùng
lặp nhưng không đảm bảo mọi cảnh khó đều được chọn.

Ba frame trong lô 12 ảnh được phân tích ở `SELECTION.md` là `frame_0182.jpg` (rank 1, score 0.9591,
28 box và 18 vùng mơ hồ), `frame_0099.jpg` (rank 8, score 0.9063, U=0.9460, A=0.7778) và
`frame_0392.jpg` (rank 15, score 0.8874, U=0.9747, A=0.6667). Chúng lần lượt đại diện cho cảnh
đông xe nhiều vùng khó, xe tối/vệt đèn và cảnh có bất định rất cao. `frame_0372.jpg` (rank 6,
score 0.9101) không được chọn vì nằm giữa `frame_0369.jpg` và `frame_0380.jpg` trong cùng chuỗi
thời gian; dùng nó sẽ tốn lượt rà mà ít tăng đa dạng. Điểm bất định chỉ là tín hiệu ưu tiên kiểm tra,
không chứng minh ảnh đó chắc chắn cải thiện mô hình: điểm có thể bị ảnh hưởng bởi nhiễu confidence,
nhãn tham chiếu chưa rà và tương quan giữa các frame.

## 4. Các vòng học chủ động (active learning)

Bảng dưới đây gộp các dòng vòng 0, 1 và 2 từ các `rounds_table.md` tương ứng. Tất cả các vòng
đánh giá cùng 20 ảnh test, 403 box tham chiếu và bỏ qua 14 box cao dưới 16 pixel:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 330 | 0.590 | -0.181 | 1.000 | 0.124 | 0.221 | 0.000 | 0.115 | 0.390 |
| 2 | yolov8n fine-tune vong 1..2 | 24 | 571 | 0.913 | +0.142 | 0.923 | 0.682 | 0.785 | 0.349 | 0.726 | 0.902 |

Mức sửa nhãn truy được từ các diff là:

| vòng | ảnh | giữ nguyên | chỉnh sửa | xoá | thêm | nhãn trước → sau |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 12 | 144 | 9 | 16 | 177 | 169 → 330 |
| 2 | 12 | 47 | 0 | 0 | 194 | 47 → 241 |

Vòng 1 có accept rate 85%: giữ 144, chỉnh 9, xoá 16 và thêm 177 box. Sau fine-tune vòng 1, AP50
giảm 0.181 so với cold start; recall giảm từ 0.489 xuống 0.124, trong đó recall xe nhỏ giảm về
0.000. Vòng 2 có accept rate 100%: giữ 47 và thêm 194 box. Sau khi train trên cả 24 ảnh, AP50 tăng
0.142 so với cold start và tăng 0.323 so với vòng 1; recall đạt 0.682 và F1 đạt 0.785. Recall theo
kích thước ở vòng 2 cũng tăng lên 0.349 (small), 0.726 (medium) và 0.902 (large). Precision giảm
nhẹ từ 0.925 xuống 0.923 nhưng vẫn gần mức cold start.

Ảnh `compare_round1.jpg` cho thấy một ca xấu đi rõ: ở `frame_0050.jpg`, cold start có TP=11,
FP=2, FN=7, còn vòng 1 chỉ TP=1, FP=0, FN=17. Ngược lại, `compare_round2.jpg` cho thấy vòng 2
phục hồi và tốt hơn cold start ở cùng frame: TP=15, FP=0, FN=3. Khả năng kiểm tra là vòng 1 học
quá ít ảnh/đặc trưng chưa ổn định nên bỏ qua nhiều xe nhỏ; vòng 2 bổ sung 241 box từ lô mới và
khôi phục độ phủ. Đây vẫn là so sánh với nhãn tham chiếu do model tạo, không phải đánh giá người.

`BLIND_SCAN.md` là quan sát độc lập trước khi xem pre-label: ở `frame_0312.jpg` tôi đếm 24 xe và
ghi hai vùng dễ bị bỏ sót/che khuất. `REVIEW_LOG.csv` ghi bốn ca cụ thể đã thêm box ở vòng 1; đó là
thay đổi của nhãn người sửa, không phải kết quả model sau train. `round1_diff.md` là phép đối chiếu
định lượng giữa pre-label và nhãn cuối. Một ca khó theo guideline là xe chỉ lộ một phần sau xe tải:
chỉ vẽ phần thân nhìn thấy, không kéo box sang vùng bị che; với vệt đèn trên mặt đường cũng chỉ
khoanh thân xe, không khoanh vệt sáng.

## 5. Kết luận và giới hạn

Vòng 1 tạo 330 box cho 12 ảnh và vòng 2 tạo 241 box cho 12 ảnh (trong đó thêm 194 box); phần lớn là xe cold start bỏ sót.
Vòng 1 làm kết quả giảm mạnh, nhưng vòng 2 đưa AP50 lên 0.913, cao hơn cold start 0.142 và cải
thiện rõ recall small/medium/large. Vì vậy nên dừng việc gán thêm ở thời điểm này để kiểm tra tính
ổn định của kết quả và giữ lại model vòng 2 làm mốc hiện tại.

Hai ứng viên còn yếu/bất định cho vòng sau là `frame_0392.jpg` (U=0.9747, A=0.6667, nhiều xe ở
cảnh đêm) và `frame_0372.jpg` (score=0.9101 nhưng gần các frame 0369/0380). Ca đầu có chi phí rà
cao vì nhiều box nhỏ; ca sau có nguy cơ lãng phí vì gần trùng, nên chỉ chọn nếu kết quả vòng trước
cho thấy cần thêm đại diện của đoạn thời gian đó.

Kết luận bị giới hạn bởi chỉ 20 ảnh test, 14 box rất nhỏ bị bỏ qua, video một cảnh với tương quan
thời gian cao, và nhãn tham chiếu do model tạo chưa được người rà thủ công. Nếu AP50 giảm ở một
vòng sau, trước hết cần kiểm tra `round*_diff` (box thêm/xoá/chỉnh), class id và tọa độ YOLO, việc
đóng gói đúng train split, rồi đối chiếu `compare_round*.jpg`; tuyệt đối không sửa test label để làm
đẹp chỉ số. Chỉ sau các kiểm tra đó mới quyết định train thêm hoặc chọn lô mới.
