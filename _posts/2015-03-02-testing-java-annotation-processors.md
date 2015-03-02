---
layout: post
title:  "Testing Java Annotation Processors"
date:   2015-03-02 05:07:00
categories: java annotationprocessors
---

Working with annotation processors recently, I came across an excellent series at[dr. macphail's trance](http://deors.wordpress.com/). 

*   [Code Generation using Annotation Processors in the Java language – part 1: Annotation Types](http://deors.wordpress.com/2011/09/26/annotation-types/)
*   [Code Generation using Annotation Processors in the Java language – part 2: Annotation Processors](http://deors.wordpress.com/2011/10/08/annotation-processors/)
*   [Code Generation using Annotation Processors in the Java language – part 3: Generating Source Code](http://deors.wordpress.com/2011/10/31/annotation-generators/)

It is aimed towards using annotations for code generation, but it gives a well thought out overview of using annotation processors in general. However, it is missing one piece that I feel is incredibly important to using annotation processors successfully - automated testing. What I needed was a plan for testing annotated processors.

## Approach

In general with unit testing, the goal is to test your single unit in isolation. However when your unit is a thin integration layer between a framework and your body of code, what we really need is integration level testing. An annotation processor fits this description easily - it should typically be a thin integration layer between the annotation processing framework and whatever code it is invoking. So we need to do more than test our processor in isolation - we need to make sure that the framework invokes it properly and that it interacts with the framework appropriately.

Before we address the integration test for the annotation processor, let's talk briefly about the rest of your code. As much code as possible should be separated from the annotation processor. This typically means anything that doesn't touch the ``javax.annotation.processing.*`` or ``javax.lang.model.*`` APIs. This code should then be unit tested as normal. That way, when we get to our annotation processor test, the only thing that we're testing is the code that interfaces with the processing and model APIs.

Since the annotation process is run by the compiler, we will use the Compiler API made available in Java 6 to test our annotations. The primary
