---
title: "Choosing the Perfect Message Queue: Factors to Consider"
seoTitle: "Choosing the Perfect Message Queue: Factors to Consider"
seoDescription: "I realised that even the most celebrated solutions come with their own set of trade-offs. In this blog post, I'll explain about various message queue."
datePublished: Tue Apr 30 2024 09:34:41 GMT+0000 (Coordinated Universal Time)
cuid: clvm6zcr900070ajlfc6x8y3e
slug: choosing-the-perfect-message-queue-factors-to-consider
canonical: https://keploy.io/blog/technology/choosing-the-perfect-message-queue-factors-to-consider
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711801898459/cbd37f1f-48f2-4be0-83b3-5c6708d017bb.jpeg
tags: go, redis, message-queue, rabbitmq, kafka

---

Not long ago, I was handed a problem that's no stranger to the world of programming: making asynchronous threads communicate effectively within the same process. Given the widespread nature of this issue, I expected to find an existing solution to resolve it. My search led me to the concept of *message queue,* which seemed promising for streamlining this communication challenge.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711800076416/b3f7bca3-d067-4977-960d-56953ed01cd1.gif align="center")

However, as I delved deeper, I realised that even the most celebrated solutions come with their own set of trade-offs. In this blog post, I'll explain about various message queue, their features, and what you need to know before using them.

### Breaking Down the Issue: Problem Insights and System Needs

We have two parts in our system: a graph handler and an HTTP handler. The HTTP handler waits for a signal (long polling) and its response depends on the graph request. Both handlers are built using Golang, so we need to send data from the graph handler to the HTTP handler. To solve this, we're thinking about using a ***message queue***.

The primary requirements for our message passing system are ***scalability and low latency***. We need to ensure that messages can be passed between the graph and HTTP servers efficiently, even under high load conditions. Scalability is crucial to handle increasing message volumes without compromising performance.

It's important to note that ***persistence is not a requirement*** for our system. Since the messages being passed between the servers are volatile, we can focus on optimising for scalability and low latency without the added complexity of message persistence.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711800250603/708b4394-a829-4faa-9e8a-33bee9014a9c.gif align="center")

## Exploring the Message Queue options

Upon conducting a Google search for the best message queue solution, I found so many options that it was overwhelming. Each claiming to be the ideal solution for different use cases. I realised that selecting the right message queue required a deeper understanding of my specific requirements and the features offered by each solution.

After researching various options, some message queue solutions stood out:

### RabbitMQ:

**Pros:**

* Guaranteed message delivery through consumer acknowledgments, ensuring task scheduling reliability.
    
* Persistent storage for messages, ensuring tasks are scheduled even in the event of a system failure.
    
* Scalable architecture capable of handling tens of thousands of requests per second.
    
* Easy integration using the Go driver for RabbitMQ.
    

**Cons:**

* Requires maintenance of a RabbitMQ server, adding complexity to the system.
    
* Limited scalability for extremely high-throughput scenarios, struggling to handle millions of requests per second.
    

**Use Case:** RabbitMQ is well-suited for scenarios where message delivery guarantees and task scheduling reliability are paramount. For example, in a real-time chat application, RabbitMQ ensures that chat messages are reliably delivered to all users, even under high load conditions.

**Conclusion**: For our system, RabbitMQ's features such as guaranteed *message delivery and persistent storage are not essential*. Since we don't require these features and prioritise simplicity and scalability, RabbitMQ may not be the best choice.

### Redis:

**Pros:**

* Easy integration with the Go native driver, making it straightforward to incorporate into the system.
    
* In-memory database nature provides excellent speed, ideal for high-throughput scenarios.
    
* Capable of handling millions of requests per second, making it suitable for demanding workloads.
    

**Cons:**

* Lack of message persistence, meaning message delivery is not guaranteed in case of a system failure.
    
* Requires maintenance of a Redis server, adding complexity to the system.
    

**Use Case:** Redis is well-suited for scenarios where speed and high-throughput are essential, and message persistence is not a primary concern. For example, Redis is often used to handle user sessions on websites. When a user logs in, Redis stores their session information quickly and efficiently. This allows the website to keep track of logged-in users and provide a smoother experience, even with lots of traffic.

**Conclusion:** Redis appears to be well-suited for our use case. It meets our requirements for scalability and low latency. However, it's important to note that using Redis comes with the cost of maintaining a Redis server. Overall, implementing a Redis server seems like a suitable choice to meet our messaging needs efficiently.

### Kafka:

**Pros:**

* It is highly scalable and can handle millions of messages per second across many producers and consumers.
    
* Messages in Kafka are persisted to disk, providing durability and ensuring that messages are not lost.
    
* It is designed to be fault-tolerant, with built-in replication and failover mechanisms.
    
* Kafka's design allows for high-throughput message processing, making it ideal for use cases with high message volumes.
    
* It also supports real-time stream processing, enabling applications to consume and process messages as they arrive.
    

**Cons:**

* Setting up and managing Kafka clusters can be complex, requiring expertise in distributed systems.
    
* Maintaining Kafka clusters can be resource-intensive, requiring dedicated resources for monitoring and management.
    
