# LinqCV — SaaS Boilerplate (Django/DRF + Stripe)

Repo: https://github.com/tuckerjensendev/LinqCV

## What it is
Production-oriented SaaS boilerplate for a resume optimization flow: auth, file upload, Stripe Checkout, webhook handling, background processing, and document generation.

## My role
- Built the backend + wrote the operational docs (including “what happens when orders fail” and how to recover).

## Stack (high level)
- Backend: Django 5 + Django REST Framework
- Payments: Stripe Checkout + webhooks
- Frontend: React/Vite

## What I shipped (high level)
- Order lifecycle that separates “paid” from “fulfilled” (and handles the ugly middle states).
- A recovery loop for stuck orders (retry processing, refund paid orders, restore free credits when appropriate).
- Documentation that spells out the operational reality (how to run it in dev and how to automate it in prod).

## Best places to read (docs)
- README (system overview + endpoints): https://github.com/tuckerjensendev/LinqCV/blob/main/README.md
- Order recovery playbook (stuck orders): https://github.com/tuckerjensendev/LinqCV/blob/main/ORDER_RECOVERY_README.md

## Backend topics I can write about from this work
- Order state machines (paid vs processing vs failed vs refunded)
- Webhook reliability, retries, and idempotency
- “Stuck order” recovery strategies (cron jobs, management commands, login-time repair)

## If you want blog post ideas I can write from this (examples)
- “Paid isn’t fulfilled: designing order state machines that don’t gaslight users”
- “Stuck Stripe orders: how to build an automatic recovery loop (and when to refund)”
