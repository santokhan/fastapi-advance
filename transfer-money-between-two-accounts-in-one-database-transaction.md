Here’s an advanced FastAPI pattern that stays fairly simple: **transfer money between two accounts in one database transaction**. It uses a request-scoped database session, row locks, and integer amounts to avoid rounding errors.

```python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from pydantic import BaseModel, Field
from sqlalchemy import Integer, select
from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


engine = create_async_engine(
    "postgresql+asyncpg://user:password@localhost:5432/mydb"
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

app = FastAPI()


class Base(DeclarativeBase):
    pass


class Account(Base):
    __tablename__ = "accounts"

    id: Mapped[int] = mapped_column(primary_key=True)
    balance_paisa: Mapped[int] = mapped_column(Integer, nullable=False)


class TransferRequest(BaseModel):
    sender_id: int
    receiver_id: int
    amount_paisa: int = Field(gt=0)


class TransferResponse(BaseModel):
    sender_balance_paisa: int
    receiver_balance_paisa: int


async def get_session():
    async with SessionLocal() as session:
        yield session


Session = Annotated[AsyncSession, Depends(get_session)]


@app.post("/transfers", response_model=TransferResponse)
async def transfer(data: TransferRequest, session: Session):
    if data.sender_id == data.receiver_id:
        raise HTTPException(400, "Accounts must be different")

    async with session.begin():
        # Lock both accounts in a consistent order.
        result = await session.execute(
            select(Account)
            .where(Account.id.in_([data.sender_id, data.receiver_id]))
            .order_by(Account.id)
            .with_for_update()
        )
        accounts = {account.id: account for account in result.scalars()}

        sender = accounts.get(data.sender_id)
        receiver = accounts.get(data.receiver_id)

        if sender is None or receiver is None:
            raise HTTPException(404, "Account not found")

        if sender.balance_paisa < data.amount_paisa:
            raise HTTPException(400, "Insufficient balance")

        sender.balance_paisa -= data.amount_paisa
        receiver.balance_paisa += data.amount_paisa

        response = TransferResponse(
            sender_balance_paisa=sender.balance_paisa,
            receiver_balance_paisa=receiver.balance_paisa,
        )

    return response
```

**What makes it useful:** `session.begin()` commits both balance changes together or rolls them back together. `FOR UPDATE` locks the account rows while the transfer runs, so concurrent requests cannot both spend the same balance. FastAPI’s `yield` dependency closes the session after the request. ([FastAPI][1])

This assumes the `accounts` table already exists. For a real wallet, also store a transfer record with a unique idempotency key in the same transaction, so retrying a request cannot transfer twice.

[1]: https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/?utm_source=chatgpt.com "Dependencies with yield"
