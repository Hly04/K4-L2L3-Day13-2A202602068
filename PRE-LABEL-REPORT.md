# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Bài tập cá nhân — Nguyễn Thị Hương Ly
- Thành viên: xem `TEAMMATES.md` (Nguyễn Thị Hương Ly - MSSV: 2A202602068).
- Trạng thái: `executed-by-group` (học viên trực tiếp chạy pipeline trên máy cá nhân).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Thị Hương Ly; 02/10/2026; Windows 11 + Docker Desktop (Linux kernel, amd64 / x86_64).
- Image tag và image ID; phiên bản repo: Image tag `day13-pointpillars:lc-20261001-amd64` (Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`); phiên bản repo commit `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: File `demo.pcd` (17.238 điểm, frame_id `demo`, SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`); chạy local offline không dùng mạng container.
- Checkpoint: PointPillars KITTI có sẵn trong image tại `/opt/PointPillars/pretrained/epoch_160.pth` (SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window; score threshold: 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh thứ tư sử dụng hằng số placeholder RGB=0 (bỏ reflectance gốc); $z_{\text{ground}} = 0.075\text{ m}$ được ước lượng từ bình diện mặt đường của dữ liệu PCD.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| :---: | :---: | :---: | :---: | :---: | :--- | :--- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/summary.csv`<br>`run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-demo-delta-0-voxel-0.16.png` | Mô hình chỉ phát hiện duy nhất 1 hộp thuộc lớp `vehicles` tại cự ly gần ($x \approx 8.1\text{ m}$); các đối tượng ở xa bị bỏ sót hoàn toàn do chưa bù trừ độ cao cảm biến. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/summary.csv`<br>`run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-demo-delta-1.73-voxel-0.16.png` | Phát hiện 13 hộp gồm 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`. Các hộp phân bố hợp lý theo dải đường và ôm khít các cụm điểm; đây là baseline chuẩn của KITTI. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/summary.csv`<br>`run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-demo-delta-1.73-voxel-0.32.png` | Số hộp giảm xuống 6 và toàn bộ 6 hộp đều bị gán thành `pedestrian`; mô hình mất hoàn toàn các lớp `vehicles` và `two-wheels` do kích thước pillar quá lớn. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. File `run-A/summary.csv` so với `run-B/summary.csv` và ảnh Side cho thấy ở lượt A các điểm LiDAR bị nâng cao hơn so với phân bố độ cao của tập huấn luyện KITTI, khiến mạng nơ-ron không kích hoạt được các anchor box của vật thể ở xa. Khi đặt $\text{delta} = 1.73\text{ m}$ ở lượt B, dữ liệu được đưa về đúng dải độ cao chuẩn, giúp mô hình nhận diện thêm 12 hộp. Đây là quá trình chạy lại mô hình trên phân bố đầu vào mới, không phải phép tịnh tiến các hộp cũ sau dự đoán. Điều em còn chưa chắc là độ chính xác kích thước của các hộp ở cự ly xa ngoài 40 m do mật độ điểm bị thưa.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. File `run-B/boxes-demo-delta-1.73-voxel-0.16.json` (10 vehicles, 2 pedestrian, 1 two-wheels) so với `run-C/boxes-demo-delta-1.73-voxel-0.32.json` (6 pedestrian). Khi tăng cạnh ô pillar từ 0.16 m lên 0.32 m (diện tích đáy gấp 4 lần), độ phân giải không gian cục bộ bị giảm sút nghiêm trọng, khiến đặc trưng hình học của các phương tiện lớn bị mờ nhòe và mạng phân loại nhầm thành pedestrian. Hoàn toàn không có bằng chứng để kết luận C tốt hơn B, dù thời gian inference của C ngắn hơn.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Ảnh Side là hình chiếu trực giao $X-Z$ toàn cảnh, tích hợp toàn bộ trục $Y$. Các đối tượng có cùng tọa độ $X$ nhưng chạy song song ở các làn $Y$ khác nhau sẽ bị chồng đè lên nhau trên một mặt phẳng. Do đó, góc Side chỉ hỗ trợ kiểm tra cao độ mặt đường và kiểm soát hiện tượng lơ lửng/lún đất, không thể dùng độc lập để xác định góc xoay yaw hay kết luận một đối tượng bị bỏ sót (miss).
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp? Tất cả các file JSON (kể cả lượt B) đều chưa đủ cơ sở để trực tiếp coi là ground truth vì ngưỡng lọc `score-thresh = 0.3` còn chứa các dự đoán tự động chưa qua hậu kiểm. Để làm nhãn chuẩn, cần kiểm tra đối chiếu qua 4 góc nhìn (Top, Side, Front, Góc xoay tự do) trên CVAT kết hợp với ảnh camera cùng frame.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **`case-correct`** | 0 / 13 | 0 m | Không | Bình thường (Baseline) | Dữ liệu đối chiếu gốc từ lượt B; các hộp tiếp xúc đều mặt đường, $mean\_z = 1.034\text{ m}$. |
| **`case-batch-z`** | 13 / 13 (100%) | -1.805 m | Không | DỪNG BATCH | 100% hộp bị lún đồng loạt xuống dưới mặt đất đúng 1.805 m ($mean\_z = -0.771\text{ m}$, file `qc-cases/case-batch-z.json` và `side-batch-z.png`). Lỗi do pipeline quên bước cộng ngược $z_{\text{ground}} + \text{delta}$. |
| **`case-one-box-z`** | 1 / 13 | -1.805 m (chỉ hộp đầu tiên) | Không | KIỂM TỪNG HỘP | 12/13 hộp vẫn ở vị trí chuẩn ($mean\_z = 0.895\text{ m}$, file `qc-cases/case-one-box-z.json`). Pipeline không lỗi; đây là lỗi nhận diện cục bộ ở một đối tượng cụ thể. Sửa thủ công hộp này qua nhiều view. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- **Họ và tên**: Nguyễn Thị Hương Ly (MSSV: 2A202602068)
- **Vai trò thực hiện**: Trực tiếp cấu hình môi trường Docker Desktop, nạp image và vận hành pipeline kiểm thử tự động offline; đọc, trích xuất và phân tích số liệu từ `smoke.json`, các file `summary.csv`, `boxes-*.json` và ảnh `side-*.png`.
- **Quan sát thực nghiệm A/B/C**: Kết quả cho thấy rõ việc thay đổi biểu diễn dữ liệu đầu vào (cả về độ cao sensor delta lẫn kích thước pillar) tác động mang tính quyết định đến chất lượng phát hiện 3D. Đổi delta từ 0 lên 1.73 m giúp phát hiện thêm 12 hộp; trong khi tăng pillar size lên 0.32 m làm mô hình mất hoàn toàn khả năng nhận diện lớp ô tô.
- **Diễn giải phép biến đổi z**:
  + Chiều chuẩn hóa vào mô hình: $z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$ (đưa điểm về hệ tọa độ chuẩn của mạng).
  + Chiều khôi phục ra thế giới thực: $z_{\text{source}} = z_{\text{model}} + z_{\text{ground}} + \text{delta}$ (đưa hộp dự đoán về hệ tọa độ của xe).
- **Quyết định xử lý lỗi**: Nếu gặp hiện tượng cả batch bị lệch một hằng số độ cao giống nhau như `case-batch-z`, hành động đúng là dừng việc chỉnh sửa thủ công và yêu cầu kiểm tra pipeline chuyển đổi; nếu chỉ có một hộp đơn lẻ bị sai như `case-one-box-z`, pipeline vẫn đảm bảo và tiến hành nắn chỉnh hộp đó theo bằng chứng hình học.
- **Điều chưa chắc chắn**: Ở các vùng rìa xa của đám mây điểm (> 40 m), mật độ điểm laser rất thưa, hình dạng hình học của vật thể không còn đủ sắc nét để phân định chắc chắn chiều dài thực của thân xe nếu chỉ dựa trên LiDAR.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:

