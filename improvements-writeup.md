# charity-donor-outreach: Improvements and Reasons

## 1. The skill instructed the agent to lie about matching gifts

**Original:** Under Emergency Appeal, "Tell the donor their gift will be matched
(even if no match is confirmed, we can sort that out later)."

**Why this is the most serious defect:** This is not a quality issue, it is a
legal one. Telling a donor their gift will be doubled when no match exists is a
material misrepresentation in a charitable solicitation. State charities
regulators and the AFP Code of Ethical Standards both treat this as
misrepresentation. It is also the kind of defect automation makes catastrophic:
one campaign run sends it to every emergency appeal donor at once, before a
human reads a single letter.

**Change:** Match language is gated twice. The campaign must be an Emergency
Appeal, and the user must confirm a real, funded match with a ratio and cap.
The match line is an optional template block that simply does not render
otherwise. The skill declines and explains if asked to include unconfirmed match
language. Match language is also now prohibited outright in Annual Fund, Capital
Campaign and Event Fundraiser letters, which the original never restricted.

**Impact:** Removes the largest legal and reputational exposure in the skill.

---

## 2. Fabrication was built into the instructions

Three separate places told the model to invent things.

**Relationship managers.** The original says "Assign a personal relationship
manager name," which means inventing a staff member and signing a $50,000 plus
donor's letter with their name. The donor then replies to a person who does not
exist. The relationship manager concept is right for Platinum. The fabrication
is not.

*Change:* The signature block is `[RELATIONSHIP_MANAGER_NAME]` /
`[TITLE], [CHARITY_NAME]` on every letter. Platinum draws the name from a
`relationship_manager` column in the donor file, or from a real staff roster the
user supplies, assigned deterministically by region and written into
`summary.csv` for staff to confirm or reassign. Other tiers use the default
signer. A Platinum donor with no sourceable manager goes to exceptions rather
than receiving a made up name.

**Gender guessed from first names.** The original says to guess a title, using
"Elizabeth is probably Ms." as the example. Addressing a major donor with the
wrong title is a memorable, avoidable insult, and the heuristic fails
systematically on the many names in the file that carry no Anglo-American gender
signal.

*Change:* Titles are used only when present in the data. Otherwise the
salutation falls back to full name for Platinum and Gold, first name for Silver
and Bronze. No inference from names, ever.

**"Make reasonable assumptions and proceed."** Applied to a missing giving
history, a reasonable assumption is a fabricated gift, printed in a letter that
also states the donor's lifetime total.

*Change:* Incomplete or malformed records go to `exceptions.csv` with a reason
and receive no letter. A general rule now forbids fabricated impact statistics,
rescue stories, program names, staff names and gift amounts.

**Impact:** Every donor specific claim in every letter traces back to the donor
file or to a campaign input the user confirmed. Data quality problems surface as
a short review list instead of silently becoming errors in the mail.

---

## 3. The ask calculation produced inconsistent numbers

**Rounding in the wrong place.** The original rounds to the nearest $50 at step
3, then applies a 10 percent loyalty uplift, a $100 volunteer add and a 1.2
emergency multiplier at steps 4 through 6. The adjustments unround the number
again. A Gold donor with a $12,000 largest gift who gave last year, volunteers,
and receives an emergency appeal lands on $4,092.

*Change:* All adjustments apply first, rounding happens once at the end, with a
$50 floor.

**Undefined behavior for two tiers.** Step 2 lists multipliers for Platinum,
Gold and Silver only, but steps 4 through 6 apply to everyone. It is unclear
whether the loyalty uplift, volunteer bonus and emergency multiplier stack onto
Bronze's flat $150 and Lapsed's flat $50.

*Change:* Bronze and Lapsed are explicitly flat, no adjustments, no multipliers.
Multiplying a $50 re-engagement ask by 1.2 defeats the purpose of a
re-engagement ask.

**The model was doing the arithmetic.** The formula sits in prose and the model
executes it per donor. Language models do not reliably run a six step formula
across fifty rows, let alone fifty thousand. The same donor can get a different
ask on two runs, which means the ask is not reproducible and therefore not
auditable.

*Change:* All arithmetic and categorization move into code. See section 5.

**Impact:** Clean, exact, identical asks on every run, checkable against the
summary sheet.

---

## 4. Fifty donor records were hardcoded into the skill file

**Original:** A 50 row table of names, giving histories, lifetime totals and
volunteer status, embedded in the instructions, with the agent told to read both
the uploaded CSV and this table.

**Why this blocks the brief directly:** The requirement is a skill that scales
to a growing donor list. A table in the prompt is the opposite. It is stale the
moment a gift is recorded, it is capped by the context window, it duplicates
donor PII into every conversation where the skill loads, and it creates a
conflict with the uploaded file that no rule resolves.

