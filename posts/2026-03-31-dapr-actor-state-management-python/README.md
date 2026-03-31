# How to Implement Actor State Management in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Actor, Python, State Management, Microservice

Description: Learn how to manage Dapr actor state in Python using the Python SDK, including reading, writing, and deleting state within actor methods.

---

## Actor State in Python

Dapr actors in Python persist state through the `_state_manager` object available on every actor instance. This state manager serializes values as JSON and stores them in the configured Dapr state store, scoped to the individual actor instance. When an actor is deactivated and later reactivated, its state is automatically restored.

## Installing the SDK

```bash
pip install dapr dapr-ext-fastapi fastapi uvicorn
```

## Defining a Stateful Actor

```python
from dapr.actor import Actor
from dapr.actor.runtime.context import ActorRuntimeContext
from dapr.actor import ActorInterface, actormethod


class GameSessionInterface(ActorInterface):

    @actormethod(name="start_game")
    async def start_game(self, player_count: int) -> dict:
        ...

    @actormethod(name="record_score")
    async def record_score(self, player_id: str, score: int) -> None:
        ...

    @actormethod(name="get_leaderboard")
    async def get_leaderboard(self) -> list:
        ...

    @actormethod(name="end_game")
    async def end_game(self) -> dict:
        ...


class GameSessionActor(Actor, GameSessionInterface):

    def __init__(self, ctx: ActorRuntimeContext, actor_id):
        super().__init__(ctx, actor_id)

    async def _on_activate(self) -> None:
        """Initialize state if this is the first activation."""
        exists, _ = await self._state_manager.try_get_state("scores")
        if not exists:
            await self._state_manager.set_state("scores", {})
            await self._state_manager.set_state("status", "waiting")
            await self._state_manager.set_state("player_count", 0)

    async def start_game(self, player_count: int) -> dict:
        await self._state_manager.set_state("player_count", player_count)
        await self._state_manager.set_state("status", "active")
        await self._state_manager.set_state("scores", {})
        await self._state_manager.save_state()
        return {"status": "started", "player_count": player_count}

    async def record_score(self, player_id: str, score: int) -> None:
        scores = await self._state_manager.get_state("scores")
        if player_id not in scores:
            scores[player_id] = 0
        scores[player_id] += score
        await self._state_manager.set_state("scores", scores)
        await self._state_manager.save_state()

    async def get_leaderboard(self) -> list:
        scores = await self._state_manager.get_state("scores")
        leaderboard = sorted(
            [{"player": k, "score": v} for k, v in scores.items()],
            key=lambda x: x["score"],
            reverse=True,
        )
        return leaderboard

    async def end_game(self) -> dict:
        leaderboard = await self.get_leaderboard()
        await self._state_manager.set_state("status", "finished")
        await self._state_manager.save_state()
        winner = leaderboard[0] if leaderboard else None
        return {"status": "finished", "winner": winner, "leaderboard": leaderboard}
```

## Using try_get_state for Safe Reads

When state may not exist yet, use `try_get_state` which returns a tuple of `(found, value)`:

```python
async def get_player_score(self, player_id: str) -> int:
    exists, scores = await self._state_manager.try_get_state("scores")
    if not exists or scores is None:
        return 0
    return scores.get(player_id, 0)
```

## Deleting State

Remove individual state keys when they are no longer needed:

```python
async def clear_game(self) -> None:
    await self._state_manager.remove_state("scores")
    await self._state_manager.remove_state("status")
    await self._state_manager.remove_state("player_count")
    await self._state_manager.save_state()
```

## Registering and Running

```python
from fastapi import FastAPI
from dapr.ext.fastapi import DaprActor

app = FastAPI()
actor = DaprActor(app)

@app.on_event("startup")
async def startup():
    await actor.register_actor(GameSessionActor)
```

```bash
dapr run --app-id game-service --app-port 8000 -- uvicorn main:app --port 8000
```

## Summary

Dapr actor state management in Python uses the `_state_manager` property with `get_state`, `set_state`, `try_get_state`, `remove_state`, and `save_state` methods. Always call `save_state()` after mutations to flush changes to the backing store. Use `try_get_state` for optional state to safely handle missing keys without exceptions.
