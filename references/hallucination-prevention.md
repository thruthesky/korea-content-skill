# Hallucination Prevention — Failure-Mode Catalog + Countermeasures

This reference exists because a post that **passed** the previous 32-gate 90/100 rubric (`ph-cebu-oslob-whale-sharks`, post_id 2024) was later found to contain fabricated and outdated facts. The rubric scored structure and length, not truth — because the rubric was self-evaluated, it could not detect what the author had invented.

This doc catalogs the specific failure modes observed, the countermeasures added to the skill in response, and the red-team prompt template for the step-6 independent fact-check pass.

> 🔴 **Read this before drafting any factual post.** If you skip it, you will reproduce the same hallucinations.

---

## 1. Failure modes observed (real incidents)

### 1.1 Fabricated academic citation

**What happened**: The draft cited `"Schleimer et al., 2018, Travel and Tourism Ethics 저널"`. No journal named *Travel and Tourism Ethics* exists. The author invented a plausible-sounding journal name to attach to a real-but-generically-attributed finding ("Guilty Pleasure" tourist attitudes are indeed documented in Oslob research — but in *Tourism Management* (Ziegler et al. 2018), not the invented journal).

**Root cause**: LLMs are trained to produce authoritative-looking prose, which includes specific-looking citations. When the rubric rewarded "specific sources," the author fabricated plausible specifics to score well. The generic form of the claim was true; the specific attribution was invented.

**Countermeasures added**:
- **G33**: Every academic / journal citation must include a DOI or direct URL. Citations without links fail outright.
- **Drafting rule**: If you don't have a specific URL for a specific paper, use a vague hedge ("여러 국제 연구에서", "multiple studies have found") — never invent a journal name.
- **Claim extraction**: Every citation gets a row in `claims.json` with `source_url`. A citation with `source_url: null` is automatically REMOVEd or SOFTENed.
- **Red-team**: The fact-check agent checks whether cited journals exist and whether the DOI resolves.

### 1.2 Fabricated anchor-number ("1,300년 인류사")

**What happened**: The draft stated `"1,300년 인류사에서 고래상어에 의한 사망 사례는 단 한 건도 없다"` ("in 1,300 years of human history, no whale-shark-caused human deaths have been recorded"). The number "1,300년" is nonsensical — human history is vastly longer; there is no 1,300-year anchor for whale-shark-attack records.

**Root cause**: The author wanted to sound authoritative with a specific number. When no specific number was in the research notes, it invented a plausible-sounding one. This is the purest form of LLM confabulation.

**Countermeasures added**:
- **G36**: Every "since Year" / "N-year history" / specific anchor-date claim must have a source. No inventing anchor durations.
- **Drafting rule**: Absolute-sounding phrases ("N년 만에 처음", "X 세기 동안") must be sourced or replaced with vague-but-true alternatives ("현대 해양 관찰 기록에서", "in recent decades").
- **Claim extraction**: Every anchor-number appears in `claims.json` with a source.
- **Red-team**: The fact-check agent checks whether the anchor duration is plausible and sourced.

### 1.3 Outdated safety info (emergency number "117")

**What happened**: The draft said `"필리핀 경찰 긴급번호는 117"`. The Philippines officially consolidated emergency hotlines under **911** in August 2016 (RA 10844). "117" is deprecated. Publishing this in a travel safety guide can actively harm a reader in an emergency.

**Root cause**: The author's training data contained pre-2016 references. The rubric's "Accuracy" gate is self-evaluated and didn't recognize the fact was stale. This is not fabrication per se — it's outdated knowledge, but the impact is the same: wrong information published as current.

**Countermeasures added**:
- **G34**: Every phone number / emergency code / contact hotline links to the owning entity's **current** official page. Each is verified within 12 months.
- **Risk class `safety`**: claims with `risk_class: "safety"` and `confidence < "high"` cannot ship.
- **Drafting rule**: Emergency numbers, embassy contacts, and other safety-critical info are always WebFetched fresh from the owning agency's current page — never recalled from memory.
- **Red-team**: The fact-check agent verifies every emergency number against the current official source.

### 1.4 Wrong statute reference (RA 10654 vs FAO 193)

**What happened**: The draft said `"공화국법(Republic Act) 10654에 의거해 어기면 형사 처벌"` — citing RA 10654 as the whale-shark protection law. RA 10654 (2015) is an amendment to the general Fisheries Code, not specifically the whale-shark ban. The actual prohibition is **Fisheries Administrative Order No. 193 (1998)**; wildlife penalties flow through **RA 9147 (2001)**.

**Root cause**: The model recognized that a real Philippine fisheries law exists and attached a plausible number (RA 10654 does exist and does concern fisheries), but got the specific applicability wrong. This is subtle — the law is real, just not the right one for the claim.

