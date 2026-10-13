# Playbook: Node.js

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

## Detection

- `package.json` không có `@angular/core`, `vue`, `react` trong dependencies (xem `detection.md`, dòng 8).
- Test framework đang dùng, xét theo thứ tự:
  - `devDependencies` có `vitest`, `jest` (hoặc `ts-jest`, `@swc/jest`, `babel-jest`), `mocha` (thường kèm `chai`, `sinon`);
  - file cấu hình `vitest.config.*`, `jest.config.*`, trường `"jest"` trong `package.json`, `.mocharc.*`;
  - script `test` trong `package.json`;
  - không có gì ở trên nhưng test mẫu import `node:test` và `node:assert`: dùng test runner có sẵn của Node.
- Hệ module: `"type": "module"` trong `package.json` hoặc file `.mjs` → ESM; ngược lại là CommonJS. TypeScript khi có `tsconfig.json` và mã nguồn `.ts`.
- Trình quản lý gói theo lockfile: `package-lock.json` → npm, `yarn.lock` → yarn, `pnpm-lock.yaml` → pnpm, `bun.lockb`/`bun.lock` → bun. Dùng đúng trình quản lý đó cho mọi lệnh.
- Phiên bản Node: trường `engines.node`, `.nvmrc`, `.node-version`. So với `node -v` ở bước kiểm tra toolchain.

## Default toolchain

Nếu dự án chưa có test framework, đề xuất trong kế hoạch (cần chấp thuận):

- **ESM hoặc TypeScript:** Vitest (`vitest`, `@vitest/coverage-v8`). Chạy ESM/TS không cần cấu hình thêm.
- **CommonJS thuần JavaScript:** Jest (`jest`).
- Dự án không muốn thêm dependency: dùng `node:test` + `node:assert/strict` (Node ≥ 18), không cần cài gì.

Nếu dự án đã dùng:

- **Jest:** viết test Jest. TypeScript đi qua `ts-jest`, `babel-jest` hoặc `@swc/jest` đã cấu hình sẵn; không đổi transformer.
- **Mocha:** viết test Mocha với thư viện assertion (`chai`, `node:assert`) và thư viện mock (`sinon`, `testdouble`) mà dự án đang dùng. Không thêm `sinon` nếu dự án chưa có; khi đó tiêm dependency giả bằng tay.
- **Vitest:** viết test Vitest; import `describe/it/expect/vi` từ `vitest` trừ khi cấu hình bật `globals: true` và test mẫu không import.
- **AVA, tap, uvu:** không có hướng dẫn chi tiết ở playbook này. Theo test mẫu của dự án; nếu không đủ căn cứ, báo người dùng.

## Test location and naming

- Theo vị trí test mẫu. Hai kiểu phổ biến:
  - cạnh file nguồn: `src/order/order-service.js` → `src/order/order-service.test.js`;
  - thư mục riêng: `test/` hoặc `__tests__/`, lặp lại cấu trúc thư mục của `src/`.
- Hậu tố: `.test.*` hoặc `.spec.*` theo test mẫu. Phần mở rộng theo ngôn ngữ và hệ module của file nguồn (`.js`, `.mjs`, `.cjs`, `.ts`).
- Kiểm tra pattern mà runner dùng để tìm test: `testMatch`/`testRegex`/`roots` của Jest, `include` của Vitest, `spec` trong `.mocharc` hoặc tham số trong script `test`. Tên file mới phải khớp pattern để được chạy.
- Tên test case: theo test mẫu, ví dụ `describe('<Tên đơn vị>')` lồng `describe('<method>')` và `it('should ... when ...')`, hoặc câu mô tả tiếng Việt nếu dự án viết như vậy.

## Test patterns

- Cấu trúc Arrange-Act-Assert; mỗi `it` kiểm tra một hành vi.
- Hàm `async`: luôn `await` hoặc `return` promise. Lỗi bất đồng bộ kiểm tra bằng `await expect(fn()).rejects.toThrow(...)` (Jest/Vitest), `await assert.rejects(...)` (`node:assert`) hoặc `chai-as-promised` nếu dự án có.
- Lỗi đồng bộ: `expect(() => fn()).toThrow(<KiểuLỗi hoặc message>)`. Kiểm tra thêm kiểu lỗi, `code` hoặc message, không chỉ "có ném lỗi".
- Giá trị biên: `it.each`/`test.each` (Jest/Vitest) hoặc vòng lặp tạo `it` (Mocha, `node:test`).
- Express/Koa/Fastify handler: gọi thẳng hàm handler hoặc service với `req`/`res` giả tối thiểu. Chỉ dùng `supertest` hoặc `fastify.inject` khi dự án đã dùng; không mở cổng mạng thật.
- So sánh object bằng `toEqual`/`toStrictEqual`/`deepStrictEqual`; số thực bằng `toBeCloseTo`, hoặc so sánh sau khi làm tròn đúng như code nguồn làm.

