---
title: "Efficient TCP Server Connection Management"
seoTitle: "Efficient TCP Server Connection Management"
seoDescription: "Learn about effective connection handling in TCP servers using asynchronous I/O, thread pools, and goroutines to optimize performance and scalability"
datePublished: Sat Jan 04 2025 05:51:51 GMT+0000 (Coordinated Universal Time)
cuid: cm5hrnwce00000al80es53gi2
slug: efficient-tcp-server-connection-management
canonical: https://keploy.io/blog/technology/efficient-tcp-server-connection-management
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712028083666/4f539748-49ad-4457-873c-c00724b5d084.webp
tags: go, concurrency, tcp, goroutines

---

TCP connections, they are everywhere, almost every online interaction you make, whether it's streaming your favourite video, sending an important email, or just casually browsing through different websites. They are like the foundational building blocks of the internet and so it's important for them to be consistent and reliable.

As you can see TCP servers get and handle billions of requests in a day, so it's important for them to be able to do this efficiently. Some of the common ways that TCP servers employ to do this are:

1. **Asynchronous I/O(select/poll/epoll)**: In traditional blocking I/O, when you initiating an operation, like reading from a socket, it gets blocked until that operation is completed, which can be absolutely detrimental to a TCP server's performance. The asynchronous I/O was developed to deal with this exact problem. The core idea is for the operating system to notify the application when the operation has completed. You provide the application with a set of file descriptors and then the operating system monitors those file descriptors behind the scenes. The application blocks on the select/poll/epoll function and whenever any event occurs, the operating system sends a signal and the function unblocks. Your application can then only handle the connection that needs attention without wasting time on connections that are idle.
    

Below is a simple code example for the same:

```go
package main

import (
    "fmt"
    "net"
    "os"
    "syscall"
)

func main() {
    const maxConnections = 10 // You can adjust this
    pollfds := make([]syscall.PollFd, maxConnections)
    connections := make([]net.Conn, maxConnections)
    activeConnections := 0  // Number of currently active connections

    // Set up listener
    listener, err := net.Listen("tcp", ":8080")
    if err != nil {
        fmt.Println("Error setting up listener:", err)
        os.Exit(1)
    }
    defer listener.Close()
    fmt.Println("Server listening on port 8080")

    for {
        // Add new connections to the pollfds slice
        if activeConnections < maxConnections {
            conn, err := listener.Accept()
            if err != nil {
                fmt.Println("Error accepting connection:", err)
                continue
            }

            // Get the file descriptor from the connection
            rawConn, err := conn.SyscallConn()
            if err != nil {
                fmt.Println("Error getting syscall connection:", err)
                conn.Close()
                continue
            }

            var fd int
            err = rawConn.Control(func(f uintptr) {
                fd = int(f)
            })
            if err != nil {
                fmt.Println("Error getting file descriptor:", err)
                conn.Close()
                continue
            }

            pollfds[activeConnections] = syscall.PollFd{
                Fd:     int32(fd),
                Events: syscall.POLLIN, // Monitor for readability
            }
            connections[activeConnections] = conn
            activeConnections++
            fmt.Println("New connection established")
        }

        // Wait for events using poll()
        n, err := syscall.Poll(pollfds[:activeConnections], -1) // -1 for indefinite timeout
        if err != nil {
            fmt.Println("Error in poll:", err)
            continue
        }

        // Handle events
        for i := 0; i < activeConnections; i++ {
            if pollfds[i].Revents&syscall.POLLIN != 0 {
                conn := connections[i]
                buf := make([]byte, 1024)
                n, err := conn.Read(buf)
                if err != nil {
                    fmt.Println("Error reading from connection:", err)
                    conn.Close()
                    pollfds[i], pollfds[activeConnections-1] = pollfds[activeConnections-1], pollfds[i]
                    connections[i], connections[activeConnections-1] = connections[activeConnections-1], connections[i]
                    activeConnections--
                    continue
                }

                fmt.Printf("Received data: %s\n", string(buf[:n]))
            }
        }
    }
}
```

