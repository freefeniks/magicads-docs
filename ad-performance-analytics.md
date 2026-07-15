---
title: "Ad Performance Analytics"
description: "Purchase, install, configure and use the MagicAds Ad Performance Analytics plugin — connect Meta, Google Ads and TikTok, measure real ROAS, and attribute it to the creatives you generated in MagicAds."
---

# Ad Performance Analytics

Ad Performance Analytics is a paid plugin for the MagicAds platform. It closes the **create → publish → measure** loop: connect your Meta, Google Ads and TikTok ad accounts, pull spend, conversions and revenue into one unified cockpit, and attribute that performance back to the specific creatives you generated inside MagicAds.

Unlike the generation studios, this plugin does not create or publish anything — it **reads** reporting data from the ad networks and matches it to your creatives. Connecting accounts, syncing and every dashboard are free; the only credit-metered action is generating an AI performance analysis.

This guide covers the full lifecycle: where to buy it, how to install it, how to register the developer apps on each ad network, how to wire up the credentials, how to configure pricing and access, what your end users must do, and how every screen works.

---

## 1. What you get

Ad Performance Analytics ships five user-facing screens plus an admin configuration screen.

| Screen | What it does |
|--------|--------------|
| **Overview** | Unified cockpit — headline KPIs with period-over-period deltas, a spend-vs-revenue trend chart with per-day ROAS, a by-network breakdown, budget pacing and fatigue alerts. |
| **Creatives** | Creative-level ROAS leaderboard — ranks the creatives you generated in MagicAds by real ad performance, with a top-performer spotlight, attribution coverage meter and a manual ad-to-creative linker. |
| **Campaigns** | Every campaign across all networks, rolled up and ranked, with a budget-share bar and color-coded ROAS. |
| **AI Insights** | On-demand, credit-metered AI analysis — a headline verdict, health read, wins, watch-outs and prioritized, categorized recommendations. |
| **Connections** | Connect / pause / sync / disconnect ad accounts per network, with encrypted tokens and live sync status. |
| **Admin config** | Per-provider API credentials, feature + free-tier switches, AI insight pricing. |

### Supported networks

| Network | Reads from | Credentials the admin supplies |
|---------|-----------|-------------------------------|
| **Meta Ads** (Facebook / Instagram) | Meta Marketing API (Graph) | App ID + App Secret |
| **Google Ads** | Google Ads API (GAQL) | OAuth Client ID + Secret + Developer Token (+ optional Manager/MCC ID) |
| **TikTok Ads** | TikTok Marketing API | App ID + App Secret |

AI insights are generated with the platform's existing **OpenAI** key (from AI Settings) — no separate key for this plugin.

<Note>
This is a self-hosted product, so **each install uses its own developer apps** on Meta, Google and TikTok. There is no shared vendor app — you (the admin) register the apps once, and your users then connect their own ad accounts through them. Section 6 walks through each registration.
</Note>

---

## 2. Where to purchase

Ad Performance Analytics is a **paid plugin** distributed through the in-app plugin marketplace. Purchasing and installation both happen inside your MagicAds admin.

1. Sign in as an **admin**.
2. Go to **Admin → General Settings → Plugins**.
3. Find the **Ad Performance Analytics** card in the marketplace catalog.
4. The card CTA depends on your license and purchase state:
   - **Free / already owned** → installs directly.
   - **Paid** → routes you to the plugin checkout to complete the purchase.
   - **Extended License holders** → plugins flagged "free for Extended License" install without an extra purchase. Your license tier comes from the activation you performed when setting up MagicAds.

---

## 3. Install / activate

Install it from the marketplace card:

1. **Admin → General Settings → Plugins → Ad Performance Analytics**.
2. Click **Install** on the card.

Behind the scenes the platform downloads the plugin archive, unpacks it, runs its migrations and flips the extension's `installed` flag. This creates five database tables:

- `ad_analytics_settings` — the single settings row (feature flags, AI pricing, encrypted provider credentials).
- `ad_accounts` — each user's connected ad accounts, with encrypted tokens.
- `ad_metrics` — normalized daily metric rows (one per ad, per day, per network).
- `creative_ad_links` — the attribution bridge between an external ad and a MagicAds creative.
- `ad_insights` — cached AI analyses.

