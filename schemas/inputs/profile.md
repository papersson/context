# Input Schema: profile

Structured personal/work profile for planning and routines.

## Sections
- Identity: role, core_skills, niche, superpower
- Current Situation: status, work, location, next_role, date/context note
- Health: height_cm, weight_kg, goals, training/week, cardio
- Daily Rhythm: wake_time, last_meal_time, sleep_time, morning_flow (ordered steps)
- Goals: immediate, prep_focus (time window), long_term_target, study_focus, milestones
- Challenges: list of succinct challenge statements
- Food: likes (list), strategy, key_success
- What Works: list of patterns that help
- Environment Hacks: list
- Metrics: list of tracked metrics with target or scale
- Tech Stack: frontend, backend, data, infra, tooling, principles
- Active Projects: name + one-line focus
- Preferences: scheduling, evenings, learning, weekends, planning_rhythm

## Formatting
- Markdown headings for sections above
- Bulleted lists for items, simple key: value lines accepted
- Dates in ISO when specified (e.g., 2025-11-01)

## Minimal Example
```
## Identity
- Role: Data-focused software engineer (B2I)
- Core Skills: ML & AI, data eng, distributed systems
- Niche: AI × high-performance × distributed
- Superpower: Fast mastery; deep single-thread focus

## Current Situation (September 2025)
- Status: Accepted Apple consultant offer (starts 2025-11-01)
- Work: Transitioning out of B2I
- Location: Remote
- Next Role: Self-employed consultant via middleman at Apple

## Health
- Height: 194 cm
- Weight: 100 kg
- Goal: Lose 5–10 kg fat; maintain/increase muscle
- Training: 5–6x/week strength
- Cardio: 60-min walk daily

## Daily Rhythm
- Wake: 06:00
- Last Meal: 17:00
- Sleep: 22:00
- Morning Flow: Wake → Water + YouTube (60m) → Gym → Recovery (1–2h) → Shower → Deep Work (2h) → Walk (60m)
```

