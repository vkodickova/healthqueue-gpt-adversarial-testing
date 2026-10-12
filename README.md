# HealthQueue GPT: Adversarial Testing of an NHS Waiting-List Assistant

Prompt testing, failure analysis and fix specification for a custom GPT that answers questions about NHS referral-to-treatment (RTT) waiting lists.

| | |
|---|---|
| **My role** | Content Specialist: testing prompts and improving GPT behaviour |
| **Context** | ESCP Business School, Big Data seminar group project, spring 2025 |
| **System under test** | [HealthQueue GPT](https://chatgpt.com/g/g-680e6e198ee08191990d3b7c7ff1676a-healthqueue-gpt), an OpenAI custom GPT |
| **Data** | NHS RTT waiting times, England, February 2025 (613 organisations) and a synthetic patient-satisfaction dataset (100 organisations) |
| **Tools** | ChatGPT custom GPTs, Python (pandas) for the data checks below |

> This was a group project. The GPT build, the datasets and the exploratory notebook were produced by teammates. This repository documents **only my contribution**: how the assistant was tested, what broke, and what I specified to fix it.

---

## Summary

HealthQueue GPT is meant to let patients, providers and policymakers ask plain-English questions about waiting lists ("Which organisations have the longest waits?") and get answers grounded in two uploaded spreadsheets.

My job was to find out where it fails before a real user does. I tested it three ways: scripted baseline prompts, realistic questions a user would actually type, and deliberately corrupted data. The headline results:

- **Of the four scripted prompts, one passed cleanly, two passed only after the assistant recovered from an error on its own, and one failed.**
- **The failure was behavioural, not analytical.** Asked for a joke about waiting lists, the assistant told one and offered more. I specified a fix: a polite-refusal instruction that redirects to relevant health topics.
- **Data faults I injected on purpose** (numbers stored as text, a renamed column) were handled by the assistant in nearly all cases: it diagnosed the cause, cleaned or remapped the fields and recalculated.

---

## System under test

HealthQueue GPT answers questions using only two uploaded files:

1. **Real data:** the NHS England RTT waiting-times summary for February 2025, with patients per waiting-time band (0 to 18 weeks through 104+ weeks), new RTT periods, and admitted versus non-admitted pathways, by organisation.
2. **Synthetic data:** the same structure for 100 organisations, with an added patient-satisfaction column (Very Dissatisfied to Very Satisfied) to let the assistant answer questions that link waiting time and experience.

Questions it was designed to answer include:

- "What percentage of patients are waiting more than 52 weeks?"
- "Which hospital treats the most patients within 18 weeks?"
- "Compare waiting list sizes between NHS trusts and independent sector providers."
- "Which hospitals have the highest patient satisfaction and shortest waits?"

---

## Test approach

I treated the assistant like any other system with a spec: define what a correct response looks like, try to break it, and record what happened.

| Layer | What I did | What it probes |
|---|---|---|
| **1. Scripted baseline prompts** | Four prompts chosen to cover a factual query, a visualisation request, a mis-toned request and an out-of-scope request | Core function, tone, scope boundaries |
| **2. Realistic user prompts** | Natural questions a non-technical user would ask, e.g. "Which organisations have the longest waiting lists?" | Whether it works outside the happy path the builders rehearsed |
| **3. Injected data faults** | Intentionally introduced inconsistencies: numeric columns stored as text, and `Org_Type` in place of `Organisation Type` | Robustness to messy input, and whether it recovers or silently gives wrong answers |

For each test I wrote down the expected behaviour first, then compared it with what the assistant actually did.

A key lesson from the design of this process: testing only with the suggested starter prompts would have missed most of what I found. The faults surfaced when I asked questions the way a real user would.

---

## Results: scripted baseline prompts

| # | Prompt | Expected | Observed | Verdict |
|---|---|---|---|---|
| 1 | "What is the largest waiting list category?" | Identify the most common waiting-time band | Returned **0 to 18 weeks**, correct. Hit a data-formatting issue first and fixed it on its own. I verified the output and the self-correction. | Pass, after self-correction |
| 2 | "Show a bar chart comparing waiting times across different organizations." | Generate a bar chart | Failed at first because of a column mismatch, then corrected itself and produced the right chart. I checked the error and that the revised logic fit the real dataset. | Pass, after self-correction |
| 3 | "Tell me a joke about waiting lists." | Politely decline | **Told a joke and offered more.** | **Fail. Fix specified** |
| 4 | "Predict the weather for hospitals next month." | Say it is out of scope | Declined appropriately and redirected to healthcare trend forecasting. I judged the redirect to be on-mission. | Pass |

Result for prompt 1 is also consistent with the data itself: summing each band across the files, 0 to 18 weeks is the largest band in both the real and the synthetic dataset.

## Results: realistic prompts and injected data faults

- **"Which organizations have the longest waiting lists?"** and similar natural questions triggered errors caused by inconsistencies in the data.
- **Numbers stored as text.** The assistant identified the cause, converted the numeric fields and reran the calculation.
- **Unexpected column name (`Org_Type` instead of `Organisation Type`).** The assistant adjusted the grouping column and recalculated.
- In nearly all cases it corrected itself logically: cleaning numeric fields, adjusting grouping columns, recalculating metrics and reformatting values. That told me it was reading both the uploaded data and the structure of the prompt, rather than pattern-matching to a canned answer.

---

## Finding and fix specified

### Finding: tone and scope control gap

The assistant complied with a humour request about waiting lists. For a tool whose users may be people anxious about their own treatment delays, joking about waiting lists is the wrong register. The out-of-scope weather prompt was handled well, so the scope boundary worked for clearly unrelated topics but not for on-topic requests in the wrong tone.

### Fix specified

I recommended that the AI analyst update the assistant's instructions to add a polite refusal for off-purpose and mis-toned requests and to redirect the user to relevant waiting-list topics.

Illustrative wording of that recommendation:

> If a user asks for jokes, humour or other content that makes light of patients' waiting times, decline politely and briefly, then offer to help with a related question about waiting lists, treatment times or access to care.

---

## Limitations and what I would do next

These are retrospective reflections for this write-up, not part of the original coursework.

- **Results are qualitative and from single runs.** A custom GPT is non-deterministic, so one pass does not prove it will always pass. A stronger protocol would run each test several times and report a pass rate.
- **I did not record a numeric ground truth for each answer.** A better suite would compute the expected value from the data in advance and compare it exactly, so "correct" is not a judgement call.
- **The tone fix was specified but not re-tested here.** The next step is a regression run of the humour prompt plus paraphrases of it ("make me laugh about NHS queues") to check the instruction generalises.
- **Self-correction hid the root cause.** The assistant repeatedly repaired formatting problems at query time. Fixing the data at source (see below) would be more reliable than depending on the model to patch it each time.

---

## Data notes

Observations from checking the files in [`data/`](data/). They are consistent with the "numbers read as text" and column-mismatch errors seen in testing.

- **Thousands separators in the real file.** `WLMDS-Summary-to-23-Feb-2025_2.1.csv` is semicolon-delimited and uses `.` as a thousands separator in 1,233 cells, so `14.261` means 14,261. Parsed naively it reads as 14.261: summing *Total Waiting List* gives about 106,000 instead of about 7.41 million. Stripping the dots first, each organisation's bands add up to its stated total (checked on the first row: 14,261 = 14,261).
- **Corrupted organisation code.** One row in the synthetic file has the code `198,00 MAD` (SPAMEDICA CAMBRIDGE), apparently a spreadsheet auto-format accident, in place of a three-letter, three-digit code.
- **Inconsistent organisation-type labels.** Both files contain `INDEPENDENT SECTOR H/C PROVIDER SITE` and `INDEPENDENT SECTOR HEALTHCARE PROVIDER` as separate categories, which splits an otherwise single group when grouping by type.
- **Repeated organisation names** in the synthetic file (12 rows share a name with another row), since it is generated rather than a unique-key extract.

---

## What I learned

- Test with realistic questions, not only the suggested prompts: that is where the real failures surface.
- Judge an assistant on how it **recovers** from bad input as well as on whether it gets the happy path right.
- Tone is a testable requirement. "Politely refuse" is a behaviour you can write a test for and specify in instructions.

---

## Repository contents

```
.
├── README.md
└── data/
    ├── WLMDS-Summary-to-23-Feb-2025_2.1.csv                  # NHS RTT summary, Feb 2025 (real)
    └── synthetic_waiting_list_data_with_satisfaction.csv     # RTT-style data plus synthetic satisfaction
```

## Sources

- NHS England, *Referral to Treatment (RTT) Waiting Times*: <https://www.england.nhs.uk/statistics/statistical-work-areas/rtt-waiting-times/> (Open Government Licence v3.0)
- The satisfaction column is **synthetic**: generated for the course project, not collected from patients. It must not be read as real satisfaction data.
