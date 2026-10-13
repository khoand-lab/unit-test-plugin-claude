# Playbook: Android Java native

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

Playbook này chỉ dành cho **unit test chạy trên JVM** (`src/test`). Không viết instrumented test (`src/androidTest`), không chạy emulator hay thiết bị.

## Detection

- `build.gradle`/`build.gradle.kts` của module có plugin `com.android.application` hoặc `com.android.library` (xem `detection.md`, dòng 2), và mã nguồn chính là Java (`src/main/java/**/*.java`). Module chủ yếu Kotlin là stack không được hỗ trợ.
- Cấu trúc: `settings.gradle(.kts)` liệt kê module (`include ':app', ':core'`); mỗi module có `build.gradle(.kts)` và `src/main/AndroidManifest.xml`. Xét từng module trong `scope`.
- Phiên bản: Android Gradle Plugin (`com.android.application` version trong `plugins {}` của file gốc, hoặc `gradle/libs.versions.toml`), Gradle (`gradle/wrapper/gradle-wrapper.properties`), `compileSdk`, `minSdk`, `compileOptions { sourceCompatibility }`.
- Build variant: `buildTypes` (`debug`, `release`) và `productFlavors`. Task test có dạng `test<Flavor><BuildType>UnitTest`; không có flavor thì là `testDebugUnitTest`.
- Thư viện test có sẵn: `testImplementation` (unit test JVM) và `androidTestImplementation` (instrumented, không dùng). Xét `junit:junit` (JUnit 4), `org.junit.jupiter` (JUnit 5, cần plugin `de.mannodermaus.android-junit5` hoặc cấu hình `useJUnitPlatform()`), `org.mockito:mockito-core`, `org.robolectric:robolectric`, `androidx.test:core`, `androidx.arch.core:core-testing`, `com.google.truth:truth`, `org.assertj:assertj-core`.
- `testOptions { unitTests { ... } }`: `isReturnDefaultValues`/`returnDefaultValues = true` (API Android trả giá trị mặc định thay vì ném lỗi), `isIncludeAndroidResources` (cần cho Robolectric).
- Toolchain: JDK (AGP 8+ cần JDK 17), Android SDK (`local.properties` có `sdk.dir`, hoặc biến `ANDROID_HOME`/`ANDROID_SDK_ROOT`) với đúng `platforms/android-<compileSdk>`. Thiếu SDK thì không build được, kể cả unit test JVM.

## Default toolchain

Nếu module chưa có unit test JVM, đề xuất trong kế hoạch (cần chấp thuận):

- `testImplementation 'junit:junit:4.13.2'` (mặc định của template Android Studio) + `testImplementation 'org.mockito:mockito-core:<5.x>'`.
- Không thêm Robolectric, JUnit 5 hay thư viện assertion mới nếu chưa được duyệt riêng.

Nếu dự án đã có:

- Dùng đúng JUnit 4 hoặc JUnit 5 đang dùng; không trộn hai loại trong một module nếu chưa có `junit-vintage-engine`.
- Assertion theo test mẫu: `org.junit.Assert`, Truth, AssertJ hoặc Hamcrest.
- **Robolectric:** chỉ dùng khi dự án đã có trong `testImplementation`, và chỉ cho đơn vị bắt buộc cần framework Android thật (`Context`, `SharedPreferences`, resource). Test Robolectric vẫn là unit test JVM.

## Test location and naming

- Vị trí: `<module>/src/test/java/<cùng package với class được test>/`. Test theo variant đặt ở `src/testDebug/java/` chỉ khi dự án đã làm vậy.
- Tên file mặc định: `<ClassName>Test.java`. Theo quy ước dự án nếu khác (`*Tests`, `*UnitTest`).
- Tên method: theo test mẫu, ví dụ `methodName_condition_expectedResult`, `shouldXxxWhenYyy`, hoặc `@DisplayName` (JUnit 5).
- Không đặt test mới vào `src/androidTest`.

## Test patterns

