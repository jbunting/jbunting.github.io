---
layout: post
title:  "On Acceptance Test Suites"
date:   2015-03-01 17:40:00
categories: testing
---

Testing is key.  In this article, I'm going to talk about acceptance test suites for libraries.  The particular type of libraries that I am interested in talking about are ones that provide an abstraction layer.  We have two of these types of libraries at my day job.  

The first is an abstraction for querying key-value data stores (Cassandra, HBase, Accumulo).  It allows us to write our application against one API while supporting it with the multiple databases that our customers use. Clients of this API write data model mappers and run queries. Implementations adapt those data model mappers and queries to the database-specific API.

The second is an abstraction for running simple analytics on various distributed processing platforms (Hadoop, Storm).  It allows us to have write our analytics against a common API while supporting it on the multiple platforms that our present and future customers wish to run. Clients of this API write small tasks and link them together in a data flow.  Implementations run these tasks on their specific distributed processing platform.

Now, you may or may not agree with the need or wisdom for either or both of these abstraction layers.  But let's put that aside a moment, because eventually, most developers encounter a need for some sort of abstraction layer.  In both cases, we needed a common acceptance test suite.  Why?  Because we have multiple code bases, all backed by different systems, that we expect the same behavior from.  Because when we discover a bug on one implementation we want to write a regression test on all implementations to ensure that bug doesn't exist elsewhere.  The goal of the common acceptance test suite is to write a single test suite that can be simply and easily, with the minimum of adapters, run against any implementation.

Step 1

Write a test harness.  In this case I am referring to a test harness as a utility that your clients can use in their tests to avoid having to tie their testing infrastructure to a particular implementation.  It is an implementation that:

 * uses minimal resources
 * allows checks for correctness
 * exhibits "perfect" behavior - does exactly what the API claims it will do
 * often is not useful outside of the test environment

While at first this test harness does not appear to be a part of the acceptance test suite, it will become our integral first step.

Step 2

Write a common interface to adapt the test harness to an implementation.

Step 3

Write a series of acceptance tests, exercising different aspects of your abstraction layer, against the test harness.

Step 4

Write a master "suite" class that includes all of the acceptance tests.  This suite should be written as a test case that your implementations can extend by providing the implementation in step 2.

Step 4

Write your first implementation suite.  This should be a minimal amount of glue code to run your tests from Step 3 against the actual test harness from Step 1.

Step 5

Write an implementation suite for each implementation.  Glue code that implements the interface in Step 2.

