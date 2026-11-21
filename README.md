# EB Games Pokémon TCG monitor

Checks EB Games (ebgames.com.au) every 15 minutes, one second past each quarter hour, using GitHub Actions, for **sealed Pokémon TCG product** and posts to Discord. It works the same way as `jb-monitor`.

- **Alerts**: an embed when a product is **newly listed**, comes **back in stock**, or its **pre-orders reopen**. Each one shows the price, status, pre-order deposit, release date, delivery and Click & Collect availability, any promotion, and an image.
- **Live board**: one card for every product currently available, ordered Pre-orders → In stock → Click & Collect only. It's edited in place on every run, so you pin it once. A message holds 9 cards plus the header. When the count crosses 9, 19 and so on, the board is reposted, and you re-pin it.

It tracks every Pokémon "Trading Cards Type" except **singles**: boosters, booster bundles, blisters, booster boxes, ETBs, collections and decks. If EB adds a new type, it's picked up automatically. To also track sleeves, binders, playmats and deck boxes, set `INCLUDE_ACCESSORIES=true`.

## Statuses

| Status | Meaning on EB | Alerts? |
|---|---|---|
| 🟣 Pre-order | Pre-order open (deposit shown) | yes |
| 🟢 In stock | Can add to cart, home delivery available | yes |
| 🟠 Click & Collect only | Can add to cart, but delivery is unavailable | yes |
| ⚫ Presold out | Pre-order allocation gone (EB shows "PRESOLD OUT") | no (alerts when it reopens) |
| ⚫ Sold out | Can't be bought, or has dropped out of EB's results | no (alerts when it returns) |

EB doesn't expose stock counts or per-customer limits, so neither appears in alerts or on the board.

## Setup

1. **Create Discord webhooks.** Go to Channel settings → Integrations → Webhooks → New Webhook → Copy URL. You need one for alerts, for example `#eb-alerts`. A second one for the board, for example `#eb-stock`, is optional.
2. **Push this folder to its own public GitHub repo.** It must be public because the runs wait up to about 7 minutes each, which is around 20k runner-minutes a month. That's free on public repos only.
   ```bash
   git init && git add . && git commit -m "EB Games Pokemon monitor"
   git remote add origin https://github.com/<you>/eb-monitor.git
   git push -u origin main
   ```
3. **Add the settings** under repo → Settings → Secrets and variables → Actions:

   | Kind | Name | Value |
   |---|---|---|
   | Secret | `DISCORD_WEBHOOK_URL` | Alerts webhook (required) |
   | Secret | `DISCORD_BOARD_WEBHOOK_URL` | Board webhook (optional) |
   | Variable | `DISCORD_PING` | `<@USER_ID>` or `<@&ROLE_ID>` (optional) |
   | Variable | `INCLUDE_ACCESSORIES` | `true` (optional) |
4. **Start it.** Go to Actions → *EB Games Pokemon monitor* → Run workflow. The first run seeds the state without alerting and posts the board. Pin the board message. To preview an alert, run the workflow again with **Send a sample alert** ticked.

## How it works

- EB's web pages sit behind a Cloudflare bot check. The search page, though, loads its results from `GET /search/query`, which returns plain JSON. The monitor reads that endpoint, the same one the website uses.
- Each check makes two requests, plus more pages if needed. The first reads the Pokémon subcategory's "Trading Cards Type" facet. The second asks for every type except singles. With accessories on, a third search (`q=pokemon` in TCG Accessories) runs, and only titles mentioning Pokémon are kept.
- `state.json` remembers each SKU and its status, and is committed back only when something changes. Items that drop out of results are marked sold out, so they alert again when they return. If a check comes back empty, it's ignored rather than marking everything sold out.

## Running locally

```bash
python monitor.py --dry-run                         # fetch + print table and board JSON, no Discord, no state
python monitor.py --test-alert                      # send one sample alert (needs .env)
python monitor.py --loop 30 --state state.local.json   # near-live polling on an always-on machine
```

Loop mode refuses intervals under 15 seconds. It backs off for up to 10 minutes on errors and logs to `logs/polls.csv` and `logs/alerts.csv`.

## Caveats

- **Cloudflare is the main risk.** The JSON endpoint currently answers scripted requests, but Cloudflare may start challenging it, especially from GitHub's datacenter IPs. If it does, runs fail with "EB Games returned a Cloudflare challenge" and GitHub emails you. In that case, run it in loop mode from a home machine instead. The monitor doesn't try to get past the challenge.
- Release dates are plain dates. The monitor treats them as midnight AEST.
- GitHub's scheduled runs are best-effort. They're occasionally late or dropped, and they're disabled after 60 days without repo activity.
