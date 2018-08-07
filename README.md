# Table of Contents

- [Stacks and Heap](#stacks-and-heap)
    * 
    

# Stacks and Heap

The Java memory model used internally in the JVM divides memory between thread stacks and the heap. This diagram illustrates the Java memory model from a logic perspective:

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory1.png)

**Each thread running in the Java virtual machine has its own thread stack.** The **thread stack contains information about what methods the thread has called to reach the current point of execution**. I will refer to this as the "call stack". As the thread executes its code, the call stack changes.

**The thread stack also contains all local variables for each method being executed (all methods on the call stack)**. A thread can only access it's own thread stack. Local variables created by a thread are invisible to all other threads than the thread who created it. Even if two threads are executing the exact same code, the two threads will still create the local variables of that code in each their own thread stack. Thus, each thread has its own version of each local variable.

**All local variables of primitive types ( boolean, byte, short, char, int, long, float, double) are fully stored on the thread stack and are thus not visible to other threads.** One thread may pass a copy of a pritimive variable to another thread, but it cannot share the primitive local variable itself.

**The heap contains all objects created in your Java application, regardless of what thread created the object.** This includes the object versions of the primitive types (e.g. Byte, Integer, Long etc.). It does not matter if an object was created and assigned to a local variable, or created as a member variable of another object, the object is still stored on the heap.

![memory](https://github.com/rgederin/java-memory-model/blob/master/img/memory2.png)   
    