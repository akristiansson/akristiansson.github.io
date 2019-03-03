## Implementing HyperLogLog in Power BI
### a.k.a. counting lots of things

![Rainman](/uploads/cards.gif)

Have you ever needed to count unique _things_ like transactions or customers? Of course you have, and counting things is thankfully very straightforward in the Power Pivot universe, just use `DISTINCTCOUNT` and off you go.

Have you ever needed to count lots and lots of things though? Maybe billions of things? You can still use the same function but the sheer amount of data itself may present a different challenge, especially if you're working with Excel (or Power BI Desktop).

But, if you're willing to trade some accuracy for more data points (or more _information_ rather) there's an excellent option. If you can live with an error of say ±2%, you can use the [HyperLogLog](https://en.wikipedia.org/wiki/HyperLogLog) algorithm to estimate the number of distinct elements in your data set.

This post lays out how to implement HyperLogLog in Power Pivot or Power BI, and in a way that is surprisingly easy. And to be clear, we will maintain the ability to drill and slice our data just like we normally would. In fact, apart from some loss of accuracy we're making no concessions whatsoever, i.e. we're not introducing any additional complexity to the model.

We'll start with the why-this-works though, and please bear with me as it will get a bit geeky (or interesting, your choice).

### HyperLogLog to the rescue
The HyperLogLog algorithm has been around since 2007, you can find the original research paper [here](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf) if you're that way inclined. It builds on prior work along a similar vein; the problem itself, counting lots of things, is nothing new.

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

First things first, unless our ID's just happen to be random integers (usually 32 or 64-bit) we need transform them into just that. Unfortunately, there's really no obvious way of generating these using Power BI or Power Query (that's to say there _is_ a way, it's just a bit clunky). Most of the time you can do this first step in your database however, which is what we're going to do using SQL Server.

My inspiration for this solution came from a [pure SQL implementation of HyperLogLog](https://www.periscopedata.com/blog/hyperloglog-in-pure-sql) and from there we're going to borrow the first few steps, namely creating buckets and calculating the most significant bit for each bucket. We will tweak the SQL slightly (this was originally done in PostgreSQL) but I would advise you to read this post for a more detailed explaination of each step.

What we want to do first is to generate a numeric hash of the order number. A hash is a random number or string but the output is always the same for the same input, i.e. in our use case different lines on the same order will generate the same hash. There's no obviuos way of generating a numeric hash in SQL Server so we'll do this in a few different steps.

```SQL
  hashbytes('sha2_256', cast(SalesOrderNumber as varchar(max))
```

[Hashbytes](https://docs.microsoft.com/en-us/sql/t-sql/functions/hashbytes-transact-sql?view=sql-server-2017) is a function which will generate (in this case) a 256-bit string, that's too long to convert to an integer so we'll use the following cast to 'truncate' the value at 4 bytes (32 bits, matching SQL Servers `integer` type).

```SQL
  cast(hashbytes('sha2_256', cast(SalesOrderNumber as varchar(max)) as binary(4))
```

This then, is the binary version of our hash. To convert this into an integer we'll do a bitwise & operation with 2^31-1 which is the largest 32-bit integer (i.e. 32 1's). This will also force all integers to be positive, which is a requirement later on when calculate the logarithm of them.

```SQL
  cast(hashbytes('sha2_256', cast(SalesOrderNumber as varchar(max)) as binary(4)) & 2147483647
```

Ok, we're nearly there, we're nearly at the point where the DAX/PowerPivot magic starts. Still with me?

In this case we'll aim for an error rate around ±5% so we'll use 512 buckets, the bucket numbers will range from 0-511 so to generate a bucket number we'll use the following:

```SQL
  cast(hashbytes('sha2_256', cast(SalesOrderNumber as varchar(max)) as binary(4)) & 511 as bucket_number
```

Our query will be grouped by this bucket number along with any other attributes we want to capture. To calculate the most significant bit by bucket number we'll use this:

```SQL
31 - floor(log(min(cast(hashbytes('sha2_256', cast(SalesOrderNumber as varchar(max))) as binary(4)) & 2147483647), 2)) as bucket_msb
```



cast(hashbytes('sha2_256', cast(fs.SalesOrderNumber as varchar(max))) as binary(4))

will most likely be quicker if nothing else) but we'll stick to Power Query, i.e. M, for now.

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
