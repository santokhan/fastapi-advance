Here’s a **cursor-paginated feed**. Unlike page numbers, the cursor keeps “load more” stable when new posts are added.

This example assumes you already have a `Post` model and the `get_session` dependency from the previous example.

```python
from typing import Annotated

from fastapi import Depends, FastAPI, Query
from pydantic import BaseModel
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()


class PostOut(BaseModel):
    id: int
    title: str


class FeedPage(BaseModel):
    items: list[PostOut]
    next_cursor: int | None


@app.get("/feed", response_model=FeedPage)
async def get_feed(
    session: Annotated[AsyncSession, Depends(get_session)],
    limit: Annotated[int, Query(ge=1, le=50)] = 10,
    cursor: int | None = None,
):
    query = select(Post).order_by(Post.id.desc()).limit(limit + 1)

    if cursor is not None:
        query = query.where(Post.id < cursor)

    result = await session.execute(query)
    posts = result.scalars().all()

    has_more = len(posts) > limit
    posts = posts[:limit]

    return FeedPage(
        items=[PostOut(id=post.id, title=post.title) for post in posts],
        next_cursor=posts[-1].id if has_more else None,
    )
```

Call `/feed?limit=10` first. If the response contains `"next_cursor": 42`, call `/feed?limit=10&cursor=42` for the next batch.

The small trick is fetching **`limit + 1`** rows: the extra row tells us whether another page exists. The `Query` declaration also validates the page size and includes its limits in FastAPI’s API docs. ([FastAPI][1])

This version orders by increasing post ID, newest first. If your feed is ordered by ranking or publication time instead, the cursor must include the fields used for that ordering.

[1]: https://fastapi.tiangolo.com/tutorial/query-params-str-validations/?utm_source=chatgpt.com "Query Parameters and String Validations"