It also adds an `ad_analytics_feature` toggle to the plans table and registers the plugin's scheduled sync.

<Warning>
Installation only makes the routes, tables and screens exist. The plugin stays hidden from users until you **enable the feature**, **configure at least one network's credentials**, and grant access by plan (next sections). On a fresh install everything is off.
</Warning>

To remove the plugin later, click **Uninstall** on the same card. Uninstall drops the plugin tables; the per-plan column and the marketplace row are left intact so a reinstall keeps existing plan grants.

---

## 4. How measurement works (concepts)

Understanding the flow prevents the most common confusion.

1. **You generate creatives** in MagicAds (Image Studio, Video Studio, etc.). They live in your creative library.
2. **A user runs a creative as a paid ad** on Meta, Google or TikTok — this happens **in those platforms' ad managers**, not in MagicAds. MagicAds does not launch paid campaigns for you.
3. **The plugin syncs** the ad's daily metrics from the network's reporting API into `ad_metrics`.
4. **Attribution links** each external ad back to the MagicAds creative that produced it, so the Creatives leaderboard can show ROAS per generated creative.

Attribution happens two ways:

- **Manual linking** (always works) — on the Creatives screen a user links a spending ad to the creative it came from, using a visual picker.
- **Tag matching** (optional) — if the user names their ad, or sets the landing-page `utm_content`, to include a `magicads_<creativeId>` token, the sync auto-links it. The creative id is the number shown on each creative.

<Note>
The ad network never sees the creative's filename, so attribution can only match on the **ad name** or **utm_content** — both set by the user inside the ad manager. Manual linking is the reliable default; the tag is a convenience for users who name ads consistently.
</Note>

---

## 5. Configure the AI insights key

The dashboards, sync and attribution need **no** AI key. Only the **AI Insights** screen calls OpenAI, and it reuses the platform's central, encrypted key store.

1. Go to **Admin → AI Settings** (`/app/admin/ai`).
2. Add your **OpenAI** key (fills `openai_key`). The account needs access to a chat model (default `gpt-4o-mini`) and a funded balance.
3. Save.

If no OpenAI key is present, the AI Insights screen shows a notice and the **Generate** button is disabled; everything else in the plugin still works.

---

## 6. Register the ad-network apps (admin)

This is the one-time setup that lets your users connect. You'll create a developer app on each network, whitelist the plugin's callback URL, and paste the resulting credentials into the plugin config.

First, get your exact **callback URLs**. Open **Admin → General Settings → Plugins → Ad Performance Analytics** — each provider block shows a read-only "Redirect / Callback URL" field. They follow this pattern (replace the domain with your site):

```
https://YOUR-DOMAIN/app/ad-analytics/callback/meta
https://YOUR-DOMAIN/app/ad-analytics/callback/google
https://YOUR-DOMAIN/app/ad-analytics/callback/tiktok
```

<Warning>
Copy the callback URLs from the config screen verbatim. They must match what you register on each network **exactly** (including https and no trailing slash), or the OAuth round-trip will fail the security check.
</Warning>

### 6a. Meta Ads (Facebook / Instagram)

