# Zava Retail Analysis: Findings and Agent Candidates

This document summarises the seven analysis questions in `all_7_analysis_db.ipynb`, the answer to each (with the evidence behind it), and the problems the data supports that suit an AI **agent** rather than a chatbot. Each problem was tested against the live `zava` PostgreSQL database, and all figures come from it.

**How to read the key terms**

| Term | Meaning |
|---|---|
| Units | Individual items sold or held, not dollars. |
| Seasonal swing | Gap between a category's busiest and quietest month, relative to its average month. A swing of 0.948 means the busiest month is about 95 percentage points above the quietest one. |
| Months of supply | How many months current stock would last at the recent sales rate. |
| RLS (row-level security) | A database rule that limits each manager to the rows for their own store. |

---

## Summary at a glance

| # | Question | Headline answer |
|---|---|---|
| 1 | Which categories are most seasonal? | **Garden & Outdoor** swings most (0.948), peaking in **April** and bottoming in **December**. |
| 2 | Which stores diverge from national seasonality? | **Kirkland, Everett, Redmond and Spokane** diverge most. **Online and Seattle** track the national pattern most closely. |
| 3 | What actually happened in 2023? | 91,301 units sold. Online and Seattle account for 56% of units. Monthly totals are flat, but individual categories are not. |
| 4 | What products are bought together? | National top pairs are all hand tools and occur about 9 times more often than chance. Store-level differences are too small to separate from noise. |
| 5 | Where is inventory misaligned with demand? | Stock mirrors customer headcount, not sales. **Tacoma** holds the most excess; **Seattle and Online** are the most under-stocked relative to demand. |
| 6 | Which products have the widest inventory-to-sales gap? | All top 20 products hold **over 5 years of stock** at the recent sales rate. |
| 7 | How does a store manager's view differ from the super manager's? | A store manager sees **1.5% to 28.4%** of orders, depending on the store. Security rules work when the restricted role is used. |

---

## Question 1: Seasonal demand patterns

**Answer.** Garden & Outdoor has the sharpest seasonal swing, peaking in April and lowest in December.

| Category | Seasonal swing | Peak month | Lowest month |
|---|---|---|---|
| Garden & Outdoor | 0.948 | Apr | Dec |
| Storage & Organization | 0.891 | Feb | Jun |
| Lumber & Building Materials | 0.621 | Jun | Dec |
| Electrical | 0.552 | Dec | Apr |
| Hardware | 0.547 | Oct | Jul |
| Power Tools | 0.515 | Aug | Dec |
| Plumbing | 0.435 | Oct | Aug |
| Paint & Finishes | 0.400 | Mar | Nov |
| Hand Tools | 0.233 | Jul | Oct |

**Why we trust it.** Each complete calendar year is normalised to that year's category average before the years are averaged, so growth between years does not distort the result. The SQL result was independently recomputed in pandas and matched exactly.

**Consistency.** Garden & Outdoor peaked in April in every year in the data (2020 to 2026). Paint & Finishes peaked in March in 6 of the 7 years, Storage & Organization in February in 6 of 7, and Electrical in December in 5 of 7. Hand Tools, Plumbing and Power Tools move around from year to year, so their peaks are less reliable.

**Limit.** The database has **no supplier lead-time data**, so it shows when demand peaks but not how far ahead to order.

## Question 2: Store alignment to national seasonality

**Answer.** Four large stores follow the national pattern closely; four small stores deviate by three to five times as much.

| Store | Deviation from national pattern (lower is closer) | Share of units sold |
|---|---|---|
| Online | 0.032 | 28.4% |
| Seattle | 0.034 | 27.5% |
| Bellevue | 0.042 | 18.4% |
| Tacoma | 0.054 | 13.3% |
| Spokane | 0.096 | 5.0% |
| Redmond | 0.120 | 2.6% |
| Everett | 0.122 | 3.4% |
| Kirkland | 0.152 | 1.5% |

**Limit.** The stores that deviate most are also the smallest. Part of the deviation may be random noise from low volumes rather than a genuine local seasonal pattern, so treat it as a signal to investigate, not proof.

## Question 3: 2023 sales snapshot

**Answer.** 91,301 units were sold in 2023. Online (26,155 units) and Seattle (25,052) together account for about 56% of units.

| Store | 2023 units | 2023 revenue |
|---|---|---|
| Online | 26,155 | 1,771,472 |
| Seattle | 25,052 | 1,700,747 |
| Bellevue | 16,804 | 1,180,296 |
| Tacoma | 12,313 | 844,883 |
| Spokane | 4,462 | 318,811 |
| Everett | 3,085 | 199,467 |
| Redmond | 2,159 | 149,597 |
| Kirkland | 1,271 | 83,909 |