**Countermeasures added**:
- **G35**: Every law / statute / regulation number cited links to official statute text (`lawphil.net`, `officialgazette.gov.ph`) and the cited law must actually cover the claimed subject matter.
- **Drafting rule**: Law references go into `claims.json` with `risk_class: "legal"` and `confidence: "high"` required. Low-confidence legal claims are dropped or researched until verified.
- **Red-team**: The fact-check agent verifies the law's scope by reading the linked text.

### 1.5 Possibly-fabricated proper noun ("발릴란 balilan" as Cebuano for whale shark)

**What happened**: The draft said `"현지에서 고래상어는 부탄딩(butanding) 또는 발릴란(balilan)으로 불리며"`. The common Filipino term is *butanding* (Tagalog/Bikol). In Cebuano-speaking areas (which includes Oslob), the documented term is typically *tuki* or *tuki-tuki* — not "balilan." "Balilan" could not be verified as a Cebuano whale-shark name.

**Root cause**: The model appeared to transliterate or invent a plausible-looking alternate name. Partial pattern-matching on the kind of name Filipino fish get, without a source.

**Countermeasures added**:
- **Drafting rule**: Foreign-language proper nouns (alternative names, regional variants) must be sourced from a verifiable source (dictionary, academic paper, official tourism page). If unsourced, use only the well-documented name (*butanding*) and skip the unverified alternate.
- **Claim extraction**: Proper nouns count as testable claims.
- **Red-team**: The fact-check agent verifies alternate-name claims against dictionaries and authoritative language sources.

### 1.6 Wrong contact info (embassy phone number)

**What happened**: The draft listed `"주필리핀 대한민국 대사관 (+63 2 8856 9210)"`. The publicly listed main line is different (variants seen: `+63 2 7798 2700`). Neither could be confirmed as the current official number without a fresh fetch from the embassy's own site.

**Root cause**: Same as 1.3 — outdated or mismatched knowledge. Safety-critical contact info recalled from training data without fresh verification.

**Countermeasures added**: Same as 1.3 — G34 requires every phone number to link to the owning entity's current official page.

### 1.7 Unsourced statistic (e.g. "40% of observed sharks have scars")

**What happened**: The draft cited `"관찰 개체의 40% 이상에서 보트 프로펠러와 스노클러 접촉으로 생긴 흉터가 확인"`. The LAMAVE research does document high injury rates, but the specific 40% figure was attributed generically without a paper or year. The figure may be accurate for a specific study but the sourcing in the post was not specific enough to verify.

**Root cause**: Aggregating a specific-sounding percentage from vague memory of the topic area, rather than citing a specific study.

**Countermeasures added**:
- **G37**: Absolute and specific-percentage claims have a sourced basis; prefer vague alternatives when hard numbers can't be sourced.
- **Drafting rule**: Percentages go into `claims.json`; unsourced ones are SOFTENed to vague language ("high injury rates", "상당수").

---

## 2. Anti-patterns — recognize these in your own prose

When you catch yourself about to write any of these, **stop and verify** or rephrase.

| Anti-pattern | Why it's risky | Safe rephrase |
|---|---|---|
| "X et al., YYYY, in *Journal Name*" | Invites fabricated journal | "여러 연구에 따르면" + a real link if available |
| "N-year 인류사에서…" | Invites invented anchor number | "현대에는", "오늘날까지" |
| "긴급번호는 NNN" (any phone number) | Invites stale/wrong number | WebFetch fresh, link to official page |
| "공화국법 RA-NNNNN" (any statute) | Invites mismatched law | Link to official statute text; verify scope |
| "전 세계에서 유일", "최초", "단 한 건도" | Invites false absolute | "주요 사례 중", "드문" |
| "관찰 개체의 N%" (any specific percentage) | Invites made-up stat | "상당수", "다수" — or cite specific study |
| "[alternative foreign name]" without a source | Invites invented transliteration | Use only the well-documented name |

---

## 3. Red-team prompt template

Use this (or equivalent) as the prompt for the step-6 independent fact-check agent. Use a fresh context — ideally a different model invocation. Copy/paste and substitute `{FULL_DRAFT}` and `{CLAIMS_JSON}`.

