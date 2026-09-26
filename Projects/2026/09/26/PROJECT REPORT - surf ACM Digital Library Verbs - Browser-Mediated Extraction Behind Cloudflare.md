---
title: "surf ACM Digital Library Verbs — Browser-Mediated Extraction Behind Cloudflare"
aliases: [surf acm, ACM DL verbs, SURF-20260926-ACMDL]
tags: [project-report, surf-cli, browser-automation, dom-scraping, cloudflare, glazed, go]
status: active
type: project-report
created: 2026-09-26
updated: 2026-09-26
audience: engineer adding a new site to surf-go, or auditing how the acm verbs work
repo: /Users/manuel.odendahl/code/wesen/surf-cli
---

# surf ACM Digital Library Verbs — Browser-Mediated Extraction Behind Cloudflare

## 0. What this document is

This is the project report for ticket `SURF-20260926-ACMDL` in the
surf-cli repository: adding ACM Digital Library (dl.acm.org) support to
the `surf` CLI as four new commands — `acm search`, `acm paper`,
`acm download` and `acm cite`. It explains the architecture the verbs run
on, the contracts the site actually exposes (each one verified live), the
two findings that determined the entire design, the implementation of each
verb, the test strategy, and the failure modes that were hit during the
work. It is written so that an engineer who has never touched surf-go can
understand how the system works from this document alone, and can use it
as the reference for adding the next site.

All evidence lives in the ticket workspace:

```text
/Users/manuel.odendahl/code/wesen/surf-cli/ttmp/2026/09/26/SURF-20260926-ACMDL--add-acm-digital-library-support-search-paper-detail-pdf-download/
├── reference/01-verified-acm-digital-library-page-contracts.md   every contract + its probe
├── reference/02-diary.md                                          chronological build log, failures included
├── design-doc/01-…-intern-guide.md                               the full design written before the code
└── scripts/01…07-*.js                                            the replayable probes
```

## 1. The problem

The ACM Digital Library is the publication platform of the ACM: roughly
850,000 full-text records (proceedings, journals, magazines) plus a
bibliographic index of about 4 million works. A CLI that can search it,
read a paper's metadata, references and citations, download PDFs, and
export citations turns the literature review part of research work into
composable shell commands.

Three properties of the site shape every design decision:

1. **There is no API.** dl.acm.org is server-rendered Silverchair. A
   network capture of a search page load contains exactly one same-origin
   content request — the HTML document itself — plus analytics, fonts,
   and a survey widget. Everything a verb needs is in the rendered DOM.
   This classifies the work as a *DOM-scraping browser-side verb* in the
   surf-go taxonomy, not an *API-first verb*.
2. **Cloudflare sits in front of the site.** Any client that is not a real
   browser receives `403` with `cf-mitigated: challenge`. This is
   verified, not assumed, and it rules out the simplest download design.
3. **The site is the only PDF renderer worth using.** The citation export
   endpoint returns structured CSL data that the site's own JavaScript
   renders into BibTeX/EndNote text — the server never produces the
   citation string.

The consequences: the verbs must read the DOM inside a real browser tab,
must move PDF bytes through that tab, and must drive the site's own
citation widget instead of calling its endpoints directly.

## 2. The execution substrate

`surf` does not embed a browser. It is a layered client over a Chrome
instance the user already runs, attached through the surf native host:

```text
┌─────────────────────────── terminal ────────────────────────────────┐
│  surf acm search "gnn"            cobra + glazed CLI command        │
└────────────────────────────────────┬────────────────────────────────┘
                                     │ typed settings → fetch helper
┌────────────────────────────────────▼────────────────────────────────┐
│  go/internal/cli/commands/acm_search.go                             │
│    · builds the target URL from the verified grammar                │
│    · embeds the page script (go:embed scripts/acm_search.js)        │
│    · passes options via a SURF_OPTIONS prelude                      │
│    · owns the tab strategy (create / reuse / close)                  │
└────────────────────────────────────┬────────────────────────────────┘
                                     │ ExecuteTool("js", {code…})
                                     │ one JSON request per call
┌────────────────────────────────────▼────────────────────────────────┐
│  transport client → unix socket → native host router                │
│  ("js" → EXECUTE_JAVASCRIPT, "tab.new" → NEW_TAB, …)                 │
└────────────────────────────────────┬────────────────────────────────┘
┌────────────────────────────────────▼────────────────────────────────┐
│  extension service worker drives the real Chrome tab                │
└────────────────────────────────────┬────────────────────────────────┘
┌────────────────────────────────────▼────────────────────────────────┐
│  dl.acm.org page: server-rendered HTML; the embedded script runs in  │
│  the page's main world, with cookies and Cloudflare clearance        │
└─────────────────────────────────────────────────────────────────────┘
```

