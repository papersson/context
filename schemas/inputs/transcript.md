# Input Schema: transcript

Meeting or interview transcript text.

## Formatting
- Each line begins with `Speaker:` followed by utterance
- Optional timestamps in `[hh:mm]` or `[hh:mm:ss]`
- Preserve paragraph breaks between topics

Example:
```
[00:03] Alice: Status update on Project X...
Bob: Next steps are...
```

