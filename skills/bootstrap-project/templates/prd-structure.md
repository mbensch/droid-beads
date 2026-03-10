# PRD Interview Structure

Use this guide when conducting a PRD interview with the user. Ask one section at a time.
Synthesize their answers into the section format below. Don't ask every sub-question literally --
adapt to the conversation flow and fill in what you can infer.

---

## Section Guide

### Summary (1 paragraph)
What is the product in one paragraph? What does it do and who is it for?

Questions to probe:
- What's the one-sentence pitch?
- What type of product is it? (desktop app, web app, CLI tool, library, API, etc.)
- Who are the primary users?

---

### Problem
What problem does this product solve? What is the current state of the world without it?

Questions to probe:
- What pain does the user have today?
- How do they currently solve this? What's wrong with the current solution?
- Why does this problem need to be solved now?

---

### Product Vision
What does the product look like when it's working? Walk through the core experience.

Questions to probe:
- Describe what the user opens / runs and what they see
- What are the 2-3 core surfaces or interactions?
- What's the "aha" moment?

---

### Core Capabilities
List the key capabilities (not implementation details). Each capability should be something
the user can do or observe.

Questions to probe:
- What are the 3-5 most important things users can do?
- Are there any programmatic / API capabilities?
- What integrations are important?

---

### User Scenarios
2-5 concrete walkthroughs of a user completing a real task from start to finish.

Questions to probe:
- Walk me through a typical day using this product
- What's the most common task a user would do?
- What's the "power user" scenario?

---

### Target Users
Who uses this? A table of user profiles with how they use the product.

Questions to probe:
- Who is the primary user?
- Are there secondary user types?
- What is their technical level?

---

### Non-Functional Requirements
Performance, resource, platform, and reliability requirements.

Questions to probe:
- Any performance targets? (startup time, response time, throughput)
- Platform requirements? (macOS, Linux, Windows, mobile, web)
- Any resource constraints? (binary size, RAM, CPU)
- Security / privacy requirements?

---

### Out of Scope
What are you deliberately NOT building in v1?

Questions to probe:
- What obvious extensions will you defer?
- What would be scope creep?

---

### Open Questions for Engineering
Any unresolved technical decisions that engineering needs to answer.

Questions to probe:
- Any known trade-offs you haven't decided on?
- Any risky technical bets?
- Any external dependencies that are uncertain?

---

## Output Format

Write the PRD as a markdown document with these sections in order:
1. Summary
2. Problem
3. Product Vision
4. Core Capabilities (numbered list with subsections)
5. User Scenarios (numbered, step-by-step)
6. Target Users (table)
7. Non-Functional Requirements (tables or bullet lists)
8. Out of Scope
9. Open Questions for Engineering (numbered list)
