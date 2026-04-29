# tvl — Travel CLI

A command-line tool for hotel search and booking management. Designed to be called by AI agents through [Agent Skills](https://agentskills.io).

## Booking is handled separately

This CLI **does not** include a `book` command. The booking endpoint of the travel API is x402-paywalled (HTTP 402 Payment Required + USDC on Base), and payment is handled by Coinbase's [`awal`](https://www.npmjs.com/package/awal) CLI:

```bash
npx awal@latest x402 pay <booking-url> -X POST -d '{...}'
```

This split keeps wallet/key handling fully inside Coinbase's secure infrastructure. See the [travel-skills](https://github.com/your-org/travel-skills) repo for the full skill that orchestrates search + auth + fund + pay.

## Install

```bash
npm install -g tvl
```

Or run without installing:

```bash
npx tvl@latest search-hotel --location "Tokyo" --checkin 2026-05-01 --checkout 2026-05-05
```

## Configuration

```bash
export TVL_API_URL="https://api.your-travel-service.com"
```

If unset, the CLI uses the default URL baked into `src/utils/api.js`.

## Commands

### `tvl search-hotel`

Search hotels by location and dates.

```bash
tvl search-hotel \
  --location "Tokyo" \
  --checkin 2026-05-01 \
  --checkout 2026-05-05 \
  --rooms "2" \
  --limit 5
```

Options:
- `--location <name>` (required) — City or destination
- `--checkin <date>` (required) — `YYYY-MM-DD`
- `--checkout <date>` (required) — `YYYY-MM-DD`
- `--rooms <occupancy>` — e.g. `"2"` (1 room, 2 adults), `"2,5"` (2 adults + child age 5). Multiple rooms separated by `;`
- `--lat <num>` `--lng <num>` — coordinates for precision
- `--min-price <usd>` `--max-price <usd>` — price filters in USD
- `--limit <n>` — max results (default 5)
- `--filters <list>` — comma-separated: `free_breakfast,swimming_pool,ocean_view,all_inclusive`

Returns JSON with `sessionId` (use it in follow-up calls) and an array of hotels with `packageId` for each.

### `tvl search-package`

Get room types and rate packages for a specific hotel.

```bash
tvl search-package \
  --hotel-id "38894021" \
  --session-id "0aidL5kXJVtAS9oz" \
  --checkin 2026-06-14 \
  --checkout 2026-06-18 \
  --rooms "2"
```

### `tvl manage-booking`

Retrieve booking details.

```bash
tvl manage-booking \
  --booking-id "MO45L74J" \
  --last-name "nguyn" \
  --email "iris@travala.com"
```

### `tvl cancel`

Cancel a booking.

```bash
tvl cancel \
  --booking-id "MO45L74J" \
  --last-name "nguyn" \
  --email "iris@travala.com"
```

## Booking workflow (full picture)

```bash
# 1. Search
tvl search-hotel --location "Tokyo" --checkin 2026-05-01 --checkout 2026-05-05
# → returns sessionId + list of hotels with packageIds

# 2. (Optional) Browse rooms for one hotel
tvl search-package --hotel-id h_001 --session-id abc123

# 3. Authenticate the wallet (one time)
npx awal@latest authenticate --email me@example.com

# 4. Book and pay in one call (x402)
npx awal@latest x402 pay https://api.your-travel-service.com/v1/bookings \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"packageId":"pkg_a","sessionId":"abc123","guest":{"firstName":"Nguyen","lastName":"Van A","email":"guest@example.com"}}' \
  --max-amount 1850000000

# 5. Look up the booking later
tvl manage-booking --booking-id BK_2026_05_001 --last-name "Van A" --email guest@example.com
```

## Output format

All commands output JSON to `stdout`. Errors go to `stderr` with a non-zero exit code:

```json
{ "error": "...", "code": "..." }
```

## License

MIT
