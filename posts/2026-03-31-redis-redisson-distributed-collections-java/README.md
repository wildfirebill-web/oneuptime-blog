# How to Use Redisson Distributed Collections in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, Redisson

Description: Learn how to use Redisson distributed collections like RList, RSet, RQueue, and RSortedSet to share collection state across JVM instances.

---

Redisson maps Redis data structures to standard Java collection interfaces. `RList` implements `List`, `RSet` implements `Set`, and `RQueue` implements `Queue` - all backed by Redis and safe to use from multiple JVM instances simultaneously.

## RList - Distributed List

```java
RList<String> list = client.getList("task:queue");

list.add("task1");
list.add("task2");
list.add(0, "urgent-task"); // insert at index

System.out.println(list.get(0)); // urgent-task
System.out.println(list.size()); // 3

list.remove("task1");
list.forEach(System.out::println);
```

## RSet - Distributed Set

```java
RSet<String> onlineUsers = client.getSet("users:online");

onlineUsers.add("alice");
onlineUsers.add("bob");
onlineUsers.add("alice"); // duplicate, ignored

System.out.println(onlineUsers.size()); // 2
System.out.println(onlineUsers.contains("alice")); // true

// Set operations
RSet<String> premiumUsers = client.getSet("users:premium");
premiumUsers.add("alice");
premiumUsers.add("charlie");

Set<String> intersection = onlineUsers.readIntersection("users:premium");
// intersection = [alice]
```

## RQueue - Distributed Queue

```java
RQueue<String> workQueue = client.getQueue("jobs:pending");

workQueue.offer("job-001");
workQueue.offer("job-002");

String nextJob = workQueue.poll(); // removes and returns head
String peek = workQueue.peek();    // returns head without removing
```

## RBlockingQueue - Blocking Queue

```java
RBlockingQueue<String> blockingQueue = client.getBlockingQueue("events");

// Producer thread
blockingQueue.put("event-data");

// Consumer thread (blocks until item is available)
String event = blockingQueue.take();

// Poll with timeout
String item = blockingQueue.poll(5, TimeUnit.SECONDS);
```

## RDeque - Double-Ended Queue

```java
RDeque<String> deque = client.getDeque("history:browsing");

deque.addFirst("page-A");
deque.addLast("page-B");
deque.addFirst("page-C");

System.out.println(deque.peekFirst()); // page-C
System.out.println(deque.peekLast());  // page-B
```

## RSortedSet - Distributed Sorted Set

```java
RSortedSet<Integer> sortedSet = client.getSortedSet("scores");
sortedSet.add(30);
sortedSet.add(10);
sortedSet.add(20);

sortedSet.forEach(System.out::println); // 10, 20, 30 - sorted order
```

## RScoredSortedSet - Scored Elements

```java
RScoredSortedSet<String> leaderboard = client.getScoredSortedSet("leaderboard");
leaderboard.add(1500.0, "alice");
leaderboard.add(2300.0, "bob");
leaderboard.add(900.0, "charlie");

// Get top 3
Collection<String> top3 = leaderboard.valueRangeReversed(0, 2);
// [bob, alice, charlie]

Double score = leaderboard.getScore("alice"); // 1500.0
```

## Summary

Redisson distributed collections implement standard Java interfaces (`List`, `Set`, `Queue`) backed by Redis, making shared state across multiple JVM instances transparent and type-safe. Blocking collections like `RBlockingQueue` are especially useful for producer-consumer patterns where workers should wait for work to arrive rather than polling.
