# nash_banking

**Latest release: v1.0.10** — see the [Changelog](changelog.md) for what's new.

nash_banking is a premium, feature-complete banking system for FiveM. It ships with a desktop UI, two phone apps (LB Phone / Quasar Phone), a TPE (payment terminal) system, cards, investments, crypto, businesses, savings, subscriptions, and an admin panel.

## Features

- Desktop banking UI (React + Tailwind)
- Phone apps: personal banking & business banking (LB Phone, Quasar Phone V3, Quasar Phone Pro - auto-detected)
- ATM interactions with animated card insertion, per-action toggles
- TPE payment system (in-script export + P2P via inventory item), with server-side hooks to route receipts to an external business system
- Physical & virtual cards with limits
- Subscriptions (Standard / Plus / Premium) - rates, limits & features driven by config
- Savings account with configurable interest rate per tier
- Stock investments & crypto market
- Businesses (personal-owned or job-linked) with employees, TPE income routing, transactions
- Fully configurable displayed RIB (IBAN / BIC / bank name and address)
- Discord webhook logs
- ESX / QBCore / QBOX / custom framework support via a unified bridge
- ox_inventory / qs-inventory / qb-inventory / custom inventory support via a unified bridge
- Runtime locales - add any language by dropping a `locales/<lang>.lua` file (no rebuild needed)

## Requirements

- ox_lib
- oxmysql
- An inventory (ox_inventory / qs-inventory / qb-inventory / custom)
- A framework (ESX / QBCore / QBOX / custom)
- (optional) A phone: lb-phone, qs-smartphone (Quasar V3), or qs-smartphone-pro

## Links

- [Installation](installation/README.md)
- [Compatibility](compatibility/README.md)
- [Configuration Files](config/README.md)
- [FAQ](faq.md)
- [Common Errors](common-errors.md)
- [Guides](guides/README.md)
- [Developer API](developer-api/README.md)
