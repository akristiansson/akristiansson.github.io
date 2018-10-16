## Implementing HyperLogLog in Power BI
### a.k.a. counting lots of things

![Rainman](/uploads/cards.gif)

Have you ever needed to count unique _things_ like transactions, users...or cards? Of course you have, and counting things is thankfully very straightforward in the Power BI suite, just use `DISTINCTCOUNT` and off you go.

Have you ever needed to count lots and lots of things though? Maybe billions of things? You can still use the same function but the sheer amount of data itself may present a different challenge, especially if you're working with Power BI Desktop or Excel.

However, if you're willing to trade some accuracy for more data points (or perhaps more _information_) there's an excellent option. If you can live with an error of say ±2%, you can use the [HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog) algorithm to estimate the number of distinct elements in your data set.

This post lays out how to implement HyperLogLog in Power BI or Power Pivot, and in a way that is surprisingly easy.

### HyperLogLog to the rescue
The HyperLogLog algorithm has been around since 2007, you can find the original research paper [here](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf) if you're that way inclined. It builds on prior work along a similar vein; the problem itself, i.e. counting lots of things, is nothing new.

The main principle behind the LogLog family of algorithms is as follows...

> The cardinality of a multiset of uniformly distributed random numbers can be estimated by calculating the maximum number of leading zeros in the binary representation of each number in the set.

Thanks Wikipedia, that really cleared things up!

What's key here is the fact that in a _random binary number_ the chance of a zero in any given position is 50/50, a coinflip basically. And, if I told you I just flipped 10 heads in a row you'd probably assume I'd been busy flipping for quite some time. So, just like when flipping coins, if we've observed a long sequence of leading zeroes in a group of numbers, chances are we've searched through a lot of different ones.

So, with random independent 0's and 1's it's not surprising to hear that the second half of the Wikipedia paragraph above reads:

> If the maximum number of leading zeros observed is n, an estimate for the number of distinct elements in the set is 2<sup>n</sup>.

|Binary | Most significant bit | Share of elements       | Approximate count |
|-------|----------------------|-------------------------|-------------------|
|1xxxx  | 1                    | 0.5&#x00B9 = 50%   | 2<sup>0</sup> = 1 |
|01xxx  | 2                    | 0.5&#x00B2 = 25%   | 2<sup>1</sup> = 2 |
|001xx  | 3                    | 0.5<span><sup>3</sup></span> = 12.5% | 2<sup>2</sup> = 4 |
|0001x  | 4                    | 0.5<sup>4</sup> = 6.25% | 2<sup>3</sup> = 8 |

This approximation might strike you as quite naive, and it is. HyperLogLog doesn't quite end there, to minimise variance we group the numbers into buckets and work with a harmonised mean of the most significant bit. The more buckets, the more accurate but also more data to keep in memory ultimately.

The number of buckets will be a factor of 2n, i.e. 1, 2, 4, 8 and so on. To get an error rate around ±5% we have to go up to 512 buckets. Of course, if we don't have enough numbers we won't fill all buckets but that's OK.

If you have a desired error rate in mind you can resolve the number of buckets as follows:

### Implementing

First step in implementing this solution is to get our hands on some random numbers. Chances are we're counting something like users or orders with an identifier attached, and in this example we'll count unique users from the Contoso `FactOnlineSales` table. More specifically, we're going to aggregate sales and quantity by day and product while maintaining an rough count of customers.

We're going to use this query as our source data (ca. 12m rows)
```sql
select
  DateKey
 ,ProductKey
 ,CustomerKey
 ,SalesAmount
 ,SalesQuantity
from dbo.FactOnlineSales
```

First things first, unless our ID's just happen to be random integers (usually 32 or 64-bit) we need transform them into just that. Unfortunately, there's really no obvious way of generating these using Power BI or Power Query (that's to say there _is_ a way, it's just a bit clunky). Often, with a bit of luck or ingenuity, you can do this first step in your database (which will most likely be quicker if nothing else) but we'll stick to Power Query, i.e. M, for now.

M can compress binary data as gzip, and the gzip footer just so happens to contain a 32-bit checksum which we can access and use as a random hash. The footer is 8 bytes, the first 4 of which contain the checksum. We convert the gzip binary to a list in order to extract the relevant bytes and in turn convert this to a non-negative number.

```
let
  CompressText = (x as text) as binary =>
    Binary.Compress(
      Text.ToBinary(x, BinaryEncoding.Base64),
      Compression.GZip
    ),
  GZipChecksum = (x as text) as binary =>
    Binary.FromList(
      List.FirstN(
        List.LastN(
          Binary.ToList(CompressText(x))
          ,8
        )
        ,4
      )
    ),
  CalculateHash = (x as text) as number =>
    BinaryFormat.UnsignedInteger32(
      GZipChecksum(x)
    ),
  Source = [...]
in
  Source
```

What we're going to do using this function is to Assign a `hash_value` to each user ID (i.e. `CustomerKey`)

We're then going to   
- Use the hash to create a bucket number (`bucket_number`)
- Group the data by `bucket_number`, `DateKey` and `ProductKey` and add aggregates for `SalesAmount` and `SalesQuantity` as well as the _minimum_ `hash_value`
- Calculate the most significant bit (`bucket_msb`) from the `hash_value`
