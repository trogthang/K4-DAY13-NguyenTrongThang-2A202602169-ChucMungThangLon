# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: K4-DAY13-ChucMungThangLon
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-group`.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: runner tự động chạy trên Linux amd64, 2026-10-01T07:41–07:42 UTC (theo smoke.json).
- Image tag và image ID: `day13-pointpillars:lc-20261001-amd64` / `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; repo revision `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` (KITTI 000008 chuyển đổi), frame_id=`demo`, sha256=`3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`.
- Checkpoint: PointPillars KITTI có sẵn trong image; `/opt/PointPillars/pretrained/epoch_160.pth`, sha256=`482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window; score threshold: 0.3.
- Giả định kênh thứ tư/intensity và nguồn z_ground: reflectance bị bỏ, dùng kênh hằng (RGB=0 placeholder); z_ground=0.075 m (ước lượng từ PCD). PCD đã dịch z +1.73 m so gốc KITTI, giữ x/y.

## Ba lượt inference thật

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | Chỉ 1 hộp `vehicles` (score 0.322, gần ngưỡng 0.3). Không có delta dịch z, model nhận input ở hệ gốc PCD (+1.73 m) → nhầm tầng cao, hầu hết đối tượng nằm ngoài vùng mặt đất mà checkpoint KITTI kỳ vọng → model gần như không nhận ra vật thể. |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | 13 hộp gồm 10 `vehicles`, 1 `two-wheels`, 2 `pedestrian`. Score cao nhất 0.933. Delta=1.73 đưa z input về gần hệ gốc KITTI → model nhận dạng tốt hơn nhiều. Đây là baseline. |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | 6 hộp, toàn bộ `pedestrian`, không có `vehicles` hay `two-wheels`. Pillar lớn hơn (0.32 vs 0.16) làm mất chi tiết không gian → model không phân biệt được xe, chỉ phát hiện người (kích thước nhỏ hơn). mean_z≈1.091, gần B. |

- A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?
  - Có, khác hoàn toàn. Khi delta=0 (A), model nhận input z cao hơn ~1.73 m so với khi delta=1.73 (B). Model chạy lại trên input đã dịch nên phản ứng phi tuyến: A chỉ ra 1 hộp (score thấp 0.322), B ra 13 hộp (score cao nhất 0.933). Đây **không phải** dịch output cùng 1.73 m — số hộp, vị trí, class đều thay đổi vì mạng neural xử lý input khác. Nếu chỉ dịch output thì A sẽ vẫn có 1 hộp duy nhất với z dịch đi, nhưng thực tế B có 13 hộp hoàn toàn khác.
- B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?
  - Đổi pillar từ 0.16 lên 0.32: số hộp giảm từ 13 xuống 6, tất cả class thành `pedestrian`. Pillar lớn hơn mất độ phân giải không gian, model không phân biệt xe/xe hai bánh. Tuy nhiên, **chưa đủ bằng chứng để kết luận cấu hình nào "tốt hơn"** — nhiều hộp hơn hoặc score cao hơn không tự chứng minh đúng hơn; cần ground truth hoặc reference đã duyệt.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?
  - Chỉ dùng front-window nên vật ngoài ROI phía trước không được model dự đoán — không phải bằng chứng model bỏ sót. Ảnh Side là hình chiếu x-z toàn scene, nhiều đối tượng khác y có thể chồng nhau → không dùng Side một mình để kiểm yaw hay hình học từng hộp. Side giúp nhận lỗi z hệ thống (batch offset) nhưng không thay cho Top/Front/camera.
- JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?
  - Cả ba JSON A/B/C đều là prediction từ pretrained KITTI, không phải nhãn đúng. Chưa JSON nào đủ cơ sở import vào CVAT. Cần kiểm: (1) ground truth hoặc reference, (2) đối chiếu multi-view với camera, (3) hướng/kích thước từng hộp qua Top/Front/Side, (4) tìm hộp thiếu/thừa ở vùng không nằm trong ROI.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 (giữ nguyên z gốc từ B) | Không đổi — giữ nguyên toàn bộ | Không lỗi (bản sao prediction B giữ phép chuyển nguồn) | So z từng hộp case-correct.json với run-B JSON: giống hệt. Ảnh side-correct.png trùng side run-B. |
| case-batch-z | 13 / 13 | −1.805 m (= delta + z_ground = 1.73 + 0.075) | Không — chỉ z bị trừ; x, y, class, length, width, height, yaw, score giữ nguyên | **Dừng batch** — tất cả 13 hộp cùng lệch −1.805 m → lỗi pipeline (quên phép chuyển z ngược), không phải lỗi từng hộp. Yêu cầu LC kiểm phép chuyển frame, tạo lại prediction từ pipeline đúng. | Ví dụ: hộp 0 (vehicles) z correct=0.921 → batch-z=−0.884 (chênh đúng −1.805). Hộp 12 (pedestrian) z correct=1.159 → batch-z=−0.646 (chênh −1.805). Mọi hộp đều âm z trên side-batch-z.png. |
| case-one-box-z | 1 / 13 | −1.805 m (chỉ hộp đầu tiên — vehicles tại x≈8.09) | Không — chỉ z của hộp 0 đổi; 12 hộp còn lại và tất cả trường khác giữ nguyên | **Kiểm từng hộp** — chỉ 1 hộp lệch, các hộp khác bình thường → lỗi đối tượng cục bộ, không phải pipeline. Cần kiểm multi-view hộp ID 0 để xem z có bám mặt đường không. | Hộp 0: z correct=0.921, one-box-z=−0.884 (lệch −1.805). Hộp 1–12: z giống hệt case-correct. Trên side-one-box-z.png, chỉ 1 hộp nằm dưới z=0 rõ rệt, các hộp khác ở vị trí bình thường. |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

Mỗi thành viên tự viết một mục: vai trò đã làm; một quan sát A/B/C có dẫn file hoặc hộp/vùng; diễn giải phép z thuận/ngược; một quyết định lỗi batch và hành động; điều chưa chắc. Chỉ đọc kết quả chuẩn bị trước thì ghi rõ chưa tự chạy.

<!-- ============================================================ -->
<!-- THÀNH VIÊN 1: Phan Tấn Đạt — MSSV 2A202602160              -->
<!-- Lượt A: Vận hành lệnh | Lượt B: Kiểm cấu hình/JSON | Lượt C: Xem hình học -->
<!-- ============================================================ -->

**Phan Tấn Đạt**:
- Vai trò: Vận hành lệnh (A), Kiểm cấu hình/JSON (B), Xem hình học (C).
- Quan sát: So A/B — A chỉ có 1 hộp `vehicles` (score 0.322, z=0.330, `run-A/boxes-demo-delta-0-voxel-0.16.json`) trong khi B có 13 hộp đa class (mean_z=1.034, `run-B/boxes-demo-delta-1.73-voxel-0.16.json`). Khi chạy lệnh lượt A (delta=0), model nhận input ở hệ gốc PCD đã dịch +1.73 m, nằm ngoài vùng mà checkpoint KITTI kỳ vọng → gần như không nhận ra vật thể. Khi kiểm JSON lượt B, xác nhận delta=1.73, voxel_size=0.16 và z_ground=0.075 đúng cấu hình; 13 hộp score từ 0.318 đến 0.933.
- Phép z thuận/ngược: `z_model = z_source - z_ground - delta` (thuận, trước inference) và `z_source = z_model + z_ground + delta` (ngược, sau inference). Nếu quên phép ngược, tất cả hộp sẽ lệch cùng `delta + z_ground = 1.805 m` — chính xác là case-batch-z.
- Quyết định lỗi batch: Khi thấy 13/13 hộp cùng lệch −1.805 m trong case-batch-z → **dừng sửa tay, báo LC kiểm pipeline**. Khi chỉ 1/13 hộp lệch (case-one-box-z) → kiểm từng đối tượng qua nhiều view.
- Điều chưa chắc: Chưa tự chạy model nên không xác nhận được runtime/memory thực tế. z_ground=0.075 là ước lượng từ PCD, chưa rõ có phản ánh mặt đường cục bộ chính xác cho từng hộp không.

**Lưu Quang Hùng**:
- Vai trò: Kiểm cấu hình/JSON (A), Xem hình học (B), Ghi log (C).
- Quan sát: So B/C — khi kiểm hình học lượt B thấy 10 hộp `vehicles` có footprint hợp lý (length 3.1–4.2 m, width 1.5–1.7 m trong `run-B/boxes-demo-delta-1.73-voxel-0.16.json`), nhưng lượt C (pillar=0.32) chỉ còn 6 hộp toàn `pedestrian` với length 0.59–1.07 m (`run-C/boxes-demo-delta-1.73-voxel-0.32.json`). Pillar lớn hơn gấp đôi (0.32 vs 0.16) làm mất chi tiết spatial → model mất khả năng phân biệt xe vs người. Tuy nhiên, nhiều hộp hơn không tự chứng minh đúng hơn — cần ground truth.
- Phép z thuận/ngược: Trước inference, script trừ z_ground và delta để đưa input về hệ model KITTI. Sau inference, cộng ngược lại để đưa hộp về hệ nguồn PCD. Quên bước cộng ngược = mọi hộp chìm xuống cùng 1.805 m → đây là lỗi pipeline, không phải lỗi riêng từng hộp.
- Quyết định lỗi batch: case-batch-z: 13/13 hộp cùng lệch → **dừng batch**, yêu cầu kiểm pipeline trước khi sửa tay. case-one-box-z: 1/13 lệch → **kiểm từng hộp**, xem multi-view hộp đó.
- Điều chưa chắc: Lượt B có hộp tại x=55.58, y=−20.29 (xa nhất) — ở khoảng cách này PCD thưa, chưa đủ cơ sở xác nhận class `vehicles` chỉ từ JSON. Cần camera/Top view để đối chiếu. Chưa tự chạy nên ghi `provided-results`.

**Nguyễn Trọng Thắng**:
- Vai trò: Xem hình học (A), Ghi log (B), Vận hành lệnh (C).
- Quan sát: Lượt A chỉ có 1 hộp duy nhất — xem hình học thấy hộp `vehicles` tại (x=13.15, y=−0.45, z=0.33) với kích thước 3.62×1.52×1.46 m (`run-A/boxes-demo-delta-0-voxel-0.16.json`). Kích thước hợp lý cho sedan, nhưng score chỉ 0.322 (gần ngưỡng 0.3). So với B cùng vùng — B có hộp (x=14.77, y=−1.08, z=0.90) score 0.928, gần cùng vị trí nhưng z khác vì model chạy trên input khác, không phải dịch output. Điều này minh họa rằng đổi delta trước inference thay đổi cả vị trí/score/class, không chỉ dịch z.
- Phép z thuận/ngược: Thuận: `z_model = z_source − 0.075 − delta` để đưa về hệ KITTI checkpoint. Ngược: `z_source = z_model + 0.075 + delta` để trả hộp về hệ PCD nguồn. Khi delta=0, offset chỉ là z_ground=0.075; khi delta=1.73, offset=1.805. Case-batch-z giả lập quên phép ngược.
- Quyết định lỗi batch: Nhìn ảnh `side-batch-z.png` thấy mọi hộp nằm dưới z=0 → dấu hiệu lỗi cả pipeline. Nhìn `side-one-box-z.png` chỉ 1 hộp chìm → lỗi đối tượng, kiểm view riêng. Hành động: batch → dừng, one-box → kiểm từng hộp.
- Điều chưa chắc: Ảnh Side chiếu x-z, chồng nhiều đối tượng khác y → không chắc vị trí y-axis từ Side. Cần Top view để xác nhận footprint nhưng gói minh họa chỉ có Side. Chưa tự chạy nên ghi `provided-results`.

**Nguyễn Phương Thảo**:
- Vai trò: Ghi log (A), Vận hành lệnh (B), Kiểm cấu hình/JSON (C).
- Quan sát: Kiểm JSON lượt C (`run-C/boxes-demo-delta-1.73-voxel-0.32.json`) — delta=1.73 giữ nguyên như B nhưng voxel_size=0.32 (gấp đôi). Kết quả: 6 hộp toàn `pedestrian`, score 0.30–0.81, không có `vehicles` hay `two-wheels`. So với B (13 hộp, 3 class): cùng delta, cùng PCD, cùng checkpoint → sự khác biệt hoàn toàn đến từ pillar size. Voxel lớn hơn làm giảm phân giải biểu diễn input → model nhận dạng sai class (xe thành người) chứ không chỉ bỏ sót. Theo `summary.csv` lượt C, mean_z=1.091, gần B (1.034) → z không phải nguồn gốc khác biệt.
- Phép z thuận/ngược: Pipeline KITTI: trước inference trừ (z_ground + delta), sau inference cộng ngược. Phép này đảm bảo hộp output ở hệ nguồn PCD. Nếu quên cộng ngược → hộp chìm cùng 1.805 m (case-batch-z). Đây khác với dịch z input trước model (thay đổi A/B) vì dịch input thay đổi cả output non-linearly, còn quên phép ngược chỉ dịch output cùng hằng số.
- Quyết định lỗi batch: case-batch-z → 13/13 hộp cùng lệch → **dừng**, kiểm pipeline. case-one-box-z → 1/13 → **kiểm từng hộp**. Không import bất kỳ case nào vào CVAT vì đây là controlled examples từ helper, không phải inference hay nhãn đúng.
- Điều chưa chắc: Lượt C có hộp `pedestrian` tại (x=9.11, y=0.40) — vùng này ở B là `vehicles` (x=8.09, y=1.21). Cùng vùng mà class khác hoàn toàn → cần camera và multi-view để biết thật sự là gì; chỉ JSON chưa đủ kết luận. Chưa tự chạy nên ghi `provided-results`.

## LC ghi nhận riêng

> LC ghi nhận ngày 01/10/2026. **Kết luận: ĐẠT — CẦN XÁC NHẬN NGƯỜI CHẠY.**

- **Quyền dùng PCD/image và đúng ca:** Gói Student KITTI 000008 (giấy phép CC BY-NC-SA 3.0), không dùng dữ liệu Robotaxi. Input SHA-256 `3b5ea3da…` và image `sha256:e03983bd…` (amd64) khớp `smoke.json`.
- **Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:** Đã nhận output + `smoke.json`: passed, amd64, chạy 14:41–14:42; nạp image 48,8 s, A/B/C 5,5 / 4,9 / 3,1 s, kết quả 1/13/6. **Lượt chạy này có trước lúc nhóm viết báo cáo**, nhưng báo cáo ghi `provided-results` và từng người ghi "chưa tự chạy": mâu thuẫn. `smoke.json` không trùng nhóm nào đã nộp.
- **Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:** Đủ `run-A/B/C` và `qc-cases`. Không sửa JSON. Không đưa ca lỗi vào CVAT.
- **Nhận xét từng thành viên và quyết định dừng pipeline:** Đủ 4 người, số liệu khớp file thật, hiểu đúng phép z và quyết định batch/one-box.
- **Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:** **Đồng ý chuyển sang chỉnh/QC.** Nhóm cần xác nhận ai chạy lượt 14:41. Nếu là thành viên nhóm thì sửa trạng thái thành `executed-by-group`; nếu nhận kết quả từ nhóm khác thì giữ `provided-results`, và vẫn cần một lượt chạy riêng.

**Nên sửa:**
1. Xác nhận người chạy lượt 14:41; sửa trạng thái và vai trò cho khớp thực tế.
2. Xoá họ tên, MSSV nằm trong comment HTML ẩn của báo cáo.
