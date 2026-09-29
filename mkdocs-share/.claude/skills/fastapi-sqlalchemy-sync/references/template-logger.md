# Template: utils/logger.py

```python
import logging
import sys

from app.config.settings import settings


def get_logger():
    app_logger = logging.getLogger(settings.SERVICE_NAME)
    if app_logger.hasHandlers() is False:
        stream_handler = logging.StreamHandler(stream=sys.stdout)
        simple_formatter = logging.Formatter(
            "{asctime} - {name} - {levelname} - {module} - {funcName} - {message}",
            style="{",
        )
        stream_handler.setFormatter(simple_formatter)
        app_logger.setLevel(logging.INFO)
        app_logger.addHandler(stream_handler)
        app_logger.propagate = False

    return app_logger
```

## Rules

- Logger name is `settings.SERVICE_NAME` — distinguishes logs across services in shared sinks
- Always call `get_logger()` at module level — never inside a function
- The `hasHandlers()` guard prevents duplicate handlers when the module is imported multiple times
- `propagate = False` prevents double logging if the root logger also has handlers
- Log level is `INFO` — change to `logging.DEBUG` locally if needed, never commit `DEBUG` level
- Output goes to stdout only — no file handlers
