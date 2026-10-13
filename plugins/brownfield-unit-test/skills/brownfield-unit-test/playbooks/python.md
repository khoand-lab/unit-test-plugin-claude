# Playbook: Python

Tên các mục giữ bằng tiếng Anh để công cụ kiểm tra cấu trúc nhận diện. Nội dung ưu tiên thấp hơn quy ước thực tế của dự án: dự án đã dùng công cụ nào thì theo công cụ đó.

## Detection

- Có một trong các file: `pyproject.toml`, `setup.py`, `setup.cfg`, `requirements*.txt`, `Pipfile`, `poetry.lock`, `uv.lock` (xem `detection.md`).
- Test framework đang dùng, xét theo thứ tự:
  - `pytest` trong dependency dev (`[project.optional-dependencies]`, `[dependency-groups]`, `[tool.poetry.group.dev.dependencies]`, `requirements-dev.txt`, `Pipfile [dev-packages]`);
  - cấu hình `[tool.pytest.ini_options]` trong `pyproject.toml`, `pytest.ini`, `[tool:pytest]` trong `setup.cfg`, `tox.ini`, `conftest.py`;
  - test mẫu `import unittest` và class kế thừa `unittest.TestCase`, không dùng fixture pytest: dự án dùng `unittest`;
  - `tox.ini`/`noxfile.py`: đọc lệnh test trong đó.
- Trình quản lý môi trường, xét theo thứ tự:
  - `uv.lock` → uv;
  - `poetry.lock` hoặc `[tool.poetry]` → Poetry;
  - `Pipfile.lock` → Pipenv;
  - `pdm.lock` → PDM;
  - còn lại: pip với virtualenv. Tìm venv sẵn có ở `.venv/`, `venv/`, `env/`.
- Phiên bản Python: `requires-python`, `.python-version`, `python_requires`. So với phiên bản của interpreter sẽ dùng.
- Framework ứng dụng (Django, Flask, FastAPI) qua dependency. Django có `manage.py` và `DJANGO_SETTINGS_MODULE`, nên xem mục Test patterns.

## Default toolchain

Nếu dự án chưa có test framework, đề xuất trong kế hoạch (cần chấp thuận): `pytest`, `pytest-cov`, và thêm `pytest-mock` nếu cần. Chúng được thêm vào nhóm dependency dev theo đúng trình quản lý của dự án.

Nếu dự án đã dùng:

- **pytest:** viết test pytest, dùng fixture và `conftest.py` sẵn có. Không thêm plugin mới (`pytest-mock`, `freezegun`, `pytest-asyncio`, `responses`...) nếu dự án chưa có. Thay vào đó, dùng `unittest.mock` và `monkeypatch` có sẵn.
- **unittest:** viết class `unittest.TestCase`, dùng `self.assert*` và `unittest.mock`. Không chuyển sang pytest, kể cả khi pytest có thể chạy được test unittest.
- **Django:** dùng `django.test.SimpleTestCase` cho unit test không cần DB. Nếu dự án đã có `pytest-django`, dùng cách viết của nó. Test cần DB là integration test, nằm ngoài phạm vi.

## Test location and naming

- Theo vị trí test mẫu. Hai kiểu phổ biến:
  - thư mục riêng: `tests/` lặp lại cấu trúc package, ví dụ `src/shop/discount/service.py` → `tests/discount/test_service.py`;
  - cạnh mã nguồn: `shop/discount/tests/test_service.py`.
- Tên file `test_*.py` hoặc `*_test.py` theo test mẫu. Kiểm tra `python_files`, `python_classes`, `python_functions` và `testpaths` trong cấu hình pytest. File mới phải khớp để được thu thập.
- Chế độ import:
  - Nếu thư mục test có `__init__.py`, file mới trong thư mục con cũng cần `__init__.py`.
  - Nếu không có, hai file test trùng tên ở hai thư mục sẽ xung đột với import mode mặc định. Đặt tên khác nhau, hoặc làm theo `--import-mode` mà dự án dùng.
- Cách import đơn vị: theo test mẫu (`from shop.discount.service import ...`). Chú ý layout `src/`: dự án có thể cài editable (`pip install -e .`) hoặc đặt `pythonpath = ["src"]` trong cấu hình pytest. Không tự sửa `sys.path` trong file test.
- Tên test: theo test mẫu, ví dụ `test_<hàm>_<tình huống>_<kết quả>`, hoặc class `Test<Đơn vị>` gom các hàm test.

## Test patterns

- Cấu trúc Arrange-Act-Assert; mỗi hàm test kiểm tra một hành vi.
- pytest:
  - assert thuần (`assert result == expected`);
  - lỗi: `with pytest.raises(KiểuLỗi, match="...")`;
  - nhiều bộ dữ liệu: `@pytest.mark.parametrize` và `ids=` để tên dễ đọc.
