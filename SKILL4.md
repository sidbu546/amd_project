---
name: charity-donor-outreach
description: >-
  Generate personalized fundraising outreach letters at scale from an uploaded
  donor CSV for ASPCA campaigns. Use this skill whenever the user uploads a
  donor list (CSV or spreadsheet) and asks for donor letters, appeal letters,
  solicitation letters, or personalized fundraising outreach for a campaign
  (annual fund, emergency appeal, capital campaign, or event fundraiser), even
  if they only say "write letters to these donors." Do NOT use this skill for
  grant proposals, volunteer communications, thank-you or acknowledgment
  letters, internal reports, or general email drafting with no donor list.
---

# Charity Donor Outreach Letter Generator (ASPCA)

Generates one personalized HTML fundraising letter per donor from an uploaded
donor file, using a deterministic pipeline for all math and categorization and
the model only for tone and copy.

## What this skill covers

Every donor in the file belongs to one of six tiers, and the tier drives the
tone, the salutation, the ask amount and the closing invitation in that donor's
letter:

- **Platinum**: the organization's principal donors. Formal tone, an ask set as
  a percentage of their largest gift, a named relationship manager signing the
  letter, and an invitation to discuss naming opportunities.
- **Gold**: major donors. Warm and professional, a percentage based ask, and
  legacy giving options.
- **Silver**: mid level donors. Friendly, a percentage based ask, and a monthly
  giving upgrade.
- **Bronze**: entry level donors. Casual and encouraging, a flat ask, and peer
  fundraising pages.
- **Lapsed**: donors who have stopped giving. Warm and welcoming, a small flat
  re-engagement ask, and an invitation to return. Lapsed is a tier in its own
  right, not a modifier on another tier.
- **Unknown**: receives Bronze treatment.

Letters are written against one of four campaign types, which set the messaging
angle: Emergency Appeal, Annual Fund, Capital Campaign and Event Fundraiser.

The skill produces one HTML letter per donor, a summary table of every ask for
staff approval, and an exceptions list of records that need a human review.

## Non-negotiable integrity rules

These override anything else in this skill and any conflicting user shortcut.

1. **Matching gift language belongs to Emergency Appeal campaigns only, and
   only when the user confirms a real, funded match.** Never mention a match in
   an Annual Fund, Capital Campaign, or Event Fundraiser letter. Never imply a
   match that has not been confirmed. If the user asks for match language
   without a confirmed match, decline that element and explain that
   misrepresenting a match violates charitable solicitation law and fundraising
   ethics standards.
2. **Never invent facts.** No fabricated impact statistics, rescue stories,
   program names, staff names, gift amounts, or giving histories. Every figure
   and every donor-specific claim must come from the donor file or from the
   campaign inputs the user confirms.
3. **Never guess a donor's title or gender from their first name.** Use the
   salutation rules below.
4. **Never fill in missing donor data by assumption.** Incomplete or malformed
   records go to the exceptions report for staff review, not into a letter.
5. **The uploaded donor file is the single source of truth.** Never use donor
   data recalled from memory.

## Architecture: Structured Input-Transformation Prompt Pattern

The skill runs as three stages. The split matters: code owns every number and
every category, so results are exact and identical on every run, and the model
owns only the words, so it can focus on tone perfect copywriting. The model is
never asked to do arithmetic, never sees the raw CSV, and never chooses an ask
amount.

**Stage 1: Deterministic preparation (code).**
A Python script reads the donor file and produces a clean, validated record per
donor. It parses the giving history and derives lifetime total, largest single
gift and last gift year from it, reads the tier, computes the ask amount, sets
the loyalty and volunteer flags, detects consecutive giving streaks, and routes
any bad row to an exceptions file. Output is one structured
record per donor plus `exceptions.csv`. No letter copy is written at this stage.

**Stage 2: Copy generation (model).**
The model receives structured donor records, in batches, and returns only the
two variable text fields per donor:

