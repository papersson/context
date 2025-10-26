# Output Schema: decision_log

Meeting decisions distilled from a transcript.

## Fields (per entry)
- id: stable identifier (e.g., D-001)
- decision: short statement of the decision
- owners: list of people responsible
- date: ISO date if present
- rationale: brief why/how summary (optional)

## Format
- Markdown list of entries or a Markdown table with the above fields

Example (list):
```
- id: D-001
  decision: Adopt Option B for API auth
  owners: [Alice, Bob]
  date: 2025-10-26
  rationale: Lower integration risk; aligns with existing infra
```

