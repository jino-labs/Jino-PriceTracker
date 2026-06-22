# Jino-PriceTracker

Monitors a product's price and availability across multiple online stores and
notifies you when a price drop or restock you defined happens — so good moments
to buy aren't missed by checking each store by hand.

## Status

Greenfield. No application code yet — only objective specs and tooling.

## Stack

- **Backend:** ASP.NET Core
- **Frontend:** Blazor
- **Database:** SQL Server (MSSQL)
- **Orchestration / deploy:** .NET Aspire

## Objectives

Work is defined as nine standalone objectives in
[`docs/objectives/objectives.md`](docs/objectives/objectives.md), each mirrored
by a GitHub feature issue (#4–#12). That document is the source of truth.

## Development

Agent skills (Aspire + Blazor) are vendored project-local and tracked via
`skills-lock.json`. On a fresh clone, restore the machine-local symlinks:

```sh
npx skills experimental_install
```
