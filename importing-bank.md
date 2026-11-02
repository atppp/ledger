# Banks, cards, and pay stubs

The first step in a Beancount system is often to import day-to-day transactions. For most people that means a checking account and credit cards. Most institutions now let you download recent transactions as a CSV for free. That file is a good place to build the first skeleton of the system and get familiar with it.

## Columns

Each institution uses its own format. The columns you actually need are usually these:

- **date**
- **description**
- **amount**

and sometimes these two:

- **type**. Often the amount already carries the sign, so you can tell a debit from a credit without any other columns. Some institutions always give a positive amount; then you need this column to decide the sign.
- **category**. Values such as Gas, Groceries, or Restaurants. Useful when mapping the row to an expense account.

## Example

A credit card download CSV might look like the following. A negative amount is a charge. A positive amount is a payment.

```csv
date,description,amount,category
2024-03-04,Shell Oil,-40,Gas
2024-03-06,Safeway,-80,Groceries
2024-03-15,Payment thank you,200,Payment
```

Here is how I might represent these three rows in Beancount. Gas and Groceries become expense accounts. The payment is not an expense. It moves money from checking onto the card.

```beancount
2024-03-04 * "Shell Oil"
  Expenses:Gas                            40 USD
  Liabilities:CreditCard:Chase:Freedom   -40 USD

2024-03-06 * "Safeway"
  Expenses:Groceries                      80 USD
  Liabilities:CreditCard:Chase:Freedom   -80 USD

2024-03-15 * "Payment thank you"
  Liabilities:CreditCard:Chase:Freedom   200 USD
  Assets:Bank:Chase:Checking            -200 USD
```

The account name is your design choice. Some people prefer `Assets:Bank:Checking:Chase`, because they have other checking accounts and want those grouped together in the UI. I prefer to group by bank, so Chase is the parent and the account type sits under it.

## The script

Your first task is to write a Python script that turns the CSV into Beancount transactions. It is often just a loop over each row. Since the Beancount file is also plain text, you can simply generate it directly. However, I found using the Beancount Python library gives more robust logic and structure. I build entries with `beancount.core.data`, amounts with `beancount.core.amount`, and write the file with `beancount.parser.printer`.

For each row, you decide on the other account that balances the transaction:

1. If the row is a credit, which bank account did this money come from?
2. If the row is a debit, which expense category should it go into?

You can often infer both from the description and the category column. The credit case has to be right: if a payment is booked to the wrong bank account, that account will not balance later. You probably cannot get every debit right — that is too much work. My approach is to get the large amounts right, and let the rest fall through to `Expenses:Unknown`. I check that account from time to time and categorize any new large items that landed there. Small leftovers in `Expenses:Unknown` usually take more work to categorize than they are worth for budgeting.

Some people train a small machine-learning model to map a description to a Beancount account. Everyone designs expense categories differently, so you can be creative. Before you get granular, ask what the categories are for. Often the point is to track spending, and broad buckets such as healthcare, utilities, food, travel, and shopping are enough.

## Source of truth

Many people I know download a monthly CSV, import it, categorize it, and then treat the generated Beancount file as the source of truth. I find that less robust. I treat the CSV as the source of truth, and a Makefile regenerates the whole Beancount file whenever new rows are added. That is a bit less scalable, but more reproducible. For example, I can change a description rule so it maps to a new category — say split `Expenses:Utilities` into `Expenses:Utilities:Electricity` and `Expenses:Utilities:Gas` — and regenerate.

One-off cases can be annoying. A check deposit, say a tax credit, may need its own category. I leave a permanent rule in the script:

```python
if date == "2024-04-12" and amount == "1250.00" and check_number == "10482":
    counter_account = "Income:IRS"
```

That is probably not the most scalable solution. I still find it more robust than treating the Beancount entries as the source of truth.

## Beyond CSV

