---
layout: post
title: Dotnet Performance Counters
date: "2015-06-09 20:36"
tags: [dotnet, clr, performance-analysis, performance-reporting]
permalink: /2015/06/09/dotnet-performance-counters/
gh-repo: sarkershantonu/sarkershantonu.github.io
excerpt: "Blog on performance-analysis"
gh-badge: [star,follow]
comments: true
---
In this article we are going to know about different performance counters in dotnet. 

# A: Memory Counters
- As we monitor dotnet run time environment (CLR) during performance test, we need to know what to look for if we are measuring performance affected by Memory allocation , deallocation. 
- This is a very important if we are measuring dotnet performance impact for a particular application. 

Before starting this, i would request readers to go through basic dotnet CLR introduction. [This post might help in very basic level](https://sarkershantonu.github.io/2015/04/27/dotnet-clr-how-it-works/)
 
### Memory Counter : Allocation/Size
- **Large Object Heap size (byte)** : Size of large object(grater 85kb) heap which are spatially handled by GC directly(not using generational model). 
- **Gen 0 heap size (byte)**: Allocatable size of Generation 0(youngest) objects. 
- **Gen 1 heap size (byte)**: Allocatable size of Generation 1. 
- **Gen 2 heap size (byte)**: Allocatable size for Generation 2(oldest) objects. 
- **Number Bytes in all Heaps** : Total of (Gen 1 Heap Size + Gen 2 Heap Size + Large Object Heap Size).
    - This excludes generation 0 heap size as it will be collected after soonest GC collection.
    - This shows current memory allocated in GC Heaps. 

Note : All are updated after end of a GC collection (not allocation). All sizes are important to analyze memory allocation amount while profiling. So, these are important counters for memory leak analysis. 

- **Number of Total committed Bytes** : Size of Virtual Memory committed by GC. In windows, initially it is inside RAM, committed, which has reserved allocated memory in page file.
- **Number  Total reserved Bytes** : Size of Virtual Memory reserved by GC. In windows, reserved virtual memory is allocated with committed memory in page files(to disk) to support when not enough RAM space.

Committed and reserved indicate, **how much resource hungry** the application is. So, important counters for measuring **server/run time environment capacity**. 
### Memory Counter : Promotion(Generation Upgrade)
- **Finalization Survivors** : Number of survived objects after a GC event due to wait to be finalized.
    - Reference holding objects are not counted here 
    - It is not cumulative
    - Updated after each GC event
    - Shows count of the survivors during that particular collection only. 

This is very important performance counter for the application where developers use explicit finalization of objects. This indicates **extra overheads due to finalization**. 
- **Promoted Finalization-Memory from Gen 0** : Number of serviced object from generation 0 to generation 1 only due to wait to be finalized.. (partial part of Finalization Survivors )
- **Promoted Memory from Gen 0** (To Gen 1): Number of survived objects from Gen 0 to 1 after GC.So, this updates after each GC. 
- **Promoted Memory from Gen 1** : (To Gen 2): Number of survived objects from Gen 1 to 2 after GC.So, this updates after each GC.
- **Gen 0 Promoted Bytes/Sec** : It shows the rate (per second)of promotion from Gen 0 to 1.
- **Gen 1 Promoted Bytes/Sec** : It shows the rate (per second)of promotion from Gen 1 to 2.
    - This is oldest generation Nothing is promoted from generation 2 because it is the oldest generation. 
    - Indicator of very long-lived objects promotion rat

This **both promotion rates are major counters for memory leak and performance bottlenecks**.  

Note : Both rates measured between two collected samples. 
### Memory Counter : De-allocation/Collection
- **Number of Gen 0(youngest object) Collections** : This is the number of collected Generation 0 objects by GC since application started. So, it is updated after each gen 0 GC collection. This counter displays the last observed GC collection. 
- **Number Gen 1 Collections** : This is the number of collected Generation 1 objects by GC since application started. So, it is updated after each gen 1 GC collection
- **Number Gen 2 Collections** :(Full GC that triggers all) : This is the number of collected Generation 2 objects by GC since application started. So, it is updated after each gen 2 GC collection.

**Note** : Higher generation GC include all lower generations so each GC generation even counter is also explicitly incremented by a higher generation (0 by1 or 2, 1 by 2) GC event. 
### Memory Counter : GC Processing
- **Number if Induced GC** : It shows the highest number of **Explicit(from code) GC call**. 

Most cases this is **bad practice to control GC events** as GC collect can not only clear memory, but also keeps referral objects to higher generation. So, this should one of default performance monitoring counters.

Best practice is **No Call GC** from code, lets CLR decide when to collect, spatially for managed code. But, sometimes , due critical resource dependencies, it is okay to call GC to cleanup non reference tails and objects. 
- **Number of GC Handles** : It shows how many GC handlers are currently working which are separate handlers from CLR and managed code.  
- **Number of Pinned Objects** : When a GC can't move object in memory, it is pinned. This shows the number of pinned objects after latest GC event inside heaps . 

And, this shows pinned objects generation wise, Gen 1 pinned will be listed in Gen1 only, and so as for other generations. This is **useful counter** when the application runs under **low resources or at memory leak** condition where memory overflown. (OOM)
 
- **Number of Sink Blocks in use** : Counters of used synchronization blocks. Synchronization blocks are used to store synchronous information in per object context(  per-object data structures).
    - Holds weak references to managed objects 
    - Scanned by GC.

Note : They can also store non sync info like COM interop metadata. This is a **major performance counters** indicating dotnet inner resource availability. 
- **Percent (%) Time in GC** : It shows a GC collection time out of last GC collection. 

Example , let say 5 second time interval from last GC and current GC and current GC took 2sec, so this will show 40% GC time. So, it is updated after each GC call. This is a **very important counter** as it shoes **how much % work done by GC**

- **Allocated Bytes/sec**: It shows the rate of allocation, so it is updated after each GC collection and measured among two sample intervals. This is an **important performance counter where GC performance is in question**. To measure, how fast the allocation happens, this helps. 

# B:CLR Thread Counters
When we monitor dotnet run time environment (CLR) during performance test, we need to know what to look for if we are measuring performance affected by Thread processing, locking and unlocking thread.
- **Number of current physical Threads** : Shows native OS threads created and owned ny CLR.

Notes:
- This does not include the threads used by the runtime in its internal operations
- It is a subset of the threads in the operating system process.

- **Number of current logical Threads** : Shows number of thread objects(local threads) managed by CLR including running, halted/stopped thread in last collected sample

Both thread numbers **indicates performance overhead**. If numbers are too high, application might be in risk to manage those(GC overhead and  long queue lengths/wait time) . So these are **important** performance encounters. 
- **Current Queue Length** : Number of threads waiting to acquire a managed lock in last collated sample/observed.
- **Queue Length / sec** : Rate of waiting threads to acquire a managed lock between last two collected samples.
- **Queue Length Peak** : Number of threads waiting to acquire a managed lock since the application started. Queue lengths show wait time of threads. This indicates **how activities are waiting** for resources or events. So, these are very important counters for performance.
- **Rate of recognized threads / sec** : Rate of runtime recognized threads between last two collected samples.
    - Runtime does not create these unique(same ID) threads but must run at least once.
    - These are associated with a corresponding managed thread object and have the
- **Number of current recognized threads** : Currently recognized uniquely identified threads in runtime.
- **Total recognized Threads** : Number of total recognized uniquely identified threads.
- **Contention Rate / Sec** : Rate of failed attempts by threads on acquiring a managed lock in runtime.
- **Total Number of Contentions** : Numbers of failed attempts by threads on acquiring a managed lock in runtime. It shows total fail times, it is also an important performance counter.

# C: CLR Interop Counters
This indicates application interaction/usages with COM components, COM+ services, and external typed libraries. 
- **Numbers of  of CCWs** : Shows current number of COM callable wrappers (CCWs). That means, it actually shows number of managed objects for un-managed COM code. CCW is a proxy for a referenced managed object by an un-managed COM client.
- **Number of Stubs** : Current number of stubs created by CLR. Marshaling arguments and return values from managed to un-managed code is done by Stubs during a COM interop call or a platform invoke call.
- **Number of marshaling** : Total number of marshaled events since application started. Arguments & returned values marshaling between managed and un-managed code are in this count. This counter is not incremented if the stubs are inlined(they are usually inlined when small marshaling overhead )

Note : This counter usually **ignored for managed codebase**. But if you are using **legacy codebase** or libraries, this needed to monitor. The key thing to monitor is CCWs and if data transaction is large with un-managed code, then Stubs. 

# D: Exception Counters
As we monitor dotnet run time environment (CLR) during performance test, we need to know what to look for if we are measuring performance affected by Exception. 

### What is an exception?
When a block of code (or statement) runs under try/catch block with expected not default path (exceptions), we called exception handling. 

And, if there are too much exception for a single statement, that become overhead. Because, exception in run time environment work in separate thread which is spatially created to manage exceptions. A sample exception handling block code. 

```
try{
//code block
}catch (<Exception type> anObject){
//Do something  
}
}
finally {
//do some thing which should be done all the time
}
```

And, fundamentally, each type of exception, when it is caught, is a independent object of specific/generic exception class. So, if a piece of code leads to series of exceptions, it might hamper run time environment.

In many projects, developers decide to implement business logic exceptions via try catch which is okay for small project scope, but for complex and multilayer business validation, it becomes overhead ]
- **Exception Trowing Rate(/sec)** : How many exceptions per second are thrown? : It shows this rate. Usually this rate represents exception rate between two sample of data collection not average. So, it is a vital one. 

