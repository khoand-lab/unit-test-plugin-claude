# Pha 5: Báo cáo

Mục tiêu: cho người dùng bức tranh trung thực về những gì đã làm, những gì chưa làm được và vì sao.

## Cách lập báo cáo

1. Điền mẫu `templates/report.md` bằng kết quả của các pha trước. Giữ nguyên thứ tự và tên các mục. Mục không có nội dung thì ghi "Không có".
   - Đọc lại `templates/report.md` ngay trước khi viết, kể cả khi đã đọc từ đầu phiên hoặc phiên được tiếp tục sau bước xác nhận. Không viết báo cáo theo trí nhớ.
   - Tiêu đề các mục chép đúng từ mẫu, có cả số thứ tự (ví dụ `## 1. Tóm tắt`, `## 2. Test đã tạo`). Không đổi tên mục, không gộp hay thêm mục ngang hàng; thông tin bổ sung đặt bên trong mục phù hợp hoặc ở mục "Vấn đề và khuyến nghị".
2. **Tóm tắt:** số đơn vị đã test, số test case theo từng trạng thái, coverage trước/sau.
3. **Test đã tạo:** liệt kê từng file test đã tạo hoặc sửa, số test case và trạng thái của chúng:
   - **đã kiểm chứng (pass)**;
   - **đã gỡ**;
   - **đã disable, chưa hoàn thành**;
   - **chưa kiểm chứng**.
4. **Coverage:**
   - ghi số liệu "trước" và "sau" của phạm vi mục tiêu, cùng công cụ đo;
   - không đo được thì ghi "không đo được" kèm lý do;
   - có `coverage-target` thì ghi đạt hay chưa đạt; chưa đạt thì chỉ ra phần còn thiếu.
5. **Test fail đã xử lý:** với mỗi test theo `failed-test-policy`:
   - `remove`: ghi mã test đã thử lần cuối và lỗi cuối cùng, để người dùng có thể tự hoàn thiện;
   - `disable`: ghi vị trí test và lỗi cuối cùng.
6. **Nghi vấn bug:** đơn vị, hành vi mong đợi, hành vi quan sát được, mã test minh họa.
7. **Đơn vị bỏ qua:** lý do và gợi ý refactor để có thể test được.
8. **Vấn đề và khuyến nghị:**
   - lỗi có sẵn trong dự án (test cũ fail, lỗi build);
   - thay đổi ngoài phạm vi đã hoàn tác;
   - toolchain thiếu;
   - bước tiếp theo nên làm.
9. **Thay đổi file:** danh sách mọi file đã tạo hoặc sửa, bao gồm file cấu hình/dependency đã được chấp thuận.

## Quy tắc trung thực

- Không ghi "pass" cho test chưa thực sự chạy. Test chưa chạy luôn là "chưa kiểm chứng".
- Không làm tròn hoặc ước lượng số coverage như số đo thật.
- Ghi đủ các test đã gỡ hoặc đã disable. Không lược bỏ để báo cáo trông tốt hơn.
- Phân biệt rõ lỗi do test mới gây ra và lỗi có sẵn của dự án.
- Nếu người dùng từ chối kế hoạch, báo cáo ghi "không sinh test theo yêu cầu người dùng" cùng kết quả phân tích.

## Hiển thị và lưu báo cáo

- Luôn hiển thị báo cáo cho người dùng: toàn văn theo mẫu, đủ tiêu đề các mục, trong tin nhắn cuối của phiên. Không thay bằng đoạn tóm tắt ngắn, kể cả khi bạn thường được khuyến khích trả lời ngắn gọn.
- `save-report` không được đặt: không ghi file báo cáo.
- `save-report=true`: lưu vào `.unit-test-agent/reports/<yyyyMMdd-HHmm>-report.md` tại gốc dự án. Tạo thư mục nếu chưa có.
- `save-report=<đường dẫn>`: lưu vào đường dẫn đó.
- Ghi đường dẫn file báo cáo đã lưu ở cuối báo cáo hiển thị.
- File báo cáo không tính là thay đổi ngoài phạm vi. Gợi ý thêm `.unit-test-agent/` vào `.gitignore` nếu dự án chưa có, nhưng không tự sửa `.gitignore`.
