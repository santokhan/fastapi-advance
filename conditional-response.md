from typing import Annotated

from fastapi import Depends, FastAPI, Header, HTTPException, Response
from pydantic import BaseModel
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()


class ArticleOut(BaseModel):
    id: int
    title: str
    content: str


@app.get("/articles/{article_id}", response_model=ArticleOut)
async def get_article(
    article_id: int,
    response: Response,
    session: Annotated[AsyncSession, Depends(get_session)],
    if_none_match: Annotated[str | None, Header()] = None,
):
    article = await session.get(Article, article_id)
    if article is None:
        raise HTTPException(404, "Article not found")

    etag = f'"article-{article.id}-v{article.version}"'
    cache_headers = {
        "ETag": etag,
        "Cache-Control": "private, no-cache",
    }

    if if_none_match == etag:
        return Response(status_code=304, headers=cache_headers)

    response.headers.update(cache_headers)
    return ArticleOut(
        id=article.id,
        title=article.title,
        content=article.content,
    )
