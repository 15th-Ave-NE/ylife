# Known pitfalls — full detail

Quick index lives in `SKILL.md`. Two related traps are covered in `SKILL.md`
itself instead of here because they're tied to a section above: the
`kill -HUP` vs `systemctl restart` trap (under "The #1 rule"), and the
not-every-cache-is-keyed-`"data"` trap (under "Caching model"). Everything
else is expanded below, in the same order as the SKILL.md index.

## 1. A Tailwind class pair is not a DOM token

Light mode turned every hardcoded colour utility into a pair
(`bg-slate-100 dark:bg-slate-800`), which is fine inside a `class="..."`
attribute and broken everywhere a template passed that same string to the DOM
as a *token* instead of a class attribute:

- `classList.add`/`remove` are variadic and take one token per argument —
  passing `"bg-x dark:y"` as a single argument is one bad token, not two.
- `classList.toggle`/`contains` take exactly one token and throw
  `InvalidCharacterError` on a string containing a space.
- A selector is worse: `closest('.a dark:b')` parses as a *descendant*
  selector, so it throws or silently matches nothing.

This bit 42 call sites and one `closest()` call, and it's invisible on page
load — the throw happens on a click, or inside an IIFE whose `catch`
swallows it. In `fed.html` it killed that page's entire init block with only
a console line as a symptom.

**Fix:** use the variadic form (`classList.add(a, b)`), or
`toggleClasses(el, pair, on)` from `base.html`. For a selector, use a
`[data-*]` hook instead — restyling can't invalidate it.
**Test:** `tests/test_theme_classes.py` catches all three shapes, no browser
needed.

## 2. `text-<hue>-400` is invisible in light mode

The dark theme picks 400-level hues so they glow against near-black;
`text-emerald-400` on white is 1.87:1 contrast, so a table of percentage
changes reads as a smudge. Light counterparts must land on **shade-700** —
the first rung that clears AA on white for every hue in the set
(`emerald-600` is 3.4:1, `amber-600` only 3.0:1 — not enough). Same trap in
reverse for greys: `slate-500` is the muted-text floor at 4.76:1 on white,
so both `text-slate-500` and `text-slate-600` collapse onto it rather than
going paler. A fade overlay is the loud version of this —
`linear-gradient(…, #0f172a)` over a white card renders as a solid **black
block**, which is why `--t-fade-from/to` custom properties exist instead.

## 3. `kill -HUP` does not reload code under `--preload`

See "The #1 rule: restart, never HUP" in `SKILL.md`.

## 4. A `DeferLoad` anchor that's hidden defers nothing

`IntersectionObserver` can never fire for an element with no box, so
`deferload.js` detects that case and runs the loader **immediately** — the
panel still fills, which is exactly why this ships unnoticed. The page looks
lazy while actually fetching everything on load. Nearly every card in
`history.html` and `fed.html` is `style="display:none"` until its own loader
reveals it, so the obvious id is usually the *wrong* anchor.

**Fix:** anchor on the card's visible loading placeholder (`#forecastLoading`,
`#peLoading`), or the nearest element that's in flow from first paint — not a
visible anchor nested inside a hidden card, which is equally dead.
Registration order matters too: `deferload.js` waits for layout before
observing, so a `when()` call issued right after an `await` gets measured
against the layout at that instant — register once the page has reached its
real height.
**Test:** `tests/test_deferload_anchors.py` checks every call site for all
three shapes, no browser needed.

## 5. A custom pull-to-refresh stacks with the browser's native one

Chrome on Android (and iOS standalone) already has the gesture, so a
hand-rolled one fires **twice** — the page reloads out from under its own
animation. `pulltorefresh.js` injects `html { overscroll-behavior-y: contain }`
to take the native gesture off the table, and injects it **from JavaScript**
specifically so a script that fails to load leaves the native gesture intact
rather than removing it with nothing in its place. Its CSS is
self-contained too, since Tailwind here is compiled
(`css/tailwind.css`, rebuild with `build_css.sh`) — a class the script
invents simply isn't in the bundle. The non-passive `touchmove` this needs
costs the browser its fast-scroll path, which is why it's bound per-gesture
on a touch starting at `scrollTop === 0`, not for the page's whole lifetime.

## 6. There is no Chart.js date adapter

`base.html` loads `chart.umd.min.js` alone; a `type: 'time'` axis without an
adapter throws inside Chart.js and leaves an empty canvas. Every existing
chart on the site uses a category axis instead, which is fine for evenly
spaced series. For a genuinely irregular series, use `type: 'linear'` over
epoch milliseconds with a tick `callback` — see the two history charts in
`fedwatch.html`. A category axis would be worse than merely wrong here: it
spaces points evenly, so the 1982–90 flurry of Fed moves and the 2009–15 flat
line would occupy equal width and misstate the history the chart exists to
show.

## 7. Nested `<button>` elements

Browsers auto-close the outer button when they see a nested one, which
detaches sibling markup from its intended parent container. Always use
`<div>` or `<span>` for clickable elements that live inside a button.

## 8. `animation: … both` makes every card a permanent stacking context

`.fade-up` in `base.html` is `animation: fadeUp .35s ease both`, and the
`both` fill mode means the animation's effect never stops applying — so each
card is a CSS stacking context for the life of the page, not just for
350 ms. An absolutely-positioned menu inside card A therefore resolves its
`z-index` **within card A**, and card B further down the document paints
over it however high that number goes. On `/agents` the model and thinking
menus opened, rendered, looked correct, and silently swallowed every click —
Playwright naming the intercepting element is the only reason this was
diagnosed rather than filed as a flaky click.

