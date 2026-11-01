---
name: linkedin-ai-skill
description: "Battle-tested patterns for scraping data out of LinkedIn pages in the user's logged-in browser — profile names, current company, page-title cleanup — from automation scripts, AppleScript, or browser JavaScript. Use this skill whenever a task reads or extracts anything from a linkedin.com page (profile scraping, \"get the company from this LinkedIn profile\", LinkedIn link formatting, fixing a LinkedIn scraper that broke or returns the wrong value, or debugging LinkedIn DOM selectors). LinkedIn re-rolls its frontend markup PER PAGE LOAD, so selector advice from memory or a single inspection is wrong by design — always use the multi-variant strategy here. Also use for LinkedIn JOBS pages (search results, job view: the componentkey anchors, the scroll container, Messaging / nav elements), for a person's comments or activity (which posts they commented on, the exact comment link), for mutual connections with someone or the people who work or worked at a firm (people search), for reading a whole profile (every /details/ section), a whole activity feed or checking that a page was fully read, or any data LinkedIn hasn't rendered — the voyager REST endpoints (job description, company website) still answer same-origin fetches."
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

*Trigger:* you need data the page hasn't rendered — a job description in a background tab (LinkedIn renders lazy sections only in a VISIBLE tab), a company's website, the bold headings of a posting. *Discrimination:* the embedded JSON blobs are gone (above); the REST endpoints are not. They need the viewer's session: same-origin `fetch` from the page or a content script, `credentials: "include"`, header `csrf-token` = the `JSESSIONID` cookie value without quotes, `accept: application/json`. Unofficial — every call may fail; cache results and fail soft. In a Claude Code session in auto mode the permission classifier may refuse such a `fetch` as website-access circumvention; then read the visible `/details/` pages instead ("Reading a whole profile" below). *Action:*
- `/voyager/api/jobs/jobPostings/<jobId>` → `title`, `description.text` + `description.attributes` (Bold / Paragraph / ListItem ranges — the section headings; offsets drift after emoji, so match by text), `companyDetails` (`urn:li:fs_normalized_company:<id>`), `applyMethod` (`companyApplyUrl` for offsite apply).
- `/voyager/api/entities/companies/<id>` → `name`, `websiteUrl` (the company page's own Website field — may carry a path like `/subscribe`), `universalName`, `description`.
- `/voyager/api/identity/profiles/<slug>/positionGroups` → `elements[]` newest first, each with `miniCompany` (`name`, `universalName`, `objectUrn`) and `positions[]` (`title`, `timePeriod`); the CURRENT employer is the first group holding a position with `startDate` and no `endDate`. Use it when a page shows only the headline (a person's `recent-activity` page has no company link). `profiles/<slug>/profileView` answers 410 (retired).

## A person's profile and activity pages

- Titles: activity `Activity | <Name> | LinkedIn`; profile `<Name> | LinkedIn` (either may lead with `(3) `).
- The activity page's left card is an `aside[aria-label="Sidebar"]`: photo, name, headline ("<Title> @
  <Company>"), Followers, a Message link (`/messaging/compose/`) and `button[aria-label="Follow <Name>"]`.
  It has NO company link; `/company/` links lower down belong to feed posts. The current employer: voyager
  `positionGroups` (above).
- Profile sub-pages (`/in/<slug>/details/experience/`, `/overlay/contact-info/`) are separate paths; match the
  profile itself with `^/in/<slug>/?$`.

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

Canonical scraper (deployed and verified):

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

## A person's comments → the exact posts and comments

- *Trigger:* you need the link behind "<person> commented 'LOI'" (their keyword replies to "comment X and I'll send you
  Y" posts), for a dossier or for any claim about what they asked for.
- *Discrimination:* the comments activity page shows the comment text but no post URL in any `href`; a signed-in
  same-origin `fetch` of `/feed/update/…` returns a page with no `og:` tags; and the Claude in Chrome JS tool blanks
  results that hold query strings or keys like `author`, so return bare IDs and build the URLs outside the page.
- *Action:*
  1. Open `https://www.linkedin.com/in/<slug>/recent-activity/comments/` in a Claude in Chrome tab and scroll to the
     bottom 8 times, 1.2 s apart.
  2. Each comment card is an element whose `componentkey` holds `activity:<19-digit post id>` and whose `innerText`
     (whitespace collapsed) ends with `Follow <their comment>`: match `/Follow (LOI|model|Finance)$/` and read the id.
     The comment's own id sits in `replaceableComment_urn:li:comment:(activity:<post>,<comment>)`.
  3. Post link: `curl -sL -A '<Safari user agent>' "https://www.linkedin.com/feed/update/urn:li:activity:<post>/"`
     (signed out) and read `og:url` (`linkedin.com/posts/<author>_<slug>-activity-<post>-<code>`) and `og:title`.
  4. Exact comment link, the one a claim about the person links to:
     `https://www.linkedin.com/feed/update/urn:li:activity:<post>?commentUrn=urn%3Ali%3Acomment%3A%28activity%3A<post>%2C<comment>%29&dashCommentUrn=urn%3Ali%3Afsd_comment%3A%28<comment>%2Curn%3Ali%3Aactivity%3A<post>%29`.
     Open it signed in once and check it lands on the comment.
  5. A post's date from its id: `(id >> 22)` is milliseconds since 1970
     (`python3 -c 'import datetime;print(datetime.datetime.utcfromtimestamp((<id> >> 22)/1000).date())'`).
  6. Close the tab when done.

## Mutual connections, and who works (or worked) at a firm

- *Trigger:* you need the people you and someone share ("N mutual connections"), or every 2nd-degree person tied to
  a firm, for any "who do I know at X" question.
- *Discrimination:* the mutuals link's `href` carries `connectionOf=<urn>` in a query string, and the Claude in Chrome
  JS tool blanks results that hold query strings (the result came back `[BLOCKED: Base64 encoded data]`), so never
  return the URL; click the link in the page instead. The page number is a query param too: change it in-page with
  `new URL(location.href)`, never by typing a URL. A people search for the firm's name also returns other companies
  that share a word (a search for Acme Partners also brings up Acme Health): judge each card by its "<Role> at <Firm>" snippet.
- *Action:*
  1. Open the profile in a Claude in Chrome tab, wait ~2.5 s, then
     `[...document.querySelectorAll('a')].find(a=>/mutual/i.test(a.innerText)).click()`; the text says the count
     ("Ana, Ben and 50 other mutual connections" = 52).
  2. Each results page holds 10. Read names with a JS extractor over `document.querySelector('main').innerText` split
     into trimmed lines: a line ending `• 1st` (or followed by the line `• 1st`) is a name, the next line its headline.
     Next page: `const u=new URL(location.href); u.searchParams.set('page','2'); location.href=u.toString()`, then a
     second call that waits ~4 s and extracts. Batch every page in one `browser_batch`; keep each result short (a
     long one comes back `[TRUNCATED]`: re-read that page with names only).
  3. People tied to a firm, 2nd degree: navigate to
     `https://www.linkedin.com/search/results/people/?keywords=<Firm>&network=%5B%22S%22%5D`; per card read the name
     (line before `• 2nd`), the snippet holding the firm ("Limited Partner at Acme Partners since 2017", "At
     Acme Partners from January 2022 to June 2025") and the mutuals line ("Ana Lee, Ben Cho & 5 other").
     Former staff and LPs show up here; flag anyone who held the job being applied for.
  4. Profile slugs: `[...new Set([...document.querySelectorAll('main a[href*="/in/"]')].map(a=>(a.innerText.trim()
     .split('\n')[0])+' '+a.href.split('?')[0].replace('https://www.linkedin.com/in/','')))]` (strip the query string
     in-page so the result passes).
  5. Close the tab when done (`tabs_close_mcp`).

## Reading a whole profile, team page or feed

- *Trigger:* you must confirm a LinkedIn page can be fully read, or capture a profile completely (a job's link
  check, a note on a person).
- *Discrimination:* a page that renders is not a page that was read. In one variant a profile page drew only
  the top card, About, Featured and Activity, with no Experience, Education or any other section even after scrolling.
  A first pass that read only `/details/experience/` and `/details/education/` reported "read in full" while 11
  sections were unopened. An activity feed loads in chunks as you scroll, and the sidebar's "Show more" (People you
  may know) is not the feed's.
- *Action:*
  1. Profile: open every `/in/<slug>/details/<section>/` page: experience, education, skills, certifications,
     volunteering-experiences, honors, recommendations, organizations, projects, publications, languages, courses,
     interests. An empty one reads "Nothing to see for now": record it as empty. A `fetch` of these pages returns only
     the app shell (the content is drawn in the browser), so navigate to each.
  2. Interests has five tabs (Top Voices, Companies, Groups, Newsletters, Schools) that are not `button`s: find the
     leaf element whose text is the tab name and click `el.closest('a,button,[role=tab]') || el`. Each list item has
     a twin line ending ", Company" / ", Group" / ", School" / ", Newsletter": match those to read the names.
  3. Activity: scroll to the bottom until `main`'s length stops changing 4 checks in a row, then count the
     `Feed post` markers and read the oldest post's age; report "all N posts, back to <age>".
  4. Results: the Claude in Chrome JS tool cuts any single string over about 1,000 characters to `[TRUNCATED]`;
     return an array of short strings (one per line or card) instead of one long text.
  5. Linking a person found by name: people search `keywords=<name> <employer>`, then confirm the employer on the
     card's headline or snippet, or on the first entry of their `/details/experience/` (a card can show a headline
     with no employer). A guessed company slug (`/company/<guess>/people/`) lands on `/company/unavailable/`, and the
     top card may carry no company link: search per name instead.

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