## Mocking and isolation

- Ưu tiên tiêm dependency giả qua tham số hoặc constructor nếu đơn vị cho phép. Đây là cách ổn định nhất, không phụ thuộc hệ module.
- Mock module khi đơn vị `require`/`import` dependency trực tiếp:
  - Jest CommonJS: `jest.mock('<đường dẫn>')` ở đầu file, rồi `jest.mocked(...)` hoặc `mockResolvedValue`.
  - Jest ESM: `jest.unstable_mockModule(...)` rồi `await import(...)` đơn vị sau đó. Chỉ dùng khi dự án đã chạy Jest với ESM.
  - Vitest: `vi.mock('<đường dẫn>')` (được hoist lên đầu file); dùng `vi.hoisted` cho biến dùng trong factory.
  - Mocha: `sinon.stub(obj, 'method')` trên object được export; `proxyquire`/`esmock` chỉ khi dự án đã có.
  - `node:test`: `mock.method(obj, 'method')`; `mock.module` chỉ có ở Node mới và cần cờ thử nghiệm, không dùng nếu dự án chưa dùng.
- Mock mọi truy cập ra ngoài: HTTP (`fetch`, `axios`, `got`), cơ sở dữ liệu (driver, ORM), `fs`, message queue, email. Với `fetch` toàn cục: `vi.stubGlobal('fetch', ...)`, `jest.spyOn(global, 'fetch')`, hoặc `nock`/`msw` nếu dự án đã có.
- Thời gian:
  - Jest: `jest.useFakeTimers()` + `jest.setSystemTime(...)`; Vitest: `vi.useFakeTimers()` + `vi.setSystemTime(...)`; `node:test`: `mock.timers.enable(...)`; Mocha: `sinon.useFakeTimers(...)`.
  - Luôn khôi phục ở `afterEach` (`useRealTimers`, `restore`).
  - Đơn vị dùng `setTimeout`/`setInterval`: tiến thời gian bằng `advanceTimersByTime`/`runAllTimers`, không `await` thời gian thật.
- `Math.random`, `crypto.randomUUID`: `spyOn(...).mockReturnValue(...)`. Không kiểm soát được thì chỉ assert những gì không phụ thuộc giá trị, hoặc đưa đơn vị vào "Đơn vị bỏ qua".
- Biến môi trường: gán `process.env.X` trong test và khôi phục giá trị cũ ở `afterEach`. Không đọc file `.env` thật.
- Khôi phục mock giữa các test: `restoreAllMocks`/`vi.restoreAllMocks()`/`sinon.restore()` ở `afterEach`, trừ khi cấu hình đã bật `restoreMocks`/`clearMocks`.
- Module tạo kết nối hoặc đọc file ngay khi được import (side effect ở top-level): xem là khó test nếu không mock được module đó; không sửa production.

## Commands

Thay `npm` bằng `yarn`, `pnpm` hoặc `bun` theo lockfile. Ưu tiên script có sẵn trong `package.json`; tham số thêm sau `--` với npm. Nếu `node_modules/` chưa có, cài bằng `npm ci` (có lockfile) hoặc `npm install` và ghi vào báo cáo.

| Mục đích | Jest | Vitest | Mocha | `node:test` |
|---|---|---|---|---|
| Một file | `npx jest src/order/order-service.test.js` | `npx vitest run src/order/order-service.test.ts` | `npx mocha test/order-service.test.js` | `node --test test/order-service.test.js` |
| Một test | `npx jest <file> -t "tên test"` | `npx vitest run <file> -t "tên test"` | `npx mocha <file> --grep "tên test"` | `node --test --test-name-pattern="tên test" <file>` |
| Toàn bộ | `npm test` | `npm test` hoặc `npx vitest run` | `npm test` | `npm test` hoặc `node --test` |
| Coverage | `npx jest --coverage` | `npx vitest run --coverage` | `npx c8 mocha` | `npx c8 node --test` hoặc `node --test --experimental-test-coverage` |

