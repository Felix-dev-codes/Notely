# 837P Test File — What To Replace Before Sending

`TEST_837P_SAMPLE.txt` was produced by Bloom Note itself, not written by hand,
so it is exactly what the app generates. Every identifier in it is a
placeholder.

**Do not send it as it stands.** EI-Hub checks NPIs against the NPPES registry
and the billing provider NPI against its own records. Placeholder numbers fail
on that alone, and the response would tell you nothing about the rest of the
file.

Two ways to use it:

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
| `F802` | ICD-10 for this child, no decimal point | HI |
| `20260720` | Actual date of service, YYYYMMDD | DTP |
| `TEST0001` | Your invoice number, unique to your provider ever | BHT, CLM, REF\*6R |

If you are billing as an **agency** rather than an individual provider, ISA06
and GS02 take your **Tax ID** instead of the NPI. The app's Profile has a
"Billing As" setting that handles this.

Leave `116.66`, `92507:GN`, `UN*1`, `12:B:1` and the segment structure alone
unless the session genuinely differs — those come from the rate schedule and
the companion guide.

---

## What this file represents

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

---

## Checked against the companion guide

| | |
|---|---|
| ISA segment length | 106 characters |
| ISA16 / segment terminator | `:` at position 105, `~` at 106 |
| ISA14 | 1, requesting the return 999 |
| ISA15 | P — the guide accepts production data only |
| SE01 | 27, matching the actual count from ST through SE |
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

2. **Bilingual services.** Which rate code or modifier designates a bilingual
   service, and is the add-on a separate service line or a modifier on the base
   line?

3. **Basic versus Extended visit.** What session length separates them? We have
   assumed 30 minutes is Basic and 60 is Extended.

4. **Loop 2420A.** When several clinicians perform one core evaluation, is it
   one claim with a rendering provider per service line, or a separate claim per
   clinician?

5. **A documentation bug.** Section 5.2's ISA16 row says the component element
   separator must be `~`, and the delimiter table in 5.21 says `|`. Both
   contradict every worked example in the guide, which uses `:` — and a tilde
   there would collide with the segment terminator. The examples look right and
   the two table entries look wrong.