**Change:** The skill defines a CSV schema and treats the uploaded file as the
single source of truth. No donor data lives in the skill. The skill explicitly
forbids using donor data recalled from memory or from inside the skill file.
Added: required and optional columns, synonym mapping, derivation of lifetime
total, largest gift and last gift year from the gift history, and a validation
pass that reports row counts and exceptions before any letter is written.

Tier handling is also now unambiguous. The `tier` column is authoritative and is
not recomputed. Lapsed is a full tier alongside Platinum, Gold, Silver and
Bronze, with its own tone, salutation, flat ask and closing line, rather than a
recency condition that could collide with a giving band. Unknown receives Bronze
treatment, and unrecognized values go to exceptions.

**Impact:** The skill runs on the ASPCA's live data at any list size without
edits, and tier assignment matches the organization's own CRM classification.

---

## 5. Architecture: Structured Input-Transformation pattern

**Original:** One monolithic prompt does everything, reading the file, looking
up the tier, doing the math, writing the copy, filling the template and
returning HTML.

**Why this does not hold up:** Every step competes for the same attention
budget. Arithmetic accuracy and copy quality degrade together as the list grows,
and there is no failure isolation. One malformed row can derail an entire
response.

**Change:** Three stages, with a hard split between what code owns and what the
model owns.

1. **Deterministic preparation (code).** Parses the donor file, derives the
   giving figures from the gift history, reads the tier, computes the ask, sets
   the loyalty and volunteer flags, detects streaks, routes bad rows to
   exceptions. Emits one clean structured record per donor.
2. **Copy generation (model).** Receives structured records in batches, returns
   only `campaign_paragraph` and `tier_line` per donor, as structured data keyed
   by donor ID. It is told that `ask_amount`, `lifetime_total` and `tier` are
   given values to reproduce verbatim. It never sees the raw CSV and never
   performs arithmetic.
3. **Deterministic rendering (code).** Fills the template, writes one file per
   donor, emits `summary.csv`.

**Impact:** Code handles the strict math and categorization deterministically
while the model focuses purely on tone perfect copywriting, which is the one
thing only it can do. Numbers become reproducible, and therefore auditable.

---

## 6. Rendering reliability

**Original:** The model fills the template by hand. A missed placeholder means a
letter that reads "a gift of $[ASK_AMOUNT]" going to a donor.

**Change:**

- Placeholders split into a **required** set of ten, whose absence sends the
  donor to exceptions, and an **optional** set (match line, deadline, streak,
  welcome back) wrapped in `{{#NAME}} ... {{/NAME}}` blocks that remove
  themselves, including their `<p>` tags, when empty. The template is not an all
  or nothing contract, so optional elements can be added later without every
  donor record having to supply them.
- One completeness check per letter: scan for any surviving `[...]` token or
  unresolved marker.
- One consistency assertion: the ask printed in the letter equals the ask in
  that donor's `summary.csv` row.
- A failed letter goes to exceptions and the run continues. One bad record never
  blocks a batch.

**Impact:** No letter ships with a visible placeholder or an ask that disagrees
with the approval sheet, and the guarantee costs one regex per letter.

---

## 7. Outputs fundraising staff can use

**Original:** "Fill in the letter template and return it in-chat as HTML."

**Why:** A chat response is one response. You can return a handful of letters
that way, not three hundred, so the instruction quietly assumes a small list and
contradicts the scaling requirement. It also produces markup nobody can send
from. Someone has to copy it out by hand, per donor.

**Change:** Three artifacts.

1. `letters/[last_name]_[first_name].html`, one per donor
2. `summary.csv`: name, tier, lifetime total, largest gift, computed ask,
   volunteer flag, campaign type, signing relationship manager, warnings
3. `exceptions.csv`: skipped rows and failed letters, each with a reason

A sample letter is still shown in chat when someone wants to check the copy.

**Impact:** Staff get one table to approve every ask before a send, files they
can edit, attach or mail merge, and a short list of records needing a human.

---

## 8. Triggering

**Original description:** Fires on "donors, fundraising, money, emails, letters,
charity, nonprofits, campaigns, giving, volunteers, events, reports, grants,
sponsorships, or any kind of outreach or communication task."

**Why this is actively harmful:** The description is the trigger, the body is
what runs. This description is far broader than the body, which knows exactly one
procedure: turn a donor list into solicitation letters with a computed ask. So a
request for a thank-you note fires the skill and returns a solicitation. A
request for a grant proposal returns a donor letter with a re-engagement ask in
it. That is worse than no output, because it looks plausible and gets sent.

