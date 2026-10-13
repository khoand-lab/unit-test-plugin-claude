# Pha 3: Sinh test

Chỉ bắt đầu khi kế hoạch đã được xác nhận (`mode=confirm`) hoặc đã hiển thị (`mode=auto`).

Chỉ viết test cho các đơn vị có trong kế hoạch đã chốt. Khi quy tắc chung dưới đây khác với playbook của stack, **playbook được ưu tiên**.

## Trước khi viết mỗi đơn vị

1. Đọc mã nguồn của đơn vị và các kiểu dữ liệu, interface nó dùng trực tiếp.
2. Xác định:
   - input và output;
   - các nhánh điều kiện;
   - exception hoặc lỗi nó ném ra hay xử lý;
   - dependency cần mock;
   - chỗ dùng thời gian, số ngẫu nhiên, biến môi trường.
3. Đọc lại file test mẫu đã chọn ở pha phân tích và mục tương ứng trong playbook.

## Quy tắc viết test

### Unit test cô lập

Cô lập đơn vị khỏi mạng, database thật, file system thật, thiết bị/emulator và thời gian hệ thống.

- Dùng mock, stub hoặc fake theo mục "Mocking and isolation" của playbook.
- Không mở kết nối thật: HTTP, DB, message queue, socket.
- Không đọc/ghi file thật ngoài thư mục tạm do framework test cung cấp.
- Thời gian hiện tại, số ngẫu nhiên, UUID phải được kiểm soát để kết quả tất định. Cách làm: tiêm clock/generator, dùng fake timer, hoặc cố định seed.
- Test không phụ thuộc thứ tự chạy và không chia sẻ trạng thái có thể thay đổi giữa các test.

### Phạm vi case

Với mỗi đơn vị, bao phủ:

- luồng chính;
- từng nhánh điều kiện;
- giá trị biên: rỗng, null/undefined/None, 0, số âm, giá trị lớn nhất/nhỏ nhất, danh sách một phần tử;
- input không hợp lệ;
- mỗi exception hoặc lỗi mà đơn vị ném ra hay xử lý: ít nhất một test khẳng định hành vi lỗi đó.

Nếu có `coverage-target`, ưu tiên nhánh chưa được phủ. Không viết test trùng lặp chỉ để tăng số lượng.

### Assertion có ý nghĩa

Mỗi test case có ít nhất một assertion về hành vi quan sát được:

- giá trị trả về;
- trạng thái sau khi gọi;
- tương tác với dependency (được gọi với tham số gì, bao nhiêu lần);
- UI render hoặc sự kiện phát ra.

Không được:

- viết test chỉ gọi code mà không assert;
- assert lại đúng giá trị vừa mock mà không qua xử lý của đơn vị;
- assert "không ném exception" khi đơn vị có kết quả khác để kiểm tra.

Kỳ vọng phải suy ra từ **ý định** của code: tên, tài liệu, cách nơi khác sử dụng, quy tắc nghiệp vụ hiển nhiên. Không suy ra bằng cách chép lại hành vi hiện tại.

- Chỗ nào code có vẻ sai so với ý định: không viết kỳ vọng khớp với hành vi sai. Ghi nhận nghi vấn bug cho báo cáo (xem pha 4).

### Tuân quy ước dự án

- Đặt file test đúng vị trí và tên theo quy ước đã nhận diện.
- Đặt tên test case, cấu trúc và style assertion giống test mẫu.
- Tái sử dụng helper, builder, fixture, base class sẵn có thay vì viết lại.
- Chỉ tạo helper mới khi cần. Đặt helper trong thư mục test và dùng chung cho các test trong phiên.
- Dùng đúng import style, định dạng và thụt lề của dự án. Nếu dự án có formatter hoặc linter cho test, tuân theo cấu hình của nó.

### Giới hạn khi ghi file

- Chỉ tạo hoặc sửa file test và helper/fixture test.
- File test có sẵn: chỉ **thêm** test case mới. Không sửa hay xóa test case đã có.
- Chỉ sửa file cấu hình/dependency đã được người dùng chấp thuận ở pha 2.
- Không sửa code production, kể cả để thêm "seam" cho dễ test.
- Nếu không có sẵn git để đối chiếu, ghi lại mọi file đã tạo hoặc sửa vào danh sách theo dõi.
- Đánh dấu test case nào do agent tạo trong phiên, để pha 4 chỉ áp dụng `failed-test-policy` lên đúng các test đó.

## Tự rà soát trước khi chuyển pha

Với mỗi file test vừa viết, kiểm tra:

- mọi test case có assertion về hành vi;
- không còn kết nối thật, thời gian thật hay ngẫu nhiên không kiểm soát;
- đủ case chính, nhánh, biên và lỗi như kế hoạch;
- import và tên kiểu đúng với mã nguồn thực tế.
