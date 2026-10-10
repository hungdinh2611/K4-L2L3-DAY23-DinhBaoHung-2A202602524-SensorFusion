# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Đinh Bảo Hưng
- MSSV: 2A202602524
- Email:
- Link repo (fork): https://github.com/hungdinh2611/K4-L2L3-DAY23-DinhBaoHung-2A202602524-SensorFusion
- Commit hash nộp (`git rev-parse HEAD`): Lấy sau commit cuối và nộp trên LMS.

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: `compare`, 0–198 (199 frames), `training_segment-1005081002024129653_5313_150_5333_150_with_camera_labels.tfrecord`, seed 0.
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: 0.9701, 0.7004, 519/16/222.
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.150323 m, 502, 11.343651 m², 0, 239, 2.522613.
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: 0.135867 m, 502, 9.266811 m², 0, 239, 2.522613.
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: Fused giảm RMSE 0.014456 m so với LiDAR; cả hai có 502 matches, 0 ghost và 239 missed GT frames. Theo 519 detection TP, coverage là 502/519 ≈ 0.967; precision_track là 502/(502+0) = 1. Camera trong lab dùng tâm hộp FRONT ground-truth thêm nhiễu seeded, nên kết quả không đánh giá detector ảnh thực tế.

Chạy từ root repo:

```bash
fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0
```

`rmse = sqrt(sum_sq_err/matches)` trên vị trí 3D của confirmed tracks ghép
một-một với GT xe trong cửa sổ BEV, gate XY **2.0 m**; `null` nếu không có cặp.
Camera dùng tâm hộp 2D ground-truth FRONT có nhiễu seeded, **không** dùng camera
detector. Kết quả này không đo hiệu quả một perception system độc lập với GT.

`grade_run.log` là JSONL, mỗi `(mode,frame)` đúng một record với các trường:
`mode`, `frame`, `det_tp`, `det_fp`, `det_fn`, `valid_gt`, `confirmed`, `matches`,
`sum_sq_err`, `ghosts`, `misses`. Đảm bảo `matches+ghosts==confirmed` và
`matches+misses==valid_gt`; tổng/trung bình record phải khớp `metrics.json`.
File per-mode `metrics_lidar.json`, `metrics_fused.json`, `grade_run_lidar.log`,
`grade_run_fused.log` được giữ để đối chiếu.

## Giải thích ngắn (Parts E–H — tự viết)

1. LiDAR đo tâm xe 3D (`z` có 3 phần tử); EKF dùng các hàng vị trí của H và covariance R theo mét. Camera đo pixel 2D (`u, v`); H là Jacobian chiếu pinhole và R là covariance theo pixel. Cả hai cập nhật cùng trạng thái 6D trong `student/workspace/kalman.py`.
2. Mahalanobis chuẩn hoá sai số theo độ bất định của track và measurement. Cổng chi-square theo số chiều đo loại cặp không phù hợp trước greedy matching, thay vì coi mọi mét hoặc pixel có cùng ý nghĩa như khoảng cách Euclid.
3. Đây là track-then-fuse: một danh sách track được predict một lần, sau đó association/update LiDAR chạy trước camera trên cùng track. Camera chỉ hiệu chỉnh trạng thái; thứ tự được triển khai trong runner `fusion_lab.scripts.run_lab` và association trong `student/workspace/association.py`.
4. Sai calibration thường tạo residual/innovation camera có độ lệch có hệ thống theo vị trí ảnh. Điều đó làm tăng Mahalanobis distance, khiến association bị gate; nếu vẫn được ghép, phép update có thể kéo lệch vị trí ước lượng.
5. Sensor tường minh cho biết lượt hiện tại là LiDAR hay camera ngay cả khi không có measurement, để luôn gọi `manager.manage_tracks(...)`. Lượt LiDAR quản lý hit/miss, sinh và xoá track; lượt camera chỉ update EKF, không thay đổi score hay vòng đời.
6. Track khởi tạo với score `1/window`; mỗi hit LiDAR cộng `1/window` tối đa 1. Track được xác nhận khi score lớn hơn `confirmed_threshold`, và trạng thái confirmed được giữ qua miss. Xoá khi Pxx hoặc Pyy vượt `max_P`, confirmed có score thấp hơn `delete_threshold`, hoặc track chưa confirmed có score ≤ 0 (`student/workspace/track_management.py`).

## Bonus (không bắt buộc)

Liệt kê phần bonus đã làm, file bằng chứng trong `student/bonus/` và kết quả chính
(xem [RUBRIC.md](../RUBRIC.md) mục 2). Không làm thì ghi "Không".

- 

## Khai báo sử dụng AI (bắt buộc)

Ghi rõ, kể cả khi không dùng ("Không dùng AI"). Xem [RULES.md](../RULES.md) mục 2.

- Công cụ đã dùng (ChatGPT, Copilot, Claude, …): Copilot SDK trong VS Code.
- Dùng cho phần nào (hàm, câu hỏi, debug): Triển khai các hàm Part E–H và soạn phần giải thích kỹ thuật ở trên.
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): Chạy `pytest student/tests -q` (128 passed) trong Python 3.12.13; chạy `fusion-run-lab --config student/config/paths.yaml --fusion compare --seed 0` trên đủ frame 0–198; đối chiếu metrics với 398 record trong log, xác nhận duy nhất mỗi cặp (mode, frame), `matches + ghosts == confirmed` và `matches + misses == valid_gt`.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [x] Part A–D: không bắt buộc sửa (hoặc ghi chú nếu bạn đã sửa)
- [x] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [x] Đã tạo `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` bằng runner (không sửa tay)
- [x] Đã điền đủ file này, gồm khai báo AI
- [ ] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [ ] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
