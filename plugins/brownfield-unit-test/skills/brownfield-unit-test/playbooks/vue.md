# Playbook: Vue

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

## Detection

- `package.json` có `vue` trong dependencies (kể cả qua `nuxt`) (xem `detection.md`, dòng 6).
- Phiên bản Vue: `vue` trong `package.json`. Phiên bản quyết định bộ công cụ test:
  - Vue 3 → `@vue/test-utils` v2 (`props`, `global.plugins`, `global.stubs`);
  - Vue 2 (2.6/2.7) → `@vue/test-utils` v1 (`propsData`, `localVue`, `mocks`, `stubs` ở cấp trên). Không dùng cú pháp v2 cho dự án Vue 2 và ngược lại.
- Khung dự án (quyết định runner và cấu hình sẵn có):
  - Vite (`vite.config.*`, `@vitejs/plugin-vue`) → thường là Vitest, cấu hình `test` nằm trong `vite.config.*` hoặc `vitest.config.*`;
  - Vue CLI (`@vue/cli-service`) → Jest qua `@vue/cli-plugin-unit-jest` (script `test:unit` = `vue-cli-service test:unit`), transformer `vue-jest`/`@vue/vue3-jest`; hoặc Mocha qua `@vue/cli-plugin-unit-mocha`;
  - Nuxt (`nuxt`) → Vitest với `@nuxt/test-utils` nếu dự án đã có; component dùng auto-import cần môi trường `nuxt` hoặc import tường minh;
  - cấu hình tự làm: `jest.config.*`, trường `"jest"` trong `package.json`.
- Test runner, xét theo thứ tự: `vitest` hoặc `jest` trong devDependencies; file cấu hình; script `test`/`test:unit`.
- Môi trường DOM: `environment: 'jsdom'`/`'happy-dom'` (Vitest), `testEnvironment: 'jsdom'` (Jest). Có thể đặt riêng từng file bằng `// @vitest-environment jsdom`.
- Kiểu viết component: `<script setup>`, Composition API (`setup()`), hay Options API. Quản lý state: `pinia` (và `@pinia/testing`), `vuex` (v3 cho Vue 2, v4 cho Vue 3). Router: `vue-router` (v3 cho Vue 2, v4 cho Vue 3). Thư viện test khác: `@testing-library/vue`, `msw`.
- File setup (`setupFiles`, `setupFilesAfterEach`, `tests/setup.*`) cho biết plugin, stub và matcher nào đã được nạp toàn cục (`config.global` của test-utils).
- Trình quản lý gói theo lockfile, như playbook Node.js.

## Default toolchain

Nếu dự án chưa có test framework, đề xuất trong kế hoạch (cần chấp thuận):

- **Dự án Vite/Nuxt:** Vitest + `jsdom` + `@vue/test-utils` (v2 cho Vue 3), thêm khối `test` vào cấu hình Vite.
- **Dự án Vue CLI:** Jest qua `@vue/cli-plugin-unit-jest` (đúng bản theo Vue 2/3) + `@vue/test-utils` tương ứng.
- Thêm file setup và cấu hình runner là thay đổi cấu hình, phải nằm trong danh sách chấp thuận.

Nếu dự án đã có:

- Dùng đúng runner và thư viện đang có. Không chuyển Jest sang Vitest hoặc ngược lại, không chuyển `@vue/test-utils` sang `@testing-library/vue` hoặc ngược lại.
- Chưa có `@pinia/testing`: dùng `setActivePinia(createPinia())` hoặc truyền pinia thật vào `global.plugins`, không tự thêm package.

## Test location and naming

- Theo vị trí test mẫu. Các kiểu phổ biến:
  - cạnh file nguồn: `src/components/CartSummary.vue` → `src/components/CartSummary.spec.ts`;
  - thư mục `__tests__/` cạnh file nguồn;
  - Vue CLI: `tests/unit/CartSummary.spec.js`.
- Hậu tố `.spec.*` hoặc `.test.*` theo test mẫu. Dùng `.ts` nếu dự án dùng TypeScript.
- Kiểm tra pattern tìm test: `include` của Vitest, `testMatch` của Jest (Vue CLI mặc định `**/tests/unit/**/*.spec.[jt]s?(x)` và `**/__tests__/*.[jt]s?(x)`).
- Tên test: theo test mẫu, ví dụ `describe('CartSummary')` và `it('hiển thị ... khi ...')` hoặc `it('should ... when ...')`.

## Test patterns

