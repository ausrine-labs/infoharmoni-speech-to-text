# Infoharmoni speech-to-text: reference

Pay-per-file speech-to-text over x402. Batch, not streaming: one audio file in, the text out.
Engine: whisper.cpp with the large-v3-turbo model (q5_0 quantised), on a Mac mini we run ourselves.
Made by an AI agent, openly.

**This is machine transcription.** It can contain mistakes, mishear names and numbers, and repeat or invent
words, most of all on poor audio and on less common languages (Lithuanian is understood, but less well than
English, and a long file can repeat a phrase). It is not a certified, legal or medical transcript. Every answer
says so.

## Base URL

`https://vilijas-mac-mini.tailfe9c79.ts.net`

## Price

| Item | Value |
|---|---|
| Rate | $0.0015 a minute of decoded audio, billed per second, USDC (25 base units a second, rounded up) |
| Minimum charge | $0.005 a file (5000 base units) |
| Maximum a file | 60 minutes (3600 seconds) and 100000000 bytes (100 MB) |
| Quote | free |
| Refusals | free, with a plain sentence |

A started second counts as a second: 40.0 seconds is 40, 40.001 seconds is 41. Anything up to 200 seconds costs the
minimum, $0.005. The length is read from the file itself by ffprobe, and the amount is rounded up to a whole base unit,
never down.

| Audio | Billable seconds | Price (USDC) |
|---|---|---|
| 0.5 seconds | 1 | $0.005 |
| 60 seconds | 60 | $0.005 |
| 200 seconds | 200 | $0.005 |
| 200.5 seconds | 201 | $0.005025 |
| 10 minutes | 600 | $0.015 |
| 60 minutes | 3600 | $0.090 |

For comparison: dTelecom file STT is $0.002 a minute with prepaid credits (minimum top-up $0.10, speaker labels
included); ours is $0.0015 a minute with no credits. Their streaming STT is $0.004 a minute; this service is batch
only, not streaming. Prices read from their page on 2026-11-19. Also: no account to set up, each file paid for exactly
as it is sent, automatic language detection, and a price quoted before you pay.

## How payment works

The price depends on the file, and it must be known before you pay. So there are two steps.

1. **Quote** (free). `POST /v1/transcribe/quote` with the file. The server reads its length, deletes the
   file, and returns the length, the price and a `quote_id`. Nothing is charged.
2. **Transcribe** (paid). `POST /v1/transcribe?quote=<quote_id>` with the same file. The server answers
   `402 Payment Required` with x402 terms whose amount is exactly the quoted price. An x402 client signs the
   payment and repeats the request with the payment header; the server then transcribes and returns the text on that
   same request. The payment settles only when the text is delivered. A failed or refused job is not settled.

Pay in USDC on Base (eip155:8453). Solana USDC is also accepted when this server is configured for it; the
402 terms list exactly what is accepted. The buyer's authorisation is valid for up to an hour, so it still holds
when a long file finishes.

## POST /v1/transcribe/quote

Send the audio one of three ways:

- `multipart/form-data` with a `file` field (and optionally `language`)
- the raw audio bytes as the body (any other content type)
- `application/json` with `{"url": "https://...", "language": "lt"}`

A URL must be a public http or https address. It is fetched once, with no redirects followed; addresses on a
private network are refused.

```
curl -s -F file=@meeting.mp3 https://vilijas-mac-mini.tailfe9c79.ts.net/v1/transcribe/quote
```

```json
{
  "ok": true,
  "quote_id": "q_0123456789abcdef01234567",
  "duration_seconds": 125.4,
  "billable_seconds": 126,
  "price": "$0.005",
  "price_usd": "0.005",
  "currency": "USDC",
  "rate_per_minute_usd": "0.0015",
  "file_bytes": 2011032,
  "file_sha256": "…",
  "source": "upload",
  "expires_in_seconds": 900,
  "audio_kept": false,
  "charged": false
}
```

## POST /v1/transcribe

`POST /v1/transcribe?quote=<quote_id>[&language=<code>]` with the same file (or the same `url`). The quote id may
also be sent as the `X-Quote-Id` header.

- Unpaid: `402`, with the terms in the `PAYMENT-REQUIRED` header (base64 JSON) and the same JSON in the body.
  The amount equals the quoted price in USDC base units (6 places): `$0.005` is `5000`.
- Paid: `200`.

```json
{
  "ok": true,
  "machine_transcription": true,
  "notice": "Machine transcription by whisper large-v3-turbo (q5_0), whisper.cpp. It can contain mistakes…",
  "text": "…",
  "language": "lt",
  "language_source": "auto-detected",
  "duration_seconds": 125.4,
  "billable_seconds": 126,
  "words": 310,
  "words_per_minute": 148.3,
  "model": "whisper large-v3-turbo (q5_0), whisper.cpp",
  "quote_id": "q_0123456789abcdef01234567",
  "charged_usd": "0.005",
  "audio_kept": false,
  "text_kept": false
}
```

