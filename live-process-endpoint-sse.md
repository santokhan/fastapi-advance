Here’s a **live progress endpoint** using Server-Sent Events (SSE). The browser receives updates as the server produces them.

```python
import asyncio
from collections.abc import AsyncIterable

from fastapi import FastAPI
from fastapi.sse import EventSourceResponse

app = FastAPI()


@app.get("/reports/progress", response_class=EventSourceResponse)
async def report_progress() -> AsyncIterable[dict]:
    for percent in range(0, 101, 20):
        # Replace this with actual steps of your report job.
        await asyncio.sleep(1)

        yield {
            "percent": percent,
            "status": "complete" if percent == 100 else "working",
        }
```

In the browser:

```javascript
const events = new EventSource("/reports/progress");

events.onmessage = (event) => {
  const progress = JSON.parse(event.data);
  console.log(progress.percent, progress.status);

  if (progress.percent === 100) {
    events.close();
  }
};
```

Each `yield` sends an update without waiting for the whole response to finish. FastAPI’s built-in `EventSourceResponse` was added in **FastAPI 0.135.0**, so check your installed version before using this exact import. ([FastAPI][1])

This example performs work inside the request. For a long-running report, run the job separately and have this endpoint stream its progress from shared storage.

[1]: https://fastapi.tiangolo.com/tutorial/server-sent-events/?utm_source=chatgpt.com "Server-Sent Events (SSE)"
