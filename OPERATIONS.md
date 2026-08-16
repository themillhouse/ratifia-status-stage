# Operations — stage monitoring (internal)

Internal uptime monitoring for the Ratifia **stage** environment. This is **not** a customer-facing status page.

The customer-facing page is [`themillhouse/ratifia-status`](https://github.com/themillhouse/ratifia-status), which monitors production only and publishes to `status.ratifia.com`.

## Why this is a separate repo

Upptime has no per-site option to monitor a site while hiding it from the generated status page — one repo produces one page showing everything in it. So keeping stage off the customer page requires keeping stage in its own repo.

Stage does not belong on a public status page. A customer seeing "Stage API — down" learns nothing useful, and stage instability would drag down the published production uptime figures that the page exists to communicate.

## Why this repo is public with no page

Public, because GitHub Actions are free and unlimited on public repositories; at a 5-minute cadence a private repo would run several times over the included minute quota.

No page, because **GitHub Pages is intentionally not enabled here**. Upptime still runs every check, commits the history, and opens incident issues — which is the entire point of stage monitoring. Only the published website is omitted.

**Do not enable Pages on this repo**, and do not point a DNS record at it.

## What is monitored

| Component | Check | Notes |
|---|---|---|
| Stage API | `GET https://api.stage.ratifia.com/health` | Not gated — the stage gate fronts the dashboard, not the API. |
| Stage Dashboard | `GET https://app.stage.ratifia.com/sign-in` | Requires the stage gate cookie (below). `/sign-in` rather than `/`, since Clerk answers a signed-out root with 404. |

## The stage gate

`app.stage.ratifia.com` sits behind a Traefik rule that 307s any request lacking the `ratifia_stage` cookie to `https://ratifia.com/stage`. The check passes the gate by sending that cookie from an encrypted repository secret.

- **Secret name:** `STAGE_COOKIE_TOKEN`
- **Value:** must equal the `stageToken` in the Traefik rule (`tf-do-resources`, `apps/stage-ratifia`); it is the same opaque token the landing app sets as the cookie value after a successful password entry.

Worth being clear-eyed about: the gate keeps stage out of search engines and away from casual visitors. It is not a security boundary, and putting its token in CI does not make it one. Do not put anything in stage that would matter if the token leaked. If it does leak, rotate `STAGE_COOKIE_TOKEN` in both the Traefik rule and this repo's secret.

## Setup checklist

- [ ] **`GH_PAT` secret** — PAT with `repo` and `workflow` scopes, for committing results and opening incident issues.
- [ ] **`STAGE_COOKIE_TOKEN` secret** — the gate token described above.
- [ ] **Confirm both checks go green** — if Stage Dashboard reports down with a 307, the cookie is wrong or the Traefik token was rotated.
- [ ] **Alert routing** — an internal channel. Stage alerts must never reach a customer-facing channel.
- [ ] **Do not enable GitHub Pages.**

## A note on alert fatigue

Stage is expected to break; that is what stage is for. If these alerts become noise they will be ignored, and the prod alerts sharing that channel will be ignored with them.

Route stage to a **separate, lower-priority channel** than production, and tune or drop a check that flaps rather than learning to ignore it.
