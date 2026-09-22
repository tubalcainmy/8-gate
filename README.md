[README.md](https://github.com/user-attachments/files/32511109/README.md)
# 8-gate master template

The master copy of the Tubalcain 8-Gate qualification system. Every
installer instance is a duplicate of this file with `SITE_CONFIG`
filled in. Nothing else changes between clients.

One self-contained `index.html`. No build step, no framework, no
external JavaScript.

---

# Template mode

While `installerId` reads `REPLACE-ME` the page runs in template mode:

- a banner sits at the top saying nothing is sent or tracked
- **no lead is posted** to the webhook
- **no Meta or GA4 event fires**, including the page view

The page itself works completely, so you can click through every gate
and see the result page while you are changing something. Setting a real
`installerId` turns all three off at once, which means a duplicated file
cannot leave the guard on by accident, and the master cannot put junk
leads into the live Make.com scenario.

The payload that would have been sent is printed to the browser console,
so you can still check the shape of a lead without sending one.

---

# Duplicating this for an installer

1. Create the new repository and copy `index.html`, `README.md` and
   `assets/` into it.
2. Replace `assets/rep.jpg` with the installer rep's photo. Square crop,
   face centred, 400x400 or larger. A `.png` is fine, the page falls
   back to whichever extension exists.
3. Work down `SITE_CONFIG` and change everything marked **CHANGE**.
   Leave everything marked **KEEP** exactly as it is: those are the same
   on every instance.
4. **Set a unique `installerId`.** It routes the lead in the shared
   Make.com scenario and namespaces the lockout key, so two installers
   sharing an id would share a lockout. Setting it also switches
   template mode off.
5. Add the installer's own city to `installer.outOfAreaTerms` if it is
   not already there, so a neighbouring city typed into the field is
   caught.
6. Add the matching route in the Make.com scenario.
7. Run the whole assessment locally before you enable Pages, and check
   the banner is gone and a lead actually posts.

## What is a placeholder

Everything in `[SQUARE BRACKETS]` renders on the page as written, so
anything left unfilled is obvious on screen rather than silently wrong.
Search the file for `[` before you ship.

| Placeholder | Becomes |
| --- | --- |
| `[INSTALLER NAME]` | The installer's business name |
| `[CITY, STATE]` / `[CITY]` | Long and short forms of the territory |
| `[Area One]` to `[Area Four]` | The four named areas at Gate 2 |
| `[REP FIRST NAME]`, `[Role], [Installer Name]` | The person asking the questions |
| `[Package 1]` to `[Package 7]` | The installer's own price list, ascending |
| `[Inclusion one]`, `[Inclusion two]` | What every package includes |
| `2340000000000` | The installer's WhatsApp, format `234...` |
| `REPLACE-ME` | The unique `installerId` |

## What to keep

`unqualifiedWhatsapp`, `webhookUrl`, `metaPixelId`, `ga4MeasurementId`
and `guideUrl` are yours and identical on every instance. Copy them
across untouched. The campaign pixel stays yours per SLA clause 12;
never swap in an installer's pixel.

## Retry leads: delivered, flagged, not billed

The default. A visitor who was disqualified, used their one correction,
and then passed is delivered to the installer like any other lead, but
flagged and never billed.

- the result CTA points at the installer's line
- the payload carries `deliverable: true` and `billable: false`
- `priorDQ` records which gate rejected them, the answer they gave, and
  when. `priorDQAnswerChanged` says whether the answer to that gate
  actually changed, which separates a mis-click from an edit
- the Meta `Lead` conversion does **not** fire, so the campaign is never
  optimised toward people who game the gates. A custom
  `RetryLeadFlagged` event fires instead

To stop sending them entirely instead, set `deliverable` back to
`!isRetry` in the DELIVERY ROUTING block.

### The Make.com change this needs

```
Route 1  (first)
  Filter: billable = false
  Actions: append to the billing sheet flagged, notify yourself,
           and still forward to the installer

Route 2..N  (the per-installer routes)
  Filter: billable = true  AND  installer_id = <the installer>
```

A flagged lead must never land in the count you invoice against, and
never count toward the 30-in-30 guarantee. Leads sent before this change
carry no `billable` field, so treat a missing `billable` as `true`.

---

# How to work on this file

**Test before you push.** Open `index.html` in your browser by
double-clicking it and run the whole assessment. It works from the local
file, nothing needs a server. GitHub Pages caches hard, so a change that
looks broken live is often just a stale cache. Hard refresh with
`Ctrl+Shift+R`, or `Cmd+Shift+R` on a Mac.

**Search, do not scroll.** Line numbers drift as you edit. Use `Ctrl+F`
on the quoted code instead, and copy the old text exactly, spaces
included.

**The console commands you will need.** Open the console with `F12`.

```js
// Clear the lockout so you can run the assessment again.
// Swap in whatever installerId you set.
localStorage.removeItem('solar_gate_lock_v1__REPLACE-ME');

// Clear the shuffled option order so you get a fresh order
sessionStorage.clear();

// Fake a past disqualification, to test the return-visit screen.
// Reload after running this.
writeLock({ kind:'dq', dq:'dq-timeline', gate:'q4',
            value:'funds-building', at:new Date().toISOString(), attempts:1 });
```

Open the page with **`?brandcheck=1`** for a contrast report on the
current palette.

---

## Hero copy

`SITE_CONFIG.hero` sets the top of the page so it can mirror the ad
creative word for word. The ad earned the click, so the first screen
should look like the same offer from the same person.

```js
hero: {
  badge: "",                       // small pill above the photo; empty hides it
  headlineLead: "Spending ₦50,000+",
  headlineHighlight: "On Power Every Month?",   // accent colour, own line
  headlineTail: "I Can Help.",
  sub: "Answer 9 questions about your home, ...",
  chips: ["Pay Small Small", "7-Year Installation Guarantee", "..."],
},
```

`rep.introLine` is the second line of the speech bubble beside the photo,
under "Hi, I'm [rep.name]." Match it to the bubble on the ad.

On phones the hero is compacted so the photo, the headline and the start
of Question 1 fit on the first screen, and the page no longer scrolls on
arrival. Before this, it jumped down to Question 1 and pushed the rep's
face, and on smaller screens the headline, off the top.

## Number confirm step

After a valid form, the page reads the WhatsApp number back ("Is this your
WhatsApp number? 0803 123 4567") before anything is sent. Edit reopens the
field; changing the number hides the box again.

Numbers that look like samples are refused with their own message: a run
of one digit, a counting sequence such as 08012345678, or a copied example.
The placeholder no longer shows a sample number people could copy.

This catches typos and lazy fakes. It does not prove the number is real;
only an SMS code would do that.

## Returning visitors

A visitor who already submitted sees "Welcome back, [name]" with their
package, price, size, payback and monthly saving, and a WhatsApp button
using the same message as their original result. The summary is kept only
in their own browser, inside the lockout record, and expires with it.

New GA4 events: `number_confirm_shown`, `number_confirmed`,
`number_edit_clicked`. The gap between shown and confirmed is how many
people back out at the read-back.

## Telling installers apart in GA4

Every event from the page carries two extra parameters, `installer_id`
(from `SITE_CONFIG.installerId`) and `installer_name` (the business name).
They are set on the GA4 config, so GA4's own automatic events
(`page_view`, `session_start`, `scroll`, `click`) carry them too, not only
the qualifier's custom events.

**Register them once or GA4 will not show them.** Admin, then Custom
definitions, then Create custom dimension. Make two, both with scope
**Event**: `installer_id` and `installer_name`. Registration is not
retroactive: it only covers events from the day you create it.

After that, add `installer_id` as a secondary dimension on any report,
or as a comparison, and each campaign reads separately. Keep
`installerId` unique per instance; two installers sharing one would be
reported as one.

## Telling installers apart in Meta

Every instance fires the same pixel, so every pixel event carries the same
two parameters, `installer_id` and `installer_name`. That covers
`PageView`, `Lead`, `UnqualifiedContact` and `RetryLeadFlagged`. `Lead`
now also sends `currency: NGN` alongside its value, which Meta needs to
report the value correctly.

To split results by installer in Meta, create a custom conversion in
Events Manager: pick the `Lead` event and add the rule
`installer_id` equals `greenerafrik-ibadan`. One per installer. The same
parameter works in custom audience rules, so you can build an audience of
one installer's visitors.

# Branding

`SITE_CONFIG.brand`, around line 101:

```js
brand: {
  navy:        "#001B3D",  // darkest surface: hero, result page
  navy2:       "#062A52",  // cards on the result page
  primary:     "#0081CC",  // buttons, chips, selected states
  primaryDark: "#00629B",  // hover, and text on pale washes
  wash:        "#EEF3F8",  // page background
  blueWash:    "#E4F0FA",  // pale accent panels
},
```

**How to pick the six values from one brand colour.** Say the installer's
brand green is `#2FA36B`.

| Value | How to get it | Example |
| --- | --- | --- |
| `primary` | The brand colour itself | `#2FA36B` |
| `primaryDark` | The same colour about 25% darker | `#1F7A4E` |
| `navy` | Very dark, same hue. Near black with a tint of the brand | `#14281F` |
| `navy2` | A step lighter than `navy` | `#1D3A2C` |
| `wash` | Near white with the faintest tint | `#EFF5F1` |
| `blueWash` | A pale but visible tint | `#E3F2E9` |

Two more values are worked out for you and you normally leave them alone:

- `--primary-light`, the accent on the dark result page, is `primary`
  lightened until it is readable against `navy`. A dark brand colour
  will not disappear.
- `--on-dark-muted`, the supporting text on dark surfaces, is a light
  tint of `navy2`, so it sits in the brand's own hue.

Override them only if the brand has exact values, by uncommenting:

```js
    // primaryLight: "#7FC4F0",
    // onDarkMuted:  "#C4D3E3",
```

Button label colour is decided at load. Hand it a pale brand colour like
a yellow and the label flips to near black by itself.

**Do not brand the result colours.** Green means passed, amber means one
item to confirm on site, red means rejected. They are hardcoded on
purpose. An installer whose brand is red would otherwise make a passing
result look like an error.

**How to test branding.** Open the file, then in the console:

```js
SITE_CONFIG.brand = { navy:"#14281F", navy2:"#1D3A2C", primary:"#2FA36B",
                      primaryDark:"#1F7A4E", wash:"#EFF5F1", blueWash:"#E3F2E9" };
applyBrand();
```

The page repaints instantly. Try it before you commit the values, and
check the hero, a question card, and the dark result page.

---

## Contrast is corrected for you, but not composition

Two values are fixed at load so a palette cannot make text unreadable:

- **the button label** flips between white and near-black by whichever
  actually reads on your `primary`, measured, not guessed
- **`primaryDark`** is darkened until it clears 4.5:1 as text, because it
  is used for the correction link and the pale badge as well as for
  button hover

Open the page with **`?brandcheck=1`** and the console prints a contrast
table for every pairing, naming the value to change. Do that before you
ship a palette.

What it cannot fix is composition. **`navy` and `navy2` should share a
hue.** Changing one and leaving the other is the most common mistake and
puts cards of one colour family on a background of another. `wash` and
`blueWash` should carry a faint tint of the same hue for the same reason.

`navy` also has to be genuinely dark, because it carries white text
across the hero and the whole result page. If `?brandcheck=1` says white
on navy is under 4.5:1, darken it.

## The browser tab icon

```js
faviconPath: "",
```

Left empty, `rep.photo` is used, so a client instance gets a branded tab
without a second thing to configure. The extension does not have to be
exact: if the file is a `.png` where the config says `.jpg`, both the rep
photo and the favicon fall back to the other extension by themselves.

A photo is a poor favicon at 16px and costs every visitor the full image
download, which on Nigerian mobile data is worth avoiding. For a live
page, save a small square crop as `assets/favicon.png` at 180x180 and
point `faviconPath` at it.

## The report number

```js
reportPrefix: "",
```

Appears on the result page as `#ABC-20260918-4567`. Left empty, the
business name's initials are used. Set it per instance so a duplicated
page does not hand out another installer's report numbers.

# SITE_CONFIG field reference

Only `SITE_CONFIG` changes between instances. Everything else stays
identical, which is the whole point of the master.

| Field | What it does |
| --- | --- |
| `installer.businessName` | Appears throughout the page and in the consent line |
| `installer.whatsapp` | Where qualified buyers land. Format `234...`, no plus, no leading zero |
| `installer.unqualifiedWhatsapp` | Where every disqualified visitor goes. **Keep this as your own line** |
| `installer.cityLabel` | Long form, e.g. `Enugu, Enugu State` |
| `installer.cityShortName` | Short form, used in option labels, e.g. `Enugu` |
| `installer.locations` | The four named areas at Gate 2 |
| `installer.outOfAreaTerms` | Cities that disqualify if typed into the city field |
| `rep` | Name, title and photo of the person who appears to ask the questions |
| `tier1.packages` / `tier2.packages` | The installer's own price list |
| `warrantyMonths` | Badge on the result card |
| `financingLine` | Badge shown to part-payment and financing buyers |
| `packageInclusions` | What every package includes, shown as badges |
| `brand` | The colour scheme, see Branding |
| `installerId` | Routes the lead in the shared Make.com scenario. Must be unique |

---

# Other things you will want to edit

### The named service areas

`installer.locations`, around line 26. Four is the right number. More and
the list stops being scannable on a phone, which is what makes people
click without reading. Anywhere else in the city is still covered by the
"Elsewhere in {city}" option, which reveals a typed field.

### The minimum monthly spend

Two places, and they must agree.

```js
  if (ans.spend < 30000) { trackDisqualification('dq-spend','q1',ans.val); showDQ('dq-spend'); return; }
```

That is the real gate, around line 2129. If you change the number, also
check the wording on the `dq-spend` screen so it does not contradict it.
`SPEND_MIN` and `SPEND_MAX` around line 2034 are only input sanity
checks, catching a fat-finger extra zero. They are not the gate.

### Packages

`tier1.packages` and `tier2.packages`. Keep them in **ascending price
order**, because the list is rendered in the order you write it and a
buyer scanning prices out of sequence reads the page as broken.

`belowMinimum: true` marks a package under the installer's real entry
system. `recoverable: true` sends that visitor to the instalment recovery
step instead of rejecting them. Every option renders identically on
screen, with no tell, so a rejected visitor cannot work out the rule and
retry with a different pick.

`tier2.packages` needs `kva` as a number, because the appliance load is
matched against it.

### Warranty, inclusions and financing line

```js
  warrantyMonths: 18,
  financingLine: "Go Solar. Pay Small Small.",
  packageInclusions: [
    "Free periodic system check",
    "24/7 after-sales support",
  ],
```

Only put things in `packageInclusions` the installer will actually
honour. This is the page the buyer screenshots and quotes back later.
Whatever you put here should match the wording in the Quotation Template
and the Authority Close Kit, or a buyer who reads all three will ask
which one is true.

### The lockout window

```js
const LOCK_DAYS = 30;
```

How long a disqualification is remembered. Thirty days outlives a single
ad flight without permanently burning someone whose circumstances change.

### How many corrections a returning visitor gets

```js
  if ((lock.attempts || 1) < 2) {
```

`< 2` gives one correction. `< 1` gives none, a hard block. `< 3` gives
two, which is too generous.

### Which gates shuffle their options

```js
  shuffleQuestionOptions('q3');   // whose permission
  shuffleQuestionOptions('q7');   // progress with other installers
  shuffleQuestionOptions('q8');   // payment method
  shuffleQuestionOptions('q5r');  // instalment recovery
  shuffleQuestionOptions('q4');
  shuffleQuestionOptions('q6', ['none']);
```

Delete a line to stop that gate shuffling. The second argument pins a
`data-val` to the bottom.

Two are deliberately absent and should stay absent. **The package ladder
is not shuffled**, for the price-order reason above. **The home check
pins "None of these" last**, because a fast clicker landing on it first
would flag their own roof for no reason and turn a clean lead into an
amber one.

---

# Reference

## Gate 2, location

Four named areas shuffled, then **Elsewhere in {city}**, then **Somewhere
else**.

Selecting *Elsewhere in {city}* reveals a text field. The entry is
validated for format and checked against `installer.outOfAreaTerms`, so
someone typing a different city into a field labelled with the
installer's city is routed out rather than delivered. `territory` in the
lead reads as `Independence Layout, Enugu`, with
the raw entry kept at `answers.city.typedArea`.

*Somewhere else* disqualifies.

## Option order

Shuffled per session, deterministically against a seed in
`sessionStorage`. That means going back through the gates shows the same
order rather than reshuffling under the visitor's hands, a reload inside
the same session keeps the order, a new session gets a new one, and
session resume still works because selection is bound to `data-val`
rather than to position.

The point is that no position is ever the safe click, and a rejected
visitor who reloads learns nothing from the layout.

Stop referring to options by number anywhere, including in the close
kit. "Option 3" no longer means anything. Use the wording or the
`data-val`.

## Gate lockout

A record in `localStorage` under
`solar_gate_lock_v1__<installerId>`.

**First return:** the visitor lands back on the rejection screen they
saw, with one "answered something by mistake?" link. Using it reopens the
flow and marks the attempt.

**Second return:** the same screen, no link.

**After `LOCK_DAYS`:** the record expires and they start clean.

**A visitor who already submitted** lands on an "already completed"
screen pointing at the installer's WhatsApp, rather than back at question
one. This stops a second lead for the same person, which is the usual
pay-per-lead billing dispute.

The key must stay namespaced by `installerId`. Every instance shares one
origin on GitHub Pages, so an unnamespaced key would lock a visitor out
of every other installer's page after one rejection here.

**What it stops, and what it does not.** It survives reloads, tab closes
and browser restarts, which covers the ordinary retry and most traffic
arriving through the in-app browser on a Facebook click. A private
window, cleared site data, another browser or another phone all defeat
it. Nothing client side can change that: the page posts nothing for a
disqualified visitor, so there is no server record to check against, and
the WhatsApp number that would identify a person is only collected at
Question 9, which a rejected visitor never reaches. That is the argument
for flagging as well as blocking.

## Lead payload, the fields that matter

| Field | What it holds |
| --- | --- |
| `installer_id` | Routes the lead |
| `active_tier` | Which product delivered this buyer |
| `firstName`, `whatsapp`, `location` | As typed |
| `territory`, `city` | Area inside the service zone |
| `contactConsent` + text + timestamp | Your consent record |
| `budgetRange` | The package selected or matched. **Never call this a confirmed budget** |
| `packageRange` | System size band |
| `matchedPackage` | Tier 2 only. Name, kVA, price, what it powers, undersized flag |
| `applianceLoad` | Tier 2 only. Appliances selected, total watts, required kVA |
| `packageBracketTag` | `instalment-recovered`, or empty |
| `timeline` | When funds become available, not install preference |
| `paymentMethod` | `full`, `part`, `financing` or `unsure` |
| `competitorStatus` | How far along they are with anyone else |
| `homeFlags` | Roof, shading or electrical items reported |
| `leadStatus` | `green`, or `yellow` when a home flag is present |
| `roi` | Payback years, month one saving, twenty year total, system cost, monthly spend |
| `deliverable` | False means do not send to the installer |
| `billable` | False means do not charge for it |
| `priorDQ` | Which gate rejected them, the answer, and when |
| `priorDQAnswerChanged` | Whether that answer changed, separating a mis-click from an edit |

## GA4 and Meta events

| Event | Fires when |
| --- | --- |
| `qualifier_started` | Page loaded, Question 1 visible |
| `qualifier_resumed` | Session restored mid-assessment |
| `qualifier_disqualified` | Rejected at a gate |
| `qualifier_completed` / `generate_lead` | Passed and submitted |
| `unqualified_whatsapp_clicked` | A rejected visitor opened your nurture line |
| `dq_lockout_shown` | A rejected visitor returned. `value` is the attempt number |
| `dq_correction_used` | They used their one correction |
| `retry_lead_flagged` | They passed on the retry. Named `retry_lead_suppressed` until you make Edit C |
| `retry_whatsapp_clicked` | A flagged retry lead opened WhatsApp |
| `duplicate_submission_blocked` | A visitor who already submitted returned |

Watch `dq_lockout_shown` against `qualifier_disqualified` in week one.
That ratio is how many people are trying the gates a second time, and it
is the number a hard block would have hidden from you.

## Tracking ownership

The campaign pixel stays Tubalcain's per SLA clause 12. Do not swap in a
the installer pixel. Disqualified WhatsApp clicks fire their own events so
they can never be counted as delivered leads.
