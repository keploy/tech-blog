---
title: "Why Traditional API Testing Fails? Comparing Shadow, Production, Replay Techniques"
seoTitle: "Why Traditional API Testing Fails? Shadow, Prod Replay Tests"
seoDescription: "We had 15-day sprints including mandatory unit and API testing.

Strict Timelines (2 weeks)

80% unit test and API test mandate"
datePublished: Sat May 25 2024 19:15:31 GMT+0000 (Coordinated Universal Time)
cuid: clwmhqm6b000509la61o8b7z7
slug: why-traditional-api-testing-fails-comparing-shadow-production-replay-techniques
canonical: https://keploy.io/blog/technology/why-traditional-api-testing-fails-comparing-shadow-production-replay-techniques
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716663017299/3dfc78aa-f04a-4eab-8d69-95381390eed5.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1716702415685/746e29c7-e434-42ce-9889-03cbf6af8e09.png
tags: apis, software-engineering, software-testing, api-testing, api-testing-tools, ai-tools

---

I want to share the story of how our team at a fast-paced startup tackled the challenge of API testing without any dedicated QA team, the roadblocks that we encountered, and how we ultimately addressed these issues.

### **Baseline Challenge**

We had 15-day sprints including mandatory unit and API testing.

* Strict Timelines (2 weeks)
    
* 80% unit test and API test mandate
    

Initially, we relied on automated testing. Unit tests were stable, but API scripts quickly outdated due to frequent data changes and complex scenarios. Lacking a clear evaluation metric like code-coverage in unit tests, we wanted to cover as many user-scenarios possible for APIs. I suggested the record-replay technique for creating disposable API tests. In short, for API tests I wanted:

* Close to real-world API tests (ideally from prod)
    
* Easy disposable tests - quickly create/discard
    
* Automatic Infrastructure spin-up or service/dependency virtualization..
    

Writing test automation script was not an option since we got only 15 days to ship.

![](https://lh7-us.googleusercontent.com/Uf4BcOauEpDPhuIVxefT127Wr9r8pW_J1XqNyEII2e-f0DX2Wn6zhx7PUNYN_1dqENXmG-61YCerGFSt0gC9G2FOwwt-u8cvK4xIOdfWwp5YS9J1K44XSdyl-_eYp3GKvPWRm8kP1B8jWlfs1ow6BR3fSQ=s2048 align="center")

### **Exploring Various Shadow Testing Approaches**

Since we wanted real + disposable test scenarios we thought of directly testing in production, we knew it was risky, but it promised the most realistic test environment..

![](https://lh7-us.googleusercontent.com/3gmYEBNfR96SzmRQtVtls8drnXnxsK30wa5GfATrRT6HDxBzDlqLHlWhAnxvhWB6KBrMMco991kP5imWww5KcYJict0RAzHsOjQ___NtxDX4d7QVOW89vxHoVTVCgJRMXN9T-wAFcdsf9kVLtlVMcXZ3rQ=s2048 align="left")

**Pros** that we saw with testing in production were:

* Low code approach
    
* Easy to achieve high coverage
    
* Discovering unexpected user flows
    

We started discussions to **mirroring live traffic to new versions** of our application to **compare the response**s.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716656828862/37a77ede-795a-4032-b74a-697ef5044876.png align="center")

This method was **useful for stateless applications** but **failed to address** the complexities of our **stateful application,** which involved multiple interactions with databases and other services.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716657161882/60182546-c505-42f1-bad5-42c0ba4065e5.png align="center")

As seen below we **rejected the idea in the early discussions**, majorly because of the risks involved - all APIs were not idempotent so prod data will be altered if we duplicate traffic, which can lead to downtime or potential data leak or loss.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716657201454/051b6a8b-e2bf-4676-9e58-baed09b88571.png align="center")

Meanwhile we also discussed **replaying just the READ** operations **by filtering out the WRITE** calls with help of a **proxy**, but that's not a full proof solution since testing will not be completed. Not worth the effort...

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716657387063/6620004a-dd78-49fc-8ca4-20a0b6f53605.png align="center")

