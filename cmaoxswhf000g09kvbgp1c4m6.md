---
title: "Maintaining Auto-Generative API Tests: Need of de-duplicate tests"
seoTitle: "De-Duplicating Auto-Generated API Tests"
seoDescription: "Discover why removing duplicate auto-generated API tests is key to better test maintenance, faster execution, and improved API testing efficiency."
datePublished: Thu May 15 2025 05:36:37 GMT+0000 (Coordinated Universal Time)
cuid: cmaoxswhf000g09kvbgp1c4m6
slug: maintaining-auto-generative-api-tests-need-of-de-duplicate-tests
canonical: https://keploy.io/blog/technology/maintaining-auto-generative-api-tests-need-of-de-duplicate-tests
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1747158277443/7975d7e2-7a8e-4f9a-ac9a-2ee26b3acb36.jpeg
tags: automated-testing, api-testing, automation-testing

---

## The Evolving Landscape of Auto-Generative Testing

In today's fast-paced development environment, automatically generating tests has become a necessity. Tools leveraging AI, manual interventions, and live environment captures facilitate this process. For instance, k6 focuses on load testing, generating automated tests with minimal maintenance. On the other hand, AI generative tools like GitHub Copilot, Bard, and ChatGPT can produce many tests, though often with redundancy.

While generative tests are easier to create, they can be challenging to maintain due to the lack of context. Over time, this can lead to a cumbersome and outdated test suite. Regular monitoring and maintenance are essential to ensure tests remain relevant and functional, whether used locally or in a CI pipeline. This necessitates accountability within the team to manage and update these tests effectively.

### The Challenge of Maintaining Auto-Generative API Tests

Maintaining API tests requires keeping them up-to-date with the latest changes, ensuring they cover all relevant scenarios, and avoiding redundancy. Here are some key strategies to achieve this:

#### 1\. Modular Test Design

* **Separation of Concerns:** Break down tests into smaller, independent modules focusing on specific aspects of the API, making it easier to update individual parts without affecting the entire suite.
    
* **Reusable Components:** Create reusable components or functions for common operations like authentication, data setup, and teardown processes.
    

#### 2\. Consistent and Descriptive Naming Conventions

* Use clear and consistent naming conventions for test cases, variables, and functions to improve readability and make it easier to update tests when the API changes.
    

#### 3\. Version Control Integration

* **Track Changes:** Use version control systems like Git to track changes in both the API and the test scripts, helping understand what changed, when, and why.
    
* **Branching Strategy:** Implement a branching strategy to manage different API versions and corresponding tests, maintaining separate branches for development, testing, and production.
    

#### 4\. Continuous Integration/Continuous Deployment (CI/CD)

* **Automated Testing:** Integrate the API tests into a CI/CD pipeline to automatically run tests whenever changes are made to the API or the tests themselves. Tools like Jenkins, Travis CI, or GitHub Actions can be useful.
    
* **Feedback Loop:** Ensure a fast feedback loop to quickly identify and address issues.
    

#### 5\. [Test Data Management](https://keploy.io/blog/community/7-best-test-data-management-tools-in-2024)

* **Dynamic Data Generation:** Use dynamic data generation techniques to create test data on the fly, ensuring tests remain relevant and up-to-date.
    
* **Data Isolation:** Ensure tests do not interfere with each other by isolating test data. Use unique identifiers or clean up data after each test run.
    

#### 6\. Comprehensive Documentation

* Document the structure and logic of the test suite, including instructions on how to run and update tests and guidelines on adding new tests.
    
* Maintain up-to-date API documentation to help understand the API’s functionality and expected behavior.
    

#### 7\. Monitoring and Alerts

* **Health Checks:** Regularly monitor the health of the API and the tests, setting up alerts to notify the team when tests fail or significant API behavior changes occur.
    
* **Performance Metrics:** Track performance metrics to ensure tests run efficiently without introducing significant overhead.
    

#### 8\. Regular Review and Refactoring

* **Code Reviews:** Conduct regular code reviews to ensure tests adhere to best practices and remain efficient.
    
* **Refactoring:** Periodically refactor the test code to improve its structure, readability, and maintainability.
    

#### 9\. Backward Compatibility Testing

* Maintain tests for different API versions to ensure backward compatibility, crucial for APIs consumed by multiple clients who may not update simultaneously.
    

#### 10\. Mocking and Stubbing