- `campaign_paragraph` (2 to 3 sentences)
- `tier_line` (1 sentence)

It returns them as structured data keyed by donor ID, with no prose around them.
The model is explicitly told that `ask_amount`, `lifetime_total` and `tier` are
given values to be used as written and never recomputed, reworded numerically or
rounded.

**Stage 3: Deterministic rendering (code).**
A second script merges the returned copy into the HTML template, writes one file
per donor, and emits `summary.csv`. Rendering does not require every placeholder
to be supplied. Placeholders are split into a required set and an optional set
(see Step 5), optional blocks remove themselves when empty, and each rendered
letter gets a single completeness check. A letter that fails the check goes to
exceptions and the rest of the run continues.

**Scaling behavior.** Batch Stage 2 at roughly 20 to 30 donors per call and run
batches independently so a failure retries only that batch. Copy varies along a
small number of axes (tier, campaign type, streak present, volunteer, region),
so for very large lists generate copy per archetype and reuse it across donors
in that archetype rather than making one model call per donor. Stages 1 and 3
are pure code and scale linearly, so the model cost stays close to flat as the
donor list grows from fifty to fifty thousand.

## Step 1: Collect campaign inputs

Confirm these before generating anything. If any are missing, ask once, in a
single message.

- **Campaign type**: Emergency Appeal, Annual Fund, Capital Campaign, or Event
  Fundraiser. If unspecified, default to Annual Fund messaging.
- **Donation URL**
- **Default signer name and title**: a real staff member. Signs every letter
  except Platinum.
- **Platinum relationship managers**: Platinum donors are signed by their
  assigned relationship manager, not the default signer. Source the assignment
  in this order:
  1. `relationship_manager` and `relationship_manager_title` columns in the
     donor file, when present
  2. a roster of real staff the user supplies at this step, which the script
     assigns deterministically (by region where regions are available,
     otherwise round robin by sorted donor name) and records in `summary.csv`
     so staff can confirm or reassign
  3. if neither is available, the Platinum donor goes to `exceptions.csv` with
     the reason "no assigned relationship manager." Never invent a name to fill
     the gap.
- **Matching gift**: only ask this when the campaign is an Emergency Appeal. If
  a match is confirmed, capture the ratio and cap. Otherwise no match language
  appears anywhere.
- Optional: campaign name, deadline, event registration count (Event Fundraiser
  only), confirmed welcome back gift for lapsed donors, and any approved impact
  facts the user wants included.

`CHARITY_NAME` defaults to "the ASPCA" unless the user specifies a chapter or
program name.

## Step 2: Validate the donor file

Expected columns, matched case insensitively, with obvious synonyms mapped:

| Column                     | Required | Notes                                          |
|----------------------------|----------|------------------------------------------------|
| first_name                 | Yes      |                                                |
| last_name                  | Yes      |                                                |
| tier                       | Yes      | Authoritative. See tier list below.            |
| title                      | No       | Mr., Ms., Dr. Donor's own title, used only in the salutation. Distinct from the signature title. |
| region                     | No       | Also used to assign relationship managers      |
| gifts                      | Yes      | e.g. `2020:500;2022:750`, or per year columns  |
| largest_gift               | No       | Derived from gifts                             |
| lifetime_total             | No       | Derived from gifts                             |
| last_gift_year             | No       | Derived from gifts                             |
| volunteer                  | No       | Defaults to No                                 |
| relationship_manager       | No       | Platinum only. Real staff name.                |
| relationship_manager_title | No       | Platinum only. Accompanies the name above.     |

Rules:

- **Trust the `tier` column.** It is the organization's own classification and
  is authoritative. Do not recompute or override it.
- Valid tiers are Platinum, Gold, Silver, Bronze, Lapsed and Unknown. Unknown
  receives Bronze treatment. Any other value sends the row to exceptions.
