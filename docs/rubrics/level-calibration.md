# Level Calibration Matrix (SDE 1 → SDE 4)

## Purpose
Prevents over-hiring and under-hiring. Every interviewer must calibrate against these expectations.

---

## SDE 1 (Junior) 0-2 Years

| Dimension | Expected Signal |
|-----------|----------------|
| Scope | Executes well-defined tasks within a component |
| Coding | Implements correct solution with guidance; handles happy path |
| Design | N/A (not assessed at this level) |
| Testing | Writes basic unit tests; understands why testing matters |
| Communication | Can explain their approach; asks clarifying questions |
| Autonomy | Needs direction on what to build; can execute independently once clear |

### What "Strong Hire" looks like at SDE 1:
- Solves LC Easy in <15min, Medium with hints
- Clean code without being told (variable names, small functions)
- Asks good clarifying questions before coding
- Articulates time/space complexity when prompted

### Red flags:
- Cannot implement basic data structures (linked list, hash map)
- Writes code that doesn't compile/run
- Cannot trace through their own code with a test case

---

## SDE 2 (Mid-Level) 2-5 Years

| Dimension | Expected Signal |
|-----------|----------------|
| Scope | Owns features end-to-end; translates requirements to implementation |
| Coding | Writes production-quality code; handles edge cases proactively |
| Design (LLD) | Designs classes/interfaces for a single service; SOLID principles |
| Testing | Writes comprehensive tests; understands mocking, integration testing |
| Communication | Clearly explains trade-offs; documents design decisions |
| Autonomy | Self-directed within a team; breaks down ambiguous problems |

### What "Strong Hire" looks like at SDE 2:
- Solves LC Medium cleanly in <25min
- Designs clean class hierarchy for a given problem (LLD)
- Proactively discusses edge cases, error handling, testability
- Can walk through a feature they've built end-to-end

### Red flags:
- Cannot design beyond a single function
- No awareness of concurrency/thread safety
- "It works on my machine" attitude toward testing

---

## SDE 3 (Senior) 5-10 Years

| Dimension | Expected Signal |
|-----------|----------------|
| Scope | Drives technical direction for a team/service; cross-team influence |
| Coding | Elegant, maintainable solutions; performance-conscious |
| Design (HLD) | Designs multi-service architectures; makes infrastructure trade-offs |
| Testing | Defines testing strategy; understands chaos engineering |
| Communication | Mentors others; writes RFCs; presents to leadership |
| Autonomy | Identifies problems before they're assigned; proposes solutions |

### What "Strong Hire" looks like at SDE 3:
- Designs a complete system (HLD) with appropriate trade-offs
- Drives LLD discussion covering extensibility, caching, failure modes
- Demonstrates cross-team collaboration in past work
- Can articulate "why not" for alternative approaches

### Red flags:
- Cannot think beyond a single service
- No experience mentoring or leading design reviews
- Avoids discussing failures or trade-offs

---

## SDE 4 (Staff) 10+ Years

| Dimension | Expected Signal |
|-----------|----------------|
| Scope | Organization-wide technical strategy; multi-year roadmaps |
| Coding | Sets coding standards; reviews architecture; IC excellence |
| Design (HLD++) | Designs at scale (10M+ users); resilience, observability, cost |
| Testing | Defines quality culture; testing infrastructure ownership |
| Communication | Influences without authority; presents to VPs/CTO |
| Autonomy | Creates clarity in ambiguous org-wide problems |

### What "Strong Hire" looks like at SDE 4:
- Designs systems handling millions of requests with concrete numbers
- Articulates multi-year technical strategy aligned to business goals
- Demonstrates org-wide influence (standards, platforms, tools)
- Has shipped complex, multi-team projects from concept to production

### Red flags:
- Cannot articulate business impact of technical decisions
- Only thinks in code, not systems/organizations
- No evidence of multiplying other engineers

---

## Quick Comparison Table

| Dimension | SDE 1 | SDE 2 | SDE 3 | SDE 4 |
|-----------|-------|-------|-------|-------|
| Scope | Task | Feature | Service/Team | Org |
| Ambiguity | None | Low | Medium | High |
| Influence | Self | Team | Cross-team | Org-wide |
| Design | | LLD | HLD | HLD at scale |
| Mentoring | | Informal | Active | Defining culture |
| Coding bar | Easy+Medium | Medium clean | Medium optimal | Sets standards |

---

## Calibration Tips for Interviewers

1. **Don't conflate years with level** A 10-year dev might be SDE 2 if they lack design depth
2. **Score the signal, not the resume** Past titles ≠ demonstrated ability
3. **Use follow-up questions to disambiguate** One good question can reveal level
4. **Compare to your strongest teammate at that level** Would this person hold their own?
5. **Document specific examples** "Good at design" is useless; cite what they said/drew
