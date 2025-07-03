
# 🛠️ Skyvern Utilities Framework Documentation

## 📁 Directory Structure

```
skyvern/utils/
├── __init__.py                    # Module exports and package initialization
├── file_utils.py                  # File & directory operations
├── sanitization.py                # Data sanitization and validation
├── aio.py                         # Async/await utilities and helpers
├── prompt_engine.py               # LLM prompt management and templating
├── migrate_db.py                  # Database migrations
├── detect_os.py                   # OS detection and compatibility
├── image_utils.py                 # Image processing and manipulation
├── datetime_utils.py              # Date/time formatting and calculations
├── network_utils.py               # Network-related utilities
├── crypto_utils.py                # Cryptographic operations
└── logging_utils.py               # Enhanced logging configuration
```

---

## 📄 File Utilities (`file_utils.py`)

### Purpose
Comprehensive file and directory management utilities for safe filesystem operations

### Core Functions

#### File Operations

```python
def sanitize_filename(filename: str) -> str:
    return "".join(c for c in filename if c.isalnum() or c in ["-", "_", ".", "%", " "])

def rename_file(file_path: str, new_file_name: str) -> str:
    try:
        new_file_name = sanitize_filename(new_file_name)
        new_file_path = os.path.join(os.path.dirname(file_path), new_file_name)
        os.rename(file_path, new_file_path)
        return new_file_path
    except Exception:
        LOG.exception(f"Failed to rename file {file_path}")
        return file_path

def calculate_sha256_for_file(file_path: str) -> str:
    sha256_hash = hashlib.sha256()
    with open(file_path, "rb") as f:
        for byte_block in iter(lambda: f.read(4096), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()
```

#### Directory Management

```python
def create_folder_if_not_exist(dir: str) -> None:
    path = Path(dir)
    path.mkdir(parents=True, exist_ok=True)

def get_skyvern_temp_dir() -> str:
    temp_dir = settings.TEMP_PATH
    create_folder_if_not_exist(temp_dir)
    return temp_dir

def make_temp_directory(suffix: str | None = None, prefix: str | None = None) -> str:
    temp_dir = settings.TEMP_PATH
    create_folder_if_not_exist(temp_dir)
    return tempfile.mkdtemp(suffix=suffix, prefix=prefix, dir=temp_dir)
```

---

## 🧹 Sanitization Utilities (`sanitization.py`)

### Purpose
Input validation and data sanitization for security and consistency

### Key Features

```python
def sanitize_input(input_str: str, max_length: int = 255) -> str:
    cleaned = html.escape(input_str.strip())
    return cleaned[:max_length]

def validate_email(email: str) -> bool:
    pattern = r"^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$"
    return re.match(pattern, email) is not None

def sanitize_json(data: Any) -> Any:
    if isinstance(data, dict):
        return {k: sanitize_json(v) for k, v in data.items() if v is not None}
    elif isinstance(data, list):
        return [sanitize_json(item) for item in data if item is not None]
    elif isinstance(data, str):
        return sanitize_input(data)
    return data
```

---

## ⚡ Async Utilities (`aio.py`)

### Purpose
Helpers for async/await patterns and concurrent operations

```python
async def gather_with_concurrency(n: int, *tasks: Awaitable) -> List[Any]:
    semaphore = asyncio.Semaphore(n)
    async def sem_task(task):
        async with semaphore:
            return await task
    return await asyncio.gather(*(sem_task(task) for task in tasks))

async def timeout_after(timeout: float, coro: Awaitable, timeout_result: Any = None) -> Any:
    try:
        return await asyncio.wait_for(coro, timeout)
    except asyncio.TimeoutError:
        return timeout_result
```

---

## 🧠 Prompt Engine (`prompt_engine.py`)

### Purpose
Dynamic prompt generation and LLM interaction management

```python
class PromptEngine:
    def __init__(self, template_dir: str = "prompts"):
        self.templates = self._load_templates(template_dir)

    def _load_templates(self, dir_path: str) -> Dict[str, str]:
        templates = {}
        for file_path in Path(dir_path).glob("*.txt"):
            with open(file_path) as f:
                templates[file_path.stem] = f.read()
        return templates

    def render(self, template_name: str, context: Dict[str, Any], max_length: int = 2000) -> str:
        template = self.templates.get(template_name)
        if not template:
            raise ValueError(f"Unknown template: {template_name}")
        rendered = template.format(**context)
        return rendered[:max_length]

    def token_count(self, text: str) -> int:
        return len(text.split())
```

---

## 🔄 Database Migration (`migrate_db.py`)

### Purpose
Database schema management and version control

```python
def run_migrations(connection_string: str) -> None:
    config = Config("alembic.ini")
    config.set_main_option("sqlalchemy.url", connection_string)
    alembic.command.upgrade(config, "head")

def create_migration(message: str) -> str:
    config = Config("alembic.ini")
    return alembic.command.revision(config, message=message, autogenerate=True)
```

---

## 🖥️ OS Detection (`detect_os.py`)

### Purpose
Cross-platform compatibility and OS-specific adaptations

```python
class OSDetector:
    @property
    def is_windows(self) -> bool:
        return platform.system() == "Windows"

    @property 
    def is_linux(self) -> bool:
        return platform.system() == "Linux"

    @property 
    def is_macos(self) -> bool:
        return platform.system() == "Darwin"

    def get_os_specific_path(self, path: str) -> str:
        if self.is_windows:
            return path.replace("/", "\\")
        return path.replace("\\", "/")

    def get_platform_info(self) -> Dict[str, str]:
        return {
            "system": platform.system(),
            "release": platform.release(),
            "version": platform.version(),
            "machine": platform.machine()
        }
```

---

## 🏗️ Architecture Principles

- **Consistent Interfaces**: Uniform parameters and type annotations
- **Error Resilience**: Safe fallbacks, logging on failure
- **Performance**: Async utilities, memory-efficient processing
- **Security**: Input validation, sanitization
- **Cross-Platform**: OS-aware behavior

---

*End of documentation*
