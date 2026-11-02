# Building a Beancount Ledger

This is a distillation of my personal ledger work. It does not include my actual personal or financial data. The goal is to help someone else build a similar system for their own finances, with the help of a coding agent.

I have used a lot of accounting systems, free and commercial. None of them gave me enough room to customize. With AI available, I personally found that building on a bare-bones text accounting system is actually easier, and much more flexible. I chose [Beancount](https://github.com/beancount/beancount) because some of the best programmers I know recommended it.

## Who this is for

You are a programmer who enjoys the intersection of math, programming, and finance. Given enough time, you could write this whole system yourself. An agent only speeds that up. You still read what it produces, and send it back when you see duplicated code or a library that will not scale. Importing your financial records and reconciling them is a satisfying way to decompress from day-to-day work.

If you just want to track spending or net worth, a normal budgeting app is probably simpler.

## How these notes are organized

We start with Beancount's double-entry accounting. Then we go through how financial statements get imported. Last comes stock portfolio performance.

If you already track your portfolio with another solution — a spreadsheet, a brokerage, or a portfolio-tracking tool — you can skip Beancount entirely. The performance chapter only needs dated cash flows and current positions, so it stands on its own.

## Contents

1. [Beancount basics](beancount.md) — the slice of the tool this system uses.
2. Importing
   - [Banks, cards, and pay stubs](importing-bank.md) — CSV imports, pay stubs, and balance assertions.
   - [Brokerages and other assets](importing-market.md) — lots, trades, prices, crypto, and property.
   - [Stock options and RSUs](importing-private.md) — equity grants before and after IPO.
3. [Portfolio performance](performance.md) — cash flows, XIRR, and benchmarks.

Source: [github.com/atppp/ledger](https://github.com/atppp/ledger)
