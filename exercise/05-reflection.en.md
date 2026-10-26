**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** Teerachot
**My role on the review board:** Architect

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

I would trust AI to build an architecture view from evidences and identify assumptions that need validation. Its strongest finding was that the accepted order and outbox event need an atomic commit and a defined recovery path. I would keep stakeholder priorities, tradeoffs choosing, and the final decision with people. Product owner raised customer-facing questions about delayed loyalty points and refund reversals that the ADR could not answer. SRE raised ownership and incident-recovery needs. Those concerns depended on business and operational context outside the review evidence. AI can help the board reason, but humans must supply that context and own approval.
