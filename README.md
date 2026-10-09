# Migrating from Mailchimp to Loops: field notes

**How we tested, built, overhauled and went live** moving a membership organisation's
email automation from Mailchimp to [Loops](https://loops.so) in 2026 — audience import,
lifecycle journeys, landing pages and forms, and a live cutover of the new-member
onboarding sequence without double-emailing anyone.

Almost everything here was learned by hitting it. Where a claim came from an experiment,
the experiment is described, because several of these contradict the vendor documentation
or aren't in it at all.

**Scope.** This is about **migration and build mechanics** — data modelling, automation
wiring, verification, and cutover sequencing. The parts where a mistake emails real people.

It is deliberately **not** about copywriting, send timing or campaign optimisation. Those
are worth getting right, but they're the same problem in any tool and they're not what
makes a migration go wrong.

**Not affiliated with Loops.** No warranty; verify against your own account. Behaviour may
have changed since late 2026.

---

## Start here

If you're planning a migration rather than debugging one, read in this order:

1. **[The scaffolding](#the-scaffolding-how-we-worked-not-just-what-we-built)** — set this up
   *before* you build anything. It's the part we'd do first if we did this again, and the
   part we actually did last.
2. **[Events fire automations, properties are data](#the-one-thing-to-understand-first)** —
   get this wrong and an import emails your whole list.
3. **[A success response is not evidence](#-a-success-response-is-not-evidence-of-effect)** —
   the verification discipline that caught five of our six cutover-day bugs.
4. **[Data modelling](#data-modelling-one-fact-one-field)** — decide your property register
   before you write templates against it.
5. **[The cutover](#the-cutover-moving-a-live-sequence-without-double-emailing)** and the
   **[pre-flight checklist](#pre-flight-checklist)** when you're ready to go live.

Everything else is reference for when you hit the specific thing.

**You don't have to migrate everything at once.** We moved supporter onboarding first, then
member onboarding three months later, and left prospect nurture on the old tool entirely.
Each lane cut over independently, which meant each one was a small, reversible change
instead of one large irreversible one.

---

## The one thing to understand first

### Events fire automations. Properties are data. Never mix them.

Loops lets you trigger a workflow on a **contact property changing**. Don't.

Creating a contact with a property already set **counts as a property change**. So a CSV
or API import of 2,000 contacts who have `customerTier: pro` set will enrol all 2,000 into
any workflow triggered on that property.

Pausing the workflow does not save you: trigger matches **queue for ~24 hours and fire on
resume**.

Use **event triggers** for everything that should fire deliberately, and treat properties
as segmentation and record-keeping only. Enrolling someone then becomes two steps:

```
1. PUT  /contacts/update   → set the properties (the data)
2. POST /events/send       → fire the event (the trigger)
```

Events never fire on import, which makes an event-triggered workflow safe to build and
activate at any time — even before your audience is loaded.

**Corollary for import planning:** classify every workflow as import-safe or not, and keep
the unsafe ones in Draft until the import is done and verified. A Draft workflow cannot
fire, which is your primary safety mechanism.

**Also a trigger: adding someone to a mailing list.** If a workflow uses an add-to-list
trigger and your import assigns lists, that import is thousands of add-to-list events.
Correct order, no exceptions:

```
assign lists during the import  →  verify  →  only then activate list-triggered workflows
```

---

## 🔴 A success response is not evidence of effect

The single most expensive lesson. An API call returning `200` tells you the request
arrived, not that it did anything. We hit this **three separate times**, each silent:

| What we sent | What came back | What actually happened |
|---|---|---|
| `events/send` with `contactProperties` | success | event fired; **properties not persisted** |
| An audience-filter node keyed `filter` | success | condition **silently discarded** — the guard admitted everyone |
| A conditional section nested inside another | success | **the entire parent block rendered as nothing** |

Each looked correct in the API response, and in two cases looked correct in the editor too.

**So:**

- **Read back after every write.** Our enrolment code sets properties, re-reads the
  contact, and refuses to fire the event unless the property it depends on reads back
  correct. Failing loudly leaves a recoverable contact; firing early leaves an
  unrecoverable one (see `reEligible` below).
- **Prove behaviour by real send.** Not by status code, and not by preview.
- **An unknown custom property is silently dropped.** Send `{"myNewProp": true}` for a
  property that doesn't exist and you get `{"success": true}`, a created contact, and no
  property. **Confirm it exists before sending it.**

### Previews cannot test conditionals

Loops' email preview resolves **no merge values at all** — it prints something to the
effect of "merge value not available in preview", so every conditional block renders
hidden regardless of your data. A preview that looks empty tells you nothing.

Transactional sends reject `{contact.*}` merge tags at validation, so that's not a
workaround either.

**The only thing that tests a conditional is a real send through an active workflow, to a
contact whose properties are set.** Batch every case you want to check into one email so a
single send answers everything — see the testing section.

---

## Conditionals: the operator truth table

Established by real sends. One of these contradicts the docs.

| Goal | Use | Don't |
|---|---|---|
| boolean is true | `isTrue`, or `ifOperation="true"` | — |
| boolean is **false** | `equal` + `ifValue="false"` | ⚠ `ifOperation="false"` matches **nothing, ever** |
| "not true" | — | ⚠ `not_equal "true"` matches false **AND unset** |
| string equality | `equal` + `ifValue` | — |

Full filter-operator enum (camelCase — `is_true` and `not_empty` are rejected):

```
any · contains · notContains · equals · notEquals · greaterThan · lessThan
isTrue · isFalse · empty · notEmpty · dateEmpty · dateNotEmpty · after · before · between
```

Operators taking no value (`isTrue`, `isFalse`, `empty`, `notEmpty`) must **omit** the
`value` key. A raw JSON boolean in `value` is rejected outright.

### 🔴 The null trap

**An unset boolean is absent, not false.** In our account only ~580 of 2,455 contacts had a
given boolean set at all. So "is not a founder" written as `founder isFalse` would have
excluded nearly the whole audience.

Correct form:

```json
{"match": "any", "conditions": [
  {"type": "property", "key": "founder", "operator": "isFalse"},
  {"type": "property", "key": "founder", "operator": "empty"}
]}
```

This bites hardest on anything that *nags*. A "you haven't paid yet" block keyed off
`not_equal "true"` shows up for every paying customer whose property hasn't been
backfilled. Only ever show that kind of content on an **explicit false**.

Design your states deliberately — we settled on three:

| value | meaning | behaviour |
|---|---|---|
| `true` | known paid | full content |
| `false` | **known** unpaid | show the payment prompt |
| unset | unknown | show neither — and treat it as a bug to fix, because it renders a hollow email |

### 🔴 Nested conditional sections render NOTHING

A conditional `<Section>` inside another conditional `<Section>` saves cleanly and then
**silently drops the entire parent** at send time.

This shipped a live bug for us: a payment block wrapped two inner per-tier sections to pick
the right checkout link, so unpaid members received a welcome with **no access links and no
payment prompt** — the worst of both.

**Keep conditional sections flat, always.**

**Consequence: Loops conditionals cannot express AND.** If you need two conditions
combined, either:

1. **Encode both facts in one property** and use a single flat `equal` — e.g. instead of
   "unpaid AND tier is pro", write `paymentPrompt = "pro"` and clear it when they pay; or
2. **Chain two audience-filter nodes in series** inside the workflow — sequential filters
   are an AND, with no nesting involved.

Option 1 introduces the one thing worth calling out separately:

### A property that must be *cleared* on state change

If you encode "needs X AND is tier Y" into a single property, that property has to be
**emptied** when the state changes — otherwise a paying customer keeps seeing the payment
prompt forever.

Writing an empty string genuinely clears a string property (verified). But treat any
clear-on-change property as a liability: it's the one field where forgetting a write
produces wrong, user-visible content rather than just missing data. Derive it from the same
source value as the boolean it shadows, so the two can't disagree.

---

## Journey mechanics

### `reEligible: false` means a guard exit is permanent

Put a filter node right after your trigger so a mis-sent event exits harmlessly rather
than onboarding the wrong person. Good practice — with one sharp edge.

With `reEligible: false` (one-time per contact), **a contact who fails that guard can never
enter the workflow again** without manual intervention. Including a contact who failed only
because the property the guard checks **hadn't landed yet**.

This is why the read-back before firing the event is load-bearing rather than
belt-and-braces. The asymmetry:

- **Not enrolled** → recoverable. Fix the data, fire again.
- **Enrolled and guard-exited** → unrecoverable without hand repair.

So: never fire early. Confirm the guard's input is actually on the contact first.

### Guard on what you *mean*, positively

Write guards as positive enumeration (`tier equals pro` OR `tier equals basic`), not as
exclusion (`tier notEquals friend`). Because of the null trap, an exclusion lets a
contact with **no** tier sail straight through.

And check whether your guard needs a *status* condition too, not just a type condition. Our
guard checked membership tier but not membership status, so a churned member still carrying
their old tier would have passed it.

### Conditionals evaluate at send time — which is a feature

For a journey with emails days apart, conditionals read the contact's properties **at the
moment each email sends**, not at enrolment. So someone who upgrades on day 5 automatically
gets upgraded content from day 7 onward. Nothing to maintain.

This only works if you branch on `{contact.*}`, **not** on event properties — event
properties freeze at enrolment. It's the main reason to keep lifecycle state in contact
properties and pass nothing meaningful in the event payload.

### Editing a live workflow

Email content is **locked while a workflow is `Sending`**. The cycle is Pause → push →
Resume, which takes seconds and is safe with event triggers because nothing enrols while
paused unless you send an event.

**Activation and pausing are UI-only** in our experience — there was no CLI/API path to
start or stop a workflow, even though the workflow *graph* was fully mutable
programmatically. Plan for a human click at cutover.

**Make all your content edits before you wire up enrolment.** Otherwise a real signup can
arrive while the workflow is paused mid-edit, and the behaviour there is undocumented.

---

## Testing a journey without emailing real people

What actually works:

1. **Use a fresh plus-alias per test** (`you+test1@gmail.com`). `reEligible: false` means a
   reused address cannot re-enter — a second run against the same contact proves nothing,
   and silently looks like success.
2. **Batch every case into one email.** A single send with all candidate operators side by
   side answers a whole truth table at once, instead of burning a contact per question.
3. **Test the import-safety claim directly:** create one contact *with* the trigger
   properties pre-set and confirm it does **not** enrol, alongside one that you enrol via a
   real event.
4. **Delete test contacts afterwards** — they consume plan capacity, and a surviving
   contact makes the next test meaningless (point 1).
5. **Mailing-list membership is not required** to receive workflow email. So test contacts
   can be created with no list at all, which conveniently also means they can't receive
   real campaigns.

### Two gotchas if you test with plus-aliases

- **Some third-party APIs reject `+` in an address used as a key.** Google Workspace's
  Directory API won't accept a plus-aliased address as a group `memberKey` (`%2B` →
  404 on add, 400 "missing required field" on delete). If your onboarding touches other
  systems, expect plus-alias-only failures that will never happen for a real user. Make
  your tooling *say that*, rather than printing a red error on every clean run.
- **Automate the cleanup, and make a no-op cleanup look clean.** A cleanup script that
  always prints errors trains you to stop reading them — which is how a real failure gets
  missed.

---

## The cutover: moving a live sequence without double-emailing

This was the part with real risk: an onboarding sequence that had been running in Mailchimp
for years, and a replacement in Loops, with live signups arriving throughout.

### Stop the trigger, don't switch the old journey off

The instinct is to turn the Mailchimp journey off. Don't — **switching a Customer Journey
off can strand people mid-sequence**, halfway through a multi-week flow.

Instead: **stop applying the tag that triggers it.** New entrants stop immediately;
everyone already in the sequence finishes naturally. Then let it drain and retire it later.

Two things make this safe:

1. **Confirm what actually triggers the old journey — don't assume.** Mailchimp Customer
   Journeys are widely described as having no API. That's half true: the **read** endpoints
   exist and will tell you the trigger —
   `GET /customer-journeys/journeys/{id}` and `/{id}/steps`, where a tag trigger shows as
   `step_type: "trigger-tag_added"` with the tag name. It's **writes** and automation-email
   *content* that aren't available. Check this before betting your cutover on withholding a
   tag, in case the journey fires on something else entirely.
2. **Count who's mid-flight.** Multiply your sequence length by your signup rate. Ours was
   a 33-day sequence with two people still in it, so it fully drained in under a month.

(Separately useful: automation email bodies 404 on
`/automations/{id}/emails/{id}/content`, but the same content *is* retrievable via
`/campaigns/{id}/content`. Worth knowing if you need to port existing copy.)

### Make the handover atomic

Put both sides of the switch behind **one flag**:

- start writing the new system's properties and firing its event
- stop applying the old system's trigger tag

If those are two separate deploys, you get a window where **both** systems welcome someone,
or one where **neither** does. One flag, one commit, no gap. And rollback is reverting that
line — the old journey is still live, so it simply resumes.

Gate it per-contact against an allowlist of migrated types, not just globally, so anything
the new system doesn't handle keeps its old welcome rather than silently getting none.

### 🔴 The hard part: how do you know who already got the old sequence?

Your new journey must never re-welcome an existing customer. So you need a marker that
reliably says "this person predates the cutover" — and **choose it by testing real
contacts, not by reasoning.**

We assumed the obvious candidates would work. Both failed:

| contact | `memberStatus` | `activeMember` | `memberTier` |
|---|---|---|---|
| existing member A | **empty** | true | set |
| existing member B | set | **empty** | set |

Gating on either of the first two would have re-welcomed a real customer. The only field
present on *every* pre-existing member was the one our bulk import had derived from the
source-of-truth member list.

**Lessons that generalise:**

- **Prefer presence over value.** We gate on the field *existing*, not on what it says.
  That makes the gate immune to the stale values you'll inevitably find in imported data —
  and you will find them.
- **Include churned/inactive people in the marker.** A returning customer must also be
  excluded, or they get a second welcome years later.
- **Make a skip a result, not an error.** Declining to re-welcome someone is correct
  behaviour and must not fail the run — but log it, so a *wrongly* skipped person is
  visible. A wrongly skipped person is one message away from a fix; an unwanted duplicate
  welcome is not fixable at all.
- **Whichever field you choose becomes load-bearing.** Document it loudly. Writing it to
  someone who isn't a customer silently excludes them from ever being welcomed, with no
  error anywhere. Tell every other system that touches your contact data.

### Backfill before you go live

Any property your new journey branches on must be set on **existing** contacts before they
could possibly enrol — otherwise the unset case renders. In our case an unset payment flag
produced an email with neither the access links nor the explanation: not harmful, but
clearly broken.

Watch the scope of the backfill query. Ours selected "active subscription" from a billing
roster, which quietly excluded the **non-paying partner of a household plan** — one
subscription, two people, and only one of them looks active in billing data. Check your
filter against the edge cases in your own data model rather than trusting its logic.

### Order of operations

1. Build and activate the new journey (event-triggered, so it's safe to activate early)
2. Finish all content edits — before enrolment exists
3. Wire enrolment, **tested with test contacts only**
4. Backfill the properties the journey branches on
5. Flip the one flag: new system on, old trigger tag off
6. Watch the first few real signups
7. Let the old journey drain, then retire it

Reversing 4 and 5 sends hollow emails. Doing 5 before 3 means new customers get **nothing**
— a gap, not a migration.

---

## Overhauling the welcome journey while you're at it

A migration is the cheapest moment to fix the sequence itself, because you're rebuilding it
anyway. What we actually changed:

### One journey for all tiers, not one per tier

We nearly built separate onboarding journeys per membership level. Then we read the old
sequence properly: across **six emails there was exactly one tier-specific block.** Separate
journeys would have been five-sixths duplication, with five-sixths of the future edits.

One journey, one trigger event, and tier differences handled by conditional sections. If you
find yourself with more than a couple of conditionals, revisit — but count first.

### What our journeys actually look like

Real shapes, in case it's useful to see what came out the other end. Three live journeys,
each with a different trigger type:

| journey | trigger | shape |
|---|---|---|
| **Newsletter welcome** | add-to-list (one-time) | gate filter → **branch** → two tracks: **7 emails / 45 days** and **5 emails / 24 days** |
| **New member onboarding** | event, one-time | guard filter → **6 emails / 33 days**, one conditional block inside one email |
| **Supporter onboarding** | event, one-time | **2 emails / 7 days** |

The newsletter welcome is the interesting one, because it shows the two-track pattern:

```
AddToListTrigger  (one-time per contact)
 └─ AudienceFilter            ← gate: does this person belong here at all?
     └─ BranchNode
         ├─ AudienceFilter → 7 emails, 5/5/7/7/7/14-day gaps     (general audience)
         └─ AudienceFilter → 5 emails, 5/5/7/7-day gaps          (founders)
```

A branch with a filter on each arm is how you get mutually exclusive tracks out of a tool
whose conditionals can't express AND. It's also the shape to reach for **instead of**
duplicating a whole journey per audience — one trigger, one place to change the entry
conditions, and the arms only hold what genuinely differs.

Three deliberate choices in there:

- **The gate filter sits before the branch**, so "should this person be here at all" is
  answered once rather than repeated on every arm.
- **Cadence front-loads.** Both tracks start at 5-day gaps and widen to 7 and 14. The early
  emails do the work; the later ones are maintenance.
- **Different lengths per track.** The founder track is shorter because it had less to say,
  not because it was cut short. Resist making arms symmetrical for tidiness.

### Audit the old sequence for defects before porting it

Don't port faithfully; port deliberately. Ours had a conditional showing "your form isn't
filled out, here's the link" where the link came from a merge field **populated on only ~24
contacts**. Its default was the site homepage. So for nearly everyone, that reminder pointed
at the homepage for years.

Deriving the link from the member's tier instead of storing it per-contact fixed it
permanently and removed a field we'd otherwise have had to migrate.

### Delete copy that a process change made obsolete

Our process had changed since the sequence was written: the intake form became mandatory
*before* onboarding. So every piece of form-chasing language across two emails was now
addressing something that couldn't happen. All of it came out.

Read each email asking "is this still true?", not just "does this still render?"

### Add gating that didn't exist before

The old sequence linked to the community space and shared boards unconditionally, because
when it was written everyone reaching it had paid. We added a known-paid condition to those
access links, and an alternative block for the not-yet-paid case.

That's the kind of thing worth adding during a rebuild — the old tool could have done it,
but nobody was going to touch a working journey just for it.

### Match the old tool's visual rhythm deliberately

Loops rendered paragraphs noticeably tighter than Mailchimp did. Rather than eyeball each
email, we set an explicit spacing standard — a fixed bottom padding for body paragraphs,
section images, buttons and dividers — and applied it to every email including the one
already live.

Worth doing early. We retrofitted a live three-email sequence that had **24 paragraphs and
zero** of the standard padding.

### Small content decisions that came from our data

- **Your imported data constrains your templates.** Our first-name capture was patchy, so
  a greeting line would have read badly for a meaningful slice of the audience — we dropped
  it rather than lean on a fallback. Audit the fields your templates want to merge *before*
  writing templates around them.
- **Let the tool append its own unsubscribe footer** — don't hand-build one.

---

## The rest of the overhaul: landing pages, forms, lead magnets

Changing ESP breaks every signup form on your site. This was over half the total work, and
it's the part most likely to be forgotten until links start dying.

### Audit every form, including the invisible ones

We scanned all published pages **and** all page-builder templates via the authenticated
WordPress REST API. The obvious pages were the easy part; what the audit caught:

- a form in the **site-wide footer template** — the highest-volume one
- a form in a **popup template**
- forms in **blog-post inline templates**
- two templates that were `publish`ed but had **no display conditions**, so they rendered
  nowhere — never a live leak, but they'd have looked like one forever

Search for the **bare host** (`mailchi.mp`), not a full URL. One link was plain visible text
with no scheme and no anchor tag, and a sitemap crawl missed it entirely.

### 🔴 The public form endpoint silently drops private mailing lists

The same class of silent failure as everywhere else, and the most expensive one here.

Loops' unauthenticated form endpoint takes `email` (required), `firstName`, `lastName`,
`source`, `userGroup`, and a hidden comma-separated `mailingLists` of list IDs, returning
`{"success": true}`.

Posting a **public** list ID works. Posting a **private** list ID *also returns
`{"success": true}`* — and creates the contact with `mailingLists: {}`. No error, no warning.

Proven rather than inferred: the identical list ID assigned fine through the authenticated
API, so the ID was valid and the public endpoint was the thing dropping it. Had we deployed
on the response alone, our **highest-traffic** landing page would have collected signups onto
no list at all.

Custom boolean contact properties *do* work through the public endpoint — we verified a flag
landing correctly — so the failure is specific to private lists.

**Always verify list membership on a test submission, not just that the contact appeared.**

### Wire forms defensively inside a page builder

Our first version found the form with `document.currentScript.previousElementSibling`. That's
fragile inside a builder's HTML widget: the builder wraps widget markup in its own containers
and can re-run injected scripts. The failure mode is the nasty kind — the form silently stays
un-wired and falls back to a raw POST, which dumps the visitor on a **raw JSON response page**.

What we ended up with:

- a **stable unique id** per form, looked up with `getElementById`
- a `data-wired` attribute as a **double-bind guard** so a re-run script can't attach twice
- CSS scoped under a page-specific class, so builder and theme styles can't collide

Also: **copy-pasting a widget to move it leaves a duplicate.** Two copies share one element
id, so only the first gets wired and the second does the raw POST. Check the count after
moving anything.

### Page-builder caching will lie to you

Three separate traps, each of which cost a cycle:

1. **The link may not live where you think.** Posts can carry *both* a classic content field
   and a builder copy, and **the builder's copy is what renders**. A string replace against
   the classic field reports success and changes nothing.
2. **URLs are JSON-escaped in builder data** (`https:\/\/…`), so a naive replace of
   `https://…` matches nothing — and reports success.
3. **A REST write to builder data does not invalidate the builder's rendered-markup cache**,
   and a `?cb=timestamp` cache-buster does **not** bypass it. Clear the builder's own cache
   and the host cache, then verify with **no query string at all**.

And if your host runs an optimiser, it may move inline `<script>` into a combined external
file — so the script vanishes from page source **while still running**. Don't conclude it was
stripped; check for a wired flag on the element instead.

**Back up before editing.** We saved each original page's builder data into a draft copy
first; restoring is copying that meta back. Also prefer an **in-place swap** of the form
element over drag-and-delete in the UI — an atomic replace can't leave the duplicate.

### Don't replace a form that does more than subscribe

One of ours was a membership application: it emailed a real person **and** collected a long
free-text answer with nowhere sensible to live on a contact record.

We kept the original form's email and save-to-database actions and **only added** the new
ESP, via a small listener that posts on the form's success event — fire-and-forget, so if it
fails the application still goes through. Replacing it outright would have traded a working
process for a tidier one.

### Prove nothing was lost in the old system

Before cutting over, reconcile the old form's submission log against the new ESP. Ours
exposed `actions_count` vs `actions_succeeded_count` per submission — a direct signal for a
failed integration — and 44 unique submissions over nine months reconciled at 100%, which
closed the question properly instead of assuming it.

### Lead magnets: the magnet doesn't have to be a file

Our "worksheet" lead magnet was assumed to need a PDF and a delivery automation. It didn't.
The magnet was **already a page on the site** — the gate collects the email, then redirects
to it. No file hosting, no delivery automation, no attachment deliverability problem.

Two things that followed:

- **Don't maintain two pages for one magnet.** We built a replacement landing page, then
  deleted it: the existing page was already indexed and had the traffic. Point external
  links at the one that exists.
- 🔴 **Don't advertise a magnet the new system can't deliver yet.** One rebuilt page promised
  the worksheet while its delivery automation was still an empty draft. We softened the copy
  and **held the link swap** rather than shipping a promise nothing fulfilled.

### Attribution: one URL parameter, free forever

Every form carries a **default `source` per page**, overridable by a `?src=` URL parameter.
So the same landing page linked from a link-in-bio tool, a video description and a social
profile reports three distinct sources, with no tracking setup.

Decide the vocabulary up front and write it down — ours sprawled to a dozen values across
workstreams before anyone wrote the list.

### Small things worth copying

- **`noindex` the new landing pages** and keep them out of nav, so they don't compete with
  your real pages in search.
- **You may not need the page builder at all.** Plain pages with raw HTML in a single HTML
  block worked fine for us (the account allowed unfiltered HTML), and a full-width page
  template made the hero bleed edge-to-edge without touching the builder.
- **Old hosted landing-page URLs die when you cancel.** Inventory every external placement —
  link-in-bio, video descriptions, social profiles, email signatures, printed QR codes,
  shared documents — and swap them **before** cancelling. Ours included an email signature
  that shipped on every one-to-one email, and a shared document's reply template.
- **Search tools miss URLs inside hyperlink text.** Full-text search in at least one document
  platform doesn't index the href, only the visible words — so searching for the dying URL
  returns nothing and you have to open each candidate and inspect its links.

---

## Campaigns (one-shot) vs journeys (repeatable)

- **A campaign sends once.** To send the same content again you duplicate it. **Never point
  a finished campaign at a test recipient** — publishing spends it, and the real send then
  needs a rebuild.
- There was **no publish/send path in the API or CLI** for us — scheduling could be set
  programmatically, but the publish itself was a UI click. Plan for that.
- `--schedule-at <ISO8601>` is the better pattern than "send now": set the time, publish
  once, it fires then.
- **Lock a test send to one recipient with a saved segment**, never an ad-hoc guess.
- **No native campaign A/B.** Experiments are workflow-only, so if you relied on
  campaign-level A/B in your old tool, that capability doesn't port — plan a manual split
  (a random group property → two segments → two duplicate campaigns) or drop it.

---

## Imports

- **Suppressed contacts (unsubscribes, bounces) must import as unsubscribed.** Never
  resurrect them. Export your suppression list separately and import it with the
  subscription flag off.
- **Preference-centre unsubscribes can be irreversible by you.** If a contact opts out of a
  list through the preference centre, your team may not be able to resubscribe them. So get
  list membership right **before** a large import — a wrong default that people opt out of
  cannot be undone.
- **An export is stale on arrival.** If your old system is still acquiring contacts, export
  close to cutover or plan a delta sync. Ours was taking ~26 new contacts a month, and a
  three-month-old export would have **resurrected 16 people who had since been cleaned** —
  not a theoretical risk, a counted one.
- **Imports may not assign mailing lists.** Ours didn't, and the result was a campaign
  targeting a list reaching almost nobody while everything looked fine. Verify list
  membership on a sample after importing, not just contact count.
- There was **no contacts-count endpoint**; per-email lookup only. Confirm audience sizes in
  the UI before publishing anything.

---

## The scaffolding: how we worked, not just what we built

A migration like this runs for months, across several parallel workstreams, against systems
that have exactly one live state. Most of what went wrong was a *coordination* failure
rather than a technical one — a decision made in one conversation and unknown in the next, a
stale note treated as fact, a "quick check" that answered confidently and wrongly.

The scaffolding below is what we ended up with. It's tool-agnostic and it's the part we'd
set up **first** if we did this again.

### A written constitution, with receipts

One file of standing rules, loaded automatically into every working session, plus a set of
longer protocol documents it summarises. The structure that worked:

```
constitution/     the always-loaded rules — short, and it stays short
protocols/        the long form: one file per concern, with the incidents attached
brand/            voice and style as a loadable standard
skills/           repeatable task playbooks
```

The constitution is **symlinked** into the AI tool's config directory rather than copied, so
there is exactly one source of truth and editing it is a normal commit with history.

**Every rule is anchored to the specific incident that produced it.** Not "verify API
responses" but "a 200 meant nothing three times, here's each one." A bare rule gets
re-litigated and eventually bent; a rule with its failure attached survives, because the
cost of ignoring it is right there.

### Two tiers, with an explicit promotion bar

Learnings sort into exactly two places, and the sorting rule matters more than it sounds:

| tier | holds | example |
|---|---|---|
| **project memory** | this system's state, vendor quirks, who's who, what's half-built | "the public form endpoint drops private lists" |
| **global protocols** | how to work, regardless of tool | "a success response is not evidence of effect" |

**The bar for promotion: would this change how I work on an unrelated project with different
tools?** If no, it stays in project memory.

This is the bit that keeps the constitution useful. What kills a document like this is
entries that are really project notes wearing a principle's clothing — they accumulate, the
file gets long, and people stop reading it. We deliberately left tool-specific findings
(operator names, API quirks, which endpoint 404s) in project memory and promoted only four
things from an entire migration.

### An autonomy ladder

Write down, once, what may act without asking and what must be drafted for review. Ours
splits roughly:

- **act freely** — diagnostics, reads, tests, refactors, structure, internal docs
- **draft, don't send** — anything leaving the organisation, anything in a person's name
- **never without explicit confirmation** — deletes with no undo, provisioning or revoking
  access, anything touching another system's identifiers

Two refinements that came out of real incidents:

- **Confirmation is per-action and doesn't carry forward.** A yes to one cleanup is not a yes
  to the next one. Terse approvals ("great", "yes", "go ahead") are not green lights for
  destructive or outward-facing work — restate the scope and confirm.
- **A source-integrity gate on anything unattended.** Before an automated job publishes,
  confirm every input actually resolved. If a source failed or came back empty, the run
  **downgrades to a draft that names the missing source**. An empty result is not evidence
  that nothing happened.

### Hard caps, enforced in code

"Prefer", "generally", "lean toward" get bent every time. Use **HARD CAP / NO EXCEPTIONS**,
and enforce the same limit in the code, not only in the instructions.

Specifically: **never write a discretion carve-out.** "Unless it's a standout case" is the
phrase that gets exploited.

A worked example from this migration — our test-enrolment script refuses any address without
a `+` alias, with no override flag at all. Since a mis-fire permanently burns someone's
one-time journey enrolment, we wanted that mistake to be *impossible*, not discouraged.

### Decision records: date, verbatim words, and the triggering incident

Every non-obvious decision gets written down with **when**, **what was actually said**, and
**what went wrong that forced the decision**. Convert "next week" and "by Friday" to absolute
dates, because you will read this in six months.

Record **corrections as corrections** — what was believed, who corrected it, when. And record
**verified negatives**: "there is no such setting, I checked X, Y and Z" saves the next
session an hour and stops the same dead end being re-explored.

### Assume a parallel session

If more than one conversation can touch the same project — and on a migration this size,
several will — then:

- re-query state before asserting it; "I checked" has a timestamp
- read the log before writing a shared file, and never commit everything blindly
- a decision that lives only in a chat transcript **is not recorded**

We lost real time to two sessions holding different beliefs about what was deployed. A
shared, committed handoff document between workstreams fixed it; chat did not.

### Voice as a versioned standard, not a per-task instruction

Ours lives as one canonical file describing how the organisation sounds, what gets a draft
rejected, and which register each channel takes — with channel-specific playbooks (newsletter,
social, community announcement) that handle structure and defer to it for voice.

It's loaded *before* drafting anything outward-facing, and it gets updated when a draft gets
rejected for a reason that isn't yet written down. That last part is what makes it improve
rather than ossify: a rejection is a bug report against the standard.

The migration-relevant point: the same discipline that keeps your contact-property names from
drifting keeps your copy from drifting. Both are a register that someone has to own.

### What this actually bought us

On cutover day, six real bugs surfaced. Five were invisible to unit and plan-level tests, and
all six were caught and fixed before they reached a member — because the protocols said to
execute the real path, read back every write, and distrust a clean-looking answer from a
throwaway check.

The scaffolding isn't overhead on a project like this. It's the thing that makes a months-long
migration across parallel workstreams converge instead of drift.

---

## Pre-flight checklist

Before you let anything fire at a real person:

- [ ] Every workflow classified **import-safe** (event-triggered) or **not** (property /
      signup / add-to-list triggered). Unsafe ones in Draft until the import is done.
- [ ] Every custom property your automations reference **exists** — an unknown one is
      silently dropped.
- [ ] Every conditional tested by **real send**, not preview. Nested sections eliminated.
- [ ] Every "show a nag" conditional keyed on **explicit false**, never "not true".
- [ ] A **guard node** after each trigger, written as positive enumeration, including a
      status condition if relevant.
- [ ] Enrolment does **property write → read back → event**, and refuses to fire if the
      read-back fails.
- [ ] A tested marker that identifies **pre-existing** contacts, chosen by checking real
      data, including churned ones.
- [ ] Properties the journey branches on **backfilled** on existing contacts, with the
      filter checked against your edge cases.
- [ ] Old system's trigger **withheld in code**, not just switched off in a UI — and its
      actual trigger confirmed, not assumed.
- [ ] Both sides of the handover behind **one flag**, with a one-line rollback.
- [ ] Test-contact cleanup automated, including deleting the contact.

---

## What I'd do differently

**Write the property register first.** Half the drift came from it being implicit. One
agreed list of field names and valid values, with examples, would have prevented most of
the rework.

**Assume every write needs verification from day one.** We added read-backs reactively,
after a silent failure had already cost someone their welcome email. Building it in from
the start costs one extra request and eliminates a whole failure class.

**Test against live systems earlier.** Of six real bugs found on cutover day, **five were
invisible to unit and plan-level tests** — the plans were correct and the API calls were
what failed. One of them (a multi-seat plan's second person never being created in the
email system, so every write to them 404'd) had been broken for the life of the feature and
was found in minutes by running the real path once.

**Model multi-seat plans as people, not products, immediately.** Retrofitting that was the
single largest change.

**Don't trust your own quick verification scripts.** The throwaway script written to answer
"is this clean?" gets its answer *reported as fact*. Two of ours had silent fallbacks — a
bare `except`, a conditional that quietly measured the wrong thing — and produced confident
false negatives. Prefer the real tooling, and never write a check that can fail silently
into a clean-looking answer.

---

## Contributions

Findings from other migrations are welcome — especially anything that contradicts the
above, or that has changed since late 2026. Please keep contributions sanitised: no
customer data, no account identifiers.
