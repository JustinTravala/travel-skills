---
name: book-hotel
description: Book a hotel package and pay with USDC via the x402 payment protocol. Use this skill when the user explicitly wants to confirm and pay for a hotel — phrases like "book it", "reserve hotel #2", "go ahead and book the Park Hyatt", "confirm my reservation", "pay for this hotel". This is a real-money on-chain action; always confirm with the user before calling, run search-hotel or search-room first to get a valid packageId, and ensure the user has an authenticated wallet with sufficient USDC.
license: MIT
---

# book-hotel

Create a hotel reservation and pay with USDC. The payment is handled automatically via the **x402 protocol** — your travel API responds with HTTP 402 Payment Required, the agent's wallet pays in USDC, and the booking confirms.

**This is a real on-chain financial transaction.** Proceed carefully.

## When to use

The user has chosen a specific package and explicitly asks to book it:

- "Book the Park Hyatt for me"
- "Reserve option 2"
- "Confirm and pay for this hotel"
- "Go ahead with this one"

Do NOT use this skill speculatively. Only when the user explicitly approves payment.

## Prerequisites

You need ALL of these before calling:

1. **A valid `packageId`** from `search-hotel` or `search-room`
2. **The `sessionId`** from that same search session
3. **Guest details**: first name, last name, email (the user must give these — never invent)
4. **An authenticated Coinbase wallet** — if not, run the `authenticate-wallet` skill first (from the `coinbase/agentic-wallet-skills` package)
5. **Sufficient USDC in the wallet** — if balance is too low, run the `fund` skill first
6. **Explicit user confirmation** of package, total price, and dates

If any of these are missing, ask the user or chain to the relevant skill — do not skip ahead.

## Wallet setup check

Before booking, verify wallet status. Quick check:

```bash
npx awal@latest balance
```

If the user is not authenticated, this returns an auth error → trigger `authenticate-wallet` skill.
If USDC balance is below the booking total, trigger `fund` skill.

## Pre-booking confirmation

Always summarize and confirm before paying. Example:

> About to book and pay:
> - **Park Hyatt Tokyo**, Park Deluxe King with breakfast
> - May 1–5 (4 nights), 2 adults
> - **Total: 1,800 USDC** (paid from your Coinbase wallet)
> - Free cancellation until April 29
> - Guest: Nguyen Van A, guest@example.com
>
> Confirm? (yes/no)

Only run the pay command after the user says yes (or equivalent). If the user already explicitly authorized booking with all the above info in the same turn, you can proceed without re-asking.

## How to call

Use `awal x402 pay` to call the booking endpoint. The CLI handles 402 → sign → retry automatically:

```bash
npx awal@latest x402 pay https://qpgdy2kn7v.ap-southeast-1.awsapprunner.com/m2m-payment/book/ \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{
    "package_id": "<packageId>",
    "session_id": "<sessionId>",
    "contact": {
      "given_name": "<first>",
      "sur_name": "<last>",
      "email": "<email>",
      "phone": "<phone>"
    }
  }'
```

Note the snake_case field names (`package_id`, `session_id`, `given_name`, `sur_name`) — the booking endpoint expects this exact shape. The `contact` object is required and `phone` is part of it.

### Safety: cap maximum payment

To prevent unexpected overcharge (e.g. if the API returns a wrong amount), pass `--max-amount` in micro-USDC (1 USDC = 1,000,000 micro-units):

```bash
  --max-amount 1850000000   # caps payment at 1,850 USDC for an 1,800 USDC booking
```

Always set `--max-amount` to roughly the expected total + a small buffer (5%).

### Full example

```bash
npx awal@latest x402 pay https://qpgdy2kn7v.ap-southeast-1.awsapprunner.com/m2m-payment/book/ \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"package_id":"e47otEJtYeZYblF9","session_id":"6kVmPYwZhQJewNTp","contact":{"given_name":"Nguyen","sur_name":"Van A","email":"guest@example.com","phone":"+84336657091"}}' \
  --max-amount 1850000000
```

## Output

On success, the CLI prints both the payment receipt and the booking confirmation. The booking response is in the body:

```json
{
  "bookingId": "BK_2026_05_001",
  "status": "CONFIRMED",
  "hotelName": "Park Hyatt Tokyo",
  "checkIn": "2026-05-01",
  "checkOut": "2026-05-05",
  "totalPriceUSD": 1800,
  "cancellationDeadline": "2026-04-29T23:59:00Z",
  "paymentTxHash": "0xabc...",
  "confirmationEmail": "guest@example.com"
}
```

The transaction hash (`paymentTxHash`) appears in the `X-PAYMENT-RESPONSE` header — `awal` will surface it in the output.

## Presenting confirmation

> ✅ Booked! Confirmation **BK_2026_05_001**.
> Park Hyatt Tokyo · May 1–5 · 1,800 USDC paid
> Tx: 0xabc...def (view on basescan)
> Free cancellation until **April 29, 2026**.
> Confirmation email sent to guest@example.com.
>
> Save your booking ID — you'll need it to manage or cancel.

## Error handling

- **`402 with amount > --max-amount`** — Booking total exceeds the safety cap. Stop, tell the user the actual amount, and ask whether to proceed with a higher cap.
- **`Insufficient USDC balance`** — Trigger the `fund` skill to top up, then retry.
- **`Wallet not authenticated`** — Trigger the `authenticate-wallet` skill, then retry.
- **`Package no longer available`** (after payment attempt) — Prices/availability shift between search and book. Re-run `search-hotel` or `search-room`. If payment was already settled but booking failed, the API should refund automatically; tell the user and check `awal wallet history`.
- **`Session expired`** — Re-run search to get a new `sessionId`.
- **Network error mid-payment** — Don't retry blindly. Check `awal wallet history` to see if payment went through. If it did, contact support with the tx hash; if not, ask the user whether to retry.

## Critical rules

- Never invent guest details. If the user hasn't given them, ask.
- Never call `awal x402 pay` without explicit approval in this turn or a recent turn.
- Always set `--max-amount` to prevent runaway charges.
- If the user changes their mind ("wait, show me option 3"), do NOT pay — go back to `search-room`.
- Never expose private keys or wallet seed phrases. The agent only signs through `awal`; keys stay in Coinbase's secure infrastructure.
