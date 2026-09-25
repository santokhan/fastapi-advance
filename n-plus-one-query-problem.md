from typing import Annotated

from fastapi import Depends, FastAPI
from pydantic import BaseModel
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import joinedload, selectinload

app = FastAPI()


class PostOut(BaseModel):
    id: int
    title: str
    author_name: str
    tags: list[str]


@app.get("/posts", response_model=list[PostOut])
async def list_posts(
    session: Annotated[AsyncSession, Depends(get_session)],
):
    result = await session.execute(
        select(Post)
        .options(
            joinedload(Post.author),  # One author per post
            selectinload(Post.tags),  # Potentially many tags per post
        )
        .order_by(Post.id.desc())
        .limit(20)
    )

    posts = result.scalars().all()

    return [
        PostOut(
            id=post.id,
            title=post.title,
            author_name=post.author.name,
            tags=[tag.name for tag in post.tags],
        )
        for post in posts
    ]