- Cấu trúc Arrange-Act-Assert; mỗi `it` kiểm tra một hành vi.
- **Hàm thuần, util:** gọi trực tiếp, không mount.
- **Component:** kiểm tra theo đầu ra quan sát được (DOM, sự kiện phát ra), không theo state nội bộ.
  - `mount(Component, { props: {...} })` (v2) hoặc `mount(Component, { propsData: {...} })` (v1). Ưu tiên `mount`; `shallowMount` hoặc `global.stubs` chỉ khi component con kéo theo dependency ngoài.
  - Tìm phần tử bằng `wrapper.find('button')`, `find('[data-test="..."]')` nếu dự án dùng thuộc tính này; `get(...)` khi phần tử phải có, `find(...).exists()` để khẳng định không có.
  - Tương tác: `await wrapper.find('button').trigger('click')`, `await input.setValue('...')`. Luôn `await`: DOM chỉ cập nhật sau tick kế tiếp.
  - Đổi prop: `await wrapper.setProps({...})`.
  - Sự kiện phát ra: `expect(wrapper.emitted('apply')).toEqual([[10]])`; sự kiện không phát: `toBeUndefined()`.
  - Slot: truyền qua `slots: { default: '...' }`.
  - Không đọc `wrapper.vm` để kiểm tra state nội bộ, trừ khi không có đầu ra DOM/sự kiện nào phản ánh hành vi (ghi lý do trong test).
- **Bất đồng bộ:** sau thao tác gọi Promise (API mock), dùng `await flushPromises()` (từ `@vue/test-utils`) rồi mới assert; `await nextTick()` chỉ đủ cho cập nhật reactive đồng bộ.
- **Composable:** composable chỉ dùng `ref`/`computed`/`watch` thì gọi trực tiếp trong test. Composable dùng lifecycle hook (`onMounted`) hoặc `inject` phải chạy trong component: mount một component bọc tối thiểu (`defineComponent({ setup() { result = useX(); return () => null } })`).
- **Pinia:**
  - Test store: `setActivePinia(createPinia())` trong `beforeEach`, rồi gọi action/getter trực tiếp.
  - Component dùng store: `global.plugins: [createTestingPinia({ createSpy: vi.fn, initialState: {...} })]` nếu dự án có `@pinia/testing`; nếu không, truyền `createPinia()` thật và mock module API mà store gọi.
- **Vuex:** tạo store mới cho mỗi test (`createStore({...})` v4, `new Vuex.Store({...})` với `localVue.use(Vuex)` ở v3) với state/action giả tối thiểu; không import store toàn cục của app.
- **Router:** `createRouter({ history: createMemoryHistory(), routes })` rồi `await router.push(...)` và `await router.isReady()` trước khi mount; hoặc `global.mocks: { $route, $router }` khi component chỉ đọc route. Không dùng `createWebHistory`.
- **Plugin toàn cục (i18n, UI library):** truyền qua `global.plugins`/`global.components`; nếu file setup của dự án đã đăng ký qua `config.global` thì không đăng ký lại.
- **Snapshot:** không dùng cho test mới trừ khi dự án đã dùng rộng rãi. Không cập nhật snapshot có sẵn.
- Số thực: `toBeCloseTo`, hoặc so sánh chuỗi đã định dạng đúng như component hiển thị.

## Mocking and isolation

- HTTP: không gọi mạng thật.
  - Component/store/composable gọi qua module API riêng: mock module đó (`vi.mock('@/api/orders')`, `jest.mock(...)`), rồi `mockResolvedValue`/`mockRejectedValue`.
  - Gọi `fetch` trực tiếp: `vi.stubGlobal('fetch', vi.fn())` hoặc `jest.spyOn(globalThis, 'fetch')`; khôi phục ở `afterEach` (`vi.unstubAllGlobals()`, `mockRestore()`).
  - `axios`: mock module `axios` hoặc instance được export. Dùng `msw`/`axios-mock-adapter` chỉ khi dự án đã có.
- Alias `@/`: kiểm tra `resolve.alias` (Vite) hoặc `moduleNameMapper` (Jest) có sẵn; đường dẫn trong `vi.mock` phải khớp cách file nguồn import.
- Thời gian: `vi.useFakeTimers()`/`jest.useFakeTimers()` + `setSystemTime(...)`; luôn `useRealTimers()` trong `afterEach`. Sau khi tiến timer làm đổi state, `await nextTick()` trước khi assert DOM.
- `localStorage`/`sessionStorage`: jsdom có sẵn; `localStorage.clear()` ở `afterEach`, hoặc spy trên `Storage.prototype`.
- `window.matchMedia`, `IntersectionObserver`, `ResizeObserver`: jsdom không có. Stub trong file test và khôi phục; không sửa file setup chung nếu chưa được chấp thuận.
- Biến môi trường: `import.meta.env` (Vite) dùng `vi.stubEnv(...)`; `process.env.VUE_APP_*` (Vue CLI) gán và khôi phục ở `afterEach`.
- Dọn dẹp: gọi `wrapper.unmount()` ở cuối test (hoặc bật `enableAutoUnmount(afterEach)` trong file test) nếu component đăng ký listener toàn cục, timer hoặc `Teleport`; mount với `attachTo: document.body` thì bắt buộc unmount.

## Commands