- unittest:
  - `self.assertEqual`, `self.assertRaises` (dùng làm context manager);
  - nhiều bộ dữ liệu: `self.subTest`.
- Số thực: `pytest.approx(...)` hoặc `self.assertAlmostEqual`. Code dùng `Decimal` thì so sánh bằng `Decimal("...")` tạo từ chuỗi, không tạo từ `float`.
- Hàm `async`:
  - dùng `pytest-asyncio` hoặc `anyio` nếu dự án đã có, đánh dấu theo cách dự án làm (`@pytest.mark.asyncio`, hoặc `asyncio_mode = auto`);
  - nếu không có plugin, gọi `asyncio.run(...)` trong test sync;
  - với unittest: dùng `unittest.IsolatedAsyncioTestCase`.
- Flask/FastAPI:
  - ưu tiên test service/hàm thuần;
  - test route chỉ khi dự án đã làm, qua `app.test_client()` hoặc `fastapi.testclient.TestClient`, với dependency đã được override;
  - không kết nối DB hay mạng thật.
- Dataclass/Pydantic: so sánh cả object, hoặc so từng trường có ý nghĩa. Không assert `repr`.

## Mocking and isolation

- Ưu tiên tiêm dependency giả qua tham số hoặc constructor. `unittest.mock.Mock(spec=Lớp)` hoặc `create_autospec(Lớp)` giúp bắt lỗi gọi sai tên method.
- Patch đúng chỗ: `patch("<module dùng tên đó>.<tên>")`, không patch nơi tên đó được định nghĩa. Ví dụ: `service.py` có `from datetime import datetime`, thì patch `shop.discount.service.datetime`.
- Cách patch theo công cụ sẵn có:
  - `pytest-mock` nếu dự án có: fixture `mocker.patch(...)`;
  - pytest không có plugin: `monkeypatch.setattr(...)`, hoặc `with patch(...)`;
  - unittest: decorator `@patch` hoặc `with patch(...)`.
- Mock mọi truy cập ra ngoài:
  - HTTP (`requests`, `httpx`, `urllib`): patch hàm gọi, hoặc dùng `responses`/`respx` nếu dự án đã có;
  - DB và ORM session, file, `subprocess`, email, message queue.
- File: ưu tiên fixture `tmp_path` (pytest) hoặc `tempfile.TemporaryDirectory()`, thay vì mock `open`. Chỉ được dùng khi đơn vị cho phép truyền đường dẫn vào.
- Thời gian:
  - Đơn vị nhận `now`/`clock` qua tham số: truyền giá trị cố định.
  - Có `freezegun` hoặc `time-machine` trong dự án: dùng chúng.
  - Không có cả hai: patch `datetime`/`time.time` tại module đang dùng. Chú ý `datetime.datetime` là kiểu C, không `setattr` trực tiếp được; phải patch cả tên `datetime` trong module đó bằng một `Mock` có `now.return_value`.
- Ngẫu nhiên, UUID: patch `random.random`/`uuid.uuid4` tại module đang dùng, hoặc `random.seed` nếu đơn vị dùng module `random` toàn cục.
- Biến môi trường: `monkeypatch.setenv`/`delenv`, hoặc `patch.dict(os.environ, {...})`. Không đọc file `.env` thật.
- Module mở kết nối hoặc đọc file ngay khi import (side effect ở cấp module): xem là khó test nếu không patch được trước khi import. Không sửa production.

## Commands

Tiền tố chạy lệnh theo trình quản lý môi trường:

| Trình quản lý | Tiền tố |
|---|---|
| uv | `uv run` |
| Poetry | `poetry run` |
| Pipenv | `pipenv run` |
| PDM | `pdm run` |
| venv | `<venv>/bin/python -m` (Windows: `<venv>\Scripts\python -m`) |

- Luôn gọi `python -m pytest` thay vì `pytest`, để chắc chắn dùng đúng interpreter và thêm thư mục hiện tại vào `sys.path` như dự án mong đợi.
- Môi trường chưa có:
  - uv/Poetry/Pipenv tự tạo khi `uv sync`/`poetry install`/`pipenv install --dev`;
  - pip: `python -m venv .venv` rồi `pip install -r requirements-dev.txt`, hoặc `pip install -e ".[dev]"`, theo README.
  - Việc tạo môi trường phải ghi vào báo cáo. Nếu `.venv/` không có từ đầu phiên, đưa vào danh sách dọn dẹp, trừ khi người dùng muốn giữ.