- **Monthly totals are flat.** The whole network sold between 7,348 units (November) and 7,976 (July) per month. The category-level swings in Question 1 largely cancel each other out at network level, so the seasonality only shows when you look by category.
- **Units and revenue rank differently.** Paint & Finishes sold the most units (11,946) but Power Tools earned the most revenue (2,254,990).

## Question 4: Product affinity and cross-sell

**Answer.** Nationally, the most common pairs are hand tools bought with other hand tools, and the pattern is real and stable. The store-level "top pairs" in the notebook's matrix are too thin to treat as genuine local differences.

- **National top pairs** (orders containing both): Level 24-inch + SAE Hex Key Set (86), Ratcheting Screwdriver + Locking Pliers (84), Claw Hammer + Needle-Nose Pliers (83).
- **The national signal is real.** The top pair appears in 86 orders, where independent buying would give about 10, so roughly 9 times more often than chance. Pair counts from even-numbered and odd-numbered years correlate at 0.91, so the pattern is stable. All of the 100 most common pairs are within Hand Tools.
- **Local differences are not supported.** Of the 300 most common pairs, 5.3% have a store mix that differs from each store's share of orders, and 6.0% have a different seasonal mix. Chance alone would produce about 5%.
- **Why the store matrix looks different.** Each store's leading pair in a season rests on very few orders: 7 to 12 in the four largest stores and 3 to 6 in Spokane, Redmond, Everett and Kirkland, often tied with other pairs. For example, Kirkland's spring leader (Underground Wire 12-2 + Rigid Conduit 1-inch) rests on 3 orders.
- **Genuine seasonality.** Garden & Outdoor pairs do cluster in spring and summer (36% and 29% of their orders, against 25% each for all orders). That reflects the category's seasonal demand in Question 1, not a store-specific preference.

**Limit.** The notebook's top-3 table ranks pairs by count only, so ties are broken arbitrarily. Do not build store-specific bundles from it.

## Question 5: Inventory misalignment by store

**Answer.** Stock is not distributed in line with demand. It closely mirrors how many customers each store is assigned, while sales depend on how often those customers buy.

| Store | Share of stock | Share of assigned customers | Share of units sold | Orders per buying customer |
|---|---|---|---|---|
| Online | 23.3% | 23.1% | 28.4% | 4.87 |
| Seattle | 23.1% | 22.7% | 27.5% | 4.76 |
| Bellevue | 18.2% | 19.3% | 18.4% | 3.79 |
| Tacoma | 16.0% | 15.5% | 13.3% | 3.46 |
| Spokane | 6.2% | 6.2% | 5.0% | 3.21 |
| Everett | 5.4% | 5.5% | 3.4% | 2.48 |
| Redmond | 4.6% | 4.7% | 2.6% | 2.23 |
| Kirkland | 3.1% | 3.0% | 1.5% | 1.93 |

- **Most over-stocked relative to demand (2025):** Tacoma in 7 of 9 categories, Redmond in the other 2 (Paint, Electrical).
- **Most under-stocked relative to demand (2025):** Seattle in 5 categories, Online in 3, Bellevue in 1.
- **Example:** Tacoma holds 54,434 units of Hand Tools against 1,473 sold in 2025.

**Important limits**

- The inventory table has **no date**, so the same current stock snapshot is compared against each completed year of sales.
- A negative gap means "less stock than a demand-proportional share", **not** a stock-out. Every store holds far more stock than it sells (see Question 6).
- Suggested transfers between store pairs are **alternatives, not additive**. The same surplus cannot be moved twice.

## Question 6: Widest gap between inventory and sales velocity

**Answer.** Stock on hand is far larger than sales across the whole network. The 20 worst products all hold **over 5 years of stock** at the recent sales rate.

| Product | Units in stock | Units sold, last 12 months | Sales speed | Stock coverage |
|---|---|---|---|---|
| Natural Bristle Brush Set | 15,225 | 262 | About 21.8 units per month | Over 5 years (roughly 58 years) |
| Heavy Duty Teflon Tape | 13,805 | 241 | About 20.1 units per month | Over 5 years |
| Exterior Primer-Paint Combo | 14,293 | 254 | About 21.2 units per month | Over 5 years |
| Grease Cap Wire Nuts | 14,509 | 260 | About 21.7 units per month | Over 5 years |
| OSB Siding Panel | 11,735 | 211 | About 17.6 units per month | Over 5 years |

- **Network-wide:** 3,878,342 units are in stock, about **5.7 times** every unit ever sold in the order history (675,926).
- **Reading "21.8":** it means about 21.8 units sold per month on average, calculated from the latest 12 months of orders.
- **It is not just the top 20.** Across all 3,392 product-store stock records, none holds less than about 37 months of stock at 2025 sales, and 99% hold more than 5 years. 47 records hold stock but had no 2025 sales.
- **Value.** Stock is worth about 190.6 million at product cost, against about 5.2 million of cost of goods sold in 2025.