- Vitest phải chạy với `run`; không có `run` sẽ vào chế độ watch và không thoát.
- Jest: thêm `--watchAll=false` nếu script `test` có `--watch`. Thêm `--ci` để snapshot mới không tự được ghi.
- Chạy lại cho mục phát hiện flaky: Jest thêm `--no-cache` nếu nghi kết quả bị cache.
- Kết quả: đọc số test pass/fail/skip ở cuối output. Lỗi cú pháp/import thể hiện là "Test suite failed to run" (Jest) hoặc "Failed to load" (Vitest).
- **Thư mục/file sinh ra khi chạy:** `coverage/`, `.nyc_output/`. Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- Công cụ: Istanbul qua `jest --coverage`, provider `v8` hoặc `istanbul` của Vitest, `c8` cho Mocha và `node:test`.
- Đo không sửa cấu hình:
  - Jest: `--coverage --coverageReporters=json-summary --coverageReporters=text`, thêm `--collectCoverageFrom="<glob theo scope>"` để tính cả file chưa có test.
  - Vitest: cần `@vitest/coverage-v8` (hoặc `-istanbul`). Nếu chưa có, thêm là thay đổi dependency cần chấp thuận; không được chấp thuận thì ghi "không đo được". Tham số: `--coverage.enabled --coverage.reporter=json-summary --coverage.reporter=text --coverage.include="<glob theo scope>"`.
    Vitest mặc định không xuất báo cáo coverage khi có test fail, kể cả test cũ fail sẵn. Thêm `--coverage.reportOnFailure` để vẫn có số liệu.
  - c8: `npx c8 --reporter=json-summary --reporter=text --all --include="<glob theo scope>" <lệnh test>`. `npx` tải `c8` tạm thời, không sửa `package.json`; nếu không có mạng thì ghi "không đo được".
- Đọc số liệu từ `coverage/coverage-summary.json` hoặc bảng text, lọc theo file trong `scope`. Dùng line coverage và branch coverage.
- Thư mục `coverage/` (và `.nyc_output/` nếu có) sinh ra trong phiên phải được xóa nếu dự án không có từ đầu.

## Disabling a test

- Jest, Vitest, Mocha: `it.skip('<tên> TODO(unit-test-agent): <lý do>', ...)`. Mocha không nhận tham số lý do, nên ghi lý do trong tên test hoặc comment `// TODO(unit-test-agent): <lý do>` ngay trên dòng.
- `node:test`: `it('<tên>', { skip: 'TODO(unit-test-agent): <lý do>' }, ...)`.
- Vitest/Jest có `it.todo` nhưng dùng cho test chưa viết, không dùng để disable.
- Đặt trên từng test, không `describe.skip` cả nhóm khi nhóm còn test khác pass.

## Common pitfalls

- Quên `await` với assertion bất đồng bộ (`expect(...).rejects`): test pass dù hành vi sai. Kiểm tra mọi `rejects`/`resolves` đều có `await` hoặc `return`.
- `jest.mock` với đường dẫn sai không báo lỗi rõ ràng mà import module thật: kiểm tra mock có được gọi bằng assertion.
- Vitest `vi.mock` bị hoist: biến khai báo bằng `const` ở trên không dùng được trong factory; dùng `vi.hoisted`.
- Trộn `require` và `import` trong file test ESM: lỗi "require is not defined" hoặc "Cannot use import statement outside a module". Viết test cùng hệ module với test mẫu.
- Fake timer không được khôi phục làm test sau treo hoặc sai; luôn `useRealTimers` trong `afterEach`.
- Fake timer của Jest/Vitest chặn cả promise dùng `setTimeout` bên trong: dùng `advanceTimersByTimeAsync`/`runAllTimersAsync`.
- Test để lại handle mở (server, interval, kết nối) khiến Jest báo "did not exit" hoặc Mocha không thoát: đóng trong `afterEach`/`afterAll`, không thêm `--forceExit` để che lỗi.
- Snapshot mới được tự ghi khi chạy không có `--ci`: không dùng snapshot cho unit test mới trừ khi dự án đã dùng, và không cập nhật snapshot có sẵn (`-u`).
- Số thực: `0.1 + 0.2` khác `0.3`; dùng `toBeCloseTo` hoặc so sánh sau làm tròn.
- Output "No tests found" hoặc "0 passing" nghĩa là test chưa được chạy, không phải pass.
