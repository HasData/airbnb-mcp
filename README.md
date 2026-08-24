# Airbnb MCP Server

<!-- mcp-name: com.hasdata/airbnb -->

A hosted Model Context Protocol (MCP) server that gives Claude, Cursor, Windsurf and any other MCP client two read-only Airbnb tools. Search stays by location and dates, and read a single listing in full, all as structured JSON, with no Airbnb developer account and no partner approval.

It reads public listing pages that a signed-out visitor can see.

```
https://mcp.hasdata.com/api/mcp?apis=airbnb
```

[![Glama score](https://glama.ai/mcp/servers/HasData/airbnb-mcp/badges/score.svg)](https://glama.ai/mcp/servers/HasData/airbnb-mcp)
[![tool contract](https://github.com/HasData/airbnb-mcp/actions/workflows/contract.yml/badge.svg)](https://github.com/HasData/airbnb-mcp/actions/workflows/contract.yml)
[![MCP](https://img.shields.io/badge/MCP-remote%20%7C%20streamable%20HTTP-6366f1?style=flat-square)](https://modelcontextprotocol.io)
[![Tools](https://img.shields.io/badge/tools-2-10b981?style=flat-square)](#tools)
[![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](LICENSE)

## Contents

- [What you need](#what-you-need)
- [Quick start](#quick-start)
- [Example prompts](#example-prompts)
- [Tools](#tools)
- [Errors and failure paths](#errors-and-failure-paths)
- [Pricing, free tier and limits](#pricing-free-tier-and-limits)
- [Tool selection](#tool-selection)
- [How it compares](#how-it-compares)
- [FAQ](#faq)
- [HasData links](#hasdata-links)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

## What you need

An MCP client and a HasData API key from the [dashboard](https://app.hasdata.com/sign-up?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp), free to create with no card, and the trial covers about 200 calls at the 5-credit rate. This is a remote server, so the simplest path is a URL and an `x-api-key` header, with no container to run and no Airbnb developer account anywhere in the flow. A client that only speaks stdio reaches it through a thin launcher, published as `@hasdata/airbnb-mcp` on npm and `hasdata-airbnb-mcp` on PyPI, shown below.

## Quick start

The server URL is the same for every client. We run it hands-on in Claude Code and Claude Desktop. The other blocks follow each client's own documented format for a remote server.

| Field | Value |
| :--- | :--- |
| URL | `https://mcp.hasdata.com/api/mcp?apis=airbnb` |
| Transport | HTTP, streamable |
| Auth header | `x-api-key: HASDATA_API_KEY` |

Clients with OAuth support can add the same URL as a connector and sign in without putting a key in a config file.

<details>
<summary><b>Claude Code</b></summary>

```bash
claude mcp add --transport http airbnb "https://mcp.hasdata.com/api/mcp?apis=airbnb" \
  --header "x-api-key: HASDATA_API_KEY"
```

</details>

<details>
<summary><b>Claude Desktop</b></summary>

Settings, then Connectors, then Add custom connector, then paste `https://mcp.hasdata.com/api/mcp?apis=airbnb` and sign in.

For the config-file route, Claude Desktop loads only local (stdio) servers, so it reaches a remote server through a stdio launcher. The `@hasdata/airbnb-mcp` package is that launcher, and it reads the key from the environment. Add this to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "airbnb": {
      "command": "npx",
      "args": ["-y", "@hasdata/airbnb-mcp"],
      "env": { "HASDATA_API_KEY": "YOUR_KEY" }
    }
  }
}
```

For Python instead of Node, swap the launcher for the PyPI package, which `uvx` runs without a manual install:

```json
{
  "mcpServers": {
    "airbnb": {
      "command": "uvx",
      "args": ["hasdata-airbnb-mcp"],
      "env": { "HASDATA_API_KEY": "YOUR_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>Cursor</b></summary>

`~/.cursor/mcp.json` for every project, or `.cursor/mcp.json` for one:

```json
{
  "mcpServers": {
    "airbnb": {
      "url": "https://mcp.hasdata.com/api/mcp?apis=airbnb",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>Windsurf</b></summary>

`~/.codeium/windsurf/mcp_config.json`. Windsurf calls the field `serverUrl`, not `url`:

```json
{
  "mcpServers": {
    "airbnb": {
      "serverUrl": "https://mcp.hasdata.com/api/mcp?apis=airbnb",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>VS Code</b></summary>

`.vscode/mcp.json` in the workspace:

```json
{
  "servers": {
    "airbnb": {
      "type": "http",
      "url": "https://mcp.hasdata.com/api/mcp?apis=airbnb",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

## Example prompts

Prompts, not code. Paste one in and the agent picks the tool itself. Each is annotated with the calls it takes, because every successful call costs 5 credits.

> Search Austin stays for two adults from September 15 to 18 and give me the ten highest-rated under $200 a night.

*One call, 5 credits. Rating and nightly price come back on the search result.*

> Take the top result and pull its full amenities, the guest and bedroom count, and the host details.

*One call, 5 credits. Those live on the property page, which the details tool reads by URL.*

> Compare the nightly price of a three-night stay for two guests in Austin against Nashville.

*Two calls, 10 credits, one search per city.*

> For this listing URL, tell me how many beds and baths it has and whether the host is a Superhost.

*One call, 5 credits.*

A search result is deliberately light, enough to rank and shortlist. The amenities, beds, host and full description come from the property call, so a prompt that shortlists then inspects three homes is one search plus three property calls.

## Tools

Two tools, read-only. Samples below are trimmed from real calls, and the numbers move as Airbnb updates. Read them as shapes. Each tool name links to its endpoint reference, which carries the full field list.

The samples are the payload, not the whole response. A `tools/call` result carries one text block, and that text is itself JSON holding `url`, `status`, `text` and `json`, with the scraped data under `json`. From a raw JSON-RPC response the path is `result.content[0].text`, parsed, then `.json`. A chat client unwraps that for you and code talking to the endpoint directly does not.

### Get Airbnb listings

[`hasdata_airbnb_listing_getAirbnbListings`](https://docs.hasdata.com/apis/airbnb/listing?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp)

A page of search results by location and dates.

| Parameter | Type | Required | Notes |
| :--- | :--- | :--- | :--- |
| `location` | string | yes | The place to search, such as `Austin, Texas` |
| `checkIn` | string | yes | Check-in date, `YYYY-MM-DD` |
| `checkOut` | string | | Check-out date, `YYYY-MM-DD` |
| `adults` / `children` / `infants` / `pets` | number | | Guest composition, each a count |
| `nextPageToken` | string | | The `pagination.nextPageToken` from the previous response |

Returns a `properties` array and `pagination`, whose `nextPageToken` and `pageTokens` walk the result set. Each property carries `id`, `url`, `title`, `latitude`, `longitude`, a short `description` tagline, a `photos` array, `rating`, `reviews`, a `badges` array such as `Guest favorite` or `Superhost`, and a `price` object. `price` holds `originalPrice`, an optional `discountedPrice` when the stay is discounted, a `qualifier` like `for 3 nights`, and a `breakdown`.

> A search result is intentionally light. Beds, baths, amenities, the host and the full description are not here, they come from the property tool below. Do not expect a bedroom count on a search result.

```json
{
  "id": "17545365",
  "url": "https://www.airbnb.com/rooms/17545365",
  "title": "Home in East Austin",
  "latitude": 30.25741,
  "longitude": -97.73366,
  "description": "Downtown Casa - neighborhood feel, close to it all",
  "rating": 4.87,
  "reviews": 601,
  "badges": ["Guest favorite"],
  "price": {
    "originalPrice": "$607",
    "discountedPrice": "$447",
    "qualifier": "for 3 nights",
    "breakdown": [{ "description": "3 nights x $149.00", "price": "$447.00" }]
  }
}
```

### Get Airbnb property details

[`hasdata_airbnb_property_getAirbnbPropertyDetails`](https://docs.hasdata.com/apis/airbnb/property?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp)

One listing in full, by its URL.

| Parameter | Type | Required | Notes |
| :--- | :--- | :--- | :--- |
| `url` | string | yes | An Airbnb listing URL, the `url` field from a search result |

Returns `title`, an `overview` array like `["4 guests", "2 bedrooms", "2 beds", "2 baths"]`, the full `description`, `rating`, `reviews`, `address`, `latitude`, `longitude`, a `photos` array, `guestCapacity`, a `host` object, an `amenities` array, and `safetyAndPropertyInfo`. The `host` object carries `name`, `isSuperhost`, `isVerified`, the host's own `reviews`, `rating` and `yearsHosting`. Each amenity carries a `title`, a `type`, an optional `description` and an `available` flag, so a filter reads `available`, it does not assume every listed amenity is present.

```json
{
  "id": "17545365",
  "title": "Downtown Casa - neighborhood feel, close to it all",
  "overview": ["4 guests", "2 bedrooms", "2 beds", "2 baths"],
  "rating": 4.87,
  "reviews": 601,
  "address": "Austin, Texas, United States",
  "guestCapacity": 4,
  "host": { "name": "Deanna", "isSuperhost": true, "isVerified": true, "reviews": 1273, "rating": 4.86, "yearsHosting": 11 },
  "amenities": [
    { "title": "Wifi", "type": "SYSTEM_WI_FI", "available": true },
    { "title": "Free washer – In unit", "type": "SYSTEM_WASHER", "available": true }
  ]
}
```

## Errors and failure paths

Your client almost never sees an HTTP error code from a tool call. The MCP layer answers 200 and puts the failure inside the result, with `isError` set to `true` and the reason as text. The agent reads a message where you might expect a status line.

**A wrong key surfaces as tool output, not as a failed connection.** `tools/list` accepts any non-empty key and returns both tools, so the client completes its handshake and shows green. The first tool call then comes back with `isError: true` and the text `HasData API error: 401 Unauthorized`. Watch for that string, because nothing earlier in the flow reports the problem.

**A missing key is the one real HTTP error.** Authorization runs before any tool, and the connection itself fails with 401. CORS headers are present, and a browser client reads the status and not an opaque network failure.

**An argument that breaks a tool's schema is rejected before it becomes a scrape.** The server answers with `isError: true` and the text `MCP error -32602: Input validation error`, naming the offending field. Nothing is fetched and nothing is charged.

**A search with no availability returns a successful result with an empty `properties` array**, not an error. A location and date range with nothing open still comes back with `requestMetadata.status` set to `ok`. Test for the array length before you iterate.

**A listing that has been removed returns 400** with `requestMetadata.status` set to `error`. A URL from an old search can point at a home that is gone.

Results that carry data also carry a `requestMetadata.id` worth quoting in support.

## Pricing, free tier and limits

Each Airbnb tool costs **5 credits per successful call**. Response size does not change the price. A search page with dozens of stays costs the same as one with two.

The free trial is **1,000 credits over 30 days with no card**, which is 200 Airbnb calls. After that an active account keeps getting 100 credits topped up each day whenever its balance drops below 100, so a low-volume agent runs on the free tier indefinitely.

Paid plans start at **$49 a month** for 200,000 credits, which is 40,000 calls. The unit price falls with volume, from **$1.23 per 1,000 calls** on the entry plan to **$0.50** on Business, **$0.42** on Growth and **$0.37** on the largest [high-volume plans](https://hasdata.com/prices?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp).

Your plan also sets concurrency. The free trial allows 1 request at a time, Startup 15, Business 30, Growth 50, and the high-volume plans run from 200 to 1,500. Handle the overflow case defensively in anything unattended.

A request that comes back non-200 is not billed. A successful call that finds nothing is still a call.

## Tool selection

The `apis` query parameter decides which tools your agent sees. Fewer tools means less context spent on tool definitions, and fewer chances for the model to reach for the wrong one.

```
?apis=airbnb                     the two tools in this repo
?apis=airbnb,booking             add Booking.com stays
?apis=airbnb,google_maps         add Google Maps places
```

The parameter takes provider names like `airbnb` and individual API names like `airbnb_listing`. Misspelled names are ignored. If every name is wrong the request fails with 400, and the body lists both what it did not recognise and every valid value. Drop the parameter and the same endpoint exposes all 57 HasData tools.

## How it compares

Airbnb has no public search API. Its API program is a partner and co-host integration for managing your own listings, not a way to read the market. For searching stays and reading arbitrary listings, scraping the public pages is the only route, and this server does that behind a stable schema.

| | Airbnb partner API | This server |
| :--- | :--- | :--- |
| Purpose | Manage your own listings | Read the public market |
| Access | Partner approval | One key and one URL |
| Search across the market | No | Yes |
| Setup | Business onboarding | None |
| Output | Partner payloads | Structured JSON, price and rating pre-parsed |

**What this server does not do.** No booking, no messaging, no host dashboard, no guest data. It reads what a signed-out visitor can see.

## FAQ

### Is there an official Airbnb MCP server?

Airbnb does not publish one. This one is maintained by HasData and reads public pages, which is why it needs no Airbnb developer account.

### What is an Airbnb MCP server?

A server that exposes Airbnb data as tools an AI client can call. The client sends a tool call over the Model Context Protocol, the server fetches the data and returns structured JSON, and the model works with the result. This one exposes two tools and runs remotely.

### Do I need an Airbnb API key or partner account?

No. The only credential is your HasData key. There is no partner onboarding, because the tools read public Airbnb pages.

### Why does the search result not show beds or amenities?

Because Airbnb does not put them on the search card. They live on the property page, which the details tool reads by URL. Search to shortlist, then call the property tool for depth.

### Can I filter by guests, pets or dates?

Yes. The listing tool takes `checkIn`, `checkOut` and the `adults`, `children`, `infants` and `pets` counts, and returns availability and pricing for that party and window.

### Can I use this together with other HasData APIs?

Yes. The `apis` parameter takes a list, and `?apis=airbnb,booking` gives your agent Airbnb plus Booking.com. [Drop the parameter](#tool-selection) and you get everything.

### Compliance and personal data

HasData accesses publicly available data only. A platform's terms may restrict automated access, and you are responsible for your own compliance. Where the data you collect includes personal information, make sure you have a lawful basis for it under GDPR, CCPA or the equivalent rules in your jurisdiction.

## HasData links

| | |
| :--- | :--- |
| Product page and request builder | [Airbnb Scraper API](https://hasdata.com/apis/airbnb-api?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp) |
| Server documentation | [MCP server docs](https://docs.hasdata.com/mcp-server?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp) |
| All 57 tools in one server | [HasData/hasdata-mcp](https://github.com/HasData/hasdata-mcp) |
| Client walkthroughs | [MCP clients and integrations](https://hasdata.com/integrations/mcp?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp) |
| Everything else we scrape | [Airbnb Scraper API and 54 more](https://hasdata.com/apis/?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp) |
| Plans and credit costs | [Plans and credit costs](https://hasdata.com/prices?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp) |
| Keys and usage | [HasData dashboard](https://app.hasdata.com?utm_source=github&utm_medium=syndication&utm_campaign=airbnb-mcp) |

## Development

This repository is configuration and documentation for a remote server. There is no build step and nothing to containerize.

The tests in `test/` assert the tool contract, the part that can break without a commit here. They check that `?apis=airbnb` returns exactly two tools, that every tool still declares its required parameters, that no name changed, and that the key in use is actually accepted. That last check calls a tool for real and costs 5 credits, which is the price of a canary that can fail for the right reason.

```bash
# macOS and Linux
HASDATA_API_KEY=your_key_here npm test

# Windows PowerShell
$env:HASDATA_API_KEY="your_key_here"; npm test
```

The same suite runs in CI on every push and once a week on a schedule, because the upstream tool list can change without anyone touching this repository. A failure means the tool list moved, the key stopped working, or the endpoint was unreachable, and the assertion message says which.

## Contributing

Corrections to the tool tables and the response samples are the most useful contribution, because those are the parts that drift. Include the call you made and the response you got. Pull requests from forks run the suite without a key, and the live checks skip instead of going red.

## License

MIT. See [LICENSE](LICENSE).
