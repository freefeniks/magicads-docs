# Berkine changelog format

The authoring spec for release entries on the Berkine changelog page.

Take this file into any other repository, hand it to whoever (or whatever) is writing up the
releases there, and what comes back should paste straight into `config/changelog.php` with no
reformatting.

---

## 1. Where the data lives

| Thing | File |
| --- | --- |
| The entries | `config/changelog.php` → `releases` array |
| The rendering | `resources/views/changelog.blade.php` |
| The route | `routes/web.php` → `Route::view('changelog', 'changelog')->name('changelog')` |

There is no controller and no database table. A release is an array entry, nothing more. The view
reads `config('changelog.releases')` at render time, so a new entry is live as soon as the config
cache is cleared.

**Never edit the Blade file to add a release.** The only thing that changes on ship day is the config
array.

---

## 2. Entry shape

One array per release. Eight keys, always in this order:

```php
[
    'date' => '2026-08-21',
    'product' => 'Berkine platform',
    'kind' => 'platform',
    'version' => '2026.8',
    'tag' => 'Breaking',
    'title' => 'Crypto payments move to Helio',
    'summary' => 'One paragraph explaining why it was worth shipping.',
    'changes' => [
        'added' => ['…'],
        'improved' => ['…'],
        'breaking' => ['…'],
    ],
],
```

### `date` — required

ISO `YYYY-MM-DD`. Parsed with `Carbon::parse()`.

This is **the only ordering key**. Everything else about position is derived from it, so an entry
dropped in the wrong place in the file still lands in the right year and the right slot.

It must be the real ship date. Three figures in the page masthead are computed from these dates
(last shipped, releases in the last twelve months, products covered), and the page is written so it
cannot claim a number the list below it disagrees with. Fudging a date moves a headline figure.

### `product` — required

The listing the release belongs to, spelled **exactly** as the product record spells it.

The view resolves product names against published products (`App\Support\Catalog::cards()` for
Scripts and Plugins). A name that resolves gets linked to its product page; a name that does not is
set as plain text instead of a dead link. So a typo does not break the page, it just silently drops
the link.

The platform itself is always `'Berkine platform'` — it is not a listing, so it never links.

### `kind` — required

Exactly one of:

| Value | Renders as | Means |
| --- | --- | --- |
| `script` | Script | A standalone application |
| `plugin` | Plugin | An add-on |
| `platform` | Platform | This site: checkout, licences, the buyer area |

Anything else renders as the raw string you typed, which will look wrong. Use these three.

### `version` — required

As tagged, no `v` prefix — the view adds it. Two conventions in use:

- Products: dotted numeric, `3.4`, `2.1`, `1.9`, `2.0`
- Platform: calendar, `2026.8`, `2026.6`, `2025.9`

`product` + `version` together form the entry's permalink anchor, via
`Str::slug($product.' '.$version)` → `#magicdesk-3-4`, `#berkine-platform-2026-8`. **The pair must be
unique across the whole log** or two entries share an anchor and support links point at the wrong
release.

### `tag` — required, may be null

The only badge on the page. Three permitted values:

- `'Breaking'`
- `'Security'`
- `null`

Always write the key, even when it is `null`. Do not omit it.

The tag is not decorative, it is derived:

- `changes.breaking` present → `'Breaking'`
- `changes.security` present and no breaking changes → `'Security'`
- otherwise → `null`

Most releases are `null`. A badge on an ordinary release makes the badge meaningless.

### `title` — required

What the release did, in one line. Renders after the product name as a single sentence:
**MagicDesk** *Model policies, set per workspace*.

- Sentence case, no trailing full stop.
- Six to nine words. It has to survive being read next to the product name.
- Describe the outcome, not the work. "Durable runs, resumable from any step", not "Refactored the
  run engine".

### `summary` — required

One paragraph. Why it was worth shipping.

- No bullet points, no line breaks, no headings. One string.
- Two to four sentences, roughly 40–70 words.
- Full stop at the end.
- The strongest pattern in the existing log: **name the problem first, then what now happens.**
  "The old screen matched patterns, which meant it caught the phrasings we had seen and missed the
  ones we had not. It now classifies intent before a prompt reaches a tool-using agent…"
- Say what was wrong plainly. The page advertises that it includes the things that broke; a summary
  that only sells is off-voice.

### `changes` — required

An array keyed by kind of change. **Five permitted keys and no others:**

```
added  improved  fixed  security  breaking
```

Any other key is silently ignored by the view — it will not appear on the page and nothing will
warn you.

Include only the groups that apply. Two or three groups per entry is typical; five is almost never
honest.

Each group is a flat list of strings.

---

## 3. Ordering rules

### Between entries

Newest first, by `date`. The file follows that convention for readability, but the view sorts
regardless, then groups by year and sorts the years descending. Each year becomes an anchored
section and a chapter-nav item.

### Between change groups

Fixed by the view, not by your array:

1. **Breaking**
2. **Security**
3. **Added**
4. **Improved**
5. **Fixed**

A breaking change or a security fix has to be the first thing seen in an entry, whichever key the
author happened to type first. Write them in this order anyway so the source reads like the page.

Breaking and Security also render in the accent colour; the other three are muted.

### Within a group

Preserved exactly as authored. Put the item with the largest consequence first — the thing a reader
has to act on before the thing that is merely nice.

---

## 4. Writing the bullets

Volume, per the existing log:

| Group | Typical count |
| --- | --- |
| `breaking` | 1–2 |
| `security` | 2–3 |
| `added` | 2–3 |
| `improved` | 1–2 |
| `fixed` | 1–2 |

