---
layout: post
title:  "Using SLF4J in Maven Plugins"
date:   2012-08-06 06:00:00
categories: maven slf4j 
---

Recently a co-worker said to me, "Why is slf4j complaining about 'no binding' when I run a build?" My response, quite unimpressively, was "Yeah...it does that.". See, we have a process that exists in a common library. There are several ways to invoke this process, including a REST API in a webapp, a CLI application, and a maven plugin. That common library uses slf4j for logging. The webapp and the CLI application log everything just fine using logback and slf4j-simple respectively. The maven plugin doesn't. So when the follow-up question of, "Well how do I see my log statements?" was posed, it got me to thinking. There's no reason that we can't write an slf4j implementation that can be used by a maven plugin. So that's [what I did](https://bitbucket.org/peachjean/slf4j-mojo). I'm going to try to detail a little bit of what I did, why, and how to use it.

## The Approach

Adding a new slfj4 implementation is pretty straightforward. You simply need to implement [ILoggerFactory](https://bitbucket.org/peachjean/slf4j-mojo/src/0.3/src/main/java/net/peachjean/slf4j/mojo/MojoLoggerFactory.java) and [Logger](https://bitbucket.org/peachjean/slf4j-mojo/src/0.3/src/main/java/net/peachjean/slf4j/mojo/MojoLoggerAdapter.java). Then you add a copy of [StaticLoggerBinder](https://bitbucket.org/peachjean/slf4j-mojo/src/0.3/src/main/java/org/slf4j/impl/StaticLoggerBinder.java) that refers to your implementations to your project.

The one issue that I ran into is that, while loggers must be statically available, maven only makes the mojo logger available at an instance level. My first attempt to address this was the use a static thread local variable that holds the mojo logger. The downside of this is that the mojo has to be single-threaded. Then, after thinking about it, and learning some details about the maven classloading strategy, I decided to make it a static AtomicReference. This means that threads launched from within the plugin will still have access to the logger. In addition, because the slf4j-mojo library is specified as a dependency of the plugin, the static value is local to the plugin execution.

The last bit is to ensure that we provide our adapter with the plugin Log object during plugin execution. For this, I created an [AbstractLoggingMojo](https://bitbucket.org/peachjean/slf4j-mojo/src/0.3/src/main/java/net/peachjean/slf4j/mojo/AbstractLoggingMojo.java). It sets the "current" maven logger before execution, and unsets it afterwards.

## Adding Logging to your Mojo

First, include the slf4j-mojo dependency in your pom file (yes, it's deployed to maven central).

``` xml
  <dependency>
    <groupId>net.peachjean.slf4j.mojo</groupId>
    <artifactId>slf4j-mojo</artifactId>
    <version>0.3</version>
  </dependency>
```

Second, instead of extending AbstractMojo, extend AbstractLoggingMojo.

``` java
  public class MyMojo extends AbstractLoggingMojo {
    @Override
    protected void executeWithLogging() throws MojoExecutionException,MojoFailureException{
      getLog().info("HERE I AM!");
    }
  }
```

There it is. You can easily use your slf4j-enabled library in your maven plugin without sacrificing logging capabilities.