We also thought of maintaining a DB Replica in sync but that was way too **expensive** and there was a **replication lag**, leading to additional problems.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716657748848/59705c5a-7561-483f-921b-00eec79da4c7.png align="center")

Our discussions came down to lower(**Non-Prod**) environment - **Database Replication and Traffic Replay -** we IMPLEMENTED THIS!

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716657887485/6e9b3278-799c-4a97-9b6b-90dd9fdd2445.png align="center")

We created a regularly-updating **snapshot of prod dB** and also tried replaying duplicated 10% of the traffic. While this method allowed for high-fidelity testing, it took a **big operational effort, cost of the test environment shot up.** We solved 2 of the requirements, however service-**infra state challenge** remained the same and tests were **still flaky**.

✅ Close to real-world API tests (ideally from prod)

✅ Easy disposable(quick create/discard) tests

📛 Automatic Infrastructure spin-up works for a simple service but it's not scalable(**Slow and Expensive**) where there are **multiple services.**

![One Eternity Later | SpongeBob Time Card #9](https://i.ytimg.com/vi/U7CZcd-UYmU/maxresdefault.jpg align="left")

**Final approach!** - If Database is the problem, get rid of the database!

**Record and Replay with Service Virtualisation -** Eventually, we realised that we **don't need** the complete **snapshot** and we can just record the **particular query being made for that API call**. This allowed us to simulate user behaviour accurately and conduct effective tests without the overhead of maintaining a complete database replica. Running our tests became **lightweight and cheap.**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716658559322/6a1afa27-1d4a-461e-82e8-11a1a738e9b5.png align="center")

### **Developing Keploy**

We did explore a combination of open-source tools built to make record-replay testing easier - [vcr](https://github.com/vcr/vcr), and [pact](https://github.com/pact-foundation), and and [wiremock](https://github.com/wiremock/wiremock), and a variety of other tried and true projects. There are all great tools for their intended use case but didn’t work for me, these are the shortfalls we encountered -

1. **VCR**: I explored multiple VCR libraries.
    
    1. Most require code changes to get integrated.
        
    2. Dont support DBs like MySQL, Postgres, Redis, etc.
        
    3. The call never went out of the applications and so stubbed out at the client interface level.
        
2. **Pact**:
    
    1. To use Pact, I need to first integrate the SDK on frontend to record the expected contracts.
        
    2. Deploy a broker that tracks these recorded contract recordings across environment deployments.
        
    3. Then run those tests on the backend, which may fail because I need to also ensure details like Auth are handled - which can again require lots of work to get it to work reliably.
        
    4. After all this plumbing and changing my code - The benefit is contract testing. My goal was to achieve more than just contracts - test behaviour.
        
3. **Wiremock**:
    
    1. Again no DBs.
        
    2. Correlating incoming calls to outgoing calls is left to the user.
        
    3. If I want to record/replay the calls, intercepting or directing calls is also left to the user. I wanted something which can achieve this transparently without application awareness and code changes.
        
4. \[Edit\] **Mountebank** - This one is new to me, somehow didn’t pop in my initial research. Got to know from this [reddit-comment](https://www.reddit.com/r/QualityAssurance/comments/1d0ju67/comment/l5ovs8w/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button). Seems to be similar to wire mock though:
    
    1. Again no DBs, although there seems to be some provisions to add custom protocols.
        
    2. Transparent record and replay not possible. This problem becomes exponentially harder when trying to do for DBs.
        
    3. Incoming and outgoing correlation is left to the user.
        

Since we wanted to record and replay everything (DB, internal/external APIs) at the wire level without code changes - we started writing a different implementation. From these experiences, [**Keploy open source**](https://github.com/keploy/keploy) project was born. Idea was to capture API requests and database responses during normal operation and creates test cases that can be replayed in any environment including CI/CD.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716659237074/5f78acc3-2420-48e5-acfb-74ee6ccb9e3d.png align="center")

### **Continuous Improvement**

We continue to refine [**Keploy**](http://keploy.io), aiming to reduce the maintenance burden and increase its adaptability to changes in the application. Our goal is to make API testing as seamless and efficient as possible. One of the hidden advantage with this approach was to identify **performance issues.** Shadow testing has uncovered hard-to-detect performance issues, such as excessive network calls. For example, we did have usecases where we uncovered some very hard to performance issues. Eg: Recently one of users recorded the application locally via keploy and saw there were 3000+ Redis mocks for single request to the API. After deeper inspection it was discovered that there was aggregation logic in the application which fetches 3000 entries in separate network calls from the Redis server. This problem is very hard to find locally, because simple lookups are very fast in Redis, there is also no latency locally and Redis also does [pipelining](https://redis.io/docs/latest/develop/use/pipelining/). After identifying this, the user converted this to a single batch call. I'm not claiming this to be a performance monitor, but its been very useful to see all the network requests data that my application does to databases and other services to understand what data exchanges are actually happening and also how many :)

### **Limitations of Testing in Production**

Since our **use case** was to mainly use these as **regression tests** that are initially validated manually, and inform me about changes in application behavior and force me to normalize them if they are expected.

1. **Adopting production:** it might take some time to gain confidence to record from production, but recording API calls locally to create test and stubs also solved the purpose upto good extent. Infact! running these tests locally went quite well because the tests didn't require any infra spin-up and ran very fast (eg: 1000 test in ~1 min for a medium size app ).
    
2. **Effort Required to Develop Parsers for Mocking/Stubbing Out Dependencies:** was high. However, it was clear that there can be high confidence on the mocks, if:
    
    1. Not done at any library interface level but at the wire level.
        
    2. Transparent to the application and without any config/behiviour change to the application.
        
    3. Created from recordings of the real Dependency like DBs or other services. I have seen issues with mocks that are manually created and so may not be "realisitic".
        
    4. Update-able easily when the mocked system changes.
        
    5. Tested for flakiness
        
3. **Over reliance:** Developers might become overly reliant on shadow tests to detect regressions. This can lead to issues if the system fails to catch some regressions, requiring a recalibration of expectations. **Integration with Code Coverage Libraries** to minimize over-reliance has been explored, providing transparency about covered branches in addition to lines of code. Future integration with mutation coverage is also considered.
    
4. **Data Privacy:** Since shadow testing starts during development, identifying keys related to personally identifiable information (PIi) is crucial. These filters need to propagate to production to maintain privacy
    
5. **The Oracle Problem:** where determining the correctness of test results is challenging. Can be partially solved by finding duplicate test cases based on branch/line coverage. While it might filter some legitimate use-cases, it saves the users from reviewing 1000s of mostly redundant tests.
    
6. **Handling Test Noise:** Strategies to handle noise include running tests multiple times to identify inconsistent fields and using techniques like time freezing to prevent session token expiration. [https://keploy.io/docs/keploy-cloud/time-freezing/](https://keploy.io/docs/keploy-cloud/time-freezing/)
    

In my next blog I'll be writing about our journey of further challenges we faced with the SDK we built to record and replay APIs along with stubs and then moving on to the [<mark>eBPF approach</mark>](https://keploy.io/docs/keploy-explained/how-keploy-works/). The core requirement I had was, how can I iterate quickly with my applications when I make small code changes without having to invest in writing tests scripts and maintaining test data or env. Would love to hear more thoughts :)

### **Invitation to Collaborate**

If you find these challenges familiar or are interested in Keploy, we encourage you to join our community. We believe in solving problems together and are eager to share insights and improvements with developers facing similar issues.

This journey taught us the importance of adapting testing strategies to fit the development environment and the invaluable role of automation in maintaining software quality under tight deadlines. Thank you for reading, and we look forward to your feedback and collaboration in making API testing better for everyone.