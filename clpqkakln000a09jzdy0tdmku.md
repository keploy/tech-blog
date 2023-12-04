---
title: "MongoDB in Mock Mode: Acting the Server Part"
seoTitle: "MongoDB in Mock Mode: Acting the Server Part"
seoDescription: "By leveraging the network interactions between the MongoDB client driver and the server, we can emulate the MongoDB server's behaviour."
datePublished: Mon Dec 04 2023 07:00:10 GMT+0000 (Coordinated Universal Time)
cuid: clpqkakln000a09jzdy0tdmku
slug: mongodb-in-mock-mode-acting-the-server-part
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1699120189805/b6e50012-6109-434a-95cc-1e3e142b9b87.png
tags: go, http, mongodb, proxy-server

---

In the contemporary software development landscape, unit tests have become paramount for ensuring software quality. A prevalent practice during these tests involves mocking outgoing calls to decouple the code under test from external dependencies. Imagine, for a moment, a scenario where instead of writing mocks with pre-defined behaviours, you duplicate the behaviour of a real-world server. Picture a restaurant where the chef's apprentice mimics each of the chef's moves in real-time, creating an identical dish concurrently. That's precisely what we're diving into here. By leveraging the genuine network interactions between the MongoDB client driver and the server, we can emulate the MongoDB server's behaviour, acting almost like the database server itself.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1698287726832/1bcd2dad-1da5-47c7-826f-44973577d4d8.png align="center")

### **Introducing Keploy: Bringing the Mocking Concept to Life**

Keploy harnesses the aforementioned concept to mock MongoDB operations within test cases. Instead of relying on static mock responses, it captures and uses genuine network interactions to reproduce the actions of servers like MongoDB. By doing so, it ensures complete isolation in testing environments, allowing developers to validate their applications without any external dependencies or interference.

### **The Initial Hurdle: Grasping Packet Capturing**

Keploy employs a mechanism that seamlessly redirects network packets. When an application sends a request, instead of heading straight to the intended server, the packets are gracefully rerouted to Keploy's proxy. But the story doesn't end here. This proxy, acting as an intermediary, then diligently forwards the traffic to the actual server. This transparent redirection ensures that the application remains blissfully unaware of this detour, while Keploy captures, analyzes, and possibly even alters the packets in transit. This methodology not only allows for efficient packet analysis but also provides an avenue for testing and mocking server behaviours.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1699114495956/7b6c0fce-319a-45ce-a3df-da965fba60a2.png align="center")

### **Beyond Binaries: Parsing Packets for Human-Friendly Mocks**

Armed with a trove of captured network packets, we stand at the next juncture of our journey. Raw binary data, while invaluable, isn't conducive to generating intuitive and actionable mocks. To transition from a stream of 1s and 0s to something more legible to our eyes, we must parse these packets. Doing so allows us to store them in a format that's not just understandable but can also be manipulated easily to create meaningful test scenarios. Join me as we unravel the MongoDB protocol and transform these intricate packets into human-friendly representations.

### Expecting HTTP: MongoDB's Surprising Protocol Strategy

While integrating support for DynamoDB, a managed NoSQL database, we observed it uses the HTTP protocol in the SDK/CLI for communication with remote servers. Elasticsearch also uses a RESTful HTTP API for client-server interaction. This observation led to the assumption that MongoDB, too, would be utilizing the advantageous HTTP/HTTPS protocols for efficient and secure client-server communication. The key benefits would be:

* ***Simplicity***: It will be easier to write drivers in different languages because of the simple text protocol.
    
* ***Popularity***: HTTP is a well-known and widely used protocol. So, we can quickly integrate other tools using the HTTP interface.
    

Contrary to expectations, the MongoDB network packets routed through Keploy's proxy do not conform to the typical `HTTP 'METHOD /url HTTP/1.1'` format. The data read from the client/server connections isn't ASCII-encoded. When employing a network sniffing tool alongside an application executing MongoDB calls, it became evident that a binary protocol governs communication.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692087818356/d5115455-3dfe-40fe-9b57-074446af10fd.gif align="center")

