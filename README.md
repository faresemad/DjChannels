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
- Then, adjust your project’s `asgi.py` file, e.g. `myproject/asgi.py`, to wrap the Django ASGI application:
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
## Implement a Chat Server
### Add the room view
- Add the room view to `chat/views.py`:
```python
def room(request, room_name):
    return render(request, "chat/room.html", {"room_name": room_name})
```
- Create the route for the room view in `chat/urls.py`:
```python
urlpatterns = [
    ...
    path("<str:room_name>/", views.room, name="room"),
]
```
### Write your first consumer
- Create a new file `chat/consumers.py`:
```python
# chat/consumers.py
import json

from channels.generic.websocket import WebsocketConsumer


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.accept()

    def disconnect(self, close_code):
        pass

    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]

        self.send(text_data=json.dumps({"message": message}))
```
- Create a new file `chat/routing.py`:
```python
# chat/routing.py
from django.urls import re_path

from . import consumers

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<room_name>\w+)/$", consumers.ChatConsumer.as_asgi()),
]
```
- The next step is to point the main **ASGI** configuration at the **chat.routing** module. In `mysite/asgi.py`, import `AuthMiddlewareStack`, `URLRouter`, and `chat.routing`; and insert a `'websocket'` key in the `ProtocolTypeRouter` list in the following format:
```python
# project/asgi.py
import os

from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.security.websocket import AllowedHostsOriginValidator
from django.core.asgi import get_asgi_application

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")
# Initialize Django ASGI application early to ensure the AppRegistry
# is populated before importing code that may import ORM models.
django_asgi_app = get_asgi_application()

import chat.routing

application = ProtocolTypeRouter(
    {
        "http": django_asgi_app,
        "websocket": AllowedHostsOriginValidator(
            AuthMiddlewareStack(URLRouter(chat.routing.websocket_urlpatterns))
        ),
    }
)
```
### Enable a channel layer
```bash
docker run -p 6379:6379 -d redis:5
```
```bash
python3 -m pip install channels_redis
```
- Before we can use a channel layer, we must configure it. Edit the `project/settings.py` file and add a `CHANNEL_LAYERS` setting to the bottom. It should look like
```python
# project/settings.py
# Channels
ASGI_APPLICATION = "mysite.asgi.application"
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("127.0.0.1", 6379)],
        },
    },
}
```
- But iam using inmemory channel layer
```python
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels.layers.InMemoryChannelLayer"
    }
}
```
- Now that we have a channel layer, let’s use it in `ChatConsumer`. Put the following code in `chat/consumers.py`, replacing the old code:
```python
# chat/consumers.py
import json

from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = "chat_%s" % self.room_name

        # Join room group
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name, self.channel_name
        )

        self.accept()

    def disconnect(self, close_code):
        # Leave room group
        async_to_sync(self.channel_layer.group_discard)(
            self.room_group_name, self.channel_name
        )

    # Receive message from WebSocket
    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]

        # Send message to room group
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name, {"type": "chat_message", "message": message}
        )

    # Receive message from room group
    def chat_message(self, event):
        message = event["message"]

        # Send message to WebSocket
        self.send(text_data=json.dumps({"message": message}))
```

- When a user posts a message, a JavaScript function will transmit the message over WebSocket to a ChatConsumer. The ChatConsumer will receive that message and forward it to the group corresponding to the room name.

- `self.scope["url_route"]["kwargs"]["room_name"]`
    - Obtains the `'room_name'` parameter from the URL route in `chat/routing.py` that opened the **WebSocket** connection to the **consumer**.
    - بيحصل علي الباراميتر اللي اسمه `room_name` من الراوت اللي فاتح الكونكشن

- `self.room_group_name = "chat_%s" % self.room_name`
    - Constructs a Channels group name directly from the user-specified room name, without any quoting or escaping.
    - بيبني اسم للجروب بيستخدم فيه اسم الروم اللي اتحط في الURL

- `async_to_sync(self.channel_layer.group_add)(self.room_group_name, self.channel_name)`
    - Joins a group.
    - بيضيف الكونسيومر ده للجروب اللي اسمه `self.room_group_name`

- `async_to_sync(self.channel_layer.group_discard)(self.room_group_name, self.channel_name)`
    - Leaves a group.
    - بيشيل الكونسيومر ده من الجروب اللي اسمه `self.room_group_name`

- `async_to_sync(self.channel_layer.group_send)(self.room_group_name, {"type": "chat_message", "message": message})`
    - Sends an event to a group.
    - An event has a special `'type'` key corresponding to the name of the method that should be invoked on consumers that receive the event.
    - بيبعت ايفنت للجروب اللي اسمه `self.room_group_name`
    - `{"type": "chat_message", "message": message}`: بيبعت ايفنت من نوع `chat_message` و بيبعت معاه الرساله اللي اتبعتت
    - `chat_message` هو الاسم اللي هيتعرف عليه الكونسيومر اللي هيستقبل الايفنت ده
    - `message` هو الاسم اللي هيستخدمه الكونسيومر اللي هيستقبل الايفنت ده للوصول للرساله اللي اتبعتت
    - `self.send(text_data=json.dumps({"message": message}))`: بيبعت الرساله اللي اتبعتت للكونسيومر اللي هيستقبل الايفنت ده

## Rewrite Chat Server as Asynchronous
- Let’s rewrite `ChatConsumer` to be asynchronous. Put the following code in `chat/consumers.py`:
```python
# chat/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = "chat_%s" % self.room_name

        # Join room group
        await self.channel_layer.group_add(self.room_group_name, self.channel_name)

        await self.accept()

    async def disconnect(self, close_code):
        # Leave room group
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    # Receive message from WebSocket
    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"] # comes from the frontend

        # Send message to room group
        await self.channel_layer.group_send(
            self.room_group_name, {"type": "chat_message", "msgGroup": message}
        )

    # Receive message from room group
    async def chat_message(self, event):
        message = event["msgGroup"]

        # Send message to WebSocket
        await self.send(text_data=json.dumps({"message": message}))
```

- This new code is for **ChatConsumer** is very similar to the original code, with the following differences:
    - `WebsocketConsumer` has been replaced by `AsyncWebsocketConsumer`.
    - All methods have been made `async def`.
    - `await` has been added before all calls into `self.channel_layer`.
    - `async_to_sync` has been removed from the import list.
    - `async_to_sync` has been removed from all calls into `self.channel_layer`.
    - `ChatConsumer` now inherits from `AsyncWebsocketConsumer` rather than `WebsocketConsumer`.
    - All methods are `async def` rather than just `def`.
    - `await` is used to call asynchronous functions that perform I/O.
    - `async_to_sync` is no longer needed when calling methods on the channel layer.