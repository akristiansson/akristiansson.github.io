## Implementing HyperLogLog in Power BI
### a.k.a. counting lots of things

![Rainman](/uploads/cards.gif)

Have you ever needed to count unique _things_ like transactions, users...or cards? Of course you have, and counting things is thankfully very straightforward in the Power BI suite, just use `DISTINCTCOUNT` and off you go.

Have you ever needed to count lots and lots of things though? Maybe billions of things? You can still use the same function but the sheer amount of data itself may present a different challenge, especially if you're working with Power BI Desktop or Excel.

However, if you're willing to trade some accuracy for more data points (or perhaps more _information_) there's an excellent option. If you can live with an error of say ±2%, you can use the [HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog) algorithm to estimate the number of distinct elements in your data set.

This post lays out how to implement HyperLogLog in Power BI or Power Pivot, and in a way that is surprisingly easy.

### HyperLogLog to the rescue
The HyperLogLog algorithm has been around since 2007, see [here](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf) for the original research paper. It builds on similar prior work as the problem itself, i.e. counting lots of things, is nothing new.

The idea behind the LogLog family of algorithms is, as per Wikipedia, simply that "the cardinality of a multiset of uniformly distributed random numbers can be estimated by calculating the maximum number of leading zeros in the binary representation of each number in the set".

Thanks Wikipedia, that really cleared things up. I'm no mathematician (or linguist) but I'll make an attempt at translating this into English:

"The number of different _things_ in a list of _non-unique random numbers_ with the _same binary length_ can be estimated by calculating the _maximum number of leading zeroes_ in the binary representation of that same number."

Any better? What's key here is the fact that in a _random_ binary number the chance of a zero in each position is 50/50. And just like when flipping a coin, to produce a long sequence of leading zeroes you probably need to flip a lot of coins.

Let's try an example, let's go back to counting cards.

Suit  Value Hash
H     2
H     3
H     4


If the number of leading zeroes is `n` then the number of distinct elements is `2n`


### Implementing