- Cấu trúc Arrange-Act-Assert; mỗi test kiểm tra một hành vi.
- **Lớp Java thuần** (model, validator, use case, repository có dependency tiêm qua constructor): khởi tạo trực tiếp, mock dependency bằng Mockito (`@RunWith(MockitoJUnitRunner.class)` với JUnit 4, `@ExtendWith(MockitoExtension.class)` với JUnit 5, hoặc `Mockito.mock(...)`).
- **ViewModel (`androidx.lifecycle.ViewModel`) dùng `LiveData`:** cần `InstantTaskExecutorRule` (`androidx.arch.core:core-testing`) để `setValue`/`postValue` chạy đồng bộ. Nếu dự án chưa có thư viện này, thêm là thay đổi dependency cần chấp thuận; không được duyệt thì kiểm tra qua callback/đầu ra khác hoặc đưa vào "Đơn vị bỏ qua". Đọc giá trị bằng `getValue()` hoặc `observeForever` (gỡ observer ở cuối test).
- **Code bất đồng bộ** (`Executor`, `ExecutorService`, `Handler`): nếu executor được tiêm vào, truyền executor chạy đồng bộ (`Runnable::run`). Nếu class tự tạo thread hoặc `new Handler(Looper.getMainLooper())`: khó test, xem mục Mocking.
- **Activity, Fragment, View, Adapter, Service, BroadcastReceiver:** không viết unit test JVM thuần (cần framework Android). Chỉ test bằng Robolectric khi dự án đã dùng; nếu không, đưa vào "Đơn vị bỏ qua" và gợi ý tách logic ra lớp thuần/ViewModel.
- Exception: `assertThrows(...)` (JUnit 4.13+ và JUnit 5) và kiểm tra thêm message; không dùng `@Test(expected = ...)` cho test mới trừ khi test mẫu dùng.
- Giá trị biên: JUnit 4 dùng `@RunWith(Parameterized.class)` hoặc nhiều method riêng theo test mẫu; JUnit 5 dùng `@ParameterizedTest`.

## Mocking and isolation

- API Android trong unit test JVM là bản stub: gọi `TextUtils.isEmpty`, `Log.d`, `Uri.parse`, `SystemClock`… sẽ ném `RuntimeException("Method ... not mocked.")`, trừ khi bật `returnDefaultValues` (khi đó trả `null`/`0`/`false`, dễ cho kết quả sai). Không tự bật cờ này: là thay đổi cấu hình cần chấp thuận. Ưu tiên test đơn vị không gọi API Android; đơn vị gọi API Android mà không tách được thì đưa vào "Đơn vị bỏ qua" (hoặc dùng Robolectric nếu dự án đã có).
- `Context`, `SharedPreferences`, `Resources`: mock bằng Mockito (`mock(Context.class)`, `when(context.getSharedPreferences(...)).thenReturn(prefs)`) nếu đơn vị nhận chúng qua constructor/tham số.
- Mạng: mock interface API (Retrofit service, client tự viết). Có `MockWebServer` (`okhttp3.mockwebserver`) thì chỉ dùng khi dự án đã có; không mở kết nối thật.
- Database (Room DAO, `SQLiteDatabase`): mock DAO/repository; không tạo database thật trong unit test JVM.
- Thời gian: đơn vị nhận `Clock`/`Supplier<Long>`/`TimeProvider` thì truyền giá trị cố định. Đơn vị gọi thẳng `System.currentTimeMillis()`/`new Date()`: chỉ assert phần không phụ thuộc thời gian, hoặc `Mockito.mockStatic` (Mockito 5 có sẵn inline mock maker) khi dự án đã dùng; nếu không, đưa vào "Đơn vị bỏ qua" kèm gợi ý tiêm `Clock`.
- Class `final` hoặc `static`: Mockito 5 mock được class `final`; vẫn ưu tiên mock interface.
- Singleton tạo dependency bên trong (`getInstance()`, `new` trong method): xem là khó test, không sửa production.

## Commands

Luôn dùng Gradle wrapper của dự án: `./gradlew` (macOS/Linux) hoặc `gradlew.bat` (Windows). Thay `app` bằng tên module, `Debug` bằng variant đúng.

| Mục đích | Lệnh |
|---|---|
| Một class | `./gradlew :app:testDebugUnitTest --tests "com.example.shop.discount.DiscountCalculatorTest"` |
| Một method | `./gradlew :app:testDebugUnitTest --tests "com.example.shop.discount.DiscountCalculatorTest.rate_gold_returns20Percent"` |
| Toàn bộ module | `./gradlew :app:testDebugUnitTest` |
| Coverage | xem mục Coverage |

