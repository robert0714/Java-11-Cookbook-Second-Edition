# Concurrent and Multithreaded Programming
Concurrent programming has always been a difficult task. It is a source of many hard-to-solve problems. In this chapter, we will show you different ways to incorporate concurrency and some best practices, such as immutability, which helps to create multithreaded processing. We will also discuss the implementation of some commonly used patterns, such as divide and- conquer and publish-subscribe, using the constructs provided by Java. We will cover the following recipes:

* Using the basic element of concurrency—thread
* Different synchronization approaches
* Immutability as a means of achieving concurrency
* Using concurrent collections
* Using the executor service to execute async tasks
* Using fork/join to implement divide-and-conquer
* Using flow to implement the publish-subscribe pattern

## Introduction
Concurrency—the ability to execute several procedures in parallel—becomes increasingly important as big-data analysis moves into the mainstream of modern applications. Having CPUs or several cores in one CPU helps increase the throughput, but the growth rate of the data volume will always outpace hardware advances. Besides, even in a multiple-CPU system, one still has to structure the code and think about resource-sharing to take advantage of the available computational power.  

In the previous chapters, we demonstrated how lambdas with functional interfaces and parallel streams made concurrent processing part of the toolkit of every Java programmer. One can easily take advantage of this functionality with minimal, if any, guidance.

