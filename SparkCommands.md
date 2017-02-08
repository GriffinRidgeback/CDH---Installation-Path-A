# To run Spark

```
sudo -u spark spark-shell
```

# In the Spark shell
Type
```
sc.[TAB]
```
to see a list of methods available on the context.  __Note:__  You should be able to do this for any variable.

# Loading files in local mode
Start the shell:
```
spark-shell --master local[n|*] --driver-memory 2g
```

Load the data from the local file system using fully-qualified paths; tilde replacement doesn't work.  Check the path specified as output from the RDD creation.

_first_ is a good method to sanity check the load; _collect_ is good for bringing in small datasets as an array.

The act of creating an RDD does not cause any distributed computation to take place on the cluster.  Rather, RDDs define logical data sets that are intermediate steps in a computation.  Distributed computation occurs upon invoking an _action_ on an RDD, such as __count__.

# Scala anonymous functions

```
head.filter(x => !isHeader(x)).length
```
can be simplified to:
```
head.filter(!isHeader(_)).length
```

The underscore represents the argument to the anonymous function.

# Tuples

Used to create pairs, triples, and larger  collections of values of different types as a simple way to represent records.

```
val line = head(5)
```

is actually

```
val line = head.apply(5)
```

since array elements are accessed via the _apply_ function call and not special operators.

# Implicits

Converting the id variables and the matched boolean variable is pretty straightforward
once we know about the appropriate toXYZ conversion functions. Unlike the
contains method and split method that we worked with earlier, the toInt and
toBoolean methods aren’t defined on Java’s String class. Instead, they are defined in
a Scala class called StringOps that uses one of Scala’s more powerful (and arguably
somewhat dangerous) features: implicit type conversion. Implicits work like this: if you
call a method on a Scala object, and the Scala compiler does not see a definition for
that method in the class definition for that object, the compiler will try to convert
your object to an instance of a class that does have that method defined. In this case,
the compiler will see that Java’s String class does not have a toInt method defined,
but the StringOps class does, and that the StringOps class has a method that can
convert an instance of the String class into an instance of the StringOps class. The
compiler silently performs the conversion of our String object into a StringOps
object, and then calls the toInt method on the new object.

# More on Scala Implicits

Scala
In Scala the conversion to RDDs with special functions (e.g., to expose numeric functions
on an RDD[Double]) is handled automatically using implicit conversions. As
mentioned in “Initializing a SparkContext” on page 17, we need to add import
org.apache.spark.SparkContext._ for these conversions to work. You can see the
implicit conversions listed in the SparkContext object’s ScalaDoc. These implicits
turn an RDD into various wrapper classes, such as DoubleRDDFunctions (for RDDs
of numeric data) and PairRDDFunctions (for key/value pairs), to expose additional
functions such as mean() and variance().
Implicits, while quite powerful, can sometimes be confusing. If you call a function
like mean() on an RDD, you might look at the Scaladocs for the RDD class and notice
there is no mean() function. The call manages to succeed because of implicit conversions
between RDD[Double] and DoubleRDDFunctions. When searching for functions
on your RDD in Scaladoc, make sure to look at functions that are available in these
wrapper classes.

To convert
them all at once, we can use the slice method on the Scala Array class to extract a
contiguous subset of the array, and then use the map higher-order function to convert
each element of the slice from a String to a Double:

```
val rawscores = pieces.slice(2, 11)
rawscores.map(s => s.toDouble)
rawscores.map(_.toDouble)
```

To handle values like ?, do this:
```
def toDouble(s: String) = {
if ("?".equals(s)) Double.NaN else s.toDouble
}
val scores = rawscores.map(toDouble)
```

A simple technique is:
```
def parse(line: String) = {
val pieces = line.split(',')
val id1 = pieces(0).toInt
val id2 = pieces(1).toInt
val scores = pieces.slice(2, 11).map(toDouble)
val matched = pieces(11).toBoolean
(id1, id2, scores, matched)
}
val tup = parse(line)
```

We can retrieve the values of individual fields from our tuple by using the positional
functions, starting from _1, or via the productElement method, which starts counting
from 0. We can also get the size of any tuple via the productArity method:
```
tup._1
tup.productElement(0)
tup.productArity
```

Although it is very easy and convenient to create tuples in Scala, addressing all of the
elements of a record by position instead of by a meaningful name can make our code
difficult to understand. What we would really like is a way of creating a simple record
type that would allow us to address our fields by name, instead of by position. Fortunately,
Scala provides a convenient syntax for creating these records, called case
classes. A case class is a simple type of immutable class that comes with implementations
of all of the basic Java class methods,
```
case class MatchData(id1: Int, id2: Int,
scores: Array[Double], matched: Boolean)
```

