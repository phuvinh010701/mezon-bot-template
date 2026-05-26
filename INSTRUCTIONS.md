# Mezon Python Bot Instruction

Tài liệu này tổng hợp cấu trúc và quy ước của repo gốc `mezon-bot-template` để dùng làm nền khi viết một bot Mezon Python khác. Mục tiêu là giữ lại phần hạ tầng đã ổn định như FastAPI lifespan, Mezon SDK, dependency injection, async SQLAlchemy, handler command routing, interactive forms, migration và test pattern; chỉ thay domain nghiệp vụ.

## 1. Nguyên tắc khi fork repo

- Giữ Python `3.13` và dùng `uv sync` để cài dependency theo `uv.lock`.
- Giữ entrypoint `run.py -> app.main:app`; app chạy bằng `python run.py --reload`.
- Giữ FastAPI làm host/lifecycle cho bot, không đặt logic nghiệp vụ trong route HTTP.
- Giữ `dependency-injector` làm ranh giới đăng ký repository, service, Mezon client và handler.
- Giữ mô hình async end-to-end: Mezon callback, service, repository và SQLAlchemy đều async.
- Thay domain bằng cách thêm/sửa model, repository, service, handler; không viết logic trực tiếp trong `app/main.py`.
- Với generated files như file export, ghi vào `exports/` và không commit trừ khi có yêu cầu.
- Không dựa vào `mezon_bot_template.egg-info/`; đây là output packaging bị ignore.

## 2. Runtime và cấu hình

Repo dùng `pydantic-settings` trong `app/core/settings/app.py`. `.env` được load với `override=True`, nên biến trong `.env` sẽ override môi trường hiện tại khi app khởi động.

Các biến bắt buộc:

- `APP_ENV`: dùng `dev` để bật OpenAPI/docs, `prod` để tắt docs.
- `DB_URI`: connection string dạng `postgresql+asyncpg://user:pass@host:port/db`.
- `MEZON_CLIENT_ID`: client id của bot.
- `MEZON_API_KEY`: API key của bot.

Các biến nên giữ:

- `MEZON_BOT_REQUIRE_MENTION=false`: nếu `true`, bot chỉ xử lý message có mention bot.
- `S3_ENDPOINT_URL`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_BUCKET_NAME`, `S3_REGION`, `S3_PUBLIC_URL_BASE`: dùng khi bot cần upload file qua S3-compatible storage.


## 3. Luồng khởi động ứng dụng

Giữ cấu trúc trong `app/main.py`:

1. Tạo `Container()`.
2. Wire container cho API modules.
3. Lấy `mezon_client` từ container.
4. `await mezon_client.login()`.
5. Lấy `handler_manager` từ container.
6. Đăng ký callback:
   - `mezon_client.on_channel_message(handler_manager.handle_channel_message)`
   - `mezon_client.on_message_button_clicked(handler_manager.handle_button_click)`
7. Lưu container/client/handler manager vào `app.state`.
8. Khi shutdown thì `await mezon_client.disconnect()`.

Không nên đăng ký handler hoặc khởi tạo service thủ công trong `main.py`; mọi dependency nên đi qua `app/dependencies/container.py`.

## 4. Dependency Injection Container

`app/dependencies/container.py` là nơi đăng ký chính. Khi thêm domain mới, làm đủ các bước:

1. Tạo SQLAlchemy model trong `app/database/models/<domain>.py`.
2. Export model trong `app/database/models/__init__.py` để Alembic autogenerate thấy metadata.
3. Tạo repository trong `app/database/repositories/<domain>.py`.
4. Tạo service trong `app/services/<domain>/service.py`.
5. Tạo handler trong `app/services/bot/handlers/<domain>.py`.
6. Đăng ký repository/service/handler trong `Container`.
7. Thêm handler vào `handler_manager = providers.Singleton(... handlers=providers.List(...))`.

Pattern đăng ký:

```python
domain_repository = providers.Singleton(
    DomainRepository,
    session_factory=db_session_factory,
)

domain_service = providers.Singleton(
    DomainService,
    domain_repository=domain_repository,
)

domain_handler = providers.Singleton(
    DomainHandler,
    client=mezon_client,
    domain_service=domain_service,
)

handler_manager = providers.Singleton(
    HandlerManager,
    handlers=providers.List(
        expert_handler,
        program_handler,
        domain_handler,
    ),
    client_id=app_settings.mezon_client_id,
    require_mention=app_settings.mezon_bot_require_mention,
)
```

## 5. Command Handler Pattern

Command được khai báo bằng decorator `@command(...)` trong `app/services/bot/handlers/base.py`.

Ví dụ handler tối thiểu:

```python
from typing import Any

