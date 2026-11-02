# Stock options and RSUs

This chapter is about equity grants — mainly stock options and RSUs from a pre-IPO company, and RSUs once the company is public.

## Stock options

As an example, you are granted options on 480 shares of ACME Corporation. The vesting schedule is 4 years, so 10 options vest every month. The strike price is $10 and the fair market value is $40 per share. Assume these are NSOs: unlike ISOs, you owe ordinary income tax on the spread at exercise; with ISOs the spread can count toward AMT instead.

Let's say 10 options have vested so far. If I were to exercise all of them, I would have to pay $10 × 10 shares = $100 for the strike price, plus tax on the spread of $30 × 10 shares = $300. The spread is generally taxed as ordinary income. For simplicity, we will assume the tax rate is 50%, which means I would need to pay $150 in tax + $100 = $250 to own these 10 shares, valued at $400 in total.

Given that, I represent an option vest as follows. Note that I did not actually exercise. The thought experiment above is what I use to quantify how to model this vesting event:

```beancount
2024-06-01 * "Option vest"
  ; the vested options, at the $10 strike
  Assets:Brokerage:Acme:Options    10 ACME.NSO {10 USD}
  Equity:StockOptions:Acme        -10 ACME.NSO {10 USD}
  ; the $100 strike you have not paid
  Assets:Brokerage:Acme:Options  -100 USD
  Equity:StockOptions:Acme        100 USD
  ; estimated tax on the $300 spread
  Equity:StockOptions:Acme        150 USD
  Liabilities:EstimatedTaxes     -150 USD

2024-06-01 price ACME.NSO 40 USD
```

`Equity:StockOptions:Acme` is a catch-all account to balance this transaction, so the asset and liability accounts reflect the underlying reality.

At this point, if you choose "At market value", Fava should show `Acme:Options` as worth $300 (after paying the $10-per-share strike price), with estimated-tax liabilities of -$150. Net worth is $150 in this case. That gives a sense of what these stock options would actually be worth if the company were to IPO right now at exactly $40 per share.

## Stock option exercise

Let's say 5 options are exercised at a fair market value of $40 per share. The strike is $10 per share, so $50 for the 5 shares. The spread is $150 (which counts as this year's income), and at 50% the tax is $75. Checking pays the strike and the tax, $125 in total.

```beancount
2024-09-01 * "Exercise"
  ; exercise the options
  Assets:Brokerage:Acme:Options           -5 ACME.NSO {10 USD}
  Equity:StockOptions:Acme                 5 ACME.NSO {10 USD}
  ; receive the shares
  Assets:Brokerage:Acme:Equity             5 ACME.SHARES {40 USD, 2024-09-01}
  ; pay the strike and the tax
  Assets:Bank:Chase:Checking            -125 USD
  Liabilities:Taxes:Withholding:TY2024    75 USD
  Income:Work:Acme:OptionExercise       -150 USD
  ; the unpaid strike on these options comes off
  Assets:Brokerage:Acme:Options           50 USD
  Equity:StockOptions:Acme               -50 USD
  ; the estimated tax on these options comes off
  Equity:StockOptions:Acme               -75 USD
  Liabilities:EstimatedTaxes              75 USD

2024-09-01 price ACME.SHARES 40 USD
```

Note that the 5 shares are at $40 per share. That is the cost basis you use to compute the capital gain when you eventually sell them. The $75 withholding is typically an aggregate number reported by the option administrator, while the actual tax payments show up on a separate pay stub, which may look like this:

```beancount
2024-09-15 * "Pay stub"
  Liabilities:Taxes:Withholding:TY2024  -75 USD
  Expenses:Taxes:TY2024:Federal          45 USD
  Expenses:Taxes:TY2024:State            20 USD
  Expenses:Taxes:TY2024:SocialSecurity    7 USD
  Expenses:Taxes:TY2024:Medicare          3 USD

2024-09-16 balance Liabilities:Taxes:Withholding:TY2024  0 USD
```

In other words, the account `Liabilities:Taxes:Withholding:TY2024` is simply used to park the withheld amount, so you can cross-check the two places that report the same tax payment against each other.

## Fair market value update

If a later series raises fair market value to $50 per share, you can do the following:

```beancount
2025-01-15 price ACME.NSO 50 USD
2025-01-15 price ACME.SHARES 50 USD

2025-01-15 * "Estimated tax adjustment"
  Equity:StockOptions:Acme     25 USD
  Liabilities:EstimatedTaxes  -25 USD
```

The spread on the 5 remaining options goes from $30 to $40. Half of that extra $10 per share, times 5 shares, is another $25 of estimated tax. Fava should show `Acme:Options` as worth $200 (5 × $50, minus the $50 unpaid strike), with estimated-tax liabilities of -$100, so the remaining options now add $100 to your net worth, up from $75.

