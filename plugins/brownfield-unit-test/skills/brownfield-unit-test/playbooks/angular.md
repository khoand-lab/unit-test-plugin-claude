# Playbook: Angular

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

## Detection

- `package.json` có `@angular/core` trong dependencies, thường kèm `angular.json` (xem `detection.md`). Workspace nhiều project: đọc `projects` trong `angular.json` và chỉ làm với project chứa file trong `scope`.
- Phiên bản Angular: `@angular/core` trong `package.json`. Phiên bản ảnh hưởng tới:
  - standalone component (mặc định từ v17) hay `NgModule`;
  - zone.js hay zoneless (`provideZonelessChangeDetection`, mặc định ở dự án mới từ v21);
  - signal (`signal`, `input()`, `output()`) hay `@Input`/`@Output`.
- Test runner, xét theo thứ tự:
  - builder của target `test` trong `angular.json`:
    - `@angular/build:unit-test` → Vitest (runner mặc định của dự án mới từ v21), hoặc runner ghi trong `options.runner`;
    - `@angular-devkit/build-angular:karma` hoặc `@angular/build:karma` → Karma + Jasmine;
    - `@angular-builders/jest` → Jest;
  - `devDependencies` có `jest-preset-angular` hoặc file `jest.config.*`/`setup-jest.ts` → Jest, chạy bằng script `test`;
  - `karma.conf.js` → Karma.
- Thư viện assertion theo runner: Jasmine (`toBeTrue`, `jasmine.createSpyObj`), Jest (`jest.fn`), Vitest (`vi.fn`). Không trộn.
- Trình quản lý gói theo lockfile, như playbook Node.js.

## Default toolchain

- Angular CLI luôn tạo sẵn cấu hình test, nên dự án Angular hầu như đã có runner. Dùng đúng runner đó; không chuyển Karma sang Jest/Vitest hoặc ngược lại.
- Target `test` bị xóa hoặc không có runner nào: đề xuất trong kế hoạch (cần chấp thuận) runner mặc định của phiên bản Angular đang dùng. Từ v21 là Vitest qua `@angular/build:unit-test`; trước đó là Karma + Jasmine. Thêm target `test` vào `angular.json` là thay đổi cấu hình, phải nằm trong danh sách chấp thuận.
- Không thêm `@testing-library/angular`, `ng-mocks`, `spectator` nếu dự án chưa có. Nếu đã có, dùng theo test mẫu.

## Test location and naming

- Cạnh file nguồn, hậu tố `.spec.ts`: `src/app/cart/cart.service.ts` → `src/app/cart/cart.service.spec.ts`. Chỉ đặt chỗ khác nếu test mẫu làm vậy.
- Kiểm tra pattern mà builder tìm test: `include` trong options của target `test`, `testMatch` của Jest, hoặc `tsconfig.spec.json` (`include`/`files`). File mới phải khớp pattern để được chạy và được biên dịch.
- Tên test: `describe('<TênClass>')`, lồng `describe('<method>')` nếu test mẫu có, và `it('should ... when ...')` hoặc câu mô tả theo ngôn ngữ của test mẫu.
- Mỗi file spec kiểm tra một đơn vị (component, service, pipe, directive, guard, resolver, interceptor).

## Test patterns

- Cấu trúc Arrange-Act-Assert; mỗi `it` kiểm tra một hành vi.
- **Service, pipe, hàm thuần không cần DI:** khởi tạo trực tiếp bằng `new` với dependency giả. Nhanh hơn và rõ ràng hơn TestBed.
- **Service cần DI (`inject()`):** `TestBed.configureTestingModule({ providers: [Đơn vị, { provide: Dep, useValue: depGiả }] })`, rồi `TestBed.inject(Đơn vị)`. Service dùng `inject()` trong field không khởi tạo được bằng `new`; khi đó bắt buộc dùng TestBed hoặc `TestBed.runInInjectionContext`.
- **Component:**
  - standalone: `imports: [Component]`; theo `NgModule`: `declarations: [Component]` và `imports` mà module của nó cần;
  - `fixture = TestBed.createComponent(...)`, gán input (`fixture.componentRef.setInput('ten', giáTrị)` hoạt động cả với `@Input` lẫn `input()`), rồi `fixture.detectChanges()` hoặc `await fixture.whenStable()`;
  - kiểm tra DOM qua `fixture.nativeElement.querySelector(...)` hoặc `fixture.debugElement.query(By.css(...))`, theo test mẫu;
  - kiểm tra output: subscribe `component.tenOutput` hoặc spy `emit`, rồi kích hoạt sự kiện trên DOM (`click()`, `dispatchEvent`);
  - component con không liên quan: `NO_ERRORS_SCHEMA` chỉ khi test mẫu dùng; với standalone, dùng `TestBed.overrideComponent(..., { remove: { imports: [...] }, add: { imports: [StubCon] } })`.
