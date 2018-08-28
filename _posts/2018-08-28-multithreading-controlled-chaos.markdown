---
layout: post
title:  "Multithreading is Controlled Chaos"
date:   2018-08-28 15:40:56
categories: multithreading
---

The demise of [Moore's Law](https://en.wikipedia.org/wiki/Moore%27s_law) over the last decade or so has created a great necessity for the art of multithreading. Multithreading allows for multiple processes to be run on the collective pool of computing power offered by one or more cores.

For an everyday analogy, imagine if you had the ability to make copies of yourself. One you could go to work, the other could clean the house, one could walk the dog, and one could mow the grass. If you coordinate the multiple copies of yourself properly, you can massively increase the productivity of your life. But without coordination, the results would probably not be so good. Perhaps two copies of yourself would get into a fight to use the stove at the same time, and maybe they'd accidentally light the kitchen on fire as a result.

It's *even worse* in programming, because, it's *very* easy to create a multithreaded program that produces abnormal results. This can result in some very nasty and subtle bugs that may find their way into the final version of your product. For organizations that simply cannot afford to make mistakes, like financial institutions, this is clearly unacceptable.

So let's see an example of how miscoordination between threads can produce disastrous results. On a side note, the rest of the article will assume familiarity with basic Java syntax. I'll try my best to explain the more esoteric parts, but it might also be helpful to see [outside resources](http://tutorials.jenkov.com/java-concurrency/creating-and-starting-threads.html).

So here is the main method of an application:

```java
package io.github.wilsontheory;

import java.util.concurrent.CountDownLatch;

public class App
{
    public static void main( String[] args )
    {
        StringBuilder mutableThing = new StringBuilder();

        CountDownLatch latch = new CountDownLatch(5);

        Thread t = new Thread(new ObjectAlterationThread(mutableThing, 1, latch));
        Thread t2 = new Thread(new ObjectAlterationThread(mutableThing, 2, latch));
        Thread t3 = new Thread(new ObjectAlterationThread(mutableThing, 3, latch));
        Thread t4 = new Thread(new ObjectAlterationThread(mutableThing, 4, latch));
        Thread t5 = new Thread(new ObjectAlterationThread(mutableThing, 5, latch));

        try {
            t.start();
            t2.start();
            t3.start();
            t4.start();
            t5.start();
            latch.await();
        } catch (InterruptedException e){
            e.printStackTrace();
        }

        System.out.println(mutableThing.toString());
    }
}

```
I create a StringBuilder, which is passed into five threads that are manually constructed. In Java, one way to create a Thread is to use it's regular constructor. That constructor requires an instance of a class the implements the Runnable interface. In this case, I create a Java class named *ObjectAlterationThread* to fulfill that purpose.

```java
package io.github.wilsontheory;

import java.util.concurrent.CountDownLatch;

public class ObjectAlterationThread implements Runnable {

    private StringBuilder mutableThing;
    private int seedNumber;
    private CountDownLatch latch;

    public ObjectAlterationThread(StringBuilder mutableMap, int seedNumber, CountDownLatch latch){
        this.mutableThing = mutableMap;
        this.seedNumber = seedNumber;
        this.latch = latch;
    }
    @Override
    public void run() {
        try {
            Thread.sleep(100);
            int x = (523423413 % 32);
            int y = (261114124 % 14);
            int z = (253025323 % 23);
        } catch (Exception e){
            e.printStackTrace();
        }
        mutableThing.append(seedNumber);
        latch.countDown();
    }
}

```

In the main program, after the five threads are constructed, the .start() method is run on each of them in the same order in which they were created. In this program, CountDownLatch object is basically freezing the main thread and waiting for five "okay, I'm finished" signals from the five threads that were started. Each thread will work independently, as they're *unable to communicate with one another*.

The constructor of each ObjectAlterationThread takes a reference to our mutable object (a StringBuilder in this case), an integer seed number to keep track of the sequential order in which these threads are run in the actual program, and a reference to the CountDownLatch which is used to wait for all the threads to finish before we reach the call to System.out.println() in the main program.

When .start() is called on each of the threads, the Runnable object (in this case, an instance of ObjectAlterationThread), has .run() called on it. To produce a little bit of computational load in each of the threads, I have a 100 millisecond sleep timer and a little bit of module arithmetic performed in the .run() method. Once that's done. the seed number of that Thread is appended into the StringBuilder named *mutableThing*.

Note that all five threads have references to the same StringBuilder object. Although each thread in Java has its own stack, they all share the same heap. StringBuilder, as an object, lives in the heap.

So if multithreading was straightforward, what would you expect as the output to this program?

12345, in theory. But in practice, running this program five times gave me the following outputs:

```
12435
2345
345
213
352
```
Not only are the numbers out of order, but some runs of the program do not even return five digits as an output. In those cases, it's clear that calls to mutableThing.append(seedNumber) are failing. The exact reason goes beyond of the scope of this post, but we can at least determine here that StringBuilder doesn't have a mechanism to operate properly when one instance of it is being used between multiple threads. In threading terminology, it's "not thread-safe".

Note that without the computational "noise" added in the .run() method of ObjectAlterationThread the results can still be incomplete and/or out of order, albeit less common in my test runs.

#### Static Methods in Multiple Threads

I started on this topic because of some recent work on a number of utility classes -- i.e. Java classes that simply exist to store variables and methods (usually static) to be utilized by other classes.

Let's modify the program to store the mutableThing as well as the .append() method in a utility class.

Here' the new main method, almost unchanged:

```java
package io.github.wilsontheory;

import java.util.concurrent.CountDownLatch;

public class App
{
    public static void main( String[] args )
    {
        CountDownLatch latch = new CountDownLatch(5);

        Thread t = new Thread(new ObjectAlterationThread(1, latch));
        Thread t2 = new Thread(new ObjectAlterationThread(2, latch));
        Thread t3 = new Thread(new ObjectAlterationThread(3, latch));
        Thread t4 = new Thread(new ObjectAlterationThread(4, latch));
        Thread t5 = new Thread(new ObjectAlterationThread(5, latch));
        try {
            t.start();
            t2.start();
            t3.start();
            t4.start();
            t5.start();
            latch.await();
        } catch (InterruptedException e){
            e.printStackTrace();
        }

        System.out.println(UtilityClass.mutableThing.toString());
    }
}

```

Here is the UtilityClass, short and sweet:

```java
package io.github.wilsontheory;

public class UtilityClass {
    public static StringBuilder mutableThing = new StringBuilder();

    static void append(int i){
        mutableThing.append(i);
    }
}

```

And here's the new ObjectAlterationThread, which now simply appends its seed number to the instance of StringBuilder named mutableThing through the UtilityClass method .append();

```java
package io.github.wilsontheory;

import java.util.concurrent.CountDownLatch;

public class ObjectAlterationThread implements Runnable {

    private int seedNumber;
    private CountDownLatch latch;

    public ObjectAlterationThread(int seedNumber, CountDownLatch latch){
        this.seedNumber = seedNumber;
        this.latch = latch;
    }
    @Override
    public void run() {
        try {
            Thread.sleep(100);
            int x = (523423413 % 32);
            int y = (261114124 % 14);
            int z = (253025323 % 23);
        } catch (Exception e){
            e.printStackTrace();
        }
        UtilityClass.append(seedNumber);
        latch.countDown();
    }
}

```

I run the program five more times and get the following results:

```
21453
12345
15432
245
4135
```

Still no good, and calls to the .append() method are still being dropped. But now that I think of it, there is a special keyword in Java that might help us here. The *synchronized* keyword, which can be attached to our static .append() method.

What this synchronized keyword will do is allow the thread calling .append() to obtain the mutex of UtilityClass. Only one thread can own the mutex at a time, and the threads waiting to call .append() will be blocked until they get it.

```java
package io.github.wilsontheory;

public class UtilityClass {
    public static StringBuilder mutableThing = new StringBuilder();

    static synchronized void append(int i){
        mutableThing.append(i);
    }
}
```

Because of this blocking cascade, it is assured that each thread gets the UtilityClass once, and that they do not call the .append() method at the same time. Because of this, the output now has the correct number of digits when I run the program:

```
53214
13524
14235
15324
12345
```

But even if each thread now has some 1-on-1 time with the .append() method living inside UtilityClass, the order is not guaranteed here. None of these five runs had the order of "12345" that I originally wanted.

It's true that if I comment out the "noise" I added in the .run() method of ObjectAlterationThread that I have a much better chance of getting "12345" as the output.

```java
package io.github.wilsontheory;

import java.util.concurrent.CountDownLatch;

public class ObjectAlterationThread implements Runnable {

    private int seedNumber;
    private CountDownLatch latch;

    public ObjectAlterationThread(int seedNumber, CountDownLatch latch){
        this.seedNumber = seedNumber;
        this.latch = latch;
    }
    @Override
    public void run() {
        try {
//            Thread.sleep(100);
//            int x = (523423413 % 32);
//            int y = (261114124 % 14);
//            int z = (253025323 % 23);
        } catch (Exception e){
            e.printStackTrace();
        }
        UtilityClass.append(seedNumber);
        latch.countDown();
    }
}

```

And here's the yield from running the modified application five more times:

```
12345
12345
15432
12345
12345
```

Yeah, definitely better. But as you can see, there was a mistake made on the third run. A design flaw like this is simply unacceptable.

Unless you want to devise a complicated, programmatic mechanism to force thread 2 to wait for the completion of thread 1 before executing UtilityClass.append(), and for thread 3 to wait for thread 2 to execute UtilityClass.append() and so forth, there is no way for us to guarantee that the threads will finish in the order that we want them to and thus produce the correct output with certainty. So unless you can control that chaos, you'll either have to change the design of your algorithm or give up your multithreaded architecture.

#### What's the solution?

There are many ways to solve this, but in general you'll have to find a way to ensure the order of the seed integers based on the thread they are coming from. For example, you need to ensure that the first item in your output can only be seeded by the first thread, second by the second, and so forth.

So let's ensure that each index in the array corresponds to the order of the threads by altering out .append() method in UtilityClass to the following:

```java
static synchronized void append(int i){
//        mutableThing.append(i);
    mutableThing[i - 1] = i;
}
```

If you print out the array...

```java
//        System.out.println(UtilityClass.mutableThing.toString());
        System.out.println(Arrays.toString(UtilityClass.mutableThing));
```

...you get the output you originally wanted.

```
[1, 2, 3, 4, 5]
```

Well, not exactly. You still have to join the array together into a single String to get the exact same output, but you get the point. This data structure guarantees the order of the inputs from each of the threads, because the positions are predetermined and intrinsic to the data structure itself. A StringBuilder doesn't care who's appending to it, or from where. It just builds.

If you want to run the experiment from above on your own, tweak it, or improve it, please [feel free](https://github.com/wilsontheory/multithreadminidemo). If you found any errors or issues with this post, please [let me know](mailto:brian.l.wilson@protonmail.com). See you next time!