from app.services.bot.handlers.base import BaseMessageHandler, command


class TaskHandler(BaseMessageHandler):
    def __init__(self, client, task_service):
        super().__init__(client)
        self.task_service = task_service

    @command("*task")
    async def handle_task(
        self,
        message: Any,
        subcmd: str = None,
        rest: str = None,
    ) -> None:
        if not subcmd:
            await self.reply_message(message, "Dùng `*task list` hoặc `*task add`.")
            return

        if subcmd == "list":
            await self._handle_list(message)
        elif subcmd == "add":
            await self._handle_add(message)
        else:
            await self.reply_message(message, f"Lệnh con không hợp lệ: `{subcmd}`")
```

Quy tắc parse argument:

- Handler manager lấy token đầu tiên làm command, ví dụ `*expert`.
- Các tham số sau `self, message` được parse từ nội dung message.
- Tham số cuối kiểu `str` sẽ nhận toàn bộ phần còn lại.
- `int` và `float` consume đúng một token.
- Tham số có default là optional; tham số không default là required.

Ví dụ:

```python
@command("*ask")
async def handle_ask(self, message, prompt: str) -> None:
    ...
```

Message `*ask hello world` sẽ truyền `prompt="hello world"`.

Mention mode:

- Nếu `MEZON_BOT_REQUIRE_MENTION=true`, message không mention bot sẽ bị bỏ qua.
- Khi có mention, alias không prefix được hỗ trợ: `@Bot expert list` có thể map sang `*expert`.
- Prefix được strip bằng tập ký tự `*!/@`.

## 6. Button Click và Interactive Forms

Bot dùng Mezon interactive components:

- `InteractiveBuilder` để tạo form/embed.
- `ButtonBuilder` để tạo button.
- `MessageActionRow` và `MessageComponent` để gửi components.
- `event.extra_data` chứa dữ liệu form, parse qua `form_tracker.parse_extra_data(event.extra_data)`.

Pattern form:

```python
form = InteractiveBuilder("Tạo item")
form.set_description("Điền thông tin bên dưới")
form.set_color("#5865F2")
form.add_input_field("name", "Tên", placeholder="Tên item")

await self.reply_message(
    message,
    "Vui lòng điền form:",
    embeds=[InteractiveMessageProps(**form.build())],
    components=self._build_buttons([
        ("save_item", "Lưu", ButtonMessageStyle.SUCCESS),
        ("cancel", "Hủy", ButtonMessageStyle.DANGER),
    ]),
)
```

Pattern xử lý button:

```python
async def handle_button_click(self, event) -> None:
    extra_data = form_tracker.parse_extra_data(event.extra_data)

    if event.button_id == "cancel":
        await self.edit_message(event.channel_id, event.message_id, "Đã hủy.", components=[])
        return

    if event.button_id == "save_item":
        await self._handle_save_item(event, extra_data)
        return

    if event.button_id.startswith("edit_item:"):
        item_id = int(event.button_id.split(":")[1])
        await self._handle_edit_item(event, item_id, extra_data)
        return
```

Gotcha hiện tại:

- `HandlerManager.handle_button_click()` đang route button click bằng điều kiện `isinstance(h, (ExpertHandler, ProgramHandler))`.
- Khi thêm handler mới có button, nên refactor thành capability-based:

```python
for handler, _info in self._command_map.values():
    if hasattr(handler, "handle_button_click"):
        await handler.handle_button_click(event)
```

Hoặc thêm handler mới vào tuple `isinstance`, nhưng cách đó kém mở rộng.

## 7. Service Layer Pattern

Service nên là nơi chứa rule nghiệp vụ, validation nghiệp vụ, mapping ORM -> DTO. Handler chỉ nên điều phối input/output.

Pattern DTO:

```python
from dataclasses import dataclass
from typing import Optional


@dataclass
class TaskData:
    id: int
    title: str
    description: Optional[str] = None
```

Pattern service:

```python
class TaskService:
    def __init__(self, task_repository):
        self._repository = task_repository

    async def create_task(self, data: TaskData) -> TaskData:
        task = Task(title=data.title, description=data.description)
        created = await self._repository.create(task)
        return self._to_data(created)

    def _to_data(self, task: Task) -> TaskData:
        return TaskData(
            id=task.id,
            title=task.title,
            description=task.description,
        )