<Steps>
<Step title="Create a Meta app">
Go to [developers.facebook.com](https://developers.facebook.com) → **My Apps** → **Create App**. Choose the **Business** app type and give it a name.
</Step>
<Step title="Add the Marketing API">
In the app dashboard, add the **Marketing API** product. This is what exposes ad-account insights.
</Step>
<Step title="Add Facebook Login for OAuth">
Add the **Facebook Login** product. Under its **Settings**, set **Valid OAuth Redirect URIs** to your Meta callback URL from above.
</Step>
<Step title="Copy the credentials">
Open **App Settings → Basic**. Copy the **App ID** and **App Secret**.
</Step>
<Step title="Request permissions & verify the business">
Under **App Review → Permissions and Features**, request **ads_read** (and **business_management**). Advanced access requires **Business Verification** — complete it in **Business Settings**. Until approved, the app can only read ad accounts owned by the app's own developers/testers.
</Step>
<Step title="Save into MagicAds">
In the plugin config, open the **Meta Ads** block, paste the **App ID** and **App Secret**, toggle **Meta** on, and **Save**.
</Step>
</Steps>

### 6b. Google Ads

Google is the most involved because it needs both an OAuth client **and** a developer token.

<Steps>
<Step title="Create a Google Cloud project & OAuth consent screen">
In the [Google Cloud Console](https://console.cloud.google.com), create a project. Open **APIs & Services → OAuth consent screen**, choose **External**, fill the app details, and add the scope `https://www.googleapis.com/auth/adwords`. Add yourself as a test user (or publish the consent screen).
</Step>
<Step title="Create an OAuth client ID">
Go to **APIs & Services → Credentials → Create Credentials → OAuth client ID**. Choose **Web application**. Under **Authorized redirect URIs**, add your Google callback URL from above. Copy the **Client ID** and **Client Secret**.
</Step>
<Step title="Get a Google Ads developer token">
Sign in to a **Google Ads Manager (MCC)** account → **Tools & Settings → API Center**. Copy the **Developer token**. New tokens start with **test-account-only** access; apply for **Basic/Standard** access so it can read live accounts (approval can take a few days).
</Step>
<Step title="Note your Manager (MCC) ID (optional)">
If your accounts sit under a Manager account, copy its customer ID (the `123-456-7890` number). This becomes the login-customer-id.
</Step>
<Step title="Save into MagicAds">
In the plugin config → **Google Ads** block, paste the **OAuth Client ID**, **OAuth Client Secret**, **Developer Token**, and optionally the **Manager (MCC) Customer ID**. Toggle **Google Ads** on, and **Save**.
</Step>
</Steps>

<Note>
Google's developer-token approval is the single biggest onboarding hurdle. Until it reaches Basic/Standard access, connections only work against test accounts. Plan for this lead time.
</Note>

### 6c. TikTok Ads

<Steps>
<Step title="Create a TikTok for Business developer app">
Go to the [TikTok for Business developer portal](https://business-api.tiktok.com/portal/docs) → **My Apps** → create a new app. Fill in the app details.
</Step>
<Step title="Set the redirect URI & scope">
Set the app's **Redirect URI** to your TikTok callback URL from above. Request the **Reporting** (Ads reporting) scope.
</Step>
<Step title="Submit for approval">
Submit the app for review. TikTok must approve the app before advertisers outside your own org can authorize it.
</Step>
<Step title="Copy the credentials">
From the app page, copy the **App ID** and **App Secret**.
</Step>
<Step title="Save into MagicAds">
In the plugin config → **TikTok Ads** block, paste the **App ID** and **App Secret**, toggle **TikTok** on, and **Save**.
</Step>
</Steps>

<Tip>
You don't have to configure all three networks. Each provider block has its own enable switch and independent readiness — set up only the networks you want, and the rest simply won't appear for users. Ship with one and add the others later.
</Tip>

---

## 7. Configure the plugin

Go to **Admin → General Settings → Plugins → Ad Performance Analytics** (`/app/admin/general/plugins/ad-analytics`). The screen has these sections.

### General

| Setting | Purpose |
|---------|---------|
| **Enable Ad Performance Analytics** | Master switch. When on, the plugin appears in user menus. Off by default. |
| **Free Tier Access** | When on, users **without** a paid plan can use the plugin. When off, only users on a plan that includes the feature get access. |
| **AI insight rate (credits per report)** | Flat credit cost for generating one AI analysis. Integer 1–999, default **3**. Connecting, syncing and dashboards are always free. |

### Per-provider credentials

One block each for **Meta Ads**, **Google Ads** and **TikTok Ads**, each with an enable switch, its credential fields (from Section 6), a setup hint, a docs link, and the read-only callback URL to whitelist.

Click **Save** to persist all sections.

---

## 8. Grant access by plan

Access is gated in two layers.

1. **Platform layer** — the master **Enable Ad Performance Analytics** switch must be on.
2. **User layer** — who actually gets it:
   - **Subscribed users** (on a plan): governed by the plan's **Ad Performance Analytics** feature toggle. Edit each plan and enable it where it should be included.
   - **Non-subscribed users** (no plan): access only when both **Enable Ad Performance Analytics** and **Free Tier Access** are on.

When the plugin is offered platform-wide but the current user's plan doesn't include it, it appears as a **locked, upgrade-to-unlock** entry that nudges the user toward an eligible plan.

To set it up:

1. Enable the master switch in the plugin config.
2. Go to **Admin → Pricing Plans**, edit each plan, and turn the **Ad Performance Analytics** feature on for the plans that should include it.
3. Optionally enable **Free Tier Access** for plan-less users.

---

## 9. Enable the sync cron

Syncing is cron-driven, so it runs on shared hosting and VPS alike. The host only needs Laravel's single scheduler entry:

```
* * * * * cd /path-to-project && php artisan schedule:run >> /dev/null 2>&1
```

With that in place, the plugin:

- **Hourly** — queues a metrics sync for every active ad account whose throttle window has passed.
- **Daily (03:30)** — prunes normalized rows older than the retention window.

Users can also trigger a sync manually with the **Sync now** button (top-right of every analytics page) or per-account from **Connections**. Syncing is idempotent — a missed run self-heals on the next tick.

---

## 10. Go-live checklist

<Steps>
<Step title="Install the plugin">
Admin → General Settings → Plugins → Ad Performance Analytics → **Install**.
</Step>
<Step title="Register at least one network app">
Follow Section 6 for Meta, Google and/or TikTok. Whitelist the exact callback URLs.
</Step>
<Step title="Save the credentials">
Paste each network's credentials into the plugin config and enable that network.
</Step>
<Step title="Add an OpenAI key (optional)">
Admin → AI Settings → add an **OpenAI** key to enable the AI Insights screen.
</Step>
<Step title="Set pricing & enable the feature">
Set the AI insight rate, turn on **Enable Ad Performance Analytics**, and enable **Free Tier Access** if desired.
</Step>
<Step title="Grant plan access">
Enable the feature on the plans that should include it.
</Step>
<Step title="Confirm the cron">
Make sure `php artisan schedule:run` runs every minute on the host.
</Step>
<Step title="Verify as a user">
Log in as a user on an eligible plan, connect an ad account, run **Sync now**, and confirm data appears.
</Step>
</Steps>

---

## 11. What your users do (and what they configure on their own ad platforms)

This is the part to communicate clearly to your customers.

**Users never enter API keys.** The developer apps you registered in Section 6 are the "application"; your users only need to authorize their own ad accounts through them.

<Steps>
<Step title="Connect an account">
The user opens **Ad Analytics → Connections**, clicks **Connect** on a network, logs into their Meta / Google / TikTok account on the provider's own consent screen, and approves read access to their ad accounts. Their tokens are captured and stored encrypted.
</Step>
<Step title="Sync">
Metrics import automatically on the cron, or immediately via **Sync now**.
</Step>
<Step title="Link creatives (for creative-level ROAS)">
On the **Creatives** screen, the user links spending ads to the MagicAds creatives that produced them — either by naming their ad / setting `utm_content` to include the `magicads_<id>` code (auto-links on sync), or with the manual visual picker.
</Step>
</Steps>

What users must have on their **own** ad platforms:

- An ad account on the network with **admin or analyst access** (so the OAuth consent can grant read access).
- Live or historical campaigns — the plugin reads what exists; it does not create campaigns.
- For **TikTok revenue/ROAS**: revenue is opt-in because the metric name depends on the advertiser's attribution setup. Spend, clicks, impressions and conversions always import; to enable TikTok ROAS, set `ad-analytics.providers.tiktok.value_metric` in the config file to the revenue metric your advertiser reports support.

<Note>
Running a creative as a **paid ad** is always done by the user inside Meta / Google / TikTok Ads Manager. MagicAds generates the creative and measures the result — it does not launch or manage paid campaigns.
</Note>

---

## 12. Using the screens

All screens live under **Analytics → Ad Analytics** in the user sidebar and share a header with a date-range control (7d / 30d / 90d / MTD), a network filter, a live KPI strip and a **Sync now** button.

### Overview

The cockpit. Headline KPI cards (Spend, ROAS, Revenue, Conversions, CTR, CPA) each show a period-over-period delta. Below them: a **Spend vs. Revenue** area chart whose tooltip also shows that day's ROAS; a **By network** breakdown with brand-colored bars, ROAS and budget share; **Budget pacing** (month-to-date spend and a run-rate projection); and **Fatigue alerts** flagging ads whose CTR is sliding week-over-week (with the network shown).

### Campaigns

Every campaign across all networks as a card: network icon, color-coded ROAS pill, a five-metric grid (Spend, Revenue, Conversions, CTR, CPA) and a budget-share bar. Sort by any metric; filter by network and date range.

### Creatives

The signature screen — creative-level ROAS. An **attribution coverage** meter shows how much spend is linked. The **leaderboard** spotlights your top performer and grids the rest, each card showing the creative thumbnail, color-coded ROAS, key metrics and the network(s) it ran on. Below, **Link ads to creatives** lists spending ads that aren't linked yet; the visual picker lets you match each to a generated creative (with thumbnails and video previews). Existing links show as removable chips.

### AI Insights

Generate an on-demand analysis for the current range and network filter. Each report includes a headline verdict, an overall health badge, "what's working" and "watch outs" lists, and prioritized recommendations tagged by category (budget, creative, targeting, bidding, attribution, scaling, testing), impact, effort, the target entity and the expected effect. Reports are cached in history so a paid generation is never repeated. Generating costs the configured credit rate and is charged only on success.

### Connections

Add accounts per network (one-click OAuth), then pause, sync or disconnect each. Cards show status, currency, last-synced time and any sync error.

---

## 13. Billing

- The **only** billable action is generating an **AI insight** — charged the flat rate set in the config (default 3 credits).
- Users are **only charged on success** — a failed AI call costs nothing.
- Credits are deducted atomically, drawing from `credits` then `credits_prepaid`, so concurrent actions can't overdraw.
- Connecting accounts, syncing, and every dashboard are always free.

---

## 14. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Plugin doesn't appear for a user | Master switch off, plan lacks the feature, or free tier off for a plan-less user | Enable the master switch; enable the feature on their plan; or turn on Free Tier Access. |
| "Security verification failed" on connect | Callback URL mismatch | Ensure the redirect URI registered on the network matches the config screen's URL exactly (https, no trailing slash). |
| ":network is not configured yet" | Provider disabled or credentials missing | Enable the provider block and paste its credentials in the plugin config. |
| Meta connect returns no ad accounts | `ads_read` not granted / business not verified | Complete App Review + Business Verification, or test with an account owned by the app's developers. |
| Google connect fails or returns nothing | Developer token still test-only, or consent scope missing | Get Basic/Standard access for the developer token; confirm the `adwords` scope on the consent screen. |
| TikTok revenue / ROAS shows 0 | No revenue metric configured | Set `ad-analytics.providers.tiktok.value_metric` to your advertiser's revenue metric. |
| No data after connecting | Sync hasn't run | Click **Sync now**, or confirm the `schedule:run` cron is active. |
| Creatives leaderboard empty | No ads linked to creatives | Link ads on the Creatives screen, or add the `magicads_<id>` tag to ad names / `utm_content`. |
| "AI insights are not configured" | No OpenAI key | Add an OpenAI key in Admin → AI Settings. |
| "You do not have enough credits" | Balance below the insight rate | Top up credits or lower the AI insight rate. |
| Token expired / reconnect prompt | OAuth token lapsed (esp. Meta long-lived, Google refresh) | Reconnect the account from Connections. |

<Note>
Metric data is stored in `ad_metrics` and pruned to the configured retention window. Ad-account tokens are encrypted at rest with your app `APP_KEY`. Credits are never deducted for failed AI generations.
</Note>