```
You are fact-checking a Korean-language article drafted by another AI for a Korean-expat
community site. Your job is to find factual errors — fabrications, outdated info, wrong
numbers, misattributed citations, wrong statutes, fake phone numbers. Do NOT evaluate style,
structure, or word choice. Only facts.

Article (Markdown):
<<<
{FULL_DRAFT}
>>>

Claim ledger (claims.json):
<<<
{CLAIMS_JSON}
>>>

Return strictly valid JSON — no prose commentary. Use exactly this schema:

{
  "definite_errors": [
    { "id": "E001",
      "quote": "exact text fragment from draft",
      "problem": "what's wrong",
      "evidence": "what the truth is and how you verified (include a URL)",
      "severity": "safety | legal | financial | contact | credibility" }
  ],
  "likely_errors": [
    { "id": "E002",
      "quote": "...",
      "problem": "...",
      "evidence": "..." }
  ],
  "plausible_unverified": [
    { "id": "E003",
      "quote": "...",
      "note": "plausible but not verified against a primary source — user should confirm" }
  ]
}

Rules:

1. A claim is a "definite error" only if you can cite a contradicting PRIMARY source
   (the owning entity's official site, a published paper with a DOI, a government statute
   page). Otherwise it's "likely" or "plausible".
2. Every specific phone number, coordinate, and law must be verified against the owning
   entity's own current website — do NOT rely on blog summaries.
3. If a cited paper / journal has no findable DOI or URL, that's a definite error
   (fabrication) — severity: credibility.
4. If an emergency or safety number is outdated, that's a definite error — severity: safety.
5. Proper nouns in foreign languages (alternative names, regional variants) must be sourced
   to a dictionary, academic paper, or official tourism/government page.
6. Absolute claims ("only one in the world", "first in Philippines", "no recorded cases")
   must have a primary source or be downgraded.
7. Fix suggestions are NOT your job. Just identify problems.
8. Do not invent problems — if the draft is correct, return empty arrays.

Respond with ONLY the JSON object, no preamble or postscript.
```

### How the author consumes the red-team output

```python
if len(red_team.definite_errors) > 0:
    # BLOCK submission. Fix each error in the draft, re-extract claims,
    # re-run red-team. Max 2 revision rounds; then abandon the reservation.
    revise_each(red_team.definite_errors)
    rerun_from_step_5()

elif len(red_team.likely_errors) > 0:
    # Each likely error must be either:
    #  (a) fixed in the draft, or
    #  (b) countered with a primary source the red-team didn't consider.
    for err in red_team.likely_errors:
        resolve_or_counter(err)

# plausible_unverified is informational only — doesn't block submit,
# but goes into the post's audit log for future re-verification.
log_plausible_unverified(red_team.plausible_unverified)

if pass_all_gates and score >= 90 and red_team.definite_errors == []:
    submit_post()
```

---

## 4. Verification cadence by risk class

| Risk class | Verify when | Verify against | Max staleness |
|---|---|---|---|
| **safety** (emergency, embassy, medical) | Every draft | Official government / embassy page | 6 months |
| **legal** (statutes, visa rules) | Every draft | Official statute text | 12 months |
| **financial** (prices, fees, rates) | Every draft | Vendor's own current rate card or recent (≤3 months) news | 3 months |
| **contact** (phones, addresses, coords) | Every draft | Owning business's own site, Google Maps | 6 months |
| **descriptive** (history, culture, biology) | Every draft | ≥1 credible secondary source (2+ for contested) | Usually stable |

The author **must** WebFetch fresh for safety/legal/financial/contact claims — training-data recall is not acceptable for these categories. Descriptive claims may rely on research-phase secondary sources.

---

## 5. Post-publish re-verification (recommended)

Even with steps 5–7 passed, facts decay:

- Prices rise
- Phone numbers change when a business moves
- Laws get amended
- Embassy contact numbers get restructured
- Businesses close

Recommended: run the red-team agent against published posts every **30 days** for the first 90 days after publish, then every **90 days** after. Any `definite_errors` surfaced post-publish should trigger a PATCH to the post.

---

## 6. Common false positives in the red-team output

When the red-team flags something but the author is confident, check these legitimate sources of friction:

- **Transliteration variance**: Korean transliteration of foreign place names can differ ("오슬롭" vs "오슬롭"). Red-team may flag rare variants.
- **Legitimate regional variation**: A business may have different phone numbers for different regions.
- **Recent changes**: The red-team's training cutoff may be older; very recent changes (embassy moves, law amendments) may not be in its training data.
- **Approximate numbers**: "약 3만 명" vs exact census figure — the author's hedging is usually fine.

In these cases, counter the red-team's finding with a primary source URL that resolves the ambiguity. Don't dismiss flags without evidence — the red-team is usually right.

---

## 7. When in doubt — the prime directive

**Drop the specific and keep the general.** A Korean reader will forgive vagueness ("매년 수만 명", "여러 연구에 따르면"). They will not forgive:

- A wrong emergency number they dial in a crisis.
- A fabricated law citation they try to reference in a legal matter.
- A made-up academic paper they try to look up.
- A nonsensical historical anchor that makes the entire article look careless.

Vagueness is a feature, not a bug, when you can't verify. The site's credibility survives one vague paragraph far better than it survives one fabricated citation.
