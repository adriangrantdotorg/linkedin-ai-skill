---
name: linkedin-ai-skill
description: "Battle-tested patterns for scraping data out of LinkedIn pages in your logged-in browser — profile names, current company, page-title cleanup — from automation scripts, AppleScript, or browser JavaScript. Use this skill whenever a task reads or extracts anything from a linkedin.com page (profile scraping, \"get the company from this LinkedIn profile\", LinkedIn link formatting, fixing a LinkedIn scraper that broke or returns the wrong value, or debugging LinkedIn DOM selectors). LinkedIn re-rolls its frontend markup PER PAGE LOAD, so selector advice from memory or a single inspection is wrong by design — always use the multi-variant strategy here. Also use for LinkedIn JOBS pages (search results, job view: the componentkey anchors, the scroll container, Messaging / nav elements) or any data LinkedIn hasn't rendered — the voyager REST endpoints (job description, company website) still answer same-origin fetches."
---

# Scraping LinkedIn (logged-in DOM)

Target: linkedin.com pages rendered in your logged-in Chromium browser, scraped via JavaScript injection (any automation tool's run-JavaScript action, or `osascript` → Chrome `execute … javascript`). Requires Chrome's View → Developer → **Allow JavaScript from Apple Events**.

## The cardinal fact: markup is re-rolled per page load

LinkedIn A/B-serves **different frontend variants of the same page on every load** — verified live: the same profile tab returned one DOM shape, then a different one after re-render, with zero code changes. Consequences, all non-negotiable:

- **Never** rely on one inspection of one page. A selector that works on the page you probed fails on the next load.
- **Never** use CSS class names — they're hashed per build (`_0d982122`, `cb81723c`) and rotate.
- A scraper must collect candidates from **every known variant** and disambiguate structurally (see below).
- When a scraper "suddenly broke", suspect a variant you haven't seen before — re-probe the live page, don't stare at the code.
- Verify fixes on **fresh page loads** (open the URL in a new tab), not on the already-rendered tab you probed.

## What's NOT available (don't waste time)

On logged-in SPA pages there is **no** `meta[name=description]`/og: tags, **no** JSON-LD, **no** `<code>` data blobs (the classic `bpr-guid` voyager JSON embedded in the page is gone), and `document.title` is just `"Name | LinkedIn"` (sometimes prefixed `(3) ` with the notification count) — no company in it. The rendered DOM is the only source *in the page* — but see the next section: the voyager REST endpoints still answer.

## The voyager API still answers same-origin fetches

*Trigger:* you need data the page hasn't rendered — a job description in a background tab (LinkedIn renders lazy sections only in a VISIBLE tab), a company's website, the bold headings of a posting. *Discrimination:* the embedded JSON blobs are gone (above); the REST endpoints are not. They need the viewer's session: same-origin `fetch` from the page or a content script, `credentials: "include"`, header `csrf-token` = the `JSESSIONID` cookie value without quotes, `accept: application/json`. Unofficial — every call may fail; cache results and fail soft. *Action:*
- `/voyager/api/jobs/jobPostings/<jobId>` → `title`, `description.text` + `description.attributes` (Bold / Paragraph / ListItem ranges — the section headings; offsets drift after emoji, so match by text), `companyDetails` (`urn:li:fs_normalized_company:<id>`), `applyMethod` (`companyApplyUrl` for offsite apply).
- `/voyager/api/entities/companies/<id>` → `name`, `websiteUrl` (the company page's own Website field — may carry a path like `/subscribe`), `universalName`, `description`.

## Jobs pages: the anchors (search results + job view)

Stable across loads because they are LinkedIn's own `componentkey` / `data-testid` / ids, not classes:
- Job cards: `[componentkey^="job-card-component-ref-<jobId>"]`, the `Dismiss <title> job` button; classic: `li[data-occludable-job-id]`.
- Detail sections: `[componentkey="JobDetails_AboutTheJob_<jobId>"]` (same on `/jobs/view/<id>/` and in the `?currentJobId=` pane); "… more" is `[data-testid="expandable-text-button"]` (text holds "…" while collapsed, "less" once open; its handler focuses the text).
- Search page: list `[componentkey="SearchResultsMainContent"]` (a `[data-testid="lazy-column"]`), header `[componentkey="searchResultsHeaderComponent"]`, filters `#JobsSearchFilters` in a sticky `[role="toolbar"]`, both columns in a wrapper fixed at 1128px; global nav = the `<header>` holding `[data-testid="primary-nav"]` in a 52px grid row; the page scrolls in `main#workspace` (`overflow-y: scroll`), NOT the lazy columns; Messaging is the classic `#msg-overlay`, mounted inside the `#interop-outlet` shadow root (not in a background tab).

## Profile top card: the three known variants

The "current company" in a profile's top card (right rail / under the name) has been observed in three markups — same URL, different loads:

| Variant | Company marker | Notes |
|---|---|---|
| A | `img[alt="View company: <Name>"]` inside `a[href*="/company/"]` | Feed posts in the activity section use the **same** alt pattern — document order alone picks the wrong one |
| B | `img[alt="<Name> logo"]` inside `a[href*="/company/"]` | Schools link to `/school/` instead — the href distinguishes them |
| C | alt-less `img[src*="company-logo"]` + a bare `<span>` with the name; **no link at all** | School logos ALSO use `company-logo` CDN URLs — src can't tell school from employer |

## The disambiguator that survives all variants: vertical position

Across every variant: **the top card is the highest company element on the page** (feed/experience modules render lower), and **the company row always sits above the school row**. So: collect candidates from all three patterns, compute absolute Y (`rect.top + window.scrollY` — scroll-proof), and take the minimum. Skip zero-size (hidden) elements and negative-Y stubs.

Canonical scraper:

```js
(function () {
  var cands = [];
  function push(name, el) {
    if (!name) return;
    var r = el.getBoundingClientRect();
    if (!r.width && !r.height) return;
    var y = r.top + window.scrollY;
    if (y < 0) return;
    cands.push({ name: name.trim(), y: y });
  }
  Array.prototype.slice.call(document.querySelectorAll('img[alt^="View company"]')).forEach(function (i) {
    push(i.getAttribute('alt').replace(/^View company:\s*/, ''), i);
  });
  Array.prototype.slice.call(document.querySelectorAll('a[href*="/company/"] img[alt$=" logo"]')).forEach(function (i) {
    push(i.getAttribute('alt').replace(/\s*logo$/, ''), i);
  });
  Array.prototype.slice.call(document.querySelectorAll('img[src*="company-logo"]')).forEach(function (i) {
    if ((i.getAttribute('alt') || '').length) return; // covered above
    var row = i.parentElement, name = '';
    for (var d = 0; d < 4 && row && !name; d++) {
      var t = (row.textContent || '').trim();
      if (t && t.length < 60 && t.indexOf('\n') === -1) name = t;
      else row = row.parentElement;
    }
    push(name, i);
  });
  if (cands.length) {
    cands.sort(function (a, b) { return a.y - b.y; });
    return cands[0].name;
  }
  var btn = document.querySelector('button[aria-label^="Current company"]'); // legacy DOM, pre-2026
  if (btn) {
    var m = (btn.getAttribute('aria-label') || '').match(/^Current company:\s*(.+?)\.\s*Click/);
    if (m) return m[1].trim();
    return (btn.innerText || '').trim().split('\n')[0];
  }
  return '';
})();
```

Known edge: a profile whose top card lists only a school (no current employer) returns the school as "company" — variant C gives no way to tell them apart. Acceptable; the consumer should let the user override.

## Anchors that held up vs. anchors that failed

**Held up:** alt-text patterns (`View company: …`, `… logo`), href shapes (`/company/`, `/school/`, `a[href*="contact-info"]` for locating the top card), CDN src substrings (`company-logo`), absolute-Y geometry, text-leaf matching.

**Failed in practice:** hashed class names (rotate per build); `aria-label="Current company: …"` buttons (old DOM, gone from current variants — kept only as last-ditch fallback); document-order "first match" (feed modules can precede the top card match); scoping to the top-card `<section>` (the right rail lives **outside** the section containing the name and Contact info in some variants); assuming the DOM you probed minutes ago is the DOM that's there now.

## Debugging workflow

1. Test JS from the shell against the real tab — this is byte-identical to what an automation tool's run-JavaScript action does:
   ```applescript
   tell application "Google Chrome"
     repeat with w in windows
       repeat with t in tabs of w
         if URL of t contains "<profile-slug>" then
           return execute t javascript "<JS — escape \\ then \" when building from a file>"
         end if
       end repeat
     end repeat
   end tell
   ```
   (Find the tab by URL substring — "active tab of front window" silently probes whatever tab the user switched to.)
2. When the value is wrong, don't guess — enumerate: dump all candidate elements with alt/href/absolute-Y/closest-section and identify which one is the top card *on this render*.
3. Test on **at least two different profiles** and on **fresh loads** before declaring a selector fixed.
4. If the scraper runs inside an automation tool, do the final verification through that tool itself, since that exercises its own variable handling and front-browser resolution.