This changes the last line of the method to be:
```
MatchData(id1, id2, scores, matched)
```

Allows access of fields by name
```
md.matched
md.id1
```

# Histogram
to count how many of the MatchData
records in parsed have a value of true or false for the matched field. Fortunately, the
RDD[T] class defines an action called countByValue that performs this kind of computation
very efficiently and returns the results to the client as a Map[T,Long]. Calling
countByValue on a projection of the matched field from MatchData will execute a
Spark job and return the results to the client:
```
val matchCounts = parsed.map(md => md.matched).countByValue()
```
Scala’s Map class does not have methods for sorting its contents on the keys or the values,
but we can convert a Map into a Scala Seq type, which does provide support for
sorting. Scala’s Seq is similar to Java’s List interface, in that it is an iterable collection
that has a defined length and the ability to look up values by index:
```
val matchCountsSeq = matchCounts.toSeq
matchCountsSeq.sortBy(_._1).foreach(println)
matchCountsSeq.sortBy(_._2).foreach(println)
matchCountsSeq.sortBy(_._1).reverse.foreach(println)
```

# Pair RDDs

In addition to the RDD[Double] implicit actions, Spark supports implicit type conversion
for the RDD[Tuple2[K, V]] type that provides methods for performing per-key
aggregations like groupByKey and reduceByKey, as well as methods that enable joining
multiple RDDs that have keys of the same type.

# Summary stats
```
parsed.map(md => md.scores(0)).stats()
```

Unfortunately, the missing NaN values that we are using as placeholders in our arrays
are tripping up Spark’s summary statistics. Even more unfortunate, Spark does not
currently have a nice way of excluding and/or counting up the missing values for us,
so we have to filter them out manually using the isNaN function from Java’s Double
class:
```
import java.lang.Double.isNaN
parsed.map(md => md.scores(0)).filter(!isNaN(_)).stats()
StatCounter = (count: 5748125, mean: 0.7129, stdev: 0.3887, max: 1.0, min: 0.0)
```

If we were so inclined, we could get all of the statistics for the values in the scores
array this way, using Scala’s Range construct to create a loop that would iterate
through each index value and compute the statistics for the column, like so:
```
val stats = (0 until 9).map(i => {
parsed.map(md => md.scores(i)).filter(!isNaN(_)).stats()
})
stats(1)
...
StatCounter = (count: 103698, mean: 0.9000, stdev: 0.2713, max:
```

# Creating Scala classes for Spark work
Note that we’re
marking this class as Serializable because we will be using instances of this class
inside Spark RDDs, and our job will fail if Spark cannot serialize the data contained
inside an RDD.

# Underscore parameters

_1 is a method name. Specifically tuples have a method named _1, which returns the first element of the tuple. So _._1 < _._1 means "call the _1 method on both arguments and check whether the first is less than the second".

_._1 calls the method _1 on the wildcard parameter _, which gets the first element of a tuple. Thus, sortWith(_._1 < _._1) sorts the list of tuple by their first element.