According to Microsoft forums, a rate more than **100 exceptions/s** is consider as poor performance. In practical , it depends on application.

- **Number of Thrown Exceptions** : How many exception are thrown? : This shows the number of exceptions since the application started. If any exception is re-thrown , counted again.
- **Filtered Exception Rate(/sec)** : It shows how many particular exception per second are thrown? It is same as exception throwing rate just with added filter. 
- **Finally rate(/sec)** : It shows how many finally block executed(which caught an exception) per second. It does not shows finally blocks which are not throws exception. 
- **Throw to Catch Depth / Sec** : This shows the level(depth) of exception, from level that throws up to where it is caught in rate(per second). That means, this shows the exception depth (nested exception) rate among two samples. 

### Notes : 
- This both counters includes handled and un-handled exceptions.
- Usually there will be two kinds of exceptions 
    - Dot NET exceptions: Thrown from dotnet managed code. 
    - Converted Un- managed exceptions: Thrown from converted un managed code(C++ libs, Com, or CLR core) that are converted into .NET exceptions.   
- All exception rates represents data rate between two sample of collection not average of all collected data. 
- Exception/sec is the most useful and usually used. 

# E:JIT Counters  
When we monitor dotnet run time environment (CLR) during performance test, we need to know what to look for if we are measuring performance affected by JIT compilation and how JIT is performing.  