- Chỉ chạy task unit test của **một** variant; `./gradlew test` chạy mọi variant và chậm gấp nhiều lần.
- Không chạy `connectedAndroidTest`, `connectedCheck`, `assemble*` hay task cần thiết bị.
- Chạy lại để phát hiện flaky: thêm `--rerun-tasks` hoặc `--rerun`, nếu không task bị bỏ qua vì up-to-date.
- Lần build đầu có thể tải dependency và mất vài phút; tăng thời gian chờ thay vì kết luận là treo. Nếu không có mạng và cache trống thì ghi "không chạy được" kèm lý do.
- Kết quả: dòng `BUILD SUCCESSFUL`/`BUILD FAILED` và `N tests completed, M failed`; chi tiết ở `<module>/build/test-results/test<Variant>UnitTest/*.xml` và `build/reports/tests/test<Variant>UnitTest/index.html`. Lỗi biên dịch nằm ở task `compile<Variant>UnitTestJavaWithJavac`.
- **Thư mục/file sinh ra khi chạy:** `.gradle/` và `build/` ở gốc dự án, `app/build/` (và `<module>/build/` của mọi module đã chạy). Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- Công cụ: JaCoCo.
  - AGP 8+: `android { buildTypes { debug { enableUnitTestCoverage = true } } }` rồi chạy `./gradlew :app:createDebugUnitTestCoverageReport`; báo cáo ở `<module>/build/reports/coverage/test/debug/report.xml` (và `html/`).
  - Dự án có task JaCoCo tự viết (`jacocoTestReport`, `jacocoDebugUnitTestReport`) thì dùng task đó.
- Nếu chưa bật: bật `enableUnitTestCoverage` là thay đổi cấu hình cần chấp thuận; không được duyệt thì ghi "không đo được".
- Đọc line coverage và branch coverage từ `report.xml` (thẻ `<counter type="LINE">`, `<counter type="BRANCH">`), lọc theo class/package trong `scope`. Loại khỏi mẫu số các class sinh tự động (`R`, `BuildConfig`, `*_Impl`, `Hilt_*`, `*_Factory`).
- Khi có test fail, task coverage có thể không chạy; thêm `--continue` hoặc ghi lý do không đo được.
- Thư mục `build/` là output của Gradle; không cần xóa nhưng không được đưa vào danh sách file thay đổi.

## Disabling a test

- JUnit 4: `@Ignore("TODO(unit-test-agent): <lý do>")` trên từng method.
- JUnit 5: `@Disabled("TODO(unit-test-agent): <lý do>")` trên từng method.
- Không đặt trên cả class khi class còn test khác pass.

## Common pitfalls

- `RuntimeException: Method ... in android.... not mocked`: đơn vị gọi API Android trong unit test JVM. Không tự bật `returnDefaultValues`; xem mục Mocking.
- `LiveData` không cập nhật hoặc ném lỗi `Looper`: thiếu `InstantTaskExecutorRule`.
- Chạy `./gradlew test` (mọi variant) thay vì `testDebugUnitTest`: chậm và có thể fail ở variant `release` do cấu hình ký.
- Thiếu `sdk.dir`/`ANDROID_HOME` hoặc thiếu `platforms/android-<compileSdk>`: build fail trước khi chạy test. Báo thiếu toolchain, không đổi `compileSdk`.
- JDK không khớp (AGP 8+ cần JDK 17+): lỗi "Android Gradle plugin requires Java 17". Báo người dùng, không sửa `gradle.properties` (`org.gradle.java.home`).
- Test JUnit 5 trong module chỉ cấu hình JUnit 4: Gradle báo "0 tests" hoặc bỏ qua; kiểm tra số test đã chạy.
- `--tests` không khớp tên class: lỗi "No tests found for given includes" — nghĩa là test chưa được chạy, không phải pass.
- Mockito strict stubs (`MockitoJUnitRunner.StrictStubs` hoặc `MockitoExtension`) báo stub thừa: bỏ stub thừa, không chuyển sang `lenient()` chỉ để test pass.
