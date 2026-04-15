# QA Notes — Ideas to Explore

A scratchpad for topics worth expanding into full playbook sections.

---

## Synthetic Test Data

**Why it matters:** real user data can't be used in test environments (privacy, compliance). Synthetic data solves this without mocking everything by hand.

**Tools to look at:**
- [Tonic.ai](https://www.tonic.ai) — generates realistic, privacy-safe datasets from production schemas
- [Faker](https://faker.readthedocs.io) — lightweight, code-level data generation for unit/integration tests
- [Mimesis](https://mimesis.name) — similar to Faker, better multilingual support

**Open questions:**
- Does the product handle edge cases in generated data (empty strings, Unicode, boundary values)?
- How do we seed consistent data for repeatable async pipeline tests?
- What's the right layer — DB snapshots vs generated fixtures vs factory functions?

---

## Self-Healing Tests

**Why it matters:** UI and E2E tests break constantly due to selector changes. Self-healing reduces maintenance overhead significantly.

**Approaches:**
- AI-assisted selector fallback (Testim, Mabl, Healenium) — when a locator fails, the tool tries alternative selectors
- Playwright's built-in auto-waiting reduces a class of flakiness already
- Semantic locators (by role, label, text) age better than CSS/XPath

**Open questions:**
- Is self-healing worth the vendor lock-in at early stage?
- Could semantic locators (Playwright `getByRole`, `getByLabel`) cover 80% of the need without a paid tool?
- Where does flakiness actually come from in this stack — async jobs, LLM latency, or selectors?

---

*Add ideas here as they come up. Promote to a full section when there's enough to say.*