* Use mocking and stubbing to simulate external dependencies and isolate the API tests, helping to test the API in a controlled environment and making the tests more reliable.
    

### The Need for Marking Duplicate Tests

Consider a scenario where your API development team has used various tools to generate hundreds of tests automatically. As development progresses, the number of tests grows, reaching a point where you have 500 tests. However, many of these tests may be redundant, covering the same code paths and scenarios multiple times. Maintaining such a large and redundant test suite can be overwhelming, leading to inefficiencies and increased maintenance costs.

Technically, it's better to have 50 highly relevant and unique tests that comprehensively cover different code paths than to maintain 500 tests with considerable overlap. Redundant tests can inflate test execution times, complicate debugging, and make it challenging to identify critical issues. Therefore, deduplicating tests to focus on those that add the most value is crucial.

### Keploy: Simplifying Test Generation and Maintenance

Keploy is a powerful tool that not only creates auto-generated tests but also provides various ways to maintain them. Unlike tools that solely rely on AI for test generation, Keploy combines AI test generation with a basic record and replay model. Users can create test cases by making API calls or through live interactions. Additionally, Keploy offers a feature to deduplicate redundant tests based on code paths.

#### [How Keploy Works](https://keploy.io/docs/keploy-explained/how-keploy-works/) ??

Keploy helps create end-to-end tests and realistic data mocks via API calls, with the ability to replay them to check for any breaks due to API changes or bugs. However, covering all scenarios through API calls alone can be challenging. In real-world scenarios, numerous tests may be created using scripts or Keploy's test generation feature, which uses the OpenAPI schema to generate tests. This leads to the need to mark the relevant tests.

#### Deduplication in Keploy

To identify relevant tests, it's crucial to mark duplicate tests. Unique test cases are the most suitable. The challenge lies in determining the uniqueness of tests. While custom logic can compare request parameters like URLs and request bodies, this approach may not be entirely accurate. Even applying AI models to deduplicate tests can be helpful, but without a quantitative benchmark, it remains challenging.

#### Coverage-Based Deduplication

What about coverage? Any test, whether an API test, integration test, or unit test, will cover certain parts of the code. If we can extract the number of lines covered by each test, we can deduplicate based on that. This approach ensures that tests are unique and cover different parts of the codebase, enhancing the overall effectiveness of the test suite.

Keploy's deduplication works on this principle, calculating the best test cases covering most of the codebase. Since Keploy operates as a CLI tool, users only need to use the deduplication command:

```bash
keploy dedup -c "<CMD_TO_RUN_APP>"
```

To obtain the deduplication data, users must add a simple Keploy SDK to their applications. Since Keploy tests are API tests, importing a Keploy middleware will help intercept the coverage data dynamically after each test execution.

Install the `keploy/sdk` and `nyc` package : -

```bash
npm i @keploy/sdk nyc@15.0.0
```

Add the the following on top of your main application js file (index.js/server.js/app.js/main.js) : -

```javascript
const kmiddleware = require('@keploy/sdk/dist/v2/dedup/middleware.js')

app.use(kmiddleware())
```

or If you are using Java we will be needing the JacocoAgent for getting the coverage and keploy-sdk which has added hooks to extract coverage data at runtime and reset it after each n every request.

Put the latest keploy-sdk in your pom.xml file : -

```xml
<dependency>
    <groupId>io.keploy</groupId>
    <artifactId>keploy-sdk</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```

Now that we have added keploy-sdk, let's import it in our main class : -

```java
import io.keploy.servlet.KeployMiddleware;

@Import(KeployMiddleware.class)
public class SamplesJavaApplication {
    public static void main(String[] args) {
        SpringApplication.run(SamplesJavaApplication.class, args);
    }
}
```

After you run the deduplication command stated above, You can see a *dedupData.yaml* being created on the root directory of the your project folder. This will look something like this. Each test is mapped with lines covered with individual files.

