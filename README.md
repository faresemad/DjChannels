# DjChannels
## Basic Setup & Installation
`python -m pip install -U channels["daphne"]`

- Add `dephne` to `INSTALLED_APPS` in `settings.py`
```python
INSTALLED_APPS = (
    "daphne",
    ...
)
```
- Then, adjust your project’s asgi.py file, e.g. myproject/asgi.py, to wrap the Django ASGI application:
```python
import os

from channels.routing import ProtocolTypeRouter
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')
# Initialize Django ASGI application early to ensure the AppRegistry
# is populated before importing code that may import ORM models.
django_asgi_app = get_asgi_application()

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    # Just HTTP for now. (We can add other protocols later.)
})
```
- And finally, add `ASGI_APPLICATION` in `settings.py`
```python
ASGI_APPLICATION = "myproject.asgi.application"
```