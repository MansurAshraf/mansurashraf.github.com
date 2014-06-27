---
layout: post
title: "Scalding: Finding K Nearest Neighbors for fun and profit"
description: ""
category: Scala
tags: [Scalding,Hadoop,Scala]
---
{% include JB/setup %}

Imagine you run a popular e-commerce site focused on hip and trendy clothing for women and you got a whiz bang recommendation engine that recommends products based on customer's previous shopping history.
While the recommendation engine is doing a decent job recommending to regular customers, it suffers from cold start problem when new or anonymous users browse the site catalog.
In this blog we will attempt to solve the cold start problem by recommending similar products to the product customer is viewing by leveraging Scalding and a few machine learning algorithms.
 Scalding is a scala based abstraction over raw map reduce from twitter.

Before we start cranking code lets get some business rules out of the way.
Recommended products must be in the same category. For example if someone is viewing a waffle maker and we recommended him a towel, it would be considered a bad recommendation.
Recommended products must be in the same sub category as well. Our imaginary site sells twenty different types of jeans, if a customer is looking at a Slimming Bootcut, Rinse Wash jeans we should try to recommend other bootcut jeans.
Recommended products must be in the same price range. For example if someone is looking at a $20 dollar shoes, we won't recommend her 100$ shoes.
For simplicity sake we will assume that our product catalog is available in CSV format and our products have five attributes namely *DEPARTMENT, SUB_DEPARTMENT, PRODUCT, DESCRIPTION, REG_PRICE, SALE_PRICE*. A sample of product catalog is available here. Lets start by reading the product catalog in a scalding job and doing a self join on DEPARTMENT, SUB_DEPARTMENT. This partitioning of catalog will take care of the first business rule of recommending products only from the same category.

```

    /* * Schema of our product catalog */
    val inputSchema = ('DEPARTMENT, 'SUB_DEPARTMENT, 'PRODUCT, 'DESCRIPTION, 'REG_PRICE, 'SALE_PRICE)

    /* Duplicate schema used for self joining */
    val renameSchema = ('DEPARTMENT1, 'SUB_DEPARTMENT1, 'PRODUCT1, 'DESCRIPTION1, 'REG_PRICE1, 'SALE_PRICE1)

    /* Read in the catalog */  
    val productMatrix = Csv("input", separator = ",", fields = inputSchema, quote = "\"").read

    /* Read in the catalog a second time for joining */
    val productMatrixDuplicate = Csv("input", separator = ",", fields = inputSchema, quote = "\"").read.rename(inputSchema -> renameSchema)

    /** * Do a self join based on DEPARTMENT and SUB_DEPARTMENT */
    productMatrix.joinWithSmaller(('DEPARTMENT,'SUB_DEPARTMENT) -> ('DEPARTMENT1,'SUB_DEPARTMENT1), productMatrixDuplicate)
``

Once we have read in the catalog, we need to figure out how to define similarity between products. From our business rules we know that a product is similar if its in the same price range and also if its in the same category and sub category. Additionally we would like to recommend products of similar styles even within a sub category.  Lets take a look at a few of the records to figure out which columns are useful for similarity calculation.

``
    "DEPARTMENT","SUB_DEPARTMENT","PRODUCT","DESCRIPTION","REG_PRICE","SALE_PRICE"
    "women","shoes","Marc Fisher Shoes","Pacca Pumps shoes","75.00","64.99"
    "women","shoes","Bandolino Shoes","Bayard Wedge Sandals shoes","59.00","49.99"
    "women","shoes","Nine West Shoes","Rocha Platform Pumps shoes","79.00","59.99"
    "women","shoes","MICHAEL Michael Kors Shoes","Fulton Moc Flats shoes","110.00","79.99"
```

Looking at the data we can use REG_PRICE and SALE_PRICE to calculate similarity based on price range. We will calculate Tanimoto Distance between products using their REG_PRICE and SALE_PRICE. Discussion on Tanimoto Distance is out of scope but in practical terms when two items have same reg price and sale price, the result is 0.0. When they have nothing in common, it’s 1.0.

Looking at the data again we can see that the style information is embedded inside DESCRIPTION column. Unfortunately we cannot use Tanimoto Distance for description based similarity without converting and normalizing description into numerical values. Instead we will calculate NGram distance between product description. A good description of NGram distance is provided here pdf, but for our application NGram distance will return a value between 1 and 0. A value of 1 means the specified strings are identical and 0 means the string are maximally different. Luckily both Tanimoto Distance and NGram Distance calculation are provided in Apache Mahout and Lucene respectively, so we won't have to hand roll our own implementation.

Once we we have calculated Tanimoto and NGram distance, we can combine two distances by calculating how far they are from a perfect match and adding the difference together. An easy way to make sure that we have calculated every thing correctly is to calculate the the distance between the same product and making sure that the distance is 0.0

```
    /** * output Schema */
    val outputSChema = ('PRODUCT, 'PRODUCT1, 'Distance)

    /** * Map over the grouped fields and calculate distance  */
    .mapTo('* -> outputSChema) {
    in: (String, String, String, String, Double, Double, String, String, String, String, Double, Double) => calculateDistance(in)
    }

    /**
     * Calculates Tanimoto and NGram distance based on different product features and combine them together.
     * @param in * @return
     */
    def calculateDistance(in: (String, String, String, String, Double, Double, String, String, String, String, Double, Double)) = {
    val (_, _, p1_product, p1_description, p1_regPrice, p1_salePrice, _, _, p2_product, p2_description, p2_regPrice, p2_salePrice) = in

    val ngramDistance = 1 - MathUtils.round(ngram.getDistance(p1_description, p2_description).toDouble, SCALE)

    val p1_vector = new DenseVector(Array(p1_regPrice, p1_salePrice))
    val p2_vector = new DenseVector(Array(p2_regPrice, p2_salePrice))
    val tanimotoDistance = MathUtils.round(tanimotoDistanceMeasure.distance(p1_vector, p2_vector), SCALE)

    val distance = MathUtils.round((tanimotoDistance + ngramDistance), SCALE) val result = (p1_product, p2_product, distance) result }
```

In the end we can sort the result by distance and take the top K similar products

```
    .groupBy('PRODUCT) { g=>
     g.sortBy('Distance).take(3) }

     .write(Csv("output", separator = ",", fields = outputSChema))
```
Complete recommendation list based on the sample product catalog is here but lets look at the recommendation for one product and verify that all our business goals are met.

```
    //products
    "women","shoes","Rampage Shoes","Weatherly Booties shoes","59.99","35.99"
    "women","shoes","Marc Fisher Shoes","Pacca Pumps shoes","75.00","64.99"
    "women","shoes","Bandolino Shoes","Bayard Wedge Sandals shoes","59.00","49.99"
    "women","shoes","Nine West Shoes","Rocha Platform Pumps shoes","79.00","59.99"
    "women","shoes","MICHAEL Michael Kors Shoes","Fulton Moc Flats shoes","110.00","79.99"

    //top 3 recommendations for "Marc Fisher Shoes Pacca Pumps shoes"
     Nine West Shoes,0.48493
     Bandolino Shoes,0.73206
     MICHAEL Michael Kors Shoes,0.75641

```

As we can see that the top recommendation for "Marc Fisher Shoes Pacca Pumps" is "Nine West Shoes Rocha Platform Pumps". These two shoes are not only within similar price range but are also of same shoe style. Based on our result, it seems like we are able to meet all three of our business objectives even with a very small sample of data. The complete source code for Product Recommendation job is located here.
