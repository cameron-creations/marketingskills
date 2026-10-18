# NumberBroom

US phone number verification for outbound sales and messaging: line type, carrier, an activity score with a likely-disconnected flag, and a TCPA litigator flag. Checked one number at a time through a REST API or MCP server, or a whole list at a time through a paid scrub. Pay per use, no subscription.

> **Disclosure.** This guide was written by NumberBroom's maker. NumberBroom is not a Verified Partner and paid nothing for this page. Capabilities, fields and prices below come from NumberBroom's own published OpenAPI spec and `llms.txt` as read on 2026-10-17; check [numberbroom.com/pricing](https://numberbroom.com/pricing) before quoting a price.

## Capabilities

| Integration | Available | Notes |
|-------------|-----------|-------|
| API | ✓ | REST at `https://numberbroom.com/api`, OpenAPI spec at `https://numberbroom.com/openapi.json` |
| MCP | ✓ | Remote server, Streamable HTTP, `https://numberbroom.com/mcp` |
| CLI | - | None listed in NumberBroom's API docs |
| SDK | - | None listed in NumberBroom's API docs |

No-code routes exist as well: a [GoHighLevel workflow recipe](https://numberbroom.com/gohighlevel-litigator-scrub) and a [Google Sheets script](https://numberbroom.com/google-sheets-phone-scrub).

## Authentication

- **Type**: API key as a Bearer token (`nb_live_...`)
- **Header**: `Authorization: Bearer nb_live_YOUR_KEY`
- **Get a key**: a person creates it in Settings at https://numberbroom.com/settings. An agent cannot create one; see https://numberbroom.com/auth.md.
- **MCP sign-in**: clients that cannot send a header can connect with an OAuth sign-in instead, and the key stays inside the connection.
- **Limits**: up to 10 keys per account. Unless a key has its own daily limit it allows 5,000 lookups a day, or 500 through the MCP server, and an account allows 5,000 a day across all its keys. Limits reset at midnight UTC.

## When to Use NumberBroom vs. Other Options

| Scenario | Option |
|----------|--------|
| Line type for each number at send time, and you do not need a litigator flag | A carrier lookup from your messaging platform. [twilio.md](twilio.md) lists Twilio's Carrier Lookup at about $0.005 a lookup (check the current price); NumberBroom is $0.20 a verification |
| Line type, carrier, litigator flag and activity score for one number | NumberBroom `POST /v1/verify` or the `verify_phone_number` MCP tool |
| A purchased lead list before cold calling or texting it | NumberBroom list scrub (up to 10,000 numbers a job, paid per list by card), or a TCPA litigator list sold by subscription. Compare coverage, update frequency and terms for your volume |
| Numbers outside the US, including Canada | Not covered: NumberBroom does not check them and does not charge for them |
| Email addresses | [truelist.md](truelist.md) |
| Trying it first | Free: a 20-row preview of a list scrub, and a single-number lookup at https://numberbroom.com/carrier-lookup (5 a day, no card) |

NumberBroom does not send messages or hold consent records. Its own docs say litigator screening reduces TCPA risk and does not eliminate it, and that none of it is legal advice. Its description of the litigator data (court-sourced, refreshed daily) is its own statement, which this guide has not verified.

`POST /v1/verify` does not check do-not-call registries, which is why `dncEvaluated` is always `false`. A list scrub checks the federal Do Not Call Registry only under the account holder's own verified FTC subscription number, and only on lists of up to 1,000 numbers.

## Common Agent Operations

### Verify one number

```bash
POST https://numberbroom.com/api/v1/verify
Authorization: Bearer nb_live_YOUR_KEY
Content-Type: application/json

{ "phone": "(212) 555-0100" }
```

The number can be in any common US format. A call costs $0.20. Credit is debited before the carrier lookup and returned if the lookup produces no usable answer, and an answer of `valid: false` is free. One case is still charged: a US number the carrier reports as not valid comes back as a 200 with `outcome: "invalid"` and `keep: false`, because the lookup ran and answered.

| Field | Meaning |
|-------|---------|
| `phone`, `e164` | The input as sent, and the parsed number (`null` when it does not parse) |
| `valid` | `false` when the number could not be checked. Then `reason` is `unparseable`, `not_nanp` (outside the +1 plan), `outside_us` (a +1 number outside the US) or `suppressed` (its carrier data is withheld, `outcome` is `lookup_failed`, and asking again returns the same), and `message` says why. A successful answer, not an error |
| `lineType`, `carrier` | Line type and carrier name |
| `activityScore` | 0 to 100, or `null` when the carrier data has no signal |
| `isLikelyDisconnected` | `true` at an activity score of 30 or below, `null` when there is no score |
| `isLitigator`, `litigator` | The litigator flag as a boolean and as a word (`no` in NumberBroom's published example), for no-code tools that store text. `null` and `not_checked` when no litigator check ran, which includes a number the carrier reports as not valid |
| `outcome`, `keep` | NumberBroom's classification of the row (`clean`, `invalid` and `lookup_failed` appear in its docs) and whether it keeps it |
| `dncEvaluated` | Always `false` on this endpoint (see below) |
| `charged`, `creditsRemaining` | What this call cost and the balance left |

A dead mobile still reports as mobile, so read disconnection from the activity score, not the line type.

None of these is charged:

| Status | Meaning |
|--------|---------|
| 400 | No `phone` field, or one too long to be a number |
| 401 | Missing, invalid or revoked key |
| 402 | Balance below $0.20. A person tops up in Settings |
| 429 | The key's or the account's daily limit was reached (`rate_limited`), or the account's daily processing limit, which list uploads count toward (`processing_limit`) |
| 502 | The carrier lookup produced no usable answer (`lookup_failed`). Refunded; safe to retry later |
| 503 | Lookups are paused on NumberBroom's side; try again shortly |

### Check the balance

```bash
GET https://numberbroom.com/api/v1/credits
```

Free. Returns the credit, the rate per lookup, how many lookups remain, and the key's and account's daily limits.

### Add credit

```bash
POST https://numberbroom.com/api/v1/credits/checkout

{ "amount": 25 }
```

Returns a Stripe Checkout link for $10 to $5,000 of credit. The call spends nothing; a person pays by card.

### Scrub a whole list

```bash
POST https://numberbroom.com/api/v1/lists

{ "numbers": ["(212) 555-0100", "(212) 555-0101"], "scrubMode": "full" }
```

Up to 10,000 numbers a job. Returns the price and a Stripe Checkout link for a person to pay; nothing is taken from API credit. `scrubMode` is `full` (keeps verified mobile numbers only) or `litigator_only` (removes litigators and invalid numbers, keeps valid landlines, VoIP and disconnected numbers for teams that dial those by hand). Then poll:

```bash
GET https://numberbroom.com/api/v1/lists/{jobId}
```

This returns the status, counts (clean, litigator, VoIP or landline, disconnected, invalid, errors) and, once finished, a download link that lasts an hour. Numbers outside the US come back in their own file and are not charged.

## MCP Server

Add the hosted server to your client. For Claude Code, `.mcp.json`:

```json
{
  "mcpServers": {
    "numberbroom": {
      "type": "http",
      "url": "https://numberbroom.com/mcp",
      "headers": { "Authorization": "Bearer ${NUMBERBROOM_API_KEY}" }
    }
  }
}
```

Set `NUMBERBROOM_API_KEY` in the client's environment. Setup for other clients is at https://numberbroom.com/mcp-server.

| Tool | Does | Cost |
|------|------|------|
| `verify_phone_number` | Line type, carrier, activity score and litigator flag for one US number | $0.20 a call from credit; unparseable and non-US numbers are not charged |
| `get_credit_balance` | Remaining credit | Free |
| `buy_credits` | A Stripe Checkout link that adds credit | Free to call; a person pays |
| `scrub_list` | Sends a list (up to 10,000 numbers) and returns the price and a checkout link | Free to call; a person pays per list |
| `get_list_status` | Status and counts, with one-hour download links when done | Free |

## Pricing

Published 2026-10-17; no subscription, contract or minimum commitment.

- Single verification (API or MCP): $0.20, with a $10 minimum top-up.
- List scrub: $0.20 a number, $5 flat for lists of 25 numbers or fewer, $0.16 past 2,000 rows and $0.08 past 5,000. That makes 1,000 numbers $200 and 10,000 numbers $1,280.
- Optional add-ons on a list scrub: a contact grade, a business line lookup and an outside-US number check, each priced on the pricing page.
- A number repeated in a list is checked and charged once. A row the provider cannot answer is refunded automatically.

## Tradeoffs

- US numbers only.
- The price per number is higher than a bare carrier lookup, and it includes the litigator check and activity score. If you need line type alone at scale, a messaging platform's lookup costs less.
- A person has to create the key and pay by card. An agent can check a balance and request a payment link but cannot fund the account itself.
- Single verifications are capped at 5,000 a day per account; larger volumes go through the list scrub.