```yaml
- id: test-1
  executedLinesByFile:
    /Users/keploy/samples-typescript/wednesday/server/utils/index.js:
      - 6
      - 7
      - 8
      - 10
    /Users/keploy/samples-typescript/wednesday/server/services/slack.js: []
    /Users/keploy/samples-typescript/wednesday/server/services/circuitbreaker.js:
      - 5
      - 7
      - 9
      - 10
    /Users/keploy/samples-typescript/wednesday/server/middleware/gqlAuth/index.js:
      - 11
      - 13
      - 21
    /Users/keploy/samples-typescript/wednesday/server/utils/constants.js:
      - 1
      - 2
      - 8
      - 13
    /Users/keploy/samples-typescript/wednesday/server/database/models/index.js:
      - 5
      - 7
      - 9
      - 11
      - 12
      - 13
    /Users/keploy/samples-typescript/wednesday/server/database/models/products.js:
      - 2
      - 41
      - 47
    /Users/keploy/samples-typescript/wednesday/server/database/models/stores.js:
      - 2
      - 42
- id: test-2
  executedLinesByFile:
    /Users/keploy/samples-typescript/wednesday/server/utils/index.js:
      - 8
      - 29
      - 30
      - 32
    /Users/keploy/samples-typescript/wednesday/server/services/slack.js:
      - 7
      - 8
    /Users/keploy/samples-typescript/wednesday/server/services/circuitbreaker.js:
      - 12
      - 14
    /Users/keploy/samples-typescript/wednesday/server/utils/gqlSchemaParsers.js:
      - 9
      - 12
      - 14
      - 28
    /Users/keploy/samples-typescript/wednesday/server/database/models/products.js:
      - 47
      - 52
      - 61
      - 67
      - 73
```

So since we have got the data we can proceed towards the processing part. Which will be common for all languages. After getting the coverage data for each request Keploy uses a hashing algorithm. In which it takes each file path and encode them into smaller keys. Also their values which are list of line numbers are stored in a sorted set which can be further made into uniques keys by appending a delimiter between them. At last after iterating the whole dedup object and maintaining a structured map which uniques keys. We will be able to get the exact duplicates i.e the tests which covering the exact number of lines. Mostly 90% duplicates are removed in this step only. After that we will see the overlapping tests or those tests which are superset of the other two tests. For example if test-3 covers line *4,5,6,8,9* of main.go and and test-4 contains *4,5,6,8,9,11,14.* We'll consider the test-4 as more relevant and discard test-3. There can be other optimisations as well a custom computetations but the core idea is to get the coverage of each test. And discard those tests which are not adding any extra coverage than the previous calculated coverage. This can be a useful when specially you are capturing the tests from any live environment /production.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1720864020513/7cefddad-b5cc-4fdc-8d7b-c281429a3965.png align="center")

After a successful deduplication session, You will be able to see the logs telling that how many duplicates are found by keploy. You can also see duplicates.yaml file created after the keploy session. Since tests are in the yaml format, you can verify if the results displayed by the keploy fits in your case. If you explicitly want to keep a testcase. You can remove it from the list of duplicates in the yaml. At last if you want your test suite to be free of the duplicates. You can just run the following command -

```bash
keploy dedup --rm "test-set-1"
```

This will remove the duplicate tests, resulting in a test suite which can be easily manage and have the same amount of coverage%, which your bulky generated tests had.

### Conclusion

Maintaining auto-generative API tests is critical for ensuring robust and reliable APIs. By following the outlined strategies and leveraging tools like Keploy, you can simplify the maintenance process, avoid redundancy, and ensure comprehensive test coverage. Embrace these practices to enhance your API testing and maintenance workflow.

## Faq’s

### **1\. What are auto-generative API tests?**

Auto-generative API tests are created automatically from real API traffic. They mimic real user behavior, capturing inputs, outputs, and dependencies. This approach saves time and improves test coverage.  
However, it can also lead to large volumes of similar or redundant tests.

## **2\. Why do auto-generated tests need de-duplication?**

Without de-duplication, tests may repeat the same scenarios with minor variations. This bloats your test suite and increases execution time. It can also make debugging failures harder due to noise. De-duplicating keeps the suite lean, relevant, and efficient.

### **3\. How do duplicate API tests impact CI/CD performance?**

Duplicate tests slow down your CI/CD pipeline by increasing test duration. They consume unnecessary compute resources and increase costs. More tests also mean more logs and harder analysis.  
Efficient test sets improve both speed and clarity in results.

### **4\. How can you identify duplicate tests in auto-generated suites?**

Use clustering algorithms or hashing to compare request/response patterns. Keploy and similar tools can flag or merge near-identical test cases. Tagging and test metadata help track variations. Regular reviews keep the test suite optimized over time.

### **5\. What best practices help maintain clean auto-generated API tests?**

Schedule periodic test audits to prune or merge redundant tests. Use tagging, version control, and naming conventions for easy management. Integrate de-duplication logic into your test generation tools.  
Focus on covering edge cases instead of repeating happy paths.