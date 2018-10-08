## Implementing hyperloglog in Power BI
### a.k.a. counting lots of things

![Rainman](/uploads/cards.gif)

Have you ever needed to count things like transactions, users or cards? Of course you have, and naturally counting is very straightforward in the Power BI suite, just use `DISTINCTCOUNT` and off you go.

Have you ever needed to count lots and lots of things though? Maybe billions of things? You can still use the same function but the sheer amount of data itself may present a different challenge, especially if you're working with Power BI Desktop or Excel.

However, if you're willing to trade some accuracy for increased volume of data, or _information_, there's an excellent option. If you can live with an approximate count (say, ±2%) you can use the hyperloglog algorithm to estimate the number of unique things, or the cardinality of your data.

This post lays out how to implement hyperloglog in Power BI or Power Pivot, and in a way that is surprisingly easy.

###Hyperloglog to the rescue
