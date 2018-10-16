## Implementing HyperLogLog in Power BI
### a.k.a. counting lots of things

![Rainman](/uploads/cards.gif)

Have you ever needed to count unique _things_ like transactions, users...or cards? Of course you have, and counting things is thankfully very straightforward in the Power BI suite, just use `DISTINCTCOUNT` and off you go.

Have you ever needed to count lots and lots of things though? Maybe billions of things? You can still use the same function but the sheer amount of data itself may present a different challenge, especially if you're working with Power BI Desktop or Excel.

However, if you're willing to trade some accuracy for more data points (or perhaps more _information_) there's an excellent option. If you can live with an error of say ±2%, you can use the [HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog) algorithm to estimate the number of distinct elements in your data set.

This post lays out how to implement HyperLogLog in Power BI or Power Pivot, and in a way that is surprisingly easy.

### HyperLogLog to the rescue
The HyperLogLog algorithm has been around since 2007, you can find the original research paper [here](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf) if you're that way inclined. It builds on prior work along a similar vein; the problem itself, i.e. counting lots of things, is nothing new.

The idea behind the LogLog family of algorithms is, as per Wikipedia, that "the cardinality of a multiset of uniformly distributed random numbers can be estimated by calculating the maximum number of leading zeros in the binary representation of each number in the set".

Thanks Wikipedia, that really cleared things up. I'll make an attempt at translating this into English:

"The number of different _things_ in a list of _non-unique random numbers_ with the _same binary length_ can be estimated by calculating the _maximum number of leading zeroes_ in the binary representation of that same number."

Any better? What's key here is the fact that in a _random_ binary number the chance of a zero in each position is 50/50, a coinflip basically. And, if I told you I just flipped 10 heads in a row you'd probably assume I'd been busy flipping for quite some time. So, just like when flipping coins, if we see a long sequence of leading zeroes chances are we've observed a lot of different numbers.

So, with random independent 0's and 1's it's not surprising to hear that the second half of the Wikipedia paragraph above reads "If the maximum number of leading zeros observed is n, an estimate for the number of distinct elements in the set is 2<sup>n</sup>."

|Binary | Most significant bit | Share of elements       | Approximate count |
|-------|----------------------|-------------------------|-------------------|
|1xxxx  | 1                    | 0.5&#x00B9 = 50%   | 2<sup>0</sup> = 1 |
|01xxx  | 2                    | 0.5&#x00B2 = 25%   | 2<sup>1</sup> = 2 |
|001xx  | 3                    | 0.5<sup>3</sup> = 12.5% | 2<sup>2</sup> = 4 |
|0001x  | 4                    | 0.5<sup>4</sup> = 6.25% | 2<sup>3</sup> = 8 |

0.5 x 0.5 x 0.5

Let's try an example, let's go back to counting cards.

Suit  Value Hash
H     2
H     3
H     4


If the number of leading zeroes is `n` then the number of distinct elements is `2n`


### Implementing
