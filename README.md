# ADR (Architecture Decision Record) collection for Apache Iceberg

This is ADR (Architecture Decision Record) collection containing Golden Path recommendations for working with Apache Iceberg.

## Agents.md

This ADR collection is compiled into [_agents.md](_agents.md) file that you can copy into your Apache Iceberg project as `AGENTS.md` to help your AI agent to take advantage of our ADR recommendations.

## Building

To created compiled [_agents.md](_agents.md) file execute the following command:

```bash
yax build
```

Yax build is called via [this CI pipeline](.github/workflows/yax.yml), so usually you don't have to run it manually.