## Cashless exercise

A sell-to-cover is a single transaction in which you exercise your options and sell enough of the shares to cover the strike price and the tax. You do not pay for the exercise out of pocket — hence the name "cashless exercise". Let's say the company has IPO'd at $50 per share, and you want to make a cashless exercise of the remaining 5 stock options. The spread is $200, and at 50% the tax is $100. Together with the $50 strike (5 options at $10 per share), that is $150, which is 3 shares at $50. You keep the other 2. Here is how the Beancount entry might look:

```beancount
2025-06-01 * "Cashless exercise"
  ; exercise the options
  Assets:Brokerage:Acme:Options           -5 ACME.NSO {10 USD}
  Equity:StockOptions:Acme                 5 ACME.NSO {10 USD}
  ; keep 2 shares; the other 3 are sold at $50 to cover the strike and the tax
  Assets:Brokerage:Acme:Equity             2 ACME.SHARES {50 USD, 2025-06-01}
  Liabilities:Taxes:Withholding:TY2025   100 USD
  Income:Work:Acme:OptionExercise       -200 USD
  ; the unpaid strike on these options comes off
  Assets:Brokerage:Acme:Options           50 USD
  Equity:StockOptions:Acme               -50 USD
  ; the estimated tax on these options comes off
  Equity:StockOptions:Acme              -100 USD
  Liabilities:EstimatedTaxes             100 USD

2025-06-01 price ACME.SHARES 50 USD
```

## Double-trigger RSU

An RSU is a promise of shares, and there is no strike price. A double-trigger RSU does not pay out merely because you have time-vested. You also need a liquidity event, usually the IPO. Until then you do not have the shares, and you do not owe tax. Say the schedule is the same as for the options, 10 shares a month. That monthly vest is only the first trigger.

Let's say 10 of them have time-vested, and the fair market value is $40. If the company were to IPO at that moment and these settled, that would be $400 of ordinary income. At 50%, the tax would be $200. Nothing is paid. I still book the vest, so I can see what they are worth. The cost stays $0, since nothing was paid and the shares have not been delivered. The price directive records the $40 fair market value per share, so Fava can value the 10 RSUs at market ($400).

```beancount
2024-06-01 * "RSU vest"
  Assets:Brokerage:Acme:RSU     10 ACME.RSU {0 USD}
  Equity:RSU:Acme              -10 ACME.RSU {0 USD}
  Equity:RSU:Acme              200 USD
  Liabilities:EstimatedTaxes  -200 USD

2024-06-01 price ACME.RSU 40 USD
```

`Equity:RSU:Acme` is the catch-all, the same idea as `Equity:StockOptions:Acme`, so the RSU account and the estimated tax are the numbers that mean something. If you choose "At market value", Fava should show the RSUs as worth $400, with estimated-tax liabilities of -$200. Net worth is $200. That gives a sense of what these RSUs would actually be worth if the company were to IPO right now at exactly $40 per share.

Let's say the IPO price is $50. Those same 10 shares are now $500 of income, and the tax is $250. Selling 5 shares at $50 covers the tax, and you keep the other 5. The $200 estimate based on the $40 fair market value comes off. The withholding is the actual $250 at the IPO price.

```beancount
2025-06-01 * "RSU settlement"
  Assets:Brokerage:Acme:RSU              -10 ACME.RSU {0 USD}
  Equity:RSU:Acme                         10 ACME.RSU {0 USD}
  Assets:Brokerage:Acme:Equity             5 ACME.SHARES {50 USD, 2025-06-01}
  Liabilities:Taxes:Withholding:TY2025   250 USD
  Income:Work:Acme:RSU                  -500 USD
  Equity:RSU:Acme                       -200 USD
  Liabilities:EstimatedTaxes             200 USD

2025-06-01 price ACME.SHARES 50 USD
```

## Public company RSU

Once the company is public, the vest date is when you actually receive the shares. The whole fair market value that day is ordinary income, and the company usually keeps some shares to cover the tax. You do not pay that tax out of pocket.

Say 10 RSUs vest at $50. That is $500 of income. At 50%, the tax is $250, which is 5 shares. The company keeps those 5, and you keep the other 5. The cost of the shares you keep is the $50 vest price, so there is no unrealized gain that day.

```beancount
2025-09-15 * "RSU vest"
  ; 10 RSUs vest; 5 shares are kept, 5 are withheld for tax
  Assets:Brokerage:Acme:Equity             5 ACME.SHARES {50 USD, 2025-09-15}
  Liabilities:Taxes:Withholding:TY2025   250 USD
  Income:Work:Acme:RSU                  -500 USD
```

The $250 sits in `Liabilities:Taxes:Withholding:TY2025`, same as an option exercise. A later pay stub splits it into the actual tax payments.

