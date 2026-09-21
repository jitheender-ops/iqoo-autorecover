# AutoRecover: iQOO Hackathon Edition
*(Track: FinTech and Commerce)*

AutoRecover is a deterministic, AI-powered payment recovery engine built specifically for the diverse and complex Indian payments landscape. 

Failed payments, abandoned checkouts, and missed subscriptions cost Indian merchants millions in lost revenue every month. However, standard debt recovery involves manual follow-ups that scale poorly, alienate customers, and frequently violate compliance regulations (like calling customers at legally protected hours).

**AutoRecover** automates the entire recovery pipeline using deterministic rules, integrating natively with Indian payment gateways (like Razorpay) and executing intelligent, compliance-first recovery actions.

## Built for Bharat
This solution was engineered specifically for the "how India actually transacts" requirement of the iQOO Hackathon's FinTech track:
*   **Hinglish AI Voice Recovery:** AI voice agents that speak the local context (Hinglish/Hindi) to negotiate promises-to-pay.
*   **Native Bilingual Dashboards:** The customer-facing payment portal is fully localized in both English and Hindi.
*   **Regulatory Guardrails:** Hardcoded blackout window enforcement. The engine mathematically guarantees no calls or messages are sent outside legally permitted Indian collection hours (e.g., 09:30 - 18:30 IST).
*   **Razorpay Integration:** Deeply integrated with Indian payment rails for webhooks and payment link generation.

## Features

### 1. The Merchant Console (FastAPI / HTML)
A complete operational control center for merchants to track their receivables.
*   **Ledger Tracking:** View all at-risk and recovered payments.
*   **Safety Center:** An audit trail of every automated action, guaranteeing compliance.
*   **Granular Analytics:** Broken down by payment rails (UPI, Cards, Netbanking).

### 2. Analytical Dashboard (Streamlit)
A high-level view of the recovery funnel, showcasing recovery rates, attribution, and AI agent performance over time.

### 3. Customer Self-Serve
A secure, localized dashboard where customers are routed to seamlessly settle their dues without human intervention.

## Extreme Technical Rigor
AutoRecover is not a weekend prototype. It is built to enterprise standards:
*   **1,000+ Automated Tests:** Guaranteeing deterministic outcomes.
*   **Strict PII Protection:** Hardcoded assertions prevent Personally Identifiable Information (PII) from leaking to unauthorized console pages.
*   **Mypy Strict / Semgrep:** Enforcing rigorous type-safety and code quality.

## Running Locally (Demo Mode)

The project ships with a complete offline demo environment. You don't need Razorpay keys or a Postgres database to evaluate it.

```bash
# Clone the repository
git clone https://github.com/jitheender-ops/iqoo-autorecover.git
cd iqoo-autorecover

# Start the offline demo (seeds synthetic Indian payment data)
./run.sh --demo
```

### Accessing the Interfaces
Once running, you can access:
*   **The Directory:** `http://127.0.0.1:8000/demo` (Links to all views)
*   **The Operational Console:** `http://127.0.0.1:8000/console` (Password: `demo-console-password`)
*   **The Analytical Dashboard:** `http://127.0.0.1:8501` (Password: `demo-console-password`)

---
*Built by jitheender-ops for the iQOO Hackathon 2026.*
