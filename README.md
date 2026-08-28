# Object Eraser AI — Xoá Vật Thể Trong Ảnh Bằng AI (Windows)

Công cụ desktop xoá vật thể/watermark/timestamp khỏi ảnh bằng AI (inpainting),
theo cơ chế "bôi vùng cần xoá, AI tự lấp đầy nội dung hợp lý" — có thêm khả
năng xử lý **hàng loạt theo vị trí cố định** mà Windows Photos chưa có.

## 0. Trạng thái dự án — đọc trước khi dùng

Toàn bộ phần **lõi xử lý** (`object_eraser/core/`) đã được viết và **chạy
test thật, có kết quả PASS** trong lúc phát triển: hình học mask (% và neo
góc), đọc/ghi ảnh + EXIF, undo/redo, và pipeline batch (cô lập lỗi, mirror
thư mục, log không mất khi crash). Bộ test nằm ở `tests/`, chạy bằng:

```
python tests/test_mask.py
python tests/test_image_io.py
python tests/test_history.py
python tests/test_batch_processor.py
python tests/test_config.py
```

(hoặc `pytest tests/` nếu có cài pytest).

**Phần giao diện** (`object_eraser/gui/`, dùng PySide6) được viết theo đúng
API PySide6/Qt6 chuẩn, đã rà soát kỹ bằng tay + kiểm tra cú pháp, nhưng
**CHƯA được chạy/nhìn thấy trên màn hình thật** — môi trường viết code này
không có sẵn PySide6, GPU, hay màn hình để mở app lên xem. Việc này khác với
phần lõi, vốn có thể chạy thật bằng Python có sẵn.

Vì vậy: hãy coi V1 này là **bản source code đầy đủ, sẵn sàng chạy thử**, chứ
không phải bản đã verify chạy mượt 100%. Lần đầu chạy `python main.py` trên
máy bạn nên coi là một bước debug bình thường (thường chỉ là vài lỗi
`pip install` thiếu gói, hoặc 1-2 chỗ sai tên hàm PySide6 dễ sửa) — không
phải dấu hiệu code sai kiến trúc. Phần "Xử lý sự cố" bên dưới liệt kê các
lỗi hay gặp nhất khi mới chạy lần đầu.

**Đã làm (V1 - mục 3 trong bản đặc tả gốc):** xoá thủ công 1 ảnh (brush +
rectangle, Auto/Manual, Undo/Redo, so sánh trước/sau), batch vị trí cố định
(vẽ mask 1 lần, neo theo % hoặc theo góc, xem trước, chạy batch có
Tạm dừng/Huỷ, log lỗi, giữ EXIF, không ghi đè ảnh gốc).

