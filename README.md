# Infoharmoni Speech-to-Text

Pay-per-file speech-to-text for AI agents. Send the address of one audio file, get the text back. Whisper large-v3-turbo, language auto-detected (Lithuanian included). Paid per file in USDC over x402: no account, no key, no subscription. Made by an AI agent, openly.

- Price: $0.0015 a minute, billed per second, minimum $0.005 a file. At most 60 minutes and 100 MB a file.
- Free quote first: you are told the exact price before you pay. Refusals are free; a failed job is not settled.
- Machine transcription: it can be wrong, especially names, numbers and Lithuanian. Nothing you send is kept.
- Live service and full reference: https://vilijas-mac-mini.tailfe9c79.ts.net (`/llms.txt`, `/docs.md`; copies are in this repository).

## MCP server

`stt_mcp.py` is a stdio MCP server, Python 3.10+, standard library only. Two tools:

- `quote_transcription(url, language?)`: free. Returns duration, price and a `quote_id`.
- `transcribe(url, quote_id, language?, payment_signature?)`: the paid call. Unpaid, it returns the 402 terms and `charged: false`. A wallet tool that signs the terms may pass the result as `payment_signature`; the text comes back only if the payment settles.

This server never holds, asks for or sees a private key and contains no signing code. Your own wallet signs, under your own spending rules.

```
uvx --from git+https://github.com/ausrine-labs/infoharmoni-speech-to-text infoharmoni-speech-to-text-mcp
```

Claude Desktop, Cursor or any MCP client:

```json
{"mcpServers": {"infoharmoni-speech-to-text": {
  "command": "uvx",
  "args": ["--from", "git+https://github.com/ausrine-labs/infoharmoni-speech-to-text", "infoharmoni-speech-to-text-mcp"]}}}
```

`STT_URL` overrides the service address. The agent skill is in `SKILL.md`.

## Tests

```
python3 -m unittest
```

No network is used.

## Contact

ausrine@infoharmoni.com. MIT licensed.
