# CFOup — MVP Execution PRD
**Version:** 1.0 | **Date:** April 2026 | **Status:** Active  
**For:** Cursor + Matheus (dev execution)

---

## The One Question This MVP Must Answer

> **"Quanto dinheiro vou ter daqui a 90 dias?"**

Every feature decision flows from this. If it doesn't help answer this question for the pilot company, it's not in the MVP.

---

## What's In. What's Out.

| IN — Build Now | OUT — Roadmap v2 |
|---|---|
| Onboarding (company profile) | AI chat assistant |
| Bank statement upload (PDF/Excel) | Stripe billing |
| Transaction classification | Multi-plan pricing |
| Cash dashboard (4 core metrics) | Multi-company support |
| 90-day cash forecast | Break-even modeling |
| 3–5 hardcoded alerts | Accountant access |
| | API bank integration (Open Finance) |
| | Mobile app |

---

## Users

**Primary (MVP):** Ronaldo (proxy for pilot company owner)  
**Secondary:** Matheus (admin — can see all companies for debugging)

No other user types in V1.

---

## Milestones

### M1 — Shell (3–5 days)
**Done when:** You can log in, see an empty dashboard, and log out.

- [ ] Supabase project created
- [ ] Auth: email/password (Supabase Auth)
- [ ] Next.js app deployed on Vercel
- [ ] Route protection (redirect to login if unauthenticated)
- [ ] Company profile form (name, sector, employees, monthly revenue range)
- [ ] Company saved to Supabase

**DB tables needed:** `users`, `companies`

---

### M2 — Data Ingestion (5–7 days)
**Done when:** A bank statement file is uploaded and raw transactions appear in the DB.

- [ ] File upload UI (drag-and-drop or button)
- [ ] Accept: PDF and Excel (.xlsx)
- [ ] Store original file in Supabase Storage
- [ ] Parse PDF → extract rows (date, description, amount)
- [ ] Parse Excel → same output
- [ ] Save raw transactions to DB with status `unclassified`
- [ ] Show upload history (filename, date, row count)

**DB tables needed:** `bank_accounts`, `transactions`  
**Note:** Don't build a perfect parser. Build one that works for the pilot company's bank format (Itaú or Bradesco — confirm with pilot).

---

### M3 — Classification (4–6 days)
**Done when:** Every uploaded transaction has a category, and the owner can correct it.

- [ ] Connect classification engine (already built: `classificacao_financeira_padrao_v1`)
- [ ] Auto-classify on upload
- [ ] Classification review screen: table of transactions with category shown
- [ ] Owner can change category via dropdown
- [ ] Manual change saves to DB with `source = 'manual'`
- [ ] Recalculate dashboard metrics after any manual change

**Categories (MVP):**  
`Receita` | `Custo de Mercadoria` | `Pessoal` | `Despesas Operacionais` | `Impostos` | `Financiamento` | `Retirada do Sócio` | `Transferência Interna` | `Não Classificado`

---

### M4 — Dashboard (5–7 days)
**Done when:** Owner sees 4 numbers that matter, updated in real time after any classification change.

#### The 4 Core Metrics

| Metric | Formula | Display |
|---|---|---|
| **Caixa Atual** | Sum of all transactions (last known balance) | R$ 142.300 |
| **Receita do Mês** | Sum of `Receita` transactions in current month | R$ 87.500 |
| **Saída do Mês** | Sum of all non-revenue, non-transfer transactions in current month | R$ 61.200 |
| **Saldo Projetado (30d)** | Caixa Atual + avg monthly net × 1 | R$ 168.600 |

#### Supporting Charts
- Bar chart: monthly revenue vs expenses (last 6 months)
- Simple cash flow line: actual + 90d projection

**UI principle:** Mobile-first. One screen. No tabs. Numbers big enough to read on a phone.

---

### M5 — 90-Day Forecast (3–4 days)
**Done when:** Owner sees a projected cash position for next 30 / 60 / 90 days.

**Algorithm (MVP — keep it simple):**
1. Calculate average monthly net cash flow (last 3 months of data)
2. Project forward: `Caixa Atual + (avg_net × N_months)`
3. Show as 3 cards: 30d / 60d / 90d

**No ML. No seasonality. Just averages.** Add sophistication post-pilot based on feedback.

---

### M6 — Alerts (2–3 days)
**Done when:** 3 alerts fire correctly and appear in the dashboard.

| Alert | Trigger | Message |
|---|---|---|
| Caixa crítico | Projected 30d balance < 0 | "Atenção: seu caixa pode ficar negativo em 30 dias" |
| Queda de receita | Current month revenue < 70% of prior month | "Receita este mês está 30% abaixo do mês anterior" |
| Saída incomum | Any single category spikes > 40% vs prior month | "Despesas com [categoria] aumentaram 40% este mês" |

Alerts appear as a banner on the dashboard. No email in MVP. In-app only.

---

### M7 — Pilot Deploy (2–3 days)
**Done when:** Pilot company owner can use the app on their phone without help from Matheus.

- [ ] Deploy to Vercel (production URL)
- [ ] Supabase RLS policies — each company only sees their own data
- [ ] Create account for pilot company
- [ ] Upload their real bank statement
- [ ] Validate classifications with Ronaldo
- [ ] Fix obvious UX friction from first real use session
- [ ] Document 3 things to improve (input for v1.1)

---

## Database Schema (MVP Only)

```sql
-- users (handled by Supabase Auth)

-- companies
create table companies (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users not null,
  name text not null,
  sector text,
  employees_range text, -- '1-10', '11-30', '31-100'
  monthly_revenue_range text, -- '<50k', '50-200k', '200k-1M', '>1M'
  country text default 'BR',
  created_at timestamptz default now()
);

-- bank_accounts
create table bank_accounts (
  id uuid primary key default gen_random_uuid(),
  company_id uuid references companies not null,
  institution text,
  account_type text default 'checking',
  file_name text,
  file_url text,
  row_count int,
  created_at timestamptz default now()
);

-- transactions
create table transactions (
  id uuid primary key default gen_random_uuid(),
  bank_account_id uuid references bank_accounts not null,
  company_id uuid references companies not null,
  date date not null,
  description text,
  amount numeric not null, -- positive = credit, negative = debit
  category text default 'Não Classificado',
  classification_source text default 'ai', -- 'ai' | 'manual'
  created_at timestamptz default now()
);

-- Row Level Security (enable on all tables)
-- Policy: users can only access rows where company.user_id = auth.uid()
```

---

## Definition of Done (Global)

A feature is done when:
1. It works on mobile (375px width)
2. It works with real data from the pilot company's bank statement
3. Ronaldo can use it without explanation
4. No console errors in production

---

## Tech Constraints

- **No new dependencies without discussion.** Keep the stack lean.
- **Supabase Edge Functions** for anything that runs server-side (classification, forecast calc)
- **No hardcoded secrets** — all via env vars
- **Portuguese only** — all UI copy in PT-BR

---

## Post-MVP Roadmap (Don't Build Now)

1. AI chat (Claude API with financial context)
2. Stripe billing + plan management
3. Open Finance / API bank integration
4. Break-even and margin modeling
5. Accountant view (read-only)
6. Email alerts
7. Mobile app (React Native or PWA)
8. Multi-company support

---

*CFOup — Execution PRD v1.0 — April 2026*