Four properties of this chain matter daily:

- **The script runs in the real page context.** This is the entire value
  of surf: a `fetch()` from inside the page carries the session cookies
  and the Cloudflare clearance, where the same request from Go receives a
  403.
- **Each `ExecuteTool` call is one socket round trip, and its result is
  capped at roughly 900,000 characters.** A 2.5 MB PDF is about 3.4 MB as
  base64. This single number dictates the download design in §7.
- **`window` persists between separate `ExecuteTool("js", …)` calls on
  the same tab.** Each call is a fresh script evaluation, but the tab's
  JavaScript global state survives. The download verb depends on this.
- **Numbers in tool responses decode as exact `int64`/`float64`, not as
  `float64`-only JSON.** Verbs read counts through `numberAsInt64` /
  `numberAsFloat`; a bare type assertion is a latent bug (a citation
  count of 0 today and 10 tomorrow must both decode).

Tab ownership follows a fixed contract. A tab the command created is
closed by default when the command finishes (`--keep-tab-open` opts out);
a tab supplied by `--tab-id`/`--window-id` is never closed. Read-only
flows may retry on transient tab lifecycle errors ("navigated or
closed", "Detached while handling command") via `withOwnedTabRetry`.

## 3. The investigation method

Every contract was established live before design, with three techniques.
The probes are in the ticket's `scripts/` directory and can be replayed
against any open tab.

1. **Network capture on navigation.** `surf network clear` → navigate →
   `surf network list`. This proved the no-API property and produced the
   search URL grammar. Its limitation was also learned here: capture
   follows the *active* tab, and it records page loads but silently
   misses in-page `fetch`/XHR traffic.
2. **In-page probes via `surf js`.** Small scripts returning plain
   objects: result-item shape, pagination links, sort menu, reference
   entries, cited-by entries, author microdata.
3. **Bundle archaeology.** When the citation export form's POST payload
   was needed, the site's own `main.bundle-*.js` was fetched in-page and
   searched. This recovered `UX.exportCitation`: the POST payload
   (`dois`, `targetFile: custom-<style>`, `format`) and, more
   importantly, the response shape — CSL-JSON rendered client-side by
   embedded citeproc-js.

One negative result deserves its own paragraph: **HTTP Range requests
are not honored.** A fetch with `Range: bytes=0-99` on a paper PDF
returns `200` with the full 2,508,571-byte body and `Content-Range:
null`. The download design could not use ranged fetches and had to be
revised before implementation — the design document was corrected and
the probe recorded in the contracts reference.

## 4. The verified site contracts

The full set with probes is in
`ttmp/2026/09/26/SURF-20260926-ACMDL…/reference/01-…-contracts.md`. The
subset that shapes the code:

**Search.**

```text
GET https://dl.acm.org/action/doSearch?AllField=<q>[&Title=<q>]
    &pageSize=<1..100>&startPage=<0-based>[&sortBy=<value>]
```

| Concern | Verified fact |
|---|---|
| Result items | `div.issue-item` — the first `div.search-result` on the page is a header block, never a result |
| Total hits | `.hitsLength` ("56,516"), inside that header block |
| Pagination | `startPage` is 0-based; `pageSize` honored up to 100 |
| Sort values | `EpubDate_asc`, `EpubDate_desc`, `downloaded`, `cited` — taken from the rendered menu, not guessed |
| Zero results | `.hitsLength` reads "0", zero `div.issue-item`, no dedicated empty-state element |
| Authors | Client-truncated: hidden authors are *removed from the DOM* behind a `+ N authors` button |
| Abstract | Truncated to four lines; the full text exists only on the paper page |

**Paper page (`/doi/<doi>`).**

| Concern | Verified fact |
|---|---|
| Title / abstract | `h1` / `#abstract` (the rendered "Abstract" heading word must be stripped) |
| Authors | schema.org microdata `[property='author']` with `givenName`/`familyName`; each author renders **twice** — dedupe by name |
| Venue / pages / DOI | `.core-self-citation` (three lines: venue+volume+issue, pages, DOI link) |
| PDF grammar | `/doi/pdf/<doi>`, `/doi/pdf/<doi>?download=true`, `/doi/epdf/<doi>` (eReader; unused) |

**References.** Fully server-rendered — present in the DOM on load, no
activation needed: `.bibliolist[data-list='references'] > .biblioentry`
with `.label` ("[1]"), `.citation-content` (the full reference text), an
optional `/doi/` link (only when the referenced work is in the DL), and a
Google Scholar lookup link whose `doi=` parameter recovers DOIs for
off-DL references.

**Cited-by.** The badge total lives at
`a[href='#tab-citations'] .citation .bold` ("6,871"). The first page is
server-rendered as `li.citedByEntry` (Crossref data; includes non-ACM
papers; the title is an *unclassed* third-child span). The complete list
is the "View all" URL, `/action/ajaxShowCitedBy?doi=<doi>` — one HTML
page of `li.references__item` entries, **5.8 MB** for the 6,871-citation
probe paper. The badge total and the list length disagree (6,871 vs
6,811); both numbers are surfaced, never reconciled silently.

**Citation export.** The page carries a form posting to
`/action/exportCiteProcCitation`, but the response is CSL-JSON plus the
CSL style XML; the citation text is produced by citeproc-js in the
browser. The verb therefore drives the widget: click
`[data-target="#exportCitation"]`, wait for `.copy__text.csl-response` to
fill, set `#citation-format`, dispatch `change`, wait again. A verified
pitfall: dispatching `change` without the trigger click first leaves the
widget stuck at "Loading …".

**Login state.** The account bar is `div.loginBar`; logged-in pages
render `a.indivLogin .profile-text` (the user's name) and a
`/action/doLogout` link. The "log in to your account" link on paper
pages renders *even while logged in* and is not a logged-out marker.
Since ACM's Open Access transition, none of the four verbs requires
login; the state is reported, not depended on.

## 5. The Cloudflare experiment

The single most consequential pair of measurements in the ticket:

```text
fetch("https://dl.acm.org/doi/pdf/<doi>", {credentials: "include"})
  from inside a logged-in tab
  → 200, application/pdf, 2,508,571 bytes, magic bytes "%PDF-"

curl / Go http.Get on the same URL, with an honest User-Agent
  → 403, cf-mitigated: challenge, server: cloudflare
  → HTML challenge page referencing challenges.cloudflare.com
```

Two rules follow. First, the download verb must be browser-mediated; the
marketplace-verb pattern of "resolve in the tab, fetch the bytes from Go
with `http.Get`" is verifiably broken here. Second, when the tab ever
renders Cloudflare's interstitial, the verbs classify it
(`cloudflare-challenge`), abort without retrying, and tell the user to
clear it by hand — the same policy as the marketplace captcha rule. An
interstitial was observed twice during development, on rapid
back-to-back multi-navigation runs; single-navigation flows never hit it.

## 6. The four verbs

All four are dual-mode Glazed commands (Markdown by default,
`--with-glaze-output` for rows), share the DOI normalizer, the error
vocabulary, and the tab-ownership contract, and register under
`surf acm`:

```text
surf acm search <query>   [--title] [--sort latest|earliest|cited|downloaded]
                          [--page N] [--page-size N] [--limit N]
surf acm paper  <doi|url> [--with-references] [--with-citations]
                          [--all-citations] [--limit-citations N]
surf acm download <doi|url> --save-to <dir|file> [--force] [--dry-run]
surf acm cite <doi|url>   [--style bibtex|endnote|acm]
```

`normalizeACMDOI` accepts bare DOIs, `doi:` prefixed, `doi.org` and
`dl.acm.org` URLs (including `/doi/pdf/…` and `/doi/epdf/…`), and reduces
them all to `10.1145/<id>` + the paper page URL. Anything else — other
registrants, other hosts, bare words — is rejected with an actionable
error. The CLI `--page` is 1-based and maps to the site's 0-based
`startPage`; the mapping lives in one tested function because it is an
off-by-one a mock cannot catch.

The page scripts return a normalized error object instead of throwing:

```text
{ error: { kind, message } }   kind ∈ cloudflare-challenge, not-found,
                               no-results, not-a-pdf, extract-failed
```

`acmExtractScriptError` lifts that into a Go error, and `extract-failed`
always carries `location.href` — it is the "the site changed" signal
that tells a maintainer to replay the probes before reading any code.

`acm paper --all-citations` is the one multi-navigation flow: extract
the paper, then navigate the same owned tab to the View all URL, wait,
and run the cited-by-all extraction in the page. The 5.8 MB page must be
navigated to; routing it through the `js` output cap is impossible.

## 7. The download pipeline

`acm download` is where all the constraints compound. The design, with
the numbers that justify each step:

```text
resolve    tab opens the paper page; the script returns title + pdfURL
           (derived from the DOI; the page only confirms metadata)

fetch      page script fetches the WHOLE PDF once (Range is not honored)
           into window.__surfAcmPdf = { url, bytes: Uint8Array, total },
           verifying the payload starts with "%PDF" before caching —
           a paywalled or challenged response is an HTML page and must
           never be cached or written

chunk      each round trip slices [offset, offset+500000) from the cache,
           base64-encodes it in 0x8000 slices (call-stack limits), and
           returns { b64, length, total, done }
           500,000 bytes ≈ 666 KB base64, safely under the ~900,000-char
           js output cap; a 2.5 MB PDF is 5–6 round trips, the 76 MB
           Rex West paper was ~153

commit     Go writes to <dest>.part, verifies %PDF- on the first chunk,
           renames on completion, removes the .part on any failure,
           refuses to overwrite an existing file without --force
```

Two consequences of the page-side cache deserve emphasis. Retries become
cheap: every step is a read-only GET of a public PDF, so transient tab
lifecycle errors retry, and a retried chunk re-slices the cache instead
of re-downloading — the opposite of the marketplace mint rule, where a
replay would double-count. And the User-Agent question that cost the
marketplace verbs an hour (Go's default UA getting 403) disappears,
because the browser sends its own.

The resolve phase also serves `--dry-run`, which prints the plan (title,
source URL, destination, chunk size) without fetching.

## 8. Test strategy

Three layers, following the surf-go tutorial contract:

**Unit tests** cover URL builders (golden URLs including the 1-based→
0-based page mapping and the sort enum), DOI normalization (acceptance
and rejection tables), response parsing fixtures, row shaping, and
Markdown rendering. They also carry the *script-text locks* — the only
layer that can see a URL parameter name or a selector string:

```go
if !strings.Contains(script, "h4.issue-item__title a") { … }
if strings.Contains(scriptNoComments, "Range") { … }   // server ignores Range
if strings.Contains(goSource, "http.Get(") { … }        // Go must never fetch bytes
```

A lock must assert on code, not on documentation. The first version of
the Range lock failed against the download script's own header comments
explaining *why* there are no Range requests; the fix is a shared
`acmStripScriptComments` helper, matching the rule already recorded in
the API-first tutorial.

**Mock-host integration tests** drive the real command against a fake
unix-socket host that asserts the exact tool sequence. The paper
verb's `--all-citations` test taught the harness lesson of this ticket:
a frame-counted mock (accept exactly N connections) deadlocks whenever
the readiness probe loop fires a variable number of probes around a
navigation. The stateful mock — track `currentURL`, answer readiness
probes against it, accept connections until the listener closes, assert
the sequence afterwards — is the shape that survives, and the download
test uses the same shape with real base64 chunks, verifying `.part`
cleanup, magic bytes, and title-based naming.

**Real-browser validation** replays the ticket probes and runs each verb
against the live site: the two-orderings rule (the same query under
`earliest` and `latest` must move the first hit — it does: 1955 JACM vs
a 2026 paper), the zero-hits state, glaze rows, tab hygiene, and, for
download, actually opening the PDFs.

## 9. Failure modes recorded during the work

Each of these is documented in the ticket diary with its diagnostic:

1. **The header block.** Selecting `div.search-result` as items emits one
   bogus "result" per page whose title is the query echo. Item selection
   is `div.issue-item`, and a lock enforces the absence of the other.
2. **DOM-truncated author lists.** The search page removes hidden
   authors from the DOM; the rows ship the visible authors plus an
   `authorCountHidden` count, and only the paper page provides the full
   list (via microdata).
3. **`.loa` on detail pages is the editorial board.** The first author
   probe returned the editors' names on a paper with four different
   authors. Authors come from `[property='author']` microdata, deduped
   because each author renders twice.
4. **Readiness by exact URL in a fragile mock.** The first integration
   test deadlocked because the mock's ready-probe response used a
   slightly different URL than the command's `URLExact` expectation;
   readiness matches by `URLPrefix` now, which is also more robust
   against server-side URL normalization.
5. **The `waitForCondition` wrapper.** `waitForCondition` wraps its
   predicate's result in `{value, waitedMs}`. A predicate that returned
   an object made the cite verb return a nested wrapper instead of the
   citation text — the first endNote run rendered an empty citation
   while the page showed 866 characters. Predicates return plain values.
6. **Range optimism.** The original download design assumed ranged
   fetches; the probe disproved it before implementation. The revised
   cached-buffer design shipped instead.
7. **The intermittent interstitial.** Two live `--all-citations` runs
   hit a real Cloudflare challenge; isolated retries of the same URL
   succeeded. The abort-with-actionable-message path is the correct
   response, and it worked as designed.

## 10. Validation evidence

The commands, with real output excerpts:

```text
$ surf acm search "graph neural network" --limit 5
- Scope: Searched The ACM Full-Text Collection (849,300 records)
- Total hits: 56516
### 1. [Stuart-Landau Oscillatory Graph Neural Network](…3792675)
- WWW '26: Proceedings of the ACM Web Conference 2026 · April 2026 · Pages 1469–1480
- Authors: Kaicheng Zhang; David N. Reynolds (+2 more)

$ surf acm paper 10.1145/359545.359563 --with-references --with-citations
- Venue: Communications of the ACM, Volume 21, Issue 7
## References (4)
## Cited by (total: 6871; showing 15)

$ surf acm download 10.1145/1276377.1276401 --save-to ~/Downloads/stamen-papers
- Saved: …/Apparent ridges for line drawing.pdf
- Size: 6964502 bytes          (file: PDF document, version 1.6)

$ surf acm cite 10.1145/359545.359563 --style bibtex
@article{10.1145/359545.359563, author = {Lamport, Leslie},
  title = {Time, clocks, and the ordering of events…}, …}
```

The final integration test was real work: a 20-file literature batch for
a line-drawing rendering project, fetched in one session — 15 `acm
download` runs and roughly 20 `acm search` runs, covering papers from 76
MB (Rex West 2021, ~153 chunks) down to a 343 KB HAL report, plus a
Medium post extracted as Markdown with its GitHub-gist shader code. No
truncated or corrupted PDFs; no interstitials on single-navigation flows.
Two adjacent results worth recording: HAL serves its PDFs without
Cloudflare (a plain `curl` works — the browser-mediated design is an
ACM-specific necessity, not a general one), and Medium's article body
extracts cleanly from the logged-in browser session while its code
samples live in embedded gists that must be fetched separately.

## 11. Artifacts and status

Production code, all in `go/internal/cli/commands/`:

| File | Role |
|---|---|
| `acm_doi.go` | DOI/URL normalization, search/cited-by/PDF URL builders, sort enum |
| `acm_errors.go` | script error vocabulary lift + `acmStripScriptComments` |
| `acm.go` | the `surf acm` group |
| `acm_search.go` + `scripts/acm_search.js` | search verb |
| `acm_paper.go` + `scripts/acm_paper.js` | paper, references, cited-by, view-all |
| `acm_download.go` + `scripts/acm_download.js` | browser-mediated PDF download |
| `acm_cite.go` + `scripts/acm_cite.js` | citation export via the site's citeproc |
| `acm_*_test.go`, `cmd/surf-go/integration_test.go` | the three test layers |

Registration lives in `go/cmd/surf-go/main.go`; both Go packages are
green (`go test ./internal/cli/commands/ ./cmd/surf-go/`). The ticket is
functionally complete: all five docmgr tasks checked, diary and
contracts reference updated through the implementation, commits
`b9bd337` (design docs), `0a585f7` (search), `2943781` (paper),
`af4a2bf` (download), `bd1c3df` (cite), `e3ea044` (docs completion).

## 12. Open questions

- The logged-out header shape is unverified (an incognito `window.new`
  fails host-side with a null-id error, and the user's session was not
  to be logged out). `loggedIn` stays a best-effort presence marker.
- The Cloudflare interstitial is classified by page title
  (`Just a moment…`); more markers should be recorded the first time one
  is actually observed in a surf tab.
- `pageSize` is verified up to 100; larger values are unprobed and the
  flag is capped accordingly.
- The `/pb/widgets/citedBy` pagination endpoint returns labels-only JSON
  config; the paginated content surface was never needed and remains
  unmapped.
- Fielded search beyond `Title=` (`Author=`, `Keyword=`) is expected to
  follow the same grammar but was not probed; `--title` is the only
  field flag shipped.