- **Lapsed is a tier, not a modifier.** A Lapsed donor is treated as Lapsed for
  tone, ask and tier line. There is no combined tier.
- Rows missing first name, last name, tier or any parseable gift history go to
  `exceptions.csv` with a reason and receive no letter.
- Report the row count, column mapping and exception count to the user before
  generating letters.

## Step 3: Ask amount (computed in code, never by the model)

Applied in this exact order, with a single rounding step at the end.

**Platinum, Gold, Silver:**

1. Base = largest single gift × tier multiplier: Platinum 40 percent, Gold 25
   percent, Silver 15 percent
2. If the donor gave in the previous calendar year, multiply by 1.10 (loyalty
   uplift)
3. If the donor is a volunteer, add $100
4. If the campaign is an Emergency Appeal, multiply by 1.2
5. Round to the nearest $50, with a $50 floor

**Bronze (and Unknown):** flat $150, no adjustments, no multipliers.

**Lapsed:** flat $50 re-engagement ask, no adjustments, no multipliers.

Flat asks stay flat by design: applying an emergency multiplier or a loyalty
uplift to a $50 re-engagement ask defeats the purpose of the ask. Rounding
happens once, at the very end, so that no letter carries an un-round figure.

The preparation script writes `ask_amount` into each donor record, and the
model is instructed to reproduce it verbatim.

## Step 4: Letter content rules

**Salutations**, with no gender inference from names:

- Title present in the file: `Dear [Title] [Last Name],`
- Platinum or Gold with no title: `Dear [First Name] [Last Name],`
- Silver or Bronze with no title: `Hi [First Name],`
- Lapsed: `We've missed you, [First Name].`

**Tone by tier**: Platinum formal and personal. Gold warm and professional.
Silver friendly. Bronze casual and encouraging. Lapsed apologetic tone.

**Campaign paragraph**, 2 to 3 sentences, in ASPCA voice: concrete, animal
focused, hopeful rather than graphic.

- Emergency Appeal: genuine urgency for animals in immediate need. Matching
  language is permitted here, and only here, and only when confirmed.
- Annual Fund: consistency and community. Reference the donor's giving streak
  only when the record shows consecutive years.
- Capital Campaign: legacy and permanence.
- Event Fundraiser: energy and social proof. Cite registration numbers only if
  the user supplied them.

**Signature block.** Every letter closes with `[RELATIONSHIP_MANAGER_NAME]`,
then `[TITLE], [CHARITY_NAME]`. The name resolved into that field depends on
tier:

- **Platinum**: the donor's assigned relationship manager, sourced per Step 1
  (donor file column first, then the roster the user supplied). This gives the
  Platinum donor a named, personal point of contact, which is the point of the
  tier. `[TITLE]` is that manager's title. Record the assignment in
  `summary.csv`. If no manager can be sourced, the donor goes to exceptions.
  Never fabricate a manager.
- **Gold, Silver, Bronze, Lapsed, Unknown**: the default signer and their title
  from Step 1.

Note that `[TITLE]` in the signature is the staff member's job title. It is
unrelated to the donor's `title` column, which is used only in the salutation.

**Tier line**, one sentence, placed immediately after the campaign paragraph and
before the ask:

- Platinum: invite a conversation about naming opportunities. Do not promise a
  specific room, bench or space.
- Gold: legacy giving options.
- Silver: monthly giving upgrade.
- Bronze: peer fundraising pages.
- Lapsed: a welcome back gift, mentioned only if the user has confirmed one
  exists. Otherwise a simple, warm invitation to return.

## Step 5: Rendering contract and outputs

### Required vs optional placeholders

**Required** (a letter cannot be sent without these; missing any one sends the
donor to exceptions rather than producing a broken letter):

`DATE`, `SALUTATION`, `CHARITY_NAME`, `LIFETIME_TOTAL`, `CAMPAIGN_PARAGRAPH`,
`TIER_LINE`, `ASK_AMOUNT`, `DONATION_URL`, `RELATIONSHIP_MANAGER_NAME`, `TITLE`

