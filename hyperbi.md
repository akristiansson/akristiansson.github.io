## Implementing HyperLogLog in Power BI
### a.k.a. counting lots of things

![Rainman](/uploads/cards.gif)

Have you ever needed to count _things_ like transactions, users or cards? Of course you have. Counting is naturally very straightforward in the Power BI suite, just use `DISTINCTCOUNT` and off you go.

Have you ever needed to count lots and lots of things though? Maybe billions of things? You can still use the same function but the sheer amount of data itself may present a different challenge, especially if you're working with Power BI Desktop or Excel.

However, if you're willing to trade some accuracy for increased volume of data points, or rather _more information_, there's an excellent option. If you can live with an approximate count, say ±2%, you can use the [HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog) algorithm to estimate the number of unique things in your data set, or the _estimated cardinality_ of your data.

This post lays out how to implement HyperLogLog in Power BI or Power Pivot, and in a way that is surprisingly easy.

### HyperLogLog to the rescue
The HyperLogLog algorithm

### Implementing
