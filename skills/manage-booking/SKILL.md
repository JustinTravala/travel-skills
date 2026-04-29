---
name: manage-booking
description: Look up details of an existing hotel booking. Use this skill when the user wants to check the status of their reservation, see check-in instructions, verify booking details, or asks "what's the status of my booking", "show me my reservation", "when is my hotel check-in", "did my booking go through". Requires a booking ID, the last name, and the email on the booking.
license: MIT
---

# manage-booking

Retrieve the current state of an existing booking. Read-only — no changes are made.

## When to use

- "What's the status of booking BK_2026_05_001?"
- "Show me my reservation"
- "Did my booking go through?"
- "When do I check in?"

If the user wants to **cancel**, use the `cancel-booking` skill instead.

## Prerequisites

You need:

1. A `bookingId` (returned when the booking was created)
2. The **last name** on the booking, used as a verification check
3. The **email** used at booking time, also used as a verification check

If the user doesn't have the booking ID, ask them to find their confirmation email. Don't guess.

## How to call

```bash
npx @tvl-justin/travel-cli@latest manage-booking \
  --booking-id "<bookingId>" \
  --last-name "<lastName>" \
  --email "<email>"
```

## Output

```json
{
  "bookingId": "BK_2026_05_001",
  "status": "CONFIRMED",
  "hotelName": "Park Hyatt Tokyo",
  "address": "3-7-1-2 Nishi Shinjuku, Tokyo",
  "checkIn": "2026-05-01",
  "checkOut": "2026-05-05",
  "guestName": "Nguyen Van A",
  "totalPriceUSD": 1800,
  "cancellationDeadline": "2026-04-29T23:59:00Z",
  "isCancellable": true
}
```

## Presenting to the user

Give a clean overview, not the raw JSON:

> Booking **BK_2026_05_001** — Confirmed ✅
> **Park Hyatt Tokyo** · 3-7-1-2 Nishi Shinjuku, Tokyo
> Check-in: May 1, 2026 · Check-out: May 5, 2026
> Total: $1,800 (paid)
> Free cancellation until April 29, 2026.

## Errors

- `404 Not Found` — Booking ID not found, or last name / email doesn't match. Ask the user to double-check all three.
- `Network error` — Can't reach the API. Tell the user and offer to retry.
