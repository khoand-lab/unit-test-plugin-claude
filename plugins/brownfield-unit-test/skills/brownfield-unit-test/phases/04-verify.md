# Pha 4: Chạy và kiểm chứng

Mục tiêu: mọi test mới hoặc biên dịch và chạy xanh, hoặc được xử lý minh bạch. Không để lại test fail đang chạy. Không che giấu lỗi.

Nếu toolchain thiếu và người dùng đã chọn tiếp tục không kiểm chứng:

- bỏ qua bước 1–5;
- đánh dấu mọi test mới là **"chưa kiểm chứng"** kèm lý do;
- vẫn làm bước 6.

## 1. Chạy từng file test mới

- Chạy riêng từng file test mới hoặc đã sửa bằng lệnh chạy đơn lẻ trong playbook.
- Test pass thì chuyển sang bước 4.
- Không chạy được vì lỗi không liên quan tới test mới (ví dụ lỗi build có sẵn của dự án): dừng thử, đánh dấu các test đó là "chưa kiểm chứng" kèm lý do.
- Lệnh không kết thúc: xử lý theo mục "Lệnh chạy quá lâu" bên dưới.

### Lệnh chạy quá lâu

Áp dụng cho mọi lệnh test, build hay coverage mà bạn chạy trong pha này (cả bước 5).

- Ưu tiên chạy lệnh ở chế độ tự kết thúc (không watch) và đặt giới hạn thời gian cho chính lệnh đó, nếu cách bạn chạy lệnh cho phép.
- Lệnh vẫn treo: chỉ được dừng tiến trình do chính lệnh đó tạo ra, tức tiến trình bạn vừa khởi chạy và biết chắc là của mình.
- Lệnh bị treo rồi được dừng (do giới hạn thời gian hoặc do bạn dừng tiến trình của mình): ghi lệnh bị treo và cách đã xử lý vào mục 7 của báo cáo.
- Không xác định được tiến trình của mình, hoặc không dừng được: không dừng tiến trình nào khác. Báo người dùng, đánh dấu test bị ảnh hưởng là "chưa kiểm chứng" kèm lý do "lệnh test bị treo", và ghi vào báo cáo.
- Không bao giờ liệt kê tiến trình của hệ thống rồi dừng tiến trình theo tên (ví dụ mọi tiến trình node). Xem nguyên tắc an toàn số 7.

## 2. Vòng tự sửa

Test mới fail hoặc không biên dịch được thì phân tích lỗi và sửa **file test**.

- **Một vòng** = một lần sửa + một lần chạy lại.
- Giới hạn `max-fix-attempts` (mặc định 3, hợp lệ 1–10) **tính riêng cho từng file test**.
  - Giá trị không hợp lệ đã được thay bằng 3 và ghi trong kế hoạch ở pha 2.

Được phép sửa các lỗi của chính test: sai import, sai tên kiểu, mock sai cách, thiếu stub, dữ liệu test sai, kỳ vọng hiểu sai ý định của code.

Không được làm để test pass:

- sửa kỳ vọng cho khớp với hành vi mà bạn cho là bug;
- tự ý skip/ignore;
- bắt rồi nuốt exception;
- xóa assertion hoặc làm yếu assertion (ví dụ đổi so sánh cụ thể thành "không null");
- sửa code production.

**Nghi vấn bug trong code production:**

- giữ kỳ vọng đúng với ý định, dừng sửa test đó;
- ghi nghi vấn vào báo cáo kèm mã test minh họa và lỗi quan sát được;
- gỡ test minh họa khỏi dự án như test fail;
- test minh họa chỉ được thêm lại vào bộ test khi người dùng đồng ý.

## 3. Hết số vòng mà vẫn fail: `failed-test-policy`

Chỉ áp dụng lên **test case do agent tạo trong phiên**. Test có sẵn giữ nguyên.

**`remove` (mặc định):**

