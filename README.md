# GTM / Microsoft Clarity Integration

A Google Tag Manager container recipe that installs [Microsoft Clarity](https://clarity.microsoft.com/)
alongside GA4, and relays selected GA4 events into Clarity as custom events using
[Markus Baersch's `clarity-events`](https://github.com/mbaersch/clarity-events) Community
Template — chained off the existing GA4 event tags via GTM tag sequencing (teardown tags), with
no duplicated triggers to maintain.

Container JSON: [`ms-clarity-gtm-integration.json`](./ms-clarity-gtm-integration.json)
([Google Drive mirror](https://drive.google.com/file/d/1GEjbuUW9lFjtrGWqUns4vpe_PRbEENDX/view?usp=drivesdk))

Companion write-up: [MS Clarity via GTM (Blogger)](https://drewspen.blogspot.com/)

> **Incidental code in this container.** This is a Clarity recipe first. Two of its tags — the
> CookieYes consent-interaction event and the session-duration milestone event — belong to their
> own complete, separately-published recipes and are included here only to illustrate a realistic
> container. For the full build-out of each, see:
> - [CCPA-Style CookieYes Consent Implementation in GTM](https://drewspen.blogspot.com/2026/07/cookieyes-consent-implementation-in-gtm.html)
> - [Session Duration Timer in GTM](https://drewspen.blogspot.com/2026/07/session-duration-timer-in-gtm.html)
>
> Everything below documents only what's relevant to the Clarity integration; the CookieYes and
> Session Duration tags are covered here just enough to explain how they feed Clarity.

---

## Why Clarity Next to GA4

- **It's free.** No paid tier, no traffic cap, no session-recording quota. Heatmaps, unlimited
  recordings, funnels, filters, and Copilot AI insights are included from project creation — the
  same "free at any scale" model as GA4 itself.
- **It drops into GTM with one official template.** `Microsoft Clarity - Official` is a
  Sandboxed JavaScript Custom Template — no inline `<script>` injected, no extra CSP hash
  required beyond what your `googletagmanager.com` / `www.clarity.ms` allow-listing already
  covers.
- **It complements GA4 rather than duplicating it.** GA4 tells you *what* happened; Clarity shows
  you *why* — heatmaps, session recordings, JS error tracking, rage/dead-click detection, and
  AI-generated session summaries.

## What's Inside the Container

| Folder | Contains |
|---|---|
| Analytics | GA4 configuration tag, shared config/event-settings variables, Browser Client ID, Browser Session ID, Root Domain, Measurement Stream routing. |
| Microsoft Clarity | The base `Microsoft Clarity` install tag, the `Microsoft Clarity Project ID` constant, and the two Clarity event tags (`Microsoft Clarity CookieYes Interaction`, `Microsoft Clarity Session Duration`). |
| CookieYes CMP *(incidental — [full recipe](https://drewspen.blogspot.com/2026/07/cookieyes-consent-implementation-in-gtm.html))* | The CookieYes consent-interaction trigger, click-element variables, and the GA4 event tag that reports consent clicks. |
| Session Duration *(incidental — [full recipe](https://drewspen.blogspot.com/2026/07/session-duration-timer-in-gtm.html))* | The `Session or User Seconds Duration` custom-template tag, its cookie/dataLayer variables, the Session Duration Event trigger, and the GA4 event tag that reports duration milestones. |
| Element Hierarchy | Click Element Parent/Grandparent Class Name variables used to build the CookieYes interaction label. |

### Tags

| Tag | Type | Fires On | Purpose |
|---|---|---|---|
| `Google Analytics Configuration` | Google Tag (`googtag`) | All Pages | Standard GA4 config tag. Teardown → `Microsoft Clarity`. |
| `Microsoft Clarity` | Custom Template (`Microsoft Clarity - Official`, `cvt_MQDKZ`) | *No trigger — fires as teardown of `Google Analytics Configuration`* | Injects the Clarity tracking script; passes `userId`/`sessionId` from the same Browser Client ID / Browser Session ID variables used by GA4. |
| `Session Duration` *(incidental)* | Custom Template (`Session or User Seconds Duration`) | Custom trigger | Writes a duration cookie / dataLayer value used to detect session-length milestones. Full logic: [Session Duration Timer in GTM](https://drewspen.blogspot.com/2026/07/session-duration-timer-in-gtm.html). |
| `Session Duration Send Event` *(incidental)* | GA4 Event (`gaawe`) | `Session Duration Event` trigger | Reports the duration milestone to GA4. Teardown → `Microsoft Clarity Session Duration`. |
| `Microsoft Clarity Session Duration` | Custom Template (`Microsoft Clarity Events`, `cvt_WV8T5`) | *No trigger — fires as teardown of `Session Duration Send Event`* | Sends `session_duration_{n}sec` as a Clarity custom event. |
| `CookieYes Interaction Send Event` *(incidental)* | GA4 Event (`gaawe`) | `CookieYes Click Interaction` trigger | Reports a CMP interaction to GA4. Teardown → `Microsoft Clarity CookieYes Interaction`. Full logic: [CookieYes Consent Implementation in GTM](https://drewspen.blogspot.com/2026/07/cookieyes-consent-implementation-in-gtm.html). |
| `Microsoft Clarity CookieYes Interaction` | Custom Template (`Microsoft Clarity Events`, `cvt_WV8T5`) | *No trigger — fires as teardown of `CookieYes Interaction Send Event`* | Sends `cmp_click` as a Clarity custom event, and sets a `cmp_interaction` custom tag on the Clarity session for later filtering. |

### Triggers

| Trigger | Type | Conditions |
|---|---|---|
| `CookieYes Click Interaction` *(incidental)* | Click | `{{CookieYes Element Interaction}}` does **not** match `^(undefined\|null\|0\|true\|false\|NaN\|)$` — i.e. fires only when a real interaction label was captured. See the [CookieYes post](https://drewspen.blogspot.com/2026/07/cookieyes-consent-implementation-in-gtm.html) for how that label is resolved. |
| `Session Duration Event` *(incidental)* | Custom Event | `{{Session Duration dataLayer}}` matches `^(6\|11)$` **AND** `{{is Session Duration Event}}` equals `true` — fires at the 6- and 11-second duration checkpoints. See the [Session Duration post](https://drewspen.blogspot.com/2026/07/session-duration-timer-in-gtm.html) for the full timer mechanics. |

## The Chaining Pattern: Teardown Tags vs. Shared Triggers

The two Clarity event tags in this container (`Microsoft Clarity CookieYes Interaction`,
`Microsoft Clarity Session Duration`) have **no firing trigger of their own**. Each is configured
as a **teardown tag** on its corresponding GA4 event tag, with *"stop teardown on failure"*
enabled:

```
GA4 Event Tag: CookieYes Interaction Send Event
  Trigger:            CookieYes Click Interaction
  Teardown (on success) → Microsoft Clarity CookieYes Interaction

GA4 Event Tag: Session Duration Send Event
  Trigger:            Session Duration Event
  Teardown (on success) → Microsoft Clarity Session Duration
```

This is a "clean up" chaining pattern: there is exactly one trigger per event, attached to the
GA4 tag. The Clarity tag rides along afterward. If the GA4 tag fails (blocked by consent, network
error, etc.), the paired Clarity event never fires either, and there is only one place to update
firing logic if the trigger conditions ever change.

**This is not the only valid approach.** You can equally give the Clarity event tag the identical
trigger already used by its paired GA4 event tag (`CookieYes Click Interaction` or
`Session Duration Event`), letting the two tags fire independently off the same condition. That
trades the ordering dependency of teardown chaining for having the same trigger referenced in two
places. Both patterns get the event into GA4 and Clarity; pick whichever your team finds easier
to maintain.

## Templates Used

| Template | Author | Repository | Notes |
|---|---|---|---|
| `Microsoft Clarity - Official` | Microsoft | [microsoft/clarity-gtm-template](https://github.com/microsoft) | Core of this recipe. |
| `Microsoft Clarity Events` | Markus Baersch | [mbaersch/clarity-events](https://github.com/mbaersch/clarity-events) | Core of this recipe. |
| `Session or User Seconds Duration` | Kent Spencer (drewspen) | — | *Incidental.* Documented in full at [Session Duration Timer in GTM](https://drewspen.blogspot.com/2026/07/session-duration-timer-in-gtm.html). |
| `CookieYes CMP` (gallery template) | cookieyeshq | [cookieyeshq/cookieyes-gtm-template](https://github.com/cookieyeshq/cookieyes-gtm-template) | *Incidental.* Documented in full at [CookieYes Consent Implementation in GTM](https://drewspen.blogspot.com/2026/07/cookieyes-consent-implementation-in-gtm.html). |
| `Get Root Domain` | Markus Baersch | [mbaersch/get-root-domain](https://github.com/mbaersch/get-root-domain) | Shared utility, carried over from earlier recipes in this series. |
| `Timestamp` | luratic | [luratic/Timestamp](https://github.com/luratic/Timestamp) | Shared utility, carried over from earlier recipes in this series. |
| `If Else If — Advanced Lookup Table` | sublimetrix | [sublimetrix/gtm-template-ifelseif](https://github.com/sublimetrix/gtm-template-ifelseif) | Shared utility, carried over from earlier recipes in this series. |

## What Clarity Adds on Top of GA4

- **Heatmaps** — click, scroll, and area maps, generated automatically per page with no
  configuration.
- **Session recordings** — DOM-based replays of real visits (not video), filterable by the
  custom events this container sends.
- **JavaScript error tracking** — automatic error logging with a direct jump to the exact
  session/timestamp where the error occurred.
- **Rage clicks, dead clicks & quick backs** — automatic UX-friction detection with no manual
  event tagging.
- **Excessive scrolling detection** — flags visitors scrolling back and forth repeatedly.
- **Copilot AI summaries** — AI-generated per-session and per-heatmap summaries.
- **Smart Funnels** — funnel analysis from Smart Events and page visits.
- **Segments** — saved filter combinations (device, region, referral source, custom tags like
  `cmp_interaction` set by this container) for repeatable analysis.
- **Official GA4 cross-linking** — Microsoft's GA4 integration adds a `clarity_session_url`
  parameter to GA4 events, linking straight to the matching Clarity recording.
- **Privacy-first defaults** — automatic masking of password/payment fields, no PII collection,
  GDPR/CCPA-aligned defaults.

## Import Instructions

1. Download [`ms-clarity-gtm-integration.json`](./ms-clarity-gtm-integration.json) from this repo,
   or the [Google Drive mirror](https://drive.google.com/file/d/1GEjbuUW9lFjtrGWqUns4vpe_PRbEENDX/view?usp=drivesdk).
2. In GTM: **Admin → Import Container**.
3. Upload the JSON file.
4. Choose **Merge** (not Overwrite) to preserve your existing container setup.
5. Update the `Microsoft Clarity Project ID` Constant variable with your real Clarity project ID
   — found in your project URL: `https://clarity.microsoft.com/projects/view/<projectId>/`.
6. Verify the `Microsoft Clarity` tag's teardown link to `Google Analytics Configuration`
   survived the merge (visible in the tag list / tag detail view).
7. Open GTM Preview mode, load a page, and confirm `Google Analytics Configuration` fires
   followed immediately by `Microsoft Clarity` in the same sequence.
8. Trigger one of the relayed events (a CookieYes consent click, or a session-duration
   checkpoint) and confirm both the GA4 event tag and its paired Clarity teardown tag fire. If
   you need the full CookieYes or Session Duration mechanics rather than just this Clarity relay,
   see the linked posts above.
9. In the Clarity dashboard, open **Recordings** and confirm `cmp_click` and
   `session_duration_*sec` appear as filterable custom events.
10. Adjust consent-check requirements (`analytics_storage`, etc.) on the Clarity tags to match
    your CMP setup — this container assumes CookieYes; see the
    [CookieYes post](https://drewspen.blogspot.com/2026/07/cookieyes-consent-implementation-in-gtm.html)
    for the consent-mode details.

## Related Recipes in This Series

- [CCPA-Style CookieYes Consent Implementation in GTM](https://drewspen.blogspot.com/2026/07/cookieyes-consent-implementation-in-gtm.html) — full CMP setup, region-based consent defaults, GPC override, and the teardown-tag pattern this recipe reuses.
- [Session Duration Timer in GTM](https://drewspen.blogspot.com/2026/07/session-duration-timer-in-gtm.html) — full sandboxed-JS timer engine, cookie/dataLayer dual output, and milestone trigger configuration.
- [GTM / GA Page Count Measurement](https://drewspen.blogspot.com/2026/06/gtm-ga-page-count-measurement.html) — the page-view counter recipe this series builds alongside.

## License

Container recipe and documentation: MIT. Third-party Custom Templates retain their own licenses
— see each linked repository above.
