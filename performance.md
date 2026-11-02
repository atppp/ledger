# Portfolio performance

**This chapter does not provide investment or tax advice. It describes one possible framework for measuring performance. The last two sections share my own opinions and beliefs; they are not recommendations for your situation. Please make your own decisions based on your circumstances, preferences, and beliefs.**

How do you quantify investment return? Say you put $100 in a savings account that pays monthly interest and compounds. Two years later the balance is $110.

The simplest metric is gain %. Here the gain (interest) is $10 and the cost is $100 (the original deposit), so gain % is 10%.

## Why not gain %

The savings example makes gain % look easy: $10 of interest on a $100 deposit is 10%. In general the gain is usually easy to measure. The hard part is the denominator — the cost you divide by.

Say you deposited $100k, earned $10k of interest over a few years, withdrew $90k, later earned another $2k, then deposited $10k. The account now holds $32k. The gain is clearly the $12k of interest — but what is the cost? Total deposits of $110k? The original $100k? The $20k of your own money still in the account (balance minus gain)? Or $10k, because the last deposit has not earned interest yet? Some banks use something more complex, such as average daily balance.

There is no widely agreed way to define cost in general. Even for your own finances, you will likely want different definitions for different accounts.

The Extended Internal Rate of Return (XIRR) comes to the rescue.

## XIRR

A 10% gain over one year is very different from a 10% gain over ten years. To account for time, people often use an annualized return: assume an average yearly return *r* that compounds:

$$
100 \times (1 + r)^{2} = 110
$$

Solving this gives *r* ≈ 4.88%. That is a 4.88% gain each year, on average.

Annualized return is harder when deposits or other cash flows are irregular. Say you deposited $100 in year 1 and $200 in year 2, and in year 4 the balance is $350. XIRR assumes an average annualized return *r*. The first $100 stayed in the pool for 3 years and the $200 for only 2, so their grown values sum to:

$$
100 \times (1 + r)^{3} + 200 \times (1 + r)^{2} = 350
$$

