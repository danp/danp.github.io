---
layout: default
title: Introduction to Ledger
date: "2011-08-28"
aliases:
- /2011/08/28/introduction-to-ledger.html
---

[ledger]: http://www.ledger-cli.org/
[GitHub]: https://github.com/jwiegley/ledger
[this blog post]: http://bugsplat.info/2011-07-09-program-your-finances-reporting-for-fun-and-profit.html
[bucketwise]: https://github.com/jamis/bucketwise
[Java Blend]: http://www.javablendcoffee.com/
[Ethiopian Sidama Ardi]: http://javablendcoffee.myshopify.com/products/fair-trade-organic-ethiopian-natural
[Chemex]: http://www.chemexcoffeemaker.com/
[double-entry]: http://en.wikipedia.org/wiki/Double-entry_bookkeeping_system
[freenode]: http://freenode.net/
[1]: https://github.com/danp/ledger-scripts

Since July 30th, I've been using [ledger][] (on [GitHub][]) to track my personal finances. What did I use before? I'm ashamed to say nothing beyond my banks' sites.

I stumbled upon ledger via [this blog post][], linked from Hacker News. It took me a while to try it, for a few reasons. First, I had a weird notion that there would be a better time in my various "fiscal cycles" to start using such a thing. Should I wait until the end of the monthly statement periods for my various accounts? No, this was just procrastination. Second, I need to easily deal in both USD and CAD. I live in Canada and am paid in CAD but being a US citizen from Arizona I still have many dealings in the US that involve USD. My Canadian and US RBC accounts are linked and it's easy (and instant) to transfer between them, though I usually only go from my Canadian accounts to my US accounts. That had been a big barrier in trying things like [bucketwise][], though I do like its concept.

Once I researched ledger, decided it would accommodate my currency needs, and got over my procrastination, I started keeping a transaction journal in ledger's format. The format is pretty simple both for humans and machines. Here are two sample transactions from my journal:

    2011/08/21 Payday
        Assets:RBC:Canada:Checking             $1,000.00
        Income:Salary

    2011/08/22 Java Blend
        Expenses:Food:Coffee                       $7.88
        Assets:RBC:Canada:Checking

On 2011/08/21, I got paid. $1,000.00 went into my checking account from the `Income:Salary` account. As ledger is a [double-entry][] system, money must come from an account and go to an account and transactions must balance to $0. With those rules, the file format allows assumptions where reasonable, as with the `Payday` transaction: there is no amount on the `Income:Salary` line, so it's assumed to be $-1,000.00 as that's what's needed to make the transaction balance. On 2011/08/22, I went to [Java Blend][]. I got a latte and some [Ethiopian Sidama Ardi][] for use in my [Chemex][]. The receipt was for $7.88, so this transaction transfers that amount into my `Expenses:Food:Coffee` account from my checking account.

Keeping a journal doesn't directly involve ledger (the command) at all. Ledger is really a reporting tool; it never writes data to disk itself. You're responsible for creating the journal, then ledger can be used to extract reports from it. With the journal file format being simple, it's easy to either maintain it by hand or generate it from whatever sources you like. I've been using the included emacs mode which helps with completion along with some [simple scripts][1].

With the above transactions in a file called `finances.ledger`, here's how you would get some balances:

    $ ledger --file finances.ledger balance
                 $992.12  Assets:RBC:Canada:Checking
                   $7.88  Expenses:Food:Coffee
              $-1,000.00  Income:Salary
    --------------------
                       0

As expected, my checking account is at $992.18, I've spent $7.88 on coffee, and my employer is out $1,000.00 after paying me. Along with balances, it's often handy to see a list of postings affecting one or more accounts. Postings are the individual movements of money within transactions. Usually there are two postings per transaction (a credit and a debit) but there can be as many as needed. Here's a register report for my checking account with the same two transactions:

    $ ledger --file finances.ledger register Assets:RBC:Canada:Checking
    11-Aug-21 Payday                As:RBC:Canada:Checking    $1,000.00    $1,000.00
    11-Aug-22 Java Blend            As:RBC:Canada:Checking       $-7.88      $992.12

Each line shows the amount for the posting as well as a running balance for the account. This just scratches the surface, of course; there are many output options and ways to select accounts, postings, transactions and more.

There are two other commands I (well, really [my scripts][1]) use to help me in maintaining my journal file: `ledger print` and `ledger entry`.

`ledger print` does something I really like: it takes a journal file as input and prints it out in the canonical journal format. This makes it especially easy to get data into the journal file and then reformat it to look nice. Combined with the `--sort d` option to sort by date, it keeps the file in predictable order. Perfect for cleaning up and keeping reasonable diffs in a git repository, which is what another one of my scripts helps me with.

`ledger entry` helps generate transactions for adding to your journal. Again, though, it's up to you to actually add what it generates to your journal; ledger does not actually do any writing to disk. Based on the arguments you give it, `ledger entry` tries to find a similar previous transaction to use as a template. For example, if I went to Java Blend again but only got a latte, I might use it like so:

    $ ledger --file finances.ledger entry java 8/27 3.50
    2011/08/27 Java Blend
        Expenses:Food:Coffee                       $3.50
        Assets:RBC:Canada:Checking

You can see it figured out the full payee and reused the crediting and debiting accounts from before.

This is but a very basic introduction to ledger but maybe enough to pique your interest. If you want to find out more, check out the [ledger][] site. There's also `#ledger` on [freenode][] which the author and users (including me!) hang out in. In my next post on ledger, I'll discuss handling of both CAD and USD and other aspects of how I use it.
