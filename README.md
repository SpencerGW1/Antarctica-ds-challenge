# Antarctica Asset Management — Data Science Challenge

This repository contains my submission for the Data Scientist technical
challenge: Question 1 (mandatory) and Question 2 (chosen over Question 3).

## Repository Structure
├── question1.py            — GitHub repo scraper and GUI with embedded Gemini chatbot

├── question2_cleaning.py   — Hedge fund returns data cleaning

├── question2_analysis.py   — Factor regression and strategy analysis

├── requirements.txt        — Python dependencies

└── .gitignore

## Setup

1. Clone this repository
2. Create a virtual environment and activate it:
python -m venv venv
venv\Scripts\activate        (Windows)
source venv/bin/activate     (Mac/Linux)

3. Install dependencies:
pip install -r requirements.txt

---

## Question 1 — GitHub Repository Scraper + AI Chatbot

### What it does

A Tkinter GUI that:
1. Takes a GitHub username and a folder location
2. Scrapes that user's public repository names via HTML parsing
3. Saves the repo names to an Excel file in the chosen folder
4. Gemini AI chatbot embedded into the GUI that answers questions about the scraped repository names

### Running it
python question1.py

### Adding your own Gemini API key

This app requires a free Gemini API key, which is not included in this repository.

Option 1 — .env file (recommended):

1. Get a free key from Google AI Studio
2. Create a file named .env in the project root
3. Add this line to it: GEMINI_API_KEY=your_key_here
4. The app loads this automatically — no code changes needed

Option 2 — in-app: type your key directly into the "Gemini API Key" field in the GUI. If the field is left blank, the app falls back to the .env key.

### Usage

1. Enter a GitHub username (e.g. `SpencerGW1`)
2. Enter a folder path where the Excel file should be saved
   (e.g. `C:\Users\YourName\Documents`)
3. Click "Scrape Repositories" — an Excel file named
   `{username}_github_repos.xlsx` will be saved to that folder
4. Type a question about the scraped repos in the "Ask a question"
   box (e.g. "What does the first repo do?") and click "Ask"
5. The chatbot's response appears in the Chat History box below

### Known limitations

- This implementation uses the `google-generativeai` package, which
  Google has since deprecated in favour of `google-genai`. Given the
  project timeline, the original package was retained as it remains
  fully functional; a production version would migrate to the
  supported SDK.
- GitHub paginates repository listings; this scraper retrieves repos
  from the first page only. For users with many repositories, a
  production version would need to handle pagination.
---

## Question 2 — Hedge Fund Factor Regression Analysis

### Approach

The analysis is split into two stages, each in its own file:

- **Data cleaning** (`question2_cleaning.py`, `DataCleaner` class) — loads
  the `returns data` sheet and addresses four data quality issues found on
  inspection: the 2018/2019 date blocks were out of chronological order;
  two factor columns (`Value vs Growth`, `Interest Rates`) contained extreme
  outliers caused by a units/scaling error (values were roughly 10^6 too
  large); the `Crowding` factor had 24 leading zeros representing missing
  data before the factor was first tracked (Jan 2006–Dec 2007), not genuine
  zero returns; and the fund itself had two zero-return months, which were
  inspected and retained as plausible flat returns in low-volatility periods
  rather than treated as missing.

- **Modelling and strategy analysis** (`question2_analysis.py`,
  `FactorModel` and `StrategyAnalysis` classes) — fits the regression,
  evaluates it, and compares the fund against a replicating factor portfolio.

### Setup

Place the provided data.xlsx file in the project root before running the cleaning script. The analysis script reads cleaned_data.csv, so the cleaning script must be run first.

python question2_cleaning.py    # produces cleaned_data.csv

python question2_analysis.py    # runs regression, evaluation, and strategy comparison

### 2.1 — Model fitting

Before fitting, multicollinearity was assessed via VIF (all factors below
3.5 - no concerning collinearity) and feature relevance was screened using
simple correlations with fund returns and a cross-validated Lasso, both of
which consistently pointed to `Value vs Growth` and `Credit` as the
strongest predictors.

Four candidate specifications were then compared
(varying which additional factors were included); the two-factor model
(`Value vs Growth`, `Credit`) was selected, as it achieved comparable
explanatory power (R²) to larger specifications without the additional
factors being statistically significant.

Notably, Momentum had the second-highest raw correlation with the fund yet was insignificant in every regression that also included Value vs Growth. This is a multicollinearity effect: Value vs Growth and Momentum are strongly negatively correlated (≈ -0.7), so once the former is in the model the latter carries little additional explanatory information — a clear illustration of why univariate correlation and multivariate significance can diverge, and why factors were not selected on correlation alone.

Final model (HAC-robust standard errors, 1 lag, used as a precaution given
the time-series nature of the data):

| Term | Coefficient | p-value |
|---|---|---|
| Alpha (const) | 0.0083 | <0.001 |
| Value vs Growth | -0.590 | <0.001 |
| Credit | 0.154 | 0.012 |

