---
layout: post
title: "A better way to implement the Singleton (anti?) pattern for easier testing using AmbientSingleton"
date: 2012-01-10 18:03:00 +0000
tags: [ddd, ef, eventsourcing, extensibility, gadgets, mocking, moq, msbuild, mvc, nuget, patterns, programming, t4, technology, vsx, wcf, webapi]
---

##  [A better way to implement the Singleton (anti?) pattern for easier testing using AmbientSingleton](http://blogs.clariusconsulting.net/kzu/a-better-way-to-implement-the-singleton-anti-pattern-for-easier-testing-using-ambientsingleton/ "A better way to implement the Singleton \(anti?\) pattern for easier testing using AmbientSingleton")

January 10, 2012 6:03 pm

In .NET singletons are typically implemented with a static variable that you control access to:
    
    
    public class SystemClock : IClock
    {
        private SystemClock() { }
    
        static SystemClock()
        {
            Instance = new SystemClock();
        }
    
        public static IClock Instance { get; private set; }
    }

Fairly easy. Problem is, of course that there’s no way you can test a class that uses this singleton now. The easy thing that you might be tempted to do is to just make the setter internal and starting replacing the singleton clock with mocks of it. Problem is, as you may have guessed, that now the tests cannot be run in multiple threads simultaneously. You have made no provisions for shielding changes across threads, so if one test setups up the singleton with a mock with certain values, and another goes and changes it before the asserts get through, you’ll be seeing intermittent failures on random tests when run together, but no single failure when run individually.

