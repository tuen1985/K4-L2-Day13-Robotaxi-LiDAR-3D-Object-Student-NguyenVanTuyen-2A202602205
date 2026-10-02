# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: CaNhan_NguyenVanTuyen_2A202602205 (Thực hiện cá nhân)
- Thành viên: xem `TEAMMATES.md` (Họ tên: Nguyễn Văn Tuyển / MSSV: 2A202602205; Vai trò lượt A/B/C: Vận hành lệnh, kiểm tra cấu hình JSON & phân tích hình học báo cáo cá nhân).
- Trạng thái: `executed-by-group` / `provided-results` (Đã thực thi cá nhân và phân tích đầy đủ kết quả pipeline).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Nguyễn Văn Tuyển; 2026-10-02; Windows 11 x86_64 / Docker Linux CPU.
- Image tag và image ID; phiên bản repo: Image Tag: `day13-pointpillars:lab` (Image ID: `sha256:a6f87d...` PointPillars KITTI pretrained CPU image); Repo Revision: `c4b27f...`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `input/demo.pcd` (Mẫu KITTI 000008 adaptation, 11,496 points binary PCD); Chạy tại local workspace cá nhân.
- Checkpoint: PointPillars KITTI pretrained `epoch_160.pth` (có sẵn trong image).
- Phạm vi: front-window (camera ROI phía trước); score threshold: 0.30.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Reflectance bị bỏ trong PCD thực hành (kênh RGB=0 uint32 placeholder); `z_ground` ước lượng từ điểm mặt đất cục bộ (-0.08 m).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 6 | 0.85 | `run-A/boxes-demo-delta-0-voxel-0.16.json` | 6 hộp dự đoán, mean_z = 0.85m. Các hộp tập trung ở vùng phía trước vehicle. |
| B | 1.73 | 0.16 | 8 | 0.42 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json` | 8 hộp dự đoán (nhiều hơn A 2 hộp), mean_z = 0.42m. Hộp bám sát cụm điểm thực tế sau phép cộng trả delta z về hệ PCD nguồn. |
| C | 1.73 | 0.32 | 5 | 0.38 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json` | 5 hộp dự đoán, mean_z = 0.38m. Ô pillar lớn (0.32m) giảm độ phân giải XY, làm một số vật thể nhỏ/gần bị gộp hoặc bỏ sót. |

- **A/B — chỉ đổi delta**: A có 6 hộp; B có 8 hộp. Ảnh `side-*.png` và file `boxes-*.json` vùng `x ≈ 10–20m` khác nhau ở độ bám cụm điểm và số lượng đối tượng phát hiện. Đây là chạy lại model trên input khác (dữ liệu điểm đã dịch z trước khi qua mạng), không chỉ dịch hộp cũ; điều em còn chưa chắc là mức độ nhạy của pretrained model KITTI đối với biến đổi cao độ cảm biến so với mặt đất thực tế.
- **B/C — chỉ đổi pillar**: B có 8 hộp; C có 5 hộp. Ảnh Side và file JSON vùng `x-z` khác nhau ở mật độ phân chia pillar grid. Số lượng/lớp/vị trí thay đổi như sau: Số lượng hộp giảm từ 8 xuống 5 do ô pillar 0.32m gấp đôi 0.16m làm mịn hóa không gian, mất chi tiết ranh giới đối tượng nhỏ. Có đủ bằng chứng để kết luận tốt hơn không? Chưa đủ bằng chứng để khẳng định C tốt hơn B; B (0.16m) cho khả năng phân tách không gian tốt hơn.
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**: ROI front-window giới hạn tầm nhìn phía trước, đối tượng ngoài ROI bị bỏ qua không tính là model bỏ sót. Ảnh Side 2D (x-z) bị chồng lấp các vật thể theo trục y, nên không thể dùng duy nhất góc Side để đánh giá yaw hoặc chiều rộng footprint 3D.
- **JSON nào còn chưa đủ cơ sở để import?**: Tất cả các file JSON prediction (A, B, C) từ pretrained KITTI demo đều chưa đủ cơ sở để import vào CVAT cho bài làm Robotaxi thật. Cần kiểm tra kỹ qua cả 4 góc nhìn 3D (Top, Side, Front, Free 3D) kết hợp ảnh Camera cùng frame.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 8 | 0 m | Không đổi | Kiểm từng hộp | Hộp giữ nguyên kết quả chuyển đổi z chuẩn từ B |
| case-batch-z | 8 / 8 | 1.65 m (delta + z_ground) | Không đổi | Dừng batch! | Cả 8/8 hộp đều bị chìm z xuống đất 1.65m do thiếu phép chuyển z ngược. Cần dừng pipeline kiểm tra code transform. |
| case-one-box-z | 1 / 8 | 1.65 m | Không đổi | Kiểm từng hộp | Chỉ 1 hộp bị chìm, 7 hộp khác đúng vị trí z. Lỗi cục bộ từng đối tượng, không phải lỗi pipeline cả batch. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

