# Playbook: Flutter

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

Playbook này chỉ dành cho **unit test và widget test** chạy bằng `flutter test` trên máy dev (không cần thiết bị). Không viết integration test (`integration_test/`, `flutter drive`), không chạy emulator/simulator.

## Detection

- `pubspec.yaml` có `flutter` trong `dependencies` (`flutter: sdk: flutter`) (xem `detection.md`, dòng 1). Thư mục `android/`, `ios/` thuộc dự án Flutter, không phải module riêng.
- Package Dart thuần (không có `flutter` trong dependencies) dùng `dart test` thay cho `flutter test`; các nguyên tắc còn lại của playbook vẫn áp dụng.
- Monorepo: nhiều `pubspec.yaml` (thường trong `packages/*`), có thể dùng `melos` (`melos.yaml`) hoặc Dart workspace (`workspace:` trong `pubspec.yaml` gốc). Mỗi package là một module; chạy test trong thư mục của package đó.
- Phiên bản: `environment: sdk:` và `flutter:` trong `pubspec.yaml`; phiên bản đã cài: `flutter --version`. Ràng buộc SDK không khớp làm `flutter pub get` thất bại.
- Thư viện test trong `dev_dependencies`: `flutter_test` (mặc định), `test`, `mocktail`, `mockito` (+ `build_runner`, file `*.mocks.dart` sinh ra), `bloc_test`, `fake_async`, `clock`, `http` (`MockClient` trong `package:http/testing.dart`), `network_image_mock`, `golden_toolkit`.
- Quản lý state: `provider`, `flutter_bloc`/`bloc`, `riverpod`, `get`, `ChangeNotifier`/`ValueNotifier` thuần.
- Cấu hình test: `dart_test.yaml` (tag, timeout, platform), `test/flutter_test_config.dart` (cấu hình chạy trước mọi test trong thư mục).
- Lint: `analysis_options.yaml` (`flutter_lints`, `very_good_analysis`…); test mới không được thêm cảnh báo mới.
- Toolchain: Flutter SDK (`flutter --version`). `flutter test` chạy trên máy dev, không cần Android SDK/Xcode.

## Default toolchain

Nếu dự án chưa có test, đề xuất trong kế hoạch (cần chấp thuận):

- `flutter_test` (đã có sẵn trong template `flutter create`, kiểm tra `dev_dependencies`).
- `mocktail` cho mock (không cần sinh code). Thêm là thay đổi `pubspec.yaml` cần chấp thuận.

Nếu dự án đã có:

- Dùng đúng thư viện mock đang dùng. Dự án dùng `mockito` thì dùng `@GenerateMocks`/`@GenerateNiceMocks` và chạy `dart run build_runner build --delete-conflicting-outputs` để sinh `*.mocks.dart`; không trộn `mocktail` vào.
- Dùng `bloc_test` cho Bloc/Cubit chỉ khi dự án đã có.
- Không có thư viện mock: viết fake tay bằng `implements` interface/abstract class trong file test, không tự thêm package.

## Test location and naming

- Vị trí: `test/` mô phỏng cấu trúc `lib/`: `lib/src/discount/discount_calculator.dart` → `test/src/discount/discount_calculator_test.dart`. Theo test mẫu nếu dự án tổ chức khác (ví dụ `test/unit/`, `test/widget/`).
- Tên file **phải** kết thúc bằng `_test.dart`; file khác không được `flutter test` tìm thấy.
- Import code bằng `package:<tên_package>/...` (tên trong `pubspec.yaml`), không import tương đối vượt ra ngoài `test/`.
- Tên test: theo test mẫu, ví dụ `group('DiscountCalculator', ...)` → `group('rate', ...)` → `test('returns 0.2 for gold', ...)`, hoặc mô tả tiếng Việt nếu test mẫu dùng.

## Test patterns