**Chưa làm (V2 - mục 4, đúng như bản đặc tả gốc ghi "làm sau khi V1 ổn
định"):** batch tự động nhận diện vật thể (Segment Anything/YOLO), và
template matching để tự căn chỉnh vị trí khi khung hình lệch nhẹ. Kiến trúc
engine đã trừu tượng hoá sẵn để bổ sung sau — xem mục "Thêm engine AI mới"
bên dưới.

---

## 1. Cài đặt

1. Cài Python 3.11 hoặc 3.12 (64-bit) nếu máy chưa có.
2. Mở terminal tại thư mục dự án, cài thư viện:
   ```
   pip install -r requirements.txt
   ```
   Nếu máy có card NVIDIA và muốn dùng GPU cho nhanh, đọc kỹ ghi chú ở đầu
   `requirements.txt` — cần cài `torch` bản CUDA **trước** bước này.
3. Chạy thử: `python main.py`
4. Ở lần chạy đầu tiên khi bấm "Chạy AI", IOPaint sẽ tự tải model LaMa về
   máy (cần mạng, mất khoảng 1 phút tuỳ đường truyền) — chỉ cần làm 1 lần,
   các lần sau dùng lại model đã tải.
5. (Tuỳ chọn) Vào **Tệp → Cài đặt** để bật/tắt GPU hoặc đổi thư mục output
   mặc định.

## 2. Xoá vật thể trên 1 ảnh

1. Mở tool → kéo-thả ảnh vào cửa sổ chính hoặc bấm "📂 Mở ảnh...".
2. Bôi lên vùng chứa vật thể cần xoá (chỉnh cỡ cọ ở thanh bên nếu cần; có
   thể chuyển sang công cụ hình chữ nhật cho vật thể vuông vắn).
3. Chọn **Auto** (xoá ngay sau mỗi nét bôi) hoặc **Manual** (bôi hết các
   vùng rồi bấm "▶ Chạy AI" 1 lần — nhanh hơn khi có nhiều vùng).
4. Chưa ưng thì "↶ Hoàn tác nét vẽ" (khi đang bôi dở) hoặc **Undo/Redo**
   (sau khi AI đã chạy), hoặc "↺ Khôi phục ảnh gốc" để về lại ảnh ban đầu.
5. Tick "Xem ảnh gốc" để so sánh trước/sau bất cứ lúc nào.
6. Bấm "💾 Lưu ảnh..." — ảnh kết quả lưu ra file mới, ảnh gốc giữ nguyên.

## 3. Xoá hàng loạt — vị trí cố định

*(dùng khi vật thể luôn ở cùng 1 chỗ trên nhiều ảnh: watermark, timestamp,
logo góc màn hình...)*

1. Vào tab **"Batch – Vị trí cố định"**.
2. **1️⃣ Chọn ảnh mẫu** — nên chọn ảnh có độ phân giải phổ biến nhất trong bộ
   ảnh của bạn. Thư mục nguồn sẽ tự đặt theo thư mục chứa ảnh mẫu (đổi lại ở
   bước 3 nếu cần).
3. Bôi mask đúng vị trí vật thể cần xoá trên ảnh mẫu (giống hệt thao tác ở
   tab xoá 1 ảnh).
4. **2️⃣ Chọn kiểu neo vị trí**: theo % toàn ảnh, hoặc "Neo theo góc" nếu vật
   thể luôn cách 1 mép ảnh một khoảng **pixel cố định** (chính xác hơn % khi
   các ảnh trong bộ có kích thước/tỉ lệ chênh lệch nhau).
5. **3️⃣ Kiểm tra/đổi lại ảnh nguồn** — chọn 1 file hoặc cả thư mục, tick
   "Bao gồm thư mục con" nếu cần.
6. **4️⃣ Bấm "👁 Xem trước"** — mask sẽ được áp thử lên 2-3 ảnh khác trong bộ
   để bạn kiểm tra trước khi chạy cả batch. Có cảnh báo nếu mask có vẻ bị
   lệch/tràn biên trên ảnh nào đó (thường do ảnh đó khác tỉ lệ khung hình).
7. **5️⃣ Chọn thư mục lưu kết quả** (mặc định: `<thư mục nguồn>/output`).
8. Bấm "▶ Chạy Batch", theo dõi tiến trình — có thể "⏸ Tạm dừng" hoặc
   "✕ Huỷ" bất cứ lúc nào.
9. Xem bảng kết quả khi xong: từng ảnh thành công/lỗi, kèm ghi chú lỗi. File
   `batch_log.csv` trong thư mục output ghi lại toàn bộ, kể cả khi app bị
   tắt đột ngột giữa batch.

## 4. (Chưa làm ở V1) Xoá hàng loạt — tự động nhận diện

Tính năng này (mục 4.1 trong bản đặc tả gốc: dùng Segment Anything/YOLO để
tự tạo mask theo từng ảnh, không cần vị trí cố định) **chưa được xây dựng**.
Đây là tính năng V2, đúng theo kế hoạch gốc là làm sau khi V1 ổn định. Kiến
trúc `InpaintEngine` đã trừu tượng hoá sẵn nên khi cần, chỉ phải thêm 1 bước
"tự sinh mask" trước khi gọi `engine.inpaint()` — không cần sửa lại phần
đọc/ghi ảnh, batch, hay UI.

## 5. Xử lý sự cố thường gặp

- **`ModuleNotFoundError` khi chạy `python main.py`:** thiếu gói trong
  `requirements.txt` — chạy lại `pip install -r requirements.txt`.
- **Lỗi khi khởi tạo engine AI ("Không khởi tạo được model..."):** thường do
  chưa có mạng ở lần chạy đầu (IOPaint cần tải model), hoặc phiên bản
  `iopaint`/`torch` cài không khớp — xem thông báo lỗi chi tiết (đã được
  thiết kế để chỉ rõ nguyên nhân) và `github.com/Sanster/IOPaint` để đối
  chiếu phiên bản mới nhất.
- **Vùng vừa xoá bị mờ/lem:** bôi mask rộng hơn vật thể vài pixel (xem mẹo ở
  mục 6), hoặc cân nhắc engine khác cho vùng phức tạp (Stable Diffusion —
  chưa nối dây sẵn trong V1, xem mục "Thêm engine AI mới").
- **Batch chạy chậm:** bật GPU trong Cài đặt nếu máy có card NVIDIA (cần cài
  đúng bản `torch` CUDA — xem `requirements.txt`).
- **Mask lệch vị trí ở 1 số ảnh khi batch:** ảnh đó khác tỉ lệ khung hình so
  với ảnh mẫu — chuyển sang "Neo theo góc", hoặc tách nhóm ảnh cùng tỉ lệ ra
  chạy riêng. Dùng nút "Xem trước" để phát hiện sớm trước khi chạy cả batch.
- **Không đọc được file HEIC:** cần gói `pillow-heif` (đã có trong
  `requirements.txt` — nếu vẫn lỗi, chạy `pip install pillow-heif` riêng để
  xem thông báo lỗi cài đặt cụ thể).
- **File .exe đóng gói bị lỗi "DLL load failed" / thiếu module:** xem ghi
  chú chi tiết ở đầu `build.spec`.

## 6. Mẹo để kết quả đẹp hơn

- Bôi mask rộng hơn vật thể khoảng 5–10px viền để AI có đủ ngữ cảnh lấp đầy
  tự nhiên.
- Nền phức tạp (hoa văn, chữ, đường thẳng) thường khó hơn cho LaMa — nếu kết
  quả chưa ưng, đây là nơi engine Stable Diffusion (chưa có sẵn ở V1) sẽ cho
  kết quả tốt hơn.
- Luôn "Xem trước" trên vài ảnh trước khi chạy cả nghìn ảnh, tránh phải làm
  lại từ đầu nếu cài đặt neo vị trí chưa đúng.

---

## 7. Kiến trúc mã nguồn

```
main.py                          Điểm khởi động app
object_eraser/
  config.py                      Cài đặt app, lưu JSON ở %APPDATA%
  core/                          KHÔNG phụ thuộc GUI - có thể unit-test độc lập
    mask.py                      Hình học mask: % và neo góc, render ra ảnh mask
    image_io.py                  Đọc/ghi ảnh, EXIF, HEIC, không ghi đè gốc
    inpaint_engine.py            Trừu tượng hoá AI engine (mặc định: LaMa/IOPaint)
    batch_processor.py           Vòng lặp batch: tiến trình, tạm dừng/huỷ, log, cô lập lỗi
    history.py                   Undo/redo (nét vẽ dở + phiên bản ảnh đã áp AI)
  gui/                           PySide6
    canvas_widget.py             Canvas vẽ mask dùng chung (zoom, brush/rect, overlay)
    main_window.py               Cửa sổ chính + tab "Xóa 1 ảnh" + quản lý engine dùng chung
    batch_tab.py                 Tab "Batch - Vị trí cố định"
    settings_dialog.py           Màn hình Cài đặt
  utils/logging_setup.py         Logging ra file xoay vòng, dùng khi báo lỗi
tests/                           Test cho toàn bộ core/ (không cần GUI/AI để chạy)
build.spec                       Cấu hình PyInstaller
```

**Nguyên tắc quan trọng nhất khi đọc/sửa code:** `CanvasWidget` (dùng ở cả 2
tab) luôn vẽ bằng **toạ độ pixel thô** — không biết gì về %/neo góc. Việc
"neo lại" mask theo %/góc chỉ xảy ra ở **1 chỗ duy nhất**:
`BatchFixedPositionTab._build_mask_template()` trong `batch_tab.py`, gọi
`reanchor_rect`/`reanchor_stroke` trong `core/mask.py`. Preview và Chạy Batch
dùng chung kết quả của đúng hàm này nên không thể lệch pha nhau. Nếu sau này
sửa logic neo vị trí, đây là nơi cần sửa.

## 8. Chạy test

```
python tests/test_mask.py
python tests/test_image_io.py
python tests/test_history.py
python tests/test_batch_processor.py
python tests/test_config.py
```

Các test này CHỈ cần Pillow + numpy (không cần PySide6/torch/iopaint), nên
chạy được ngay cả trên máy CI không có GPU. Test cho phần GUI/AI thật
(PySide6 hiển thị đúng, IOPaint inpaint ra ảnh đẹp) cần chạy tay trên máy có
đủ thư viện + màn hình - chưa có trong bộ test tự động này.

## 9. Đóng gói ra file .exe

```
pip install pyinstaller
pyinstaller build.spec
```

Đọc kỹ ghi chú ở đầu `build.spec` trước khi build — đóng gói app có
PyTorch/IOPaint gần như luôn cần build thử vài lần mới ra bản chạy trơn tru
trên máy khác, đây là điều bình thường với loại app này.

## 10. Thêm engine AI mới (Stable Diffusion, API cloud...)

`core/inpaint_engine.py` định nghĩa 1 giao diện chung `InpaintEngine` với 1
hàm cần cài: `inpaint(image, mask) -> image`. Để thêm engine mới:

1. Viết class kế thừa `InpaintEngine`, cài `inpaint()`.
2. Thêm 1 `EngineInfo` vào danh sách `AVAILABLE_ENGINES` (sẽ tự hiện ra
   trong màn hình Cài đặt).
3. Đăng ký trong `create_engine()`.

Không cần sửa gì ở `batch_processor.py`, `canvas_widget.py`, hay bất kỳ tab
GUI nào — tất cả đều gọi qua giao diện `InpaintEngine` chung.
