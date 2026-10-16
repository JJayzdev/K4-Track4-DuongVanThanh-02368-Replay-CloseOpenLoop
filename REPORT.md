# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Dương Văn Thành |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602368 |
| GitHub | JJayzdev |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split_seed: 851, model_seed: 1754 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 (bước fix) và 5 (các bước còn lại) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.256 | 19.9% | BLOCK |
| IDM | 1.493 | 0.1% | (baseline) |
| EgoMLP | 0.760 ± 0.005 | 12.0..22.8% | BLOCK |
| PercMLP | 0.622 ± 0.002 | 0.6..1.0% | BLOCK |
| PercMLP+ (sau khi sửa) | 0.824 ± 0.018 | 0.2..0.4% | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong Bước 0 kiểm toán dữ liệu ([evidence/data_audit.json](file:///e:/VinAI/Track4/K4-Track4-DuongVanThanh-02368-Replay-CloseOpenLoop/evidence/data_audit.json)), tôi phát hiện 3 vấn đề nghiêm trọng:
1. Cột `Acc_LV` (gia tốc ghi nhận của xe dẫn) hoàn toàn không đáng tin cậy vì hệ số tương quan với đạo hàm vận tốc d(Speed_LV)/dt chỉ đạt 0.0454 (gần như bằng 0), trong khi ở xe theo sau cột `Acc_FAV` có độ tương quan rất cao là 0.9291. Do đó, ta không thể dùng `Acc_LV` thô mà phải thay thế bằng gia tốc tính từ đạo hàm của `Speed_LV` sau khi đã làm mượt (smoothed) bằng bộ lọc để triệt tiêu nhiễu đo lường.
2. Có chính xác 383 quỹ đạo bị loại bỏ vì trùng lặp hoàn toàn (từ 9.993 quỹ đạo thô giảm xuống còn 9.610 quỹ đạo sạch). Các bản ghi trùng lặp này xuất hiện ở các Trajectory_ID cách xa nhau; nếu chia dữ liệu ngẫu nhiên theo kiểu thông thường (naive split), các bản sao giống hệt nhau sẽ rơi đồng thời vào cả tập train và tập test, dẫn đến hiện tượng rò rỉ dữ liệu (data leakage) và đánh giá quá lạc quan.
3. Việc phân chia train/test bắt buộc phải thực hiện theo từng quỹ đạo độc lập (trajectory-level split với 851 split_seed) thay vì chia theo từng khung hình (frame-level). Lý do là các khung hình liên tiếp trong cùng một quỹ đạo (chuỗi 7–11 giây, 70–109 bước ở tần số 10 Hz) có tính tự tương quan thời gian cực kỳ mạnh; chia theo khung hình sẽ khiến mô hình học vẹt trạng thái ngay sát trước đó của tập kiểm tra.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Từ kết quả phát lại ([evidence/replay.json](file:///e:/VinAI/Track4/K4-Track4-DuongVanThanh-02368-Replay-CloseOpenLoop/evidence/replay.json)):
1. Bản refactor `float32` bị khác bit ở toàn bộ 600/600 đoạn (not_bit_identical = 600) do phép cộng dấu phẩy động trong máy tính không có tính chất kết hợp `(0.1+0.2)+0.3 != 0.1+(0.2+0.3)` và việc đổi thứ tự tích luỹ làm lệch các bit cuối. Tuy nhiên, có đúng 0 đoạn vượt dung sai (`differ_beyond_tol = 0` với atol=1e-3, rtol=1e-3), độ lệch tuyệt đối cực đại chỉ là 8.94e-05 m/s². Điều này chứng minh việc kiểm thử phần mềm tự hành đòi hỏi kiểm tra dung sai số học (tolerance) thay vì so khớp bit tuyệt đối.
2. Thay đổi luật AEB (ngưỡng TTC can thiệp 1.5 s) làm lệch đúng 14/600 đoạn (các đoạn [54, 63, 72, 137, 231...]). Khi mở một đoạn bị lệch (như đoạn 54), các khung hình bị lệch xuất hiện chính xác ở thời điểm khoảng cách co hẹp nhanh khiến TTC rơi xuống dưới 1.5 s. Khi đó luật AEB mới kích hoạt phanh khẩn cấp cưỡng bức (lệnh gia tốc vọt xuống mức phanh tối đa, lệch lớn nhất tới 1.976 m/s² so với lệnh IDM mượt mà trước đó).
3. Chế độ phát nhanh nhất (`fastest`) giả định hệ thống có tài nguyên tính toán vô hạn nên không hề có khái niệm hạn chót (deadlines do not exist), ghi nhận 0 khung hình bị trễ (late = 0) và 0 khung hình bị rớt (dropped = 0). Ngược lại, phát theo thời gian thực (`realtime`) mô phỏng đúng độ trễ xử lý thực tế, làm lộ ra 580 khung trễ và 577 khung bị rớt trên tổng số 59.378 khung hình, khiến 381/600 đoạn bị thay đổi hành vi và sai lệch gia tốc lên tới 24.94 m/s².

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Theo kết quả khảo sát kịch bản ([evidence/scenarios.json](file:///e:/VinAI/Track4/K4-Track4-DuongVanThanh-02368-Replay-CloseOpenLoop/evidence/scenarios.json)):
1. Lưới kịch bản được xây dựng gồm 60 khoảng cách (từ 2.76 m đến 59.31 m) và 45 vận tốc tiến gần (từ 0.0 đến 10.0 m/s với tốc độ cruise 10.0 m/s), tạo thành 2.700 ô kịch bản. Ở độ trễ phản ứng t_react = 0.0 s, vùng vật lý không thể tránh khỏi va chạm (unavoidable) chiếm 0.96% (~1.0%) diện tích lưới (26/2.700 ô).
2. Khi tăng độ trễ phản ứng từ 0.0 s lên 0.6 s, khoảng cách an toàn bị mất thêm một đoạn `dv * t_react`, khiến vùng bất khả thi tăng vọt lên 5.15% (~5.1%, 139/2.700 ô). Vùng va chạm của IDM trên lưới cũng tăng tương ứng từ 1.0% lên 5.1%, và diện tích va chạm quy trách nhiệm (attributable area = diện tích đâm trừ diện tích bất khả thi) của IDM duy trì chính xác bằng 0.0% (mismatch away from frontier = 0 ở cả hai độ trễ), chứng minh IDM xử lý hoàn hảo mọi tình huống có thể tránh được về mặt vật lý.
3. Phân tích tập log cho thấy dữ liệu lái xe tự nhiên của con người hoàn toàn không phủ các vùng nguy hiểm: mức phanh gắt nhất của người lái trong 9.610 quỹ đạo (phân vị 99.9%) chỉ đạt 3.62 m/s² (trong khi cơ cấu chấp hành có thể phanh tới 9.0 m/s²), và chỉ có 614/9.610 quỹ đạo (6.4%) có sự kiện xe dẫn phanh. Có tới 5.22% diện tích lưới kịch bản đòi hỏi mức phanh lớn hơn 3.62 m/s². Vì vậy các mô hình học máy chỉ học từ log thông thường sẽ bị mù hoàn toàn ở các góc nguy hiểm có gia tốc phanh lớn.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh vòng hở (L2) và vòng kín (va chạm) trích từ [evidence/closedloop.json](file:///e:/VinAI/Track4/K4-Track4-DuongVanThanh-02368-Replay-CloseOpenLoop/evidence/closedloop.json):

| Bộ điều khiển | L2 vòng hở trung bình (m) | Hạng L2 | Tỷ lệ va chạm vòng kín (%) | Hạng vòng kín |
|---|---|---|---|---|
| const-vel | 1.256 m | 3 | 19.9% | 4 |
| IDM | 1.493 m | 4 | 0.1% | 1 |
| EgoMLP | 0.760 m | 2 | 17.0% (tệ nhất 22.8%) | 3 |
| PercMLP | 0.622 m | 1 | 0.8% (tệ nhất 1.0%) | 2 |

Ba cặp bị đảo lộn thứ tự hoàn toàn giữa hai cách chấm là: `(const-vel, IDM)`, `(IDM, EgoMLP)`, và `(IDM, PercMLP)`.
- IDM có sai số L2 vòng hở tệ nhất (1.493 m, hạng 4) nhưng lại đạt tỷ lệ va chạm vòng kín thấp nhất (0.1%, hạng 1, chỉ đâm 2/2.883 quỹ đạo). Lý do là IDM được thiết kế theo luật điều khiển phản hồi với mục tiêu duy trì khoảng cách an toàn động lực học; nó không cố gắng bắt chước chính xác từng biến thiên nhỏ trong hành vi ga/phanh của người lái trong log (vốn mang tính chủ quan), nhưng khi đưa vào vòng kín, các phản hồi điều khiển kịp thời giúp xe không bao giờ đâm.
- EgoMLP có L2 vòng hở rất đẹp (0.760 m, tốt hơn nhiều so với const-vel 1.256 m và IDM 1.493 m) nhưng khi chạy vòng kín lại đâm tới 17.0% (thậm chí 22.8% ở seed xấu nhất, và trên lát cắt xe dẫn phanh đâm tới 51.1%). Nguyên nhân là EgoMLP bị mù hoàn toàn thông tin môi trường (chỉ nhận lịch sử vận tốc bản thân, không có khoảng cách `gap` và vận tốc xe dẫn `v_lead`). Trong vòng hở, do xe người lái trong log hiếm khi phanh gấp nên mô hình dự đoán trôi mượt theo vận tốc cũ vẫn đạt L2 thấp. Khi tự lái vòng kín, gặp tình huống xe dẫn phanh gấp, EgoMLP không hề nhận biết được chướng ngại vật phía trước để giảm tốc, dẫn đến đâm liên tục.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Dữ liệu qua 5 seed huấn luyện độc lập của EgoMLP ([evidence/closedloop.json](file:///e:/VinAI/Track4/K4-Track4-DuongVanThanh-02368-Replay-CloseOpenLoop/evidence/closedloop.json)) thể hiện sự dao động rất lớn:
1. Tỷ lệ va chạm của EgoMLP dao động từ 12.0% đến 22.8% qua 5 seed (seed 1754: 19.4%, seed 1755: 16.8%, seed 1756: 12.0%, seed 1757: 13.7%, seed 1758: 22.8%), tức chênh lệch tới 10.8 điểm phần trăm giữa seed tốt nhất và seed xấu nhất.
2. Khoảng tin cậy Wilson 95% cho thấy sự tách biệt hoàn toàn giữa hai thái cực: seed 1756 tốt nhất có CI95 là [10.9%, 13.3%], trong khi seed 1758 xấu nhất có CI95 là [21.3%, 24.4%]. Hai khoảng tin cậy này hoàn toàn không giao nhau (disjoint), chứng tỏ sự khác biệt hiệu năng giữa các lần khởi tạo ngẫu nhiên là có ý nghĩa thống kê sâu sắc chứ không phải do sai số lấy mẫu ngẫu nhiên.
3. Nếu kỹ sư chỉ chạy đúng một lần duy nhất và ngẫu nhiên chọn trúng seed 1756, họ có thể tự tin báo cáo tỷ lệ va chạm là 12.0% và tin rằng mô hình khá ổn định. Điều này cực kỳ nguy hiểm trong công nghệ xe tự hành vì nó che giấu sự bất ổn định bản chất của quá trình tối ưu phi lồi (non-convex optimization), dẫn đến việc đánh giá sai lệch gần một nửa số ca va chạm thực tế và đưa một hệ thống chưa kiểm soát được rủi ro ra vận hành thực địa.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh va chạm toàn diện (trước và sau) ở độ trễ phản ứng t_react = 0.6 s ([evidence/agents.json](file:///e:/VinAI/Track4/K4-Track4-DuongVanThanh-02368-Replay-CloseOpenLoop/evidence/agents.json)):

| Bộ điều khiển | Xe sau phát lại log (Tổng / Va chạm sau) | Xe sau phản ứng (Tổng / Va chạm sau) |
|---|---|---|
| const-vel | 73.4% / 0.0% | 73.4% / 0.0% |
| IDM | 70.1% / 64.9% | 5.1% / 0.0% |
| EgoMLP | 73.5% / 0.0% | 73.5% / 0.0% |
| PercMLP | 95.4% / 77.3% | 18.6% / 0.0% |
| PercMLP+ | 69.7% / 64.6% | 5.3% / 0.0% |

1. Giá trị thời khoảng cách `T` tối ưu sau khi hiệu chỉnh là T = 1.4 s với khoảng cách Wasserstein 1-D đạt W1 = 0.3504 (rất sát với trung vị headway của người lái trong log là 1.85 s). Xe sau này chân thực hơn hẳn tác nhân không phản ứng (tác nhân câm có W1 = 0.8970).
2. Khi xe sau chỉ phát lại log, IDM bị tính tới 64.9% va chạm từ phía sau (và PercMLP bị tới 77.3%). Đây hoàn toàn là "va chạm giả" (fake collision) do lỗi phương pháp kiểm thử: xe sau trong log đang chạy theo quỹ đạo ghi sẵn và không hề có tri giác để nhận biết rằng xe thử nghiệm (ego) vừa thực hiện phanh gấp khẩn cấp để tránh vật cản phía trước.
3. Khi thay bằng tác nhân xe sau có phản ứng, va chạm phía sau của IDM lập tức giảm từ 64.9% về đúng 0.0%, đưa tổng va chạm của IDM từ 70.1% xuống chỉ còn 5.1% (đúng bằng giới hạn vật lý bất khả thi). Điều này thay đổi hoàn toàn kết luận: nếu dùng xe phát lại, một bộ điều khiển phanh tốt để cứu mạng phía trước sẽ bị coi là tồi tệ do bị đâm sau liên tục, trong khi xe sau phản ứng khôi phục lại đánh giá công bằng và chính xác cho hệ thống.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

1. Mô hình PercMLP bị cổng phát hành chặn đứng vì vi phạm 4 quy tắc ([evidence/gate.json](file:///e:/VinAI/Track4/K4-Track4-DuongVanThanh-02368-Replay-CloseOpenLoop/evidence/gate.json)): R2 trên log (1.0%, cận trên 1.5% > baseline 0.1% + 1.0%), R3 trên lát cắt phanh (4.7%, cận trên 8.8% > baseline 1.1% + 5.0%), R4 trên lưới kịch bản (13.4% > baseline 0.0% + 2.0%), và R6 trong thế giới phản ứng (18.6% > baseline 5.1% + 2.0%). Có hai nguyên nhân gốc: (a) R4 và R6 do dịch chuyển phân phối dữ liệu (covariate shift) vì tập log không hề có các tình huống xe cắt vào bất ngờ ở khoảng cách hẹp; (b) R2 và R3 do sai số tích luỹ (compounding errors) khi mô hình tự lái đi lệch khỏi phân phối trạng thái của người lái và không biết cách tự khôi phục.
2. Tôi sửa bằng 2 kỹ thuật phối hợp: (a) Kịch bản hoá bổ sung (Cut-in Augmentation) sinh 20.000 kịch bản cắt vào mô phỏng (thêm 197.180 cửa sổ); (b) Thuật toán DAgger (Dataset Aggregation) qua 2 vòng: cho mô hình tự lái trên tập log, ghi nhận lại chính các trạng thái sai lệch mà nó đi vào, rồi truy vấn luật chuyên gia IDM để gán nhãn dự đoán hành động sửa sai chuẩn xác. Giải pháp này chữa đúng nguyên nhân gốc vì vừa lấp đầy vùng kịch bản thiếu trong log, vừa dạy mạng nơ-ron cơ chế tự phục hồi khi có sai số nhỏ.
3. Sau khi sửa, PercMLP+ vượt qua toàn bộ 7 quy tắc của cổng phát hành và được RELEASE (va chạm trên log giảm còn 0.2..0.4%, trên lưới kịch bản giảm còn 0.1%). Tuy nhiên, PercMLP+ KHÔNG tốt hơn IDM (va chạm trên log vẫn là 0.2..0.4% so với IDM là 0.1%; trên tình huống kiểm tra cruise speed 20 m/s chưa từng huấn luyện, IDM va chạm 0.0% trong khi PercMLP+ va chạm 0.6..1.2%). PercMLP+ về bản chất là một phiên bản học xấp xỉ lại IDM nên không thể vượt qua chuyên gia của nó. Do đó, CHƯA NÊN phát hành ngay để thay thế IDM; trước khi quyết định cần tiến hành kiểm thử mở rộng trên các phân phối tốc độ cao hơn, thử nghiệm trong môi trường mô phỏng 3D đầy đủ cảm biến, và chạy thử nghiệm bóng (shadow mode) song song với IDM trên xe thật.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Để có cái nhìn khách quan của kỹ sư kiểm thử, bài lab này tồn tại 3 giới hạn nền tảng khiến kết luận an toàn trong phòng thí nghiệm chưa thể suy diễn trực tiếp ra thế giới thực:
1. Mô hình hoá chuyển động 1 chiều dọc thuần tuý (1D longitudinal motion): Mô phỏng giả định mọi xe chạy trên một đường ray thẳng hoàn hảo, bỏ qua chuyển động ngang (lateral dynamics), góc đánh lái, độ trượt lốp, khả năng chuyển làn tránh vật cản hoặc các tương tác tại nút giao. Trong thực tế, phanh khẩn cấp trên đường cong hoặc mặt đường ướt có thể gây mất lái nghiêm trọng.
2. Thiếu vắng mô hình cảm biến thực tế (perception pipeline abstraction): Toàn bộ đầu vào của mô hình là các biến trạng thái hoàn hảo (perfect ground-truth gap, speed, lead_v), chỉ có một phép cộng nhiễu Gauss sơ sài. Trong thực tế, hệ thống phải đối mặt với độ trễ xử lý camera/lidar, nhận diện nhầm vật thể, mất dấu mục tiêu (false negatives/positives) trong điều kiện sương mù, ngược sáng hoặc mưa bão.
3. Giả định tương tác xe dẫn là cố định (non-reactive lead vehicle): Xe dẫn trong log hoàn toàn không nhận biết hay phản ứng với hành vi của xe thử nghiệm. Thế giới giao thông thực tế là một trò chơi tương tác đa tác nhân (multi-agent game) phức tạp, nơi hành động của một xe sẽ làm thay đổi tâm lý và phản xạ của các tài xế xung quanh.

Các bước tiếp theo cần triển khai để thu hẹp khoảng cách Sim-to-Real:
- Tích hợp kiểm thử vào môi trường mô phỏng không gian 3D hoàn chỉnh (như CARLA hoặc AirSim) có dựng mô hình vật lý động lực học xe nhiều bậc tự do và giả lập cảm biến quang học/radar.
- Kiểm thử phần cứng trong vòng lặp (Hardware-In-The-Loop - HIL) để kiểm tra độ trễ bus CAN, hiện tượng quá nhiệt tính toán và sai số chấp hành phanh thủy lực.
- Triển khai chế độ chạy bóng (Shadow Mode) trên đội xe thử nghiệm thực địa: mô hình nơ-ron nhận dữ liệu thực và đưa ra quyết định ảo song song với tài xế an toàn để đo lường độ sai lệch trước khi cấp quyền điều khiển thực tế.
