# VarnishPreProcessing
A Concept of preprocessing using varnish

## Introduction

Short time before writing this article, i have seen the awesome talk from Fabian Schrank about Lizards&Pumpkins. It has inspired me, to think about the topic of "cache vs pregeneration". Here are my toughts:

## General Concepts:

### Caching Concept:

Pages are cached upon request. No request = no content. flushing cache might be dangerous 
in high traffic times. 

### Pregeneration Concept:  Lizards and Pumpkins

Pages are splitted up in blocks, which are pregenerated using some worker threads. Can be parallelized very simply, by adding more workers. 

## My Concept in combining both technologys:

1. Lets assume you have a typical configuration:  2 Servers and a Varnish-Server as "loadbalancer":
```
+-----------------+
|                 |
|  Webserver 1    <---+
|                 |   |
+-----------------+   |        +---------------------+
                      |        |                     |
                      +--------+    Varnish          |
                      |        |                     |
                      |        +---------------------+
+-----------------+   |
|                 |   |
|  Webserver 2    <---+
|                 |
+-----------------+
```
2. Introduce a new "Worker Application", which will use some sort of queue technology. Put it onto another Server*

*Note: The "Worker" Application could be on one of the Servers, but its easier to explain the idea if you assume its on its own server.
```
+-----------------+
|                 |
|  Webserver 1    <---+
|                 |   |
+-----------------+   |        +---------------------+
                      |        |                     |
                      +--------+    Varnish          |
                      |        |                     |
                      |        +---------------------+
+-----------------+   |
|                 |   |
|  Webserver 2    <---+
|                 |
+-----------------+


+-----------------+
|                 |
|  Worker Server  |
|                 |
+-----------------+
```

3. Reconfigure the Varnish to use only Webserver1.  
 Create an GET/POST/Cookie Parameter which will force Varnish into using Server 2. Keep this parameter secret.
 
```

    +-----------------+   Every Request goes here, except there is a special Parameter set
    |                 |
    |  Webserver 1    <---+
    |                 |   |
    +-----------------+   |        +---------------------+
                          |        |                     |
                          +--------+    Varnish          |
                          x        |                     |
                          x        +---------------------+
    +-----------------+   x
    |                 |   x
    |  Webserver 2    <xxxx Only using Parameter ( GET/POST/Cookie)
    |                 |     Parameter is secret.
    +-----------------+


    +-----------------+
    |                 |
    |  Worker Server  |
    |                 |
    +-----------------+

```

4. Create some Backend Module for Magento, which will inform the worker about the changes being made. I suppose its sufficient to just transfer the urls of the products/categorys which are affected by the cache-invalidate. The worker should put them into a queue. Disable the normal "purge"-functionality if there is any. 

```



                             +-----------------+
                             |                 |
                      +------+  Webserver 1    <
                      |      |                 |
                      |      +-----------------+            +---------------------+
                      |                                     |                     |
   Contact Worker     |                                     |    Varnish          |
   as soon Changes    |                                     |                     |
   being made in      |                                     +---------------------+
   Backend            |      +-----------------+
                      |      |                 |
                      +------+  Webserver 2    <
                      |      |                 |
                      |      +-----------------+
                      |
                      |
                      |      +-----------------+
                      |      |                 |
                      +------>  Worker Server  |
                             |                 |
                             +-----------------+


```


5. The worker will process the  entrys which are on the queue and contact the varnish. Add the "special" Parameter from Step 3, to force varnish requesting Server 2.

```
+-----------------+
|                 |
|  Webserver 1    |
|                 |
+-----------------+            +---------------------+
                               |                     |
                          +----+    Varnish          <-----------+
                          |    |                     |           |
                          |    +---------------------+           |
+-----------------+       |                                      |
|                 |       |                                      |
|  Webserver 2    <-------+                                      |
|                 |                                              |
+-----------------+   "Cache"-Miss -> Cache misses could still appear.                                           |
                                                                 |
                                                                 |
+-----------------+                                              |
|                 |                                              |
>  Worker Server  +----------------------------------------------+
|                 |
+-----------------+
```

6. ??? (maybe scale it up by adding more webservers and workers?)
7. Profit


## Benefits of this solution:

No "Varnish"-Flushes anymore -> You could just tell the worker to "regenerate" all pages.

Invalidation is easy -> Just contact the worker and tell him which things to "regenerate". 


## Downsides:

"Cache"-Miss -> Cache misses could still appear.

Inconsitencies are hard to avoid -> could still occur. You will have to regenerate everything from time to time.


## Credits

Elias Kotlyar, 2016