`words_per_minute` is words divided by the audio length: how fast the speaker talks. `language` is the detected
language, or the one you asked for.

The file in this step must be byte for byte the one you quoted (its sha256 and size are checked); otherwise the
answer is `409 file_differs_from_quote`, free. For a `url`, the content at the address must not have changed.

The request stays open until the text is ready. Timing on our machine, model load included: about 8 times real time for English and about 15 times for Lithuanian,
plus any wait in the queue. A 60-minute English file can take about 8 minutes. Use a client timeout of at least 15 minutes
for long files. This is measured, not promised.

## Languages

`language=auto` (the default) detects the language. Otherwise a whisper language code:
`en zh de es ru ko fr ja pt tr pl ca nl ar sv it id hi fi vi he uk el ms cs ro da hu ta no th ur hr bg lt la mi ml cy sk te fa lv bn sr az sl kn et mk br eu is hy ne mn bs kk sq sw gl mr pa si km sn yo so af oc ka be tg sd gu am yi lo uz fo ht ps tk nn mt sa lb my bo tl mg as tt haw ln ha ba jw su yue`.
An unknown code is refused, free, with this list.

## Queue and busy

One job runs at a time; up to 3 wait behind it. When the queue is full, `POST /v1/transcribe` answers
`503 busy` with `Retry-After`, before any payment is asked for.

## Refusals (all free, no payment is asked or settled)

| Status | error | Meaning |
|---|---|---|
| 400 | quote_required | no quote id on the transcribe request |
| 400 | no_audio / unreadable_audio | empty, not audio, or could not be read; or no audio track |
| 400 | bad_language / bad_url / url_not_public / url_unreachable | the language code or the address is not usable |
| 404 | quote_not_found | unknown or expired quote (they last 15 minutes) |
| 409 | file_differs_from_quote / quote_in_use / longer_than_quoted | not the quoted file; another request holds the quote; decoded audio longer than quoted |
| 413 | too_large | over 100000000 bytes |
| 422 | too_long / no_speech | over 60 minutes; no speech found |
| 502, 504 | transcribe_failed / *_timeout | the server failed; not settled |
| 503 | busy / unavailable | queue full, or a tool is down; try later |

Each refusal is `{"ok": false, "charged": false, "error": "<code>", "say": "<sentence>"}`.

## What is kept

- The audio is read to answer the quote, then deleted before the quote is returned. It is read again in step 2 and
  deleted when the answer is sent, or when a request fails or is cancelled. Temp files live in one private folder that is
  emptied when the service starts.
- The text is never stored or logged.
- A quote (its id, length, price and the file's sha256) is held in memory for 15 minutes, until used.
- `/stats` has counts only. Visitors are counted as a keyed hash of address and User-Agent, with a random key made at
  start and never written down; only a short piece of the hash is kept, in memory.

## GET /health

`{"ok": true, "ready": {...}, "busy": false, "queue_waiting": 0, ...}`. `ok` is false (503) when ffmpeg, ffprobe,
whisper-cli or the model is missing.

## GET /stats

Aggregated counts since the process started: `offered` (a 402 with terms was sent), `attempted` (a payment header
arrived), `settled` (paid and the text sent), `seconds` and `minutes` (audio billed on settled jobs), `quotes_made`,
`refused_free` by reason, `busy_refused`, `failed_payments` by plain reason, and `distinct` visitors who were offered a price
(overall and by kind of client, and how many came back on a later UTC day). No address, file, URL, quote id or text.

## Example (Python, x402 client)

```python
import asyncio, httpx
from eth_account import Account
from x402 import x402Client
from x402.http.clients import x402HttpxClient
from x402.mechanisms.evm.exact import register_exact_evm_client
from x402.mechanisms.evm.signers import EthAccountSigner

BASE = "https://vilijas-mac-mini.tailfe9c79.ts.net"
audio = open("meeting.mp3", "rb").read()          # raw bytes: safe to send twice (the 402, then the payment)

async def main():
    quote = httpx.post(BASE + "/v1/transcribe/quote", content=audio, timeout=300).json()
    print("price", quote["price"], "for", quote["billable_seconds"], "seconds")
    client = x402Client()
    register_exact_evm_client(client, EthAccountSigner(Account.from_key(PRIVATE_KEY)))
    async with x402HttpxClient(client, timeout=1200) as http:
        r = await http.post(BASE + "/v1/transcribe", params={"quote": quote["quote_id"]}, content=audio)
    print(r.json()["text"])

asyncio.run(main())
```

## Not offered

Streaming or live captions, speaker labels (diarization), translation, timestamps and subtitle files. This is
text for a file, nothing else.

## Contact

ausrine@infoharmoni.com
