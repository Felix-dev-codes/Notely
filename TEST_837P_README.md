# 837P Test Files — What To Replace Before Sending

Four files, all produced by Bloom Note itself rather than written by hand, so
each is exactly what the app generates. Every identifier in them is a
placeholder.

| File | Contents | Claims | Total |
|---|---|---|---|
| `TEST_837P_SAMPLE.txt` | One speech session, 60 min | 1 | $116.66 |
| `TEST_837P_SI_5x60.txt` | Special Instruction / ABA, one authorised week at 5×60 | 5 | $556.75 |
| `TEST_837P_SI_10x60.txt` | Same case at 10×60 | 10 | $1,113.50 |
| `TEST_837P_SI_20x60.txt` | Same case at 20×60 | 20 | $2,227.00 |

Send **one** of them. The SI files are the same case at three authorisation
levels — pick whichever matches what you want to test. Do not hand-trim claims out of
the larger file: `SE01` counts segments and would no longer match, which fails
the 999 before anything else is read.

**Do not send any of them as they stand.** EI-Hub checks NPIs against the NPPES registry
and the billing provider NPI against its own records. Placeholder numbers fail
on that alone, and the response would tell you nothing about the rest of the
file.

Two ways to use them:

1. **Preferred** — enter your real details in the app and generate a fresh file.
   That exercises the whole path and is what you will do in production anyway.
2. Replace the values below by hand if you would rather not put real data in the
   app yet.

---

## Values to replace

| Placeholder | Replace with | Appears in |
|---|---|---|
| `0000000001` | Your billing provider NPI, 10 digits | ISA06, GS02, NM1\*41, NM1\*85, NM1\*82 |
| `000000000` | Your Tax ID, 9 digits, no dash | REF\*EI |
| `PROVLAST` / `PROVFIRST` | Rendering provider surname / given name | NM1\*41, NM1\*85, NM1\*82, PER |
| `1 REPLACE STREET` | Billing provider street | N3 |
| `7185550000` | Billing contact phone, digits only | PER |
| `CHILDLAST` / `CHILDFIRST` | Child surname / given name | NM1\*IL |
| `REPLACEEI` | Child's EI-Hub reference number | NM1\*IL, REF\*EA |
| `20230401` | Child date of birth, YYYYMMDD | DMG |
| `REPLACEAUTH` | Service authorisation number | REF\*G1 |
| `DOCLAST` / `DOCFIRST` | Ordering physician surname / given name | NM1\*DN |
| `0000000002` | Ordering physician NPI, 10 digits | NM1\*DN |
| `F802` (speech) / `F840` (SI) | ICD-10 for this child, no decimal point | HI |
| `20260720` (speech) / 6-10 July 2026 (SI) | Actual date of service, YYYYMMDD | one DTP per claim |
| `TEST0001` / `TESTSI5` / `TESTSI10` / `TESTSI01` | Your invoice number, unique to your provider ever | BHT, CLM, REF\*6R |

If you are billing as an **agency** rather than an individual provider, ISA06
and GS02 take your **Tax ID** instead of the NPI. The app's Profile has a
"Billing As" setting that handles this.

Leave the amounts, the codes and their modifiers, the unit counts, `12:B:1`
and the segment structure alone unless the session genuinely differs — those
come from the rate schedule and the companion guide.

Changing the **number of claims** means regenerating rather than editing: every
claim you add or remove changes the segment count that `SE01` has to match.

---

## What the speech file represents

One in-person speech therapy session, one hour, delivered in the child's home.

| | |
|---|---|
| Code | 92507, speech-language treatment |
| Modifier | GN, under a speech plan of care |
| Units | 1 — 92507 is untimed, one unit per session |
| Amount | $116.66, NYC extended visit rate for OT/PT/ST |
| Place of service | 12, home |
| Visit type | CV1, regular |
| Frequency | 1, original claim |

## What the SI / ABA files represent

One authorised week of Special Instruction, Monday 6 July 2026 to Friday the
10th, in the child's home.

The notation is sessions **per week**. The mandate is the weekly total; how it
is spread is the clinician's call, and the same authorisation can legitimately
be met over three, four or five days:

| Pattern | Hours/week | Over 5 days | Over 4 days | Over 3 days |
|---|---|---|---|---|
| 5×60 | 5 | 1 hr/day | 1–2 hr/day | 2 hr/day |
| 10×60 | 10 | 2 hr/day | 2–3 hr/day | 3–4 hr/day |
| 20×60 | 20 | 4 hr/day | 5 hr/day | 6–7 hr/day |

