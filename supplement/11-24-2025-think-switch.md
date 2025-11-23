```python
from pydantic import BaseModel, Field
from typing import Optional, Callable, Any, Awaitable


class Filter:
    class Valves(BaseModel):
        priority: int = Field(default=0, description="Filter priority")

    def __init__(self):
        self.valves = self.Valves()
        self.toggle = True
        self.icon = """data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIGZpbGw9Im5vbmUiIHZpZXdCb3g9IjAgMCAyNCAyNCIgc3Ryb2tlLXdpZHRoPSIxLjUiIHN0cm9rZT0iY3VycmVudENvbG9yIj4KICA8cGF0aCBzdHJva2UtbGluZWNhcD0icm91bmQiIHN0cm9rZS1saW5lam9pbj0icm91bmQiIGQ9Ik0xMiAxOHYtNS4yNW0wIDBhNi4wMSA2LjAxIDAgMCAwIDEuNS0uMTg5bS0xLjUuMTg5YTYuMDEgNi4wMSAwIDAgMS0xLjUtLjE4OW0zLjc1IDcuNDc4YTEyLjA2IDEyLjA2IDAgMCAxLTQuNSAwbTMuNzUgMi4zODNhMTQuNDA2IDE0LjQwNiAwIDAgMS0zIDBNMTQuMjUgMTh2LS4xOTJjMC0uOTgzLjY1OC0xLjgyMyAxLjUwOC0yLjMxNmE3LjUgNy41IDAgMSAwLTcuNTE3IDBjLjg1LjQ5MyAxLjUwOSAxLjMzMyAxLjUwOSAyLjMxNlYxOCIgLz4KPC9zdmc+"""
        self.think_map = {
            "ChatGPT 5": "ChatGPT 5 think",
            "ChatGPT 5 codex": "ChatGPT 5 codex high",
            "ChatGPT 5.1": "ChatGPT 5.1 think",
            "Claude sonnet 4.5": "Claude sonnet 4.5 think",
            "Claude sonnet 4": "Claude sonnet 4 think",
            "Grok 4": "Grok 4 heavy",
            "Grok 4.1": "Grok 4.1 think",
            "DeepSeek v3": "DeepSeek r1",
            "DeepSeek chat": "DeepSeek reasoner",
            "Qwen 3": "Qwen 3 think",
        }

    async def inlet(
        self,
        body: dict,
        __event_emitter__: Optional[Callable[[Any], Awaitable[None]]],
    ) -> dict:
        model_name = body.get("model", "")
        if model_name in self.think_map:
            think_model = self.think_map[model_name]
            body["model"] = think_model
            if __event_emitter__:
                await __event_emitter__(
                    {
                        "type": "status",
                        "data": {
                            "description": f"Switched to thinking mode: {think_model}",
                            "done": True,
                        },
                    }
                )
        elif model_name and __event_emitter__:
            await __event_emitter__(
                {
                    "type": "status",
                    "data": {
                        "description": f"Current model {model_name} has no higher-level reasoning version available",
                        "done": True,
                    },
                }
            )

        return body
```