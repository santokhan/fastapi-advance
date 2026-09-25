Here’s a pattern for **reusing an HTTP client** when your FastAPI app calls another API.

```python
from contextlib import asynccontextmanager

import httpx
from fastapi import FastAPI, HTTPException, Request


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Runs once when this worker starts.
    async with httpx.AsyncClient(
        base_url="https://api.example.com",
        timeout=5.0,
    ) as client:
        app.state.http_client = client
        yield
    # The client closes when the app shuts down.


app = FastAPI(lifespan=lifespan)


@app.get("/products/{product_id}")
async def get_product(product_id: int, request: Request):
    client: httpx.AsyncClient = request.app.state.http_client

    try:
        response = await client.get(f"/products/{product_id}")
        response.raise_for_status()
    except httpx.TimeoutException:
        raise HTTPException(504, "Product service timed out")
    except httpx.HTTPStatusError:
        raise HTTPException(502, "Product service returned an error")
    except httpx.RequestError:
        raise HTTPException(502, "Could not reach product service")

    return response.json()
```

The `lifespan` function opens the client at startup and closes it at shutdown. Reusing the client allows HTTPX to reuse connections instead of creating a new client for every request. With your four-worker setup, **each worker creates its own client**. ([fastapi.tiangolo.com][1])

[1]: https://fastapi.tiangolo.com/advanced/events/?utm_source=chatgpt.com "Lifespan Events"