- Cấu trúc Arrange-Act-Assert; mỗi `test`/`testWidgets` kiểm tra một hành vi. Khởi tạo trong `setUp`, dọn trong `tearDown`.
- **Lớp Dart thuần, hàm thuần:** gọi trực tiếp, `expect(actual, matcher)`.
- **Exception:** `expect(() => sut.apply(-1, 0.1), throwsA(isA<ArgumentError>()))`; với Future: `expect(sut.load(), throwsA(...))` hoặc `await expectLater(...)`.
- **Bất đồng bộ:** test là `async` và `await` trực tiếp. Stream: `expectLater(stream, emitsInOrder([...]))`.
- **ChangeNotifier/ValueNotifier:** gọi method, kiểm tra property; kiểm tra số lần `notifyListeners` bằng listener đếm khi đó là hành vi cần kiểm tra. `dispose()` ở `tearDown`.
- **Bloc/Cubit:** `blocTest<DiscountCubit, DiscountState>(..., build:, act:, expect:)` nếu có `bloc_test`; nếu không, `expectLater(cubit.stream, emitsInOrder([...]))` rồi gọi action. Đóng cubit ở `tearDown`.
- **Widget:** `testWidgets('...', (tester) async { ... })`.
  - Bọc widget bằng `MaterialApp(home: ...)` (hoặc wrapper dự án đã có) để có `Directionality`, theme, `Navigator`, localization.
  - Cung cấp dependency qua `Provider`/`BlocProvider`/`ProviderScope(overrides: [...])` với bản giả, theo cách dự án làm.
  - Tìm bằng `find.text`, `find.byType`, `find.byKey`, `find.widgetWithText`; ưu tiên văn bản/semantics người dùng thấy. Kiểm tra bằng `findsOneWidget`, `findsNothing`.
  - Tương tác: `await tester.tap(...)`, `await tester.enterText(...)`, sau đó `await tester.pump()` (một frame) hoặc `pumpAndSettle()` (chờ animation). `pumpAndSettle` treo nếu có animation vô hạn (`CircularProgressIndicator`): dùng `pump()` hoặc `pump(Duration)`.
  - Future từ dependency giả: sau `pumpWidget`, gọi `await tester.pump()` để `FutureBuilder`/state cập nhật.
- **Golden/snapshot:** không dùng cho test mới nếu dự án chưa có golden test. Không cập nhật golden có sẵn (`--update-goldens`).
- Số thực: `closeTo(expected, 0.0001)`.

## Mocking and isolation

- `mocktail`: `class MockCustomerApi extends Mock implements CustomerApi {}`; `when(() => api.fetchCustomer(any())).thenAnswer((_) async => customer)`; `verify(() => api.fetchCustomer('c-1')).called(1)`. Kiểu tùy biến dùng với `any()` cần `registerFallbackValue(...)` trong `setUpAll`.
- `mockito`: mock sinh bởi `build_runner`; `when(api.fetchCustomer(any)).thenAnswer(...)`. File `*.mocks.dart` sinh ra là file mới trong `test/`, phải liệt kê trong danh sách thay đổi.
- HTTP: đơn vị nhận `http.Client` thì truyền `MockClient((request) async => http.Response(body, 200))` (`package:http/testing.dart`). `flutter test` chặn mọi request HTTP thật bằng `HttpOverrides` (trả 400) — test gọi mạng thật sẽ fail chứ không chạm mạng; đây là dấu hiệu đơn vị chưa được tiêm client.
- Thời gian: đơn vị nhận `DateTime Function()` hoặc dùng `package:clock` thì cố định giá trị (`withClock(Clock.fixed(...), () {...})`). Timer/`Future.delayed`: `fakeAsync((async) { ...; async.elapse(...); })` cho unit test, `tester.pump(Duration)` cho widget test. Đơn vị gọi thẳng `DateTime.now()`: chỉ assert phần không phụ thuộc thời gian, hoặc đưa vào "Đơn vị bỏ qua" kèm gợi ý tiêm clock.
- `SharedPreferences`: `SharedPreferences.setMockInitialValues({})` trước khi gọi `getInstance()` (cần `TestWidgetsFlutterBinding.ensureInitialized()` trong unit test).
- Platform channel (plugin như `path_provider`, `url_launcher`): `TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger.setMockMethodCallHandler(channel, handler)` và gỡ ở `tearDown`. Nếu mock quá phức tạp, đưa đơn vị vào "Đơn vị bỏ qua".
- File system (`dart:io` `File`): chỉ dùng thư mục tạm (`Directory.systemTemp.createTempSync()`) và xóa ở `tearDown`, khi đơn vị nhận đường dẫn qua tham số.
- Singleton/service locator (`GetIt`): đăng ký bản giả trong `setUp` và `reset()` ở `tearDown` nếu dự án đã làm vậy.

## Commands

Chạy trong thư mục chứa `pubspec.yaml` của module. Nếu `.dart_tool/` chưa có hoặc `pubspec.lock` cũ hơn `pubspec.yaml`, chạy `flutter pub get` trước và ghi vào báo cáo.

