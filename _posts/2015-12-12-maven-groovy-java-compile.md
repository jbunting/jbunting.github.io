---
layout: post
title:  "Compiling Groovy and Java Simultaneously with Maven"
date:   2015-12-12 22:00:00
categories: groovy maven
---

I was reading a blog post today discussing [Gradle][gradle] vs [Maven][maven]. One of the assertions made was that Gradle made adding [Groovy][groovy] sources to the product incredibly easy while it would have been near impossible with Maven. While I agree that Gradle makes this incredibly easy, and Maven makes some things very difficult, this particular task is not that much harder with Maven. That being said, I regularly see developers that I work with scouring the internet to figure out how to do this. So I'd like to quickly share how I approach this problem in a good number of our projects. Here's what we do:

 [gradle]: http://gradle.org/
 [maven]: https://maven.apache.org/ 
 [groovy]: http://www.groovy-lang.org/

# Changes

In our root `pom.xml` we add this compiler plugin definition:

```
  <build>
    <plugins>
      <plugin>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.3</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
          <compilerId>groovy-eclipse-compiler</compilerId>
        </configuration>
        <dependencies>
          <dependency>
            <groupId>org.codehaus.groovy</groupId>
            <artifactId>groovy-eclipse-compiler</artifactId>
            <version>2.9.2-01</version>
          </dependency>
          <dependency>
            <groupId>org.codehaus.groovy</groupId>
            <artifactId>groovy-eclipse-batch</artifactId>
            <version>2.4.3-01</version>
          </dependency>
        </dependencies>
      </plugin>
    </plugins>
  </build>
```

and add this groovy dependency to our `dependencyManagement` section:

```
  <dependencyManagement>
    <dependencies>
      ....
      <dependency>
        <groupId>org.codehaus.groovy</groupId>
        <artifactId>groovy-all</artifactId>
        <version>2.4.5</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
```

You can still specify the target and source version of Java that you wish, and you may use an older version of Groovy. In order to use an older version of groovy update both the `groovy-all` dependency and the `groovy-eclipse-batch` dependency inside the compiler plugin.

At this point, simply mingle your Groovy and Java source side by side in `src/main/java` and `src/test/java`. Provided you use the proper extension (`.java` or `.groovy`) for each file the compiler will simply handle it.

# Explanation

The `maven-compiler-plugin` allows specifying compilers other than `javac`. So what we've done here is leveraged that feature. The Groovy Eclipse team has written a maven compiler plugin that leverages the Eclipse compiler to simultaneously compile both Groovy and Java sources.

For more details see the project page [here][groovy-eclipse-maven].

 [groovy-eclipse-maven]: https://github.com/groovy/groovy-eclipse/wiki/Groovy-Eclipse-Maven-plugin