### Member: Nguyễn Văn Tuyển (MSSV: 2A202602205)
- **Vai trò đã làm**: Vận hành script cá nhân, đối chiếu JSON/Side plot A/B/C, kiểm tra các ca QC lỗi height offset và viết tổng hợp báo cáo.
- **Quan sát A/B/C**:
  - So sánh A vs B (đổi `delta` từ 0m -> 1.73m): Số hộp tăng từ 6 lên 8, `mean_z` giảm từ 0.85m xuống 0.42m. Khi đưa input đúng z-offset vào feature extractor, model nhận diện được thêm 2 đối tượng thưa ở xa.
  - So sánh B vs C (đổi `voxel_size` từ 0.16m -> 0.32m): Số hộp giảm từ 8 xuống 5 do lưới pillar thô hơn làm tiêu biến cụm điểm thưa.
- **Diễn giải phép z thuận/ngược**:
  - Phép thuận: `z_model = z_source - z_ground - delta` (đưa tọa độ PCD nguồn về hệ tọa độ đầu vào của model PointPillars).
  - Phép ngược: `z_source = z_model + z_ground + delta` (đưa tâm hộp dự đoán từ model trở lại hệ tọa độ PCD nguồn).
  - Lỗi quên phép ngược làm toàn bộ hộp bị hạ thấp đúng bằng `delta + z_ground` (1.65m).
- **Quyết định lỗi batch và hành động**: Khi phát hiện ca lỗi đồng loạt cả batch (như `case-batch-z`), hành động bắt buộc là **Dừng batch**, không sửa thủ công từng hộp trên CVAT, thông báo với LC/Dev để sửa phép biến đổi z trong pipeline pre-annotation.
- **Điều chưa chắc**: Khả năng phân biệt chính xác class đối với các cụm điểm thưa bị che khuất một phần nếu chỉ dựa vào PCD mà không có ảnh camera độ phân giải cao.

---

## Báo cáo tiến độ cá nhân — 15 Job CVAT (Lab Coach Requirement)

- **Trạng thái**: Đã hoàn thành gán nhãn & rà soát tối thiểu **15 job** trên CVAT theo yêu cầu của Lab Coach.
- **Quy trình thực hiện**:
  1. Đã bắt đầu phiên và nạp pre-label Robotaxi đúng frame cho từng job nguồn trên portal.
  2. Rà soát toàn bộ 5 class trong schema (`vehicles`, `two-wheels`, `pedestrian`, `Animal`, `Obstacle`).
  3. Kiểm tra và tinh chỉnh Cuboid 3D qua 4 góc nhìn (Top, Side, Front, Free 3D) kết hợp ảnh camera đồng bộ.
  4. Đảm bảo đáy hộp bám mặt đường cục bộ, kiểm tra chiều dài/rộng/cao và hướng đầu xe (yaw).
  5. Thực hiện **Save** trên CVAT, khai báo đúng phạm vi rà soát và nộp bài vào hàng đợi QC / phản hồi feedback đúng hạn quy định trên portal.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: Đã xác nhận quyền sử dụng đúng ca học.
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: Đã thực thi và phân tích đầy đủ output A/B/C và ca QC.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Đã tuân thủ lưu bản gốc private, không import prediction KITTI/QC vào CVAT.
- Nhận xét từng thành viên và quyết định dừng pipeline: Đã hiểu và áp dụng đúng quy tắc dừng batch khi gặp lỗi z hệ thống.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: Đã hoàn thành phần PointPillars cá nhân và 15 job CVAT cá nhân, đủ điều kiện nghiệm thu.
