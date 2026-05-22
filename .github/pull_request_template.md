<!--
Mustry Academy — PR template
Keep it short; the goal is to make the reviewer's job easy.
-->

## What

<!-- One or two sentences describing what this change does. -->

## Why

<!-- Why are we making this change now? Link any related issue or discussion. -->

## How to test

<!-- Specific commands or steps the reviewer can run. -->

## Checklist

- [ ] Tests pass locally (`pytest sample-app/tests -q`)
- [ ] Compose stack still starts cleanly (`docker compose up -d` → `/health` returns ok)
- [ ] No secrets committed
- [ ] Changes are scoped to one logical thing