**Limit.** A gap this large may be a genuine over-buying problem, or it may mean stock is recorded in different units to sales (for example cases versus single items). Confirm with the inventory owner before acting. The "latest 12 months" window ends at the latest order date in the data (31 December 2026), so it includes orders dated after the database's current date (see Problem 2 below).

## Question 7: Super manager versus store manager

**Answer.** A store manager sees only their own store. The super manager sees everything.

| View | Orders visible | Share of all orders | Units in stock | Units sold | Matches direct check |
|---|---|---|---|---|---|
| Super manager (all stores) | 198,783 | 100.0% | 3,878,342 | 675,926 | Yes |
| Online | 56,467 | 28.4% | 902,892 | 191,905 | Yes |
| Seattle | 54,579 | 27.5% | 897,648 | 186,034 | Yes |
| Bellevue | 36,502 | 18.4% | 705,678 | 124,511 | Yes |
| Tacoma | 26,426 | 13.3% | 619,160 | 89,735 | Yes |
| Spokane | 9,934 | 5.0% | 241,358 | 33,450 | Yes |
| Everett | 6,718 | 3.4% | 211,113 | 22,770 | Yes |
| Redmond | 5,189 | 2.6% | 180,179 | 17,383 | Yes |
| Kirkland | 2,968 | 1.5% | 120,314 | 10,138 | Yes |

- Every store row was checked against a separate direct filter on that store and matched.
- Customer counts overlap between stores, because a manager sees customers assigned to the store and customers who ordered there. They do not add up to the 50,000 network total.

**Security finding.** The notebook's default login (`postgres`) is a superuser that **bypasses** row-level security. Any app or agent that should be store-scoped must connect as a restricted role. The notebook switches to the `store_manager` role to show the real store-level view. Separately, the `store_manager` role currently has no working password, so direct logins with the documented password fail.

---

## Validation: are the problems real?

The three problems you shared were treated as hypotheses and tested against the database. Two were only partly supported and one was not supported.

| Hypothesis | Supported by the data | Not supported by the data | Verdict |
|---|---|---|---|
| Predictive procurement without lead-time data | Seasonality is real and repeatable: Garden & Outdoor peaked in April in every year from 2020 to 2026. The database has no supplier, lead-time or reorder fields. | A shortage risk. Garden & Outdoor stock is 408,212 units against 1,450 sold in its busiest month of 2025, about 280 peak months of stock. | Partly supported. Kept as a conditional problem (Problem 3). |
| Cross-store inventory rebalancing | Stock is allocated unevenly: coverage ranges from 358 months (Online) to 857 months (Kirkland), and stock shares mirror customer shares rather than sales. | Stores running out of stock. No product-store record has zero stock, and the lowest coverage is about 37 months. An Electrical surplus in February has no support, because Electrical peaks in December. | Partly supported. Reframed as overstock and allocation triage (Problem 1). |
| Role-aware local merchandising | Role scoping works as designed (Question 7). National pairings are real: top pairs occur about 9 times more often than chance and are stable across years. | Local pairings that differ from national ones. Only 5.3% of the top 300 pairs have a store mix that differs from the order mix, which is what chance alone gives. Small-store "top pairs" rest on 3 to 6 orders. | Not supported as a data problem. Removed. Role scoping stays as a safeguard for every agent. |

**How this was tested.** Peak months were compared year by year, stock coverage was calculated for every product-store record, pair counts were compared with what chance would produce, and a simulation tested whether store and season mixes differ from the overall order mix.

## Problems the data supports

Each problem below needs an AI **agent**, meaning a system that monitors data, runs several steps with tools, and prepares actions for a person to approve. A chatbot can explain the numbers above but would not watch the data, do the multi-step work, or produce a ready-to-approve output. Each problem also notes which part could be handled without an agent.

### 1. Inventory overstock and allocation triage

- **Problem.** Stock is far larger than sales everywhere, and it is spread unevenly between stores.
- **Evidence (Questions 5 and 6).**
  - All 3,392 product-store stock records hold at least about 37 months of stock at 2025 sales, and 99% hold more than 5 years. No record has zero stock, and 47 hold stock with no 2025 sales.
  - The network holds 3,878,342 units against 107,954 sold in 2025, about 36 times a year of sales. At product cost the stock is worth about 190.6 million, against about 5.2 million of cost of goods sold in 2025.
  - Coverage differs by store: Online 358 months, Seattle 365, Bellevue 418, Tacoma 525, Spokane 531, Everett 646, Redmond 781, Kirkland 857. Stock shares follow customer shares, not sales.
