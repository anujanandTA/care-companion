---
name: Clinical Normalizer
description: 'Normalize messy synthetic clinical notes into canonical GraphCare entities and graph-ready triplets. Use for shorthand expansion, synonym resolution, and converting notes into Patient, Symptom, Medication, Condition, and ClinicalEvent facts.'
argument-hint: 'Paste a synthetic progress note or shorthand clinical phrase'
agent: 'agent'
---

Normalize the provided synthetic clinical text into canonical GraphCare facts.

Requirements:
- Treat the input as synthetic clinical data only.
- Expand shorthand and abbreviations into canonical medical terms.
- Use GraphCare node labels and graph-ready relationship language.
- Prefer canonical entities over raw note wording.
- Keep every fact grounded in the note text.
- If a term is ambiguous, return it in an `uncertain_terms` section instead of guessing.

Allowed node labels:
- `Patient`
- `Symptom`
- `Medication`
- `Condition`
- `ClinicalEvent`

Preferred relationship types:
- `HAS_SYMPTOM`
- `HAS_CONDITION`
- `TAKES_MEDICATION`
- `EXPERIENCED_EVENT`
- `TRIGGERS` only when the note explicitly describes event-to-event causality

Few-shot examples:

Input:
`Pt is c/o SOB and swelling in both legs. Hx of HF.`

Output:
```json
{
  "normalized_terms": {
    "Pt": "Patient",
    "c/o": "complains of",
    "SOB": "Shortness of Breath",
    "HF": "Heart Failure"
  },
  "triplets": [
    ["Patient", "HAS_SYMPTOM", "Shortness of Breath"],
    ["Patient", "HAS_SYMPTOM", "Bilateral Leg Swelling"],
    ["Patient", "HAS_CONDITION", "Heart Failure"]
  ],
  "uncertain_terms": []
}
```

Input:
`Patient denies CP today but still taking Lasix. Fell last night getting to bathroom.`

Output:
```json
{
  "normalized_terms": {
    "CP": "Chest Pain",
    "Lasix": "Furosemide"
  },
  "triplets": [
    ["Patient", "TAKES_MEDICATION", "Furosemide"],
    ["Patient", "EXPERIENCED_EVENT", "Fall"]
  ],
  "uncertain_terms": [],
  "negated_findings": [
    ["Patient", "HAS_SYMPTOM", "Chest Pain"]
  ]
}
```

Input:
`More tired this week, poor PO intake, maybe dehydrated.`

Output:
```json
{
  "normalized_terms": {
    "PO intake": "Oral Intake"
  },
  "triplets": [
    ["Patient", "HAS_SYMPTOM", "Fatigue"],
    ["Patient", "HAS_SYMPTOM", "Poor Oral Intake"]
  ],
  "uncertain_terms": [
    "dehydrated"
  ]
}
```

Now process the user input and return JSON with these keys:
- `normalized_terms`
- `triplets`
- `negated_findings`
- `uncertain_terms`
- `source_text`

Return only JSON.