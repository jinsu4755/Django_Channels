[TOC]

# 강의 코드

## 초반부 설정

### room.html

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8"/>
    <title>Chat Room</title>
</head>

<body>
    <textarea id="chat-log" cols="100" rows="20"></textarea><br/>
    <input id="chat-message-input" type="text" size="100"/><br/>
    <input id="chat-message-submit" type="button" value="Send"/>
</body>

<script>
    var roomName = {{ room_name_json }};

    var chatSocket = new WebSocket(
        'ws://' + window.location.host +
        '/ws/chat/' + roomName + '/');

    chatSocket.onmessage = function(e) {
        var data = JSON.parse(e.data);
        var message = data['message'];
        document.querySelector('#chat-log').value += (message + '\n');
    };

    chatSocket.onclose = function(e) {
        console.error('Chat socket closed unexpectedly');
    };

    document.querySelector('#chat-message-input').focus();
    document.querySelector('#chat-message-input').onkeyup = function(e) {
        if (e.keyCode === 13) {  // enter, return
            document.querySelector('#chat-message-submit').click();
        }
    };

    document.querySelector('#chat-message-submit').onclick = function(e) {
        var messageInputDom = document.querySelector('#chat-message-input');
        var message = messageInputDom.value;
        chatSocket.send(JSON.stringify({
            'message': message
        }));

        messageInputDom.value = '';
    };
</script>

</html>

```

### views.py

```python
from django.utils.safestring import mark_safe
import json

def room(request, room_name):
    return render(request, 'chat/room.html', {'room_name_json': mark_safe(json.dumps(room_name))})

```



## 소비자 설정하기

### chat/consumers.py

```python
from channels.generic.websocket import WebsocketConsumer
import json

class ChatConsumer(WebsocketConsumer):
    #websocket 연결시 실행
    def connect(self):
        self.accept()

    def disconnect(self, close_code):
        pass

    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        #클라이언트로 부터 받은 메시지

        self.send(text_data=json.dumps({
            'message':message
        }))
```



### chat/routing.py 생성

```python
from django.urls import path
from . import consumers

websocket_urlpatterns=[
    path('ws/chat/<room_name>/', consumers.ChatConsumer),
]
```

### mysite/routing.py

```python
application = ProtocolTypeRouter({
    # 클라이언트와 Channels 개발 서버가 연결 될 때, 어느 protocol 타입의 연결인지
    # (http->django views is added by default)
  	# 만약에 websocket protocol 이라면, AuthMiddlewareStack
    'websocket': AuthMiddlewareStack(
        # URLRouter 로 연결, 소비자의 라우트 연결 HTTP path를 조사
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    ),
})
```



## radis 설정 

### chat/consumers.py

```python
from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer
import json

class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = 'chat_%s' % self.room_name

        # Join room group
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name,
            self.channel_name
        )
        self.accept()

    def disconnect(self, close_code):
        # Leave room group
        async_to_sync(self.channel_layer.group_discard)(
            self.room_group_name,
            self.channel_name
        )

    # Receive message from WebSocket
    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        # Send message to room group
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message
            }
        )

    # Receive message from room group
    def chat_message(self, event):
        message = event['message']

        # Send message to WebSocket
        self.send(text_data=json.dumps({
            'message': message
        }))
```