```

Nên giữ các rule như:

- Normalize code trước khi lưu và lookup.
- Check duplicate trong service trước khi create.
- Tính toán tổng tiền/trạng thái trong service hoặc repository method chuyên dụng.
- Không để handler tự mutate ORM model.

## 8. Repository và Database Pattern

DB dùng async SQLAlchemy:

- Engine trong `app/database/connect.py`.
- `async_session_factory` là `async_scoped_session` scope theo `asyncio.current_task`.
- Repository nhận `session_factory`.
- Mỗi repository method mở session riêng bằng `async with self._get_session() as session`.
- Create/update/delete commit trong repository.
- Soft-delete dùng `deleted_at`.

Pattern repository:

```python
from typing import Optional, List
from sqlalchemy import select

from app.database.repositories.base import BaseRepository
from app.database.models.task import Task


class TaskRepository(BaseRepository):
    async def create(self, task: Task) -> Task:
        async with self._get_session() as session:
            session.add(task)
            await session.commit()
            await session.refresh(task)
            return task

    async def get_by_id(self, task_id: int) -> Optional[Task]:
        async with self._get_session() as session:
            result = await session.execute(
                select(Task).where(Task.id == task_id, Task.deleted_at.is_(None))
            )
            return result.scalars().first()

    async def list_all(self, limit: int = 50) -> List[Task]:
        async with self._get_session() as session:
            result = await session.execute(
                select(Task)
                .where(Task.deleted_at.is_(None))
                .order_by(Task.created_at.desc())
                .limit(limit)
            )
            return result.scalars().all()
```

Pattern model:

```python
from sqlalchemy import Column, Integer, String, Index

from app.database.models.common import DateTimeModelMixin
from app.database.models.rwmodel import RWModel


class Task(RWModel, DateTimeModelMixin):
    __tablename__ = "tasks"

    id = Column(Integer, primary_key=True, autoincrement=True)
    title = Column(String(200), nullable=False)
    description = Column(String(1000), nullable=True)

    __table_args__ = (
        Index("ix_tasks_title", "title"),
    )
