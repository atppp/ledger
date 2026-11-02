# Brokerages and other assets

Public market transactions can get extremely complicated. If you are not ready for that, start by importing each monthly statement as a simple dollar amount — or even just a yearly one.

Here is a sample:

```beancount
2024-01-01 balance Assets:Retirement:401k  10000 USD

2024-03-31 * "Deposit"
  Assets:Retirement:401k            1000 USD
  Income:Retirement:Contributions  -1000 USD

2024-03-31 * "Fees"
  Assets:Retirement:401k    -20 USD
  Expenses:Fees              20 USD

2024-03-31 * "Market change"
  Assets:Retirement:401k      500 USD
  Income:Retirement:Earnings -500 USD

2024-04-01 balance Assets:Retirement:401k  11480 USD
```

This is what I did for my 401(k) for a long time, until I had time to do more granular tracking.

If you also import pay stubs, the stub already books the 401(k) deferral out of your salary. The same contribution will show up again on the brokerage statement, so you need to keep one side and ignore or transfer-track the other — same idea as a bank transfer.

## Track positions

To get started, let's look at an example. I transferred $2000 from checking to Schwab. I bought 5 shares of LAMP at $195, then another 5 at $200. Later I sold one share at $215. The Schwab transaction CSV could look like this:

```csv
date,description,symbol,quantity,price,amount
2024-05-01,Transfer from checking,,,,2000
2024-05-02,Buy,LAMP,5,195,-975
2024-05-15,Buy,LAMP,5,200,-1000
2024-06-10,Sell,LAMP,1,215,215
```

We can represent these transactions as follows:

```beancount
2024-05-01 * "Transfer from checking"
  Assets:Brokerage:Schwab:USD    2000 USD
  Assets:Bank:Chase:Checking    -2000 USD

2024-05-02 * "Buy LAMP"
  Assets:Brokerage:Schwab:LAMP       5 LAMP @ 195 USD
  Assets:Brokerage:Schwab:USD     -975 USD

2024-05-15 * "Buy LAMP"
  Assets:Brokerage:Schwab:LAMP       5 LAMP @ 200 USD
  Assets:Brokerage:Schwab:USD    -1000 USD

2024-06-10 * "Sell LAMP"
  Assets:Brokerage:Schwab:LAMP      -1 LAMP @ 215 USD
  Assets:Brokerage:Schwab:USD      215 USD
```

This tracks cash flow and portfolio positions accurately, but not capital gains. For many casual investors, that is enough.

Most brokers let you export current positions. You can turn that into balance assertions like the following (I add these assertions once a year, just as a sanity check. I do not even bother writing a script; I just hand the position CSV to an agent and have it write these entries for me):

```beancount
2024-07-31 balance Assets:Brokerage:Schwab:LAMP      9 LAMP
2024-07-31 balance Assets:Brokerage:Schwab:USD     240 USD
```

## Track lots

To track capital gains, we need the concept of lots. Here is how you can represent these three stock transactions more accurately:

```beancount
2024-05-02 * "Buy LAMP"
  Assets:Brokerage:Schwab:LAMP       5 LAMP {195 USD, 2024-05-02}
  Assets:Brokerage:Schwab:USD     -975 USD

2024-05-15 * "Buy LAMP"
  Assets:Brokerage:Schwab:LAMP       5 LAMP {200 USD, 2024-05-15}
  Assets:Brokerage:Schwab:USD    -1000 USD

2024-06-10 * "Sell LAMP"
  Assets:Brokerage:Schwab:LAMP      -1 LAMP {195 USD, 2024-05-02} @ 215 USD
  Income:Investments:CapitalGain   -20 USD
  Assets:Brokerage:Schwab:USD      215 USD
```

Each buy records its own lot. The first is 5 shares at $195. The second is 5 shares at $200. The sale takes one share from the earlier lot. The gain on that share is $20, and like other income it is negative. Four shares remain from that lot, still at a cost of $195, and the later five stay at a cost of $200.

Each buy needs its own lot, usually identified by a cost basis and an acquisition date. That is what `{195 USD, 2024-05-02}` is for. Without a lot, Beancount still knows how many shares you hold, but not what you paid, so it cannot compute unrealized gain — or the realized gain when you sell.