Many institutions only keep about two years of transactions available as a CSV download. If you want older history, or the institution does not offer a CSV at all, you have to pull transactions out of a PDF — or worse, a scan. For a PDF, I use [pdfminer.six](https://github.com/pdfminer/pdfminer.six) to extract the text in a structured way, then build rules that turn that text into a CSV. There are also open-source libraries for extracting transactions from the standard PDF statements of large banks or the pay stubs of large employers, and even Beancount importers for some of them. For a scan, I run OCR first. I have found OCR unreliable, so that part stays partly manual or agent-driven.

Either way, once a CSV exists, the rest of the import works as usual.

For a more advanced, real-time setup, you can pull transactions through the [Plaid](https://plaid.com/) API instead of downloading files yourself.

## Exploring

Once the Beancount file is generated, you can use [Fava](https://beancount.github.io/fava/) to start a web server and browse the ledger. Click around and see if it makes sense.

Beancount also has a [query language](https://beancount.github.io/docs/beancount_query_language/) for more advanced reports. An agent is often an even better interface for that. Give it the Beancount files as context and ask things like: "Pull all my subscription payments. Which ones cost more this year than last year?" "Which month did I use the most electricity?" "Did every work expense on the card get reimbursed?"

## Balance assertions

You can occasionally drop a balance assertion:

```beancount
2025-01-15 balance Assets:Bank:Chase:Checking  1842.17 USD
```

I usually add this once a year, from the ending balance on the bank statement. Later, if an import books a wrong transaction, I can quickly narrow which date range the reconciliation problem is in.

Credit cards are often annoying to get right. A pending charge can later post with an earlier date, while the statement's ending balance still excludes it. Your mileage may vary.

You often cannot go back to the start of the account. Figure out the balance just before the first transaction you imported, and put in an opening-balance entry:

```beancount
2019-12-31 * "Opening balance"
  Assets:Bank:Chase:Checking   900.00 USD
  Equity:OpeningBalances      -900.00 USD
```

## Transfer de-duplication

A transfer between two of your own accounts shows up on both statements. A card payment is the most common case. If you record both rows, the same money is counted twice and your balances will be wrong.

A common workflow is: the importer compares the new download to entries already in the ledger and marks matches by date, accounts, and amount (within a few days). You resolve each transfer once; after that the Beancount file is the source of truth and the decision stays where it was written.

I found that approach less robust and more error-prone. Because I treat the CSV as the source of truth, I instead code which side to keep in the import script. I typically pick the side with the clearer description, the better date, the longer history, or the one that is easier to tell apart from other transactions — and ignore the other CSV row.

As the system grew, those keep-or-ignore decisions were scattered across many scripts and became error-prone too. To bring order to that, I added a transfer-tracking system on the side. It writes a trace in `Equity:TransferTracking`, so I can spot problems when the keep-or-ignore logic is inconsistent across importers.

Say $500 moves from Chase checking to Apple savings, and both CSVs show it. I choose Apple as the owner (Apple's import script keeps the real transaction). The downloaded CSVs might look like this:

```csv
# apple csv export
date,description,amount
2024-03-20,Transfer from Chase,500
```

```csv
# chase csv export
date,description,amount
2024-03-20,Apple Savings transfer,-500
```

In Beancount, these become three transactions.

```beancount
; from apple import
2024-03-20 * "Transfer from Chase"
  Assets:Bank:Chase:Checking  -500 USD
  Assets:Bank:Apple:Savings    500 USD

2024-03-20 * "(transfer tracking: kept): Transfer from Chase"
  Equity:TransferTracking:Assets-Bank-Chase-Checking  -500 USD
  Equity:TransferTracking:Assets-Bank-Apple-Savings    500 USD

; from chase import
2024-03-20 * "(transfer tracking: ignored): Apple Savings transfer"
  Equity:TransferTracking:Assets-Bank-Chase-Checking   500 USD
  Equity:TransferTracking:Assets-Bank-Apple-Savings   -500 USD
```

If I have not imported one of the statements, or I used the wrong account on the other side, `Equity:TransferTracking` shows a nonzero balance — sometimes in more than one account — and I see it in the UI.

This way I never fully drop a CSV row. Either a real transaction is generated along with a tracking record, or only a tracking record is generated. I found this very robust; it almost never gives me headaches now.

## Pay stubs

A paycheck transaction is usually extracted from a pay stub PDF. The checking CSV only shows the net deposit. The stub is where gross pay, taxes, and a retirement deferral are itemized.

```beancount
2024-03-15 * "Paycheck"
  Income:Salary                      -3000 USD
  Expenses:Taxes:TY2024:Federal        400 USD
  Expenses:Taxes:TY2024:State          100 USD
  Assets:Retirement:401k               200 USD
  Assets:Bank:Chase:Checking          2300 USD
```

I put the tax year in the account name so I can track tax payments year by year when I file. You can skip that if you do not need that granularity.

That $2300 also appears on the checking statement. Keep the stub as the real transaction, and leave a transfer-tracking trace for the bank row, the same way as above.

A paycheck can get a lot more complicated than this example, depending on company policy and which benefits are involved. For example, here is how I model a health FSA. At the start of the year I record the full amount I elected. Each paycheck pays a piece of that liability down (just another posting on the stub, not its own transaction). The doctor visit is already an expense on the card. The later deposit is the plan paying me back — not income, and not a second expense.

```beancount
2024-01-01 * "FSA election"
  Liabilities:HCFSA  -1000 USD
  Equity:HCFSA        1000 USD

2024-03-15 * "Paycheck"
  Income:Salary      -3000 USD
  ; ... omitted ...
  Liabilities:HCFSA    100 USD
  ; ... omitted ...

2024-04-02 * "Doctor"
  Expenses:Health          80 USD
  Liabilities:CreditCard  -80 USD

2024-04-20 * "FSA reimbursement"
  Assets:Bank:Chase:Checking   80 USD
  Equity:HCFSA                -80 USD
```

By the end of the year the liability should be back to zero. If I do not use the full election, I move what is left in `Equity:HCFSA` to something like `Equity:Benefits:Unused`, so the leftover is obvious. If I leave the job mid-year, I move the unpaid liability to `Equity:Benefits:Unpaid`.

This is just my convention; choose whatever suits you. Some people put the year in the account name, such as `Equity:HCFSA:TY2026`, so each plan year has its own balance. That makes the importer harder: a reimbursement in the grace period (up to about two and a half months after the plan year, typically through March 15) might belong to last year's FSA or this year's.

## Tax year close-out

The year in the tax account name is there so I can see each year's tax in one place when I file. Once the return is done, I close that year. I do not want decades of `Expenses:Taxes:TY2010`, `TY2011`, and so on still showing up in the UI.

Say the stubs for 2024 added up to $5,200 federal and $1,400 state. You file, and the IRS owes you $400. That refund is a negative tax expense, not income.

```beancount
2025-04-15 * "Federal tax refund"
  Expenses:Taxes:TY2024:Federal     -400 USD
  Assets:Bank:Chase:Checking         400 USD
```

What is left in those accounts should now match the tax on the return. I drop a balance assertion, move the leftover to an accumulated account, and close the year.

```beancount
2025-04-16 balance Expenses:Taxes:TY2024:Federal  4800 USD
2025-04-16 balance Expenses:Taxes:TY2024:State    1400 USD

2025-04-16 * "Close TY2024 federal"
  Expenses:Taxes:TY2024:Federal         -4800 USD
  Expenses:Taxes:Accumulated:Federal     4800 USD

2025-04-16 * "Close TY2024 state"
  Expenses:Taxes:TY2024:State           -1400 USD
  Expenses:Taxes:Accumulated:State       1400 USD

2025-04-16 close Expenses:Taxes:TY2024
```

If you use the `close_tree` plugin, that last line closes the federal and state accounts under it, and they hide in Fava.

## Other currencies and units

In all my examples, every transaction is in USD, the primary currency in my ledger. Beancount can track other currencies and arbitrary units the same way.

Say you travel to Canada and exchange $300 for 400 Canadian dollars. Dinner that night is 80 CAD. The `@ 0.75 USD` is the rate on that exchange, and it is what makes the USD and CAD postings balance.

```beancount
2024-07-01 * "Exchange CAD"
  Assets:Cash:CAD               400 CAD @ 0.75 USD
  Assets:Bank:Chase:Checking   -300 USD

2024-07-02 * "Dinner in Toronto"
  Expenses:Food      80 CAD
  Assets:Cash:CAD   -80 CAD
```

A later price directive records the rate once it has moved. Fava can then show the remaining CAD in USD if you choose to, or how it fluctuates against the dollar over time.

```beancount
2024-08-01 price CAD 0.73 USD
```

You can track airline miles and credit card points too. A flight or a hotel booked with them barely shows up in other accounts, so you could underestimate what you spent if you only import the usual bills. For example, I keep United miles as their own commodity, `UA`, priced at about 1.2 cents. Earning them is income. Spending them is an expense.

```beancount
2024-04-10 * "United flight"
  Assets:Rewards:UA    2500 UA
  Income:Rewards:UA   -2500 UA

2024-04-10 price UA 0.012 USD

2024-08-21 * "Award flight"
  Assets:Rewards:UA       -25000 UA
  Expenses:Travel:Flights  25000 UA
```

Some people even track [vacation days accrued from a job](https://beancount.github.io/docs/command_line_accounting_cookbook/#vacation-hours). You can track anything you can write as a transaction, and get a fuller picture of your finances.
