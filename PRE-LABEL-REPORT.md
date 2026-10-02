# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: K4-L2L3-Day13-Robotaxi
- Thành viên: (Nhữ Đình Chiến - MSSV: 2A202602130, tham gia đủ các lượt A, B, C).
- Trạng thái: `executed-by-group` (đã chạy Docker container thực tế qua runner `student-bundle.py` trên máy local, xác thực `smoke.json` status `passed`).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nhữ Đình Chiến; 2026-10-02 20:05 (UTC+7); Windows 11 x86_64, Docker Desktop (WSL2 engine Linux amd64, 4 CPU, 4GB RAM limit).
- Image tag và image ID; phiên bản repo:
  - Image tag: `day13-pointpillars:lc-20261001-amd64`
  - Image ID: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`
  - Phiên bản repo: `0831856d921609312d42c7582c366e5a311bb7b1` (HEAD: `e226b93`)
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp:
  - File: `input/demo.pcd` (frame_id: `demo`, 17,238 điểm)
  - Dataset gốc: KITTI Vision Benchmark Suite / MMDetection3D demo 000008 (giấy phép CC BY-NC-SA 3.0)
  - PCD SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
  - Nơi được phép chạy: Thư mục thực hành local máy học viên / không public dữ liệu.
- Checkpoint: PointPillars KITTI có sẵn trong image:
  - Checkpoint path: `/opt/PointPillars/pretrained/epoch_160.pth`
  - Checkpoint SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window (`xmin=0.0, ymin=-39.68, zmin=-3.0, xmax=69.12, ymax=39.68, zmax=1.0`); score threshold: `0.3`
- Giả định kênh thứ tư/intensity và nguồn z_ground:
  - Kênh thứ 4: PCD gốc đã lược bỏ reflectance thật và gán RGB=0 placeholder; script dùng adapter hằng số 2 lượt đọc (0.0 cho class `vehicles`, 0.7 cho `pedestrian` và `two-wheels`).
  - Nguồn z_ground: Ước lượng tự động từ đỉnh histogram cao độ z của đám mây điểm (`bin_m=0.05`), thu được `z_ground = 0.075 m`.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-demo-delta-0-voxel-0.16.png`<br>`run-A/summary.csv` | Model bị miss gần như toàn bộ xe; chỉ bắt được duy nhất 1 hộp `vehicles` tại x=13.15m, y=-0.45m, score 0.32 (vừa chạm ngưỡng 0.30). Điểm mây bị nâng quá cao so với phân bố độ cao sensor lúc train của KITTI. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-demo-delta-1.73-voxel-0.16.png`<br>`run-B/summary.csv` | Bắt được 13 hộp gồm 10 `vehicles`, 1 `two-wheels` (x=10.32m, y=5.25m, score 0.38) và 2 `pedestrian`. Các xe ở gần được nhận diện với score rất cao (0.81 - 0.93), bám khít chùm điểm trên ảnh Side. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-demo-delta-1.73-voxel-0.32.png`<br>`run-C/summary.csv` | Số hộp giảm xuống 6, nhưng toàn bộ 6 hộp đều bị gán nhãn sai thành `pedestrian` (0 `vehicles`, 0 `two-wheels`). Pillar bị mở rộng làm vỡ đặc trưng hình học của phương tiện. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh/file/vùng `side-demo-delta-*.png` khác ở toàn bộ dải x từ 3.7m đến 55.6m: B bám sát các chùm điểm xe dọc đường đi, trong khi A bỏ sót toàn bộ chùm điểm này. Đây là chạy lại model trên input khác (z đưa vào mạng đã được trừ đúng chiều cao sensor 1.73m trước khi voxel hóa), không chỉ dịch hộp cũ (nếu dịch hộp cũ thì số lượng hộp và kích thước phải giữ nguyên, chỉ đổi tâm z; nhưng ở đây cấu trúc và số lượng phát hiện thay đổi hoàn toàn từ 1 lên 13 hộp); điều em còn chưa chắc là độ chính xác hướng đầu xe (yaw) của các xe ở xa (x > 30m) vì mật độ điểm thưa dần.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh/file/vùng `side-*.png` và JSON khác ở chỗ: toàn bộ 10 xe `vehicles` và 1 xe `two-wheels` ở B đều biến mất ở C; C chỉ phát hiện 6 hộp và tất cả đều bị phân loại là `pedestrian` (ví dụ vùng x=19.43m, x=13.15m, x=9.11m). Số lượng/lớp/vị trí thay đổi như sau: số hộp giảm từ 13 xuống 6, nhãn bị thoái hóa phân loại (classification collapse) sang `pedestrian`. Có đủ bằng chứng để kết luận tốt hơn không? Có đủ bằng chứng để kết luận B tốt hơn hẳn C trên checkpoint này, vì checkpoint được train tối ưu cho kích thước pillar 0.16m; khi tăng gấp đôi kích thước pillar lên 0.32m mà không train lại mạng thì đặc trưng không gian bị gộp thô, làm sai lệch hoàn toàn anchor và head phân loại xe.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?
  - Giới hạn ROI: Checkpoint KITTI chỉ quan sát cửa sổ phía trước (front-window: x > 0, y ∈ [-39.68, 39.68]). Các vật thể ở phía sau xe (x < 0) nằm ngoài vùng quét ROI nên không thể coi là model bỏ sót trong thí nghiệm này.
  - Góc Side (hình chiếu X-Z): Giúp kiểm tra trực quan chiều dài X và cao độ Z so với mặt đất cục bộ, nhưng bị chồng lấp (chập) các đối tượng có cùng tọa độ X nhưng khác tọa độ Y (bên trái/phải làn). Đặc biệt, góc Side hoàn toàn KHÔNG THỂ đọc được hướng đầu xe (yaw) và độ rộng Y; do đó bắt buộc phải phối hợp góc Trên (Top/BEV) và ảnh camera RGB để xác định đúng góc quay và tránh lỗi đảo ngược 180°.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?
  - Cả ba file JSON (A, B, C) đều KHÔNG ĐƯỢC PHÉP import vào CVAT của học viên vì đây là dữ liệu demo KITTI học thuật, khác hoàn toàn frame của Robotaxi trong 30 job được giao.
  - Về mặt thuật toán: JSON A thiếu hầu hết đối tượng (chỉ có 1 hộp); JSON C sai hoàn toàn class (toàn bộ là `pedestrian`); JSON B dù nhận diện tốt nhất nhưng vẫn chỉ là pre-label sơ bộ: cần kiểm tra lại hướng yaw (nguy cơ lệch 180°), ranh giới đáy so với mặt đường cục bộ, kiểm tra che khuất và tìm các đối tượng thiếu thuộc các class khác (`Animal`, `Obstacle`) bằng camera.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 (0%) | 0 m | Không đổi | Pipeline chuyển đổi đúng, không có lỗi batch | File `case-correct.json` giữ nguyên 100% tọa độ và nhãn từ prediction B; ảnh `side-correct.png` có đáy các hộp nằm chuẩn trên mặt đường. |
