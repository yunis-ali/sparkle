# Haskell, meet Apache Spark

Over the last few years, [Apache Spark](https://spark.apache.org) has become
an increasingly appealing option in the industry for writing distributed
applications. For those not familiar with this framework, it lets you specify
a chain of high-level operations you want performed on (potentially huge)
distributed datasets and then splits the work into actual tasks to be
performed accross an entire cluster. The official APIs let you describe and
run such applications from Scala, Java, Python and R.

The goal of this post is to show how we, at Tweag, have been working towards
adding **Haskell** to this list.

## Spark basics

Here is a "Hello World" Spark application in Scala, stolen from the
[Quick Start](http://spark.apache.org/docs/latest/quick-start.html).

``` scala
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
import org.apache.spark.SparkConf

object SimpleApp {
  def main(args: Array[String]) {
    val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
    val conf = new SparkConf().setAppName("Simple Application")
    val sc = new SparkContext(conf)
    val logData = sc.textFile(logFile, 2).cache()
    val numAs = logData.filter(line => line.contains("a")).count()
    val numBs = logData.filter(line => line.contains("b")).count()
    println("Lines with a: %s, Lines with b: %s".format(numAs, numBs))
  }
}
```

The `main` function describes the operations to be performed on the
input file, which in this case is Spark's `README.md`. We first ask for the
input file to be loaded into two partitions. This yields a dataset (`logData`)
whose entries are lines from the original file. We then construct two datasets
out of `logData`, one with all the lines containing an "a" and the other with
all the lines containing a "b". Finally, we count the number of elements in
each and print those numbers to stdout.

From this very high-level description, reminiscent of how we are used to
writing programs in Haskell, Spark will schedule partition-level tasks to
be performed on your computer or accross an entire cluster, depending on how you run the application. If any of those tasks fail (e.g if a node gets
disconnected from the cluster), Spark will automatically reschedule this task
on another node, which means distributed applications written with Spark
are fault-tolerant out of the box.

In addition to common operations like `map`, `filter` or `fold` among others,
Spark also provides modules for large-scale graph processing, stream
processing, machine learning and dataframes, all based on this Resilient
Distributed Dataset (_RDD_) abstraction.

But how is any of this Haskell's concern?

## Distributed Haskell ?

There are some efforts to provide a platform for distributed computing
in Haskell, like [Cloud Haskell](http://haskell-distributed.github.io/).
However, none of the efforts went as far as Spark did, may it be in terms of
features, ease of use, adoption or overall maturity. Yet Haskell has a lot of
benefits to offer to the distributed computing world. Can't we find a way to
leverage Spark's well-tested distributed computing platform, while writing
Haskell code to describe our applications and even applying Haskell functions
over distributed datasets? The answer is **yes**.

The remainder of this post introduces _sparkle_, our answer to this problem,
and explains how we solved the various challenges that stood on our way. Be
warned that this is a very, very early stage demo, not a ready-to-use solution
for distributed computing in Haskell. We however believe that all the major
problems have been solved and thus wanted to share our work and excitment
with the community.

## Running Haskell code from Java

A Spark application's entry point usually is a Java or Scala program, so one of
the first problems to solve is: how can we just hand everything off to Haskell?

This one was actually reasonably easy to solve. Haskell's GHC compiler
exposes a few functions to manipulate its runtime system from C. On the other
hand, Java lets you call C functions (using JNI, the _Java Native Interface_).
If you put these two together, you can initialize GHC's RTS from Java.

In addition to that, the Haskell FFI lets one export a suitable Haskell
function to C, which we can then call from Java using _JNI_. While we won't
explore all the gory details involved in getting this to work in this post
(most of which reside in passing the right options to build a suitable shared
library with GHC's RTS and a Haskell function in it), we definitely plan on
documenting the entire procedure soon, along with a demo application.

Put simply, what we did on this front lets us call a Haskell function, like:

``` haskell
haskellMain :: IO ()
haskellMain = putStrLn "Hello from Haskell"
```

by exporting it in a shared library and loading the said library from Java,
making sure we initialize GHC's RTS before anything else.

## Calling Java from Haskell

While the previous section was about calling Haskell from Java, this one is
about the opposite direction: calling Java code from Haskell. Indeed, if we
want to write Spark applications in Haskell, we need access to Spark's
classes and methods, in order to initialize a Spark context, create
distributed datasets and perform operations on them.

Fortunately, JNI is not only about calling C from Java. It offers a bridge in
the opposite direction as well, very much like Haskell's FFI. What this means
is that any Java implementation comes with a C library for manipulating the
JVM, through dedicated [types](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/types.html)
and [functions](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/functions.html). A couple of `foreign import`s later, we were able to load
Java classes, instantiate them and call their methods, all of this from Haskell
code.

Here's how you would call the `toString()` method on a Java `Object`, using
our preliminary bindings to JNI:

``` haskell
toString :: JObject -> IO JObject
toString obj = do
  -- first, we ask the JVM to give us a handle on the Object class
  cls   <- findClass "java/lang/Object"

  -- we then get our hands on the toString method of that class
  -- with the given signature
  tostr <- findMethod cls "toString" "()Ljava/lang/String;"

  -- we finally invoke the toString method on the object we are given,
  -- which takes no argument (hence the empty list)
  callObjectMethod obj tostr []
```

At the moment we do not expose Haskell wrappers for all the types and
functions from the JNI and instead covered only what we needed to write
our initial _sparkle_ demo applications. 

Note: "signature" above refers to [the JVM's encoding of type signatures](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/types.html#wp16432).

## Online Latent Dirichlet Allocation, in Haskell

One of our goals was to be able to write the equivalent of
[this Scala application](https://gist.github.com/feynmanliang/3b6555758a27adcb527d)
in Haskell. A short description of what Latent Dirichlet Allocation consists in
can be found in [the accompanying blog post](https://databricks.com/blog/2015/09/22/large-scale-topic-modeling-improvements-to-lda-on-spark.html), but here's
a one-sentence summary: given a collection of text documents, the algorithm
tries to infer the different topics discussed in the said documents. The
"online" variation consists in learning an LDA model incrementally, instead of
having to go through the entire collection of documents before it can give us
a model. For the curious, this variation is described in [this paper](https://www.cs.princeton.edu/~blei/papers/HoffmanBleiBach2010b.pdf).

Most of the work for implementing this demo consisted in using our JNI bindings
from the previous section in order to expose the relevant Spark classes and
methods. Given that we have close to nothing in place to facilitate marshalling
between Haskell and Java at the moment, we wrote some "helper" code in Java
that we call out to. We hope to kill those bits as we go.

The end result is a function that gets loaded and called from Java:

``` haskell
sparkMain :: IO ()
sparkMain = do
    stopwords <- getStopwords

    conf <- newSparkConf "Spark Online Latent Dirichlet Allocation in Haskell!"
    sc   <- newSparkContext conf
    sqlc <- newSQLContext sc

    -- we load our collection of documents.
    -- each document is an element of the dataset 
    docs <- wholeTextFiles sc "documents/"
        >>= justValues
        >>= zipWithIndex

    -- we convert our collection of documents into a Spark DataFrame
    -- 1st column: "docId" = document id
    -- 2nd column: "text"  = document content
    docsRows <- toRows docs
    docsDF <- toDF sqlc docsRows "docId" "text"

    -- we setup the tokenizer to run on the "text"
    -- column and output the result in a "words" column
    tok  <- newTokenizer "text" "words"
    tokenizedDF <- tokenize tok docsDF

    -- we remove stopwords from the "words" column, putting
    -- the result in a "filtered" column
    swr  <- newStopWordsRemover stopwords "words" "filtered"
    filteredDF <- removeStopWords swr tokenizedDF

    -- we extract the appropriate features for LDA to run on,
    -- consisting in vectors (of numbers) for each document
    cv   <- newCountVectorizer vocabSize "filtered" "features"
    cvModel <- fitCV cv filteredDF
    countVectors <- toTokenCounts cvModel filteredDF "docId" "features"

    -- we finally run the online LDA algorithm on the features
    -- we've just extracted and extract a model of our document collection
    -- out of it
    lda  <- newLDA miniBatchFraction numTopics maxIterations
    ldamodel  <- runLDA lda countVectors

    -- this outputs the different "topics" inferred by the LDA model
    -- in a human-friendly way
    describeResults ldamodel cvModel maxTermsPerTopic

    -- dataset-dependent parameters
    where numTopics         = 10
          miniBatchFraction = 1
          vocabSize         = 600
          maxTermsPerTopic  = 10
          maxIterations     = 50

getStopwords :: IO [String]
getStopwords = fmap lines (readFile "stopwords.txt")

foreign export ccall sparkMain :: IO ()
```

```
>>> Topic #0
	atlanta -> 0.05909878836489215
	journal -> 0.03736792191830798
	constitution -> 0.03532890625936911
	moved -> 0.03254662858728971
	service -> 0.02101874084187926
	beach -> 0.01934327641726217
	washington -> 0.015744658734434487
	column -> 0.011533062032191725
	editor -> 0.01027877462336505
	coxnews -> 0.010027441771679654
-----
>>> Topic #1
	rubin -> 0.040416783706726876
	citigroup -> 0.015552806663132827
	enron -> 0.014951083930487013
	first -> 0.014679582063525697
	company -> 0.01296990190006682
	clinton -> 0.012818751484368918
	former -> 0.012721129987043
	policy -> 0.010760148602112128
	business -> 0.010315259378148107
	world -> 0.009556332963292368
-----
>>> Topic #2
	lindh -> 0.10086565288128005
	lawyers -> 0.02809974370319094
	documents -> 0.020200921183457624
	court -> 0.018622599410469412
	taliban -> 0.013285242560186242
	laden -> 0.012629363061761718
	filed -> 0.01222211815409107
	state -> 0.01131786198481124
	tuesday -> 0.010732103491090031
	lawyer -> 0.010634004656851694
-----
>>> Topic #3
	people -> 0.015649313114419607
	first -> 0.014152908458368226
	years -> 0.013390974178862767
	three -> 0.013091944852162668
	american -> 0.009765530939045816
	school -> 0.009240253041043294
	still -> 0.008550106452250891
	business -> 0.00838324576605961
	world -> 0.008210222606562219
	think -> 0.008046033467505353
-----
>>> Topic #4
	state -> 0.013993636365461115
	nuclear -> 0.013454967616138413
	president -> 0.011897250624609038
	government -> 0.011887679419735805
	officials -> 0.011538047974221234
	people -> 0.011470211845182649
	years -> 0.010494996047043077
	paint -> 0.01006800637627191
	federal -> 0.009595218276533429
	administration -> 0.009135440837571944
-----
>>> Topic #5
	democrats -> 0.048477016027800815
	republican -> 0.03372953186909447
	democratic -> 0.030479478447646494
	hockey -> 0.0301577399554842
	party -> 0.030124135903231007
	house -> 0.029891796692788394
	political -> 0.02552022631926487
	republicans -> 0.02349519155470418
	president -> 0.023354691038873904
	washington -> 0.014107623775767722
-----
>>> Topic #6
	times -> 0.023527296114520584
	undated -> 0.015653683177076237
	afghanistan -> 0.01273133526364827
	united -> 0.01242722615113162
	american -> 0.012094814854928493
	world -> 0.011546459807774698
	states -> 0.011226923415847802
	military -> 0.011124624720930143
	service -> 0.009628779935017106
	forces -> 0.009625671286829073
-----
>>> Topic #7
	enron -> 0.05909461214502476
	company -> 0.04107533978443965
	percent -> 0.020163300633107915
	stock -> 0.018054314856199252
	million -> 0.01519085855963182
	energy -> 0.014519899604770029
	business -> 0.013930016959937034
	market -> 0.013468130867634454
	companies -> 0.011941715419144001
	financial -> 0.010156805944316628
-----
>>> Topic #8
	russia -> 0.032427058124270935
	recovery -> 0.021903528020765204
	people -> 0.019697718592083194
	world -> 0.015250431491762355
	years -> 0.01463823505258555
	economic -> 0.014453659833025566
	austin -> 0.011801362135069394
	american -> 0.010444896534955691
	figure -> 0.010375499203498464
	european -> 0.01026866313368857
-----
>>> Topic #9
	engine -> 0.01997218494773725
	power -> 0.019643173099927382
	vehicles -> 0.018688305258300654
	vehicle -> 0.015415769691030158
	economy -> 0.013817767814548005
	energy -> 0.013559557886229668
	standards -> 0.013172974407801281
	model -> 0.013009609194445855
	drive -> 0.012138520071245934
	miles -> 0.011676754602531585
```

## Shipping Haskell closures to Java

## Demo: mapping a Haskell function over a dataset

## Future work