**Fix:** lift the **ancestor card** (`[data-sel-raise].ag-raised`, toggled by
the listbox's own open/close so a closed control leaves the sticky
sub-header on top) — not the menu itself. The status filter dropdown never
hit this only because it happens to live in the last card on the page, so
"the existing dropdown works" is not evidence a new one will.

## 9. `routes.py` is monolithic

5200+ lines: all routes, API endpoints, cache logic, and background tasks in
one file. Grep for the route/function you need rather than trying to skim
or hold the file's structure in your head.

## 10. Google Maps API key

yPlanner needs a valid **billing-enabled** API key. Symptom of a bad key:
"Oops! Something went wrong" with a purple stripe.

## 11. SSH deploy needs a real key

The `id_ed25519` key on this machine does not have EC2 access. Use SSM
`send-command` instead unless the user hands you a `.pem` explicitly.

## 12. Never fit an ML model in a request process

Prophet (`cmdstanpy`) and `pmdarima.auto_arima` each retain hundreds of MB
that glibc never returns to the OS — a worker that served one
`/api/forecast` request stayed ~880 MB larger for the rest of its life. Ten
such requests caused nine OOM kills in 48h, and since the kernel picks its
OOM victim globally, they took *other apps* down too, not just yStocker.

`forecast.py` now runs fits in a `subprocess`
(`python -m ystocker.forecast <TICKER> <OUT>` via `run_forecast_isolated()`),
deliberately **not** `multiprocessing`: `fork` would inherit cache locks held
by the background threads, and `spawn` re-imports the parent's `__main__` —
which under gunicorn is the venv launcher script, not this app.

## 13. Dead FRED series return HTTP 200

`MBST` and `WASDRAL` still serve well-formed CSV years after they stopped
publishing, so stale data flows in silently and corrupts anything derived
from it. Prefer the Wednesday-level `WSHO*` ids. `freshness.series_health()`
infers each series' publication cadence from its own observation dates and
flags a trailing gap of more than `cadence * 3 + 7` days — deliberately
biased toward flagging, since a false positive costs one log line and this
particular failure went unnoticed for years. Tune with
`FRESHNESS_CADENCE_TOLERANCE` / `FRESHNESS_CADENCE_GRACE_DAYS`. Note a
`stale` value of `None` means "too few observations to tell" — not the same
as healthy.

## 14. Every outbound call needs an explicit timeout

`yf.Ticker(t).info` had none, and in a daemon thread that means it can block
forever with nothing in the log — the cache warmer could park indefinitely.
Yahoo now gets a `curl_cffi` session carrying a timeout, and it **must** be
`curl_cffi` rather than `requests`: yfinance ≥1.x asserts the session type
and needs Chrome TLS impersonation, so a `requests.Session` raises
`YFDataException`, and passing nothing leaves the timeout unset entirely.
One module-level session is reused (not one per ticker) because `YfData` is
a singleton that re-binds whatever it's given — a session per ticker would
thrash the cookie/crumb it just negotiated.

## 15. reportlab fails loudly on height, silently on width

A flowable taller than the frame raises `LayoutError` and kills the whole PDF
(a single-cell `Table` can't split between rows — pass `splitInRow=1`); a
flowable *wider* than the frame is simply drawn through the margin, or off
the paper entirely, with no error. So every fixed-width flowable in
`report_pdf.py` is clamped to the measure, and preformatted text is
hard-wrapped before it's handed over. Separately, the CJK line breaker
deliberately overruns the measure by up to one em rather than start a line
with `、` or `。`, which is why the Chinese path lays out to a slightly
narrower measure and leaves a gutter for that overhang.

## 16. An `https://` link only opens a vendor's app for paths it claims

Universal links feel automatic, so the natural assumption is that pointing
at `futunn.com/en/stock/SMCI-US` opens Futubull on a phone that has it. It
does not, and nothing reports the miss — Futu's own
`/.well-known/apple-app-site-association` claims only `/qq_conn/1101195293/*`,
`/weixin_ios/*`, `/app/*` and `/deeplink/*`, so `/en/stock/*` stays a plain
web page on iOS forever, however the anchor is written.

**Read the vendor's `apple-app-site-association` / `assetlinks.json` before
assuming, and before hand-rolling a scheme.** The verified route for Futu is
`ftnn://quote/stockDetail/<stockId>/1` (from Futu's own `al:ios:url` tag),
where `stockId` is Futu's **opaque internal id, not the ticker** (SMCI is
`203319`; HK/A-share ids are 14-digit strings), and the quote page carries
dozens of unrelated `stockId`s in its "hot stocks" rails — a positional parse
links to the wrong company, which is worse than not linking at all.
`futu.py` round-trips `stockCode` + `marketLabel` back into the requested
symbol and refuses a mismatch.

## 17. A test that greps HTML for `onerror=` can pass on its own escaping

`&lt;img src=x onerror=alert(1)&gt;` is inert — a string the client
*displays*, not an element it *runs* — but it contains the needle, so a naive
substring assertion reports a vulnerability that isn't there, and (worse) an
assertion written to accommodate that noise stops catching the real thing.
`tests/test_report_email.py` strips `&lt;…&gt;` before checking, so it only
ever asserts on *live* markup. Same trap applies to `href=`, which also
appears inside escaped text.
