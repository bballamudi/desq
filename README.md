# DESQ

DESQ is a general-purpose system for frequent sequence mining. It features a
simple and intuitive pattern expression language to express various pattern
mining tasks and provides efficient algorithms for mining.

We recommend using the Scala API which allows to execute DESQ in a distributed
setup with Apache Spark. There is also a Java API for sequential processing.

## Content

1. [Quick Start](#quick-start)
2. [API](#api)
3. [I/O](#io)
4. [Pattern Expressions](#pattern-expressions)
5. [References](#references)

## Quick Start

You can either work with a DESQ release or build DESQ on your own.

#### Using a release

There are three kinds of [releases](https://github.com/rgemulla/desq/releases):

* `desq-master-<releasedate>-full.jar`: DESQ with all dependencies
* `desq-master-<releasedate>-no-spark.jar`: DESQ with all dependencies apart
  from Apache Spark (already contained in the classpath if you run DESQ on a
  Spark cluster)
* `desq-master-<releasedate>.jar`: just DESQ

To run a Spark shell to experiment with DESQ, use

```
spark-shell --jars path/to/desq-master-<releasedate>-no-spark.jar
```

TODO: how to run without Spark installed (usinf full jar?)

#### Build on your own

If you would like to build DESQ on your own, please execute the following steps.

```
git clone https://github.com/rgemulla/desq.git
cd desq
mvn clean package
```

Building DESQ with `mvn clean package -Pprovided` builds the `desq-<branch>-<releasedate>-no-spark.jar` with the excluded Apache Spark dependency.

TODO: how to run a shell with Spark with mvn (if Spark not installed)

## DESQ Examples

DESQ supports a variety of pattern mining tasks, some of which are highlighted
in this section.

#### n-Gram Mining

Consider the following input sequences, which can also be found in file
`data/readme/sequences.txt`.

```
Anna lives in Melbourne
Anna likes Bob
Bob lives in Berlin
Cathy loves Paris
Cathy likes Dave
Dave loves London
Eddie lives in New York City
```

Each line corresponds to an input sequence, each whitespace-separated word to an item.

The following piece of code mines frequent bigrams with a minimum support of 2
using DESQ's Spark API. The full example code can be found at
`src/main/scala/de/uni_mannheim/desq/examples/readme/DesqExample.scala`.

TODO: create a subfolder readme in DESQs example package and move the files there.

```scala
import de.uni_mannheim.desq.mining.spark._

// read the data
val sequences = sc.textFile("data/readme/sequences.txt")

// convert data into DESQ's internal format (DesqDataset)
val data = DesqDataset.buildFromStrings(sequences.map(s => s.split("\\s+")))

// create a Miner
val patternExpression = "(..)"
val minimumSupport = 2
val properties = DesqCount.createConf(patternExpression, minimumSupport)
val miner = DesqMiner.create(new DesqMinerContext(properties))

// do the mining; this creates another DesqDataset containing the result
val patterns = miner.mine(data)

// print the result
patterns.print()
```

You should get the following output if you run this example.
```
[lives in]@3
```

Here the [pattern expression](#pattern-expressions) `(..)` refers to bigram
mining. In general, pattern expressions define what type of patterns DESQ should
mine. If we use `(.){3}`, for example, DESQ mines 3-grams instead. And if we use
use `(.){2,4}`, DESQ mines all 2-grams, 3-grams, and 4-grams.

#### n-Gram Mining with Hierarchies

TODO: Rework example to only use hierarchies and introduce the hat operator (but stay with ngrams). Output should be PERSON lives in CITY

Consider the following additional hierarchy constraints stored in a file `data/readme/dictionary.json`. Please refer to the section about [I/O](#io) for details on the actual file format.

```
├── PERSON
│   ├── Anna
│   ├── Bob
│   ├── Cathy
│   ├── Dave
│   └── Eddie
├── CITY
│   ├── Melbourne
│   ├── Berlin
│   ├── Paris
│   ├── London
│   └── New York City
├── VERB
│   ├── lives
│   ├── likes
│   └── loves
├── PREP
    └── in
```

The dictionary requires that the input sequences are stored as integers instead of strings in order to have a mapping between items in the sequences and the dictionary. Therefore, we now use `data/readme/sequences.del`.

Suppose we are interested in mining frequent verbs optionally followed by a preposition that occur between a person and a city (`PERSON (VERB PREP?) CITY`) with a minimum support of two (`2`). Now, we load the `DesqDataset` together with the dictionary.

```scala
import de.uni_mannheim.desq.dictionary._
import de.uni_mannheim.desq.mining.spark._

val dictionary = Dictionary.loadFrom("data/readme/dictionary.json")

val data = DesqDataset.loadFromDelFile("data/readme/sequences.del", dictionary).copyWithRecomputedCountsAndFids()

val properties = DesqCount.createConf("PERSON (VERB PREP?) CITY", 2)
val miner = DesqMiner.create(new DesqMinerContext(properties))

val patterns = miner.mine(data)
patterns.print()
```

You should get the following output if you run this example.

```
[lives in]@3
[loves]@2
```

The full example code can be found at `src/main/scala/de/uni_mannheim/desq/examples/readme/DesqExampleWithDictionary.scala`.

#### TODO

A more advances tasks: e.g., words that occur frequently between a person and a city.


## I/O

The mining algorithms operate on a `DesqDataset`, which stores sequences
together with their corresponding frequencies. Additionally, a `Dictionary` is
included in a `DesqDataset` which arranges the items in a hierarchy.

Typically, input sequences are stored in files as global identifiers (e.g.
`data/readme/sequences.del`) which correspond to a dictionary (e.g.
`data/readme/dictionary.json`). Please refer to the documentation of
`DesqDataset` and `Dictionary` respectively for more details on the required
file formats.

It is also possible to build a `DesqDataset` directly from arbitrary input data
with `DesqDataset.build`. Furthermore, saving an existing `DesqDataset` with
`DesqDataset.save` and loading it again with `DesqDataset.load` is also
possible. The documentation of `DesqDataset` gives a full list of possibilities.

## Pattern Expressions

Given an appropriate hierarchy it is possible to define more powerful pattern
expressions. Generally, pattern expressions are based on regular expressions,
but additionally include capture groups (`()`) and generalizations (`^` and
`^=`). Thus, pattern expressions specify which subsequences to output (captured)
as well as the context in which these subsequences should occur (uncaptured).

#### Traditional Constraints

Pattern Expression | Description
:---: | :---:
`(...)` | 3-grams
`(.){3,5}` | 3-, 4-, and 5-grams
`(.).(.).(.)` | Skip 3-grams with gap 1
`[.*(.)]+` | All subsequences
`[.*(.)]{3,5}` | Length 3–5 subsequences
`(.)[.{0,3}(.)]+` | Bounded gap of 0–3
`(.)[.?.?(.) \| .?(.).? \| (.).?.?](.)` | Serial episodes (len 3, window 5)
`(.){5}` | Generalized 5-grams
`(a\|b)[.*(c)]*.*(d)` | Subsequences matching regex `[a\|b]c*d`

### Customized Constraints

Pattern Expression | Description
:---: | :---:
`([ADJ\|NOUN] NOUN)` | Noun modified by adjective or noun
`ENTITY (VERB+ NOUN+? PREP?) ENTITY` | Relational phrase between entities
`DigitalCamera[.{0,3}(.)]{1,4}` | Products bought after a digital camera
`([S\|T]).*(.).*([R\|T])` | Amino acid sequences that match `[S\|T].[R\|T]`

## References

K. Beedkar, R. Gemulla
**DESQ: Frequent Sequence Mining with Subsequence Constraints** [[pdf](http://dws.informatik.uni-mannheim.de/fileadmin/lehrstuehle/pi1/people/rgemulla/publications/beedkar16desq.pdf), [tech report](https://arxiv.org/abs/1609.08431)]
In ICDM (short paper), pp. 793-798, 2016