---
title: "Tracking Multiple Requests Over a Single Connection with eBPF"
seoTitle: "How to Capture Multiple Requests on the Same Connection Using eBPF"
seoDescription: "Learn how to capture multiple requests on the same connection using eBPF for better observability and performance insights."
datePublished: Fri May 09 2025 05:16:27 GMT+0000 (Coordinated Universal Time)
cuid: cmagcfuve000l09l5857r02jh
slug: tracking-multiple-requests-over-a-single-connection-with-ebpf
canonical: https://keploy.io/blog/community/tracking-multiple-requests-over-a-single-connection-with-ebpf
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1714205765489/dd10c319-c3aa-4fea-afab-7362e995f42d.png
tags: c, linux, go, http, ebpf

---

## **Why capturing HTTP Traffic is Crucial for Network Security?**

Capturing HTTP traffic is like having a digital security camera at your network's entrance, and it's essential for keeping your network safe. This 'camera' helps you monitor who's accessing your online space, allowing you to detect and prevent potential security threats, such as cyberattacks and unauthorized access before they can harm your system. This proactive approach is a fundamental practice to ensure the security and integrity of your network.

## **Behind the Scenes: The Work That Led to These Words**

I was working on tracking my ingress HTTP traffic, and for that, I used the eBPF [**workshop**](https://github.com/DataDog/ebpf-training/tree/main/workshop1) repository from Datadog. It helped me keep a close eye on my network traffic easily and effectively, giving me useful insights for my project.

I discovered that monitoring ingress HTTP traffic is feasible by attaching it to system calls such as `accept/accept4`, `read`, `write`, and `close`. This can be achieved by employing the kprobe feature of eBPF, which allows for applying hooks effectively.

* **Connection Establishment (accept/accept4)**: Here, a client creates a connection with the HTTP server, or inversely, the server acknowledges a client's connection, yielding a new file descriptor in the process.
    
* **Reading/Writing (read/write)**: This involves the client or server performing multiple read and write operations to exchange HTTP requests and responses. Each pair of requests and responses is confined within the same connection on both ends.
    
* **Connection Termination (close)**: In this final step, both the client and the server mutual[l](https://github.com/DataDog/ebpf-training/tree/main/workshop1)y decide to terminate the established connection.
    

This structured process ensures the streamlined monitoring and management of HTTP traffic[,](https://github.com/DataDog/ebpf-training/tree/main/workshop1) providing crucial insights and enhancing network performance

![System call flow of an http request](https://cdn.hashnode.com/res/hashnode/image/upload/v1696251602829/05e47952-bc2c-4cbf-ae6a-2a95ab5290b1.png?auto=compress,format&format=webp align="left")

**System call flow for an HTTP request**

## **Encountering Hurdles: The Challenge of Connection Pooling**

Nonetheless, I stumbled upon an issue. When connection pooling was active, capturing traffic data for every individual request became unattainable. This situation presented a substantial barrier to my work. My objective was to meticulously document each request and response with every API interaction. This proved to be a considerable challenge in my experience with their ingress traffic controller.

### **The Close System Call Dilemma**

I found that with connection pooling, the close system call isn't executed for every request. In the existing codebase, the `close` system call's function is to confirm the total bytes read and written on a specific connection file descriptor on the user side. Consequently, eBPF dispatches an event encapsulating these details.

### **Kernel Data Events: Reading and Writing**

Each time a `read` or `write` system call is made, the kernel sends data events carrying the current bytes written or read. The total bytes read and written are accumulated for each individual read/write system call. This total is then checked against the total bytes `read`/`write` transmitted by the `close` event. After validation, the connection is released, and subsequent requests/responses employ different connection information.

```go
type Tracker struct {
    connID structs2.ConnID

    addr          structs2.SockAddrIn
    openTimestamp uint64
    closeTimestamp    uint64
    totalWrittenBytes uint64
    totalReadBytes    uint64

    // Indicates the tracker stopped tracking due to closing the session.
    lastActivityTimestamp uint64
    sentBytes             uint64
    recvBytes             uint64

    recvBuf []byte
    sentBuf []byte
    mutex   sync.RWMutex
}

// This function verifies whether the request/response was complete or not
func (conn *Tracker) IsComplete() bool {
    conn.mutex.RLock()
    defer conn.mutex.RUnlock()
    return conn.closeTimestamp != 0 &&
        conn.totalReadBytes == conn.recvBytes &&
        conn.totalWrittenBytes == conn.sentBytes
}

// This function gathers data events that include information about 
// the size of data buffers and the actual data content for each 
// read/write system call in both the request and response.
func (conn *Tracker) AddDataEvent(event structs2.SocketDataEvent) {
    conn.mutex.Lock()
    defer conn.mutex.Unlock()
    conn.updateTimestamps()

    switch event.Attr.Direction {
    case structs2.EgressTraffic:
        conn.sentBuf = append(conn.sentBuf, event.Msg[:event.Attr.MsgSize]...)
        conn.sentBytes += uint64(event.Attr.MsgSize)
    case structs2.IngressTraffic:
        conn.recvBuf = append(conn.recvBuf, event.Msg[:event.Attr.MsgSize]...)
        conn.recvBytes += uint64(event.Attr.MsgSize)
    default:
    }
}

// This function collects the total bytes read & written on a single connection for an api request.
func (conn *Tracker) AddCloseEvent(event structs2.SocketCloseEvent) {
    conn.mutex.Lock()
    defer conn.mutex.Unlock()
    conn.updateTimestamps()
    if conn.closeTimestamp != 0 && conn.closeTimestamp != event.TimestampNano {
        log.Printf("changed close info timestamp from %v to %v", conn.closeTimestamp, event.TimestampNano)
    }
    conn.closeTimestamp = event.TimestampNano

    conn.totalWrittenBytes = uint64(event.WrittenBytes)
    conn.totalReadBytes = uint64(event.ReadBytes)
}
```

Refer to [this](https://github.com/gouravkrosx/http-ingress-ebpf/blob/bff9eb3d1056d2b3a1da02274d9e0169011b85b5/connections/tracker.go#L35) for the above snippet.

### **The Connection Pooling Hiccup**

However, the hiccup with connection pooling is that the first request/response data is aggregated, but no `close` event is present to affirm it. As the second request utilizes the same connection and the data is once again summed up, this cycle continues until the server terminates the connection and then the `close` event is invoked. This method, thus, falls short in validating each request and response, resulting in logging inconsistencies.

## **Exploring a potential Solution**

After searching through GitHub, I discovered a repository named [**grafana/beyla**](https://github.com/grafana/beyla), which appeared to address connection pooling to a certain extent. Upon examining the Beyla codebase, I found that it doesn’t send the entire HTTP request/response buffer that includes both headers and body. Additionally, it doesn’t verify if the amount of data sent to the user space is accurate. However, for my particular needs, ensuring the completeness of the request and response data from the kernel space is essential.

## **Diving Into the Technical Details**

Before we delve into the solution, let’s highlight the technical tools used. It's notable that while Datadog opted for [**iovisor/bcc**](https://github.com/iovisor/bcc), my codebase made use of [**cilium/ebpf**](https://github.com/cilium/ebpf) , which is closely related to [**libbpf.**](https://github.com/libbpf/libbpf) This choice was crucial for making sure the project was strong and adaptable to my needs.

Wondering about the differences between `libbpf` and `bcc`? Get more insight [**here**](https://devops.com/libbpf-vs-bcc-for-bpf-development/) to understand the careful consideration behind choosing `cilium/ebpf`.

## **Alright, Enough Talk! Let's Solve It!**

![meme](https://cdn.hashnode.com/res/hashnode/image/upload/v1696255781073/ca096d53-58fc-4f3b-a8c1-47c6268d4df5.gif?auto=format,compress&gif-q=60&format=webm align="center")

Okay, no more blah blah. Let’s jump into how I actually sorted this out. The trick I used was pretty straightforward. I grabbed all the requests and responses on the same connection and lined them up in a queue.

Now, here’s the simple part. I noticed that by just skipping over the first request, the first response data event could share the total bytes read for the current API request, as it's already counted on the eBPF side after sending all the request data events. And the second request on the same connection could tell the total bytes sent of the previous response on that connection, thanks to it being already added up in the connection info struct.

With this easy method, I could check the total bytes read and written on that connection for each request. But here’s a tiny snag: for the last response, it's a guessing game to know if it was the complete response or not. But for my needs, that was fine. I just assumed that a few seconds after the last activity, all my response events would have made it to the user space.

And guess what? This method is thread-safe too! I made sure to add a read-write mutex before operations in the functions to keep things running smoothly. This approach worked like a charm, whether connection pooling was on or off. And there you have it, problem solved!

### **Glimpses into my kernel and user space**

**Kernel Space:**

```c

// A struct representing a unique ID that is composed of the pid, the file
// descriptor and the creation time of the struct.
struct conn_id_t
{
    // Process ID
    u32 pid;
    // The file descriptor to the opened network connection.
    s32 fd;
    // Timestamp at the initialization of the struct.
    u64 tsid;
};

// This struct contains information collected when a connection is established,
// via an accept4() syscall.
struct conn_info_t
{
    // Connection identifier.
    struct conn_id_t conn_id;

    // The number of bytes written/read on this connection.
    s64 wr_bytes;
    s64 rd_bytes;

    // A flag indicating we identified the connection as HTTP.
    bool is_http;
};

// Struct describing the data event being sent to the user mode.
struct socket_data_event_t
{

    // The timestamp when syscall completed (return probe was triggered).
    u64 timestamp_ns;

    // Connection identifier (PID, FD, etc.).
    struct conn_id_t conn_id;

    // The type of the actual data that the msg field encodes, which is used by the caller
    // to determine how to interpret the data.
    enum traffic_direction_t direction;

    // The size of the original message. We use this to truncate msg field to minimize the amount
    // of data being transferred.
    u32 msg_size;

    // A 0-based position number for this event on the connection, in terms of byte position.
    // The position is for the first byte of this message.
    u64 pos;

    // Actual buffer
    char msg[MAX_MSG_SIZE];

    //To verify the request data
    s64 validate_rd_bytes;

    //To verify the response data
    s64 validate_wr_bytes;
};
```

Refer to [this](https://github.com/gouravkrosx/http-ingress-ebpf/blob/bff9eb3d1056d2b3a1da02274d9e0169011b85b5/headers/k_structs.h#L103).

```c
static __inline void perf_submit_buf(struct pt_regs *ctx, const enum traffic_direction_t direction, char *buf, int buf_size, int offset, struct conn_info_t *conn_info, struct socket_data_event_t *event)

{

    bpf_printk("Direction of traffic of connection:%d is:%d", conn_info->conn_id.fd, direction);

    switch (direction)
    {
    case kEgress:
        event->pos = conn_info->wr_bytes + offset;
        // to validate request data when response comes.
        bpf_printk("Read Bytes:%d (current request)on connection:%d", conn_info->rd_bytes, conn_info->conn_id.fd);
        bpf_probe_read(&event->validate_rd_bytes, sizeof(event->validate_rd_bytes), &conn_info->rd_bytes);
        conn_info->rd_bytes = 0;
        break;
    case kIngress:
        event->pos = conn_info->rd_bytes + offset;
        // wr_bytes = 0, for the first request, but non-zero for the previous response.
        bpf_printk("Written Bytes:%d (previous response)on connection:%d", conn_info->wr_bytes, conn_info->conn_id.fd);
        bpf_probe_read(&event->validate_wr_bytes, sizeof(event->validate_wr_bytes), &conn_info->wr_bytes);
        conn_info->wr_bytes = 0;
        break;
    }

    // 16384 bytes
    // 0x3fff is hexadecimal number of 16383 (all ones in binary)
    // Took 16383 ~ 16KiB, because doing bitwise AND with this number will give the same number hence no chance of decrement in the buf_size.
    asm volatile("%[buf_size] &= 0x3fff;\n" ::[buf_size] "+r"(buf_size)
                 :);
    bpf_probe_read(&event->msg, buf_size & 0x3fff, buf);

    if (buf_size > 0)
    {
        bpf_probe_read(&event->msg_size, sizeof(event->msg_size), &buf_size);
        bpf_printk("Submitting the data event with buffer size:%lu to the userspace", event->msg_size);
        bpf_ringbuf_output(&socket_data_events, event, sizeof(*event), 0);
    }
}

static inline __attribute__((__always_inline__)) void process_data(struct pt_regs *ctx, u64 id, enum traffic_direction_t direction, const struct data_args_t *args, int bytes_count)
{
    ...........

    // Update the conn_info total written/read bytes.
    switch (direction)
    {
    case kEgress:
        conn_info->wr_bytes += bytes_count;
        break;
    case kIngress:
        conn_info->rd_bytes += bytes_count;
        break;
    }
}
```

Refer [1](https://github.com/gouravkrosx/http-ingress-ebpf/blob/bff9eb3d1056d2b3a1da02274d9e0169011b85b5/ebpf/keploy_ebpf.c#L252), [2](https://github.com/gouravkrosx/http-ingress-ebpf/blob/bff9eb3d1056d2b3a1da02274d9e0169011b85b5/headers/k_helpers.h#L206) & [3](https://github.com/gouravkrosx/http-ingress-ebpf/blob/bff9eb3d1056d2b3a1da02274d9e0169011b85b5/headers/k_helpers.h#L142)

**Userspace:**

```go
type Tracker struct {
    connID         structs2.ConnID
    addr           structs2.SockAddrIn
    openTimestamp  uint64
    closeTimestamp uint64

    // Indicates the tracker stopped tracking due to closing the session.
    lastActivityTimestamp uint64

    // Queues to handle multiple ingress traffic on the same connection (keep-alive)
    totalSentBytesQ   []uint64
    totalRecvBytesQ   []uint64
    currentSentBytesQ []uint64
    currentRecvBytesQ []uint64
    currentSentBufQ   [][]byte
    currentRecvBufQ   [][]byte

    // Individual parameters to store current request and response data
    sentBytes uint64
    recvBytes uint64
    SentBuf   []byte
    RecvBuf   []byte

    // Additional fields to know when to capture request or response info
    receivedResponse bool
    receivedRequest  bool
    recTestCounter   int32 //atomic counter
    firstRequest     bool

    mutex  sync.RWMutex
    logger *zap.Logger
}

// IsComplete() checks if the current connection has valid request & response info to capture
// and also returns the request and response data buffer.
func (conn *Tracker) IsComplete() (bool, []byte, []byte) {
    conn.mutex.Lock()
    defer conn.mutex.Unlock()

    // Get the current timestamp in nanoseconds.
    currentTimestamp := uint64(time.Now().UnixNano())

    // Calculate the time elapsed since the last activity in nanoseconds.
    elapsedTime := currentTimestamp - conn.lastActivityTimestamp

    //Caveat: Added a timeout of 4 seconds, after this duration we assume that the last response data event would have come.
    // This will ensure that we capture the requests responses where Connection:keep-alive is enabled.

    recordTraffic := false

    requestBuf, responseBuf := []byte{}, []byte{}

    //if recTestCounter > 0, it means that we have num(recTestCounter) of request and response present in the queues to record.
    if conn.recTestCounter > 0 {
        if (len(conn.currentRecvBytesQ) > 0 && len(conn.totalRecvBytesQ) > 0) &&
            (len(conn.currentSentBytesQ) > 0 && len(conn.totalSentBytesQ) > 0) {
            validReq, validRes := false, false

            expectedRecvBytes := conn.currentRecvBytesQ[0]
            actualRecvBytes := conn.totalRecvBytesQ[0]

            //popping out the current request info
            conn.currentRecvBytesQ = conn.currentRecvBytesQ[1:]
            conn.totalRecvBytesQ = conn.totalRecvBytesQ[1:]

            if conn.verifyRequestData(expectedRecvBytes, actualRecvBytes) {
                validReq = true
            } else {
                conn.logger.Debug("Malformed request", zap.Any("ExpectedRecvBytes", expectedRecvBytes), zap.Any("ActualRecvBytes", actualRecvBytes))
                recordTraffic = false
            }

            expectedSentBytes := conn.currentSentBytesQ[0]
            actualSentBytes := conn.totalSentBytesQ[0]

            //popping out the current response info
            conn.currentSentBytesQ = conn.currentSentBytesQ[1:]
            conn.totalSentBytesQ = conn.totalSentBytesQ[1:]

            if conn.verifyResponseData(expectedSentBytes, actualSentBytes) {
                validRes = true
            } else {
                conn.logger.Debug("Malformed response", zap.Any("ExpectedSentBytes", expectedSentBytes), zap.Any("ActualSentBytes", actualSentBytes))
                recordTraffic = false
            }

            if len(conn.currentRecvBufQ) > 0 && len(conn.currentSentBufQ) > 0 { //validated request, response
                requestBuf = conn.currentRecvBufQ[0]
                responseBuf = conn.currentSentBufQ[0]

                //popping out the current request & response data
                conn.currentRecvBufQ = conn.currentRecvBufQ[1:]
                conn.currentSentBufQ = conn.currentSentBufQ[1:]
            } else {
                conn.logger.Debug("no data buffer for request or response", zap.Any("Length of RecvBufQueue", len(conn.currentRecvBufQ)), zap.Any("Length of SentBufQueue", len(conn.currentSentBufQ)))
                recordTraffic = false
            }

            recordTraffic = validReq && validRes
        } else {
            conn.logger.Error("malformed request or response")
            recordTraffic = false
        }

        conn.logger.Debug(fmt.Sprintf("recording traffic after verifying the request and reponse data:%v", recordTraffic))

        // // decrease the recTestCounter
        conn.decRecordTestCount()
        conn.logger.Debug("verified recording", zap.Any("recordTraffic", recordTraffic))
    } else if conn.receivedResponse && elapsedTime >= uint64(time.Second*4) { // Check if 4 second has passed since the last activity.
        conn.logger.Debug("might be last request on the connection")

        if len(conn.currentRecvBytesQ) > 0 && len(conn.totalRecvBytesQ) > 0 {

            expectedRecvBytes := conn.currentRecvBytesQ[0]
            actualRecvBytes := conn.totalRecvBytesQ[0]

            //popping out the current request info
            conn.currentRecvBytesQ = conn.currentRecvBytesQ[1:]
            conn.totalRecvBytesQ = conn.totalRecvBytesQ[1:]

            if conn.verifyRequestData(expectedRecvBytes, actualRecvBytes) {
                recordTraffic = true
            } else {
                conn.logger.Debug("Malformed request", zap.Any("ExpectedRecvBytes", expectedRecvBytes), zap.Any("ActualRecvBytes", actualRecvBytes))
                recordTraffic = false
            }

            if len(conn.currentRecvBufQ) > 0 { //validated request, invalided response
                requestBuf = conn.currentRecvBufQ[0]
                //popping out the current request data
                conn.currentRecvBufQ = conn.currentRecvBufQ[1:]

                responseBuf = conn.SentBuf
            } else {
                conn.logger.Debug("no data buffer for request", zap.Any("Length of RecvBufQueue", len(conn.currentRecvBufQ)))
                recordTraffic = false
            }

        } else {
            conn.logger.Error("malformed request")
            recordTraffic = false
        }

        conn.logger.Debug(fmt.Sprintf("recording traffic after verifying the request data (but not response data):%v", recordTraffic))
        //treat immediate next request as first request (4 seconds after last activity)
        conn.resetConnection()

        conn.logger.Debug("unverified recording", zap.Any("recordTraffic", recordTraffic))
    }

    return recordTraffic, requestBuf, responseBuf
}

func (conn *Tracker) AddDataEvent(event structs2.SocketDataEvent) {
    .....
    .....
    conn.logger.Debug(fmt.Sprintf("Got a data event from eBPF, Direction:%v || current Event Size:%v || ConnectionID:%v\n", direction, event.MsgSize, event.ConnID))

    switch event.Direction {
    case structs2.EgressTraffic:
        // Assign the size of the message to the variable msgLengt
        msgLength := event.MsgSize
        // If the size of the message exceeds the maximum allowed size,
        // set msgLength to the maximum allowed size instead
        if event.MsgSize > structs2.EventBodyMaxSize {
            msgLength = structs2.EventBodyMaxSize
        }
        // Append the message (up to msgLength) to the connection's sent buffer
        conn.SentBuf = append(conn.SentBuf, event.Msg[:msgLength]...)
        conn.sentBytes += uint64(event.MsgSize)

        //Handling multiple request on same connection to support connection:keep-alive
        if conn.firstRequest || conn.receivedRequest {
            conn.currentRecvBytesQ = append(conn.currentRecvBytesQ, conn.recvBytes)
            conn.recvBytes = 0

            conn.currentRecvBufQ = append(conn.currentRecvBufQ, conn.RecvBuf)
            conn.RecvBuf = []byte{}

            conn.receivedRequest = false
            conn.receivedResponse = true

            conn.totalRecvBytesQ = append(conn.totalRecvBytesQ, uint64(event.ValidateReadBytes))
            conn.firstRequest = false
        }

    case structs2.IngressTraffic:
        // Assign the size of the message to the variable msgLength
        msgLength := event.MsgSize
        // If the size of the message exceeds the maximum allowed size,
        // set msgLength to the maximum allowed size instead
        if event.MsgSize > structs2.EventBodyMaxSize {
            msgLength = structs2.EventBodyMaxSize
        }
        // Append the message (up to msgLength) to the connection's receive buffer
        conn.RecvBuf = append(conn.RecvBuf, event.Msg[:msgLength]...)
        conn.recvBytes += uint64(event.MsgSize)

        //Handling multiple request on same connection to support connection:keep-alive
        if !conn.firstRequest || conn.receivedResponse {
            conn.currentSentBytesQ = append(conn.currentSentBytesQ, conn.sentBytes)
            conn.sentBytes = 0

            conn.currentSentBufQ = append(conn.currentSentBufQ, conn.SentBuf)
            conn.SentBuf = []byte{}

            conn.receivedRequest = true
            conn.receivedResponse = false

            conn.totalSentBytesQ = append(conn.totalSentBytesQ, uint64(event.ValidateWrittenBytes))

            //Record a test case for the current request/
            conn.incRecordTestCount()
        }

    default:
    }
}
```

Refer to [this](https://github.com/gouravkrosx/http-ingress-ebpf/blob/bff9eb3d1056d2b3a1da02274d9e0169011b85b5/connections/tracker.go#L274) for the above snippet.

## **The Vital Need to Capture Every Request and Response**

In my current role as a core contributor at [**Keploy**](https://github.com/keploy/keploy), the necessity to capture every request and response was paramount. Ensuring that no data was missed for every individual API request was crucial in enhancing Keploy's support system.

[**Keploy**](https://github.com/keploy/keploy) stands as an open-source, user-friendly backend testing tool crafted with developers in mind. It not only simplifies backend testing for engineering teams but is also robust and easily extendable.

By recording API calls and database queries, Keploy efficiently generates test cases and data mocks/stubs from user traffic. This process markedly accelerates releases while boosting reliability, ensuring that your testing phase is both comprehensive and efficient.

Sample Test case generated using **Keploy** by capturi[ng req](https://github.com/keploy/keploy)uest and response metrics:

```yaml
version: api.keploy.io/v1beta2
kind: Http
name: test-1
spec:
    metadata: {}
    req:
        method: POST
        proto_major: 1
        proto_minor: 1
        url: http://localhost:8082/url
        header:
            Accept: '*/*'
            Content-Length: "33"
            Content-Type: application/json
            Host: localhost:8082
            User-Agent: curl/7.81.0
        body: |-
            {
              "url": "https://google.com"
            }
        body_type: ""
    resp:
        status_code: 200
        header:
            Content-Length: "66"
            Content-Type: application/json; charset=UTF-8
            Date: Wed, 27 Sep 2023 02:17:20 GMT
        body: |
            {"ts":1695781040077972081,"url":"http://localhost:8082/Lhr4BWAi"}
        body_type: ""
        status_message: ""
        proto_major: 0
        proto_minor: 0
    objects: []
    assertions:
        noise:
            - header.Date
    created: 1695781040
```

## **Conclusion**

And there we have it, folks! From the initial hurdles with connection pooling and capturing every request and response to exploring solutions and implementing a thread-safe method that worked whether connection pooling was enabled or not, the journey has been both enlightening and productive. It is through these challenges and their subsequent solutions that tools like Keploy can continue to grow and serve the developer community more efficiently.

Remember, the journey doesn’t end here. Continuous improvement is the key to innovation. I'm open to any suggestions or improvements that could be made to enhance this process further. Your insights are not just welcomed; they are eagerly anticipated!

Feel free to check out the work and share your thoughts or suggestions on my [**GitHub Repository**](https://github.com/gouravkrosx/http-ingress-ebpf/tree/master). Let’s keep the conversation going and work together to make technology even more robust and reliable.

For further reading and reference, you can visit the following links:

1. [**DataDog eBPF Training**](https://github.com/DataDog/ebpf-training/tree/main/workshop1)
    
2. [**Grafana Beyla Repository**](https://github.com/grafana/beyla)
    

Thank you for joining me on this journey! Your collaboration and insights are invaluable in continuing to enhance and develop these technological solutions.