| case-batch-z | 13 / 13 (100%) | -1.805 m (bị chìm xuống dưới đất) | Không đổi (x, y, yaw, size, class giữ nguyên) | **Dừng batch**, báo LC kiểm tra pipeline chuyển hệ tọa độ | Toàn bộ 13/13 hộp đều bị trừ đúng lượng `delta + z_ground = 1.73 + 0.075 = 1.805 m` trong `case-batch-z.json`. Ảnh `side-batch-z.png` cho thấy tất cả các hộp bị chìm hẳn xuống dưới vạch tham chiếu z=0. Đây là lỗi quên phép biến đổi z ngược, KHÔNG ĐƯỢC sửa tay từng hộp. |
| case-one-box-z | 1 / 13 (7.7%) | -1.805 m (chỉ duy nhất hộp đầu tiên tại index 0) | Không đổi | **Kiểm từng hộp**, không dừng batch | Trong `case-one-box-z.json`, chỉ duy nhất hộp index 0 (xe tại x=8.09m) bị lệch z=-0.885m (lệch -1.805m so với 0.92m gốc), còn lại 12 hộp khác giữ nguyên cao độ đúng. Đây là lỗi cục bộ của một đối tượng đơn lẻ hoặc do địa hình mấp mô, cần dùng nhiều view để chỉnh riêng hộp đó. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

