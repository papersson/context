# Output Schema: action_items

Action items derived from a decision log.

## Fields (per item)
- id: stable identifier (e.g., A-001)
- decision_id: reference to related decision
- owner: person responsible
- task: concrete task description
- due: ISO date or empty if unspecified
- status: `todo` | `in_progress` | `done`

## Format
- Markdown table with the above fields

Example:

| id   | decision_id | owner | task                         | due        | status |
|------|-------------|-------|------------------------------|------------|--------|
| A-001| D-001       | Bob   | Implement Option B in auth   | 2025-11-10 | todo   |

