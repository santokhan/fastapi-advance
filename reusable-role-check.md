Here’s a **reusable role check** for FastAPI. Assume `get_current_user` already authenticates the request and returns a user with a `role` field.

```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException

app = FastAPI()


def require_role(*allowed_roles: str):
    async def check_role(
        user: Annotated[User, Depends(get_current_user)],
    ) -> User:
        if user.role not in allowed_roles:
            raise HTTPException(
                status_code=403,
                detail="You do not have permission",
            )
        return user

    return check_role


@app.post("/admin/categories")
async def create_category(
    user: Annotated[User, Depends(require_role("admin"))],
):
    return {"created_by": user.id}


@app.get("/reports")
async def view_reports(
    user: Annotated[User, Depends(require_role("admin", "manager"))],
):
    return {"requested_by": user.id}
```

`require_role("admin")` creates a dependency configured for that route. FastAPI first runs `get_current_user`, then passes its result into `check_role`. The endpoint runs only if the role is allowed. FastAPI supports this kind of parameterized dependency. ([FastAPI][1])

For a larger app, you can use the same pattern with specific permissions such as `"category:create"` and `"reports:read"` instead of broad roles.

[1]: https://fastapi.tiangolo.com/advanced/advanced-dependencies/?utm_source=chatgpt.com "Advanced Dependencies"
