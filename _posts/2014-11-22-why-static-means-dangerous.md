---
layout: post
category : java
tagline: "Why your own custom java.util.logging logger can be a bad thing..."
tags : [java, logging]
---
{% include JB/setup %}

## Simple Java platform without any weird buzzwords...

Recently I had a chance to work with pretty old legacy code. If you would like
to feel exactly the same thing like me imagine complex web application working
without dependency injection for services. Actually there was no services layer
at all ;-). Instead of this -- bunch of util **static** helper classes that
connects to database or 3rd party services so we shouldn't even speak any object
instances.

No matter how often you like to test something, that was a big challenge
to modify something in the core part with a few (unit) tests on the board. 
Awesome!

...but that wasn't the end. In order to test or change anything the quickest way
of doing that was to use hot redeployment (in this case for jetty).

## Where are my beatiful logs ? My favourite Sherlock Holmes job!

Hot redeploy is so fast (I'm skipping here such great solutions as JRebel)
because you don't need to restart JVM, restart web container (jetty in this
case).
The only one thing you need to do is to `touch` (Unix command) specified jetty
context XML file with webapp context descriptor. Jetty just reloads only one
webapp without restarting anything else (even 1-2 minutes are saved per major
redeploy in comparison to full cold restart).

...but here comes the problem...

After first quick reload part of my logs just have disappeared. The most awkward
thing was that framework logs were available as usually anytime I redeployed the
application.

Finally I found the suspicious place -- of course in the project codebase.
Take a look at the following code (Please note it's not the exact original
class):


    import java.util.logging.Logger;
    import java.util.logging.LogManager;
    
    public class ProjectLogger extends Logger {
        
        private static ProjectLogger instance;
        
        public static synchronized Logger getLogger(Class<?> class) {
            if (instance != null) {
                return instance;
            } else {
                String pkgName = ProjectLogger.class.getPackage().getName();
                instance = new ProjectLogger(pkg.getName());
                LogManager manager = LogManager.getLogManager();
                
                // The below line is crucial here. It activates this logger instance
                manager.addLogger(instance);
                return instance;
            }
        }
        
        private ProjectLogger(String pkgName) {
            // There is only one constructor from parent class we can use
            super(pkgName, null)
            
            // Custom logging initialization goes here
        }
        
        public void info(String message, Object[] args) {
            // Customized implementation
        }
        
        public void warning(String message, Object[] args) {
            // Customized implementation
        }
        
        
        public void isInfoEnabled() {
            // New method goes here
        }
        
        public void isWarningEnabled() {
            // New method goes here
        }
        
        // Other useful methods originally not implemented for original Logger class
        
    }
    
The crucial to understand for me was that:

1. **LogManager** class (see its [source](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/logging/LogManager.java) 
   code) is initialized at the very begining of JVM lifecycle. Please note the
   `static {...}` initializer which assigns reference to a static`manager` 
   variable.

2. The class if it's loaded **once** by class loader (i.e. by 
   `LogManager.getLogManager()` call) it will be always present -- the static 
   variable in itself will be always defined and by class contact this 
   reference cannot be changed unless you are subclassing it.

