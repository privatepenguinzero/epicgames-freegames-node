# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

| Command | Usage |
|---------|-------|
| `npm run start` | Run in development via vite-node |
| `npm run build` | Build to `dist/` (rimraf + tsc) |
| `npm run lint` | Type-check (`tsc --noEmit`) + lint (eslint) |
| `npm run docs` | Generate TypeDoc in `docs/` |
| `npm run docker:build` | Build Alpine Docker image |
| `npm run docker:build-debian` | Build Debian Docker image (fallback for Chromium issues) |

Config is loaded from `config/config.json` or `config/config.json5` (JSON5 syntax supported). Environment variables also work for single-account setups. See `AppConfig` in `src/common/config/classes.ts` for every option and its env var key.

## Project Overview

Automates checking for free games on the Epic Games Store and sends a checkout link via one of 12 notification methods. Uses **device code OAuth** (no password needed — logs in via Epic's device authorization flow).

Flow (in `src/index.ts`'s `main()`):
1. Start Express web server (for device code login redirects)
2. For each account (concurrency-limited via `p-queue`):
   - Convert imported cookies if present
   - Launch headless Chromium (puppeteer-extra + stealth)
   - **Login**: try device auth refresh → cookie login → new device code auth (polled with timeout)
   - Check/accept EULAs
   - Discover free games via Epic's GraphQL/Store APIs
   - Validate offers (country, expiry, blacklist), check ownership via offers validation API
   - Generate checkout URL, send notification
3. Clean up browser processes and temp files

## Architecture

### Source layout

```
src/
  index.ts                  — Entry point, main loop orchestrating everything
  purchase.ts               — Checkout URL generation
  notify.ts                 — Notification dispatch (routes to correct notifier by type)
  device-login.ts           — Epic device code OAuth flow (web redirect + polling)
  eula-manager.ts           — EULA status check / auto-accept

  common/
    config/
      classes.ts            — AppConfig, AccountConfig, notifier configs, enums
      setup.ts              — Config loading (JSON5 file → class-transformer → class-validator)
    constants.ts            — All Epic Games API endpoint URLs (GraphQL, OAuth, store)
    puppeteer.ts            — Puppeteer launch args, stealth plugin, browser cleanup
    cookie.ts               — Cookie persistence (tough-cookie file store), cookie format conversion
    device-auths.ts         — OAuth token persistence on disk
    server.ts               — Express web server setup
    localtunnel.ts          — LocalTunnel for exposing web server without port forwarding
    logger.ts               — pino logger (pretty-printed)

  puppet/
    base.ts                 — Base class: shared browser/page management, request() via page.evaluate
    login.ts                — Cookie-based session refresh (EPIC_SSO_RM cookie)
    free-games.ts           — Free game discovery (weekly, catalog promotions, or both)

  notifiers/
    notifier-service.ts     — Abstract base class
    index.ts                — Re-exports all notifiers
    discord.ts, email.ts, telegram.ts, pushover.ts, apprise.ts, slack.ts,
    gotify.ts, homeassistant.ts, bark.ts, ntfy.ts, webhook.ts, local.ts

  interfaces/
    types.ts                — Core TypeScript interfaces (OfferInfo, purchase types, etc.)
    notification-reason.ts  — NotificationReason enum
    promotions-response.ts, search-store-query-response.ts, get-catalog-offer-response.ts,
    bundles-content.ts, offers-validation.ts, page-slug-mapping.ts, product-info.ts,
    offer-response.ts
```

### Key classes (all in `src/common/config/classes.ts`)

- **`AppConfig`** — root config: cron schedule, accounts, notifiers, search strategy, browser settings, device auth client credentials, etc.
- **`AccountConfig`** — per-account config (email + optional override notifiers)
- **`WebPortalConfig`** — web server settings (base URL, listen opts, localtunnel toggle)
- **`NotifierConfig`** subclasses — `EmailConfig`, `DiscordConfig`, `TelegramConfig`, `PushoverConfig`, `AppriseConfig`, `GotifyConfig`, `SlackConfig`, `HomeassistantConfig`, `BarkConfig`, `NtfyConfig`, `WebhookConfig`, `LocalConfig`
- **`SearchStrategy`** enum — `WEEKLY`, `PROMOTION`, or `ALL` (default)

### Puppeteer strategy

The `PuppetBase` class manages a shared page. Its `request()` method calls `page.evaluate()` to `fetch()` Epic's GraphQL API from within the browser context — this means browser cookies (including `EPIC_BEARER_TOKEN`) are sent automatically. Cookies are synced between the tough-cookie file store and Puppeteer on each `setupPage()`/`teardownPage()`.

### Search strategies (`PuppetFreeGames.getFreeGames()`)

1. **`WEEKLY`** — Fetches `freeGamesPromotions` endpoint, then resolves actual offer IDs via `getMappingByPageSlug` GraphQL query (falling back to CMS API for older product pages)
2. **`PROMOTION`** — Searches the catalog GraphQL for any item with `discountPrice === 0`
3. **`ALL`** — Runs both strategies and deduplicates; continues if at least one succeeds

### Login flow (in order of attempt)

1. **Device auth refresh** (`DeviceLogin.refreshDeviceAuth`) — uses refresh_token from stored OAuth response
2. **Cookie login** (`PuppetLogin.refreshCookieLogin`) — uses EPIC_SSO_RM cookie to visit login redirect
3. **New device code auth** (`DeviceLogin.newDeviceAuthLogin`) — sends notification with login URL; user visits → redirected to Epic → completes login → redirected back → tokens saved

### Notifications

12 notification types, each implementing `NotifierService.sendNotification(account, reason, url)`. Notifications can be configured globally or per-account. They fire in parallel via `Promise.all`. The `reason` argument controls the message body (LOGIN, PURCHASE, TEST, PRIVACY_POLICY_ACCEPTANCE).

## Important details

- **Node.js >= 22** required (ESM, `.nvmrc` = 22)
- **No password-based login** — only device code OAuth (since v5). `account.password`/`account.totp` were removed.
- **Cron schedule should be <= every 6 hours** — the device auth refresh token expires after 8 hours. Default is `0 0,6,12,18 * * *`.
- **Quick single-account setup**: set `EMAIL` env var, optionally `RUN_ON_STARTUP=true`, plus notifier env vars — no JSON config file needed.
- **Cookie import**: Export cookies from browser (EditThisCookie extension) as `<email>-cookies.json` for initial auth. Only `EPIC_SSO_RM`, `EPIC_SESSION_AP`, `EPIC_DEVICE` cookies are imported.
- **Error screenshots**: Saved to `config/errors/` on Puppeteer failures.
- **Logging**: Uses pino with pino-pretty formatting. Level controlled by `logLevel` config or `LOG_LEVEL` env.
- **Docker memory**: Recommended 2GB limit (`-m 2g`) due to Chromium memory consumption.
