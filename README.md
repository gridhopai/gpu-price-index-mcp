# GridHop GPU Price & Carbon Index

A free, read-only [MCP](https://modelcontextprotocol.io) server that tells an AI agent what a GPU costs to rent right now, across several clouds and marketplaces, and how clean the electricity is where that GPU sits.

No account, no API key, nothing to install. It cannot rent anything; it only answers questions.

Want your agent to actually run the job, not just price it? That is [GridHop](https://gridhop.dev/waitlist?src=github), coming next. [Join the waitlist](https://gridhop.dev/waitlist?src=github).

```
https://gridhop.dev/v1/mcp
```

## Add it to your agent

**Claude Code**

```
claude mcp add --transport http gridhop https://gridhop.dev/v1/mcp
```

**Cursor** (`~/.cursor/mcp.json`)

```json
{ "mcpServers": { "gridhop": { "url": "https://gridhop.dev/v1/mcp" } } }
```

**Any other MCP client**: streamable HTTP, URL above, no authentication.

## What you can ask

- "What's the cheapest RTX 4090 I can rent right now?"
- "Find an 8×H100 machine in a datacenter, under $25 an hour."
- "Which A100 offer runs on the cleanest grid?"
- "Compare L4 prices on AWS and Google Cloud in Europe."
- "Which GPUs with at least 48 GB can I rent, and what do they cost?"
- "Has the H100 price gone up or down this week?"
- "Which European grid is cleanest right now, and when is power cheapest in Texas?"

## The tools

| Tool | What it answers |
| --- | --- |
| `find_offers` | The cheapest or cleanest offers for a GPU class right now. |
| `list_gpus` | Which GPU classes are on offer: VRAM, providers, machine sizes, cheapest on-demand and spot price. |
| `price_history` | How the cheapest price of a class has moved, per day or per hour. |
| `grid_carbon` | How clean and how expensive the electricity is on each power grid, now and at its best hour. |

### `find_offers`

| Argument | Meaning |
| --- | --- |
| `gpu_class` (required) | GPU model. Loose forms work: `4090`, `RTX 4090`, `H100`, `A100-80GB`. |
| `count` | GPUs on one machine. Default 1. |
| `vram_min` | Minimum VRAM per GPU, GB. |
| `region_filter` | Two-letter country codes, e.g. `["US", "DE"]`. |
| `tier` | `datacenter` drops independent marketplace hosts. |
| `max_price_hr` | Ceiling in USD per hour for the whole machine. |
| `providers` | Any of `vastai`, `runpod`, `aws_spot`, `gcp_spot`. |
| `include_interruptible` | Omit for on-demand prices plus AWS and Google Cloud spot. `true` adds marketplace minimum bids. `false` drops all spot. |
| `min_reliability` | Drop hosts below this provider reliability score. Default 0.9. |
| `max_carbon_g_kwh` | Only hosts on a grid at or below this carbon intensity. |
| `sort_by` | `price` (default) or `carbon`. |
| `limit` | Up to 50 offers. Default 10. |

Each offer comes back with provider, GPU class and count, price per hour, country and region, host tier, reliability score, whether it can be interrupted, and where known: grid carbon intensity (g CO₂/kWh), how old that reading is, the grid zone it refers to, and a rough estimate of grams of CO₂ per hour for the GPUs at full load.

One offer from a real answer to "cheapest RTX 4090 in the US" (15 November 2026):

```json
{
  "provider": "vastai",
  "gpu_class": "RTX4090",
  "gpu_count": 1,
  "vram_gb": 24,
  "price_usd_hr": 0.575,
  "price_per_gpu_hr": 0.575,
  "region": "Texas",
  "country": "US",
  "tier": "community",
  "reliability": 0.995,
  "interruptible": false,
  "carbon_g_kwh": 410,
  "carbon_source": "eia",
  "carbon_lag_h": 48,
  "est_gco2_per_hr": 184,
  "grid_zone": "US-TEX-ERCO",
  "carbon_uncertain": false
}
```

| Field | Meaning |
| --- | --- |
| `price_usd_hr`, `price_per_gpu_hr` | List price for the whole machine, and per GPU. |
| `tier` | `datacenter`, or `community` for an independent host on a marketplace. |
| `reliability` | The provider's own score, 0 to 1. Empty where the provider publishes none. |
| `interruptible` | `true` for spot and bid prices: the machine can be taken back. |
| `carbon_g_kwh` | Grid carbon intensity, g CO₂eq per kWh. Empty where no open data covers the grid. |
| `carbon_source` | `entsoe_gen` (Europe), `neso` (Great Britain) or `eia` (US). |
| `carbon_lag_h` | Age of the carbon reading in hours. 0 to 3 is live; US readings are 24 or 48 (same hour of day, one or two days earlier). |
| `est_gco2_per_hr` | Rough grams of CO₂ per hour: carbon intensity × nameplate GPU power × GPU count. |
| `grid_zone` | The power grid the reading refers to. Pass it to `grid_carbon` for more. |
| `carbon_uncertain` | `true` when part of that grid's generation has no fuel type and the mix is partly assumed. |

The answer wraps the offers with `observed_at` (snapshot time), `total_matching`, `warnings` (things the agent should tell you, such as stale carbon readings), and `sources`.

### `list_gpus`

One optional argument, `vram_min` (GB). Returns one row per GPU class, cheapest first.

### `price_history`

| Argument | Meaning |
| --- | --- |
| `gpu_class` (required) | As for `find_offers`. `H100` returns one series per variant. |
| `granularity` | `daily` (default) or `hourly` (last 72 hours). |
| `days` | How far back, 1 to 90. Default 14. |

Each point has the cheapest on-demand price seen, the typical one (median of that day's hourly lows), the cheapest AWS or Google Cloud spot price, all per GPU, and how many hourly snapshots the day has. A summary gives the low, the high and the change over the period. Prices follow `find_offers`' defaults: no marketplace minimum bids, no hosts under 0.9 reliability. History starts 1 November 2026 and is hourly from 10 November.

### `grid_carbon`

| Argument | Meaning |
| --- | --- |
| `region_filter` | Two-letter country codes. |
| `grid_zone` | One grid by id, e.g. `SE-SE3`, `US-TEX-ERCO`. The ids match `grid_zone` on offers. |
| `sort_by` | `carbon` (default, cleanest first) or `power_price`. |
| `limit` | Up to 60 grids. Default 20. |

Per grid: carbon intensity now and how old the reading is, the low and high of the last 24 hours and when, the hour of day that is usually cleanest, the day-ahead wholesale power price now and the cheapest hour in the next 24. Wholesale prices cover Europe (ENTSO-E), Texas (ERCOT) and California (CAISO); a GPU host may pay a fixed tariff, so a cheap hour on the grid does not by itself make a GPU cheaper.

## Where the data comes from

Refreshed every hour. Every result carries `observed_at`, the time of the snapshot.

- **Prices**: Vast.ai and RunPod public listings, the AWS EC2 spot price feed, and Google Cloud's Billing Catalog (vCPU + memory + GPU, added up per machine type).
- **Carbon intensity**: computed from each grid's generation mix with IPCC lifecycle emission factors. Generation data: ENTSO-E Transparency Platform (Europe), NESO Carbon Intensity API (Great Britain, CC BY 4.0), US Energy Information Administration (US).

## What to keep in mind

- **Prices are list prices at snapshot time**, before any fee, and they move. A marketplace machine can be rented by someone else a minute later.
- **Google Cloud rows are what a machine would cost, not proof it is available.** Spot GPU capacity is often unavailable; the tool says so when it shows those rows.
- **Marketplace bid prices are minimum bids**, a floor rather than the price of keeping the machine, so they are left out unless you ask for them.
- **Carbon values cover about half of the offers**: Europe, Great Britain and most US states. Elsewhere the field is empty rather than guessed.
- **US carbon readings are one or two days old** (same hour of day), because the source publishes late. `carbon_lag_h` says how old each reading is.
- **Some readings are estimates.** Where a grid operator reports part of its generation without a fuel type, the offer is marked `carbon_uncertain`.
- Carbon intensity counts what is generated in a grid zone, not imports. The CO₂-per-hour figure uses nameplate GPU power only; it is an estimate, not a measurement.
- Outside the US, Vast.ai gives only a host's country, so countries with several grid zones use one default zone. `grid_zone` shows which.

## What is logged

To see whether the server works and what people look for, each request is logged with: the name and version your agent reports for itself, the country Cloudflare saw the request come from, the tool called, its arguments, and the outcome. **No IP address, account, or anything else that identifies a person is stored.**

## Limits

60 requests a minute per client. Beyond that the server answers `429` with a `Retry-After` header. A normal agent session uses a handful.

## Status

Early and free. The server may change; breaking changes will get a new path (`/v2/mcp`).

## What comes next: GridHop

This index is the first public piece of GridHop, a broker for AI agents. Your agent hands over a GPU job with a budget and a deadline; GridHop picks the machine and the hour across marketplaces, runs it, and returns the results. One prepaid balance, spend caps enforced on our side, scheduling by price and by grid carbon.

It is not open yet. **[Join the waitlist](https://gridhop.dev/waitlist?src=github)** to hear when it is, and to tell us what you would run.

Questions, wrong prices, a GPU class you could not find: write to [support@gridhop.dev](mailto:support@gridhop.dev) or [open an issue](https://github.com/gridhopai/gpu-price-index-mcp/issues).
