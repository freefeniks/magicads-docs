---
title: "Teams"
description: "Install, configure and use the MagicAds Teams extension — let a user build a team, invite members, share credits, collaborate on projects, and track everything in one activity feed."
---

# Teams

Teams is a **free** extension (plugin slug: `magicads-team`) for the MagicAds platform. It turns a single account into a small workspace: one person becomes a **team owner**, invites others as **members**, funds them with credits, shares projects with them, and watches all of it in a live activity feed.

It's built for agencies, marketing squads and small businesses where one person holds the budget (credits and plan) and everyone else creates under that umbrella. Members keep their own login and their own credit balance — the owner just tops them up and shares work with them.

<Check>
**Teams is a free plugin.** There's nothing to purchase — install it from the plugin marketplace and it's ready to configure. You still control *who* can use it through the master switch, plans and the free-tier toggle (covered below).
</Check>

This guide covers everything: how it's structured, how to install and configure it, how access and seat limits are decided, and how the team panel works day to day.

---

## 1. What you get

Teams adds one user-facing panel plus an admin configuration screen.

| Piece | Where | What it does |
|-------|-------|--------------|
| **Team panel** | Workspace menu → **Teams** (`/app/user/team`) | Create or manage a team, invite members, transfer credits, share projects, and browse the activity feed. |
| **Admin config** | General Settings → Plugins → **Teams** (`/app/admin/general/plugins/team`) | Turn the feature on, allow free-tier access, and set the free-tier seat limit. |

Inside the team panel, an owner can:

