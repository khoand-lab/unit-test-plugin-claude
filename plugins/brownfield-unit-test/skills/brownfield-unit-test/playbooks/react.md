# Playbook: React

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

## Detection

- `package.json` có `react` trong dependencies (kể cả qua `next`), không có `react-native` (xem `detection.md`, dòng 7).
- Phiên bản React: `react` trong `package.json`. Phiên bản ảnh hưởng tới:
  - `act` import từ `react` (v18.3+) hay từ `react-dom/test-utils` (bản cũ, đã bị bỏ ở v19);
  - `react-test-renderer` đã deprecated từ v19; không dùng cho test mới.
- Khung dự án (quyết định runner và cấu hình sẵn có):
  - Vite (`vite.config.*`, `vite` trong devDependencies) → thường là Vitest, cấu hình `test` nằm trong `vite.config.*` hoặc `vitest.config.*`;
  - Create React App (`react-scripts`) → Jest chạy qua `react-scripts test`, cấu hình ẩn; file setup là `src/setupTests.*`;
  - Next.js (`next`) → Jest qua `next/jest` trong `jest.config.*`, hoặc Vitest;
  - cấu hình tự làm: `jest.config.*`, trường `"jest"` trong `package.json`.
- Test runner, xét theo thứ tự: `vitest` hoặc `jest` trong devDependencies; file cấu hình; script `test`.
- Môi trường DOM: `environment: 'jsdom'`/`'happy-dom'` (Vitest), `testEnvironment: 'jsdom'` (Jest, cần `jest-environment-jsdom` từ Jest 28). Có thể đặt riêng từng file bằng comment `// @vitest-environment jsdom` hoặc `/** @jest-environment jsdom */`.
- Thư viện test: `@testing-library/react`, `@testing-library/user-event`, `@testing-library/jest-dom` (matcher `toBeInTheDocument`), `msw`, `enzyme` (dự án cũ). File setup (`setupFiles`/`setupFilesAfterEach`, `setupTests.*`) cho biết matcher nào đã được nạp.
- Trình quản lý gói theo lockfile, như playbook Node.js.

## Default toolchain

Nếu dự án chưa có test framework, đề xuất trong kế hoạch (cần chấp thuận):

- **Dự án Vite:** Vitest + `jsdom` + `@testing-library/react` + `@testing-library/user-event` + `@testing-library/jest-dom`, thêm khối `test` vào cấu hình Vite.
- **Dự án khác:** Jest + `jest-environment-jsdom` + các thư viện Testing Library trên, kèm transformer phù hợp (`babel-jest` với preset React, hoặc `ts-jest`/`@swc/jest` cho TypeScript).
- Thêm file setup và cấu hình runner là thay đổi cấu hình, phải nằm trong danh sách chấp thuận.

Nếu dự án đã có:

- Dùng đúng runner và thư viện đang có. Không chuyển Jest sang Vitest hoặc ngược lại.
- Chưa có `@testing-library/user-event`: dùng `fireEvent` từ `@testing-library/react`, không tự thêm package. Tương tự với `@testing-library/jest-dom`: không có thì assert bằng thuộc tính DOM (`element.textContent`, `button.disabled`).
- **Enzyme:** dự án cũ còn dùng thì viết test mới theo test mẫu nếu adapter khớp phiên bản React; không có adapter cho React 18+ thì dùng Testing Library nếu đã có, ngược lại báo người dùng.

## Test location and naming

- Theo vị trí test mẫu. Hai kiểu phổ biến:
  - cạnh file nguồn: `src/cart/CartSummary.tsx` → `src/cart/CartSummary.test.tsx`;
  - thư mục `__tests__/` cạnh file nguồn hoặc ở gốc `src/`.
- Hậu tố `.test.*` hoặc `.spec.*` theo test mẫu. File test chứa JSX phải có phần mở rộng `.jsx`/`.tsx` (hoặc `.js` nếu transformer của dự án xử lý JSX trong `.js`).
- Kiểm tra pattern tìm test: `include` của Vitest, `testMatch`/`testRegex`/`roots` của Jest. CRA tìm `src/**/*.{test,spec}.*` và `src/**/__tests__/**`.
- Tên test: theo test mẫu, ví dụ `describe('<TênComponent>')` và `it('hiển thị ... khi ...')` hoặc `it('should ... when ...')`.

## Test patterns

