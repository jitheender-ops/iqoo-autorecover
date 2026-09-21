# AutoRecover — iQOO Hackathon 2026 submission

**Track:** FinTech and Commerce
**Repository:** https://github.com/jitheender-ops/iqoo-autorecover
**Built by:** Jitheender Kumar Manapuram

This file is the submission summary. The full technical documentation is in
[README.md](README.md); every claim below links to the code or the test that
backs it.

---

## The problem

A failed payment is not a lost customer. It is money still on the table, and
almost nobody goes back for it properly.

The industry default is a fixed 3-retry schedule. Measured against this
project's own eval harness it recovers **19.2%** of failed volume — meaning
**₹81.5 lakh per ₹1 crore walks away** — and **8.0% of its attempts are false
retries**: gateway cost spent chasing money the customer had already paid. It
knows nothing about *why* the payment failed, which bank is down right now, or
whether this customer already asked to be left alone.

The other half of the problem is never handled at all. Sending a link is easy;
*listening to the reply* is where recovery actually happens — a promise to pay,
a request for instalments, a disputed invoice, a phone call. Manual dunning
does not scale to that, and it breaks the rules while failing: calls at 11pm,
four separate messages to one B2B buyer, automated chasing of an invoice the
customer has disputed.

## The solution

AutoRecover decides **whether**, **when** and **on which rail** to retry a
failed payment — and then handles the reply. The architectural claim is one
sentence:

> **No language model ever authorises money movement.** It selects one of five
> fixed actions; deterministic code executes, and a guardrail gate stands
> between the two.