To run this code snippet, you need have go installed on your system, add this to a file called main.go and run it with

```bash
go run main.go
```

You can use the curl request below to test the server:

```bash
curl -X POST --data "Hello, Server!" http://localhost:8080
```

Now when you make several telnet or curl calls with keep alive on or off you will get the following message:

```bash
New connection established
```

When data is sent to the server, it will say

```bash
Received data: Hello, Server!
```

Let's delve a bit deeper into how select, poll and epoll work. All three mechanisms are fundamentally designed for asynchronous I/O. They allow a program to monitor multiple file descriptors and get notified when one or more become ready for I/O operations (reading, writing, etc.). But there are some key differences between them.

### **Scalability**

select: This has limitations with scalability with a large number of file descriptors. Its underlying implementation often involves a linear scan through all monitored file descriptors on each invocation, which becomes a bottleneck.

**poll:** Improves upon `select`, though it may still exhibit overhead when dealing with numerous descriptors.

**epoll (Linux-specific):** Designed for high scalability scenarios. It introduces a concept of an "event list" maintained by the kernel, which avoids the need to rescan all file descriptors on every `epoll_wait` call.

**Handling of File Descriptor Updates:**

* **select/poll:** To add or remove descriptors you want to monitor, you need to modify the data structures passed to these functions on each call, which can add overhead.
    
* **epoll:** Uses an `epoll` control file descriptor and dedicated system calls (`epoll_ctl`) to add, remove, or modify monitored file descriptors. This offers more efficient updates, especially in dynamic scenarios.
    

**Level-Triggered vs. Edge-Triggered:**

* **select/poll:** Primarily level-triggered. They return a file descriptor as ready if the associated condition (readability, writeability) persists. This can lead to repeated notifications if you don't consume all available data.
    
* **epoll:** Supports both level-triggered and edge-triggered modes. Edge-triggered mode signals a file descriptor as ready only when an event occurs, not if the condition continues to hold. This can help prevent redundant notifications.
    

### **Use Case**

* **select:**
    
    * Best suited for simple scenarios with a small number of file descriptors.
        
    * Acceptable if portability across a wide range of Unix-like systems is a major requirement.
        
* **poll:**
    
    * Suitable when better scalability than `select` is needed, but truly high-performance demands are not present.
        
    * Offers slightly easier management of file descriptors than `select`.
        
* **epoll:**
    
    * Ideal for high-performance, high-concurrency servers on Linux, especially those handling large numbers of connections.
        
    * Its edge-triggered mode can often increase efficiency.
        

2. **Thread Pools:** In thread pools we basically pre alllocate a specific number of threads from the memory so that we don't have to deal with the creation and deletion of new threads everytime a new connection comes in. Whenever a new connection comes in, it is assigned to a thread that takes care of its execution. This significantly reduces the overhead cost of creating a destroying threads. This also gives you a control over the number of threads created so that your system doesn't get overwhelmed by the number of threads. The obvious problem with this approach is when the thread pool runs out, the new incoming connections have to wait for one of the threads in the pool to get free, to handle it. Also, even though the overhead of creating a destroying threads is reduced, there is still overhead in switching between the threads when they handle different connections. There is also a scaling problem with this approach, if the traffic suddenly increases, we will not be able to scale the system at that speed. Below is a code snippet demonstrating the approach:
    

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Simulate a task/ job for the thread pool
type Job struct {
    id int
}

// The actual worker function performing tasks
func worker(id int, jobs <-chan Job, wg *sync.WaitGroup) {
    defer wg.Done() // Indicate to WaitGroup that the worker has finished

    for job := range jobs {
        fmt.Println("Worker", id, "started job", job.id)
        time.Sleep(time.Second) // Simulate work
        fmt.Println("Worker", id, "finished job", job.id)
    }
}