- **Agent solution: Inventory Triage Agent.** For each product and store it calculates coverage, then classifies the record as a likely unit or data error, real overstock, or misallocated stock. It recommends options (hold new purchases, transfer between stores, review pricing), drafts them for approval, and reruns on a schedule.
- **Why not a chatbot.** It must evaluate thousands of product-store combinations, avoid counting the same surplus twice, and keep monitoring. A chatbot can only answer "which products are overstocked" when asked.
- **Without an agent.** A scheduled report can list the overstocked products. The agent adds classification, option comparison and follow-up.
- **Safeguards and gaps.** The first step is confirming that stock and sales use the same units. The data has no transfer costs, safety-stock rules or stock history. Nothing should be moved or cancelled without human approval.

### 2. Data integrity and reconciliation

- **Problem.** Some of the data used for decisions looks wrong or incomplete, and nothing currently checks for it.
- **Evidence.**
  - 8,172 orders (4.1%) are dated after the database's current date of 6 October 2026, running from 7 October to 31 December 2026. The "latest 12 months" window in Question 6 ends at the latest order date, so it includes them.
  - The stock-to-sales gaps in Questions 5 and 6 are so large that they may come from stock and sales being recorded in different units.
  - Fields needed for planning are missing: supplier, lead time, reorder point, safety stock, and a date on inventory records.
  - The notebook's default login bypasses row-level security, and the `store_manager` role has no working password (Question 7).
- **Agent solution: Data Reconciliation Agent.** On each data refresh it runs these checks, traces any failure across the orders, items, products and inventory tables, classifies the likely cause, and drafts an issue for the data owner. It can also hold back downstream recommendations, such as Problem 1, until the checks pass.
- **Why not a chatbot.** The work is a recurring investigation across several tables, followed by a decision on whether results can be trusted.
- **Without an agent.** Detection alone is a set of scheduled SQL tests. The agent adds cross-table investigation and issue drafting.
- **Safeguards.** It should be read-only on source data and connect as a restricted role.

### 3. Seasonal buy planning (conditional)

- **Problem.** Demand peaks are predictable, but nothing links them to ordering deadlines, and the data has no lead times.
- **Evidence (Questions 1 and 3).** Garden & Outdoor peaked in April in every year in the data, Paint & Finishes in March in 6 of 7 years, and Storage & Organization in February in 6 of 7. The database has no supplier or lead-time fields.
- **Why it is conditional.** There is no purchasing risk today: Garden & Outdoor stock covers about 280 peak months of demand. This becomes a real problem only after Problems 1 and 2 are resolved and stock levels are credible.
- **Agent solution: Supply Chain Forecasting Agent.** The agent takes each category's peak month plus lead-time assumptions the user supplies (for example "120 days of ocean freight"), works backward to an order deadline, and alerts buyers before it passes.
- **Worked example.** With a 120-day lead time and an April peak, the order is due by about 2 December (for a 1 April start) or 16 December (for mid-April), so a November alert leaves a few weeks of buffer.
- **Why not a chatbot.** It must run on a schedule for every category, recompute as stock changes, and alert buyers unprompted.
- **Safeguards.** Lead times are assumptions and are shown with every recommendation. The agent drafts a calendar and does not place orders.

### Not an agent problem: product affinity

Question 4 found a real national pattern: hand tools bought together about 9 times more often than chance. That is a finding for a scheduled report or a simple recommender, not a problem that needs an agent. The data does not show store-level differences that would justify a store-specific agent.

### Safeguard for every agent: role-scoped access

Question 7 shows the database enforces store scoping only for restricted roles. Any agent that serves store managers must connect as such a role, rather than relying on its own filtering.

### How the problems relate to the seven questions

| Question | 1. Inventory triage | 2. Data reconciliation | 3. Seasonal buy planning (conditional) |
|---|---|---|---|
| 1. Seasonal patterns | | | Primary |
| 2. Store alignment | Supporting | | |
| 3. 2023 snapshot | | | Supporting |
| 4. Product affinity | Finding only, not an agent problem | | |
| 5. Inventory by store | Primary | Supporting | |
| 6. Inventory vs sales velocity | Primary | Supporting | |
| 7. Super vs store manager | | Supporting | |

---

## What the data cannot tell us

- Supplier lead times, transfer costs and safety-stock targets are not in the database.
- Inventory has no history, only a current snapshot.
- Stock and sales units may not be comparable, so the very large stock-to-sales gaps need confirming.
- 8,172 orders are dated after the database's current date, so any "latest 12 months" figure includes future-dated rows.
- Small stores have low volumes, so their seasonal deviations and product pairings are weaker signals.
