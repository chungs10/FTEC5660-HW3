# FTEC5660 Homework 3: Fisher's 15-Point Investment Agent

Build a LangChain agent that researches a company with tools and applies the
framework from Philip Fisher's *Common Stocks and Uncommon Profits* (Part One,
Chapter 3 "What to buy: the fifteen points to look for in a common stock", and
Chapter 4 "What to buy: applying this to your own needs"). You choose 3 of
Fisher's 15 points and decide yourself what PASS means for each. For every
company your agent gives a PASS/FAIL verdict on your 3 points and a short
summary.

The companies are fictional, so the model cannot answer from memory. Unlike a
pile of reports, the evidence is spread over three sources, and your agent has
to go and get it with tools, the way Fisher's "scuttlebutt" method sends an
investor to talk to customers, competitors, suppliers and staff.

## Data and tools

`public_test/` has one folder per company. Your code receives four tools per
company (built by the provided code in `hw3.py`):

| Tool | What it gives you |
| --- | --- |
| `list_documents()` | the company's six documents and their dates |
| `read_document(name)` | one document: the 2025 letter to shareholders, four earnings calls (Q3 2025 to Q2 2026) and a news file (July 2025 to July 2026) |
| `sql_query(query)` | one read-only SQLite query on the company's database; at most 50 rows come back, so use `SUM`, `GROUP BY` and `WHERE` |
| `interview(role, topic)` | what one person says about the company; at most **10 interviews per company** |

The documents are what management says. Management's own claims (such as "we
have never cut research" or "our people are our greatest strength") are not
evidence on their own: check them against the database and the interviews.

**Database tables** (amounts in USD thousands; dates `YYYY-MM-DD`, months
`YYYY-MM`, quarters like `2025Q3`; the fiscal year is the calendar year):