Being such a common problem, the .NET framework provides a few ways to work around that, via a mechanism called [Thread Local Storage](https://bit.ly/yD4at4) (or TLS). Joe Albahari provides a great summary of the available options, so make sure you [read that link](https://bit.ly/yD4at4). In summary, they are:

  * Mark a static backing variable with [ThreadStatic] attribute:  

        
        public class SystemClock : IClock
        {
            [ThreadStatic]
            private static IClock instance = new SystemClock();
        
            private SystemClock() { }
        
            public static IClock Instance
            {
                get { return instance; }
                internal set { instance = value; }
            }
        }
        

  * Use ThreadLocal<T> instead for the private field:  

        
        public class SystemClock : IClock
        {
            private static ThreadLocal<IClock> instance = new ThreadLocal<T>(() => new SystemClock());
        
            private SystemClock() { }
        
            public static IClock Instance
            {
                get { return instance.Value; }
                internal set { instance.Value = value; }
            }
        }
        

  * Use Thread.SetData/GetData to hold the state:  

        
        public class SystemClock : IClock
        {
            private static readonly LocalDataStoreSlot slot = Thread.AllocateDataSlot();
        
            private SystemClock() { }
        
            public static IClock Instance
            {
                get
                {
                    var value = (IClock)Thread.GetData(slot);
                    // Lazy init first time for this thread.
                    if (value == null)
                    {
                        value = new SystemClock();
                        Thread.SetData(slot, value);
                    }
        
                    return value;
                }
                internal set { Thread.SetData(slot, value); }
            }
        }

These options have slightly different performance characteristics (in my tests, ThreadStatic is fastest and the other two about 5x slower than that). However, they all share a very serious limitation, that was highlighted by [Scott ages ago, back in 2003](https://bit.ly/yLqVIw). They simply don’t work if what you need is per-context state, such as per web request, per test, essentially, per Call Context. Some call this the Ambient Context pattern, but its essence in .NET is the Logical Call Context, thoroughly dissected by [Jeffrey Richter in a blog post](https://bit.ly/z5gMct). 

Given its misleading [MSDN summary](https://bit.ly/yM3fdO), and its location with the namespace System.Runtime.Remoting.Messaging, it’s not surprising that developers quickly dismiss it as a valid (and much better as we’ll see!) alternative. A logical call context flows as a thread-like state bag, but it’s not limited to a single thread. In fact, even the new threading Tasks library honors this capability, and the most prominent use of this in .NET is in [WCF E2E Tracing](https://bit.ly/wl5Veh), which uses the [CorrelationManager.ActivityId](https://bit.ly/xXghu1), which is persisted in the logical call context and flows even across AppDomains. 

You basically use [CallContext.LogicalGetData](https://bit.ly/xfvdTu) and [LogicalSetData](https://bit.ly/ygvdMd) instead of the Thread.SetData/GetData from the example above. It is, however, a bit cumbersome, and I just wish we had something like ThreadLocal<T> to leverage for this “smart singleton” (or “ambient singleton”) capability.

Enter the [NETFx AmbientSingleton](https://bit.ly/w5NcfX) class, which gives you just that ![Smile](https://web.archive.org/web/20210127041224im_/http://blogs.clariusconsulting.net/kzu/files/2012/01/wlEmoticon-smile.png):
    
    
    public class SystemClock : IClock
    {
        private static readonly AmbientSingleton<IClock> instance = new AmbientSingleton<IClock>(() => new SystemClock());
    
        private SystemClock() { }
    
        public static IClock Instance
        {
            get { return instance.Value; }
            internal set { instance.Value = value; }
        }
    }

It is safe to set the value of the singleton in multiple threads simultaneously, and the value you set will be preserved in the entire call chain, even if the test or code under test spawns new threads via the thread pool or the task pool (which ultimately also uses the thread pool by default).

There is also a static AmbientSingleton.Create factory method with a couple overloads, to cut on the generic parameters noise.

The entire source for the class is very straightforward, as you’d expect: 
    
    
    static partial class AmbientSingleton
    {
        public static AmbientSingleton<T> Create<T>(T defaultValue)
        {
            return new AmbientSingleton<T>(defaultValue);
        }
    
        public static AmbientSingleton<T> Create<T>(Func<T> defaultValueFactory)
        {
            return new AmbientSingleton<T>(defaultValueFactory);
        }
    }
    
    partial class AmbientSingleton<T>
    {
        private string slotName = Guid.NewGuid().ToString();
        private Lazy<T> defaultValue;
    
        /// </summary>
        public AmbientSingleton()
            : this(() => default(T))
        {
        }
    
        public AmbientSingleton(T defaultValue)
            : this(() => defaultValue)
        {
        }
    
        public AmbientSingleton(Func<T> defaultValueFactory)
        {
            Guard.NotNull(() => defaultValueFactory, defaultValueFactory);
    
            this.defaultValue = new Lazy<T>(defaultValueFactory);
        }
    
        public T Value
        {
            get
            {
                var contextValue = CallContext.LogicalGetData(this.slotName);
                if (contextValue != null)
                    return (T)contextValue;
    
                return this.defaultValue.Value;
            }
            set
            {
                CallContext.LogicalSetData(this.slotName, value);
            }
        }
    }

As you can see, the _Ambient Singleton_ behaves like a regular singleton, without the overhead of per-thread initialization, when it’s not needed. This is another drawback of the TLS approaches: if the object initialization is expensive, you’ll be paying for that once per thread. In the ambient singleton, however, only when a particular call context requires a different value for the singleton, the added behavior kicks in. In my tests this approach is the closest to the speed of [ThreadStatic], measuring only about 1.8x the ThreadStatic approach (compared with 5x for the ThreadLocal<T> and Thread data). All in all, a very compelling way of implementing singletons, when they are unavoidable (although you could say there’s always another way of doing it ![Winking smile](https://web.archive.org/web/20210127041224im_/http://blogs.clariusconsulting.net/kzu/files/2012/01/wlEmoticon-winkingsmile.png), but that’s another post).

You can install the [nuget package to get the source AmbientSingleton<T> class](https://bit.ly/w5NcfX), which is partial and defaults to internal, as you’d typically not expose it as a public type in your libraries, and is rather an internal implementation detail of your singletons. You can of course create another partial of it and make it public if you wish. It’s fully documented, and lives in the System namespace.

In order to keep your code coverage goals, you can also install the [xUnit tests](https://bit.ly/ym4hcr) which provide 100% coverage for this source.

/kzu
