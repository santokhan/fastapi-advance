Here’s an example for **your mixed news feed**: a response that can contain different item types while keeping each type strictly defined.

```python
from typing import Annotated, Literal

from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI()


class VideoPost(BaseModel):
    type: Literal["videopost"]
    id: int
    title: str
    video_url: str


class Poll(BaseModel):
    type: Literal["poll"]
    id: int
    question: str
    options: list[str]


class Reel(BaseModel):
    id: int
    video_url: str


class ReelsContainer(BaseModel):
    type: Literal["reels"]
    id: int
    items: list[Reel]


FeedItem = Annotated[
    VideoPost | Poll | ReelsContainer,
    Field(discriminator="type"),
]


@app.get("/feed", response_model=list[FeedItem])
async def get_feed():
    return [
        {
            "type": "videopost",
            "id": 1,
            "title": "New video",
            "video_url": "/videos/1.mp4",
        },
        {
            "type": "poll",
            "id": 2,
            "question": "Which phone do you prefer?",
            "options": ["Samsung", "Xiaomi"],
        },
        {
            "type": "reels",
            "id": 3,
            "items": [
                {"id": 10, "video_url": "/reels/10.mp4"},
                {"id": 11, "video_url": "/reels/11.mp4"},
            ],
        },
    ]
```

The `type` field is the **discriminator**. Pydantic reads it to decide whether an item should be validated as `VideoPost`, `Poll`, or `ReelsContainer`. This is similar to a TypeScript discriminated union. ([Pydantic Docs][1])

```typescript
type FeedItem = VideoPost | Poll | ReelsContainer;

if (item.type === "poll") {
  console.log(item.question); // TypeScript knows this is a Poll
}
```

This fits the feed structure you described earlier: single content items and containers can appear in one ordered array.

[1]: https://docs.pydantic.dev/latest/concepts/unions/?utm_source=chatgpt.com "Unions"
