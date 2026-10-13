# Playbook: iOS Swift

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

Playbook này chỉ dành cho **unit test** (target test chạy logic trong process, không có UI). Không viết UI test (`XCUITest`, target `*UITests`), không điều khiển simulator bằng thao tác giao diện.

## Detection

- Có `*.xcodeproj`/`*.xcworkspace`, hoặc `Package.swift`, và mã nguồn là Swift (xem `detection.md`, dòng 3). Dự án chủ yếu Objective-C là stack không được hỗ trợ.
- Kiểu dự án:
  - **Swift Package** (`Package.swift`): target nguồn ở `Sources/<Target>/`, target test ở `Tests/<Target>Tests/`, khai báo bằng `.testTarget(name:dependencies:)`. Phiên bản tool ở dòng đầu `// swift-tools-version:x.y`.
  - **Xcode project** (`*.xcodeproj`): target app và target test (`<App>Tests`, loại "Unit Testing Bundle"). Có `*.xcworkspace` (CocoaPods, nhiều project) thì luôn build qua workspace. Scheme dùng chung nằm ở `*.xcodeproj/xcshareddata/xcschemes/*.xcscheme`; scheme có mục `<TestAction>` chứa target test mới chạy được test.
  - Dự án dùng **XcodeGen** (`project.yml`) hoặc **Tuist** (`Project.swift`): file `.xcodeproj` được sinh ra; target test mới phải khai báo trong file nguồn đó, không sửa `.xcodeproj` trực tiếp.
- Framework test có sẵn: `import XCTest` (XCTest) hoặc `import Testing` (Swift Testing, macro `@Test`, `#expect`; cần Swift 6 / Xcode 16 trở lên). Một target có thể dùng cả hai.
- Thư viện test khác: `Quick`/`Nimble`, `SnapshotTesting`, `ViewInspector`, `OHHTTPStubs` (khai báo trong `Package.swift`, `Podfile`, hoặc mục Package Dependencies của project).
- Quản lý dependency: Swift Package Manager (`Package.resolved`), CocoaPods (`Podfile`, `Podfile.lock`), Carthage (`Cartfile`).
- Toolchain: **chỉ chạy được trên macOS** có Xcode (hoặc Command Line Tools cho Swift Package thuần không phụ thuộc UIKit). Kiểm tra `xcodebuild -version`, `swift --version`, `xcrun simctl list devices available`. Trên Windows/Linux không có Xcode và không có framework iOS (UIKit, SwiftUI); Swift toolchain trên Linux/Windows chỉ build được package không import framework Apple.

## Default toolchain

Nếu dự án chưa có target test, đề xuất trong kế hoạch (cần chấp thuận):

- **Swift Package:** thêm `.testTarget(name: "<Target>Tests", dependencies: ["<Target>"])` vào `Package.swift`, dùng **XCTest**. Đây là thay đổi manifest cần chấp thuận.
- **Xcode project:** thêm target "Unit Testing Bundle" và đưa vào scheme. Không tự sửa `project.pbxproj` bằng tay: hướng dẫn người dùng thêm target qua Xcode (hoặc qua `project.yml`/`Project.swift` nếu dự án dùng XcodeGen/Tuist), rồi mới sinh test.

Nếu dự án đã có:

- Dùng đúng XCTest hoặc Swift Testing theo test mẫu. Chỉ dùng Swift Testing khi dự án đã dùng và toolchain hỗ trợ.
- Dùng Quick/Nimble chỉ khi test mẫu dùng.
- Không thêm thư viện mock: Swift không có mock sinh tự động phổ biến được mặc định; viết mock/stub tay theo protocol (xem mục Mocking).

## Test location and naming

- Swift Package: `Tests/<Target>Tests/`, mô phỏng thư mục con của `Sources/<Target>/` nếu test mẫu làm vậy.
- Xcode project: thư mục của target test (`<App>Tests/`). File phải được **thêm vào target test**: dự án dùng thư mục đồng bộ (Xcode 16 "folder", `PBXFileSystemSynchronizedRootGroup` trong `project.pbxproj`) thì chỉ cần tạo file trong thư mục; dự án dùng group cũ thì file mới phải được thêm vào target, nếu không sẽ không được biên dịch. Không sửa `project.pbxproj` bằng tay khi chưa được chấp thuận; báo người dùng nếu cần thêm file vào target.
- Tên file: `<TypeName>Tests.swift`. Theo quy ước dự án nếu khác.
- Tên class test (XCTest): `final class <TypeName>Tests: XCTestCase`. Tên method **phải** bắt đầu bằng `test` (ví dụ `test_rate_whenGold_returns20Percent`, `testRateReturnsTwentyPercentForGold`); method không có tiền tố `test` không chạy.
- Swift Testing: `@Suite struct <TypeName>Tests` và `@Test("mô tả") func ...`.
- Truy cập code: `@testable import <Module>` để dùng được kiểu `internal`. Kiểu `private`/`fileprivate` không truy cập được; kiểm tra qua API công khai.

