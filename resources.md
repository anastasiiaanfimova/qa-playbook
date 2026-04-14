# Resources

Useful repositories, tools, and references for QA.
Worth checking periodically for new releases and updates.

## QA tools

### LLM / AI output testing
- [confident-ai/deepeval](https://github.com/confident-ai/deepeval) — pytest-like LLM eval framework; pre-built metrics for RAG, agents, conversational AI; `@observe` tracing
- [promptfoo/promptfoo](https://github.com/promptfoo/promptfoo) — prompt comparison + red-teaming/security scanning; used by OpenAI and Anthropic internally
- [Shredmetal/llmtest](https://github.com/Shredmetal/llmtest) — behavioral testing in plain English, no embeddings or exact matching (beta, worth watching)

### Async pipeline & webhook testing
- [kubeshop/tracetest](https://github.com/kubeshop/tracetest) — tests async operations via OpenTelemetry traces instead of mocks; assert side-effects across queues and async APIs without test doubles

### API testing
- [stepci/stepci](https://github.com/stepci/stepci) — YAML-config API testing, REST/GraphQL/gRPC/tRPC in one tool, CI-native
- [TestCraft-App/api-automation-agent](https://github.com/TestCraft-App/api-automation-agent) — LLM generates test frameworks from OpenAPI/Postman specs

### Load & performance testing
- [grafana/k6](https://github.com/grafana/k6) — still best for developer-friendly load testing; v1.0 released 2025

## Curated lists
- [malomarrec/awesome-qa](https://github.com/malomarrec/awesome-qa) — most comprehensive QA tool index
- [atinfo/awesome-test-automation](https://github.com/atinfo/awesome-test-automation) — language-specific automation tools

## References
- [danilop/non-deterministic-software-testing](https://github.com/danilop/non-deterministic-software-testing) — 7 patterns for testing non-deterministic systems (Danilo Poccia, AWS); good conceptual foundation
- [TestGuild](https://testguild.com) — podcast and blog tracking QA tool ecosystem
- [Abstracta blog](https://abstracta.us/blog) — practical testing articles
