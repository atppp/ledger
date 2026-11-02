# Beancount basics

A ledger sorts every dollar into five buckets:

- **Assets.** What you have. Cash, bank balances, investments.
- **Liabilities.** What you owe. A credit card, a mortgage.
- **Income.** Money that came in. Salary, interest, dividends.
- **Expenses.** Money that went out.
- **Equity.** Whatever does not fit into the other four. A rounding error is the usual example. A simple ledger can leave this bucket empty. In a real Beancount system it is almost impossible to avoid. For this introduction, we will ignore it.

A normal accounting system stores all five as positive numbers, then applies minus signs when it does the math. With equity set aside, net worth is defined as (this is **not** the Beancount convention):

*net worth = assets − liabilities = income − expenses*

Or, moved to one side (still **not** the Beancount convention):

*assets − liabilities − income + expenses = 0*

In those formulas, expenses are subtracted when you compute net worth, and added when you check that the books balance. The sign flips depending on the math you want, which is confusing and easy to get wrong.

Beancount stores a normal liability or income balance as a negative number. The minus signs move into the numbers, and the balance check becomes:

```text
assets + liabilities + income + expenses = 0
```

This convention makes the math very simple. You *always* add things up, whether you are computing net worth, checking that a transaction balances, or running some more complicated cross-check or query.

Say you are paid $3000, you buy furniture for $1000 on a credit card, and you later pay $500 of that card from checking. Here is how you represent those transactions in Beancount:

```beancount
2024-01-15 * "Paycheck"
  Assets:Bank:Checking     3000 USD
  Income:Salary           -3000 USD

2024-01-20 * "Furniture"
  Expenses:Furniture       1000 USD
  Liabilities:CreditCard  -1000 USD

2024-02-01 * "Pay credit card"
  Liabilities:CreditCard    500 USD
  Assets:Bank:Checking     -500 USD
```

The paycheck makes income negative. That is strange, given how people talk about a paycheck. A negative income balance means you earned $3000. The [Beancount docs](https://beancount.github.io/docs/the_double_entry_counting_method/) offer some ways to make sense of this. I found none of them completely satisfying. Just remember the convention: it is what makes the math simpler.

The furniture is a positive expense and a negative liability. You owe $1000, stored as −$1000. Paying $500 of the card adds $500 back to the liability (leaving −$500 still owed) and takes $500 out of checking.

In the end you have:

```text
assets       =  2500
liabilities  =  −500
income       = −3000
expenses     =  1000
```

net worth = assets + liabilities = 2000.

Also, note the four buckets sum to zero:

```text
2500 + (−500) + (−3000) + 1000 = 0
```

Beancount requires every transaction to balance on its own. That naturally guarantees the whole ledger always balances. This self-balance is the point of double-entry accounting: an amount leaves one account and arrives in another, and both sides are written down. Money is never recorded in just one account.

If we write only this (a simple cash-flow record):

```beancount
2024-01-20 * "Furniture"
  Liabilities:CreditCard  -1000 USD
```

Beancount rejects it with an error, because the transaction does not balance. That is useful. It forces you to add the other side, which means thinking about the right bucket (here, an expense) for a purchase like this.

Sometimes it is hard to pick the proper bucket. Say you paid $30 for lunch for yourself and a friend, and later the friend Venmo'd you $15. The $15 looks like income on your bank statement. It should really be a negative expense. I choose this convention, but you can have a different preference.

```beancount
2024-03-01 * "Lunch"
  Expenses:Dining          30 USD
  Liabilities:CreditCard  -30 USD

2024-03-02 * "Venmo from friend"
  Assets:Bank:Checking    15 USD
  Expenses:Dining        -15 USD
```

The point is that you can categorize transactions however makes sense for your finances.
