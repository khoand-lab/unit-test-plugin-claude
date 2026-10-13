# Playbook: Java Spring Boot

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

## Detection

- `pom.xml` có parent `spring-boot-starter-parent`, dependency `spring-boot-starter-*` hoặc plugin `spring-boot-maven-plugin`.
- `build.gradle`/`build.gradle.kts` có plugin `org.springframework.boot` hoặc dependency `spring-boot-starter-*`.
- Build tool: có `pom.xml` → Maven; có `build.gradle*` → Gradle. Ưu tiên wrapper `./mvnw`/`mvnw.cmd`, `./gradlew`/`gradlew.bat` nếu có.
- Phiên bản Java: `<java.version>`, `maven.compiler.release`, `sourceCompatibility` hoặc `java { toolchain { ... } }`.

## Default toolchain

Nếu dự án chưa có test framework, đề xuất trong kế hoạch (cần chấp thuận):

- `spring-boot-starter-test` scope `test` (đã gồm JUnit 5, Mockito, AssertJ, Hamcrest).
- JUnit 5 (Jupiter) + Mockito + AssertJ.

Nếu dự án đã dùng:

- **JUnit 4** (`org.junit.Test`, `@RunWith`): viết test JUnit 4. Không trộn Jupiter vào module chỉ có JUnit 4 nếu chưa có `junit-vintage-engine`.
- **Hamcrest, Truth hoặc assertion của JUnit**: dùng đúng thư viện đó thay cho AssertJ.
- **Spock, Kotest**: stack không được hỗ trợ trong playbook này; báo người dùng.

## Test location and naming

- Vị trí: `src/test/java/<cùng package với class được test>/`.
- Tên file mặc định: `<ClassName>Test.java`. Theo quy ước dự án nếu khác (`*Tests`, `*UnitTest`, `Test*`).
- Kiểm tra `maven-surefire-plugin` `<includes>` hoặc `test { filter { ... } }` của Gradle. Tên file mới phải khớp pattern để được chạy.
- Tên test case: theo test mẫu, ví dụ `methodName_condition_expectedResult`, `shouldXxxWhenYyy`, hoặc `@DisplayName`.

## Test patterns

- **Service/component thuần:** không khởi động Spring context.
  - `@ExtendWith(MockitoExtension.class)` (JUnit 5) hoặc `@RunWith(MockitoJUnitRunner.class)` (JUnit 4).
  - `@Mock` cho dependency, `@InjectMocks` hoặc khởi tạo thủ công qua constructor.
- **Controller:** ưu tiên test class controller trực tiếp với dependency được mock. Chỉ dùng `@WebMvcTest` + `MockMvc` + `@MockBean`/`@MockitoBean` khi dự án đã dùng cách này.
- **Repository:** chỉ test query tùy biến bằng `@DataJpaTest` khi dự án đã dùng và đã có cơ sở dữ liệu nhúng (H2, HSQLDB) trong scope test. Không thêm Testcontainers hay kết nối DB thật. Nếu không, đưa repository vào "Đơn vị bỏ qua".
- `@WebMvcTest` và `@DataJpaTest` là test lát cắt, vẫn cô lập. **Không** dùng `@SpringBootTest` để viết unit test mới.
- Cấu trúc Arrange-Act-Assert. Dùng `@ParameterizedTest` + `@CsvSource`/`@ValueSource` cho các giá trị biên nếu dự án dùng JUnit 5.
- Exception: `assertThatThrownBy(...)`/`assertThrows(...)` và kiểm tra thêm kiểu cùng message hoặc thuộc tính.

## Mocking and isolation

- Mock mọi repository, client HTTP (`RestTemplate`, `WebClient`, Feign client), producer message, `JavaMailSender`.
- Thời gian:
  - class nhận `Clock` qua constructor thì truyền `Clock.fixed(...)`;
  - class gọi thẳng `LocalDateTime.now()`/`Instant.now()`: chỉ assert những gì không phụ thuộc giá trị thời gian, hoặc dùng `Mockito.mockStatic` nếu dự án có `mockito-inline` (mặc định trong Mockito 5). Nếu không kiểm soát được, ghi đơn vị vào "Đơn vị bỏ qua" kèm gợi ý tiêm `Clock`.
