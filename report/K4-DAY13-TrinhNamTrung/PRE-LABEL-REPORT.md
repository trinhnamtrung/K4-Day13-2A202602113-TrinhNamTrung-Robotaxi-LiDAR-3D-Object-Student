# Báo cáo thực hành PointPillars - Day 13

Nộp theo hướng dẫn trang lab VLearn Day 13 (repo cá nhân, thư mục `report/K4-DAY13-<TenNhom>/`). Bài làm solo theo yêu
cầu của LC. Đây là kiểm tra formative.

## Nhóm và provenance

- Mã nhóm/phòng: không có, làm solo. Thư mục nộp đặt theo tên cá nhân: `K4-DAY13-TrinhNamTrung`.
- Thành viên: xem `TEAMMATES.md` (1 người).
- Trạng thái: `executed-by-group` (tự chạy solo trên laptop cá nhân, không dùng máy LC, không dùng kết quả có sẵn).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Trịnh Nam Trung, laptop cá nhân. Ngày 02/10/2026, lượt inference
  chạy từ 17:29:55 đến 17:30:34 (+07:00). Windows 11 Home 64-bit, CPU AMD64, Docker Desktop 29.7.2 (WSL2), Linux
  containers x86_64, 8 CPU. Mỗi container chạy với `--cpus 4 --memory 4g --network none`, input và code mount read-only.
- Image tag và image ID; phiên bản repo:
  - Gói `student-prelabel-amd64.zip` (release `student-prelabel-v1`), SHA256
    `f58ca33705fc9c6845ea56a7a5deee9bd2f95b3db2deb32e73f46d82a9737aa9`, khớp `SHA256SUMS.txt`.
  - Image theo `manifest.json` của gói: `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`, tag
    nguồn `day13-pointpillars:lc-20261001-amd64`, linux/amd64.
  - Sau khi `docker load`, image trên máy có ID `sha256:e7b6032b36dfc01b51da2fb29d752942bb51f7e9f5c3fc5d0e73276a7c8e9931`,
    không có tag, tạo ngày 30/09/2026 23:14 (+07:00).
  - Code trong gói: `repo_revision 0831856`; `preannotate.py` sha256 `65edf6ac95926f79...`; `pipeline-qc-cases.py` sha256
    `c177fc008f79223e...`. Repo cá nhân ở commit `e226b93`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint: `input/demo.pcd` trong gói Student, `frame_id = demo`. KITTI
  000008 (bản demo của MMDetection3D), 17.238 điểm, giấy phép CC BY-NC-SA 3.0; x/y giữ nguyên, z dịch +1,73 m, bỏ
  reflectance thật, RGB = 0. PCD sha256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`. Chạy trên
  laptop cá nhân theo hướng dẫn gói Student, không dùng dữ liệu Robotaxi.
- Checkpoint: PointPillars KITTI có sẵn trong image, `/opt/PointPillars/pretrained/epoch_160.pth`, sha256
  `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window theo preset KITTI, `range = (0, -39.68, -3, 69.12, 39.68, 1)` m trong hệ model. Chỉ chạy lượt
  `identity`, không thêm `--full-scene`, log ghi `rear=0` ở cả ba lượt. Score threshold 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: PCD không có reflectance thật. Script đọc cloud hai lần với reflectance
  hằng: 0.0 để lấy `vehicles`, 0.7 để lấy `pedestrian` và `two-wheels`; người/xe hai bánh có tâm nằm trong hộp xe thì bị
  bỏ. Đây không phải intensity được phục hồi. `z_ground = 0.075 m`, ước lượng từ đỉnh histogram z của PCD (ngăn 0,05 m),
  giống nhau ở cả ba lượt.

Ghi chú về cách chạy:

1. Lệnh `student-bundle.py run --bundle . --out ..\ket-qua-nhom-01` dừng sau bước nạp image. Trong
   `ket-qua-nhom-01/smoke.json`, bước `docker-load` là `passed` nhưng `status` là `failed`, lỗi `docker image inspect
   sha256:e03983bd...` không tìm thấy image (log trong `run-log.txt`).
2. Nguyên nhân: Docker trên máy dùng image store classic (overlay2) nên đặt ID theo digest config `e7b6032b...`, còn
   manifest của gói ghi digest OCI index `e03983bd...`. Mở `manifest.json` bên trong `image.tar.gz` thấy index `e03983bd...`
   có config là `e7b6032b...`, tức vẫn là cùng một image.