In this chapter, we will describe some other—old (before Java 9) and new—Java features and APIs that allow more control over concurrency. The high-level concurrency Java API has been around since Java 5. The JDK Enhancement Proposal (JEP) 266, More Concurrency Updates (http://openjdk.java.net/jeps/266), introduced, to Java 9 in the java.util.concurrent package.

```
"an interoperable publish-subscribe framework, enhancements to the CompletableFuture API, and various other improvements"
  But before we dive into the details of the latest additions, let's review the basics of concurrent programming in Java and see how to use them. 
```
Java has two units of execution—process and thread. A process usually represents the whole JVM, although an application can create another process using ProcessBuilder. But since the multiprocess case is outside the scope of this book, we will focus on the second unit of execution, that is, a thread, which is similar to a process but less isolated from other threads and requires fewer resources for execution. 

A process can have many threads running and at least one thread called the ***main*** thread. Threads can share resources, including memory and open files, which allows for better efficiency. But it comes with a price of higher risk of unintended mutual interference and even blocking of the execution. That is where programming skills and an understanding of the concurrency techniques are required. And that is what we are going to discuss in this chapter.

## Using the basic element of concurrency – thread
In this chapter, we will look at the java.lang.Thread class and see what it can do for concurrency and program performance in general.

### Getting ready
A Java application starts as the main thread (not counting system threads that support the process). It can then create other threads and let them run in parallel, sharing the same core via time-slicing or having a dedicated CPU for each thread. This can be done using the java.lang.Thread class that implements the Runnable functional interface with only one abstract method, run(). 

There are two ways to create a new thread: creating a subclass of Thread, or implementing the Runnable interface and passing the object of the implementing class to the Thread constructor. We can invoke the new thread by calling the start() method of the Thread class which, in turn, calls the run() method that was implemented.

Then, we can either let the new thread run until its completion or pause it and let it continue again. We can also access its properties or intermediate results if needed. 

### How to do it...
First, we create a class called AThread that extends Thread and overrides its run() method: 

```java
class AThread extends Thread {
  int i1,i2;
  AThread(int i1, int i2){
    this.i1 = i1;
    this.i2 = i2;
  }
  public void run() {
    IntStream.range(i1, i2)
             .peek(Chapter07Concurrency::doSomething)
             .forEach(System.out::println);
  }
}
```

In this example, we want the thread to generate a stream of integers in a certain range. Then, we use the peek() operation to invoke the doSomething() static method of the main class for each stream element in order to make the thread busy for some time. Refer to the following code:

```java
int doSomething(int i){
  IntStream.range(i, 100000).asDoubleStream().map(Math::sqrt).average();
  return i;
}
```
As you can see, the doSomething() method generates a stream of integers in the range of i to 99999; it then converts the stream into a stream of doubles, calculates the square root of each of the stream elements, and finally calculates an average of the stream elements. We discard the result and return the passed-in parameter as a convenience that allows us to keep the fluent style in the stream pipeline of the thread, which ends by printing out each element. Using this new class, we can demonstrate the concurrent execution of the three threads, as follows:  

```java
Thread thr1 = new AThread(1, 4);
thr1.start();

Thread thr2 = new AThread(11, 14);
thr2.start();

IntStream.range(21, 24)
         .peek(Chapter07Concurrency::doSomething)
         .forEach(System.out::println);
```

The first thread generates the integers 1, 2, and 3, the second generates the integers 11, 12, and 13, and the third thread (main one) generates 21, 22, and 23.

As mentioned before, we can rewrite the same program by creating and using a class that could implement the Runnable interface: 

```java
class ARunnable implements Runnable {
  int i1,i2;
  ARunnable(int i1, int i2){
    this.i1 = i1;
    this.i2 = i2;
  }
  public void run() {
    IntStream.range(i1, i2)
             .peek(Chapter07Concurrency::doSomething)
             .forEach(System.out::println);
  }
}
```
One can run the same three threads like this:

```java
Thread thr1 = new Thread(new ARunnable(1, 4));
thr1.start();

Thread thr2 = new Thread(new ARunnable(11, 14));
thr2.start();

IntStream.range(21, 24)
         .peek(Chapter07Concurrency::doSomething)
         .forEach(System.out::println);
We can also take advantage of Runnable being a functional interface and avoid creating an intermediate class by passing in a lambda expression instead:

Thread thr1 = new Thread(() -> IntStream.range(1, 4)
                  .peek(Chapter07Concurrency::doSomething)
                  .forEach(System.out::println));
thr1.start();

Thread thr2 = new Thread(() -> IntStream.range(11, 14)
                  .peek(Chapter07Concurrency::doSomething)
                  .forEach(System.out::println));
thr2.start();

IntStream.range(21, 24)
         .peek(Chapter07Concurrency::doSomething)
         .forEach(System.out::println);
```
Which implementation is better depends on your goal and style. Implementing Runnable has an advantage (and in some cases, is the only possible option) that allows the implementation to extend another class. It is particularly helpful when you would like to add thread-like behavior to an existing class. You can even invoke the run() method directly, without passing the object to the Thread constructor. 

Using a lambda expression wins over the Runnable implementation when only the run() method implementation is needed, no matter how big it is. If it is too big, you can isolate it in a separate method:

```java
public static void main(String arg[]) {
  Thread thr1 = new Thread(() -> runImpl(1, 4));
  thr1.start();

  Thread thr2 = new Thread(() -> runImpl(11, 14));
  thr2.start();

  runImpl(21, 24);
}

private static void runImpl(int i1, int i2){
  IntStream.range(i1, i2)
           .peek(Chapter07Concurrency::doSomething)
           .forEach(System.out::println);
}
```
One would be hard-pressed to come up with a shorter implementation of the preceding functionality.

If we run any of the preceding versions, we will get an output that looks something like this:

```bash
11
1 
21
12
22
2 
13
3 
23
```
As you can see, the three threads print out their numbers concurrently, but the sequence depends on the particular JVM implementation and underlying operating system. So, you will probably get a different output. Besides, it also may change from run to run. 

The Thread class has several constructors that allow setting the thread name and the group it belongs to. Grouping threads helps manage them if there are many threads running in parallel. The class also has several methods that provide information about the thread's status and properties, and allow us to control the thread's behavior. Add these two lines to the preceding example: 
```java
System.out.println("Id=" + thr1.getId() + ", " + thr1.getName() + ",
                   priority=" + thr1.getPriority() + ",
                   state=" + thr1.getState());
System.out.println("Id=" + thr2.getId() + ", " + thr2.getName() + ",
                   priority=" + thr2.getPriority() + ",
                   state=" + thr2.getState());
```
 The result of the preceding code will look something like this:


Next, say you add a name to each thread:

```java
Thread thr1 = new Thread(() -> runImpl(1, 4), "First Thread");
thr1.start();

Thread thr2 = new Thread(() -> runImpl(11, 14), "Second Thread");
thr2.start();
```
In this case, the output will show the following:


The thread's id is generated automatically and cannot be changed, but it can be reused after the thread is terminated. Several threads, on the other hand, can be set with the same name. The execution priority can be set programmatically with a value between Thread.MIN_PRIORITY and Thread.MAX_PRIORITY. The smaller the value, the more time the thread is allowed to run, which means it has higher priority. If not set, the priority value defaults to Thread.NORM_PRIORITY. 

The state of a thread can have one of the following values:

* NEW: When a thread has not yet started
* RUNNABLE: When a thread is being executed
* BLOCKED: When a thread is blocked and is waiting for a monitor lock
* WAITING: When a thread is waiting indefinitely for another thread to perform a particular action
* TIMED_WAITING: When a thread is waiting for another thread to perform an action for up to a specified waiting time
* TERMINATED: When a thread has exited

The sleep() method can be used to suspend the thread execution for a specified (in milliseconds) period of time. The complementary interrupt() method sends InterruptedException to the thread that can be used to wake up the sleeping thread. Let's work this out in the code and create a new class:

```java
class BRunnable implements Runnable {
  int i1, result;
  BRunnable(int i1){ this.i1 = i1; }
  public int getCurrentResult(){ return this.result; }
  public void run() {
    for(int i = i1; i < i1 + 6; i++){
      //Do something useful here
      this.result = i;
      try{ Thread.sleep(1000);
      } catch(InterruptedException ex){}
    }
  }
}
```
The preceding code produces intermediate results, which are stored in the result property. Each time a new result is produced, the thread pauses (sleeps) for one second. In this specific example, written for demonstrative purposes only, the code does not do anything particularly useful. It just iterates over a set of values and considers each of them a result. In real-world code, you would do some calculations based on the current state of the system and assign the calculated value to the result property. Now let's use this class:

```java
BRunnable r1 = new BRunnable(1);
Thread thr1 = new Thread(r1);
thr1.start();

IntStream.range(21, 29)
         .peek(i -> thr1.interrupt())
         .filter(i ->  {
           int res = r1.getCurrentResult();
           System.out.print(res + " => ");
           return res % 2 == 0;
         })
         .forEach(System.out::println);
```
The preceding code snippet generates a stream of integers—21, 22, ..., 28. After each integer is generated, the main thread interrupts the thr1 thread and lets it generate the next result, which is then accessed via the getCurrentResult() method. If the current result is an even number, the filter allows the generated number flow to be printed out. If not, it is skipped. Here is a possible result:


The output may look different on different computers, but you get the idea: this way, one thread can control the output of another thread.

### There's more...
There are two other important methods that support thread cooperation. The first is the join() method, which allows the current thread to wait until another thread is terminated. Overloaded versions of join() accept the parameters that define how long the thread has to wait before it can do something else. 

The setDaemon() method can be used to make the thread terminate automatically after all the non-daemon threads are terminated. Usually, daemon threads are used for background and supporting processes.