Then, I discovered the binary protocol used by Mongo, known as the [MongoDB wire protocol](https://www.mongodb.com/docs/manual/reference/mongodb-wire-protocol/). This choice of binary protocol is strategic, offering numerous benefits over traditional HTTP/HTTPS, enhancing both efficiency and performance. The benefits are:

* **Smaller Packet Size**: The binary protocol will reduce the size of the request-response packet. This will eliminate the HTTP overhead (like Method, URL, etc.).
    
* **Efficient Serialisation**: Binary protocols are optimised for efficient serialisation and deserialisation of data structures. This leads to faster processing at both the client and server ends.
    
* **Complex Query Support**: Since the binary protocol can be stateful, mongo can handle complex queries and indexing.
    

### Decoding the Blueprint: The Structure of MongoDB Wire Messages

In pursuit of understanding the binary packets, I knew there had to be a standardized format—a common language that MongoDB uses to communicate. This led me to search for some sort of blueprint. My search ended at the MongoDB documentation, where I found detailed information on the [MongoDB wire protocol](https://www.mongodb.com/docs/manual/reference/mongodb-wire-protocol/). The documents laid out the binary packet structure clearly, providing a roadmap to decode the communications that were, until now, a collection of impenetrable binary sequences.

* **Header:** A wire message has a standard header followed by the request-specific data.
    
    ```go
    struct MsgHeader {    
        int32   messageLength; // total message size, including this    
        int32   requestID;     // identifier for this message
        int32   responseTo;    // requestID from the original request
                               //   (used in responses from the database)    
        int32   opCode;        // message type
    }
    ```
    
* **Body**: The structure of the message body depends on the [OpCode](https://www.mongodb.com/docs/manual/reference/mongodb-wire-protocol/#opcodes) field of the header. `OpMsg` is utilized for all the CRUD operations. Although most of the OpCodes are now deprecated in the latest version of Mongo, it still employs OpQuery/OpReply for the '[hello](https://www.mongodb.com/docs/manual/reference/command/hello/)' and '[isMaster](https://www.mongodb.com/docs/v4.2/reference/command/isMaster/)' commands during the connection handshake.
    

Fortuitously, two vital resources surfaced during my search. The first, a native Go library known as [`wiremessage`](https://pkg.go.dev/go.mongodb.org/mongo-driver/x/mongo/driver/wiremessage), is employed by the MongoDB Go driver for encoding these complex messages into the wire format. The second, a repository maintained by **Coinbase** [`mongobetween`](https://github.com/coinbase/mongobetween), serves as a MongoDB proxy, and more importantly, it contains a [`Decode`](https://github.com/coinbase/mongobetween/blob/ca3d8d78d99847afc747b56c5b4ea31b85bff013/mongo/operations.go#L39) method to decode these binary packets back into Go structures or strings. This method shines a light on the esoteric binary conversation, translating it into something I can manipulate and understand, forming the foundation for the mocking mechanism that Keploy capitalizes on.

> Eureka! In the `wiremessage` library and Coinbase's decoder, we've discovered the Rosetta Stone of MongoDB’s network dialogue, transforming inscrutable binary into intelligible structures. 🎉

### **Observing the Wire Protocol in Action**

During my exploration of message flow, I encountered a series of hurdles. A glaring challenge was the lack of documentation on the connection handshake and request/response cycle between the MongoDB driver and the server via wire messages. This necessitated a hands-on approach, requiring me to delve deep into network connections and learn through experimentation.  
I developed a simple Go program to facilitate the exchange of request and response packets between the client-driver and the MongoDB server. This is the pseudo-code:

```go
func handleConnection(clientConn net.Conn, serverConn net.Conn) {
    // Read requests from the driver client connection
    requestbuffer := make([]byte, 1024)    
    n, _ := clientConn.Read(requestbuffer)

    // write the request to the MongoDB server connection
    serverConn.Write(requestBuffer)

    // Read response from the MongoDB server connection
    responsebuffer := make([]byte, 1024)    
    n, _ := serverConn.Read(responsebuffer)

    // write the response to the driver client connection
    clientConn.Write(responsebuffer)
}
```

### Spring Boot App Error: A Java Issue Uncovered

Upon testing a basic Spring Boot-based MongoDB application with the Keploy CLI commands, I encountered unexpected issues. Specifically, errors related to the MongoDB connection surfaced, indicating the complications in the interaction between the application and the database.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1696278524086/1d079629-e28c-4f61-a224-3280afc7bf83.png align="center")

### Diagnosing and Addressing Connection Errors

Exploring deeper into connection issues, I found that while the wire messages between the driver and server were transmitting flawlessly during debugging, an unexpected behaviour emerged elsewhere. Acting on a spontaneous hunch, I read from the client connection right after dispatching a response to it. To my astonishment, **another request began streaming from that same connection**. The rationale behind this lies in the way drivers handle their connections. They operate on a **Connection Pool** system, frequently reusing these connections, not only to bolster efficiency but also to enable stateful transactions and operations.

***The solution to the problem***

This can be solved by applying a **for loop** which does not break until the `client connection closes` or `EOF`. You can find the applied code [here](https://github.com/keploy/keploy/blob/4fa8be8760931bc9bdef507fd74593b1f9754a08/pkg/proxy/integrations/mongoparser/mongoparser.go#L355).

```go
func handleConnection(clientConn net.Conn, serverConn net.Conn) {  
    for {
        // Read requests from the driver client connection
        requestbuffer := make([]byte, 1024)    
        n, err := clientConn.Read(requestbuffer)
        if err!=nil { 
            // Error occurs when reading from a closed connection.
            log.Error("failed to read from client connection", err)
            break
        }
    
        // write the request to the MongoDB server connection
        serverConn.Write(requestBuffer)
    
        // Read response from the MongoDB server connection
        responsebuffer := make([]byte, 1024)    
        n, err := serverConn.Read(responsebuffer)
    
        // write the response to the driver client connection
        n, err = clientConn.Write(responsebuffer)
        if err!=nil {
            // Error occurs when writing to a closed connection.
            log.Error("failed to read from server connection", err)
            break
        }
    }
}
```

### Addressing the Persistent Java Error: Piecing Together the Puzzle

The path to resolving the Java error was proving to be as complex as it was intriguing. Despite the tools and data at my disposal, the solution remained elusive. I embarked on a meticulous analysis of the recorded packets, scrutinizing the significance of each field in the hope of uncovering what was amiss.

As I delved deeper, a potential culprit emerged: the [`moreToCome`](https://www.mongodb.com/docs/manual/reference/mongodb-wire-protocol/#flag-bits) flag bit within the **OpMsg** structure. It was a revelation that shed new light on the issue. This particular bit was set in the heartbeat call—a function that checks the vitality of the connection to the MongoDB server. It suggested that the communication could be **fragmented**, with MongoDB employing a mechanism to indicate there was more data to follow.

This insight was crucial. It meant that the data exchange was not always a straightforward request-response pair; sometimes it spanned across multiple packets. Understanding this, I realized that our handling of the network packets needed to accommodate these streaming messages. This would ensure that each segment was captured and reassembled correctly to provide a full picture of the interaction—a puzzle slowly piecing itself together.

### Perfecting Packet Handling: Implementing a Loop for Chunks

With the newfound understanding of MongoDB's `moreToCome` flag, I recognized the need for a refined approach to packet handling. To address the segmented nature of the communication, I implemented a for loop that diligently reads and processes these chunks of packets.

This loop wasn't just about reading data; it would also pass the read buffer packet to the opposite connection, ensuring no part of the communication was missed. It waits for an entire transaction to complete, handling the life cycle of the request-response process collectively. By doing this, it captures the full conversation between the client and the server, even if it spans multiple packets.

With the loop in place, each transaction is recorded as a whole, despite the fragmented nature of the transmission. This approach not only ensures the completeness of the data captured but also paves the way for accurate and reliable mock generation for our testing environment. The seemingly endless iterations are not just a repetition of tasks but an assurance that we are collecting every piece of the conversation, no matter how many packets it takes.

You can find the deployed code [here](https://github.com/keploy/keploy/blob/4fa8be8760931bc9bdef507fd74593b1f9754a08/pkg/proxy/integrations/mongoparser/mongoparser.go#L403).

> And just like that, with a resounding boom, mocks are now effortlessly spun up from the tapestry of network interactions. 🧨

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1699116558598/4e8e1fa0-0295-4d22-b3c8-cfe4f2d25f4f.gif align="center")

### Conclusion

In summary, the parser we've integrated is a game-changer. It simplifies the process of encoding and decoding binary packets, streamlining the creation of mocks for our tests. What's more, these mocks aren't just a one-off; they are mutable, adapting with ease to future changes. This flexibility ensures our testing environment remains robust and up to date, allowing us to maintain high-quality software with less effort.