Eight layers (full detail in [README.md](README.md#-architecture)):

| Layer | What it does |
|---|---|
| 1 · Ingestion | HMAC signature verify → idempotency → event store. Razorpay webhooks and merchant-pushed risks |
| 2 · Classifier | Error code → failure taxonomy. Hard declines are abandoned here, before any model is called |
| 3 · Policy agent | XGBoost or LLM → constrained JSON: `retry_now`, `retry_at`, `switch_rail`, `nudge_customer`, `abandon` |
| 4 · Guardrail gate | 12 business rules + schema validation, before execution, all violations reported |
| 5 · Messaging | Scoped LLM generation for the customer nudge; English and Hindi recovery page |
| 6 · Scheduler | 14 sweeps per tick: fire deferrals, reconcile dropped events, expire promises, chase due cases |
| 7 · Receivables | B2B dunning ladder by role, aging, disputes, payment plans, consolidated statements |
| 8 · Voice | Hinglish call handling, 4 gates, promise capture from speech |

The same pipeline also covers money that never reached a gateway — abandoned
carts, halted subscriptions, overdue invoices and failed mandate debits are
POSTed by the merchant to `/risks` and run through the identical agent →
guardrail → payment-link machinery.

## Result

5,000 scenarios × 5 seeds. Reproduce with
`.venv/bin/python -m eval.runner --scenarios 5000 --seeds 5`.

| Policy | Recovery rate | Attempts (avg) | False retries | Net ₹ per ₹1Cr |
|---|---|---|---|---|
| No retry | 0.0% | 0.00 | 0.0% | ₹0 |
| Fixed 3-retry (baseline) | 19.2% ± 0.7 | 2.73 | 8.0% | ₹17,95,744 |
| **AutoRecover · XGBoost** | **30.2% ± 0.5** | **2.32** | **0.0%** | **₹29,66,249** |
| AutoRecover · LLM agent | 26.6% ± 0.5 | 2.38 | 0.0% | ₹25,97,603 |

> **₹11.71 lakh additional net revenue per ₹1 crore of failed volume, at 15%
> fewer retry attempts and zero false retries.**

The difference is **+11.03pp, 95% CI [+10.54, +11.52], n = 25,000**, measured
under common random numbers — every policy sees the identical scenario draw and
outcomes are differenced one-to-one. Overlapping ± ranges cannot answer whether
a difference is real, so the harness does not use them.

No number in the README or in this file was written by hand; every one comes
from `eval/runner.py`.

## What we tell you before you ask

Four things a judge would find anyway, stated up front:

1. **The bank simulator is synthetic.** Its central assumption is that a retry
   is not a fresh payment — what matters is `P(the blocker cleared)`, modelled
   per failure class in `eval/bank_profiles.py`. Blended baseline recovery
   lands at 19.2%, inside the 15–30% band public figures report, and
   `tests/test_calibration.py` fails the build if a change pushes it out.
2. **The LLM lost.** 26.6% against XGBoost's 30.2% on identical scenarios —
   and the ₹ figures do not even charge the LLM for its own inference. Nothing
   in this eval justifies choosing it for this task. It stays in the harness as
   an honest negative result.
3. **The headline is net, not gross.** Recovery rate alone makes brute force
   optimal by construction. Because AutoRecover recovers more while attempting
   fewer, it dominates the baseline at *any* retry cost including ₹0, so the
   cost assumption never has to be argued.
4. **A dead provider cannot fake a row.** The policy agent swallows LLM errors
   and falls back, so a broken API key would otherwise produce a plausible LLM
   row made entirely of XGBoost decisions. The runner counts fallbacks and
   drops the row when they dominate — an earlier revision of this README
   shipped with that row blank for exactly that reason.

## Why it is safe with real money

The first question a fintech panel asks. Full list in
[README.md](README.md#-what-stops-it-from-doing-something-stupid-with-real-money):

- **Hard declines never reach the model** — fraud blocks and stolen cards are
  abandoned at Layer 2.
- **Fixed action space** — five options, validated by Pydantic. No free text.
- **12 business rules, all of them checked** — max 3 retries per payment, 5 per
  customer per rolling 24h, ₹50K amount ceiling, 72h consent window, 2
  nudges/day, time-of-day blackout, expected-value stopping rule, hard-decline
  blocklist, switch-only rule, RBI e-mandate pre-debit notice.
- **No short-circuiting** — every rule runs and every violation is recorded,
  because the audit log is the product.
- **Idempotency enforced before the API call** — deterministic key, UNIQUE
  constraint; a replayed webhook cannot create a second payment link, and the
  race between two concurrent webhooks fails closed.
- **Deferred retries re-validate at fire time** — a customer who opts out at
  23:00 does not get the 02:00 retry that was approved at 22:00.
- **Write-ahead execution** — every attempt is committed `pending` before
  Razorpay is called, so a process death mid-call leaves the money findable.
- **An open dispute freezes the case** — nothing automated touches it until a
  human resolves it.

## Built for Bharat

Indian payments are the design, not a localisation layer.

- **UPI Autopay inside RBI's e-mandate framework.** A promise backed by a
  mandate collects itself on the date the customer named. The guardrail
  enforces the ≥24h pre-debit notice — with no notice on file the debit is
  *turned into* the notice rather than attempted. Above the ₹15,000
  authentication-exempt ceiling the option is never offered, because an
  unattended debit is not lawful there. Tests:
  [tests/test_promise_mandate.py](tests/test_promise_mandate.py).
- **A true Asia/Kolkata clock.** The 11pm–7am contact blackout is computed on a
  real IST clock, not a server offset, and a deferral that would land inside
  the window is shifted forward to its edge before being parked — forward-only,
  because waiting longer is always compliant.
- **Hinglish voice with four gates.** Opt-out honoured first, a retrieval floor
  below which the agent abstains, sanitation of instructions hiding in
  retrieved text, and a grounding check the answer must pass against its cited
  passage. A confident invented number on a call about money is worse than no
  answer.
- **Hindi customer recovery page.** What went wrong in the customer's own
  words, then every way out: pay, promise a date, ask for instalments, say
  something is wrong, or stop hearing from us.
- **Razorpay-native.** Test-mode webhooks and Payment Links end to end.

## Engineering evidence

**1,016 tests across 59 files, 80% statement coverage over `src/`** — with
coverage concentrated where money moves: 100% on the idempotency guard, the
failure taxonomy and the B2B ladder; 98% on the guardrail gate and its rules.
CI is green on `main`, with Gitleaks secret scanning, Dependabot, and mutation
testing in its own workflow.

The README carries a standing **documented failure cases** section and a
hardening log recording defects found and fixed — including the release where
a trained XGBoost model was loaded and then never consulted, and the batch that
silently approved abandons where only CI could see it.

## Try it

No account, no Docker, no internet:

```bash
./run.sh --demo
```

Then open `http://127.0.0.1:8000/demo` — every capability is linked from that
one page. The whole loop runs for real: open a payable link, press Pay, and the
demo checkout sends a genuinely HMAC-signed `payment.captured` to the real
webhook endpoint, which verifies it and attributes the money through the same
code production uses. Exactly one object is faked — the Razorpay SDK client —
and demo mode refuses to start unless `APP_ENV=development`.

Deployment to Render (web services + Postgres) is configured in
[render.yaml](render.yaml); see [docs/DEPLOY.md](docs/DEPLOY.md).

## Status and what is next

**Working now:** the full pipeline against Razorpay test mode, the operations
console, model page, merchant console and customer recovery page, a
reproducible eval harness, and a one-command local demo.

**Next:** a live pilot against real merchant volume — the simulator is a model,
not a measurement; UPI Autopay collection switched on once Razorpay Recurring
Payments is enabled on the account; a real queue in place of the in-process
scheduler past one app process; and an Android merchant app over the same API,
which the server-rendered, mobile-first console is already shaped for.