- Cấu trúc Arrange-Act-Assert; mỗi `it` kiểm tra một hành vi.
- **Hàm thuần, reducer, util:** gọi trực tiếp, không render.
- **Component:** kiểm tra theo hành vi người dùng thấy, không theo chi tiết cài đặt.
  - `render(<Component prop={...} />)`, rồi tìm phần tử bằng `screen.getByRole(...)` (ưu tiên), `getByLabelText`, `getByText`; `getByTestId` chỉ khi không có cách khác.
  - `getBy*` khi phần tử phải có; `queryBy*` để khẳng định phần tử **không** có; `findBy*` (có `await`) cho phần tử xuất hiện sau thao tác bất đồng bộ.
  - Tương tác: `const user = userEvent.setup()` rồi `await user.click(...)`, `await user.type(...)`. Không có user-event thì `fireEvent.click(...)`.
  - Callback prop: truyền `vi.fn()`/`jest.fn()` và kiểm tra `toHaveBeenCalledWith(...)`.
  - Không kiểm tra state nội bộ, tên class CSS hay số lần render.
- **Custom hook:** `renderHook(() => useHook(tham số))` từ `@testing-library/react` (v13.1+; bản cũ dùng `@testing-library/react-hooks` nếu dự án có). Đổi tham số bằng `rerender(...)`; thao tác làm đổi state bọc trong `act(...)`; giá trị bất đồng bộ dùng `await waitFor(() => expect(result.current...)...)`.
- **Context/Provider:** truyền `wrapper` cho `render`/`renderHook` với Provider thật và giá trị giả. Dự án có hàm render tùy chỉnh (thường `test-utils.tsx`) thì dùng hàm đó.
- **Router:** bọc bằng `MemoryRouter` (React Router) với `initialEntries`, không dùng `BrowserRouter`. Next.js: mock `next/navigation` (`useRouter`, `usePathname`).
- **Redux/Zustand/React Query:** tạo store hoặc `QueryClient` mới cho mỗi test (`retry: false` với React Query), không dùng instance toàn cục của app.
- **Snapshot:** không dùng cho test mới trừ khi dự án đã dùng rộng rãi; nếu có, chỉ snapshot phần nhỏ, ổn định. Không cập nhật snapshot có sẵn.
- Số thực: `toBeCloseTo`, hoặc so sánh chuỗi đã định dạng đúng như component hiển thị.

## Mocking and isolation

- HTTP: không gọi mạng thật.
  - Component/hook gọi qua module API riêng (`api/orders.ts`): mock module đó (`vi.mock('../api/orders')`, `jest.mock('../api/orders')`), rồi `mockResolvedValue`/`mockRejectedValue`.
  - Gọi `fetch` trực tiếp: `vi.stubGlobal('fetch', vi.fn())` hoặc `jest.spyOn(globalThis, 'fetch')`; trả về object có `ok`, `status`, `json()`. Khôi phục ở `afterEach` (`vi.unstubAllGlobals()`, `mockRestore()`).
  - `axios`: mock module `axios` hoặc instance được export. Dùng `msw` hoặc `axios-mock-adapter` chỉ khi dự án đã có.
- Module khác (analytics, logger, component con nặng): `vi.mock`/`jest.mock` với factory trả về bản thay thế tối thiểu. Mock component con chỉ khi nó kéo theo dependency ngoài; ưu tiên render component con thật.
- Thời gian:
  - `vi.useFakeTimers()`/`jest.useFakeTimers()` + `setSystemTime(...)`; luôn `useRealTimers()` trong `afterEach`.
  - Kết hợp fake timer với user-event: `userEvent.setup({ advanceTimers: vi.advanceTimersByTime })` (hoặc `jest.advanceTimersByTime`), nếu không `await user.click` sẽ treo.
  - Tiến thời gian khiến state thay đổi: bọc trong `act(() => vi.advanceTimersByTime(ms))`.
- `localStorage`/`sessionStorage`: jsdom có sẵn; đặt dữ liệu trước test và `localStorage.clear()` ở `afterEach`, hoặc spy trên `Storage.prototype`.
- `window.matchMedia`, `IntersectionObserver`, `ResizeObserver`, `scrollTo`: jsdom không có. Nếu file setup của dự án chưa định nghĩa, stub trong file test (`vi.stubGlobal`/gán `window.X`) và khôi phục; không sửa file setup chung nếu chưa được chấp thuận.
- Biến môi trường: `import.meta.env` (Vite) dùng `vi.stubEnv('VITE_X', '...')`; `process.env` gán và khôi phục ở `afterEach`.
- Dọn dẹp DOM: Testing Library tự `cleanup` sau mỗi test khi runner có `globals`/`afterEach` toàn cục. Vitest không bật `globals` thì phải gọi `cleanup()` trong `afterEach` (hoặc file setup của dự án đã làm).

## Commands

Thay `npm` bằng `yarn`, `pnpm` hoặc `bun` theo lockfile. Ưu tiên script có sẵn trong `package.json`; tham số thêm sau `--` với npm. Nếu `node_modules/` chưa có, cài bằng `npm ci` và ghi vào báo cáo.

