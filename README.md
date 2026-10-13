# khoand11-plugins

Marketplace plugin cho [Claude Code](https://claude.com/claude-code), tại [github.com/khoand-lab/unit-test-plugin-claude](https://github.com/khoand-lab/unit-test-plugin-claude). Hiện có một plugin:

| Plugin | Phiên bản | Mô tả |
|---|---|---|
| [`brownfield-unit-test`](plugins/brownfield-unit-test) | 0.1.0 | Agent viết unit test cho dự án brownfield theo đúng quy ước test sẵn có của dự án. |

## brownfield-unit-test

Agent đọc dự án để nhận ra stack, framework test và quy ước đang dùng, rồi viết unit test theo đúng các quy ước đó. Agent tự chạy test, tự sửa test fail trong số vòng giới hạn, rồi báo cáo kết quả.

Stack hỗ trợ: Java Spring Boot, Node.js, Python, Angular, React, Vue, Android Java, iOS Swift, Flutter.

### Quy trình 5 pha

1. **Phân tích:** nhận diện stack, framework test, mock, cấu trúc thư mục và quy ước đặt tên.
2. **Lập kế hoạch:** liệt kê đơn vị code sẽ test, file test sẽ tạo hoặc sửa, và dependency cần thêm.
3. **Sinh test:** viết test theo quy ước của dự án.
4. **Chạy và kiểm chứng:** chạy test, tự sửa test fail, kiểm tra test chập chờn (flaky), và kiểm tra không có thay đổi ngoài phạm vi.
5. **Báo cáo:** trạng thái từng test case, coverage trước và sau, lỗi có sẵn, nghi vấn bug và gợi ý refactor.

### Nguyên tắc an toàn

- Không sửa, xóa hay đổi tên code production.
- Không xóa, skip hay làm yếu test đã có.
- Chỉ thêm dependency hoặc sửa cấu hình build khi bạn đã đồng ý.
- Mặc định, agent dừng sau bước lập kế hoạch và chờ bạn xác nhận rồi mới ghi file.
- Không dừng tiến trình mà agent không tự khởi chạy.

## Cài đặt

Yêu cầu: Claude Code và `git`. Muốn agent chạy được test thì máy cần có toolchain của dự án, ví dụ JDK, Node.js, Python hoặc Flutter SDK.

### 1. Thêm marketplace

Từ GitHub:

```bash
claude plugin marketplace add khoand-lab/unit-test-plugin-claude
```

Từ GitLab hoặc host git khác (dùng URL đầy đủ):

```bash
claude plugin marketplace add https://gitlab.com/<username>/<tên-repo>.git
```

Từ thư mục trên máy (để thử trước khi push):

```bash
claude plugin marketplace add <đường-dẫn-tới-repo-này>
```

### 2. Cài plugin

```bash
claude plugin install brownfield-unit-test@khoand11-plugins
```

Cách khác: trong phiên Claude Code, gõ `/plugin marketplace add ...` và `/plugin install ...` với cùng tham số như trên.

Sau khi cài, mở phiên Claude Code mới trong thư mục dự án cần viết test.

> Nếu trước đây bạn đã cài agent dạng rời (các file trong `~/.claude/skills`, `~/.claude/agents`, `~/.claude/commands`), hãy gỡ bản đó trước để không có hai skill trùng nhau.

## Sử dụng

### Lệnh slash

```
/brownfield-unit-test:unit-test scope=src/main/java/com/example/shop coverage-target=80
/brownfield-unit-test:unit-test scope=OrderService#placeOrder mode=auto
/brownfield-unit-test:unit-test scope=changed:origin/main failed-test-policy=disable max-fix-attempts=5
/brownfield-unit-test:unit-test scope=src/services save-report=true
```

Với `mode=confirm` (mặc định), agent in kế hoạch rồi dừng. Trả lời "ok", hoặc yêu cầu chỉnh kế hoạch, để agent làm tiếp.

### Subagent

Subagent chạy trong context riêng, nên không làm đầy context của phiên chính. Bạn có thể yêu cầu bằng lời:

```
Dùng subagent unit-test-writer viết test cho scope=src/services coverage-target=80
```

Với `mode=confirm`, subagent trả kế hoạch về rồi dừng. Để làm tiếp, yêu cầu tiếp tục subagent đó với câu "đồng ý kế hoạch, làm tiếp".

### Ngôn ngữ tự nhiên

Skill `brownfield-unit-test` tự được nạp khi yêu cầu của bạn khớp với mô tả của nó, ví dụ: "viết unit test cho DiscountService, chạy tự động".

### Tham số

| Tham số | Mặc định | Giá trị |
|---|---|---|
| `scope` | bắt buộc | file, thư mục, module, `Class#method`, `changed:<git-ref>` |
| `coverage-target` | không đặt | 0–100 |
| `mode` | `confirm` | `confirm` / `auto` |
| `max-fix-attempts` | `3` | 1–10, tính riêng cho từng file test |
| `failed-test-policy` | `remove` | `remove` (gỡ test fail) / `disable` (giữ lại và đánh dấu skip kèm TODO) |
| `save-report` | không lưu | `true` hoặc đường dẫn file |

## Cập nhật và gỡ cài đặt

Cập nhật lên bản mới nhất:

```bash
claude plugin marketplace update khoand11-plugins
claude plugin update brownfield-unit-test@khoand11-plugins
```

Gỡ cài đặt:

```bash
claude plugin uninstall brownfield-unit-test@khoand11-plugins
claude plugin marketplace remove khoand11-plugins
```

## Cấu trúc repo

```
.claude-plugin/marketplace.json      Khai báo marketplace và danh sách plugin
plugins/brownfield-unit-test/
├── .claude-plugin/plugin.json       Manifest của plugin
├── skills/brownfield-unit-test/     Skill: quy trình 5 pha, playbook theo stack, mẫu kế hoạch và báo cáo
├── agents/unit-test-writer.md       Subagent
└── commands/unit-test.md            Lệnh /unit-test
```

Nội dung trong `plugins/brownfield-unit-test/` là bản build sinh tự động từ repo nguồn. Đừng sửa tay ở đây: mọi thay đổi cần làm ở repo nguồn, build lại, rồi copy sang.