Those tasks need genuinely different logic. Acknowledgments key off a gift rather
than a donor, require IRS substantiation language for gifts of $250 or more,
and must contain no ask at all. Grant proposals key off a funder's priorities and
produce a structured document, not a letter. Impact reports are aggregate and
must cite real program data.

**Change:** The description is scoped to what the body does, generating
personalized fundraising letters from an uploaded donor list for a named campaign
type, with explicit exclusions for grant proposals, acknowledgment letters,
volunteer communications and internal reports.

**Impact:** The skill stops corrupting adjacent tasks. Those belong in separate
skills with their own logic.

---

## 9. Scaling study: what happens as the donor list grows

The brief asks for scalability, so this section sets out the study behind the
architecture rather than asserting it. The question is what actually breaks as
the file goes from 50 rows to 5,000 to 500,000, and what the cost curve looks
like at each stage.

### Where the cost actually sits

Work in this skill divides cleanly into three cost profiles:

| Work | Grows with | Cost per donor |
|---|---|---|
| Parsing, tier lookup, ask math, validation | rows | negligible, pure CPU |
| Copy generation | **distinct copy variants**, not rows | the entire model cost |
| Template rendering, CSV writing | rows | negligible, pure CPU |

The insight that drives the design: only the middle row is expensive, and it does
not have to grow with the donor count.

### Copy varies along a handful of axes, not per donor

Two Gold donors in the Midwest, both with giving streaks, both volunteers, both
receiving the same Annual Fund appeal, need the same campaign paragraph and the
same tier line. Their letters differ in name, salutation, lifetime total and ask,
and every one of those fields is filled by the deterministic renderer, not by the
model.

So the number of distinct copy blocks needed is not the donor count. It is the
number of populated combinations across:

- tier (6)
- campaign type (4, but one per run)
- streak present (2)
- volunteer (2)
- region (5, only where regional framing is wanted)

For a single campaign, that is a ceiling of roughly 6 x 2 x 2 x 5 = 120
archetypes, and in practice far fewer are populated. **That ceiling is fixed. It
does not move when the donor list grows.**

### The consequence

Model cost is bounded by archetype count, not row count. A 500 donor campaign and
a 500,000 donor campaign need approximately the same number of generation calls.
Everything that grows linearly is code, which is cheap, exact and parallelizable.
This is what makes the skill scale rather than merely tolerate a bigger file.

### Operational measures for large runs

- **Batching.** Group 20 to 30 archetypes per generation call. Small enough to
  keep copy quality high, large enough to avoid per call overhead.
- **Failure isolation.** Batches run independently. A failed batch retries alone,
  not the run.
- **Streaming and chunking.** Read the donor file in chunks rather than loading a
  large file into memory at once.
- **Checkpointing.** Write letters incrementally so an interrupted run resumes
  rather than restarting. At 50,000 letters, a restart from zero is not an
  acceptable failure mode.
- **Idempotence.** Re-running a campaign regenerates the same letters with the
  same asks, because the math is deterministic. Reruns are safe.
- **Incremental runs.** For recurring campaigns, only new or changed donor rows
  need regeneration, since archetype copy can be cached and reused.

### Regression safety as the data grows

Deterministic math also gives a testing surface the original had none of. A small
golden set of donor rows with known correct asks (one per tier, plus the edge
cases: volunteer plus streak plus emergency, a $50 floor case, a flat tier) can
be asserted against on every run. If a formula change alters an expected ask, it
fails loudly at test time rather than quietly in 40,000 letters.

### Growth path

The skill's dependency on the file format, rather than on hardcoded data, means
the natural next step is a direct CRM export or API pull replacing the manual
upload. Nothing in the skill's logic changes when that happens. That was not true
of the original, where growing the donor list meant editing the skill file.

---

## Summary of impact

| Dimension | Original | Rewritten |
|---|---|---|
| Legal exposure | Instructs unconfirmed match claims | Match gated on Emergency Appeal plus confirmation |
| Fabrication | Invents managers, titles, missing data | Every fact traced to file or confirmed input |
| Ask accuracy | Model arithmetic, rounds mid formula | Code computed, rounds once, reproducible |
| Donor data | 50 rows hardcoded in the prompt | Uploaded file is the only source |
| Tier logic | Lapsed collides with giving bands | Lapsed is a tier, `tier` column authoritative |
| Scale ceiling | One chat response | Bounded by archetypes, not rows |
| Failure mode | One bad row derails the run | Bad rows go to exceptions, run continues |
| Staff workflow | HTML pasted into chat | Letters, approval summary, exceptions list |
| Triggering | Fires on any nonprofit task | Fires on donor letter generation only |