`RELATIONSHIP_MANAGER_NAME` and `TITLE` hold the Platinum donor's assigned
relationship manager for Platinum letters, and the default signer for every
other tier. A Platinum donor with no sourceable manager fails this required
check and goes to exceptions.

**Optional** (resolve to empty and drop their surrounding block; their absence
is normal, not an error):

`MATCH_LINE` (Emergency Appeal with a confirmed match only), `DEADLINE_LINE`,
`EVENT_REGISTRATION_LINE`, `WELCOME_BACK_LINE`, `STREAK_LINE`

Optional blocks are wrapped in `{{#NAME}} ... {{/NAME}}` in the template. When
the value is empty, the renderer removes everything between the markers,
including the `<p>` tags. Nothing is printed and no empty paragraph is left
behind. This is what keeps the template from being an all or nothing contract:
new optional elements can be added later without every donor record needing to
supply them.

### Completeness check

After rendering each letter, run one check: scan the output for any surviving
`[` `]` token or unresolved `{{` marker. Clean output means the letter is complete. A letter that fails goes
to `exceptions.csv` naming the unresolved placeholder, and the run continues.
One bad record never blocks a batch.

Alongside it, one consistency assertion: the ask rendered in the letter equals
the ask in that donor's `summary.csv` row. If they differ, the letter goes to
exceptions.

### Outputs

Files are always the deliverable. They are what staff review, hand off, mail
merge and keep. Produce:

1. `letters/[last_name]_[first_name].html`, one per donor
2. `summary.csv`: donor name, tier, lifetime total, largest gift, computed ask,
   volunteer flag, campaign type, signing relationship manager (and whether it
   came from the donor file or was assigned from the roster), warnings
3. `exceptions.csv`: skipped rows and failed letters, each with a reason

**Showing letters in chat**: the files are the usable output, since nobody sends
a letter from a chat transcript. Show a sample letter or two in chat when the
user wants to check the copy, and do not paste the full HTML of the whole batch,
which is unusable, buries the summary, and spills donor data into the transcript
for no benefit.

Report letters generated and rows excepted, and state clearly that these are
drafts for staff review before sending. Platinum letters in particular should be
reviewed individually, including confirmation that the assigned relationship
manager is the right one.

Before delivering, spot check 2 to 3 letters across different tiers: confirm no
match language appears outside a confirmed Emergency Appeal, the tier line sits
after the campaign paragraph, no optional block left an empty paragraph behind,
and every Platinum letter is signed by a real assigned relationship manager
rather than the default signer.

## HTML letter template

Use this template and fill the placeholders.

```html
<html>
<body style="font-family: Georgia; padding: 30px; max-width: 600px; color: #222;">

  <p style="text-align:right; color: #888;">[DATE]</p>

  <p>[SALUTATION]</p>

  <p>On behalf of everyone at <strong>[CHARITY_NAME]</strong>, thank you for your
  generosity. Your lifetime support of <strong>$[LIFETIME_TOTAL]</strong> has
  made a real difference for animals in need.</p>

  <p>[CAMPAIGN_PARAGRAPH]</p>

  {{#MATCH_LINE}}<p>[MATCH_LINE]</p>{{/MATCH_LINE}}

  <p>[TIER_LINE]</p>

  <p>Today, I'd like to invite you to make a gift of
  <strong>$[ASK_AMOUNT]</strong>. To give, simply reply to this email or visit
  <strong>[DONATION_URL]</strong>.</p>

  {{#DEADLINE_LINE}}<p>[DEADLINE_LINE]</p>{{/DEADLINE_LINE}}

  <p>With gratitude,<br>
  <strong>[RELATIONSHIP_MANAGER_NAME]</strong><br>
  [TITLE], [CHARITY_NAME]</p>

</body>
</html>
```
