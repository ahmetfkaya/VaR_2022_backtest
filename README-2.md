# Backtesting VaR on a 60/40 portfolio in 2022

I wanted to see how standard Value at Risk (VaR) models did in 2022. That year stocks and bonds both fell, which is the case a 60/40 portfolio isn't really built for, so it felt like a good test.

The result: all three models I tried were broken way more often than they should have been, and all of them end up in the Basel red zone.

## Question

How well did 99% one-day VaR models predict the risk of a 60/40 stock/bond portfolio in 2022?

## Data and setup

- SPY (S&P 500 ETF) and AGG (US bond ETF), daily adjusted prices from Yahoo Finance using `yfinance`
- 1 Jan 2015 to 30 Dec 2022
- 60% SPY and 40% AGG, rebalanced every day
- Simple daily returns (`pct_change`). I used these instead of log returns because the portfolio return is then exactly the weighted sum of the two ETF returns.
- 99% VaR over one day
- Each day's VaR is calculated from the 250 trading days before it. I shift it by one day so the model never uses that day's own return.
- The test period is the 251 trading days of 2022

Some numbers on the portfolio itself: over 2015–2022 it made 0.03% per day on average with a daily standard deviation of 0.73%. The worst day was −7.3% in March 2020. Excess kurtosis is 15, while a normal distribution has 0, so big days happen a lot more than a bell curve would expect. In 2022 the portfolio lost 15.6%.

## Methods

**Historical VaR.** The 1% quantile of the last 250 daily returns.

**Parametric (normal) VaR.** Assumes returns are normal and uses the rolling mean and standard deviation: `VaR = μ + z(0.01) × σ`.

**Monte Carlo VaR with a Student-t.** For every day in 2022 I simulate 10,000 returns from a Student-t with the same rolling mean and standard deviation, and take the 1% quantile of the simulations. For the degrees of freedom I matched the excess kurtosis of 2015–2021 returns (22.2) using `ν = 4 + 6 / kurtosis`, which gives 4.27. I only used data up to 2021 here so 2022 doesn't leak into the model. I also checked the simulated VaR against the Student-t formula, and they match on average.

**Backtest.** An exception is a day where the actual return was below that day's VaR. With 251 days and 99% VaR you'd expect about 2.5. I used the Basel traffic light to judge the results: 0–4 is green, 5–9 yellow, 10 or more red.

## Results

| Method | Exceptions in 2022 | Expected | Exceptions Jan–Jun | Basel zone |
|---|---|---|---|---|
| Historical | 10 | 2.5 | 9 | Red |
| Parametric (normal) | 17 | 2.5 | 15 | Red |
| Monte Carlo (Student-t) | 13 | 2.5 | 12 | Red |

The normal model was the worst. It doesn't expect many extreme days, so its VaR was closer to zero than the others and it started 2022 at about −1.1%.

The Student-t model got rid of 4 of the normal model's exceptions, but those were days where the loss was only a little below the normal VaR. The really bad days (29 April, 5 May and 13 June, losses between 2.5% and 2.9%) were below all three VaRs anyway.

For all three models most of the exceptions were in the first half of the year. The VaR only went down after the bad days were already in the 250-day window.

## Why the models failed

**The window going into 2022 was calm.** On 1 January 2022 the last 250 days were mostly 2021, which was a quiet year. The historical VaR started 2022 at about −1.4% and ended at −2.49%, so it only caught up after it had already been broken a few times.

**Fat tails were only part of it.** Before 2022 the returns had very fat tails (excess kurtosis 22.2, mainly because of March 2020). But inside 2022 the excess kurtosis was only 0.85. So 2022 wasn't a year with a couple of crazy days. It was a year with lots of medium-sized bad days. That's why the Student-t helped a bit but didn't fix it. I think the bigger problem was that volatility went up faster than a 250-day estimate can follow.

**Bonds had already stopped hedging stocks.** I also looked at the 250-day rolling correlation between SPY and AGG. From 2016 to early 2020 it was negative (around −0.1 to −0.35), so bonds usually went up when stocks fell. In March 2020 it jumped to positive and stayed slightly positive after that. It was 0.04 in 2021 and 0.32 in 2022, but most of that rise happened in the second half of 2022. So the higher correlation added risk later in the year, but it doesn't really explain the breaks in the first half. That surprised me, because I expected the correlation to be the main story.

## Limitations

- It's only one test year, so it says a lot about 2022 and not much about other periods.
- Fixed 60/40 weights rebalanced daily, and no trading costs.
- The Student-t degrees of freedom come from 2015–2021 and are pushed down a lot by March 2020.
- A 250-day window with equal weights is slow. I didn't test faster options like EWMA volatility.
- I didn't calculate Expected Shortfall or run a formal test like Kupiec. I used the Basel traffic light instead.

## What I learned

The models were only as good as the year they learned from. 2021 was calm, so they weren't ready for 2022.

I expected fat tails to be the main issue, but in 2022 the level of volatility mattered more than the shape of the distribution.

The stock/bond correlation turned positive in 2020 and never went back, which I didn't know before doing this.

It's easy to accidentally use future data. Without the one-day shift the results would look a lot better than they really were.

## How to run

Open `var_backtest_2022.ipynb` in Google Colab and click Runtime → Run all. It uses `numpy`, `pandas`, `scipy`, `matplotlib` and `yfinance` (the first cell updates `yfinance`). The Monte Carlo part uses a fixed seed (42), so you should get the same numbers. The last decimals can change a little because Yahoo Finance sometimes updates its adjusted prices.
