---
layout: post
title: "Testing MapReduce with MRUnit"
description: ""
category: Hadoop
tags: [MRUnit,Hadoop,MR]
---
{% include JB/setup %}

Testing and debugging multi threaded programs is hard. Now take the same programs and massively distribute them across multiple JVMs deployed on a cluster of machines and the complexity goes off the roof.
One way to overcome this complexity is to do testing in isolation and catch as many bugs as possible locally. MRUnit is a testing framework that lets you test and debug Map Reduce jobs in isolation without spinning up a Hadoop cluster.
In this  blog post we will cover various features of MRUnit by walking through a simple MapReduce job.

Lets say we want to take the input below and create an inverted index using MapReduce.

**Input**

```
www.kohls.com,clothes,shoes,beauty,toys
www.amazon.com,books,music,toys,ebooks,movies,computers
www.ebay.com,auctions,cars,computers,books,antiques
www.macys.com,shoes,clothes,toys,jeans,sweaters
www.kroger.com,groceries
```

**Expected output**
```
antiques      www.ebay.com
auctions      www.ebay.com
beauty        www.kohls.com
books         www.ebay.com,www.amazon.com
cars          www.ebay.com
clothes       www.kohls.com,www.macys.com
computers     www.amazon.com,www.ebay.com
ebooks        www.amazon.com
jeans         www.macys.com
movies        www.amazon.com
music         www.amazon.com
shoes         www.kohls.com,www.macys.com
sweaters      www.macys.com
toys          www.macys.com,www.amazon.com,www.kohls.com
groceries     www.kroger.com
```

below are the Mapper and Reducer that do the transformation


```
public class InvertedIndexMapper extends MapReduceBase implements Mapper<LongWritable, Text, Text, Text> {
public static final int RETAIlER_INDEX = 0;

 @Override
 public void map(LongWritable longWritable, Text text, OutputCollector<Text, Text> outputCollector, Reporter reporter) throws IOException {
  final String[] record = StringUtils.split(text.toString(), ",");
  final String retailer = record[RETAIlER_INDEX];
  for (int i = 1; i < record.length; i++) {
   final String keyword = record[i];
   outputCollector.collect(new Text(keyword), new Text(retailer));
   }
  }
 }
public class InvertedIndexReducer extends MapReduceBase implements Reducer<Text, Text, Text, Text> {
@Override
 public void reduce(Text text, Iterator<Text> textIterator, OutputCollector<Text, Text> outputCollector, Reporter reporter) throws IOException {
  final String retailers = StringUtils.join(textIterator, ',');
  outputCollector.collect(text, new Text(retailers));
  }
 }
```

Implementation details are not really important but basically Mapper gets a line at a time, splits the line and emits key value pairs where Key is a category of product and value is the website which is selling the product.
For example line *retailer,category1,category2* will be emitted as *(category1,retailer)* and *(category2,retailer)*.
Reducer gets a key and a list of values, transforms the list of values to a comma delimited String and emits the key and value out.

Now lets use MRUnit to write various tests for this Job. Three key classes in MRUnits are MapDriver for Mapper Testing, ReduceDriver for Reducer Testing and MapReduceDriver for end to end MapReduce Job testing.
This is how we will setup the Test Class.

```
public class InvertedIndexJobTest {

 private MapDriver<LongWritable, Text, Text, Text> mapDriver;
 private ReduceDriver<Text, Text, Text, Text> reduceDriver;
 private MapReduceDriver<LongWritable, Text, Text, Text, Text, Text> mapReduceDriver;

 @Before
 public void setUp() throws Exception {

 final InvertedIndexMapper mapper = new InvertedIndexMapper();
 final InvertedIndexReducer reducer = new InvertedIndexReducer();

 mapDriver = MapDriver.newMapDriver(mapper);
 reduceDriver = ReduceDriver.newReduceDriver(reducer);
 mapReduceDriver = MapReduceDriver.newMapReduceDriver(mapper, reducer);
 }
}
```

