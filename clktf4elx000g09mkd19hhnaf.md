---
title: "Automated End to End tests using Property Based Testing  | Part I"
seoTitle: "Automated E2E tests using Property Based Testing"
datePublished: Wed Aug 02 2023 07:40:11 GMT+0000 (Coordinated Universal Time)
cuid: clktf4elx000g09mkd19hhnaf
slug: automated-end-to-end-tests-using-property-based-testing-part-i
canonical: https://keploy.io/blog/technology/automated-end-to-end-tests-using-property-based-testing-part-i
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1689231142933/0e808a24-763a-49d4-b2da-06404cc7749e.jpeg
tags: tdd, apis, testing, test-automation, keploy

---

> " Engineers call them edge cases. I call them: what our users do "
> 
> \- Noah Sussman

Testing remains a crucial aspect of software development. Software engineers and quality assurance professionals make use of diverse techniques and methodologies to rigorously test applications. However, even with these measures in place, bugs can still emerge. In this modern era, where advanced technologies like chatGPT are surpassing human capabilities in the tech domain, achieving absolute perfection in software testing remains a complex task.

Do you still believe that unit tests are legit 🧐? Like the quote claims, do we test only what users do? Are manual regression testing and test suite creation still necessary for QA? Well, as we go further we will uncover the answers to these questions. Let's begin with the basics and progress towards developing a test automation platform that may potentially surpass the accuracy of AI (just kidding 😅 or maybe not 🤨).

### What is Software Testing..?

Software testing is a process aimed at assessing the correctness of software. It involves evaluating various attributes such as reliability, scalability, portability, reusability, and usability. There are many ways that an application can be tested.

![Software testing chart](https://cdn.hashnode.com/res/hashnode/image/upload/v1687878473218/848a856c-1304-49db-9afc-5dc387e483c5.png align="center")

Let's get a brief about a few important testing techniques

1. **Unit Testing -** involves the testing of each unit or an individual component of the software application like a condition, function etc.
    
2. **Integration Testing -** involves testing units or individual components of the software in a group. The focus of the integration testing level is to expose defects at the time of interaction between integrated components or units. Mainly done because different developers develop different components.
    
3. **System Testing -** To Test the end-to-end flow of an application as a user is known as System testing. In this, we navigate all the necessary modules of an application and check if the end features or the end business works fine. This is also called **end-to-end testing** where the testing environment is similar to the production environment.
    
4. **Acceptance Testing -** User acceptance testing (UAT) is a type of testing, which is done by the customer before accepting the final product. Generally, UAT is done by the customer (domain expert like product manager) for their satisfaction.
    

If u want to deep dive into the testing techniques mentioned in the image you can refer [here](https://www.javatpoint.com/software-testing-tutorial).

### Is End-to-End Testing Better than Other Testing Methods..?

In this blog, our primary focus is on testing in the development phase specifically before handing the software over to the QA/PM for UAT or other types of testing. Our discussion excludes testing methodologies like Load Testing, which fall outside the scope of this blog.

During the development phase, two major testing approaches come into play: Unit Testing and End-to-End Testing.

![kind of tests we need](https://cdn.hashnode.com/res/hashnode/image/upload/v1687548287710/5c691c51-0e02-4e72-a407-28b0d08dcd18.gif align="center")

Why do most developers prefer Unit tests over end-to-end tests?

1. **Fine-grained Control -** Firstly, unit testing is white box testing that is the developer writes test cases knowing the underlying code.
    
2. **Attention to Detail -** The test cases are written at the unit level so the developer experience the heightened sense that every minute detail is tested.
    
3. **Simplicity in Creation -** Unit tests are easier to write when compared to end-to-end.
    
4. **Feedback -** The unit test suite can be run every time developer makes any change in the code to check the correctness of the code.
    
5. **Coverage** - Unit tests results in line coverage and branch coverage which showcases whether a particular line of the code is tested or not.
    

Wait 🤔 Isn't it the ideal solution for developers for testing? Well, not quite, as unit tests do not directly interact with dependencies such as databases instead, developers create mock objects to simulate these dependencies, essentially instructing the test on how to respond. The primary drawback of unit testing is its limited scope, as it primarily focuses on individual units of code and may overlook the broader functionalities of the application. Unit testing primarily serves to enhance developer productivity, foster creativity, and enable instant validation of the code.

End-to-end testing involves simulating real user scenarios by sending requests to the application and asserting the responses against the expected results. During end-to-end testing the application is connected to a database similar to the production environment, creating a realistic end-user experience. This type of testing also covers integration testing, ensuring that different components of the application function seamlessly together. Since end-to-end testing focuses on the external behaviour of the application without considering the underlying code, it falls under the category of black-box testing.

Developers typically do not write extensive end-to-end tests due to their inherent complexity. Writing such tests involves documenting each request and response in a file and then executing the tests. As a result, the responsibility of conducting end-to-end testing is often assigned to the QA team, who possess the necessary expertise and resources to handle these comprehensive tests effectively.

But let's consider this scenario: What if end-to-end (E2E) tests were remarkably easy to write and maintain? Would this enable developers to thoroughly test their code during the development phase? Could it potentially replace the need for unit tests?

### What is Automation testing ..?

Automation testing involves the automated testing of software, wherein developers or testers create a test script using testing tools and frameworks. This script is executed on the software, allowing for testing without the need for human intervention. The automated test script performs various tests on the software and produces results automatically. A notable example of a widely used automation testing tool is Selenium.

Now that we have gained an understanding of the two main keywords in our title—Automation and End-to-End Testing—let's explore the concept of automating end-to-end test cases. While numerous tools are available in the market for automating end-to-end tests, what sets this approach apart? what's new here? One aspect that sets it apart is the automation of the initial iteration of tests. Traditionally, when using automation tools, the first iteration of tests (particularly when introducing a new feature) is performed manually. The validity of these tests relies heavily on the person responsible for writing them for the first time.

In this blog, the aim of automating end-to-end tests is to eliminate this manual effort even for the initial iteration. Part-I gives a good understanding of why are we making an effort to create Automated End to End tests using Property Based Testing. Check out the following blog, [**Part II**](https://blog.keploy.io/automated-e2e-tests-using-property-based-testing-part-ii), for a practical guide on building an Automated test platform 🥳