Additional information [here](https://gist.github.com/lenards/8aa8fb2e81c67971558c) and [here](http://docs.scala-lang.org/cheatsheets/)

# Pair RDDs

Example 4-2. Creating a pair RDD using the first word as the key in Scala
```
val pairs = lines.map(x => (x.split(" ")(0), x))
```

Example 4-5. Simple filter on second element in Scala
```
pairs.filter{case (key, value) => value.length < 20}
```

Sometimes working with pairs can be awkward if we want to access only the value
part of our pair RDD. Since this is a common pattern, Spark provides the mapVal
ues(func) function, which is the same as map{case (x, y): (x, func(y))}. We
will use this function in many of our examples.

reduceByKey() is quite similar to reduce(); both take a function and use it to combine
values. it returns a new RDD
consisting of each key and the reduced value for that key.

foldByKey() is quite similar to fold(); both use a zero value of the same type of the
data in our RDD and combination function.

Example 4-8. Per-key average with reduceByKey() and mapValues() in Scala
```
rdd.mapValues(x => (x, 1)).reduceByKey((x, y) => (x._1 + y._1, x._2 + y._2))
```

so, if we have two keys of 'panda' with val1 = (0, 1) and val2 = (1, 1), this becomes:
```
x._1 = 0 and y._1 = 1
x._2 = 1 and x._2 = 1
```

The result is a tuple {1, 2}; if there were more keys with the value 'panda' then this new tuple would become x1, y1 and the next tuple for key 'panda' would become x2, y2.
What happens, I think, is that _reduceByKey_ takes the first two entries off the RDD that match the key.  Then it creates a tuple for each of the two x and y values; think of (x1, y1) and (x2, y2) stacked atop one another and a vertical oval drawn through each of the x and y elements to create a new tuple with just x or y values.

# Word count
Example 4-10. Word count in Scala
```
val input = sc.textFile("s3://...")
val words = input.flatMap(x => x.split(" "))
val result = words.map(x => (x, 1)).reduceByKey((x, y) => x + y)
```

combineByKey() is the most general of the per-key aggregation functions. Most of the
other per-key combiners are implemented using it. Like aggregate(), combineBy
Key() allows the user to return values that are not the same type as our input data.


Example 4-13. Per-key average using combineByKey() in Scala
```
val result = input.combineByKey(
(v) => (v, 1),
(acc: (Int, Int), v) => (acc._1 + v, acc._2 + 1),
(acc1: (Int, Int), acc2: (Int, Int)) => (acc1._1 + acc2._1, acc1._2 + acc2._2)
).map{ case (key, value) => (key, value._1 / value._2.toFloat) }
result.collectAsMap().foreach(println(_))
```

# To create Scala projects in Eclipse

1. Need the m2eclipse-scala plugin

<pre>
Under “Help -> Install New Software”, enter “http://alchim31.free.fr/m2e-scala/update-site” and hit enter. You should see “Maven Integration for Eclipse -> Maven Integration for Scala IDE”.
</pre>

2. Need to configure remote repository so scala-arch can be found
<pre>
I had same problem and it was solved by adding a remote catalog: http://repo1.maven.org/maven2/archetype-catalog.xml
</pre>

3. Have to add the following dependency to pom.xml once project is created

<pre>
<dependency>
        <groupId>org.specs2</groupId>
        <artifactId>specs2-junit_${scala.compat.version}</artifactId>
        <version>2.4.16</version>
        <scope>test</scope>
    </dependency>
</pre> 

# Running Spark in the pseudo-cluster
Follow the instructions on the Hadoop page for setting up the pseudo cluster - they work!
In the spark shell, to access data on the "cluster", type this:
```
val configFile = sc.textFile("hdfs://localhost:9000/user/kevin.delia/input/yarn-site.xml")
```

# Running Yarn jobs in pseudo-distributed mode
Running on a Mac with hadoop 2.7.3 and Spark 2.1.0, running yarn jar jobs results in the following error:
```
/bin/bash: /bin/java: No such file or directory
```

This [link[(http://stackoverflow.com/questions/33968422/bin-bash-bin-java-no-such-file-or-directory) provides the solution but simply put, it is this:
```
This answer is applicable for Hadoop version 2.6.0 and earlier. Disabling SIP and creating a symbolic link does provide a workaround. A better solution is to fix the hadoop-config.sh so it picks up your JAVA_HOME correctly

In HADOOP_HOME/libexec/hadoop-config.sh look for the if condition below # Attempt to set JAVA_HOME if it is not set

Remove extra parentheses in the export JAVA_HOME lines as below. Change this

   if [ -x /usr/libexec/java_home ]; then
       export JAVA_HOME=($(/usr/libexec/java_home))
   else
       export JAVA_HOME=(/Library/Java/Home)
   fi
to

   if [ -x /usr/libexec/java_home ]; then
       export JAVA_HOME=$(/usr/libexec/java_home)
   else
       export JAVA_HOME=/Library/Java/Home
   fi
Restart yarn after you have made this change. More detailed info can be found here https://issues.apache.org/jira/browse/HADOOP-8717
```

Then this will work:
```
yarn jar /Users/kevin.delia/hadoop-2.7.3/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.3.jar wordcount input output
```

# Scala/Spark notes
Spark 2.1.0 is built with Scala 2.11.8 so the maven dependencies have to match up.  Scala has versions more recent than 2.11.8 but those were not used to build Spark and are usuable only for Scala, not Spark, development.

# Running Spark in a cluster
1.start-master.sh
```
start-master.sh 
starting org.apache.spark.deploy.master.Master, logging to /Users/kevin.delia/spark-2.1.0-bin-hadoop2.7/logs/spark-kevin.delia-org.apache.spark.deploy.master.Master-1-kevindelia-MBP.local.out
```
2. start a worker
```
./spark-class org.apache.spark.deploy.worker.Worker spark://kevindelia-MBP.local:7077
```
The worker should appear in the Spark Master UI (localhost:8080)

3. Connect the REPL
```
spark-shell --master spark://kevindelia-MBP.local:7077
```
Spark will create the __derby.log__ and __metastore_db__ in the directory from which you launch the command; good idea to run from $SPARK_HOME