MRUnit supports two styles of testings. First style is to tell the framework both input and output values and let the framework do the assertions, second is the more traditional approach where you do the assertion yourself.
Lets write a test using the first approach.

```
@Test
 public void testMapperWithSingleKeyAndValue() throws Exception {
 final LongWritable inputKey = new LongWritable(0);
 final Text inputValue = new Text("www.kroger.com,groceries");

 final Text outputKey = new Text("groceries");
 final Text outputValue = new Text("www.kroger.com");

 mapDriver.withInput(inputKey, inputValue);
 mapDriver.withOutput(outputKey, outputValue);
 mapDriver.runTest();

 }
```

In the test above we tell the framework both input and output Key and Value pairs and the framework does the assertion for us. This test can be written in a more traditional way as follow

```
@Test
 public void testMapperWithSingleKeyAndValueWithAssertion() throws Exception {
 final LongWritable inputKey = new LongWritable(0);
 final Text inputValue = new Text("www.kroger.com,groceries");
 final Text outputKey = new Text("groceries");
 final Text outputValue = new Text("www.kroger.com");

 mapDriver.withInput(inputKey, inputValue);
 final List<Pair<Text, Text>> result = mapDriver.run();

 assertThat(result)
 .isNotNull()
 .hasSize(1)
 .containsExactly(new Pair<Text, Text>(outputKey, outputValue));
}
```

Sometimes Mapper emits multiple Key Value pairs for a single input. MRUnit provides a fluent API to support this use case. Here is an example

```
@Test
 public void testMapperWithSingleInputAndMultipleOutput() throws Exception {
 final LongWritable key = new LongWritable(0);
mapDriver.withInput(key, new Text("www.amazon.com,books,music,toys,ebooks,movies,computers"));
 final List<Pair<Text, Text>> result = mapDriver.run();

 final Pair<Text, Text> books = new Pair<Text, Text>(new Text("books"), new Text("www.amazon.com"));
 final Pair<Text, Text> toys = new Pair<Text, Text>(new Text("toys"), new Text("www.amazon.com"));

assertThat(result)
 .isNotNull()
 .hasSize(6)
 .contains(books, toys);
}
```

You write the test for the reduce exactly the same way.

```
@Test
 public void testReducer() throws Exception {
final Text inputKey = new Text("books");
final ImmutableList<Text> inputValue = ImmutableList.of(new Text("www.amazon.com"), new Text("www.ebay.com"));

reduceDriver.withInput(inputKey,inputValue);
final List<Pair<Text, Text>> result = reduceDriver.run();
final Pair<Text, Text> pair2 = new Pair<Text, Text>(inputKey, new Text("www.amazon.com,www.ebay.com"));

 assertThat(result)
 .isNotNull()
 .hasSize(1)
 .containsExactly(pair2);
 }
```

Finally you can use MapReduceDriver to test your Mapper, Combiner and Reducer together as a single job. You can also pass multiple key value pairs as input to your job. Test below demonstrate MapReduceDriver in action

```
@Test
 public void testMapReduce() throws Exception {
 mapReduceDriver.withInput(new LongWritable(0), new Text("www.kohls.com,clothes,shoes,beauty,toys"));
 mapReduceDriver.withInput(new LongWritable(1), new Text("www.macys.com,shoes,clothes,toys,jeans,sweaters"));

final List<Pair<Text, Text>> result = mapReduceDriver.run();

final Pair clothes = new Pair<Text, Text>(new Text("clothes"), new Text("www.kohls.com,www.macys.com"));
final Pair jeans = new Pair<Text, Text>(new Text("jeans"), new Text("www.macys.com"));

assertThat(result)
 .isNotNull()
 .hasSize(6)
 .contains(clothes, jeans);
 }
```

This covers the basic features of MRUnit. It contains tons of other feature that we have not cover here such as testing counters, passing a mock configuration etc but my favorite feature is that it allows me to put a break point in my mapper and reducer and debug through the code if i need to. Bottom line is that MRUnit is an invaluable tool for anyone working with Map Reduce and Hadoop and it certainly takes a lot of pain out of testing MapReduce jobs.