Thay `npm` bằng `yarn`, `pnpm` hoặc `bun` theo lockfile. Ưu tiên script có sẵn trong `package.json`; tham số thêm sau `--` với npm. Nếu `node_modules/` chưa có, cài bằng `npm ci` và ghi vào báo cáo.

| Mục đích | Vitest | Jest | Vue CLI (`vue-cli-service`) |
|---|---|---|---|
| Một file | `npx vitest run src/components/CartSummary.spec.ts` | `npx jest src/components/CartSummary.spec.ts` | `npx vue-cli-service test:unit tests/unit/CartSummary.spec.js` |
| Một test | `npx vitest run <file> -t "tên test"` | `npx jest <file> -t "tên test"` | thêm `-t "tên test"` |
| Toàn bộ | `npx vitest run` | `npx jest` | `npm run test:unit` |
| Coverage | `npx vitest run --coverage` | `npx jest --coverage` | `npm run test:unit -- --coverage` |

- Vitest phải chạy với `run`; không có `run` sẽ vào chế độ watch và không thoát. Script `test` là `vitest` (không `run`) thì gọi `npx vitest run` thay vì `npm test`.
- Không truyền `--watch` cho `vue-cli-service test:unit`.
- Kết quả: dòng tổng kết `Tests  N passed | M failed` (Vitest) hoặc `Tests: N passed, M failed` (Jest). "No test files found"/"No tests found" nghĩa là test chưa được chạy.
- **Thư mục/file sinh ra khi chạy:** `coverage/`; `dist/` nếu có lệnh build chạy. Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- Vitest: cần `@vitest/coverage-v8` (hoặc `-istanbul`). Nếu chưa có, thêm là thay đổi dependency cần chấp thuận; không được chấp thuận thì ghi "không đo được". Đo không sửa cấu hình: `--coverage.enabled --coverage.reporter=json-summary --coverage.reporter=text --coverage.include="<glob theo scope>"` (glob phải gồm `*.vue`, ví dụ `src/**/*.{ts,vue}`).
- Vitest mặc định không xuất báo cáo coverage khi có test fail, kể cả test cũ fail sẵn. Thêm `--coverage.reportOnFailure` để vẫn có số liệu.
- Jest/Vue CLI: `--coverage --coverageReporters=json-summary --coverageReporters=text --collectCoverageFrom="src/**/*.{js,ts,vue}"` để tính cả file chưa có test.
- Đọc số liệu từ `coverage/coverage-summary.json` hoặc bảng text, lọc theo file trong `scope`. Dùng line coverage và branch coverage.
- File chỉ lắp ráp (`main.ts`, `App.vue` chỉ gồm layout/`<RouterView>`, file khai báo route, file khai báo kiểu) nên ghi là "không cần test".
- Thư mục `coverage/` sinh ra trong phiên phải được xóa nếu dự án không có từ đầu.

## Disabling a test

- Vitest, Jest: `it.skip('<tên> TODO(unit-test-agent): <lý do>', ...)`. Nếu dự án dùng `test(...)` thì `test.skip(...)`.
- `it.todo` dùng cho test chưa viết, không dùng để disable.
- Đặt trên từng test, không `describe.skip` cả nhóm khi nhóm còn test khác pass.
- Không để lại `it.only`/`describe.only`: các test khác của dự án sẽ bị bỏ qua.

## Common pitfalls

- Quên `await` với `trigger`, `setValue`, `setProps`: assertion chạy trước khi DOM cập nhật.
- Dùng `nextTick` thay cho `flushPromises` khi chờ Promise của API mock: DOM vẫn ở trạng thái loading.
- Trộn API của test-utils v1 và v2 (`propsData`/`localVue` với Vue 3, `global` với Vue 2): lỗi hoặc prop bị bỏ qua mà không báo.
- `shallowMount` stub cả component con cần kiểm tra, làm test pass mà không kiểm tra gì. Chỉ stub component con có dependency ngoài.
- Store Pinia/Vuex dùng chung giữa các test khiến kết quả phụ thuộc thứ tự. Tạo mới trong `beforeEach`.
- `createTestingPinia` mặc định stub mọi action (`stubActions: true`): test gọi action thật sẽ không đổi state. Đặt `stubActions: false` khi cần hành vi thật.
- Cảnh báo "Failed to resolve component" hoặc "injection not found": thiếu plugin/component toàn cục trong `global`. Thêm vào `global.plugins`/`global.stubs`, không tắt cảnh báo.
- `vi.mock` bị hoist lên đầu file: biến khai báo ở trên không dùng được trong factory; dùng `vi.hoisted`.
- Component dùng `Teleport` render ra ngoài `wrapper`: tìm phần tử qua `document.body` hoặc stub `teleport: true`.
- `<script setup>` không lộ biến ra `wrapper.vm` trừ khi dùng `defineExpose`; đây là tín hiệu nên kiểm tra qua DOM.
