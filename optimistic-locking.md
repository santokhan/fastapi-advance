Another useful pattern is **optimistic locking**. It prevents one user’s older edit from overwriting a newer edit made by someone else.

Assume your `Article` model has `id`, `title`, and an integer `version` column, and you have the `get_session` dependency from the earlier examples.

```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()


class ArticleUpdate(BaseModel):
    title: str = Field(min_length=1)
    version: int = Field(ge=1)


class ArticleOut(BaseModel):
    id: int
    title: str
    version: int


@app.put("/articles/{article_id}", response_model=ArticleOut)
async def edit_article(
    article_id: int,
    data: ArticleUpdate,
    session: Annotated[AsyncSession, Depends(get_session)],
):
    async with session.begin():
        result = await session.execute(
            update(Article)
            .where(
                Article.id == article_id,
                Article.version == data.version,
            )
            .values(
                title=data.title,
                version=Article.version + 1,
            )
            .returning(
                Article.id,
                Article.title,
                Article.version,
            )
        )
        updated = result.one_or_none()

        if updated is None:
            exists = await session.scalar(
                select(Article.id).where(Article.id == article_id)
            )
            if exists is None:
                raise HTTPException(404, "Article not found")
            raise HTTPException(409, "Article changed. Reload and try again.")

    return ArticleOut(
        id=updated.id,
        title=updated.title,
        version=updated.version,
    )
```

The client sends the version it last read, for example `{"title": "New title", "version": 3}`. The update succeeds only if the database still has version `3`; it then becomes version `4`. The version check and update happen in **one SQL statement**, so two concurrent edits cannot both overwrite each other. This example uses PostgreSQL’s `RETURNING` support. ([SQLAlchemy 2.0 Documentation][1])

[1]: https://docs.sqlalchemy.org/en/20/tutorial/data_update.html?utm_source=chatgpt.com "Using UPDATE and DELETE Statements"