3. **LogManager** class [holds](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/logging/LogManager.java#158)
   tree of logger nodes as well as logger mappings.

4. `ProjectLogger` class from the project I was used to work on is removed 
   totally and created from scratch after each redeploy (I've checked that
   already by checking `instance` variable value) because Jetty 7 defines
   separate class loader for each webapp context.

*...and the most important one:*

**[addLogger()](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/util/logging/LogManager.java#424)**
adds only **weak reference** of class (to avoid GC/memory leaks)
and **returns boolean value** as a status of operation which isn't checked in 
the current class implementation.

## Conclusions so far

If it's hard for you to visualize all implications I will summarize what is
happening there between jetty hot redeploys:

1. The 1st Jetty/JVM start and 1st initialization of application creates 
   `ProjectLogger` classes and registers it's reference in Java built-in 
   `LogManager` class.

2. The 2nd and each next redeploy **destroys** `ProjectLogger` class completely
   (Dedicated classloader is destroyed). The new `ProjectLogger` class don't
   know anything about the previous class instance (the `instance` static 
   variable needs to be initialized but `LogManager` still holds weak reference
   to the old `ProjectLogger` class.

3. All classes that uses new `ProjectLogger` logger will be logging messages
   through the newly created class instance which somehow doesn't work.

# What's happening during next class initialization?

Recall `getLogger()` method:

    public static synchronized Logger getLogger(Class<?> class) {
        if (instance != null) {
            return instance;
        } else {
            // Note that logger package is always the same
            String pkgName = ProjectLogger.class.getPackage().getName();
            instance = new ProjectLogger(pkg.getName());
            LogManager manager = LogManager.getLogManager();
            
            // The below line is crucial here. It activates this logger instance
            manager.addLogger(instance);
            return instance;
        }
    }

I've found that `manager.addLogger(instance)` returns false after each next
redeploy as the previous `ProjectLogger` weak reference is defined. Wow! That
sounds tricky.

In this place I can only say: __huh I wonder how many places in Java works 
in exactly the same way__. Is it safe to reuse and reuse exactly the same
JVM instance without making any restarts? It's so real to have such similar
conditions in different places or even memory leaks. Is it safe to use and store
statically state ?

# How to fix ?

I was looking many places where I could destroy the old instance but it's not
possible -- there is no public method available for [LogManager](https://docs.oracle.com/javase/6/docs/api/java/util/logging/LogManager.html)
class which I could use to remove only that weak reference. The [reset()](https://docs.oracle.com/javase/6/docs/api/java/util/logging/LogManager.html#reset())
method is too drastic as my initial logging configuration could be lost.

Finally I've figured out trying different approaches by trial and error what's
working correctly:

    public static synchronized Logger getLogger(Class<?> class) {
        if (instance != null) {
            return instance;
        } else {
            String pkgName = ProjectLogger.class.getPackage().getName();
            instance = new ProjectLogger(pkg.getName());
            LogManager manager = LogManager.getLogManager();
            
            // --- NEW PART ---
            // We need to check the state of operation
            boolean isReallyAdded = manager.addLogger(instance);
            if (!isReallyAdded) {
                // The operation failed so there is previous weak reference there
                Logger existingLogger = manager.getLogger(pkgName);
                
                // Make sure for some way it's not a null reference somehow.
                if (existingLogger != null) {
                    newLogger.setParent(existingLogger);
                }
            }
            // --- END OF NEW PART ---
            
            return instance;
        }
    }

You would probably ask why we cannot use `existingLogger` reference directly
to assign to our  static `instance`?

The answer is simple `existingLogger` class is only a weak reference of type 
`Logger` to the original `ProjectLogger` object and even casting it to the 
`ProjectLogger` class will cause **ClassCastException**. The only one valid 
method here is to set child for the `existingLogger` logger so our current
class will be activated in the `LogManager` tree and log handlers will print
out finally your messages.


# What I have learned after that lesson?


I couldn't believe how such simple things like simple message logging job are
so complicated in practice but the most important moral for me is:


* Try to not use stateful static classes (use only static helper class without 
  state)

* Don't assume your JVM will be leaving forever (the probability of memory 
  leaks is rather higher than lower)

* Don't use hot redeployment on production in any way. Instead of this fully 
  initialize new JVM and exclude temporarily this particular server from the
  application cluster.

* Use normal non-singleton objects and avoid static variables to non primitive
  types. If you need to hold object references in that way think about this
  twice before you actually implement it.

* Do not extend classes if you don't know their ecosystem. In our case this
  was the `java.util.logging.Logger` class. Far more better solution is to use
  helper methods for helper classes. 