| Mục đích | Vitest | Jest | CRA (`react-scripts`) |
|---|---|---|---|
| Một file | `npx vitest run src/cart/CartSummary.test.tsx` | `npx jest src/cart/CartSummary.test.tsx` | `npx react-scripts test --watchAll=false src/cart/CartSummary.test.tsx` |
| Một test | `npx vitest run <file> -t "tên test"` | `npx jest <file> -t "tên test"` | thêm `-t "tên test"` |
| Toàn bộ | `npx vitest run` | `npm test` | `npm test -- --watchAll=false` |
| Coverage | `npx vitest run --coverage` | `npx jest --coverage` | `npm test -- --watchAll=false --coverage` |

- Vitest phải chạy với `run`; không có `run` sẽ vào chế độ watch và không thoát. Script `test` là `vitest` (không `run`) thì gọi `npx vitest run` thay vì `npm test`.
- CRA và nhiều script Jest mặc định watch khi có biến `CI` chưa đặt: luôn thêm `--watchAll=false` (hoặc đặt `CI=true`).
- Kết quả: dòng tổng kết `Tests  N passed | M failed` (Vitest) hoặc `Tests: N passed, M failed` (Jest). "No test files found"/"No tests found" nghĩa là test chưa được chạy.
- **Thư mục/file sinh ra khi chạy:** `coverage/`; `dist/` nếu có lệnh build chạy. Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- Vitest: cần `@vitest/coverage-v8` (hoặc `-istanbul`). Nếu chưa có, thêm là thay đổi dependency cần chấp thuận; không được chấp thuận thì ghi "không đo được". Đo không sửa cấu hình: `--coverage.enabled --coverage.reporter=json-summary --coverage.reporter=text --coverage.include="<glob theo scope>"`.
- Vitest mặc định không xuất báo cáo coverage khi có test fail, kể cả test cũ fail sẵn. Thêm `--coverage.reportOnFailure` để vẫn có số liệu.
- Jest/CRA: `--coverage --coverageReporters=json-summary --coverageReporters=text --collectCoverageFrom="<glob theo scope>"` để tính cả file chưa có test.
- Đọc số liệu từ `coverage/coverage-summary.json` hoặc bảng text, lọc theo file trong `scope`. Dùng line coverage và branch coverage.
- File chỉ lắp ráp (`main.tsx`, `index.tsx`, `App.tsx` chỉ gồm routing/provider, file khai báo kiểu) nên ghi là "không cần test" thay vì kéo số coverage xuống.
- Thư mục `coverage/` sinh ra trong phiên phải được xóa nếu dự án không có từ đầu.

## Disabling a test

- Vitest, Jest: `it.skip('<tên> TODO(unit-test-agent): <lý do>', ...)`. Nếu dự án dùng `test(...)` thì `test.skip(...)`.
- `it.todo` dùng cho test chưa viết, không dùng để disable.
- Đặt trên từng test, không `describe.skip` cả nhóm khi nhóm còn test khác pass.
- Không để lại `it.only`/`describe.only`/`fit`: các test khác của dự án sẽ bị bỏ qua.

## Common pitfalls

- Cảnh báo "An update to X inside a test was not wrapped in act(...)": test kết thúc trước khi state bất đồng bộ cập nhật. Chờ bằng `await screen.findBy...` hoặc `await waitFor(...)`; không tắt cảnh báo và không bọc `act` tràn lan.
- `waitFor` với callback không có assertion ném lỗi thì chờ không có tác dụng. Chỉ đặt một assertion bên trong `waitFor`, và không đặt thao tác gây side effect trong đó.
- Quên `await` với `user.click`/`user.type` (user-event v14 là bất đồng bộ): assertion chạy trước khi sự kiện xử lý xong.
- `getBy*` ném lỗi khi không tìm thấy, nên `expect(getBy...).not.toBeInTheDocument()` không bao giờ pass đúng cách. Dùng `queryBy*`.
- `toBeInTheDocument` không có khi file setup chưa nạp `@testing-library/jest-dom`: lỗi "Invalid Chai property" (Vitest) hoặc "is not a function" (Jest). Kiểm tra file setup trước khi dùng matcher của jest-dom.
- `vi.mock` bị hoist lên đầu file: biến khai báo ở trên không dùng được trong factory; dùng `vi.hoisted`.
- Fake timer cùng user-event mà không cấu hình `advanceTimers` làm test treo tới timeout.
- Test phụ thuộc thứ tự vì store, `QueryClient`, `localStorage` hoặc mock dùng chung không được làm mới giữa các test. Tạo mới trong mỗi test và khôi phục mock ở `afterEach` (trừ khi cấu hình đã bật `restoreMocks`).
- React `StrictMode` trong wrapper làm effect chạy hai lần; không assert số lần gọi effect nếu dự án bọc StrictMode.
- Import CSS/ảnh trong component gây lỗi cú pháp ở Jest nếu thiếu `moduleNameMapper`. Theo cấu hình có sẵn; nếu chưa có, báo trong kế hoạch thay vì tự sửa.