*r* is about 6.81%. In general, the XIRR equation cannot be solved analytically. The typical solution uses [Newton's method](https://en.wikipedia.org/wiki/Newton%27s_method).

I use XIRR as the main metric to quantify investment performance. The most important thing to get right here is to count only "external cash flows". For the example above, XIRR simply considers what was deposited from outside, and the value if you were to cash out:

| Time in the pool | What XIRR counts |
| --- | --- |
| 3 years | $100 external deposit |
| 2 years | $200 external deposit |
| current | $350 account value if we were to cash out |

The deposits add up to $300, so the other $50 is growth inside the account. It does not get its own line in the XIRR equation — do not count it as an external deposit.

Now consider dividends. Say you deposited $100 three years ago and bought shares. Two years ago they paid a $20 dividend that you took as cash without reinvesting. Today the account is worth $115 if you cash out. That dividend payout is an external cash flow (cash out):

| Time in the pool | What XIRR counts |
| --- | --- |
| 3 years | $100 external deposit |
| 2 years | $20 dividend payout (cash out) |
| current | $115 account value if we were to cash out |

$$
100 \times (1 + r)^{3} - 20 \times (1 + r)^{2} = 115
$$

What if that same $20 was reinvested into more shares instead? Today the account might be worth $140 if you cash out. XIRR simply considers this:

| Time in the pool | What XIRR counts |
| --- | --- |
| 3 years | $100 external deposit |
| current | $140 account value if we were to cash out |

If you treat the dividend payout and the reinvestment as separate transactions, the math does not change — the two terms cancel. The payout is −$20 and the reinvestment is +$20 on the same day:

$$
100 \times (1 + r)^{3} \underline{- 20 \times (1 + r)^{2}} \underline{+ 20 \times (1 + r)^{2}} = 140
$$

You can probably spot that, mathematically, you could also compute the net cash flow for each day first. This is actually a great idea to simplify things, not just for an exact cancellation like this. More importantly, it will make benchmarking more robust. More on that in the benchmark section.

XIRR can also measure aggregate performance across multiple positions — a sector, or the whole Schwab brokerage account. External cash flow is then only money that enters or leaves that boundary. If a non-reinvested dividend stayed in Schwab cash and was later used to buy other stocks, it never left Schwab, so it is not external for the overall Schwab XIRR.

## XIRR implementation

Internal vs external sounds complicated, but with a solid Beancount ledger it is not hard to code. Any other portfolio tracker works too, as long as it can give dated cash flows and current positions. My approach is to generate a cash-flow series for each asset and mark each line internal or external. Both change the position, so both affect the final account value. Only the external lines go into XIRR.

For example, here is a non-reinvested dividend and a savings interest payment — the $20 dividend and the $10 interest from the earlier examples.

```beancount
2024-06-15 * "LAMP dividend"
  Assets:Brokerage:Schwab:USD      20 USD
  Income:Investments:Dividends    -20 USD

2024-06-15 * "Savings interest"
  Assets:Bank:Apple:Savings        10 USD
  Income:Interest                 -10 USD
```

The dividend becomes two external lines, one in each asset's cash-flow series. The $20 leaves LAMP and lands in the cash account, so LAMP is negative and USD is positive. The LAMP line is not a separate Beancount posting — it is how the generator records the same dividend from LAMP's point of view when it builds LAMP's series. "External" here is relative to that one asset: the dividend is external to LAMP even though it stays inside Schwab as a whole. The interest stays in the savings account, so that line is internal.

```csv
asset,date,amount,type
Assets:Brokerage:Schwab:USD,2024-06-15,20,external
Assets:Brokerage:Schwab:LAMP,2024-06-15,-20,external
Assets:Bank:Apple:Savings,2024-06-15,10,internal
```

A stock buy or sale has to record the position change as well, not just the cash amount. For a stock you also need the current price to turn the current position into a final account value. From these you can compute XIRR per asset.

An aggregate XIRR is the same calculation on a bigger boundary, such as a sector or all of Schwab. Sum the cash-flow lines and positions of every asset inside it. Because we net cash flows per day first, when aggregating all of Schwab these two $20 lines cancel and the dividend is effectively internal — you do not have to change their type. Any movement between assets inside the boundary nets out the same way, even when each asset marked that line external.

## Benchmarks

Once you have XIRR for an investment, or an aggregate XIRR for a stock portfolio, how do you tell if it is good or bad? That is a hard question, because it depends on your risk tolerance. You may not be okay with high volatility when the money supports a short-term need, such as an emergency fund, so a low annualized return can be acceptable.

For most long-term investments, an S&P 500 index fund is the usual benchmark. SPY is one of the most common funds for that index, and the one in the example below. Warren Buffett once bet that a low-cost S&P 500 index fund would beat a group of hedge funds over ten years. He was right; the hedge-fund side conceded with months left in the bet. Maybe he picked a strong decade for the S&P 500 (he actually did not: the bet started in 2008, right before the crash), but the point still holds. It is a good benchmark to beat if you think you are skilled.

How do you compare your investment with the S&P 500? Once you have the cash flow, you can run that same cash flow through a simulated index account. Take the example above. Three years ago you put in $100, and two years ago you put in $200. On each of those days, buy that many dollars of SPY. If you later take money out, sell that many dollars of SPY on that day. Today, value whatever SPY shares are left. Make sure you use an adjusted SPY price, which treats dividends as reinvested and is adjusted for splits. XIRR on that imaginary account is what the S&P 500 would have returned on your cash flow. Compare your investment's XIRR with that number. Is it higher or lower, and by how much?

If your investment is doing particularly well, the simulated benchmark may not hold enough shares to cover the same sale transaction. You can choose a convention: cap the sale at the shares the benchmark holds, sell them all, and clear the position; or allow a negative position in the benchmark. Either way, your investment XIRR will likely be much higher than the benchmark's, so it probably does not matter that much. This is also a strong reason to compute the daily net cash flow first, before benchmarking. A pretty normal portfolio, without exceptionally good investments, can still oversell the benchmark simulation if you leave two large cash-flow items of opposite sign from the same transaction un-netted.

Risk tolerance can point you at a different benchmark. Closer to retirement, bonds may be the comparison you care about. If your thesis is tech-heavy, QQQ, XLK, or VGT may be a more aggressive benchmark.

Once you have a robust workflow here, you can get even more creative. For example, you can plot daily portfolio value against the benchmark value, or your XIRR against the benchmark XIRR, to see how you are (hopefully) beating SPY over time. You can evaluate each stock position the same way, or a sector, or one portfolio such as Schwab as a whole.

At the end of the day, it is a simple question. Would I have ended up better off if I had simply put that same money in the benchmark, kept holding it, and done nothing else? Did all that trading in other assets actually beat it?

## Tax drag

If you live in the United States, or a country with similar tax law, there is an important detail left out: tax on your investment income. The real bill is complicated — holding period (short-term or long-term?), different kinds of dividends, losses, state tax, brackets — and we will not go into those details here.

At a high level, tax drag is the gap between pre-tax XIRR and the return you keep. Tax paid is cash that has left, so it no longer compounds.

Modeling tax drag is not straightforward, and preferences differ. Do you count tax against the current account value? If you need to liquidate to pay living expenses, yes. If you are benchmarking for estate planning with a cost-basis step-up, you should not tax unrealized gain.

A simple approach is to take your yearly tax return, find your effective tax rate on investment income, and reduce investment income (realized gains, dividends whether paid out or reinvested, interest, and so on) by that rate. This should give you a more realistic XIRR.

Apply the same assumption to the benchmark. The adjusted SPY series reinvests dividends before tax, so on its own it is a pre-tax benchmark. If you compare your after-tax XIRR with a pre-tax SPY simulation, you handicap yourself and the result looks worse than it really is. You cannot use the adjusted price for this. You likely need to go back to every SPY dividend payout, reduce it by your own tax rate, and reinvest only what is left. Tax the gains the simulated account realizes when it sells, at the same rate. If you choose not to tax your unrealized gain, as in the estate planning case, do not tax the benchmark's either.

There is a subtler point about sales inside the account. Say you sell a stock with a large gain and the proceeds stay in Schwab. You still owe tax on that gain, often paid from somewhere else, so your after-tax XIRR drops. For the whole account that day's net external cash flow is zero, so the SPY simulation does nothing and its XIRR is unchanged. The more you trade inside the boundary, the more tax drag you take that the cash-flow-matched index never sees. That is another reason a buy-and-hold benchmark is hard to beat after tax.

## Why an index fund

Whether a broad index fund should be your default is an important question to ask yourself, especially for money you add through your working years and then leave for retirement or an estate. Most individual stocks lose money over their lifetimes — more than half, in U.S. data since 1926 — and a small handful of winners account for nearly all of the market's net gain ([Bessembinder, 2018](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=2900447)). A broad index fund owns the large companies in the market, its fee is small, and you do not have to guess which of those companies will be the winner. It is a dull holding. For money you expect to leave alone for a long time, that dullness is a feature, not a flaw.

The market itself is not a zero-sum game. Companies earn profits, and those profits are why a stock portfolio can grow. Beating the market is the zero-sum part. Every dollar of return above the index is a dollar someone else did not get. The people on the other side of that trade are often institutions. They do this all day, with more staff and more information than one person has, and they still cannot reliably beat SPY. The tax drag from the last section raises the hurdle even further.

That dullness also gives you time back. Monitoring a portfolio means checking prices, sitting with every drop, and carrying the market home to dinner. An index fund does not ask for that. Those hours are better spent with your family. How do you even put a price on the time you spend with your kids? And how do you model the dollar equivalent of the stress caused by a bad "stock pick"?

A good year is also easy to remember. The stock that worked stays with you. The names that went nowhere fade. That is selective memory, and it can make a win feel like a method. The index does not tell that story. Over a long period of adding money, ask yourself whether you would have been content with that plain result.

## Investment decision

None of that settles what to do with the money already in your account, or with the next dollar you add. Even if you lean toward an index fund for long-term money, you still have to decide what to keep, what to sell, and what to buy. Or perhaps you like to set aside some money to gamble on stock picks, because that is exciting for you — and even if you lose, that is good entertainment.

In my opinion, you should first pick a benchmark that fits your situation. For a lot of long-term money that benchmark is SPY, but your age may tell a different story. How soon do you need the money, and how large a drop can you live with? What to sell and what to buy should both be judged against that choice.

Is your current position in a specific stock worth holding? Some people look at its past performance, and if it beat SPY they feel good about it and keep holding. This probably is not the best way to decide, or at least not the whole story. For one thing, past performance is not indicative of future results. More importantly, the price you paid, and the gain or loss since then, are a "sunk cost" at this point. Ask, as of today, whether its future looks better than the benchmark you already chose. Just because you lost 80% on a stock does not mean you should sell it now, if you believe that from here it can bounce back faster than SPY grows. The stock's past performance overall can be a useful metric. Your particular journey to this position is only useful for the tax, not for evaluating its future potential. You may also need to optimize for the tax you pay. For example, even if a stock's future looks bad, it may still make sense to split the sale between this year and next year, so that you have a lower tax rate in each year.

What to buy is mostly belief, preference, and risk tolerance. You typically have to balance these and come up with your own thesis. For example, if you are young, and so can tolerate a temporary pullback for years, and you have high conviction that the semiconductor industry still has huge upside, then building a riskier position (than SPY) in an ETF for that sector is a sane decision.

You have probably realized by now that I am hinting at something. XIRR and a benchmark comparison cannot tell you what to do with any one position, because that decision depends on the position's future, not its past. What they can do is judge your strategy as a whole. Over the years, were your "stock picks" actually beating a benchmark fund such as SPY? This exercise should give you an honest answer, and if your "picks" as a whole were not as good as you thought, I hope it convinces you to shift toward a boring index fund. If you are one of the lucky few who actually beat your benchmark, congratulations. I hope your luck continues.

**Not financial, investment, tax, or legal advice. This is for informational and educational purposes only. You take your own risk — of losses, and of finding this too conservative and making less than you might have.**
