# Cài agent `brownfield-unit-test` cho Claude Code

Yêu cầu:

- Node.js ≥ 18 để build và cài.
- Claude Code (đã kiểm chứng với 2.1.285, xem `SOURCES.md`).

Mọi lệnh dưới đây chạy tại thư mục gốc của repo agent.

## 1. Build

```bash
npm run build -- --client claude-code
```

Kết quả nằm ở `dist/claude-code/`, là một plugin Claude Code hoàn chỉnh:

```
dist/claude-code/
├── .claude-plugin/plugin.json
├── skills/brownfield-unit-test/   (SKILL.md, phases/, playbooks/, templates/, detection.md)
├── agents/unit-test-writer.md
└── commands/unit-test.md
```

## 2. Cài đặt

### Cách A: dạng rời, mức project (khuyến nghị cho một dự án)

```bash
node scripts/install.mjs --client claude-code --target project --path <thư-mục-dự-án>
```

Lệnh này copy vào dự án đích:

- `.claude/skills/brownfield-unit-test/`
- `.claude/agents/unit-test-writer.md`
- `.claude/commands/unit-test.md`

Có thể commit các file này để cả nhóm dùng chung.

### Cách B: dạng rời, mức user (dùng cho mọi dự án trên máy)

```bash
node scripts/install.mjs --client claude-code --target user
```

Lệnh này copy các file tương tự vào `~/.claude/skills/`, `~/.claude/agents/` và `~/.claude/commands/`.

### Cách C: dạng plugin

- **Thử nhanh, không cần cài:**

  ```bash
  claude --plugin-dir <đường-dẫn-repo>/dist/claude-code
  ```

- Có thể kiểm tra plugin trước bằng `claude plugin validate dist/claude-code`.
- **Phân phối cho nhóm:** đưa `dist/claude-code` vào một plugin marketplace nội bộ, rồi cài bằng `/plugin install`.

Ở dạng plugin, tên thành phần có namespace:

- lệnh: `/brownfield-unit-test:unit-test`
- subagent: `brownfield-unit-test:unit-test-writer`

Không nên dùng đồng thời cách C với cách A/B, vì sẽ có hai bản skill trùng nhau.

Sau khi cài, khởi động lại Claude Code (hoặc mở phiên mới) trong thư mục dự án.

## 3. Cách gọi

### Bằng lệnh slash

```
/unit-test scope=src/main/java/com/example/shop coverage-target=80
/unit-test scope=OrderService#placeOrder mode=auto
/unit-test scope=changed:origin/main failed-test-policy=disable max-fix-attempts=5
/unit-test scope=src/services save-report=true
```

Lệnh chạy trong phiên hiện tại. Với `mode=confirm` (mặc định), agent in kế hoạch rồi dừng. Trả lời "ok" (hoặc yêu cầu chỉnh sửa) để agent làm tiếp.

### Bằng subagent (context riêng, không làm đầy context phiên chính)

Yêu cầu bằng lời, ví dụ:

```
Dùng subagent unit-test-writer viết test cho scope=src/main/java/com/example/shop coverage-target=80
```

Hoặc gọi trực tiếp bằng `@unit-test-writer`.

Với `mode=confirm`, subagent trả kế hoạch về rồi dừng. Xác nhận bằng cách yêu cầu tiếp tục subagent đó với câu "đồng ý kế hoạch, làm tiếp".

### Bằng ngôn ngữ tự nhiên

Skill `brownfield-unit-test` được tự nạp khi yêu cầu khớp mô tả, ví dụ: "viết unit test cho DiscountService, chạy tự động".

## 4. Tham số

| Tham số | Mặc định | Giá trị |
|---|---|---|
| `scope` | bắt buộc | file, thư mục, module, `Class#method`, `changed:<git-ref>` |
| `coverage-target` | không đặt | 0–100 |
| `mode` | `confirm` | `confirm` / `auto` |
| `max-fix-attempts` | `3` | 1–10, tính cho từng file test |
| `failed-test-policy` | `remove` | `remove` / `disable` |
| `save-report` | không lưu | `true` hoặc đường dẫn file |

## 5. Gỡ cài đặt

```bash
node scripts/install.mjs --client claude-code --target project --path <thư-mục-dự-án> --uninstall
node scripts/install.mjs --client claude-code --target user --uninstall
```

Lệnh gỡ chỉ xóa các file agent đã cài, cùng các thư mục `.claude/...` nếu chúng trở nên rỗng. Với dạng plugin, gỡ bằng `/plugin uninstall brownfield-unit-test`, hoặc ngừng truyền `--plugin-dir`.

## 6. Cập nhật

Build lại, rồi chạy lại lệnh cài. Lệnh cài ghi đè bản cũ.

Không sửa trực tiếp file đã cài. `npm run check` phát hiện bản build bị sửa tay; mọi thay đổi phải nằm trong `core/` hoặc `adapters/claude-code/`.
