## Implementing hyperloglog in Power BI
### a.k.a. counting lots of things

![Rainman](/uploads/cards.gif)

Have you ever needed to count things like transactions, users or cards? Of course you have, and--of course--counting is very straightforward in Power BI, just use `COUNT` or `DISTINCTCOUNT` and off you go.

Have you ever needed to count lots and lots of things though? Like maybe billions of things? You can use the same `COUNT` functions but the sheer amount of data itself may present a different challenge, especially if you're using Power BI Desktop or Excel.

This post lays out how to (easily) implement hyperloglog in Power BI or Power Pivot