| Mục đích | Lệnh |
|---|---|
| Một file | `flutter test test/src/discount/discount_calculator_test.dart` |
| Một test | `flutter test test/src/discount/discount_calculator_test.dart --plain-name "returns 0.2 for gold"` |
| Toàn bộ module | `flutter test` |
| Coverage | `flutter test --coverage` |

- Package Dart thuần: thay `flutter test` bằng `dart test` (coverage: `dart test --coverage=coverage` rồi `dart run coverage:format_coverage`, chỉ khi dự án đã có package `coverage`).
- `--plain-name` khớp chuỗi con của tên đầy đủ (gồm tên `group`); `--name` nhận regex.
- Kết quả: dòng cuối `All tests passed!` hoặc `Some tests failed.`, kèm bộ đếm `+N -M` (N pass, M fail) và `~K` (skip). "No tests ran" / "No tests match" nghĩa là test chưa được chạy.
- Phân tích tĩnh cho file test mới: `flutter analyze test/<file>` (hoặc `dart analyze`). Lỗi analyzer ở file test mới phải được sửa như lỗi biên dịch.
- Chạy lại cho mục phát hiện flaky: chạy lại cùng lệnh; có thể thêm `--test-randomize-ordering-seed random` để phát hiện phụ thuộc thứ tự.
- **Thư mục/file sinh ra khi chạy:** `build/` (`flutter test` sinh `build/unit_test_assets`), `coverage/`; `.dart_tool/` nếu dự án không có từ đầu. Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- `flutter test --coverage` sinh `coverage/lcov.info`. Mặc định chỉ tính file được test import; dùng `--coverage-path` nếu cần đường dẫn khác.
- Đọc `lcov.info` trực tiếp: mỗi `SF:<file>` có `LF` (số dòng có code) và `LH` (số dòng được chạy); line coverage = tổng `LH` / tổng `LF` của các file trong `scope`. Flutter không xuất branch coverage mặc định; ghi "không có" cho branch.
- Muốn tính cả file chưa có test: liệt kê các file `lib/**.dart` trong `scope` không xuất hiện trong `lcov.info` và cộng số dòng code của chúng vào mẫu số theo ước lượng, ghi rõ là ước lượng; hoặc chỉ báo trên file có trong `lcov.info` và nói rõ giới hạn. Không thêm package (`full_coverage`, `very_good_cli`) để làm việc này.
- Loại khỏi mẫu số: file sinh tự động (`*.g.dart`, `*.freezed.dart`, `*.mocks.dart`, `l10n/`), `main.dart` chỉ gọi `runApp`.
- Thư mục `coverage/` sinh ra trong phiên phải được xóa nếu dự án không có từ đầu.

## Disabling a test

- `test('<tên>', () {...}, skip: 'TODO(unit-test-agent): <lý do>')`.
- `testWidgets('<tên>', (tester) async {...}, skip: true)` kèm chú thích `// TODO(unit-test-agent): <lý do>` ngay phía trên (`testWidgets` nhận `skip` kiểu `bool`).
- Đặt trên từng test, không `skip` cả `group` khi nhóm còn test khác pass.

## Common pitfalls

- File test không kết thúc bằng `_test.dart`: không được chạy, không báo lỗi.
- `pumpAndSettle` timeout do animation vô hạn (spinner): dùng `pump()`.
- Quên `await tester.pump()` sau thao tác hoặc sau khi Future hoàn tất: UI chưa cập nhật, `find` không thấy widget.
- Widget thiếu `MaterialApp`/`Directionality`/`MediaQuery`: lỗi "No MediaQuery widget ancestor found" hoặc "No Directionality widget found".
- Gọi API của plugin hoặc `SharedPreferences` trong unit test mà chưa `TestWidgetsFlutterBinding.ensureInitialized()`: lỗi "Binding has not yet been initialized".
- HTTP thật trong test bị chặn (status 400): không phải lỗi mạng tạm thời; tiêm client giả.
- `mocktail` với `any()` cho kiểu tùy biến thiếu `registerFallbackValue`: lỗi khi chạy.
- `mockito` thiếu file `*.mocks.dart` hoặc file cũ: lỗi biên dịch; chạy lại `build_runner`.
- Timer còn chờ khi widget test kết thúc: lỗi "A Timer is still pending even after the widget tree was disposed"; `pump` đủ thời gian hoặc dùng `fakeAsync`.
- Ràng buộc SDK trong `pubspec.yaml` không khớp Flutter đã cài: `flutter pub get` thất bại. Báo người dùng, không sửa `environment:`.