- `UUID.randomUUID()`, `Random`: xử lý như thời gian.
- Dùng `ArgumentCaptor` hoặc `verify(mock).method(eq(...))` để kiểm tra tương tác. Tránh `verifyNoMoreInteractions` nếu dự án không dùng.
- Class `static`/`final` hoặc tạo dependency bằng `new` bên trong: xem là khó test, không sửa production.
- Không đọc `application.yml` thật trong unit test; truyền giá trị cấu hình trực tiếp hoặc qua `ReflectionTestUtils.setField` khi field dùng `@Value` và dự án đã làm như vậy.

## Commands

Thay `mvn` bằng `./mvnw` (hoặc `mvnw.cmd` trên Windows), `gradle` bằng `./gradlew` (hoặc `gradlew.bat`) nếu có wrapper. Dự án nhiều module: thêm `-pl <module> -am` (Maven) hoặc `:<module>:test` (Gradle).

| Mục đích | Maven | Gradle |
|---|---|---|
| Một class | `mvn -q test -Dtest=OrderServiceTest -Dsurefire.failIfNoSpecifiedTests=false` | `gradle test --tests "com.example.OrderServiceTest"` |
| Một method | `mvn -q test -Dtest=OrderServiceTest#placeOrder_throwsWhenOutOfStock` | `gradle test --tests "com.example.OrderServiceTest.placeOrder_throwsWhenOutOfStock"` |
| Toàn bộ module | `mvn test` | `gradle test` |
| Coverage | `mvn test jacoco:report` | `gradle test jacocoTestReport` |

- Chạy lại cho mục phát hiện flaky: Gradle cần `--rerun-tasks` hoặc `cleanTest`, nếu không test sẽ bị bỏ qua vì up-to-date.
- Lỗi biên dịch nằm trong output của `maven-compiler-plugin`/`compileTestJava`. Kết quả chi tiết: `target/surefire-reports/` hoặc `build/test-results/test/`.
- **Thư mục/file sinh ra khi chạy:** `target/` (Maven); `build/`, `.gradle/` ở gốc dự án (Gradle). Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- Công cụ: JaCoCo.
- Maven: nếu `jacoco-maven-plugin` đã khai báo thì dùng; nếu chưa, có thể chạy không sửa `pom.xml`: `mvn org.jacoco:jacoco-maven-plugin:0.8.12:prepare-agent test org.jacoco:jacoco-maven-plugin:0.8.12:report`.
- Dự án có test cũ đang fail thì Maven dừng trước goal `report`. Thêm `-Dmaven.test.failure.ignore=true` để vẫn sinh báo cáo. Test cũ fail không phải lý do để ghi "không đo được".
- Gradle: cần plugin `jacoco` trong build script. Nếu chưa có, thêm plugin là thay đổi cấu hình cần chấp thuận; nếu không được chấp thuận thì ghi "không đo được".
- Đọc số liệu từ `target/site/jacoco/jacoco.csv` hoặc `build/reports/jacoco/test/jacocoTestReport.xml`, lọc theo class/package trong `scope`. Dùng line coverage và branch coverage.

## Disabling a test

- JUnit 5: `@Disabled("TODO(unit-test-agent): <lý do>")`.
- JUnit 4: `@Ignore("TODO(unit-test-agent): <lý do>")`.
- Đặt trên từng method test, không đặt trên cả class khi class còn test khác pass.

## Common pitfalls

- Trộn import JUnit 4 (`org.junit.Test`) và JUnit 5 (`org.junit.jupiter.api.Test`) trong cùng file: test không chạy hoặc bị bỏ qua mà không báo lỗi.
- Mockito strict stubs báo `UnnecessaryStubbingException`: bỏ stub thừa, không chuyển sang `lenient()` chỉ để test pass.
- `@InjectMocks` không tiêm được khi constructor có kiểu trùng nhau; khởi tạo thủ công.
- Lombok: getter/builder sinh tự động, dùng được trong test; không test chính code sinh bởi Lombok.
- `equals` của entity JPA dựa trên id: so sánh bằng field cụ thể thay vì `isEqualTo` cả object.
- `BigDecimal`: dùng `isEqualByComparingTo` thay vì `isEqualTo` (`2.0` khác `2.00` theo `equals`).
- Surefire cũ (< 2.22) không chạy test JUnit 5; kiểm tra phiên bản trước khi kết luận test "pass" vì 0 test chạy.
- Output "Tests run: 0" nghĩa là test chưa được chạy, không phải pass.