This is another important counter for performance monitoring for individual code block performance. 
- **Number of bytes of MSIL processed by JIT(IL Bytes JITted)** : Shows amount of IL/MSIL code compiled by JIT  compiler since the application started(in byte). 
- **Total IL bytes processed by JIT** : Same as previous(IL Bytes JITted) but with all in count. So, it will same in one collected sample.
- **Rate of IL processing by JIT (IL Bytes JITted / sec)** : Shows rate of IL code compilation by JIT.
- **Note** : This rate represents JIT compilation rate between two collected samples not average of all. This is an important measurement for performance.,spatially for application running server capacity.
- **Number of Methods processed by JIT(Methods JITted)** : Shows total number of Methods JIT compiled since the application started. It excludes, per-compiled methods.
- **Standard JIT Failures** : It shows JIT compilation failures peak number of methods since the application started. MSIL verification, JIT internal error, Resource or Dependency Errors etc will also be included in count. This is another important counter for performance monitoring. 
- **Time % for a code in a JIT compilation phase:(% Time in JIT)** : JIT copulation occurs in phases. A phase occurs when a method and its dependencies are compiled. This time shows % of total time needed for compilation in one phase JIT. 

That means, for example, this shows how much percentage need for JIT to compile a piece of code in respect to total method compilation. This counter is updated at the end of every JIT compilation phase.(neutralize the counter for each phase) 