func main() {
    const numJobs = 10
    const numWorkers = 3

    jobs := make(chan Job, numJobs) // Buffered channel for jobs
    var wg sync.WaitGroup           // To track when all workers are done

    // Create the thread pool 
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go worker(i, jobs, &wg)
    }

    // Submit jobs to the pool
    for i := 0; i < numJobs; i++ {
        jobs := Job{id: i}
        jobs <- jobs
    }
    close(jobs) // No more jobs will be submitted

    // Wait for all workers to finish
    wg.Wait()
}
```

Common Problem With Thread Pools

Thread pools are obviously a great way to handle incoming traffic, but the problem arises when your thread pool is full and you still have new connections coming in. In that case, it becomes crucial how you store the rest of the connections so that they can be further processed by your thread pool when it is free. Some of the common ways to deal with this are - maintaining a queue of the new connections, rejecting the new signals until the pool is free again or dynamically scaling the thread pool. The technique that I used, was maintaining a queue of the new connections to keep a track of the new connections until the thread pool was free again, and when the thread pool got free again, I took out the connections one at a time from the thread pool and processed them in a goroutine. Using the rejection approach was obviously not correct as I was supposed to capture tests for all the incoming requests even though my thread pool was full. Dynamically scaling the thread pool is a tedious trail because our application can run on different machines with different amount of resources, so it is not very possible to have all the information for dynamically scaling the goroutines as and when required by the application.

It is important to note that although we are using queues to store the connections, we still need to define a specific size of the queue, based on the resources available, and the traffic burst expected. I used a simple FIFO(First-in, First-out) because i had to handle all the connections in the same way, but you can use a priority queue if your implementation demands the same. Here is a code sample on how to do the same:

```go
package main

import (
    "fmt"
    "net"
    "sync"
    "time"
)

const (
    maxQueueSize  = 20 
    threadPoolSize = 5
)

var (
    connectionQueue []net.Conn
    queueMutex      sync.Mutex
)

func main() {
    listener, err := net.Listen("tcp", "localhost:8000")
    if err != nil {
        // ... handle error
    }
    defer listener.Close()

    // Start worker threads
    for i := 0; i < threadPoolSize; i++ {
        go worker()
    }

    // Accept and enqueue connections
    for {
        conn, err := listener.Accept()
        if err != nil {
            // ... handle error
        }
        enqueueConnection(conn)
    }
}

func enqueueConnection(conn net.Conn) {
    queueMutex.Lock()
    defer queueMutex.Unlock()

    if len(connectionQueue) == maxQueueSize {
        fmt.Println("Queue full, cannot accept connection at the moment")
        return 
    }

    connectionQueue = append(connectionQueue, conn)
    fmt.Println("Connection enqueued")
}

func worker() {
    for {
        conn := dequeueConnection()
        if conn != nil {
            handleConnection(conn) 
        } else {
            // If you want worker to rest if the queue is empty:
            time.Sleep(100 * time.Millisecond) 
        }
    }
}

func dequeueConnection() net.Conn {
    queueMutex.Lock()
    defer queueMutex.Unlock()

    if len(connectionQueue) == 0 {
        return nil 
    }

    conn := connectionQueue[0]
    connectionQueue = connectionQueue[1:]
    return conn
}

func handleConnection(conn net.Conn) {
    // Handling your connection.
    conn.Close()
}
```

3. **GoRoutines**: Go offers lightweight processes called goroutines that share the same address space as the main process. They are ideal for handling concurrent tasks within a single Go application. Unlike traditional TCP servers, they do not block on a specific I/O operation and they also do not get affected by a large influx of connections as is the case with threads pools. This improves the responsiveness and the speed of the TCP servers. Here at Keploy, we use this exact approach to manage the large number of connections that come to us when we are recording the testcase. Essentially, we use select statements to handle the various operations of these connections. When a new connection comes in, a new goroutine is spawned, inside the goroutine, we have a select statement, that encompasses all the scenarios that can take place with the connection. It can either send data events, get closed or become stale. To handle all these conditions, we have cases inside the select statement that takes care of them. The select statement looks something like this:
    

```go
package main