## Test patterns

- Cấu trúc Arrange-Act-Assert (hoặc Given-When-Then theo test mẫu); mỗi test kiểm tra một hành vi.
- Khởi tạo: tạo đối tượng mới trong từng test hoặc trong `setUp()`/`setUpWithError()`; dọn ở `tearDown()`. Swift Testing: khởi tạo trong `init()` của suite.
- **Struct/class thuần, hàm thuần:** gọi trực tiếp.
- **Lỗi ném ra:** `XCTAssertThrowsError(try sut.method()) { error in XCTAssertEqual(error as? DiscountError, .unknownTier) }`; Swift Testing: `#expect(throws: DiscountError.unknownTier) { try sut.method() }`.
- **async/await:** đánh dấu method test là `async throws` và `await` trực tiếp; không dùng `XCTestExpectation` cho code async/await. Code dùng completion handler: `expectation(description:)` + `wait(for:timeout:)` (hoặc `await fulfillment(of:timeout:)` trong test async), timeout ngắn (≤ 1 giây với dependency giả).
- **`@MainActor`:** kiểu được đánh dấu `@MainActor` (ViewModel, `ObservableObject`) thì đánh dấu class test hoặc method test là `@MainActor`.
- **ViewModel (`ObservableObject`, `@Published`, `@Observable`):** kiểm tra giá trị property sau khi gọi method; không cần subscribe Combine trừ khi hành vi cần kiểm tra là chuỗi giá trị phát ra.
- **Combine publisher:** thu giá trị bằng `sink` vào mảng, lưu `AnyCancellable`, chờ bằng expectation hoặc dùng `values` (async sequence).
- **UIViewController / SwiftUI View:** không viết unit test cho view trừ khi dự án đã có pattern (`ViewInspector`, `loadViewIfNeeded()`); đưa logic ra ViewModel để test. Không dùng snapshot test nếu dự án chưa dùng.
- Số thực: `XCTAssertEqual(a, b, accuracy: 0.0001)`; Swift Testing: `#expect(abs(a - b) < 0.0001)`.
- Giá trị biên: nhiều method riêng (XCTest), hoặc `@Test(arguments: [...])` (Swift Testing).

## Mocking and isolation

- Swift không có thư viện mock runtime mặc định. Mock bằng **protocol**: đơn vị nhận dependency qua protocol (`CustomerAPI`) thì viết class/struct giả trong file test (`final class CustomerAPIStub: CustomerAPI`) trả giá trị cấu hình được và ghi lại lời gọi.
- Đơn vị tạo dependency bên trong (`URLSession.shared`, `UserDefaults.standard`, `FileManager.default`, `Date()`, singleton) mà không nhận qua tham số: xem là khó test, không sửa production; đưa vào "Đơn vị bỏ qua" kèm gợi ý tiêm protocol/closure.
- Mạng: không gọi mạng thật. Đơn vị nhận `URLSession` qua init thì có thể dùng `URLSessionConfiguration.ephemeral` với `protocolClasses = [MockURLProtocol.self]` (subclass `URLProtocol` viết trong file test); chỉ dùng `OHHTTPStubs` khi dự án đã có.
- Thời gian: đơn vị nhận `() -> Date`, `Clock` hoặc `DateProvider` thì truyền giá trị cố định. Không `sleep` trong test.
- `UserDefaults`: đơn vị nhận `UserDefaults` qua init thì truyền `UserDefaults(suiteName: "test-\(UUID())")` và `removePersistentDomain(forName:)` ở `tearDown`.
- File system: dùng `FileManager.default.temporaryDirectory` + thư mục riêng cho test và xóa ở `tearDown`, chỉ khi đơn vị nhận đường dẫn qua tham số.
- Không sửa `Info.plist`, entitlement hay cấu hình build để test chạy được.

## Commands