| Mục đích | pytest | unittest |
|---|---|---|
| Một file | `python -m pytest tests/discount/test_service.py` | `python -m unittest tests.discount.test_service` |
| Một test | `python -m pytest tests/discount/test_service.py::test_ten` (hoặc `-k "ten"`) | `python -m unittest tests.discount.test_service.TestX.test_ten` |
| Toàn bộ | `python -m pytest` | `python -m unittest discover -s tests` |
| Coverage | `python -m pytest --cov=<package>` | `python -m coverage run -m unittest discover` |

- Thêm `-q -p no:cacheprovider` để pytest không tạo `.pytest_cache/`. Nếu dự án đã có thư mục đó thì không cần.
- Kết quả:
  - đọc dòng tổng kết (`N passed, M failed, K skipped`);
  - `no tests ran` hoặc `collected 0 items` nghĩa là test chưa được thu thập;
  - `ERROR collecting` là lỗi import hoặc cú pháp.
- Không dùng `pytest -x` khi đo trạng thái ban đầu, vì nó dừng ở lỗi đầu tiên và làm sai số liệu.
- **Thư mục/file sinh ra khi chạy:** `__pycache__/` (ở mọi cấp thư mục, cả trong `src/` và `tests/`), `.pytest_cache/`, `.coverage`, `coverage.json`, `htmlcov/`; `.venv/` nếu bạn tự tạo. Ghi nhận cái nào đã có từ đầu phiên. Cái nào không có từ đầu thì phải xóa trước khi kết thúc, và cả trước khi dừng chờ xác nhận ở `mode=confirm` (pha 2). Liệt kê lại thư mục gốc để chắc không còn sót.

## Coverage

- Công cụ: `coverage.py`, gọi qua `pytest-cov` (`--cov`) hoặc `coverage run`. Nếu dự án chưa có cả hai, thêm là thay đổi dependency cần chấp thuận. Không được chấp thuận thì ghi "không đo được".
- Đo không sửa cấu hình:
  - `python -m pytest --cov=<package hoặc thư mục theo scope> --cov-branch --cov-report=term --cov-report=json:coverage.json`;
  - hoặc `coverage run --branch --source=<...> -m pytest`, rồi `coverage json -o <...>`.
- Đọc `totals.percent_covered` và số liệu theo từng file trong `files` của JSON, lọc theo `scope`. Dùng line coverage và branch coverage.
- `coverage.py` chỉ tính file đã được import. Muốn tính cả file chưa có test, dùng `--cov=<package>`/`--source=<package>` (không dùng đường dẫn file), để coverage.py tìm các module chưa import trong package đó.
- File `.coverage`, `coverage.json`, `htmlcov/` sinh ra trong phiên phải được xóa nếu dự án không có từ đầu. Không ghi ra ngoài dự án (xem pha 1, mục 7).

## Disabling a test

- pytest: `@pytest.mark.skip(reason="TODO(unit-test-agent): <lý do>")`.
- unittest: `@unittest.skip("TODO(unit-test-agent): <lý do>")`.
- Không dùng `xfail` để disable: `xfail` vẫn chạy test và có thể thành `XPASS`, gây nhầm lẫn.
- Đặt trên từng hàm/method test, không skip cả class hay module khi còn test khác pass.

## Common pitfalls

- Patch sai chỗ, tức patch nơi định nghĩa thay vì nơi dùng: mock không có tác dụng và code thật chạy. Kiểm tra mock được gọi (`assert_called_once_with`).
- `Mock()` không có `spec` chấp nhận mọi thuộc tính, nên test pass kể cả khi gọi sai tên method. Dùng `spec`/`autospec`.
- `assert mock.called_once_with(...)` (thiếu `assert_`) luôn đúng. Phải dùng `mock.assert_called_once_with(...)`.
- Hàm `async` được gọi mà không `await`: test nhận về coroutine và pass sai, kèm cảnh báo "coroutine was never awaited".
- `pytest.raises` bọc quá nhiều dòng, nên lỗi đến từ dòng arrange mà test vẫn pass. Chỉ bọc lời gọi đang kiểm tra, và dùng `match=`.
- So sánh số thực bằng `==`; tạo `Decimal` từ `float` (`Decimal(0.1)`) thay vì từ chuỗi.
- Fixture có `scope="module"`/`"session"` giữ trạng thái giữa các test. Dữ liệu có thể bị sửa thì dùng scope mặc định (`function`).
- Chạy nhầm interpreter hệ thống thay vì venv của dự án: báo `ModuleNotFoundError` dù dependency đã cài.
- `collected 0 items` không phải là pass: kiểm tra `testpaths`, tên file và tên hàm có khớp cấu hình.
- Lần chạy đầu sinh `__pycache__/` và `.pytest_cache/`. Đây là file tạm của interpreter, không phải file dự án, nhưng vẫn phải kiểm tra thư mục chúng nằm trong `.gitignore` hoặc dọn đi nếu dự án không có từ đầu.
