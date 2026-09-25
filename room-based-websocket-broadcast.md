Here’s a **room-based WebSocket broadcast**. Clients connected to the same room receive each other’s messages.

```python
from collections import defaultdict

from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

# room_id -> connected clients
rooms: dict[str, set[WebSocket]] = defaultdict(set)


@app.websocket("/ws/rooms/{room_id}")
async def room_socket(websocket: WebSocket, room_id: str):
    await websocket.accept()
    rooms[room_id].add(websocket)

    try:
        while True:
            message = await websocket.receive_text()

            for client in rooms[room_id].copy():
                await client.send_json({
                    "room_id": room_id,
                    "message": message,
                })

    except WebSocketDisconnect:
        pass
    finally:
        rooms[room_id].discard(websocket)

        if not rooms[room_id]:
            del rooms[room_id]
```

In a browser, connect with:

```javascript
const socket = new WebSocket("ws://localhost:8000/ws/rooms/123");

socket.onmessage = (event) => {
  console.log(JSON.parse(event.data));
};

socket.onopen = () => socket.send("Hello room 123");
```

**Important for your VPS setup:** this in-memory `rooms` dictionary works within **one worker process**. If you run four Gunicorn workers, clients may land on different workers and miss each other’s messages. For that setup, use a shared broker such as Redis Pub/Sub to distribute broadcasts between workers. FastAPI’s own connection-manager example has the same single-process limitation. ([FastAPI][1])

[1]: https://fastapi.tiangolo.com/advanced/websockets/?utm_source=chatgpt.com "WebSockets"
