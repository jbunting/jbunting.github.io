---
layout: post
title:  "Using Embedded Jetty With Guice Servlet"
date:   2012-08-20 06:00:00
categories: jetty guice servlet
---

Recently, we decided to convert some of our web applications to standalone 
applications. The intention was to use embedded [Jetty][jetty] to serve the 
application and [guice-servlet][guice-servlet] for our servlet/filter 
configuration. One caveat was that I needed to include the code to do this in a 
common library so that it could be reused for multiple applications. In addition 
I needed a fully tested integration. The resulting library can be found 
[here][modularwebapp]. We decided to go with Jetty 7 since 8 only adds Servlet 
3.0 support, and we have no real need for this.

 [jetty]: http://www.eclipse.org/jetty/
 [guice-servlet]: http://code.google.com/p/google-guice/wiki/Servlets
 [modularwebapp]: https://bitbucket.org/peachjean/modularwebapp
 [rest-assured]: http://code.google.com/p/rest-assured/
 [class-ModularWebapp]: https://bitbucket.org/peachjean/modularwebapp/src/1.0.0/src/main/java/net/peachjean/modularwebapp/ModularWebapp.java
 [class-ModularWebappLauncher]: https://bitbucket.org/peachjean/modularwebapp/src/1.0.0/src/main/java/net/peachjean/modularwebapp/ModularWebappLauncher.java
 [class-ModularWebappExtension]: https://bitbucket.org/peachjean/modularwebapp/src/1.0.0/src/main/java/net/peachjean/modularwebapp/ModularWebappExtension.java
 [class-ServiceLoader]: http://docs.oracle.com/javase/6/docs/api/java/util/ServiceLoader.html
 [class-ModularWebappLauncherTest]: https://bitbucket.org/peachjean/modularwebapp/src/1.0.0/src/test/java/net/peachjean/modularwebapp/ModularWebappLauncherTest.java
 [class-ModularWebappTest]: https://bitbucket.org/peachjean/modularwebapp/src/1.0.0/src/test/java/net/peachjean/modularwebapp/ModularWebappTest.java

# The Approach

Starting with a simple list of steps, we can build a main method that launches 
and configures Jetty based on the current classpath.</div>

1.  Instantiate a Jetty Server.
2.  Setup a servlet context.
3.  Instantiate Guice Modules.
4.  Instantiate an Injector.
5.  Configure the servlet context with GuiceFilter.
6.  Add singleton ServletContextListeners to the servlet context.
7.  Start the Jetty Server.

Fortunately all of these, save step 3, is the same no matter what application 
we're launching. So we have a starting point. There are two main classes: 
[ModularWebapp][class-ModularWebapp] and 
[ModularWebappLauncher][class-ModularWebappLauncher]. The first, 
``ModularWebapp``, represents our jetty server with it's configuration. It is 
the meat of the implementation. ``ModularWebappLauncher`` is simply a main 
method that utilizes ``commons-cli`` to parse the command-line options, creates 
a ``ModularWebapp``, and starts it. In addition, we have a 
[ModularWebappExtension][class-ModularWebappExtension] interface that is used 
to build up the Guice injector via the [ServiceLoader][class-ServiceLoader] 
mechanism.

### [ModularWebappExtension][class-ModularWebappExtension]

```java
public interface ModularWebappExtension {
  Iterable<Module> loadModules(ServletContext servletContext);
}
```

This simple interface is how we will integrate our Guice modules with the 
ModularWebapp. Each module or set of modules needs only implement this interface 
and define its implementation in a 
``META-INF/services/net.peachjean.modularwebapp.ModularWebappExtension`` file.

### [ModularWebapp][class-ModularWebapp]

The usage of this class is simple. All configuration is supplied upon 
instantiation, with each parameter described explicitly in the javadoc. One 
thing to note for clarification is the resourceRoot. This is a folder from which 
the webapp will serve any existing static content. Once the ModularWebapp is 
instantiated, it should be launched. The client has two options - launch the 
server and let it run in the background or launch the server and wait for it to 
shutdown. The first is useful for embedding the server in an application that is 
also doing other things, or for running tests. The second exists to be called 
from a main method - when we want to make sure that the program blocks as long 
as the server is running. This class also contains a stop method - to shutdown 
the server.

The meat of the work is done in the launch method. All 7 steps as listed above 
occur here. To walk through them:

## Instantiate a Jetty Server

```java
  server = new Server();
  Connector connector = new SelectChannelConnector();
  connector.setPort(this.port);
  server.setConnectors(new Connector[]{connector});
```

This is the straightforward, simple way to create a jetty server. We create a 
connector, set the configured port on it, and set it on the instantiated server object.

## Setup a servlet context

