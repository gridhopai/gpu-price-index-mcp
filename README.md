# GridHop GPU Price & Carbon Index

A free, read-only [MCP](https://modelcontextprotocol.io) server that tells an AI agent what a GPU costs to rent right now, across several clouds and marketplaces, and how clean the electricity is where that GPU sits.

No account, no API key, nothing to install. It cannot rent anything; it only answers questions.

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

## The tool: `find_offers`

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

To see whether the server works and what people look for, each request is logged with: the name and version your agent reports for itself, the country Cloudflare saw the request come from, the search arguments, and the outcome. **No IP address, account, or anything else that identifies a person is stored.**

## Limits

60 requests a minute per client. Beyond that the server answers `429` with a `Retry-After` header. A normal agent session uses a handful.

## Status

Early and free. It is the first public piece of GridHop, a broker that will let an agent run a GPU job where and when compute is cheapest and cleanest. The server may change; breaking changes will get a new path (`/v2/mcp`).

Questions, wrong prices, a GPU class you could not find: please [open an issue](https://github.com/gridhopai/gpu-price-index-mcp/issues).