- gỡ test case fail khỏi file test;
- nếu file test do agent tạo không còn test case nào, xóa file đó; bỏ import hoặc helper chỉ còn test đó dùng;
- lưu lại toàn bộ mã test đã thử và lỗi cuối cùng để đưa vào báo cáo, trạng thái **"đã gỡ"**.

**`disable`:**

- giữ test case và đánh dấu skip bằng cú pháp chuẩn của framework (mục "Disabling a test" của playbook);
- kèm chú thích dạng: `TODO(unit-test-agent): <lý do fail ngắn gọn>`;
- báo cáo ghi trạng thái **"đã disable, chưa hoàn thành"** kèm lỗi cuối cùng.

Sau khi áp dụng, chạy lại file test để chắc chắn phần còn lại chạy xanh.

## 4. Phát hiện test chập chờn (flaky)

- Chạy lại mỗi file test mới **ít nhất một lần nữa** sau khi pass.
- Kết quả khác nhau giữa các lần chạy thì coi là lỗi.
  - Tìm nguồn không tất định: thời gian, ngẫu nhiên, thứ tự, trạng thái chia sẻ, bất đồng bộ.
  - Sửa trong cùng giới hạn `max-fix-attempts` của file đó.
- Không sửa được thì áp dụng `failed-test-policy` và ghi "flaky" trong báo cáo.

## 5. Chạy toàn bộ test của module

Chạy toàn bộ test của mỗi module bị ảnh hưởng. Lệnh không kết thúc thì xử lý theo mục "Lệnh chạy quá lâu" ở bước 1.

- **Test có sẵn fail:**
  - đối chiếu xem nó fail cả khi không có test mới không (ví dụ tạm chạy riêng nó, hoặc so với kết quả chạy trước khi thêm test nếu có);
  - fail từ trước: giữ nguyên, ghi vào báo cáo là **lỗi có sẵn**;
  - fail do test mới gây ra (ví dụ rò rỉ trạng thái, cấu hình chung): sửa test mới trong giới hạn số vòng, không sửa test cũ.
- **Có `coverage-target` hoặc đã đo coverage "trước":** chạy lệnh coverage và ghi coverage "sau" của phạm vi mục tiêu. Không đo được thì ghi lý do.

## 6. Kiểm tra thay đổi ngoài phạm vi

Trước khi sang pha báo cáo, xác định mọi file đã thay đổi trong phiên:

- **Git repo:** chạy `git status --porcelain` và `git diff --name-only`, so với trạng thái ghi nhận đầu phiên. File đã thay đổi từ trước phiên không tính.
- **Không phải git repo:** dùng danh sách file bạn đã tạo hoặc sửa.

Mỗi file thay đổi phải thuộc một trong các loại sau:

- file test;
- helper/fixture test;
- file cấu hình/dependency đã được người dùng chấp thuận.

**File production** hoặc file cấu hình chưa được chấp thuận bị thay đổi trong phiên:

- hoàn tác về nội dung đầu phiên: `git checkout -- <file>` với file đã được theo dõi; xóa nếu là file mới tạo;
- ghi vào báo cáo.

**File sinh ra khi chạy test** (thư mục build, báo cáo coverage) nằm ngoài danh sách trên và không bị ignore: xóa nếu dự án không có thư mục đó từ trước.

Cuối bước này, liệt kê thư mục gốc của mỗi module bị ảnh hưởng và so với lúc đầu phiên để chắc chắn không còn thư mục build hay báo cáo coverage nào bị bỏ lại. Chỉ kiểm tra file nguồn (ví dụ bằng thời điểm sửa) là chưa đủ.

## Kết quả của pha

Mỗi test case mới có đúng một trạng thái:

- **đã kiểm chứng (pass)**;
- **đã gỡ**;
- **đã disable, chưa hoàn thành**;
- **chưa kiểm chứng**.

Có thêm: coverage "sau" hoặc lý do không đo được; danh sách lỗi có sẵn; nghi vấn bug; thay đổi đã hoàn tác.
