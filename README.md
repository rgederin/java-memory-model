# Table of Contents

- [Stacks and Heap](#stacks-and-heap)
    * [Hardware Memory Architecture](#hardware-memory-architecture)
    * [Java memory model and hardware architecture](#java-memory-model-and-hardware-architecture)
    * [Visibility of Shared Objects](#visibility-of-shared-objects)
    * [Race Conditions](#race-conditions)
    * [Conclusions](#conclusions)
- [String pool](#string-pool)
    * [Strings Interning](#strings-interning)
    * [Strings Allocated using constructor](#strings-allocated-using-constructor)
    * [Manual Interning](#manual-interning)
    * [Intern in Java 6](#intern-in-java-6)
    * [Intern in Java 7](#intern-in-java-7)
    * [Garbage Collection](#garbage-collection)
    * [JVM string pool implementation](#jvm-string-pool-implementation)
    * [Java 9](#java-9)
    * [Summary](#summary)

# Stacks and Heap

The Java memory model used internally in the JVM divides memory between thread stacks and the heap. This diagram illustrates the Java memory model from a logic perspective:

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory1.png)

**Each thread running in the Java virtual machine has its own thread stack.** The **thread stack contains information about what methods the thread has called to reach the current point of execution**. I will refer to this as the "call stack". As the thread executes its code, the call stack changes.

**The thread stack also contains all local variables for each method being executed (all methods on the call stack)**. A thread can only access it's own thread stack. Local variables created by a thread are invisible to all other threads than the thread who created it. Even if two threads are executing the exact same code, the two threads will still create the local variables of that code in each their own thread stack. Thus, each thread has its own version of each local variable.

**All local variables of primitive types ( boolean, byte, short, char, int, long, float, double) are fully stored on the thread stack and are thus not visible to other threads.** One thread may pass a copy of a pritimive variable to another thread, but it cannot share the primitive local variable itself.

**The heap contains all objects created in your Java application, regardless of what thread created the object.** This includes the object versions of the primitive types (e.g. Byte, Integer, Long etc.). It does not matter if an object was created and assigned to a local variable, or created as a member variable of another object, the object is still stored on the heap.

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory2.png)   
    
**A local variable may be of a primitive type, in which case it is totally kept on the thread stack.**

**A local variable may also be a reference to an object.** In that case the reference (the local variable) is stored on the thread stack, but the object itself if stored on the heap.

An object may contain methods and these methods may contain local variables. These local variables are also stored on the thread stack, even if the object the method belongs to is stored on the heap.

**An object's member variables are stored on the heap along with the object itself.** That is true both when the member variable is of a primitive type, and if it is a reference to an object.

**Static class variables are also stored on the heap along with the class definition.**

Objects on the heap can be accessed by all threads that have a reference to the object. When a thread has access to an object, it can also get access to that object's member variables. If two threads call a method on the same object at the same time, they will both have access to the object's member variables, but each thread will have its own copy of the local variables.

Here is a diagram illustrating the points above:

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory3.png)

Two threads have a set of local variables. One of the local variables (Local Variable 2) point to a shared object on the heap (Object 3). The two threads each have a different reference to the same object. Their references are local variables and are thus stored in each thread's thread stack (on each). The two different references point to the same object on the heap, though.

Notice how the shared object (Object 3) has a reference to Object 2 and Object 4 as member variables (illustrated by the arrows from Object 3 to Object 2 and Object 4). Via these member variable references in Object 3 the two threads can access Object 2 and Object 4.

The diagram also shows a local variable which point to two different objects on the heap. In this case the references point to two different objects (Object 1 and Object 5), not the same object. In theory both threads could access both Object 1 and Object 5, if both threads had references to both objects. But in the diagram above each thread only has a reference to one of the two objects.

So, what kind of Java code could lead to the above memory graph? Well, code as simple as the code below:

```
public class MySharedObject {

    //static variable pointing to instance of MySharedObject

    public static final MySharedObject sharedInstance =
        new MySharedObject();


    //member variables pointing to two objects on the heap

    public Integer object2 = new Integer(22);
    public Integer object4 = new Integer(44);

    public long member1 = 12345;
    public long member1 = 67890;
}

public class MyRunnable implements Runnable() {

    public void run() {
        methodOne();
    }

    public void methodOne() {
        int localVariable1 = 45;

        MySharedObject localVariable2 =
            MySharedObject.sharedInstance;

        //... do more with local variables.

        methodTwo();
    }

    public void methodTwo() {
        Integer localVariable1 = new Integer(99);

        //... do more with local variable.
    }
}
```

If two threads were executing the run() method then the diagram shown earlier would be the outcome. The run() method calls methodOne() and methodOne() calls methodTwo().

methodOne() declares a primitive local variable (localVariable1 of type int) and an local variable which is an object reference (localVariable2).

Each thread executing methodOne() will create its own copy of localVariable1 and localVariable2 on their respective thread stacks. The localVariable1 variables will be completely separated from each other, only living on each thread's thread stack. One thread cannot see what changes another thread makes to its copy of localVariable1.

Each thread executing methodOne() will also create their own copy of localVariable2. However, the two different copies of localVariable2 both end up pointing to the same object on the heap. The code sets localVariable2 to point to an object referenced by a static variable. There is only one copy of a static variable and this copy is stored on the heap. Thus, both of the two copies of localVariable2 end up pointing to the same instance of MySharedObject which the static variable points to. The MySharedObject instance is also stored on the heap. It corresponds to Object 3 in the diagram above.

Notice how the MySharedObject class contains two member variables too. The member variables themselves are stored on the heap along with the object. The two member variables point to two other Integer objects. These Integer objects correspond to Object 2 and Object 4 in the diagram above.

Notice also how methodTwo() creates a local variable named localVariable1. This local variable is an object reference to an Integer object. The method sets the localVariable1 reference to point to a new Integer instance. The localVariable1 reference will be stored in one copy per thread executing methodTwo(). The two Integer objects instantiated will be stored on the heap, but since the method creates a new Integer object every time the method is executed, two threads executing this method will create separate Integer instances. The Integer objects created inside methodTwo() correspond to Object 1 and Object 5 in the diagram above.

Notice also the two member variables in the class MySharedObject of type long which is a primitive type. Since these variables are member variables, they are still stored on the heap along with the object. Only local variables are stored on the thread stack.

## Hardware Memory Architecture

Here is a simplified diagram of modern computer hardware architecture:

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory4.png) 

A modern computer often has 2 or more CPUs in it. Some of these CPUs may have multiple cores too. The point is, that on a modern computer with 2 or more CPUs it is possible to have more than one thread running simultaneously. Each CPU is capable of running one thread at any given time. That means that if your Java application is multithreaded, one thread per CPU may be running simultaneously (concurrently) inside your Java application.

Each CPU contains a set of registers which are essentially in-CPU memory. The CPU can perform operations much faster on these registers than it can perform on variables in main memory. That is because the CPU can access these registers much faster than it can access main memory.

Each CPU may also have a CPU cache memory layer. In fact, most modern CPUs have a cache memory layer of some size. The CPU can access its cache memory much faster than main memory, but typically not as fast as it can access its internal registers. So, the CPU cache memory is somewhere in between the speed of the internal registers and main memory. Some CPUs may have multiple cache layers (Level 1 and Level 2), but this is not so important to know to understand how the Java memory model interacts with memory. What matters is to know that CPUs can have a cache memory layer of some sort.

A computer also contains a main memory area (RAM). All CPUs can access the main memory. The main memory area is typically much bigger than the cache memories of the CPUs.

Typically, when a CPU needs to access main memory it will read part of main memory into its CPU cache. It may even read part of the cache into its internal registers and then perform operations on it. When the CPU needs to write the result back to main memory it will flush the value from its internal register to the cache memory, and at some point flush the value back to main memory.

The values stored in the cache memory is typically flushed back to main memory when the CPU needs to store something else in the cache memory. The CPU cache can have data written to part of its memory at a time, and flush part of its memory at a time. It does not have to read / write the full cache each time it is updated. Typically the cache is updated in smaller memory blocks called "cache lines". One or more cache lines may be read into the cache memory, and one or mor cache lines may be flushed back to main memory again.

## Java memory model and hardware architecture

As already mentioned, the Java memory model and the hardware memory architecture are different. The hardware memory architecture does not distinguish between thread stacks and heap. On the hardware, both the thread stack and the heap are located in main memory. Parts of the thread stacks and heap may sometimes be present in CPU caches and in internal CPU registers. This is illustrated in this diagram:

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory5.png) 

When objects and variables can be stored in various different memory areas in the computer, certain problems may occur. The two main problems are:

* Visibility of thread updates (writes) to shared variables.
* Race conditions when reading, checking and writing shared variables.

Both of these problems will be explained in the following sections.

## Visibility of Shared Objects

If two or more threads are sharing an object, without the proper use of either volatile declarations or synchronization, updates to the shared object made by one thread may not be visible to other threads.

Imagine that the shared object is initially stored in main memory. A thread running on CPU one then reads the shared object into its CPU cache. There it makes a change to the shared object. As long as the CPU cache has not been flushed back to main memory, the changed version of the shared object is not visible to threads running on other CPUs. This way each thread may end up with its own copy of the shared object, each copy sitting in a different CPU cache.

The following diagram illustrates the sketched situation. One thread running on the left CPU copies the shared object into its CPU cache, and changes its count variable to 2. This change is not visible to other threads running on the right CPU, because the update to count has not been flushed back to main memory yet.

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory6.png)

To solve this problem you can use Java's volatile keyword. The volatile keyword can make sure that a given variable is read directly from main memory, and always written back to main memory when updated.
 
## Race Conditions 

If two or more threads share an object, and more than one thread updates variables in that shared object, race conditions may occur.

Imagine if thread A reads the variable count of a shared object into its CPU cache. Imagine too, that thread B does the same, but into a different CPU cache. Now thread A adds one to count, and thread B does the same. Now var1 has been incremented two times, once in each CPU cache.

If these increments had been carried out sequentially, the variable count would be been incremented twice and had the original value + 2 written back to main memory.

However, the two increments have been carried out concurrently without proper synchronization. Regardless of which of thread A and B that writes its updated version of count back to main memory, the updated value will only be 1 higher than the original value, despite the two increments.

This diagram illustrates an occurrence of the problem with race conditions as described above:

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory7.png)

To solve this problem you can use a Java synchronized block. A synchronized block guarantees that only one thread can enter a given critical section of the code at any given time. Synchronized blocks also guarantee that all variables accessed inside the synchronized block will be read in from main memory, and when the thread exits the synchronized block, all updated variables will be flushed back to main memory again, regardless of whether the variable is declared volatile or not.

## Conclusions

* Heap memory is used by all the parts of the application whereas stack memory is used only by one thread of execution.
* Whenever an object is created, it’s always stored in the Heap space and stack memory contains the reference to it. Stack memory only contains local primitive variables and reference variables to objects in heap space.
* Objects stored in the heap are globally accessible whereas stack memory can’t be accessed by other threads.
* Memory management in stack is done in LIFO manner whereas it’s more complex in Heap memory because it’s used globally. Heap memory is divided into Young-Generation, Old-Generation etc, more details at Java Garbage Collection.
* Stack memory is short-lived whereas heap memory lives from the start till the end of application execution.
* We can use -Xms and -Xmx JVM option to define the startup size and maximum size of heap memory. We can use -Xss to define the stack memory size.
* When stack memory is full, Java runtime throws java.lang.StackOverFlowError whereas if heap memory is full, it throws java.lang.OutOfMemoryError: Java Heap Space error.
* Stack memory size is very less when compared to Heap memory. Because of simplicity in memory allocation (LIFO), stack memory is very fast when compared to heap memory.


# String pool

Java String Pool — the special memory region where Strings are stored by the JVM. String pooling (aka string canonicalisation) is a process of replacing several String objects with equal value but different identity with a single shared String object. 

You can achieve this goal by keeping your own Map<String, String> (with possibly soft or weak references depending on your requirements) and using map values as canonicalised values. Or you can use String.intern() method which is provided to you by JDK.

At times of Java 6 using String.intern() was forbidden by many standards due to a high possibility to get an OutOfMemoryException if pooling went out of control. Oracle Java 7 implementation of string pooling was changed considerably.

## Strings Interning

Thanks to the immutability of Strings in Java, the JVM can optimize the amount of memory allocated for them by storing only one copy of each literal String in the pool. This process is called interning.

When we create a String variable and assign a value to it, the JVM searches the pool for a String of equal value.

If found, the Java compiler will simply return a reference to its memory address, without allocating additional memory.

If not found, it’ll be added to the pool (interned) and its reference will be returned.

Let’s write a small test to verify this:

```

String constantString1 = "Baeldung";
String constantString2 = "Baeldung";
         
assertThat(constantString1)
  .isSameAs(constantString2);
  
```

## Strings Allocated using constructor

When we create a String via the new operator, the Java compiler will create a new object and store it in the heap space reserved for the JVM.

Every String created like this will point to a different memory region with its own address.

Let’s see how this is different from the previous case:

```

String constantString = "Baeldung";
String newString = new String("Baeldung");
 
assertThat(constantString).isNotSameAs(newString);

```

## Manual Interning

We can manually intern a String in the Java String Pool by calling the intern() method on the object we want to intern.

Manually interning the String will store its reference in the pool, and the JVM will return this reference when needed.

Let’s create a test case for this:

```

String constantString = "interned Baeldung";
String newString = new String("interned Baeldung");
 
assertThat(constantString).isNotSameAs(newString);
 
String internedString = newString.intern();
 
assertThat(constantString)
  .isSameAs(internedString);
  
```

## Intern in Java 6

In those good old days all interned strings were stored in the PermGen – the fixed size part of heap mainly used for storing loaded classes and string pool. Besides explicitly interned strings, PermGen string pool also contained all literal strings earlier used in your program (the important word here is used – if a class or method was never loaded/called, any constants defined in it will not be loaded).

The biggest issue with such string pool in Java 6 was its location – the PermGen. PermGen has a fixed size and can not be expanded at runtime. You can set it using -XX:MaxPermSize=N option. As far as I know, the default PermGen size varies between 32M and 96M depending on the platform. You can increase its size, but its size will still be fixed. Such limitation required very careful usage of String.intern – you’d better not intern any uncontrolled user input using this method. That’s why string pooling at times of Java 6 was mostly implemented in the manually managed maps.

## Intern in Java 7

Oracle engineers made an extremely important change to the string pooling logic in Java 7 – the string pool was relocated to the heap. It means that you are no longer limited by a separate fixed size memory area. All strings are now located in the heap, as most of other ordinary objects, which allows you to manage only the heap size while tuning your application. Technically, this alone could be a sufficient reason to reconsider using String.intern() in your Java 7 programs.

## Garbage Collection

Before Java 7, the JVM placed the Java String Pool in the PermGen space, which has a fixed size — it can’t be expanded at runtime and is not eligible for garbage collection.

The risk of interning Strings in the PermGen (instead of the Heap) is that we can get an OutOfMemory error from the JVM if we intern too many Strings.

From Java 7 onwards, the Java String Pool is stored in the Heap space, which is garbage collected by the JVM. The advantage of this approach is the reduced risk of OutOfMemory error because unreferenced Strings will be removed from the pool, thereby releasing memory.

## JVM string pool implementation

**The string pool is implemented as a fixed capacity hash map with each bucket containing a list of strings with the same hash code.** 

The default pool size is 1009 (before Java7u40). It was a constant in the early versions of Java 6 and became configurable between Java6u30 and Java6u41. It is configurable in Java 7 from the beginning (at least it is configurable in Java7u02). You need to specify **-XX:StringTableSize=N**, where N is the string pool map size. Ensure it is a prime number for the better performance.

This parameter will not help you a lot in Java 6, because you are still limited by a fixed size PermGen size. The further discussion will exclude Java 6.
                                                 
**Java7 (until Java7u40)**

In Java 7, on the other hand, you are limited only by a much higher heap size. It means that you can set the string pool size to a rather high value in advance (this value depends on your application requirements). As a rule, one starts worrying about the memory consumption when the memory data set size grows to at least several hundred megabytes. In this situation, allocating 8-16 MB for a string pool with one million entries seems to be a reasonable trade off (do not use 1,000,000 as a -XX:StringTableSize value – it is not prime; use 1,000,003 instead).
  
You may expect a uniform distribution of interned strings in the buckets.

You must set a higher -XX:StringTableSize value (compared to the default 1009) if you intend to actively use String.intern() – otherwise this method performance will soon degrade to a linked list performance.
  
**Java 7u40+ and Java 8**

String pool size was increased in Java7u40 (this was a major performance update) to 60013. This value allows you to have approximately 30.000 distinct strings in the pool before your start experiencing collisions. Generally, this is sufficient for data which actually worth to intern. You can obtain this value using -XX:+PrintFlagsFinal JVM parameter.

## Performance and Optimizations

In Java 6, the only optimization we can perform is increasing the PermGen space during the program invocation with the MaxPermSize JVM option:

```
-XX:MaxPermSize=1G
```

In Java 7, we have more detailed options to examine and expand/reduce the pool size. Let’s see the two options for viewing the pool size:

```
-XX:+PrintFlagsFinal

-XX:+PrintStringTableStatistics
```

The default pool size is 1009. If we want to increase the pool size, we can use the StringTableSize JVM option:

```
-XX:StringTableSize=4901
```

Note that increasing the pool size will consume more memory but has the advantage of reducing the time required to insert the Strings into the table.

## Java 9 

Until Java 8, Strings were internally represented as an array of characters – char[], encoded in UTF-16, so that every character uses two bytes of memory.

With Java 9 a new representation is provided, called Compact Strings. This new format will choose the appropriate encoding between char[] and byte[] depending on the stored content.

Since the new String representation will use the UTF-16 encoding only when necessary, the amount of heap memory will be significantly lower, which in turn causes less Garbage Collector overhead on the JVM.

## Summary

* Stay away from String.intern() method on Java 6 due to a fixed size memory area (PermGen) used for JVM string pool storage.
* Java 7 and 8 implement the string pool in the heap memory. It means that you are limited by the whole application memory for string pooling in Java 7 and 8.
* Use -XX:StringTableSize JVM parameter in Java 7 and 8 to set the string pool map size. It is fixed, because it is implemented as a hash map with lists in the buckets. Approximate the number of distinct strings in your application (which you intend to intern) and set the pool size equal to some prime number close to this value multiplied by 2 (to reduce the likelihood of collisions). It will allow String.intern to run in the constant time and requires a rather small memory consumption per interned string (explicitly used Java WeakHashMap will consume 4-5 times more memory for the same task).
* The default value of -XX:StringTableSize parameter is 1009 in Java 6 and Java 7 until Java7u40. It was increased to 60013 in Java 7u40 (same value is used in Java 8 as well).
* If you are not sure about the string pool usage, try -XX:+PrintStringTableStatistics JVM argument. It will print you the string pool usage when your program terminates.

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory8.png)