The fund has a statistically significant monthly alpha of approximately
0.83% (roughly 10% annualised), a strong negative exposure to the
value-growth spread, and a modest positive exposure to credit.

### 2.2 — Evaluation

R² = 0.282 (adjusted 0.274) — the two factors explain only around 28% of return variance, meaning the majority of the fund's returns are not explained by these systematic factors and are instead idiosyncratic (consistent with a large, significant alpha). The model is jointly significant (F-test p ≈ 1.6e-14).
Residual diagnostics gave a nuanced picture. The Q-Q plot and a Jarque-Bera test indicated the residuals are non-normal with fat tails (excess kurtosis ≈ 4), and the Durbin-Watson statistic (≈ 1.69) showed mild positive autocorrelation. However, the Breusch-Pagan test (p = 0.989) and the residuals-vs-fitted plot found no evidence of heteroscedasticity — the residual variance is constant. Because the non-normality and mild autocorrelation can still distort default standard errors, HAC (Newey-West) standard errors were used; under these, both factors remain significant, so the model's inference is robust to the assumption violations present.

### 2.3 — Fund vs factor portfolio: Sharpe ratio

| | Sharpe Ratio |
|---|---|
| Fund | 0.985 |
| Factor portfolio | 0.017 |

The factor portfolio's risk-adjusted return is close to zero, while the
fund's is close to 1.0. This directly addresses Tyler's question: simply
replicating the fund's factor exposure and skipping the manager would
forgo almost all of the risk-adjusted return on offer. The fund's alpha,
not its factor exposure, is doing the overwhelming majority of the work,
so investing in the underlying factors instead of the fund would not be
the more profitable strategy.

(One caveat: this conclusion is specific to the two-factor model. A richer factor set might replicate more of the return; the claim is that the factors identified as relevant to this fund do not replicate it, not that no factor portfolio could.)

### 2.4 — Risk comparison

| | Fund | Factor Portfolio |
|---|---|---|
| Annualised Volatility | 10.3% | 5.4% |
| Max Drawdown | -23.4% | -18.0% |
| Downside Deviation | 6.4% | 4.6% |
| Skew | -0.25 | -0.99 |
| Excess Kurtosis | 0.64 | 3.96 |

The fund shows higher headline volatility and a slightly worse maximum
drawdown than the factor portfolio. However, the factor portfolio has
materially worse skew and excess kurtosis — far fatter tails and a more
extreme negative skew — despite its lower volatility. Headline volatility
alone understates the factor portfolio's tail risk; the two strategies
carry meaningfully different risk profiles rather than the factor
portfolio being unambiguously "safer." Read alongside 2.3, the fund takes more conventional risk but is well compensated for it, whereas the factor portfolio is less volatile but carries greater tail risk for almost no return.

### 2.5 — Stationarity of betas

36-month rolling regressions were used to estimate how the two betas
evolved over time, and an Augmented Dickey-Fuller test was applied to
each resulting beta series:

| Factor | ADF statistic | p-value | Stationary? |
|---|---|---|---|
| Value vs Growth | -1.398 | 0.583 | No |
| Credit | -1.839 | 0.361 | No |

Neither beta series is stationary at conventional significance levels,
meaning the fund's factor exposures have genuinely shifted over the
sample period rather than fluctuating around a stable long-run value.
From a risk-management perspective this is a meaningful caveat: the
regression above gives a snapshot of current exposures, but non-stationary
betas suggest the fund's risk profile is not stable over time and may be
subject to style drift, making historical betas a less reliable guide to
future risk than they would be for a fund with stationary exposures.
(The ADF test on rolling betas is somewhat informal, since adjacent windows share 35 of 36 months and are therefore autocorrelated by construction; here the formal test agrees with the clear visual drift in the rolling-beta plot, so the conclusion is well supported.)
---

## AI Usage Disclosure

AI assistance (Claude) was used in the following ways during this
project, in line with the stated AI policy - for research, methodology guidance, debugging purposes only, not for generating submitted code:

- Understanding the BeautifulSoup API for parsing GitHub's HTML
  structure (Question 1)
- Debugging a Tkinter `Text` widget rendering issue where chat responses
  were not displaying after insertion, caused by the widget's disabled
  state not being toggled before inserting text (Question 1)
- Identifying that the Gemini model name `gemini-1.5-flash` had been
  deprecated and diagnosing a subsequent quota error, leading to the use
  of `gemini-2.5-flash` (Question 1)
- Understanding the mechanics and interpretation of VIF, the
  Breusch-Pagan test, HAC standard errors, and the `statsmodels` OLS
  summary output (Question 2)
- Understanding the Augmented Dickey-Fuller test and how to interpret
  non-stationary results in the context of rolling betas (Question 2)
-  Guidance on feature-selection methodology (combining economic reasoning, correlation screening, and Lasso) and on structuring the analysis into classes for readability (Question 1 & 2)
- Debugging environment and syntax issues (package installation, matplotlib display behaviour, and indentation during the refactor)