# F:CLR Class loaders' Counters
If we are measuring performance affected by Class loaders, assemblies, app domains inside CLR, we need this. For basic idea of CLR, [please see this post](https://sarkershantonu.github.io/2015/04/27/dotnet-clr-how-it-works/)

### Related to Class loader : 
- **Bytes in Loader Heap** : Class loader committed memory (in RAM)across all app domain.
- **Current Classes Loaded** : Current number of classes loaded in all assemblies. 
- **Rate of Classes Loaded (/sec)** : Rate of classes loaded in all assemblies in the last two collected samples.
- **Rate of Load Failures(/sec)** : Rate of classes failed to load in all assemblies in the last two collected samples. 
- **Total Number of Load Failures** : Count of failed to load class since the application started. For performance measurement, **failure counts(total and rate)** are very important to identify the events. This should be trigger for **big errors**. So, need to carefully monitor the failure heuristics and related failures. 
- **Total Classes Loaded** : Cumulative Count of loaded classes in all assemblies since the application started.

### Related to Assembly: 
- **Current Assemblies** : Current number of loaded assemblies in all app domains for a running application.
- **Rate of Assemblies(/sec)** : Rate of assemblies loading across all app domain in the last two collected samples.
- **Total Assemblies** : Counts total number of loaded assemblies since the application started.  

Note : If an assembly is loaded as domain-neutral from other app domains, it counts **only once** for all type of assembly counters. 

### Related to App Domain :
- **Current app domains** : Current app domain numbers loaded in the app.  
- **Rate of app domains (/sec)**: Rate of app domain loading in the last two collected samples.
- **Rate of app domains unloaded (/sec)**: Rate of app domain unloading in the last two collected samples.
- **Total App domains** : Count of all loaded app domains since app started. 
- **Total app domains unloaded** : Count of all unloaded app domains since app started. For multiple loading same app domains, it is counted multiple times 

For performance monitoring, all **loading and unloading rate** are more important as they **indicate performance of CLR**. 

# G: CLR Security Counters
CLR has different security checks before execute an application.  These counters help us for measuring performance overhead for CLR security checks for an application.
- **% Time in RT checks** : Time % spend on access checking since last collected sample. Update after each checking. This is very important performance counter due to nature. It might be **overhead for performance**.
- **Stack Walk Depth** : Depth (in number) at last runtime code access security check by walking the stack. This indicates complexity. 
- **Total Runtime Checks** : Total number of performed runtime code access security checks since the application started. Runtime checks are done when a caller ask for a permission at current thread stack of caller. With **high Stack Walk Depth** value,  this counter indicates the **performance penalty** due to **security checks**. So, this two together becomes important performance counter. 
- **Number of Link Time Checks**: Total number of Link Time code access security checks since the app started.Link time checks due to caller demands for permission at JIT compile time. So, it is once per caller. Due to this behavior, it is **ignored while performance measurement but measure only for security purposes**. 

# H: CLR Network Counters(Asp)
A web application sending and receiving data will be observed in CLR boundary. 

### CPU/Thread Related Counters :
Very important to track performance
- **HttpWebRequests Created/sec** : Rate of creating HttpWebRequest objects per second inside AppDomain.
- **HttpWebRequests Queued/sec** : Rate of adding HttpWebRequest object to the queue per sec inside AppDomain.
- **HttpWebRequests Aborted/sec**: Rate of Abort Method calling per second in HttpWebRequest objects inside AppDomain.
- **HttpWebRequests Failed/sec**: Rate of failed HttpWebRequest objects per second inside AppDomain. This is measured by observing http status code failure (not 200-299) from server.

### Time Counters: 
Very important counters
- **HttpWebRequest Average Lifetime**: Last interval (sample) ended average completion time for all HttpWebRequest objects inside AppDomain since the process started.
    - Calculation for a single time: Time start when object created and ends when the object responses (to response stream).
    - If  GetResponse or BeginGetResponse methods are not called by app then the lifetime of the HttpWebRequest object is ignored.
    - If exception occurred(while calling GetResponse or EndGetResponse methods) the lifetime ends.
- **HttpWebRequest Average Queue Time** : Average Time-On-Queue for all HttpWebRequest objects that left the queue inside AppDomain since the process started in the last interval(sample).

### Data Counters : 
This is important when memory issues are present. 
- **Bytes Received** : Total (cumulative) bytes received by all socket objects inside an App Domain since the process started.
- **Bytes Sent** : Total(cumulative) bytes sent by all socket objects within an AppDomain since the process started.

Note : These bytes are not defined by TCP/IP as it is inside CLR.
### Connection Object counters : 
- **Connections Established**: Total(cumulative) number of Socket objects connected inside AppDomain for stream sockets since the process started.
- **Datagrams Received** : Total(cumulative) datagram packets received by all Socket objects inside AppDomain since the process started.
- **Datagrams Sent** : Total(cumulative) datagram packets sent by all Socket objects inside AppDomain since the process started. 

These connection counters are measured by monitoring Socket class calls.

Thanks... :)