# GridHop GPU Price & Carbon Index

[![mcptry](https://mcptry.com/badge/io.github.gridhopai/gpu-price-index)](https://mcptry.com/servers/io.github.gridhopai/gpu-price-index)

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

Or as a plugin, which keeps itself up to date:

```
/plugin install gridhop --marketplace gridhopai/gpu-price-index-mcp
```

On Claude Code older than 2.1.275: `/plugin marketplace add gridhopai/gpu-price-index-mcp`, then `/plugin install gridhop@gridhop`.

**Cursor** (`~/.cursor/mcp.json`)

```json
{ "mcpServers": { "gridhop": { "url": "https://gridhop.dev/v1/mcp" } } }
```

**Any other MCP client**: streamable HTTP, URL above, no authentication.

## What you can ask

- "What's the cheapest RTX 4090 I can rent right now?"
- "Find an 8×H100 machine in a datacenter, under $25 an hour."
- "Which A100 offer runs on the cleanest grid?"
- "Compare T4 prices on AWS, Azure and Google Cloud in Europe."
- "What does an 8×H100 machine cost on demand on AWS and on Azure, and how much cheaper is spot?"
- "Which GPUs with at least 48 GB can I rent, and what do they cost?"
- "My 5 TB dataset is in S3 in us-east-1. Where is an 8×H100 job of 200 hours cheapest, counting the cost of moving the data?"
- "I'll run this training six times this year at different providers. Is it worth copying the data out of AWS once?"
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
| `continents` | Any of `africa`, `asia`, `europe`, `north_america`, `oceania`, `south_america`. |
| `us_states` | Two-letter US state codes, e.g. `["TX", "CA"]`. Cloud regions count as the state the provider documents (AWS `us-east-1` is `VA`). |
| `tier` | `datacenter` drops independent marketplace hosts. |
| `max_price_hr` | Ceiling in USD per hour for the whole machine. |
| `providers` | Any of `vastai`, `runpod`, `nebius`, `nebius_spot`, `coreweave`, `lambda`, `ovh`, `aws`, `aws_spot`, `azure`, `azure_spot`, `gcp_spot`. `aws`, `azure`, `nebius`, `coreweave`, `lambda` and `ovh` are on-demand prices; the `_spot` ones are spot prices. |
| `include_interruptible` | Omit for on-demand prices plus AWS, Azure, Google Cloud and Nebius spot. `true` adds marketplace minimum bids. `false` drops all spot, which removes Google Cloud entirely. |
| `min_reliability` | Drop hosts below this provider reliability score. Default 0.9. |
| `max_carbon_g_kwh` | Only hosts on a grid at or below this carbon intensity. |
| `sort_by` | `price` (default), `carbon`, or `total`: the cost of the whole job, the default once `data_at`, `data_gb` and `hours` are given. |
| `data_at` | Where your dataset is now, as `cloud:region`: `aws:us-east-1`, `azure:eastus`, `gcp:us-central1`. Goes with `data_gb`. |
| `data_gb` | Size of the dataset, GB. |
| `hours` | How long you need the machine. |
| `jobs`, `months` | How many times the data will be needed outside its cloud, and over how many months. Default 1 and 1. |
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
| `state` | US state of the host, two-letter code, where known. |
| `tier` | `datacenter`, or `community` for an independent host on a marketplace. |
| `reliability` | The provider's own score, 0 to 1. Empty where the provider publishes none. |
| `stock` | RunPod only: its own stock level for that GPU and tier, `high`, `medium` or `low`. |
| `interruptible` | `true` for spot and bid prices: the machine can be taken back. |
| `data_transfer` | With `data_at`: the route to this machine (`same_region`, `same_cloud` or `internet`), what one copy of the dataset costs, the price per GB, and the hours the copy takes at 1 Gbit/s. |
| `job_total_usd` | With `hours` as well: machine hours plus the data transfer. |
| `break_even_hours` | Hours of running after which this offer's lower hourly price has paid for the move. Empty when it is not cheaper per hour than staying put. |
| `carbon_g_kwh` | Grid carbon intensity, g CO₂eq per kWh. Empty where no open data covers the grid. |
| `carbon_source` | `entsoe_gen` (Europe), `neso` (Great Britain) or `eia` (US). |
| `carbon_lag_h` | Age of the carbon reading in hours. 0 to 3 is live; US readings are 24 or 48 (same hour of day, one or two days earlier). |
| `est_gco2_per_hr` | Rough grams of CO₂ per hour: carbon intensity × nameplate GPU power × GPU count. |
| `grid_zone` | The power grid the reading refers to. Pass it to `grid_carbon` for more. |
| `carbon_uncertain` | `true` when part of that grid's generation has no fuel type and the mix is partly assumed. |

The answer wraps the offers with `observed_at` (snapshot time), `total_matching`, `warnings` (things the agent should tell you, such as stale carbon readings), and `sources`.

#### Counting the data

A cheaper GPU hour can be the dearer job once the dataset has to leave the cloud it lives in. Say where the data is and how big, and the tool prices the move for every offer at that cloud's list prices:

| Route | Price |
| --- | --- |
| Same cloud, same region | Free. |
| Same cloud, another region | The cloud's inter-region rate: AWS $0.02 per GB; Azure and Google Cloud $0.02 inside North America or Europe, $0.05 between them, $0.08 on the long routes. |
| Any other provider | The cloud's internet egress rate, tiered by volume: AWS from $0.09 per GB, Azure from $0.087, Google Cloud from $0.12, after the first 100 GB a month. |

The answer also gives:

- `stay_put`: the cheapest on-demand machine of that class in the region the data is in, to compare against. Empty when the cloud does not list the class there.
- `staging`: what it costs to take the dataset out `jobs` times over `months`, paying the egress every time or once into object storage that charges nothing to read from (Backblaze B2 at $6.95 per TB a month with free reads up to three times the stored size, Cloudflare R2 at $0.015 per GB a month). From the second move the one-off copy is usually the cheaper way; it suits data that changes little.

Example: 5 TB in AWS `us-east-1`, an 8×H100 machine for 200 hours (26 November 2026). Staying on AWS: $55.04 an hour, $11,008. Nebius in Finland: $36 an hour plus $441 to move the data, $7,641, ahead after 23 hours. With 100 TB the move costs $7,812 and takes about 220 hours at 1 Gbit/s, and staying put is cheaper unless the job runs past 410 hours.

These are list prices. Not counted: committed-use discounts, private interconnects, the clouds' free-egress programmes for customers who leave, storage at the destination, and bringing results back, all of which change the real bill. Copy time assumes one sustained speed; real links vary widely. Vast.ai hosts set their own bandwidth price, which is not in the data.

RTX 4090, 5090 and 3090 are consumer cards: only the marketplaces (Vast.ai, RunPod) rent them out, no cloud does. If you ask a cloud provider for one, or ask for a datacenter and none has it, the answer says so and adds `alternatives`: datacenter cards with at least as much memory (L4, A10, A6000, L40S and the like), cheapest first.

### `list_gpus`

One optional argument, `vram_min` (GB). Returns one row per GPU class, cheapest first, with `vastai_machines_listed` (machines of that class free to rent on Vast.ai now) and `runpod_stock` where those providers list it.

### `price_history`

| Argument | Meaning |
| --- | --- |
| `gpu_class` (required) | As for `find_offers`. `H100` returns one series per variant. |
| `granularity` | `daily` (default) or `hourly` (last 72 hours). |
| `days` | How far back, 1 to 90. Default 14. |

Each point has the cheapest on-demand price seen, the typical one (median of that day's hourly lows), the cheapest AWS, Azure, Google Cloud or Nebius spot price, all per GPU, and how many hourly snapshots the day has. A summary gives the low, the high and the change over the period. Prices follow `find_offers`' defaults: no marketplace minimum bids, no hosts under 0.9 reliability. History starts 1 November 2026 and is hourly from 10 November.

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

- **Prices**: Vast.ai and RunPod public listings, Nebius spot prices from Nebius's public spot-price API and its on-demand list prices via the [SkyPilot catalog](https://github.com/skypilot-org/skypilot-catalog) (checked against Nebius's pricing page), CoreWeave on-demand list prices from its public pricing page, Lambda on-demand list prices via the SkyPilot catalog (checked daily against Lambda's pricing page), OVHcloud on-demand prices from its public order catalog (European prices converted from euros at the ECB rate), AWS EC2 on-demand price files and AWS spot prices via the SkyPilot catalog, the Azure Retail Prices API (pay-as-you-go and Spot), and Google Cloud's Billing Catalog (Spot; vCPU + memory + GPU, added up per machine type).
- **Carbon intensity**: computed from each grid's generation mix with IPCC lifecycle emission factors. Generation data: ENTSO-E Transparency Platform (Europe), NESO Carbon Intensity API (Great Britain, CC BY 4.0), US Energy Information Administration (US).

## What to keep in mind

- **Prices are list prices at snapshot time**, before any fee, and they move. A marketplace machine can be rented by someone else a minute later.
- **AWS, Azure and Nebius have both on-demand and spot prices; Google Cloud has spot only.** Its on-demand prices are not in the index yet, so every Google Cloud row is interruptible and `include_interruptible: false` returns none of them.
- **Nebius spot prices move fast**: with demand, as often as every 15 minutes, several-fold within a day, and they can exceed the on-demand price. The index reads them once an hour.
- **AWS spot prices are not live.** AWS has no key-less feed of current spot prices, so they come from the SkyPilot catalog, refreshed a few times a day: the cheapest availability zone of each region, up to about half a day old. (Until 16 November 2026 this index read an AWS feed that turned out to be frozen since 2024; those prices were wrong and have been removed from the history.)
- **CoreWeave is on-demand only, whole 8-GPU nodes, with no region.** Its pricing page gives one price for all regions, so those rows have no country and no carbon value, and `region_filter` leaves them out. The spot prices on that page are updated by hand every few days, not a live market, so they are not in the index. A CoreWeave account is opened through its sales team.
- **Lambda is on-demand list prices with no region.** Lambda charges the same in every region, and its instances are first come, first served and often sold out, so those rows say what a machine costs, not where one is free. They have no country and no carbon value, and `region_filter` leaves them out.
- **OVHcloud's European prices have no country.** Its catalog gives one euro price for datacenters in several countries and does not say which datacenter sells which machine, so those rows have no country or carbon value and location filters leave them out. Rows from its US catalog are in the US. The dollar price of a European row moves a little from day to day with the exchange rate.
- **Availability is known for two providers only.** Every Vast.ai row is a machine that can be rented now, and RunPod publishes a stock level (`stock`). AWS, Azure, Google Cloud, CoreWeave, Lambda, Nebius and OVHcloud publish prices without saying whether a machine is free.
- **Cloud rows are list prices, not machines confirmed free.** On AWS and Azure the large GPU types usually need a quota increase first, and Azure Spot capacity can be unavailable. Some Azure sizes are left out until their GPU count is confirmed (NCads A10 v4, RTX PRO 6000, GB200), as are sizes with a fraction of a GPU.
- **Google Cloud rows are what a machine would cost, not proof it is available.** Spot GPU capacity is often unavailable; the tool says so when it shows those rows.
- **Marketplace bid prices are minimum bids**, a floor rather than the price of keeping the machine, so they are left out unless you ask for them.
- **Carbon values cover about half of the offers**: Europe, Great Britain and most US states. Elsewhere the field is empty rather than guessed.
- **US carbon readings are one or two days old** (same hour of day), because the source publishes late. `carbon_lag_h` says how old each reading is.
- **Some readings are estimates.** Where a grid operator reports part of its generation without a fuel type, the offer is marked `carbon_uncertain`.
- Carbon intensity counts what is generated in a grid zone, not imports. The CO₂-per-hour figure uses nameplate GPU power only; it is an estimate, not a measurement.
- **Location filters add up**: a host matches if it is in any country, continent or US state you list (`US` plus `us_states` means only those states). Offers with no published location (Lambda, CoreWeave, RunPod) match none of them.
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

## Licence

The files in this repository are under the [MIT licence](LICENSE). The hosted server and its data are a separate thing: free to use, provided as is, with the data sources credited above.

Questions, wrong prices, a GPU class you could not find: write to [support@gridhop.dev](mailto:support@gridhop.dev) or [open an issue](https://github.com/gridhopai/gpu-price-index-mcp/issues).
