# Nhận diện stack

Dùng bảng này ở pha phân tích để xác định stack của từng module trong `scope` và chọn playbook tương ứng.

## Tìm module

1. Tìm các file manifest nằm trong `scope`, hoặc ở thư mục cha gần nhất nếu `scope` là một file hoặc thư mục con. Các manifest cần tìm:
   - `pom.xml`, `build.gradle`, `build.gradle.kts`
   - `package.json`
   - `pyproject.toml`, `requirements*.txt`, `setup.py`, `setup.cfg`, `Pipfile`
   - `*.xcodeproj`, `*.xcworkspace`, `Package.swift`
   - `pubspec.yaml`
2. Mỗi thư mục chứa manifest là **một module**.
   - Monorepo có nhiều module thì nhận diện stack riêng cho từng module nằm trong `scope`.
   - Bỏ qua manifest nằm trong thư mục phụ thuộc hoặc thư mục build: `node_modules/`, `.venv/`, `venv/`, `build/`, `target/`, `dist/`, `Pods/`, `.dart_tool/`.
3. Module Gradle nhiều cấp (root có `settings.gradle`): mỗi module con có `build.gradle` riêng là một module. `build.gradle` gốc chỉ chứa cấu hình chung thì không phải module.

## Bảng dấu hiệu

Xét theo **thứ tự ưu tiên** từ trên xuống. Dấu hiệu đầu tiên khớp quyết định stack của module.

| Ưu tiên | Dấu hiệu | Stack | Playbook |
|---|---|---|---|
| 1 | `pubspec.yaml` có dependency `flutter` | Flutter | `playbooks/flutter.md` |
| 2 | `build.gradle`/`build.gradle.kts` có plugin `com.android.application` hoặc `com.android.library`, mã nguồn chính là Java | Android Java native | `playbooks/android-java.md` |
| 3 | Có `*.xcodeproj`/`*.xcworkspace`, hoặc `Package.swift`, mã nguồn là Swift | iOS Swift | `playbooks/ios-swift.md` |
| 4 | `pom.xml`/`build.gradle`/`build.gradle.kts` có `spring-boot` (plugin, parent hoặc starter) | Java Spring Boot | `playbooks/java-spring-boot.md` |
| 5 | `package.json` có `@angular/core` trong dependencies | Angular | `playbooks/angular.md` |
| 6 | `package.json` có `vue` trong dependencies (kể cả qua `nuxt`) | Vue | `playbooks/vue.md` |
| 7 | `package.json` có `react` trong dependencies (kể cả qua `next`) và không phải React Native | React | `playbooks/react.md` |
| 8 | `package.json` không khớp các dòng 5–7 | Node.js | `playbooks/nodejs.md` |
| 9 | Có `pyproject.toml`, `requirements*.txt`, `setup.py`, `setup.cfg` hoặc `Pipfile` | Python | `playbooks/python.md` |

Ghi chú:

- Frontend (Angular, Vue, React) luôn xét trước Node.js, vì mọi dự án frontend đều có `package.json`.
- Flutter xét trước Android và iOS, vì dự án Flutter có sẵn thư mục `android/` và `ios/`. Hai thư mục đó thuộc module Flutter, không phải module riêng.
- Dự án Android có mã nguồn chủ yếu là Kotlin, React Native, Ionic và Capacitor là **stack không được hỗ trợ**.
- Java/Maven/Gradle không có Spring Boot là **stack không được hỗ trợ** ở phiên bản này.
  - Báo cho người dùng.
  - Nếu người dùng đồng ý, có thể dùng `playbooks/java-spring-boot.md` cho phần JUnit/Mockito thuần. Ghi lựa chọn này vào kế hoạch.

## Không nhận diện được

Nếu không có dấu hiệu nào khớp, hoặc stack nằm ngoài danh sách:

- báo rõ cho người dùng những gì đã tìm thấy;
- không sinh test theo phỏng đoán;
- chỉ tiếp tục khi người dùng chỉ định rõ stack **và** stack đó có playbook.