- **Thành viên: Nhữ Đình Chiến (MSSV: 2A202602130)**
  - **Vai trò đã làm:** Vận hành runner `student-bundle.py` trên môi trường Docker CPU; kiểm tra tính toàn vẹn manifest/hashes; đối chiếu số liệu trích xuất từ `summary.csv`, file JSON tọa độ và ảnh Side của 3 lượt A/B/C; phân tích nguyên nhân lỗi trong các ca QC kiểm soát.
  - **Một quan sát A/B/C có dẫn chứng file/vùng:** Khi so sánh `run-B/boxes-demo-delta-1.73-voxel-0.16.json` và `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, ở lượt B có 10 xe `vehicles` phân bố từ x=3.7m đến x=55.58m và 1 xe `two-wheels` tại x=10.32m, y=5.25m. Nhưng khi chuyển sang lượt C (tăng voxel size lên 0.32m), toàn bộ các hộp xe hơi và xe hai bánh biến mất, thay vào đó model chỉ tạo ra 6 hộp và đều bị dán nhãn nhầm thành `pedestrian`. Điều này chứng minh việc thay đổi kích thước pillar ảnh hưởng trực tiếp đến kích thước feature map và khả năng kích hoạt của các anchor head đã pretrained.
  - **Diễn giải phép biến đổi z thuận/ngược:**
    - Chiều thuận (trước khi đưa vào model): $z_{model} = z_{source} - z_{ground} - delta$. Cần trừ đi $z_{ground}$ (mặt đất ước lượng) và $delta$ (chiều cao sensor giả định của checkpoint KITTI là 1.73m) để đưa đám mây điểm về đúng hệ trục tọa độ mà mô hình PointPillars được huấn luyện.
    - Chiều ngược (sau khi model dự đoán): $z_{source} = z_{model} + z_{ground} + delta$. Sau khi model trả về tọa độ hộp, bắt buộc phải cộng bù lại $(z_{ground} + delta)$ để đưa bounding box trở lại hệ tọa độ thực tế của scan LiDAR. Nếu quên bước này, toàn bộ batch sẽ bị chìm xuống đất 1.805m như trong `case-batch-z`.
  - **Quyết định lỗi batch và hành động:** Khi kiểm tra thấy 100% số hộp bị lệch cao độ z cùng một giá trị cố định (1.805m) trong khi x, y, yaw và class không đổi, em quyết định **dừng ngay pipeline**, không chỉnh sửa thủ công trên CVAT, báo ngay cho LC/kỹ thuật để sửa công thức chuyển đổi ngược trong script nạp pre-label. Ngược lại, nếu chỉ có 1 hộp bị lệch còn các hộp khác chuẩn thì mới tiến hành kiểm tra đa góc nhìn để chỉnh sửa riêng hộp đó.
  - **Điều chưa chắc:** Chưa chắc chắn về hướng yaw chính xác của xe máy (`two-wheels` tại x=10.32m) nếu chỉ dựa vào ảnh chiếu Side LiDAR, vì chùm điểm xe máy khá thưa; cần phải đối chiếu thêm góc nhìn Trên (Top view) và ảnh chụp camera cùng frame trên CVAT.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:

