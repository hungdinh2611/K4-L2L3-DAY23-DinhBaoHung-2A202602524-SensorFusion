# Báo cáo bài nộp — Day 23 Sensor Fusion Lab

> Điền file này rồi commit. Cách nộp: [hướng dẫn nộp](../SUBMISSION.md).

## Thông tin học viên

- Họ tên: Đinh Bảo Hưng
- MSSV: 2A202602524
- Email:
- Link repo (fork): https://github.com/hungdinh2611/K4-Track4-Day23-Sensor-Fusion-Student
- Commit hash nộp (`git rev-parse HEAD`):

## Tóm tắt kết quả

- `fusion_mode` (bắt buộc `compare`), `frames`, `segment`, `seed`: Chưa chạy. Segment mặc định (940.1 MB) đang bị Google Drive chặn tải vì giới hạn "Too many users have viewed or downloaded this file recently".
- `detection.precision`, `detection.recall`, `detection.tp/fp/fn`: Chưa có — cần chạy segment Waymo.
- `tracking.lidar.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: Chưa có — cần chạy segment Waymo.
- `tracking.fused.rmse`, `matches`, `sum_sq_err`, `ghost_track_frames`, `missed_gt_frames`, `mean_confirmed_tracks`: Chưa có — cần chạy segment Waymo.
- Giải thích khác biệt hai mode, đọc RMSE cùng số ghép và ghost/miss: Chưa kết luận vì không có kết quả compare; không tạo hoặc ước đoán metric.

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
- Cách bạn đã kiểm tra lại (pytest, chạy Waymo, đối chiếu công thức): Chạy `pytest student/tests -q` (128 passed) trong Python 3.12.13 và đối chiếu với hợp đồng/test trong repo. Đã tải weights theo nguồn SFA3D được README chấp thuận. Chưa chạy Waymo compare vì Google Drive chặn tải segment do giới hạn lượt truy cập.

## Checklist nộp

- [x] **Part E–H** trong `workspace/` đã implement; `pytest student/tests -q` không còn `failed`/`xfailed`
- [ ] Part A–D: không bắt buộc sửa (hoặc ghi chú nếu bạn đã sửa)
- [ ] Lần chạy chấm điểm: `--fusion compare --seed 0`, `frame_start: 0`, `frame_end: 198`
- [ ] Đã commit `student/artifacts/metrics*.json` và `student/artifacts/grade_run*.log` (không sửa tay)
- [ ] Đã điền đủ file này, gồm khai báo AI
- [ ] Không commit dữ liệu Waymo, weights, `paths.yaml`, API key
- [ ] `python tools/check_submission.py` báo `KẾT QUẢ: SẴN SÀNG NỘP`
- [ ] Đã push và nộp link repo + commit hash trên LMS ([hướng dẫn nộp](../SUBMISSION.md))