```

Sau khi thêm model:

```bash
alembic revision --autogenerate -m "add tasks table"
alembic upgrade head
```

Lưu ý:

- `alembic.ini` dùng migration path `app/database/migrations`.
- Alembic URL lấy từ `app_settings.db_uri`.
- Phải import/export model trong `app/database/models/__init__.py` để autogenerate thấy.
- Nên bỏ debug `print(DATABASE_URI, "============")` trong `app/database/migrations/env.py` nếu fork làm production sạch.

## 9. API HTTP

HTTP API chỉ nên dùng cho health/status/admin endpoint nhẹ:

- `GET /api/v1/health`: trả status/version.
- `GET /api/v1/bot/status`: ví dụ inject `MezonBotService`.

Khi thêm route mới:

1. Tạo file trong `app/api/v1/`.
2. Đăng ký router trong `app/api/v1/__init__.py`.
3. Nếu route cần dependency injection, thêm module vào `Container.wiring_config.modules`.

Không nên xử lý command bot qua HTTP route nếu Mezon SDK callback đã xử lý được.

## 10. File Export và Upload

- Upload ưu tiên `S3UploadService` nếu configured, fallback `self.client.upload_file`.

Khi bot mới cần export file:

- Tạo service riêng, ví dụ `app/services/report_export/__init__.py`.
- Không render file trong handler; handler chỉ gọi service.

Pattern upload:

```python
file_url = None
if self.s3_upload_service:
    try:
        file_url = self.s3_upload_service.upload_file(
            file_path=output_path,
            object_name=output_filename,
            content_type="application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        )
    except Exception as e:
        self.logger.warning("S3 upload failed, trying Mezon upload: %s", e)

if not file_url:
    upload_result = await self.client.upload_file(
        file_path=output_path,
        filename=output_filename,
        content_type="application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    )
    file_url = upload_result.url
```

## 11. Test Pattern

Test hiện tại chủ yếu mock service/client thay vì start app thật. Đây là hướng nên giữ cho bot mới.

Command chạy test:

```bash
pytest
pytest tests/test_word_export.py
pytest tests/test_expert_handler_acceptance.py::TestHandleAcceptanceReport::test_handles_contract_not_found
```

Nên viết test theo lớp:

- Unit test service cho rule nghiệp vụ như duplicate, normalize code, date validation.
- Unit test export service cho context mapping.
- Handler test bằng `MagicMock`/`AsyncMock`, mock `reply_message` hoặc `edit_message`.
- Button routing test để đảm bảo `button_id` gọi đúng private handler.
- Repository/integration test chỉ thêm khi cần kiểm tra SQL thật.

Fixture mock client tối thiểu:

```python
from unittest.mock import AsyncMock, MagicMock


client = MagicMock()
client.channels = MagicMock()
channel = AsyncMock()
channel.send = AsyncMock()
client.channels.fetch = AsyncMock(return_value=channel)
client.upload_file = AsyncMock()
```

## 12. Checklist tạo một bot mới từ repo này

1. Đổi tên project trong `pyproject.toml`, `README.md`, Docker metadata nếu cần.
2. Cập nhật `app_settings.app_name`, `title`, `version`.
3. Xác định domain chính của bot mới.
4. Tạo model cho domain và export trong `app/database/models/__init__.py`.
5. Tạo migration và review file migration trước khi upgrade.
6. Tạo repository async theo pattern soft-delete nếu phù hợp.
7. Tạo dataclass DTO và service nghiệp vụ.
8. Tạo handler command bằng `BaseMessageHandler` + `@command`.
9. Nếu dùng button/form, implement `handle_button_click` trong handler.
10. Refactor `HandlerManager.handle_button_click()` để route mọi handler có method `handle_button_click`.
11. Đăng ký repository/service/handler trong `Container`.
12. Thêm tests cho service rule, handler routing, form save/update/delete.
13. Chạy `ruff check .`, `ruff format .`, `pytest`.
14. Chạy app bằng `python run.py --reload`.
15. Nếu deploy Docker, đảm bảo Dockerfile copy đủ template/static asset.

## 13. Quy ước command nên dùng

Nên dùng một top-level command cho một domain:

- `*task`
- `*customer`
- `*ticket`
- `*report`

Subcommand parse trong handler:

- `*task list`
- `*task add`
- `*task find <keyword>`
- `*task edit <id|code>`
- `*task delete <id|code>`

Nên trả help khi thiếu subcommand. Không cần tạo decorator riêng cho từng subcommand nếu command vẫn thuộc cùng domain.

## 14. Các lỗi/gotcha cần tránh

- Đừng quên đăng ký handler trong `providers.List`; command sẽ không chạy nếu chỉ tạo class.
- Đừng quên export model trong `app/database/models/__init__.py`; Alembic có thể không thấy bảng mới.
- Đừng để handler làm quá nhiều nghiệp vụ; move validation/tính toán sang service.
- Đừng giữ ORM object qua nhiều async session; chuyển sang DTO ở service boundary.
- Đừng commit `.env`, `.venv`, `exports/`, generated files.
- Khi dùng `MEZON_BOT_REQUIRE_MENTION=true`, test cả cú pháp mention alias.
- Button id nên có prefix rõ ràng, ví dụ `save_task`, `edit_task:<id>`, `confirm_delete_task:<id>`, để nhiều handler không xử lý nhầm.
- Nếu nhiều handler cùng có `cancel`, mỗi handler có thể cùng nhận event. Handler nên return sớm nếu button không thuộc domain của mình; hoặc route button theo prefix trong `HandlerManager`.
- Nếu export file trong container, đảm bảo thư mục output writable.
- Nếu app production cần nhiều worker, cân nhắc Mezon callback có bị đăng ký nhiều lần không; mặc định `run.py --workers 1` an toàn hơn cho bot event listener.

## 15. Lệnh thường dùng

```bash
uv sync
alembic upgrade head
python run.py --reload
ruff check .
ruff format .
pytest
```

## 16. Template prompt cho agent/dev khi thêm domain mới

Dùng prompt này sau khi fork repo:

```text
Hãy thêm domain <DOMAIN> vào bot Mezon dựa trên kiến trúc hiện tại.

Yêu cầu:
- Giữ FastAPI lifespan và DI container hiện tại.
- Tạo SQLAlchemy model + Alembic migration nếu cần lưu DB.
- Tạo repository async, service dataclass DTO, handler command.
- Command top-level là *<command>.
- Handler dùng @command trong BaseMessageHandler.
- Nếu có form/button, implement handle_button_click và đăng ký handler trong Container.
- Không đặt nghiệp vụ trong app/main.py hoặc route HTTP.
- Viết test cho service rule và handler flow.
- Chạy ruff check, ruff format, pytest.

Luồng mong muốn:
1. *<command> hiển thị help.
2. *<command> list liệt kê dữ liệu.
3. *<command> add mở interactive form.
4. Button save validate input và gọi service create.
5. edit/delete dùng button id có dạng edit_<domain>:<id>, confirm_delete_<domain>:<id>.
```