- **Observable:** subscribe và assert trong callback, kèm kiểm tra callback đã chạy; hoặc `firstValueFrom`/`lastValueFrom` với `await`. Không để assertion nằm trong subscribe mà không biết nó có chạy hay không.
- **Signal:** đọc bằng `sig()` sau khi thay đổi. `computed` tự cập nhật; `effect` cần `TestBed.tick()` (hoặc `TestBed.flushEffects()` ở bản cũ hơn).
- **Guard/resolver dạng hàm:** gọi qua `TestBed.runInInjectionContext(() => guard(route, state))`.
- **Interceptor dạng hàm:** `provideHttpClient(withInterceptors([interceptor]))` + `provideHttpClientTesting()`.

## Mocking and isolation

- HTTP: không gọi mạng thật.
  - Service dùng `HttpClient`: `provideHttpClient()` + `provideHttpClientTesting()` (bản cũ: `HttpClientTestingModule`), rồi `HttpTestingController`:
    - `httpMock.expectOne(url hoặc { method, url })` → kiểm tra `req.request` → `req.flush(dữLiệu)` hoặc `req.flush(lỗi, { status: 500, statusText: '...' })`;
    - `httpMock.verify()` trong `afterEach` để bắt request thừa.
  - Component dùng service gọi HTTP: mock service, không mock `HttpClient`.
- Service giả, theo runner:
  - Jasmine: `jasmine.createSpyObj<Service>('Service', ['method'])`, rồi `spy.method.and.returnValue(of(...))`;
  - Jest: `{ method: jest.fn().mockReturnValue(of(...)) }`;
  - Vitest: `{ method: vi.fn().mockReturnValue(of(...)) }`.
  - Đưa vào `providers: [{ provide: Service, useValue: giả }]`.
- Router: `provideRouter([])` và `RouterTestingHarness` nếu cần điều hướng; spy `router.navigate` khi chỉ cần kiểm tra lời gọi. Không dùng `RouterTestingModule` nếu dự án đã bỏ.
- Thời gian và bất đồng bộ:
  - Dự án có zone.js: `fakeAsync` + `tick(ms)`/`flush()`, hoặc `waitForAsync`.
  - Zoneless (`fakeAsync` không dùng được): fake timer của runner (`jasmine.clock()`, `jest.useFakeTimers()`, `vi.useFakeTimers()`) và `await fixture.whenStable()`.
  - `Date`: `jasmine.clock().mockDate(...)`, `jest.setSystemTime`, `vi.setSystemTime`. Luôn khôi phục ở `afterEach`.
- `localStorage`, `window`: spy trên method (`spyOn(localStorage, 'getItem')`), hoặc provide token trừu tượng nếu dự án đã có.

## Commands

Ưu tiên script trong `package.json` (`npm test`), thêm tham số sau `--`. Nếu `node_modules/` chưa có, cài bằng `npm ci` và ghi vào báo cáo.

| Mục đích | Karma (Jasmine) | `@angular/build:unit-test` (Vitest) | Jest |
|---|---|---|---|
| Một file | `npx ng test --watch=false --browsers=ChromeHeadless --include=src/app/cart/cart.service.spec.ts` | `npx ng test --watch=false --include=src/app/cart/cart.service.spec.ts` | `npx jest src/app/cart/cart.service.spec.ts` |
| Một test | chạy cả file (không dùng `fit`/`fdescribe` vì phải sửa file) | `--include=<file>` kèm `--filter="tên test"` nếu phiên bản hỗ trợ; nếu không, chạy cả file | `npx jest <file> -t "tên test"` |
| Toàn bộ | `npx ng test --watch=false --browsers=ChromeHeadless` | `npx ng test --watch=false` | `npm test` |
| Coverage | thêm `--code-coverage` | thêm `--coverage` | thêm `--coverage` |

