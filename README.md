# beeai-spraay-tool

The first payment tool for [BeeAI Framework](https://github.com/i-am-bee/beeai-framework) agents. With it, agents can validate, estimate and prepare **USDC batch payments on Base** through the [Spraay Protocol](https://docs.spraay.app) x402 gateway. One transaction can pay up to 200 recipients.

`SpraayBatchPaymentTool` is a subclass of BeeAI's `Tool`, so you can pass it to any BeeAI agent like a built-in tool.

**Non-custodial by design:** the tool never handles private keys and never signs or broadcasts anything. `execute_batch` returns *unsigned* transactions, and the sender signs them with their own wallet.

## Install

```bash
pip install beeai-spraay-tool
```

You need Python 3.11–3.13, the same range as `beeai-framework`.

## Quick usage

```python
import asyncio

from beeai_framework.agents.requirement import RequirementAgent
from beeai_framework.backend import ChatModel

from beeai_spraay_tool import SpraayBatchPaymentTool


async def main() -> None:
    agent = RequirementAgent(
        llm=ChatModel.from_name("ollama:granite4:micro"),
        # Give this agent read-only access: it can check and price a batch but not build one.
        tools=[SpraayBatchPaymentTool(allowed_actions=["get_prices", "validate_batch", "estimate_batch"])],
    )
    response = await agent.run(
        "Validate and estimate a USDC batch on Base: "
        "0x2222222222222222222222222222222222222222 gets 150, "
        "0x3333333333333333333333333333333333333333 gets 75.5"
    )
    print(response.last_message.text)


asyncio.run(main())
```

You can also call the tool directly:

```python
result = await SpraayBatchPaymentTool().run(
    {
        "action": "estimate_batch",
        "recipients": ["0x2222222222222222222222222222222222222222", "0x3333333333333333333333333333333333333333"],
        "amounts": ["150", "75.5"],
    }
)
print(result.get_text_content())
```

See [`examples/spraay_example.py`](examples/spraay_example.py) for a multi-agent flow. In it, an analyst agent validates and estimates the batch, a reviewer agent approves it against a budget, and only then does a deterministic step build the unsigned transaction.

## Actions

| Action | Gateway endpoint | What it does |
| --- | --- | --- |
| `get_prices` | `GET /free/prices` | Gets the current token price. |
| `validate_batch` | `POST /free/validate-batch` | Checks recipients and amounts. |
| `estimate_batch` | `GET /free/estimate-batch` | Estimates gas and the protocol fee. |
| `execute_batch` | `POST /api/v1/batch/execute` (x402-paid) | Builds the unsigned batch transaction(s) for `sender` to sign. |

Every response includes a batch summary with the recipient count, the total, the protocol fee (0.3%) and the total including the fee. The summary also names the chain (Base, chain ID 8453) and the Spraay batch contract, `0x1646452F98E36A3c9Cfc3eDD8868221E207B5eEC`.

### Input rules

- **Amounts are human-readable**, for example `"1.5"` USDC. The tool converts them to raw token units using exact decimal math. Amounts with more than 6 decimal places are rejected.
- Addresses must be `0x` followed by 40 hex characters.
- A batch can have at most 200 recipients, and `recipients` and `amounts` must have the same length.
- `execute_batch` requires `sender`.

### Configuration

```python
SpraayBatchPaymentTool(
    base_url="https://gateway.spraay.app",  # Spraay gateway
    http_client=None,                        # optional httpx.AsyncClient, e.g. an x402-enabled one
    allowed_actions=None,                    # restrict actions per agent, e.g. ["get_prices"]
    timeout=30,                              # HTTP timeout in seconds
)
```

The `execute_batch` endpoint charges an x402 micropayment. Without an x402-enabled `http_client`, the tool doesn't fail. Instead it returns `status: "payment_required"` along with the gateway's payment requirements.

## Development

```bash
uv sync
uv run pytest
uv run ruff check . && uv run black --check .
uv run python -m build
```

The tests mock the gateway with `httpx.MockTransport` and need no network access.

## Links

- Spraay docs: https://docs.spraay.app
- BeeAI Framework: https://github.com/i-am-bee/beeai-framework

## License

Apache-2.0