import (
    "fmt"
    "net"
    "time"
)

func handleConnection(conn net.Conn) {
    defer conn.Close()

    // Simulate connection activity with a loop
    for {
        buf := make([]byte, 1024)
        n, err := conn.Read(buf)
        if err != nil {
            fmt.Println("Error reading from connection:", err)
            return // Connection closed or error
        }
        fmt.Printf("Received data from %v: %s\n", conn.RemoteAddr(), buf[:n])

        // Simulate some processing time
        time.Sleep(500 * time.Millisecond) 
    }
}

func main() {
    listener, err := net.Listen("tcp", ":8080")  
    if err != nil {
        fmt.Println("Error creating listener:", err)
        return
    }
    defer listener.Close()

    connections := make([]net.Conn, 0)

    for {
        conn, err := listener.Accept()
        if err != nil {
            fmt.Println("Error accepting connection:", err)
            continue 
        }
        connections = append(connections, conn)

        go handleConnection(conn) // Handle in a separate goroutine

        select {
        case conn := <-connections: 
            // A connection has been closed, find and remove it
            for i, c := range connections { 
                if c == conn {
                    connections = append(connections[:i], connections[i+1:]...) 
                    break 
                }
            }
        default:
            // No event to handle, prevent busy waiting
            time.Sleep(100 * time.Millisecond)  
        }
    }
}
```

So as we have seen, picking the correct approach to handle these large number of connections greatly impacts how TCP servers handle the ever-growing demands of the internet. From thread pools, to the power of goroutines, and with OS-level optimizations like epoll, developers have powerful tools to build robust and scalable servers.

So the next time you stream a video, send a message, or browse a website, remember the intricate dance of TCP connections and concurrency techniques working tirelessly behind the scenes. It is these techniques that make our seamless online experiences possible.

## Conclusion

TCP connections are the backbone of the internet, powering everything from video streaming to email communications. Behind the scenes, efficient handling of these connections is critical for building robust, high-performance servers. Techniques like **asynchronous I/O (select, poll, epoll)**, **thread pools**, and **goroutines** showcase the evolution of concurrency management, each offering unique advantages and trade-offs depending on the application's requirements.

For lightweight, high-concurrency applications, **goroutines** and **epoll** shine as they balance performance with scalability. Thread pools, though slightly heavier, offer structured control, particularly in environments with resource constraints. Meanwhile, asynchronous I/O mechanisms provide the foundation for scalable and responsive servers.

Choosing the right approach depends on the specific use case, hardware resources, and expected traffic patterns. Understanding these strategies equips developers to optimize server performance and deliver seamless user experiences.

## FAQs

### **What is the main difference between** `select`, `poll`, and `epoll`?

`select` and `poll` are older mechanisms that work well for a small number of file descriptors but face scalability issues due to linear scans. `epoll`, specific to Linux, is optimized for high-performance scenarios with many file descriptors, using event lists maintained by the kernel for efficiency.

### **Why are goroutines considered better than traditional threads for TCP servers?**

Goroutines are lightweight, efficient, and managed by Go’s runtime scheduler. Unlike threads, they don’t require extensive memory and context-switching overhead, allowing thousands of goroutines to run concurrently in a single application.

### **How does a thread pool improve server performance?**

A thread pool preallocates a fixed number of threads, reducing the overhead of creating and destroying threads for each connection. This ensures better resource management and limits the number of active threads to prevent overwhelming the system.

### **What is edge-triggered mode in** `epoll`, and when should it be used?

In edge-triggered mode, `epoll` notifies an application of an event only when it occurs, not if the condition persists. This reduces redundant notifications, making it suitable for high-performance scenarios where efficient event handling is critical. However, it requires careful coding to avoid missing events.