At this point, if you choose "At market value" in Fava and enable the implicit prices plugin, the UI should show +$155 of unrealized gain. If you add a later price, Fava will show a more up-to-date unrealized gain of $290:

```beancount
2024-07-31 price LAMP 230 USD
```

You can generate those price directives by pulling with [yfinance](https://github.com/ranaroussi/yfinance) from Yahoo Finance. I usually enter one price at the end of each month. I do not need Beancount to track unrealized gain every day; a long-term trend is enough.

I do not keep the LAMP shares and the cash in the same `Assets:Brokerage:Schwab` account. I split them into `:LAMP` and `:USD` so each is easier to track. For the 401(k), I keep cash and fund shares in one account, because the cash is almost always used immediately to buy the fund — two accounts in the UI would be more annoying. Either layout is fine; pick what you prefer.

## Track fees

So far, we have ignored fees in our stock transactions. Here's a more realistic example:

- **2024-05-02** (purchase of 5 shares of LAMP)
  - Listed price **$195** per share
  - Fee **$5**
- **2024-06-10** (sale of 1 share of LAMP)
  - Listed price **$215** per share
  - Fee **$2**

Here is the full accounting in a table:

| Line item | Amount (USD) |
|-----------|-------------|
| Purchase — gross trade ($195 × 5) | $975 |
| Purchase fee | $5 |
| **Total cost basis** ($975 + $5) | **$980** |
| Cost per share ($980 ÷ 5) | $196 |
| Sale — gross proceeds ($215 × 1) | $215 |
| Sale fee | $2 |
| **Net proceeds** ($215 − $2, cash settled) | **$213** |
| **Realized gain** ($213 − $196) | **$17** |

For the shares sold, realized gain is gross sale − gross purchase − purchase fee − sale fee, with the purchase amount and purchase fee prorated to the shares sold (here, 1 of 5). That is the same proceeds-minus-basis comparison the IRS uses, and the same formula works for a loss.

Here is how I translate these transactions into Beancount:

```beancount
2024-05-02 * "Buy LAMP"
  Assets:Brokerage:Schwab:LAMP       5 LAMP {196 USD, 2024-05-02}
  Assets:Brokerage:Schwab:USD     -980 USD
  Expenses:Fees                      5 USD
  Income:Investments:Fees           -5 USD

2024-06-10 * "Sell LAMP"
  Assets:Brokerage:Schwab:LAMP      -1 LAMP {196 USD, 2024-05-02} @ 215 USD
  Income:Investments:CapitalGain   -17 USD
  Assets:Brokerage:Schwab:USD      213 USD
  Expenses:Fees                      2 USD
  Income:Investments:Fees           -2 USD
```

The `@ 215 USD` is the per-share sale price. Beancount does not use it to balance the transaction — the cost basis already does that — but the implicit prices plugin records it as the latest LAMP price.

The cost per share is not the $195 trade price, because the purchase fee is folded into the basis ($980 / 5 = $196). Otherwise you would have to track the purchase fee outside Beancount to reconcile the gain on sale, which I found messier and less self-contained.

The Beancount transaction already balances without explicit fee postings, but I still want the fees visible on their own, so matching amounts go on `Expenses:Fees` and `Income:Investments:Fees` and net to zero. For a sale this is easy to understand: the $2 is part of the gain recognized as income, and it gets expensed as a fee right away. For a purchase it feels counterintuitive: I paid a $5 fee — how did that become income? Think of it this way: when this lot is eventually sold in full, that $5 is part of the total gain recognized as income, but it was expensed at purchase time.

An accountant is probably screaming, "A fee you paid is not income!" but I found this much easier and more robust in a Beancount system. You probably should not track fees explicitly this way if you find it hard to understand. For me, this makes realized and unrealized gains match the statement and the brokerage portal exactly, while keeping fees explicit.

## Dividends

A dividend simply pays cash into the brokerage cash account. It is income, so that side is negative. The nine LAMP shares pay $2 each:

```beancount
2024-08-15 * "LAMP dividend"
  Assets:Brokerage:Schwab:USD     18 USD
  Income:Investments:Dividends   -18 USD
```

You can also split the income by stock, for example `Income:Investments:Dividends:LAMP`, which makes per-stock performance easier. I do not like a pile of dividend sub-accounts in the UI, so I keep a single `Income:Investments:Dividends` account and infer the ticker from the description — less reliable. For a more robust setup, tag each dividend with its ticker, such as `#LAMP`.

If dividends are set to reinvest, this cash is used immediately to buy more LAMP. At $225 a share, the $18 buys 0.08 shares. The cash comes in and goes right back out.

```beancount
2024-08-15 * "LAMP dividend"
  Assets:Brokerage:Schwab:USD     18 USD
  Income:Investments:Dividends   -18 USD

2024-08-15 * "Reinvest LAMP"
  Assets:Brokerage:Schwab:LAMP     0.08 LAMP {225 USD, 2024-08-15}
  Assets:Brokerage:Schwab:USD       -18 USD
```

You can merge these two transactions into one, so it does not even touch `Assets:Brokerage:Schwab:USD`. They are completely equivalent, even in the performance analysis:

```beancount
2024-08-15 * "Reinvest LAMP"
  Assets:Brokerage:Schwab:LAMP     0.08 LAMP {225 USD, 2024-08-15}
  Income:Investments:Dividends      -18 USD
```

I keep them separate because the CSV often already has two rows, and merging them is more work. Separating them also helps when dividends from stock A are reinvested into stock B: performance code is easier if the payout and the reinvestment stay as two transactions.

## Corporate actions

A stock split is an occasional event. In the more common forward split, the company splits one share into multiple shares. The stock price is adjusted so the total value does not change. People usually record a stock split as a transaction, so it is clear that something happened on that date. I found that cumbersome. It interferes with later analysis and makes things more complicated.

I record split events in the CSV import script instead. This is one good reason to keep the CSV as the source of truth. If a later split happens, I add the split metadata and regenerate. The import adjusts the price, the cost basis, and the share quantity of the earlier transactions. Beancount never sees the split. Everything is already in adjusted form.

A ticker change follows the same approach. For example, Facebook used to trade as FB and now trades as META. My import script keeps a map from each old symbol to the latest one and rewrites every earlier transaction to that ticker. You see one ticker in Fava, and the later analytics stay much simpler.

## Lot matching

By default, my import script matches a sale to lots on a FIFO basis, so the shares you bought first are the ones that get sold. That is often what the brokerage does by default, too. Every so often I pick a specific lot instead, usually for tax reasons. I keep these specific-lot selections in a separate lot-overrides file — each row names the sale (account, date, symbol, quantity) and the lot to take from (its date, cost, and quantity). It does not come up often, so I maintain this file by hand. The annual 1099-B form is ultimately the source of truth if you want to automate this.

A wash sale is when you sell at a loss and buy the same security again within 30 days before or after that sale. You do not get to deduct that loss. It is added to the basis of the replacement shares, and the holding period of the sold shares carries over, so the replacements count as bought earlier than the replacement date. Even if you avoid wash sales yourself, a managed account can still trigger one.

Example: you bought 10 shares at $100 and later sold them for $90 — a $100 loss. You soon bought a replacement of 40 shares at $70. The repurchase was within 30 days, so this is a wash sale and that $100 loss is disallowed. The washed slice is 10 of those 40: $700 of purchase cost plus the disallowed $100 gives those 10 shares a basis of $800 ($80 each). The other 30 stay at $70.

Until the replacement lot is recorded, the disallowed loss sits in a per-symbol WashSale account. The sale puts it there; the replacement purchase takes it off. Once that lot is booked, the account should be zero again.

```beancount
2024-10-02 * "Sell VTI"
  Assets:Brokerage:Managed:VTI           -10 VTI {100 USD, 2024-08-21}
  Assets:Brokerage:Managed:USD           900 USD
  Assets:Brokerage:Managed:WashSale:VTI  100 USD

2024-10-10 * "Buy VTI"
  Assets:Brokerage:Managed:VTI             10 VTI {80 USD, 2024-08-29}
  Assets:Brokerage:Managed:VTI             30 VTI {70 USD, 2024-10-10}
  Assets:Brokerage:Managed:USD          -2800 USD
  Assets:Brokerage:Managed:WashSale:VTI  -100 USD
```

The sold shares were held 42 days (2024-08-21 to 2024-10-02), so the 10 washed shares count as bought 42 days before the 2024-10-10 replacement — on 2024-08-29.

The wash sale rule can get complicated. You may own VTI in another account, and the rule applies across all of them. Your import script can detect wash sales across accounts and generate the entries properly (another reason to keep the CSV as the source of truth: a later purchase can rewrite the gain on an earlier sale). You can also check whether a planned sale would trigger the rule before you make it.

## In-kind transfers

An in-kind transfer moves shares from one brokerage to another. Nothing is sold, so there is no gain. The shares keep the cost and the date you bought them.

Say 2 of the remaining shares from the $196 lot move from Schwab to Fidelity:

```beancount
2024-09-01 * "Transfer LAMP to Fidelity"
  Assets:Brokerage:Fidelity:LAMP      2 LAMP {196 USD, 2024-05-02}
  Assets:Brokerage:Schwab:LAMP       -2 LAMP {196 USD, 2024-05-02}
```

Both statements show the move. As with a cash transfer, you should pick one side to keep.

## Crypto transfers

A crypto transfer between institutions is almost an in-kind transfer. The difference is a small gas fee paid in the crypto itself. That piece is disposed of, so it has a realized gain. The rest keeps its cost and acquisition date.

Say you bought 1 ETH at $2000. You send it to Robinhood when ETH is $3000, and the network takes 0.01 ETH.

```beancount
2024-09-01 * "Transfer ETH to Robinhood"
  Assets:Brokerage:Crypto:Robinhood:ETH   0.99 ETH {2000 USD, 2024-05-02}
  Assets:Brokerage:Crypto:Coinbase:ETH      -1 ETH {2000 USD, 2024-05-02}
  Income:Investments:CapitalGain           -10 USD
  Expenses:Fees                             30 USD
```

The 0.99 ETH still has a cost of $2000 per ETH. The 0.01 ETH cost $20 and was worth $30, so the gain is $10 and the fee is $30.

## Property

A property can be modeled the same way. The house is one unit, with a cost and a date, and a later price marks it to market.

```beancount
2020-06-01 * "Buy home"
  Assets:RealEstate:Home       1 HOME {500000 USD, 2020-06-01}
  Assets:Bank:Chase:Checking  -500000 USD

2024-12-31 price HOME 700000 USD
```

At market value, Fava shows +$200,000 of unrealized gain — an idea of how much the house has gained on paper.

In reality a mortgage is usually involved, and interest makes it much more complicated. A car loan is a smaller version of the same idea.

## Car loan

Say you buy a $25,000 car, put $5,000 down from checking, and borrow the other $20,000 at 6% for three years.

```beancount
2021-03-01 * "Buy car"
  Assets:Vehicles:Car                1 CAR {25000 USD, 2021-03-01}
  Assets:Bank:Chase:Checking     -5000 USD
  Liabilities:CarLoan           -20000 USD
```

The monthly payment is $608.44. The first month's interest is 6% / 12 of the $20,000 balance, so $100. The rest pays down the loan.

```beancount
2021-04-01 * "Car loan payment"
  Liabilities:CarLoan              508.44 USD
  Expenses:Interest:CarLoan        100.00 USD
  Assets:Bank:Chase:Checking      -608.44 USD
```

Each later payment is similar. The checking amount stays $608.44, but the interest expense shrinks and more of the payment pays down the loan. The last one is almost all principal:

```beancount
2024-03-01 * "Car loan payment"
  Liabilities:CarLoan              605.41 USD
  Expenses:Interest:CarLoan          3.03 USD
  Assets:Bank:Chase:Checking      -608.44 USD

2024-03-02 balance Liabilities:CarLoan  0 USD
```

A few months later you sell the car for $15,000. The $10,000 drop is depreciation. I book it under income, so a loss shows up as a positive income balance. Use `Expenses:Depreciation` instead if you would rather see it as spending.

```beancount
2024-06-01 * "Sell car"
  Assets:Bank:Chase:Checking          15000 USD
  Assets:Vehicles:Car                    -1 CAR {25000 USD, 2021-03-01} @ 15000 USD
  Income:Depreciation                 10000 USD
```