| Table | Columns |
| --- | --- |
| `company` | name, industry |
| `industry_medians` | industry, revenue_growth_2y_pct, new_product_share_pct, new_product_sales_per_rd, operating_margin_pct (one row per industry; use the row for the company's own industry) |
| `orders` | order_id, order_date, product, customer, amount, note (about 3,000 to 4,000 orders from January 2023 to June 2026; revenue is the sum of order amounts) |
| `products` | product, launch_date |
| `monthly_costs` | month, cost_of_sales, selling_admin, research_development |
| `quarterly_cash` | quarter, operating_cash_flow, capex, cash_end |
| `capex_plan` | year, planned_capex (2026 to 2028) |
| `returns` | month, units_shipped, units_returned |
| `insider_trades` | trade_date, person, title, side (`buy` or `sell`), shares, price |

**Terms used in the data.**

| Term | Meaning |
| --- | --- |
| FY2025 | Fiscal year 2025. Here the fiscal year is the calendar year, so FY2025 is January to December 2025. |
| Quarter | Three months of the year: Q1 is January to March, Q2 April to June, Q3 July to September and Q4 October to December. The database writes `2025Q3`; the documents write "third quarter 2025". |
| Amounts | The database is in **USD thousands** (676,100 means US$676.1 million). The documents use USD millions. |
| Revenue | Money from sales: the sum of `amount` in `orders`. |
| Cost of sales | The cost of making the products that were sold. |
| Selling and admin | The cost of selling and of running the company (sales staff, offices, management). |
| R&D | Research and development spending, mainly on new products. |
| Operating margin | Operating profit as a share of revenue: (revenue − cost of sales − selling and admin − R&D) ÷ revenue. |
| Operating cash flow | Cash that the day-to-day business brings in during a quarter. |
| Cash at the end of a quarter | The cash the company holds on the last day of that quarter (`cash_end`). |
| Capex | Capital expenditure: spending on factories, machines and other long-term assets. `capex_plan` holds the planned capex for 2026 to 2028. |
| Return rate | Units returned ÷ units shipped, from the `returns` table. |
| Industry median | The middle value for companies in the same industry, from `industry_medians`. |
| Insider | A director or senior manager of the company. `insider_trades` lists the shares they bought (`buy`) or sold (`sell`). |
| Earnings call | A meeting each quarter where management presents the quarter's results to investors. Each call quotes that quarter's revenue. |

**Interviews.** Roles: `customer`, `current_employee`, `former_employee`,
`competitor`, `supplier`. Topics: `sales_team`, `working_conditions`,
`top_management`, `decision_making`, `competitive_edge`. Only some people know
about each topic; the others say they can't comment, and that answer still
uses up one interview.

| Topic | Who knows about it |
| --- | --- |
| `sales_team` | customer, competitor |
| `working_conditions` | current_employee, former_employee |
| `top_management` | former_employee, supplier |
| `decision_making` | supplier, customer |
| `competitive_edge` | competitor, customer |

People who know about a topic always agree with each other, so one good answer
is enough.

## Choose your 3 points and your rules

Pick any 3 of Fisher's 15 points and write them in `CHOSEN_POINTS` near the
top of `hw3.py`, e.g. `CHOSEN_POINTS = [1, 7, 15]`. The full questions are in
`FISHER_POINTS` in `hw3.py` and in the book.

| # | Fisher's point | Where the evidence is |
| --- | --- | --- |
| 1 | Market potential for sales growth | database |
| 2 | New products as current lines mature | database |
| 3 | Effective research and development | database |
| 4 | Above-average sales organization | interviews (`sales_team`) |
| 5 | Worthwhile profit margin | database |
| 6 | Maintaining or improving margins | database |
| 7 | Labour and personnel relations | interviews (`working_conditions`) |
| 8 | Executive relations | interviews (`top_management`) |
| 9 | Depth of management | interviews (`decision_making`) |
| 10 | Cost analysis and accounting controls | earnings calls, checked against the database |
| 11 | Advantage over competitors | interviews (`competitive_edge`) |
| 12 | Long-range outlook on profits | database |
| 13 | Growth without dilutive equity financing | database |
| 14 | Management talks freely about troubles | earnings calls, checked against the database |
| 15 | Management of unquestionable integrity | news file, checked against the database |

For each of your points, **you design the rule**: which evidence to use, what
to compute or whom to ask, and where the line between PASS and FAIL lies. PASS
means the company looks favourable on that point, as Fisher would see it.

We grade your verdicts against our own answer key. It uses one fixed rule per
point, which we do not publish, and the data never puts a company close to
the line of that rule, so a sensible reading of Fisher usually gives the same
answer. **Test your rules on the public answers**:
`public_test/ground_truth.json` gives the expected verdict on all 15 points
for each of the 20 public companies, and every point fails in 6 of them. If
your rule disagrees with the public answers, rethink it.

**Tip: calculate with tools, not in your head.** Do every sum, ratio and
comparison in SQL or Python, including the number of days between two dates
(SQLite's `julianday()` gives a date as a number of days).

## Student task

Fill in the three places in `hw3.py` marked `### YOUR CODE HERE`. You may add
imports, constants and helper functions above them, but do not change the
provided code below them:

- `CHOSEN_POINTS` lists your 3 points.
- `build_agent()` creates your model and anything else you need once.
- `analyze_companies(agent, companies)` receives
  `{company_folder: [list_documents, read_document, sql_query, interview]}`
  and returns, for every company,
  `{"points": {1: "PASS", 7: "FAIL", 15: "PASS"}, "summary": "..."}` (here with
  `CHOSEN_POINTS = [1, 7, 15]`).

Use the course techniques that fit your points: tool use for the database and
the interviews, planning which interviews to spend, routing each point to the
right source, prompt chaining, parallel calls, reflection to check your
answers, or a combination. Your code must read company data **only through
the four tools**; do not open `company.db` or `interviews.dat` yourself. Do
not hard-code folder names, company names or the public answers.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw3.py --data-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com). The
required model is `deepseek-v4-flash`. We grade with our own key, so never put
a key in your code; `.env` must stay out of git.

The program creates `results.csv` with the columns `company`, `points`,
`points_correct`, `point_errors`, `interviews_used` and `summary`, and prints
the point score on your 3 points (the percentage after "balanced"). The point
score is the average of two shares: the share of truly passing verdicts that
you marked PASS and the share of truly failing verdicts that you marked FAIL.
For example, 90% and 60% give (90% + 60%) / 2 = 75%, and marking every point
PASS gives (100% + 0%) / 2 = 50%. Once `CHOSEN_POINTS` is set, the starter
runs before you add any API code (it returns no verdicts, so it scores 0%).

`public_test/ground_truth.json` gives the expected verdicts on all 15 points
for each public company. Grading uses a private set of about 20 unseen
companies with the same tools and the same answer key rules, but different
names and wording, and scores your 3 points only.

## Homework 3 solution:
> to students: this is your report, see the homework description.

### My 3 points and rules
> for each point: why you chose it, your PASS rule in one or two sentences, and the evidence it uses. Quote Fisher where it helps.


**Point 1 — Market potential for sales growth.** Fisher asks: *"Does the company have products or services with sufficient market potential to make possible a sizable increase in sales for at least several years?"*
I chose this question because it is essential for investors to know the future success of the company and not a one-off thing that they created to sell in a heartbeat. It can also be computed using only the database and does not require the LLM for any sort of arithmetic, making it easily quantifiable with no hallucinations.
The PASS rule is using the 2-year revenue growth between the fiscal years and checking it against the industry median in the database.
Evidence used is in the SQL: checking the revenue per year, calculating the 2-year growth, then putting that number against the `industry_medians` using SQL queries.

**Point 5 — Worthwhile profit margin.** Fisher asks: *"Does the company have a worthwhile profit margin?"*

I chose this question because it is essential for investors to know how worthwhile the investment must be in order to gauge their amount of investment as well. It can also be computed using only the database and does not require the LLM for any sort of arithmetic, making it easily quantifiable with no hallucinations.
The PASS rule is computing the 2025 operating margin by summing costs (cost of sales, R&D, and selling administration) from `monthly_costs`, subtracting them from the revenue in `orders`, and checking the operating margin against the industry median operating margin in `industry_medians`.
Evidence used is in the SQL: `monthly_costs` for cost of sales, selling administration, and research development, `orders` for revenue, and `industry_medians` for the comparison.

**Point 4 — Above-average sales organization.** Fisher asks: *"Does the company have an above-average sales organization?"*

I chose this question because you need to know the state of the sales organization, and numbers alone won't tell you what's happening behind the scenes. You need to hear from customers and competitors whether the team has the edge or not, and that's judged on sentiment. This requires an LLM, but it's not arithmetic — it's looking at the words themselves. It uses multiple chains because judging an interview is more elaborate than computing a number.
The PASS rule is: PASS only if the sentiment of a customer's or competitor's quote about `sales_team` is positive. Neutral and negative both fail, because Fisher asks for "above-average" — "fine" doesn't clear the bar.
Evidence used is the `interview` tool (not SQL): the routing chain picks a role (customer or competitor), the interview tool fetches one quote on topic `sales_team`, and the sentiment chain classifies it as positive, negative, or neutral.

### Design
> a visualization of your agent design and one paragraph describing it.

```
                          ┌──────────────────────────────┐
                          │  build_agent()  (called once) │
                          │  ─────────────────────────── │
                          │  llm = ChatDeepSeek(          │
                          │    model="deepseek-v4-flash", │
                          │    temperature=0)             │
                          │                               │
                          │  interview_chain  (LLM)       │
                          │  sentiment_chain  (LLM)       │
                          │  summary_chain    (LLM)       │
                          └──────────────┬───────────────┘
                                         │
                                         ▼
              ┌──────────────────────────────────────────────────┐
              │  analyze_companies(agent, companies)             │
              │  20 companies in parallel (Semaphore = 5)        │
              │                                                   │
              │  For each company:                                │
              │  ┌─────────────────────────────────────────────┐ │
              │  │ Point 1 (SQL + Python)                      │ │
              │  │   sql_query("SELECT revenue by year")       │ │
              │  │   → parse_sql → growth = (25-23)/23 * 100   │ │
              │  │   → compare to industry_medians → PASS/FAIL │ │
              │  └─────────────────────────────────────────────┘ │
              │  ┌─────────────────────────────────────────────┐ │
              │  │ Point 5 (SQL + Python)                      │ │
              │  │   sql_query("SELECT costs 2025")            │ │
              │  │   → parse_sql → margin = (rev-costs)/rev    │ │
              │  │   → compare to industry_medians → PASS/FAIL │ │
              │  └─────────────────────────────────────────────┘ │
              │  ┌─────────────────────────────────────────────┐ │
              │  │ Point 4 (LLM chains)                        │ │
              │  │   interview_chain  → {"role": "competitor"}  │ │
              │  │   interview tool   → one quote              │ │
              │  │   sentiment_chain  → {"sentiment": "neg..."}│ │
              │  │   positive ? PASS : FAIL                    │ │
              │  └─────────────────────────────────────────────┘ │
              │  ┌─────────────────────────────────────────────┐ │
              │  │ summary_chain (LLM)                         │ │
              │  │   input: 3 verdicts + evidence              │ │
              │  │   output: 2-3 sentence summary              │ │
              │  └─────────────────────────────────────────────┘ │
              │                                                   │
              │  Return: {company: {"points": {...},             │
              │                     "summary": "..."}}           │
              └──────────────────────────────────────────────────┘
                                         │
                                         ▼
                          ┌──────────────────────────────┐
                          │  write_results → results.csv  │
                          │  print balanced accuracy      │
                          └──────────────────────────────┘
```
The design splits work by evidence type because LLMs are unreliable at arithmetic so any calculation needs to happen in SQL or Python, not in the model's head. Point 1 runs one query for revenue by year; Point 5 runs a separate query for 2025 costs and reuses the revenue figure Point 1 already fetched. Point 4 was the only one really needing LLM chains which also required two, one to get the interview and label it correctly for the JSON and another one to quantify it either as positive, neutral, or negative. Build agent only runs once to build the dict of chains and then rest of it is tool call, SQL queries, and Python. Did need one last LLM chain for the summary at the end.

### Results
> your point score on `public_test`, the companies where your verdicts disagree with the public answers, and why.


Point score on `public_test`: **60/60 (balanced 100.0%)**. All 20 companies matched the answer key on all 3 points, so there are no disagreeing companies to explain. I validated the rules on Ashgrove Labs manually first — checking that the growth number, the margin calculation, and the sales-team quote all gave the expected verdicts — and then ran the full pipeline to confirm they generalized across all 20 companies.


## Reflection: AI-Sycophancy and AI-Psychosis

**AI-sycophancy.** AI-sycophancy is when an AI goes more for an emotional response, empathizing with the user, rather than going full logic and responding with understanding, logic, and facts. I think the letters and documents like annual letters and news may be most affected by AI sycophancy since they're the most tailored response, like how marketing works. Marketing probably affects the AI the most because they want to sound upbeat and right — kinda like how a king pretends to be in control of everything so he can remain in control and not face revolts or public disapproval. The agent avoided this in Points 1 and 5 by simply relying on the databases because those are cold hard facts — numbers don't exactly empathize with you, and the age-old quote is "numbers don't lie." It didn't read the management letter because that can also be biased; after all, it's written by humans in a sense (databases are too, but those are numbers crunched from logic, not from corporate manipulation). An AI agent shouldn't blindly trust a management letter due to its ability to be biased or have its own agenda in lying.

**AI-psychosis.** AI-psychosis is probably the extreme of sycophancy, where the AI goes off the rails completely from the emotional and into the delusional. Kinda like when you think a rule from a game applies to real life — "if you land a jump attack in real life from the top of Mount Everest to the bottom, you'll negate the fall damage." It's understandable that AI improves when learning from context, but there's a limit to what's believable. I think where it could have hurt is if we included the management letter and the news in there as well. There may also have been a bit of bias in the LLM output from Point 4, but we converted all of that into basic variables — positive, neutral, and negative — keeping it simple instead of giving the AI more chances to doubt itself. Kinda how if you give someone four choices and say it could be any of 1–4, they'll circle all of them; fewer choices give less leeway to doubt yourself. An LLM can be trusted, but only if you set boundaries and ask it to explain itself as well — hence why there are summaries at the end. I think also including more Python gave it more conclusions and structure.