| Mục đích | Swift Package | Xcode project/workspace |
|---|---|---|
| Một class | `swift test --filter DiscountCalculatorTests` | `xcodebuild test -scheme Shop -destination 'platform=iOS Simulator,name=iPhone 16' -only-testing:ShopTests/DiscountCalculatorTests` |
| Một method | `swift test --filter DiscountCalculatorTests/test_rate_whenGold_returns20Percent` | `... -only-testing:ShopTests/DiscountCalculatorTests/test_rate_whenGold_returns20Percent` |
| Toàn bộ target test | `swift test` | `xcodebuild test -scheme Shop -destination '...' -only-testing:ShopTests` |
| Coverage | `swift test --enable-code-coverage` | thêm `-enableCodeCoverage YES -resultBundlePath <thư mục tạm>/Tests.xcresult` |

- Workspace: thay `-project` mặc định bằng `-workspace Shop.xcworkspace`.
- Destination: chọn simulator có sẵn từ `xcrun simctl list devices available`; không tạo simulator mới. Framework/package chỉ dùng Foundation có thể chạy `-destination 'platform=macOS'`.
- Luôn giới hạn bằng `-only-testing:<TargetTests>` để không chạy target UI test.
- Chạy lại cho mục phát hiện flaky: chạy lại cùng lệnh; `xcodebuild` có `-test-iterations 2` (Xcode 13+).
- Lần build đầu có thể mất vài phút; tăng thời gian chờ thay vì kết luận là treo.
- Kết quả: `swift test` in `Executed N tests, with M failures`; `xcodebuild` in `** TEST SUCCEEDED **`/`** TEST FAILED **` và từng dòng `Test Case '-[ShopTests.X testY]' passed/failed`. "Executed 0 tests" nghĩa là test chưa được chạy (sai filter, file chưa thuộc target, method thiếu tiền tố `test`).
- **Không có macOS/Xcode:** không chạy được lệnh nào ở trên. Báo người dùng thiếu toolchain (xem `phases/01-analyze.md`), hỏi có tiếp tục sinh test mà không kiểm chứng không, kể cả khi `mode=auto`. Nếu tiếp tục, mọi test mới có trạng thái "chưa kiểm chứng" và báo cáo ghi lệnh để người dùng tự chạy trên máy Mac.
- **Thư mục/file sinh ra khi chạy:** `.build/` (SwiftPM), thư mục `DerivedData`/`*.xcresult` nếu đặt trong dự án. Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- Swift Package: `swift test --enable-code-coverage`, rồi `swift test --show-codecov-path` để lấy đường dẫn file JSON (định dạng llvm-cov export). Đọc `data[0].files[]`, lọc theo file trong `scope`, dùng `summary.lines` và `summary.branches` (branch có thể bằng 0 nếu toolchain không xuất).
- Xcode: `-enableCodeCoverage YES -resultBundlePath <tạm>/Tests.xcresult`, rồi `xcrun xccov view --report --json <tạm>/Tests.xcresult`; đọc `lineCoverage` theo file/target. Xcode chỉ cho line coverage.
- Đặt result bundle trong thư mục tạm ngoài dự án hoặc xóa sau phiên; không để `.xcresult` lại trong dự án.
- Không có toolchain: ghi "không đo được" kèm lý do.
- Loại khỏi mẫu số file chỉ lắp ráp (`App.swift` có `@main`, `AppDelegate`, `SceneDelegate`, file chỉ khai báo view).

## Disabling a test

- XCTest: đầu method gọi `throw XCTSkip("TODO(unit-test-agent): <lý do>")` (method phải là `throws`).
- Swift Testing: `@Test("...", .disabled("TODO(unit-test-agent): <lý do>"))`.
- Không đổi tên method bỏ tiền tố `test` để "tắt" test, không comment cả class.

## Common pitfalls

- Method test không bắt đầu bằng `test` (XCTest): không chạy và không báo lỗi.
- File test mới chưa thuộc target test (Xcode project dùng group): không được biên dịch, "Executed 0 tests".
- Quên `@testable import`: không thấy kiểu `internal`.
- Kiểu `@MainActor` dùng trong test không đánh dấu `@MainActor`: lỗi biên dịch về actor isolation (Swift 6 strict concurrency).
- `XCTestExpectation` không được `fulfill` hoặc timeout quá dài làm test treo; dùng async/await khi có thể.
- Chạy `xcodebuild test` không có `-only-testing`: chạy cả UI test, cần simulator khởi động giao diện, chậm và dễ fail.
- Simulator không tồn tại: lỗi "Unable to find a destination matching"; chọn từ `simctl list`, không tạo mới.
- Swift Package dùng UIKit/SwiftUI không chạy được bằng `swift test` trên macOS (thiếu SDK iOS); dùng `xcodebuild -scheme <Package> -destination 'platform=iOS Simulator,...'`.
- Biên dịch lỗi do thay đổi trong test làm cả target test không build: mọi test đều fail. Sửa lỗi biên dịch trước khi đánh giá từng test.