* While Kafka is designed for high throughput, it may introduce some latency in message processing compared to more lightweight solutions.
    

**Use Cases:**

* **Log Aggregation:** Kafka is commonly used for log aggregation, collecting and storing log data from various sources.
    
* **Stream Processing:** Kafka's real-time processing capabilities make it suitable for stream processing applications, such as real-time analytics.
    
* **Event Sourcing:** Kafka can be used for event sourcing, storing a log of events that represent changes to a system's state.
    

**Conclusion:** Kafka offers a robust and scalable solution for message queuing and stream processing, making it suitable for use cases that require high throughput and fault tolerance. However, its complexity, additional latency and operational overhead makes it unsuitable for our use case.

## Discussion and Decision Making: Selecting the Right Message Queue

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711801082039/96a01168-1181-47dc-8f99-6d292fbb378b.jpeg align="center")

After sharing my findings with the engineering team, we initiated a discussion to select the appropriate message queue for our system. Following a thorough examination of the features and limitations of different message queues, I recommended using Redis. However, during the discussion, a valid concern was raised regarding the `introduction of a new dependency` to the system by using Redis. This prompted us to consider leveraging ***go channels*** for synchronising go routines as an alternative. We weighed the pros and cons of this approach.

**Pros:**

* **Simplicity:** Go channels provide a simple and straightforward way to communicate between go routines without the need for external libraries or dependencies.
    
* **In-Memory:** Messages sent over Go channels are stored in memory, ensuring fast and efficient message passing.
    
* **Built-in Synchronisation:** Go channels provide built-in synchronisation, making it easy to coordinate and control access to shared resources.
    
* **No External Dependencies:** Using Go channels would not introduce any external dependencies to the system, simplifying the overall architecture.
    
* **Low Latency:** Due to their in-memory nature and efficient design, Go channels offer low latency for message passing.
    

**Cons:**

* **Message Delivery Guarantee:** Unlike message queues like Redis or RabbitMQ, Go channels do not provide guaranteed message delivery. Messages sent over a channel can be lost if the receiving end is not ready to receive them.
    
* **Complexity for Multiple Consumers:** Implementing channels for multiple consumers can introduce complexity, as each channel needs to be managed separately.
    
* **Memory Usage:** Storing messages in memory can lead to increased memory usage, especially for systems with high message volumes or long-running processes.
    

It's finally time to unveil the results of our evaluation! ***Drumroll*** After a thorough analysis of all the pros and cons, the winner, by a landslide, is none other than the trusty Go channel approach! 🎉 Its simplicity, lightning-fast speed, and seamless communication between go routines make it the perfect fit for our system's messaging needs.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711801460063/954dd907-42f8-4da3-bec3-4c19d837f14e.jpeg align="center")

Now, we're gearing up to implement this winning solution by creating a Go package that will house our Publish/Subscribe functions. This package will help in passing messages between different handler in our system, ensuring that our communication is not just efficient, but also reliable!

```go
type queuer struct {
	queue chan string
}

func NewCommandScheduler() Queuer {
	return &queuer{
		queue: make(chan string),
	}
}

func (s *queuer) Publish(ctx context.Context, task string) error {

	select {
		case s.queue <- task:
		case <- ctx.Done():
			return ctx.Err()
	}
	
	return nil
}

func (s *queuer) Subscribe(ctx context.Context) (string, error) {
	
	var task string
	select {
	case task = <-s.queue:
	case <- ctx.Done():
		return nil, ctx.Err()
	}

	return task, nil
}
```

## Conclusion

In conclusion, our exploration of various message queues has provided valuable insights into their features and limitations. We have come to understand that there is no one-size-fits-all solution when it comes to message queues or frameworks. Instead, the best choice depends on the specific requirements of the problem at hand.

By considering factors such as scalability, latency, and complexity, we have determined that the Go channel approach is the perfect fit for our system's needs. Its simplicity, low latency, and efficient communication between go routines make it the ideal choice for our messaging requirements.

This experience has highlighted the importance of evaluating different options based on the unique needs of each project. While there may not be a universally best message queue or framework, there is always a perfect solution for a particular problem.

***Referrences***:

1. [bytebytego](https://blog.bytebytego.com/p/how-to-choose-a-message-queue-kafka)
    
2. [keploy](https://github.com/keploy/keploy)
    

## FAQ's

### What are Go channels?

Go channels are a built-in feature of the Go programming language used for communication between goroutines (concurrently executing functions). They provide a way to send and receive values between goroutines, facilitating concurrent programming in Go.

### What factors should be considered when selecting a message queue solution?

Factors to consider when selecting a message queue solution include scalability, latency, message persistence, complexity, operational overhead, and specific requirements of the use case.

### What are the primary requirements for a message passing system?

The primary requirements for a message passing system typically include scalability, low latency, and sometimes message persistence. Scalability ensures that the system can handle increasing message volumes without performance degradation, while low latency ensures fast message delivery. Message persistence ensures that messages are not lost in case of system failures.

### What are some popular message queue solutions?

Some popular message queue solutions include RabbitMQ, Redis, Kafka, and Apache ActiveMQ. Each solution has its own set of features and trade-offs, making them suitable for different use cases.