These files use the five-day spread, which is one valid arrangement of several.
The claim count and the weekly total are the same whichever is chosen — only the
dates move. Sessions sit at 09:00, 10:00, 13:00 and 14:00, an intensive ABA day
with a midday break.

| | |
|---|---|
| Code | 97155, adaptive behavior treatment with protocol modification |
| Modifier | none — the discipline modifiers GN/GO/GP cover speech, OT and PT, and SI is none of those |
| Units | 4 per session — 97155 is defined in 15-minute increments |
| Amount | $111.35 per session, NYC Extended SI visit rate |
| Diagnosis | F84.0, autism spectrum disorder |
| Place of service | 12, home |
| Visit type | CV1, regular |
| Frequency | 1, original claim |

Each service line reads `SV1*HC:97155*111.35*UN*4***1`. Note that the dollar
amount is the **visit** rate, not four times a unit price — the units come from
the CPT definition and the money comes from the rate schedule, and the two are
set independently in NY EI.

---

## Checked against the companion guide

| | |
|---|---|
| ISA segment length | 106 characters |
| ISA16 / segment terminator | `:` at position 105, `~` at 106 |
| ISA14 | 1, requesting the return 999 |
| ISA15 | P — the guide accepts production data only |
| SE01 | matches the actual count from ST through SE in all four files — 27, 71, 126 and 236 |
| CLM08 / CLM09 / CLM10 | N / Y / P as required |
| Receiver and payer | New York City, county code 70 |
| Referring provider | Loop 2310A, the ordering physician rather than the therapist |
| Times | 24-hour clock, HHMM |

---

## Open questions worth raising with PCG alongside the file

1. **Modifiers.** The file sends `GN` alone. Section 5.19 lists GN, GO, GP, 96
   and 97 as commonly used and permits any combination, and every SV1 example in
   the guide carries a single modifier with 96 in none of them. Is the
   discipline modifier alone correct for an in-person habilitative service, or
   should 96 accompany it?

2. **Several sessions on one date — the big one for intensive ABA.** At 20×60
   the child receives four separate one-hour sessions a day. The rate schedule
   has only Basic and Extended tiers, with nothing above 60 minutes, so a
   four-hour day has to bill as four Extended visits rather than one long one.
   That produces four claims sharing a date of service, procedure code, unit
   count and dollar amount, distinguished only by the patient control number
   and the `NTE` time:

   ```
   CLM*TESTSI20C1*111.35***12:B:1*Y*C*N*Y*P | NTE*ADD*CV1-0900-1000 | SV1*HC:97155*111.35*UN*4
   CLM*TESTSI20C2*111.35***12:B:1*Y*C*N*Y*P | NTE*ADD*CV1-1000-1100 | SV1*HC:97155*111.35*UN*4
   ```

   That is the shape payers normally reject as duplicates. It is not confined
   to the 20×60 case either — because the weekly mandate can be met over three
   days instead of five, even 5×60 produces two-session days on a compressed
   schedule. Any intensive authorisation runs into this.

   The companion guide does not address same-day repeat services anywhere, and
   neither does the policy FAQ. **How should a multi-session day be submitted?** Four separate
   claims as here, one claim carrying four service lines, or a single line with
   the day's full unit count? If separate claims are right, does EI-Hub need a
   repeat-service modifier to stop them deduplicating, and which one? We have
   deliberately not guessed at a modifier rather than send one the guide never
   mentions.

3. **Special Instruction modifiers.** The SI files send 97155 with no modifier
   at all, on the reasoning that GN, GO and GP are discipline modifiers for
   speech, OT and PT respectively and none of them describes Special
   Instruction. Is a bare code correct for SI, or does EI-Hub expect a modifier
   we have not identified?

4. **Bilingual services.** Which rate code or modifier designates a bilingual
   service, and is the add-on a separate service line or a modifier on the base
   line?

5. **Basic versus Extended visit.** What session length separates them? We have
   assumed 30 minutes is Basic and 60 is Extended.

6. **Loop 2420A.** When several clinicians perform one core evaluation, is it
   one claim with a rendering provider per service line, or a separate claim per
   clinician?

7. **A documentation bug.** Section 5.2's ISA16 row says the component element
   separator must be `~`, and the delimiter table in 5.21 says `|`. Both
   contradict every worked example in the guide, which uses `:` — and a tilde
   there would collide with the segment terminator. The examples look right and
   the two table entries look wrong.