```java
  ContextHandlerCollection contexts = new ContextHandlerCollection();
  ServletContextHandler root = new ServletContextHandler(contexts, "/", ServletContextHandler.SESSIONS);
  root.setResourceBase(this.resourceRoot.getAbsolutePath());
  root.addServlet(DefaultServlet.class.getCanonicalName(), "/");
```

There are two key things happening here though: we are setting the resource 
base, and we are adding the default servlet. The default servlet is important, 
because Jetty will not serve a url that does not have a servlet or a resource 
mapped to it. This means that since the Guice mapping is done with a Filter, 
without the DefaultServlet, every url will return a 404 without ever hitting the 
GuiceFilter.

## Instantiate Guice Modules

```java
  Iterable<Module> modules = instantiateModules(root.getServletContext());

...

  protected Iterable<Module> instantiateModules(final ServletContext servletContext) {
    Iterable<ModularWebappExtension> extendsServices = locateExtensions();
    final Iterable<Module> modules = Iterables.concat(Iterables.transform(extendsServices, new ModularExtensionIterableFunction(servletContext)));
    if (logger.isDebugEnabled()) {
      logger.debug("Loading guice {} modules...", Iterables.size(modules));
      for (Module module : modules) {
        logger.debug("Loading guice module {}.", module);
      }
    }
    return modules;
  }

  protected Iterable<ModularWebappExtension> locateExtensions() {
    return ServiceLoader.load(ModularWebappExtension.class);
  }
```

This is where we start adding some complexity. Modules are loaded from the 
``ModularWebappExtension`` implementations, which in turn are loaded via the 
[ServiceLoader][class-ServiceLoader]. This allows us to define a web application 
by simply placing jars containing ``ModularWebappExtension`` implementations on 
the classpath.

## Instantiate an Injector

```java
  Injector injector = Guice.createInjector(this.production ? Stage.PRODUCTION : Stage.DEVELOPMENT, modules);
```

The standard mechanism for injector creation, with the one detail that we're 
taking the "production" configuration into account.

## Configure the Servlet Context with GuiceFilter

```java
  root.addFilter(new FilterHolder(createGuiceFilter(injector)), "/*", EnumSet.allOf(DispatcherType.class));
```

This should be pretty self explanatory, but it is the key to the whole thing. 
This is what tells our embedded Jetty server about our Guice servlets 
configuration.

## Add singleton ServletContextListeners to the Servlet Context

```java
  for (Key<?> key : injector.getBindings().keySet()) {
    if (ServletContextListener.class.isAssignableFrom(key.getTypeLiteral().getRawType())) {
      root.addEventListener(injector.getInstance((Key<ServletContextListener>) key));
    }
  }
```

Since we aren't using a web.xml, we aren't already telling Jetty about our 
listeners. By default, Guice servlets doesn't do anything special with 
ServletContextListeners, but it's an extension that I've always added to my 
Guice bootstrap.

## Start the Server

```java
  server.setHandler(contexts);
  server.start();
  if (serverOutput.isPresent()) {
    serverOutput.get().println(server.dump());
  }
  if (waitForStop) {
    server.join();
  }
```

Here we finalize the setup and start. We then dump the server structure to 
output if present and, depending on how the launch method was called, wait for 
the server to shutdown.


## [ModularWebappLauncher][class-ModularWebappLauncher]

As mentioned earlier, this class pretty much just parses command line parameters 
and then instantiates and starts a ``ModularWebapp``.

# Testing

As with any integration, testing is both vitally important, and not particularly 
straightforward. Basic unit tests would be easy to write, but they don't tell us 
much. What we're most interested in is knowing that we've connected Guice, 
Jetty, and Guice Servlets together appropriately. And the only way to REALLY 
know this is to run a test that includes all three of these pieces. So, this 
integration has two test classes. The first 
([ModularWebappLauncherTest][class-ModularWebappLauncherTest]) is actually a 
fairly standard unit test, and ensures that the command line parsing happens 
appropriately and the ``ModularWebapp`` gets the correct values. The second 
([ModularWebappTest][class-ModularWebappTest]) is really an integration test. 
It launches an actual server several times, with different configurations, 
connects to it over a socket, and ensures that the supplied content is as 
expected. For asserting against this content, I've made use of the 
[rest-assured]rest-assured] project, which I've found to be invaluable for 
integration testing against RESTful (or any http) services. Its use here is 
rather trivial, but the power of its DSL for http assertions really shines when 
working with json or xml content.

# Using the Library

Using this library is easy. Simply include it in your maven dependencies (it is 
available on maven central), implement ``ModularWebappExtension``, define the 
service, and launch with the ``ModularWebappLauncher`` main class.


```
  <dependency>
    <groupId>net.peachjean.modularwebapp</groupId>
    <artifactId>modularwebapp</artifactId>
    <version>1.0.0</version>
  </dependency>
```