3. Vì vậy em chạy lại bằng các lệnh thủ công ở PRE-LABEL mục 2 và mục 4, dùng đúng image này và đúng tham số như runner
   (`--entrypoint python`, script `/practice/preannotate.py` và `/practice/pipeline-qc-cases.py` của gói, `--from KITTI`,
   `--score-thresh 0.3`, delta/pillar như bảng dưới). Output ở `ket-qua-manual-01/`, log `run-manual-log.txt`, cả bốn lệnh
   `exit=0`. Thư mục `ket-qua-nhom-01/` để nguyên làm bằng chứng lỗi.
4. Em kiểm lại những điều runner kiểm: mỗi lượt đủ JSON, ảnh Side và CSV; `frame_id = demo`, `dataset = KITTI`, `delta` và
   `voxel_size` đúng từng lượt; ba ca QC đều có `training_only: true` và trỏ đúng sha256 của JSON B
   (`2ffb4e85d1a8746b1f290f6704204bf57e5521ae3df98ea81473a087a066fcfc`). Lượt chạy này không có `smoke.json` báo passed.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Số hộp lấy từ `n_boxes`, mean_z lấy từ `mean_z` trong `run-A/B/C/summary.csv`. `mean_z`
không phải điểm chất lượng. Đáy hộp tính bằng `z - height/2` từ JSON.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `side-demo-delta-0-voxel-0.16.png`, `summary.csv` | Chỉ có 1 `vehicles` ở (13.15, -0.45), score 0.32, sát ngưỡng 0.3. Tâm z 0.33, cao 1.46 nên đáy khoảng -0.40 m. Trên ảnh Side, hộp này ở x khoảng 11-15 m và cắm xuống dưới đường z = 0, trong khi điểm mặt đất quanh đó ở khoảng 0 |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `side-demo-delta-1.73-voxel-0.16.png`, `summary.csv` | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`, score 0.32-0.93, hộp trải từ x 3.7 đến 55.6 m. Các xe gần (x < 26 m, trừ #8) có đáy từ -0.05 đến 0.15 m. Ba xe xa #3, #6, #9 (x 33-56 m) có đáy 0.32-0.46 m. Hộp #8 `vehicles` ở (9.38, 4.23) có đáy khoảng 0.65 m và chồng với #10 `two-wheels` ở (10.32, 5.25) trên ảnh Side, vùng x 9-11 m |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `side-demo-delta-1.73-voxel-0.32.png`, `summary.csv` | 6 `pedestrian`, không còn `vehicles`, score 0.30-0.81. Bốn hộp C nằm cách một xe hoặc xe hai bánh của B không quá 1.53 m theo x-y: C#0 với B#5, C#2 với B#0, C#4 với B#1, C#5 với B#10 |

- A/B, chỉ đổi delta: A có 1 hộp, B có 13 hộp. Ảnh Side của A chỉ có một hộp ở x 11-15 m với đáy dưới z = 0, còn ảnh Side
  của B có 13 hộp và phần lớn đáy sát mặt đất. Hộp A (13.15, -0.45, z 0.33) gần nhất với hộp B#1 (14.77, -1.08, z 0.90),
  cách 1.73 m theo x-y; tâm z chỉ chênh 0.57 m chứ không phải 1.73 m, score 0.32 so với 0.93. `mean_z` chênh 0.70 m (0.330
  và 1.034). Như vậy đây là model chạy lại trên input khác, không phải B dịch hộp cũ của A lên 1.73 m. Điều em chưa chắc:
  hộp A và hộp B#1 có phải cùng một xe không, vì ảnh Side chồng các vật ở y khác nhau và gói không có ảnh camera; cần xem
  Top view mới xác định được.
- B/C, chỉ đổi pillar: B có 13 hộp, C có 6 hộp. Ảnh Side của C chỉ còn các hộp hẹp màu cam (`pedestrian`), không còn hộp
  `vehicles`. Số lượng giảm từ 13 xuống 6, lớp đổi từ 10 xe + 2 người + 1 xe hai bánh thành 6 người. Về vị trí, bốn hộp C
  nằm gần chỗ B có xe hoặc xe hai bánh (C#4 cách B#1 1.53 m, C#2 cách B#0 1.29 m, C#0 cách B#5 1.47 m, C#5 cách B#10 0.35
  m). Checkpoint được train với pillar 0.16 m, đổi sang 0.32 m làm input khác với lúc train nên model có thể vẽ người ở chỗ
  B thấy xe. Không đủ bằng chứng để kết luận cấu hình nào tốt hơn: không có nhãn đúng, ảnh Side không cho thấy hình dạng
  từng vật, và nhiều hộp hơn hay score cao hơn không chứng minh là đúng hơn.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw:
  - ROI chỉ phía trước (x 0-69.12 m, |y| <= 39.68 m, `rear=0`), nên vật ở phía sau hoặc ngoài cửa sổ không được xét, không
    thể dùng để nói model bỏ sót.
  - Ảnh Side là hình chiếu x-z của cả scene, các xe ở y khác nhau chồng lên nhau nên không đếm chính xác số vật bị bỏ sót.
  - Yaw nằm trên mặt phẳng x-y nên không đọc được trên ảnh Side. Trong B, yaw của 10 xe chia hai nhóm: 5 xe từ 2.69 đến
    2.91 rad và 5 xe từ -0.36 đến -0.26 rad, lệch nhau gần pi: cùng trục dọc nhưng ngược đầu. Có thể là hai chiều xe chạy hoặc bị lật
    đầu 180 độ; cần Top view và camera để biết.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?
  - Không JSON nào trong bài được import, vì đây là KITTI demo, khác frame Robotaxi.
  - A: hộp duy nhất có đáy -0.40 m và score sát ngưỡng, chưa đủ cơ sở.
  - C: cả lớp đổi sang `pedestrian` ở chỗ B thấy xe, chưa đủ cơ sở.
  - B: cần kiểm thêm #8 (đáy 0.65 m, chồng với #10), #10 `two-wheels`, hai `pedestrian` score thấp #11 (0.34) và #12
    (0.32), các xe xa #3/#6/#9 có đáy 0.32-0.46 m so với mặt đường cục bộ, và hướng đầu xe. Cần Top/Front view và camera
    cùng frame.

## Phép đổi z

```text
Xuôi, trước model:  z_model  = z_source - z_ground - delta
Ngược, sau model:   z_source = z_model  + z_ground + delta
```

Với `z_ground = 0.075 m` của PCD này, một điểm mặt đất (z_source khoảng 0.075) khi vào model sẽ nằm ở:

| Lượt | z_model của điểm mặt đất | Ý nghĩa |
| --- | --- | --- |
| A (delta = 0) | 0.075 - 0.075 - 0 = 0 | Model thấy mặt đường ở 0, trong khi checkpoint KITTI quen mặt đường ở khoảng -1.73 |
| B (delta = 1.73) | 0.075 - 0.075 - 1.73 = -1.73 | Đúng chỗ checkpoint quen |

- Đổi delta trước inference là đổi cloud mà model nhìn thấy, nên model chạy lại và ra tập hộp khác. Vì vậy A và B khác cả
  số hộp, lớp, vị trí và score, không có quan hệ "lệch đúng 1.73 m".
- Dịch hộp sau inference chỉ cộng một hằng số vào z của các hộp đã có, model không chạy lại. Ví dụ hộp B#0: model ra
  z = -0.884 m, đổi ngược thành -0.884 + 0.075 + 1.73 = 0.921 m, đúng giá trị trong JSON B. Nếu quên bước ngược, hộp #0 sẽ
  nằm ở -0.884 m thay vì 0.921 m và mọi hộp cùng thấp đi 1.805 m, giống ca `case-batch-z` bên dưới.
- JSON do script xuất đã ở hệ nguồn, khi đọc không cộng thêm delta.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | 0 m | Không, class, x, y, L/W/H, yaw, score giống hệt B | Phép đổi z nhất quán, chuyển sang kiểm từng hộp; không có nghĩa các cuboid đã đúng | `qc-cases/case-correct.json`, `side-correct.png`, so với JSON B |
| case-batch-z | 13/13 | -1.805 m ở mọi hộp (bằng z_ground 0.075 + delta 1.73) | Không | Dừng sửa tay, báo LC kiểm phép đổi ngược z và yêu cầu tạo lại prediction | `qc-cases/case-batch-z.json`, `side-batch-z.png`: cả 13 hộp chìm dưới z = 0, đáy từ -1.15 đến -1.85 m; `manifest.json` ghi `height_offset_m = 1.805` |
| case-one-box-z | 1/13 (hộp #0 `vehicles` ở (8.09, 1.21)) | -1.805 m, chỉ ở hộp #0 | Không | Kiểm riêng hộp #0 qua nhiều góc nhìn, không kết luận pipeline sai | `qc-cases/case-one-box-z.json`, `side-one-box-z.png`: một hộp ở x 6-10 m chìm xuống (đáy -1.65 m), 12 hộp còn lại giữ như B |

Ba ca này do helper tạo từ prediction B (`training_only: true`, cùng sha256 với B), không phải kết quả inference riêng và
không phải nhãn đúng. Ca `case-correct` chỉ giữ nguyên phép đổi z, bản thân các hộp vẫn có thể sai class hoặc hình học.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược;
một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

### Trịnh Nam Trung (2A202602113)

- Vai trò: làm solo nên em kiêm cả bốn vai: tải gói và kiểm checksum, chạy lệnh, kiểm cấu hình trong CSV/JSON, xem hình
  học trên ảnh Side và ghi log (`run-log.txt`, `run-manual-log.txt`). Khi runner dừng ở bước kiểm image ID, em đối chiếu
  `manifest.json` trong `image.tar.gz` để xác nhận vẫn đúng image rồi mới chạy lại bằng lệnh thủ công của PRE-LABEL.
- Quan sát A/B/C: điều em thấy rõ nhất là ở lượt C. Chỉ đổi pillar từ 0.16 lên 0.32 mà `run-C/summary.csv` không còn xe
  nào (6 hộp đều là `pedestrian`). Trong 10 xe của B, chỉ 4 xe (#0, #1, #5, #8) có một hộp C ở gần, cách không quá 1.53
  m, và hộp đó đều là `pedestrian`; 6 xe còn lại không có hộp C nào trong vòng 5 m. Trên
  `side-demo-delta-1.73-voxel-0.32.png` chỉ còn các hộp hẹp màu cam. Như vậy đổi pillar vừa làm mất phần lớn xe, vừa làm
  đổi lớp ở một số vị trí.
- Phép z thuận/ngược: chiều xuôi trừ `z_ground + delta` để đưa cloud về đúng chỗ checkpoint KITTI quen (mặt đường ở khoảng
  -1.73), chiều ngược cộng lại lượng đó để hộp quay về hệ của PCD gốc. Đổi delta ở chiều xuôi là đổi input nên model chạy ra
  kết quả khác hẳn (A 1 hộp, B 13 hộp, tâm z không lệch đúng 1.73 m); còn dịch hộp sau model thì mọi hộp chỉ cùng tăng
  hoặc giảm một hằng số. Ví dụ hộp B#0: z_model -0.884 m, đổi ngược ra 0.921 m.
- Quyết định lỗi batch và hành động: với `case-batch-z`, cả 13/13 hộp cùng thấp đi 1.805 m, đúng bằng `z_ground + delta`,
  còn class, x, y, kích thước và yaw giữ nguyên. Đây là dấu hiệu quên phép đổi ngược ở pipeline, nên em sẽ dừng, không kéo
  tay từng hộp, báo LC kèm file/job để kiểm transform và tạo lại prediction. Khi sửa job Robotaxi, nếu mở ra thấy mọi hộp
  cùng nổi hoặc chìm một lượng giống nhau thì em cũng xử lý như vậy.
- Điều chưa chắc: hộp B#8 (`vehicles` ở 9.38, 4.23) có đáy khoảng 0.65 m và chồng với `two-wheels` B#10. Em chưa biết đó
  là xe đỗ trên chỗ cao hơn mặt đường, hay là hộp thừa trên tường/vật bên lề, vì ảnh Side chồng các vật khác y và gói không
  có camera. Các xe xa #3, #6, #9 có đáy 0.32-0.46 m cũng có thể do mặt đường xa cao hơn, vì `z_ground` chỉ là một giá trị
  cho cả scene. Cần Top view và camera cùng frame để kết luận.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