- Luôn chạy với `--watch=false`. Mặc định `ng test` chạy ở chế độ watch và không thoát.
- Karma cần Chrome/Chromium. Kiểm tra ở bước toolchain; nếu không có thì đặt `CHROME_BIN` tới Chromium/Edge có sẵn, hoặc báo thiếu toolchain.
- Dự án nhiều project: thêm tên project, ví dụ `npx ng test <tên-project> --watch=false`.
- Kết quả:
  - Karma: dòng `Executed N of M (X FAILED)`; `Executed 0 of 0` nghĩa là không có test nào được chạy.
  - Vitest/Jest: dòng tổng kết `Tests  N passed | M failed`.
  - Lỗi biên dịch TypeScript làm cả bộ test không chạy. Đọc lỗi `TS...` trước khi kết luận test fail.
- **Thư mục/file sinh ra khi chạy:** `coverage/`, `.angular/` (cache của Angular CLI, sinh ra ngay ở lần `ng test` đầu tiên), `dist/`. Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- Karma: `--code-coverage` sinh `coverage/<tên-project>/`. Mặc định có báo cáo HTML; có `coverage-summary.json` hoặc `lcov` hay không phụ thuộc `karma.conf.js`. Không sửa cấu hình: nếu không có file tổng hợp, đọc bảng text (reporter `text-summary`) trong output, hoặc tính từ `lcov.info` nếu có.
- `@angular/build:unit-test`: `--coverage` (cần `@vitest/coverage-v8` nếu runner là Vitest; nếu thiếu, thêm là thay đổi dependency cần chấp thuận). Có thể thêm `--coverage-reporters=json-summary --coverage-reporters=text` nếu phiên bản hỗ trợ.
- Jest: như playbook Node.js (`--coverageReporters=json-summary --coverageReporters=text`).
- Coverage của Angular chỉ tính file được biên dịch trong bộ test. File chưa có spec có thể không xuất hiện trong báo cáo, nên ghi rõ điều này khi báo số liệu toàn `scope`.
- Thư mục `coverage/` sinh ra trong phiên phải được xóa nếu dự án không có từ đầu.

## Disabling a test

- Jasmine: `xit('<tên>', ...)`, kèm dòng comment `// TODO(unit-test-agent): <lý do>` ngay trên. Có thể dùng `pending('TODO(unit-test-agent): <lý do>')` ở đầu thân test nếu dự án đã dùng.
- Jest/Vitest: `it.skip('<tên> TODO(unit-test-agent): <lý do>', ...)`.
- Không dùng `xdescribe`/`describe.skip` cho cả nhóm khi nhóm còn test khác pass.
- Không để lại `fit`/`fdescribe`/`it.only`: chúng làm các test khác của dự án bị bỏ qua mà không báo lỗi.

## Common pitfalls

- Quên `fixture.detectChanges()` hoặc `await fixture.whenStable()`: DOM chưa cập nhật, assertion kiểm tra giá trị cũ.
- Gán thẳng `component.input = ...` với signal input (`input()`) sẽ lỗi hoặc không có tác dụng. Dùng `fixture.componentRef.setInput`.
- `fakeAsync` trong dự án zoneless báo lỗi "zone.js is required". Dùng fake timer của runner.
- Assertion nằm trong `subscribe` của observable không phát giá trị thì test pass mà không kiểm tra gì. Kiểm tra callback đã chạy, hoặc dùng `firstValueFrom`.
- Thiếu `httpMock.verify()` nên không phát hiện request thừa hoặc thiếu.
- Import cả module tính năng thật vào TestBed làm test chậm và kéo theo dependency thật (HTTP, router). Chỉ import đơn vị cần test và provide mock.
- Dùng API của runner khác, ví dụ `jest.fn` trong dự án Jasmine hoặc `jasmine.createSpy` trong Vitest: lỗi biên dịch làm hỏng cả bộ test.
- File spec không khớp `include` của `tsconfig.spec.json` thì không được biên dịch, không chạy và không báo lỗi.
- `Executed 0 of 0 SUCCESS` không phải pass.
