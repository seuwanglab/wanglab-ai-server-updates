```python
import tiktoken
from pydantic import BaseModel, Field
from typing import Optional, Callable, Any, Awaitable
import time


class Filter:
    class Valves(BaseModel):
        priority: int = Field(default=0, description="Priority level")
        enable_warning: bool = Field(
            default=True, description="Enable warning when approaching limit"
        )
        enable_truncation: bool = Field(
            default=True, description="Enable automatic truncation at hard limit"
        )

    def __init__(self):
        self.valves = self.Valves()
        self.encoding = tiktoken.get_encoding("cl100k_base")

        # Model context limits: (warning_tokens, truncate_tokens)
        self.model_limits = {
            # Grok series
            "Grok 4": (64000, 120000),
            "Grok 4 fast": (64000, 120000),
            "Grok 4.1": (64000, 120000),
            # ChatGPT series
            "ChatGPT 4o": (64000, 120000),
            "ChatGPT 4.1": (64000, 120000),
            "ChatGPT 5": (64000, 120000),
            "ChatGPT 5 think": (64000, 120000),
            "ChatGPT 5.1": (64000, 120000),
            "ChatGPT 5.1 think": (64000, 120000),
            "ChatGPT 5 codex": (64000, 120000),
            # Gemini series
            "Gemini 2.5 flash": (256000, 512000),
            "Gemini 2.5 pro": (256000, 512000),
            "Gemini 3 pro": (128000, 256000),
            # DeepSeek series
            "DeepSeek chat": (64000, 120000),
            "DeepSeek reasoner": (64000, 120000),
            "DeepSeek v3": (64000, 120000),
            "DeepSeek r1": (64000, 120000),
            # Other models
            "Kimi k2": (64000, 120000),
            "Minimax m2": (64000, 120000),
            "GLM 4.6": (64000, 120000),
            "Qwen 3": (64000, 120000),
            "Qwen 3 max": (64000, 120000),
            "Qwen 3 think": (64000, 120000),
            "Qwen 3 coder": (64000, 120000),
            # Claude series
            "Claude sonnet 3.7": (64000, 120000),
            "Claude sonnet 3.7 think": (64000, 120000),
            "Claude sonnet 4": (64000, 120000),
            "Claude sonnet 4 think": (64000, 120000),
            "Claude sonnet 4.5": (64000, 120000),
            "Claude sonnet 4.5 think": (64000, 120000),
            "Claude opus 4.1": (64000, 120000),
            "Claude opus 4.1 think": (64000, 120000),
            "Claude haiku 4.5": (64000, 120000),
            # Local models
            "ChatGPT oss 120b": (32000, 64000),
            "Qwen 3 vl 32b": (32000, 64000),
        }

    async def inlet(
        self,
        body: dict,
        __event_emitter__: Callable[[Any], Awaitable[None]],
        __model__: Optional[dict] = None,
        **kwargs,
    ) -> dict:
        print(f"[Smart Context Manager] Processing request at {time.time()}")

        messages = body.get("messages", [])
        if not messages:
            return body

        model_name = self._get_model_name(__model__, body)
        print(f"[Smart Context Manager] Model: {model_name}")

        if model_name not in self.model_limits:
            print(f"[Smart Context Manager] Model '{model_name}' not configured, skipping")
            return body

        warning_limit, truncate_limit = self.model_limits[model_name]

        current_tokens = self._count_total_tokens(messages)
        image_count = sum(self._count_images_in_message(msg) for msg in messages)
        turn_count = self._count_turns(messages)

        print(f"[Smart Context Manager] Current: {current_tokens} tokens, {image_count} images, {turn_count} turns")

        if messages:
            last_msg_tokens = self._count_message_tokens(messages[-1])
            if last_msg_tokens > truncate_limit:
                raise Exception(
                    f"❌ Message too large ({self._format_tokens(last_msg_tokens)}), "
                    f"exceeds model limit ({self._format_tokens(truncate_limit)}). "
                    f"Please reduce text length or image count."
                )

        if self.valves.enable_truncation and current_tokens > truncate_limit:
            print(f"[Smart Context Manager] Truncating: {current_tokens} > {truncate_limit}")
            messages, truncate_info = await self._truncate_by_turns(messages, truncate_limit)

            if truncate_info["final_tokens"] > truncate_limit:
                raise Exception(
                    f"❌ Even after truncation, content ({self._format_tokens(truncate_info['final_tokens'])}) "
                    f"still exceeds limit ({self._format_tokens(truncate_limit)}). "
                    f"Please start a new conversation and reduce content."
                )

            body["messages"] = messages
            current_tokens = truncate_info["final_tokens"]

            await __event_emitter__(
                {
                    "type": "status",
                    "data": {
                        "description": (
                            f"⚠️ Context truncated | "
                            f"Kept {truncate_info['kept_turns']} turns "
                            f"({truncate_info['kept_messages']} messages, "
                            f"{truncate_info['kept_images']} images) | "
                            f"{self._format_tokens(current_tokens)}/{self._format_tokens(truncate_limit)}"
                        ),
                        "done": True,
                    },
                }
            )

        elif self.valves.enable_warning and current_tokens > warning_limit:
            print(f"[Smart Context Manager] Warning: {current_tokens} > {warning_limit}")
            remaining = truncate_limit - current_tokens
            percentage = (current_tokens / truncate_limit) * 100

            await __event_emitter__(
                {
                    "type": "status",
                    "data": {
                        "description": (
                            f"💡 Context length warning | "
                            f"Current {self._format_tokens(current_tokens)} ({percentage:.1f}%) | "
                            f"{turn_count} turns, {image_count} images | "
                            f"Remaining {self._format_tokens(remaining)}"
                        ),
                        "done": True,
                    },
                }
            )

        return body

    def _format_tokens(self, tokens: int) -> str:
        if tokens >= 1_000_000:
            return f"{tokens / 1_000_000:.1f}m tokens"
        elif tokens >= 1_000:
            return f"{tokens / 1_000:.0f}k tokens"
        else:
            return f"{tokens} tokens"

    def _get_model_name(self, __model__: Optional[dict], body: dict) -> str:
        if __model__ and isinstance(__model__, dict):
            name = __model__.get("name") or __model__.get("id")
            if name:
                return name

        metadata = body.get("metadata", {})
        if isinstance(metadata, dict):
            model_info = metadata.get("model", {})
            if isinstance(model_info, dict):
                name = model_info.get("name") or model_info.get("id")
                if name:
                    return name

        return body.get("model", "unknown")

    def _count_turns(self, messages: list) -> int:
        if not messages:
            return 0
        return sum(1 for msg in messages if msg.get("role") == "user")

    def _count_total_tokens(self, messages: list) -> int:
        total = 0
        for msg in messages:
            total += self._count_message_tokens(msg)
        return total

    def _count_message_tokens(self, msg: dict) -> int:
        content = msg.get("content", "")
        tokens = 0

        try:
            if isinstance(content, list):
                for item in content:
                    if isinstance(item, dict):
                        item_type = item.get("type", "")
                        if item_type == "text":
                            text = item.get("text", "")
                            tokens += len(self.encoding.encode(text))
                        elif item_type in ["image", "image_url"] or (
                            item_type == "file" and self._is_image_file(item)
                        ):
                            tokens += 150
            elif isinstance(content, str):
                tokens = len(self.encoding.encode(content))

            role = msg.get("role", "")
            if role:
                tokens += len(self.encoding.encode(role))
                tokens += 4

        except Exception as e:
            print(f"[Smart Context Manager] Token calculation error: {e}")
            if isinstance(content, str):
                tokens = len(content) // 4

        return tokens

    def _count_images_in_message(self, msg: dict) -> int:
        content = msg.get("content", "")
        if isinstance(content, list):
            return sum(
                1
                for item in content
                if isinstance(item, dict)
                and (
                    item.get("type", "") in ["image", "image_url"]
                    or (item.get("type") == "file" and self._is_image_file(item))
                )
            )
        return 0

    def _is_image_file(self, file_item: dict) -> bool:
        try:
            filename = file_item.get("name", "").lower()
            image_extensions = [
                ".jpg",
                ".jpeg",
                ".png",
                ".gif",
                ".bmp",
                ".webp",
                ".svg",
                ".tiff",
                ".ico",
            ]
            if any(filename.endswith(ext) for ext in image_extensions):
                return True

            mime_type = file_item.get("type", "").lower()
            if mime_type.startswith("image/"):
                return True

            return False
        except Exception:
            return False

    async def _truncate_by_turns(
        self, messages: list, target_tokens: int
    ) -> tuple[list, dict]:
        if not messages:
            return messages, {
                "kept_turns": 0,
                "kept_messages": 0,
                "kept_images": 0,
                "final_tokens": 0,
            }

        turns = self._extract_turns(messages)
        print(f"[Smart Context Manager] Extracted {len(turns)} turns from {len(messages)} messages")

        result_messages = []
        current_tokens = 0
        kept_turns = 0
        kept_images = 0

        for turn in reversed(turns):
            turn_tokens = sum(self._count_message_tokens(msg) for msg in turn)
            turn_images = sum(self._count_images_in_message(msg) for msg in turn)

            if current_tokens + turn_tokens <= target_tokens:
                result_messages = turn + result_messages
                current_tokens += turn_tokens
                kept_turns += 1
                kept_images += turn_images
                print(f"[Smart Context Manager] Kept turn {kept_turns}: {turn_tokens} tokens, {turn_images} images")
            else:
                print(f"[Smart Context Manager] Cannot add more turns: would exceed limit")
                break

        if not result_messages:
            return [], {
                "kept_turns": 0,
                "kept_messages": 0,
                "kept_images": 0,
                "final_tokens": float("inf"),
            }

        info = {
            "kept_turns": kept_turns,
            "kept_messages": len(result_messages),
            "kept_images": kept_images,
            "final_tokens": current_tokens,
        }

        print(f"[Smart Context Manager] Truncation result: {info}")
        return result_messages, info

    def _extract_turns(self, messages: list) -> list[list]:
        turns = []
        current_turn = []

        for msg in messages:
            role = msg.get("role", "")

            if role == "user":
                if current_turn:
                    turns.append(current_turn)
                current_turn = [msg]
            elif role == "assistant" and current_turn:
                current_turn.append(msg)
                turns.append(current_turn)
                current_turn = []
            elif role == "system":
                if current_turn:
                    turns.append(current_turn)
                    current_turn = []
                turns.append([msg])
            else:
                if current_turn:
                    current_turn.append(msg)

        if current_turn:
            turns.append(current_turn)

        return turns
```