Rules:

- One sentence per bullet, ending in a full stop. Occasionally two short ones.
- 8–25 words. Long enough to be specific, short enough to scan.
- No leading dash, bullet character or label — the view draws the mark.
- Start with the thing, not with "We". "Hard caps per tenant, per seat and per feature, enforced at
  dequeue." Not "We added hard caps."
- Name the mechanism, not the adjective. "enforced before the request leaves the queue" beats
  "reliable enforcement".
- Numbers where you have them: "from roughly 300ms to under 40ms", "a 400-case set from about 12
  minutes to under 3", "around 60 per cent". Hedge the number ("roughly", "about") rather than
  inflate it.
- `fixed` bullets describe **the bug**, in the past tense, not the repair: "Documents deleted at
  source stayed searchable until the next full reindex." The fix is implied by it being in the log.
- `breaking` bullets must state the action required, and point at the upgrade guide or migration
  command if there is one: "The webhook endpoint for crypto is now /payments/helio/webhook.
  Subscribe it to REGULAR_TRANSACTION in the Helio dashboard."
- `security` bullets describe the coverage gained, never the exploit path.

---

## 5. House style

- **British English.** licence, behaviour, prioritised, sanitises, diarisation.
- **"per cent"**, two words, spelled out. Not "%".
- Units closed up: `40ms`, `200ms`, `4x`.
- Numbers under ten spelled out in prose (`twelve months`, `three gateways`); figures where they are
  measurements or versions.
- No exclamation marks, no em dashes as decoration, no marketing superlatives. Show the fact.
- Product and feature names as they are actually cased: `Livewire 4`, `Laravel 13`, `PHP 8.4`,
  `Stripe`, `Helio`.
- Code-ish identifiers stay bare, no backticks — the view renders plain text, so backticks would
  print literally. Table and column names are fine as `agent_runs` written plainly:
  `agent_runs and agent_steps`.

### PHP string escaping

All strings are single-quoted PHP. Two things to watch:

- ASCII apostrophe must be escaped: `'the caller\'s own permissions'`
- A typographic apostrophe needs no escape: `'the storefront’s currency'`

Both appear in the existing file. Either is acceptable; be consistent within an entry.

None of these strings pass through `__()`. They render exactly as written, in English, on every
locale. Only the labels around them are translated.

---

## 6. Formatting

Inside `config/changelog.php`:

- Entry array at **8 spaces**
- Keys at **12 spaces**
- Group keys at **16 spaces**
- Bullets at **20 spaces**
- Trailing comma on every element, including the last
- One blank line between entries

---

## 7. Template

```php
        [
            'date' => 'YYYY-MM-DD',
            'product' => 'Exact Product Name',
            'kind' => 'script',
            'version' => '1.0',
            'tag' => null,
            'title' => 'Outcome in six to nine words',
            'summary' => 'The problem as it stood, in one sentence. What now happens instead, in one or two more. No bullets, no line breaks.',
            'changes' => [
                'breaking' => [
                    'What changed, and the action the reader has to take.',
                ],
                'security' => [
                    'The coverage gained, not the exploit.',
                ],
                'added' => [
                    'The thing, stated as a thing, with the mechanism named.',
                    'A second thing, if there genuinely is one.',
                ],
                'improved' => [
                    'What got better, with a measured number where one exists.',
                ],
                'fixed' => [
                    'The bug as it behaved, in the past tense.',
                ],
            ],
        ],
```

Delete the groups that do not apply. Do not leave empty arrays — the view skips them, but they are
noise in the source.

---

## 8. Worked example

```php
        [
            'date' => '2026-07-09',
            'product' => 'Guardrails',
            'kind' => 'plugin',
            'version' => '2.1',
            'tag' => 'Security',
            'title' => 'Injection screening rewritten',
            'summary' => 'The old screen matched patterns, which meant it caught the phrasings we had seen and missed the ones we had not. It now classifies intent before a prompt reaches a tool-using agent, and refuses rather than sanitises when it is not sure.',
            'changes' => [
                'security' => [
                    'Instruction-override attempts embedded in retrieved documents are now screened, not just user input.',
                    'Tool calls are checked against the caller\'s own permissions rather than the agent\'s.',
                    'PII redaction covers structured payloads, so an address inside a JSON tool result is masked like one in prose.',
                ],
                'improved' => [
                    'Screening runs in parallel with retrieval, which took the added latency from roughly 300ms to under 40ms.',
                ],
                'fixed' => [
                    'A moderation failure was logged and swallowed instead of blocking the response. It now fails closed.',
                ],
            ],
        ],
```

---

## 9. Checklist before committing

- [ ] `date` is ISO, real, and the entry sits newest-first in the file
- [ ] `product` matches the published product name character for character
- [ ] `kind` is one of `script`, `plugin`, `platform`
- [ ] `product` + `version` is unique across the whole log
- [ ] `tag` key is present, and agrees with the presence of `breaking` / `security`
- [ ] `title` has no trailing full stop
- [ ] `summary` is one paragraph with no bullets and ends in a full stop
- [ ] `changes` uses only the five permitted keys
- [ ] Every bullet is a full sentence ending in a full stop, with no leading dash
- [ ] Every `breaking` bullet names the action required
- [ ] ASCII apostrophes escaped as `\'`
- [ ] Indentation and trailing commas match section 6
- [ ] `php artisan config:clear` after deploying, or the new entry stays invisible
