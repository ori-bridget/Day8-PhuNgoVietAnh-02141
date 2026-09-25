# Vì sao chọn lô này?

Tôi dùng `outputs/selection_round1.csv` (50 ứng viên đầu) cùng contact sheet
`outputs/selection_round1.jpg`. Điểm `score` là điểm ưu tiên để tìm ảnh khó và đa dạng,
không phải điểm chính xác của detector. Nếu chỉ có ngân sách rà năm ảnh, đề xuất năm frame
sau:

| thứ tự ưu tiên | frame | thời điểm (s) | rank CSV | score | bằng chứng và lý do |
|---:|---|---:|---:|---:|---|
| 1 | `frame_0182.jpg` | 72.8 | 1 | 0.9591 | Điểm cao nhất; U=0.9182, 28 box và 18 vùng mơ hồ, phù hợp để rà các xe tối/xe bị che. |
| 2 | `frame_0369.jpg` | 147.6 | 2 | 0.9324 | U=0.9315 và 43 box; nhiều đối tượng trong cảnh đêm, có giá trị kiểm tra bỏ sót nhưng vẫn đủ khác các cảnh đầu video. |
| 3 | `frame_0326.jpg` | 130.4 | 4 | 0.9155 | U=0.9310, 39 box và 15 vùng mơ hồ; ưu tiên hơn frame gần trùng `frame_0331.jpg` vì sớm hơn trong cùng đoạn cảnh. |
| 4 | `frame_0099.jpg` | 39.6 | 8 | 0.9063 | Bổ sung một thời điểm khác hẳn; U=0.9460 nhưng A=0.7778 cho thấy cần người kiểm tra các box xe nhỏ/xe tối. |
| 5 | `frame_0227.jpg` | 90.8 | 11 | 0.8915 | Đa dạng theo thời gian và bố cục; giúp tránh dùng cả năm lượt cho các frame liên tiếp của một cảnh. |

Ba frame trong đúng lô 12 ảnh model đã chọn để đối chiếu trực tiếp trên CSV/contact sheet là:

- `frame_0182.jpg`: rank 1, score 0.9591, `n_ambiguous=18`; contact sheet cho thấy nhiều xe sáng đèn ở nhiều làn, gồm cả xe gần mép dưới, nên đây là ca bất định cao nhất trong lô.
- `frame_0099.jpg`: rank 8, score 0.9063, `U=0.9460`, `A=0.7778`; contact sheet cho thấy xe tối xen giữa các vệt đèn trên đường, mô hình tự tin về độ khó nhưng các phép đo đồng thuận thấp hơn.
- `frame_0392.jpg`: rank 15, score 0.8874, `U=0.9747`, `A=0.6667`, 35 box; contact sheet cho thấy cảnh đêm đông xe và nhiều xe bị cắt ở mép dưới. Không nằm trong năm lượt vì ngân sách, nhưng là ứng viên rà bổ sung tốt do bất định rất cao.

Ứng viên điểm cao nhưng không chọn là `frame_0372.jpg` (rank 6, score 0.9101, thời điểm
148.8 s). Ảnh này nằm giữa `frame_0369.jpg` (147.6 s) và `frame_0380.jpg` (152.0 s),
nên nhiều khả năng gần trùng cùng một đoạn cảnh. Với ngân sách năm ảnh, tôi giữ
`frame_0369.jpg` làm đại diện và dành lượt còn lại cho các thời điểm 39.6 s và 90.8 s.
Tương tự, `frame_0331.jpg` (rank 5, 132.4 s) được loại để tránh rà cả hai frame gần nhau
với `frame_0326.jpg`.

Phép chọn này chỉ tối ưu thứ tự rà nhãn dựa trên điểm bất định, số box và độ đa dạng thời
gian. Nó không chứng minh AP, precision hay recall của mô hình; chất lượng cuối cùng chỉ có
thể kết luận sau khi sửa nhãn, fine-tune và đánh giá trên tập test giữ riêng.