- **Build a team** — create it, rename it, or disband it.
- **Invite members** by email (works even if the person hasn't registered yet).
- **Share credits** — move credits from the owner's balance into a member's balance.
- **Share projects** — give a member viewer or editor access to one of the owner's projects.
- **Track activity** — a filterable feed of every credit use, transfer, share and membership change, with CSV export.

A member can accept or decline invitations, use the credits and projects shared with them, and leave the team.

---

## 2. Core concepts

A few rules shape everything else. Understanding these first makes the rest obvious.

<Note>
**One team per user.** Every user can belong to exactly one team at a time — either as its owner or as a member. To join another team, you must first leave (or disband) your current one.
</Note>

- **Owner vs member.** The person who creates a team is its **owner** (there's one owner per team). Everyone they add is a **member**. The owner is also counted as a membership row internally, but owners don't consume a member seat.
- **Seats count members only.** A team's seat limit is the number of **members** the owner can add — the owner themselves is never counted against it.
- **Pending invitations hold a seat.** An outstanding invitation counts toward the limit, so an owner can't over-invite and end up with more members than seats.
- **Credits are per user.** Each account has its own credit balance. Teams doesn't create a shared pool — instead the **owner transfers credits into a member's own balance**, and the member spends from there like normal.
- **Sharing widens visibility, not ownership.** When an owner shares a project, the member can see (and optionally edit) it, but the project still belongs to the owner and counts against the owner's project limits — never the member's.

---

## 3. Install / activate

Install it from the marketplace card:

1. Sign in as an **admin**.
2. Go to **Admin → General Settings → Plugins**.
3. Find the **Teams** card and click **Install**.

Behind the scenes the platform downloads the plugin archive, unpacks it, runs its migration and flips the extension's `installed` flag. This creates the feature's tables:

- `teams` — one row per team (owner + name); a user owns at most one.
- `team_members` — membership rows (`owner` / `member`); a user belongs to at most one team.
- `team_invitations` — email invitations with a token, status and 14-day expiry.
- `team_activities` — the activity feed (credit use, transfers, shares, membership changes).
- `project_user` — the sharing pivot linking a shared project to a member at `viewer` / `editor` access.

It also adds the plugin's settings to the shared `extension_settings` row (`team_feature`, `team_free_tier`) and a `free_tier_team_members` limit to `general_settings`.

<Warning>
Installation only makes the routes and tables exist. The Teams panel stays hidden from users until you **enable the feature** and grant access (next sections). On a fresh install everything is off by default.
</Warning>

To remove the plugin later, click **Uninstall** on the same card.

---

## 4. Configure Teams

Go to **Admin → General Settings → Plugins → Teams** (`/app/admin/general/plugins/team`). The screen is short.

| Setting | Purpose |
|---------|---------|
| **Enable Teams** | Master switch. When on, the Teams panel appears in the Workspace menu and eligible users can build a team. Off by default. |
| **Free Tier Access** | Lets users **without** a paid plan create a team, using the free-tier seat limit below. When off, only subscribers get Teams. |
| **Free-tier seats per team** | Maximum **members** (excluding the owner) a free-tier owner can invite. `0` blocks team creation for free-tier users. Range 0–1000. |

Click **Save** to persist all three.

<Note>
Per-plan seat limits are **not** set here. Each plan carries its own **Team Members** value — edit it under **Finance → Plans**. That value is what caps subscribers.
</Note>

---

## 5. Who can use Teams — access & seat limits

Access is decided in two layers.

**Layer 1 — platform.** The plugin must be installed and **Enable Teams** must be on. If not, Teams is hidden for everyone.

**Layer 2 — the user.** Given the platform layer is on:

- **Subscribers (on a plan):** they get Teams only when their plan's **Team Members** value is greater than zero. That number is also their seat limit.
- **Free-tier users (no plan):** they get Teams only when **Free Tier Access** is on *and* **Free-tier seats per team** is greater than zero. That number is their seat limit.

When Teams is offered platform-wide but the current user isn't eligible, it can appear as a **locked, upgrade-to-unlock** entry that nudges them toward a plan rather than hiding entirely.

### How the seat limit resolves

The effective limit for a team owner is:

| Owner's situation | Seat limit used |
|-------------------|-----------------|
| Active subscription with a plan `Team Members` value | That plan value |
| Active subscription but no plan value set | Falls back to the free-tier seat setting |
| No active subscription | The free-tier seat setting (`Free-tier seats per team`) |
| Unset or negative | `0` — no new invites (existing members stay) |

Remember: the limit counts **member seats only** and **pending invitations occupy a seat** until they're accepted, declined, revoked or expired.

---

## 6. Using the team panel

Everything lives under **Workspace → Teams**.

### Create a team

If you're not in a team yet, enter a name (2–120 characters) and create it. You become the owner and your own membership row is created automatically. You can rename the team any time, or disband it.

### Invite members

As the owner, invite people by **email address**:

- The invite works whether or not the email belongs to a registered user. Registered users get an in-app notification; everyone else gets an email.
- Invitations are valid for **14 days**, then expire.
- The panel blocks invites that would exceed your seat limit, duplicate a pending invite, target someone already on your team, or target a user who already belongs to another team.
- You can **revoke** a pending invitation, which frees the seat again.

### Accept or decline an invitation

When you have a pending invitation addressed to your email (and you're not already in a team), it shows at the top of your Teams panel. **Accept** to join — provided your email matches, you're not already in a team, and a seat is free. **Decline** to dismiss it.

<Note>
Because of the one-team rule, you can't accept an invitation while you're in another team. Leave your current team first.
</Note>

### Share credits with a member

Open a member's **transfer** action and enter an amount. The credits move **from your (the owner's) balance into that member's balance** in a single, guarded transaction — you can never transfer more than you hold. The member is notified, and the transfer is logged in the activity feed. The member then spends those credits normally on any studio.

### Share a project with a member

Open a member's **share** action, pick one of **your** projects, and choose an access level:

- **Viewer** — the member can see the project and its work.
- **Editor** — the member can also make changes.

Sharing only widens visibility; you keep ownership, and the project keeps counting against **your** project limits, not the member's. Stop sharing at any time. Removing a member, or a member leaving, automatically revokes the projects you'd shared with them.

### Watch the activity feed

The feed records the team's actions: credit usage, credit transfers, project shares/unshares, members joining/leaving/being removed, and invitations sent/revoked. Filter it by **type** or by **member**, and **export to CSV** (up to 5,000 rows) for reporting.

<Note>
Whenever a member spends credits on a studio, that consumption is recorded against the team automatically — so the owner always has visibility into who used what.
</Note>

### Leave or disband

- **Members** can **leave** a team at any time. Their shared projects are revoked on the way out.
- **Owners can't leave** their own team — instead they **disband** it. Disbanding removes all memberships, pending invitations, activity and project shares for that team. Members keep their own accounts and their own credit balances.

---

## 7. Go-live checklist

<Steps>
<Step title="Install the plugin">
Admin → General Settings → Plugins → Teams → **Install**.
</Step>
<Step title="Set plan seat limits">
Finance → Plans → set **Team Members** on each plan that should include Teams.
</Step>
<Step title="Configure free-tier access (optional)">
Teams config → turn on **Free Tier Access** and set **Free-tier seats per team** if you want plan-less users to build teams.
</Step>
<Step title="Enable the feature">
Turn on **Enable Teams** and **Save**.
</Step>
<Step title="Verify as an owner">
Log in as an eligible user → Workspace → **Teams** → create a team, invite a member, transfer a few credits and share a project.
</Step>
<Step title="Verify as a member">
From a second account, accept the invitation and confirm the shared credits and project appear.
</Step>
</Steps>

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Teams doesn't appear in the menu | Master switch off, plan has 0 seats, or free-tier off/zero for a plan-less user | Enable **Enable Teams**; set **Team Members** on the plan; or turn on Free Tier Access with a seat limit > 0. |
| "You already belong to a team" | The one-team-per-user rule | Leave (member) or disband (owner) the current team first. |
| "You have reached your team seat limit" | Member seats + pending invites fill the limit | Raise the plan's Team Members value (or the free-tier limit), or revoke a pending invite. |
| Invitation can't be accepted | Email mismatch, invite expired, or the invitee is already in a team | Invite the exact email; re-send if past 14 days; the invitee must leave their other team. |
| "You do not have enough credits to transfer" | Owner's balance below the amount | Top up the owner's credits, then retry the transfer. |
| Member can't see a shared project | Not shared with them, or share was revoked | Re-share the project from the member's row at viewer/editor access. |
| Owner can't leave the team | Owners disband, they don't leave | Use **Disband** instead — or transfer ownership by rebuilding the team. |
| A removed member still had my project | Shares are revoked on removal/leave | This is automatic; if it persists, re-open and unshare explicitly. |

<Note>
Teams never touches how anything is generated or priced — it only governs membership, credit transfers between accounts, and project visibility. Disbanding a team or uninstalling the plugin never deletes members' accounts or their credit balances.
</Note>
