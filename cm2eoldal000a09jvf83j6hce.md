---
title: "Migration Guide:  From RestAssured to Keploy"
seoTitle: "Migration Guide:  From RestAssured to Keploy"
seoDescription: "If you're tired of writing endless lines of repetitive code in RestAssured just to test your APIs, you're not alone."
datePublished: Fri Oct 18 2024 12:03:29 GMT+0000 (Coordinated Universal Time)
cuid: cm2eoldal000a09jvf83j6hce
slug: migration-guide-from-restassured-to-keploy
canonical: https://keploy.io/blog/technology/migration-guide-from-restassured-to-keploy
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1729252643948/c69ab866-4507-4fcd-ac33-c1d8fcfee0cd.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1729252680820/f34feb4d-e74b-4135-8e68-03b2f87464b1.png
tags: rest, keploy, rest-assured

---

If you're tired of writing endless lines of repetitive code in RestAssured just to test your APIs, you're not alone. API testing shouldn’t feel like pulling teeth, but let’s face it—REST Assured can make the process boring and unnecessarily time-consuming. But what if you could leave that grind behind?

In this guide, we’ll show you how to make the switch to Keploy, a smarter, zero-code way to test your APIs. Let’s make your API testing faster, easier, and, dare we say it, fun! Ready for the upgrade?

## REST Assured Overview

REST Assured is a popular Java library used for testing RESTful web services. It provides a domain-specific language (DSL) for writing tests, allowing developers to validate the responses from APIs effectively. With features like:

* Easy integration with testing frameworks like JUnit and TestNG.
    
* Support for various HTTP methods (GET, POST, PUT, DELETE).
    

However, as APIs grow more complex and testing demands increase, relying on REST Assured can become a real pain. Let’s face it:

* **Manual test writing** becomes repetitive and time-consuming.
    
* **Test maintenance** is a nightmare, especially when APIs evolve.
    
* **Coverage reporting** is not inbuilt and requires setup with libraries like Jacoco.
    
* **Complex setup** eats into development time, distracting engineers from core tasks.
    

This is where **Keploy** steps in. Keploy automates the testing process, reducing engineering effort by at least `20%` and letting your team focus on what matters—delivering high-quality software.

## Why Migrate to Keploy?

Keploy is an open-source tool designed to automate API testing by capturing API Interactions and replaying them later. Some of its key features include:

* **Autogenerate Data Mocking**: Keploy can automatically generate mocks based on interactions with various dependencies such as microservices and databases, reducing the need for manual mock creation.
    
* **Low-cost execution**: Keploy doesn’t require a dedicated, complex test environment setup. This leads to less overhead in managing parallel environments and reduces costs associated with infrastructure.
    
* **Zero-code Testing**: Unlike RestAssured, where developers need to write every test manually, Keploy offers a zero-code approach by capturing API interactions and generating tests automatically.
    
* **Easy Integration**: It integrates well with CI/CD pipelines and other testing tools such as JUnit, TestNG, GitHub Action, etc.
    
* **Comprehensive Test Coverage**: Since Keploy captures real-world API interactions, including edge cases, it helps ensure a broader and more realistic test coverage compared to the manually written tests in RestAssured.
    

## Steps to Migrate from REST Assured to Keploy

We will run a simple employee-manager application in Java with Postgres as a database for this guide.

### Step 1: Assess Your Current Test Suite

Before migrating, conduct a comprehensive assessment of your existing RestAssured test suite:

* **Identify Existing Test Cases**: Document all existing test cases and their functionality.
    
* **Note Dependencies**: Identify any dependencies that might affect the migration process.
    

Let’s run our test cases and check if everything is working fine or not

```java
mvn test
```

We will observe that all our test cases have passed, and since we have jacoco installed, we can also find out the coverage.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729161466678/70afd2f2-aec8-4259-8e26-693fa9082248.png align="center")

We have gotten around `68%` of coverage for our test suite.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729161452397/1efccfa8-7502-4e23-91e6-4bfd7c3e6111.png align="center")

Let’s move ahead to set up keploy and our migration process.

### Step 2: Set Up Keploy in Your Environment

1. **Install Keploy**: You can set up Keploy by following the installation instructions on the [Keploy GitHub repository](https://github.com/keploy/keploy).
    
2. You can verify the installation by running the command `Keploy` on the terminal, we should be able to see the below output:
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729155380281/15d7b5b3-0ec6-42ae-a2f6-4a8174559631.png align="center")
    

### Step 3: Migrate Test Cases

Let’s start the process of migrating our existing REST Assured test cases.

```java
    @BeforeEach
    public void setUp() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = 8080;

        // Clean up repository to ensure fresh data for each test
        employeeRepository.deleteAll();

        // Create and save test employee
        testEmployee = new Employee();
        testEmployee.setFirstName("John");
        testEmployee.setLastName("Doe");
        testEmployee.setEmail("john.doe@example.com");
        employeeRepository.save(testEmployee);  // save to generate ID
    }
```

Since our application runs locally on port `8080`, we have configured `RestAssured.port` to also run on `8080`, allowing Keploy to capture the API Interaction and create a new test suite when the REST Assured TestSuite is executed.

1. Let’s create a jar file for our application by running `mvn clean install -Dmaven.test.skip=true`.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729159214153/d8dcaab9-2154-4ed4-8c54-5fc63e90fe90.png align="center")
    
2. With the Jar file ready, let’s start keploy in record mode to capture the test cases. Now it's time to bring our database up and running using `docker-compose up postgres`: -
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729159392104/4cf442ef-2b84-4ea6-80b9-c12a7222534c.png align="center")
    
3. On the new terminal, let’s run `keploy record -c "java -jar target/springbootapp-0.0.1-SNAPSHOT.jar"`: -
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729159520013/33d7d889-65e5-430b-b438-0549fdd67480.png align="center")
    
4. Now, we have everything ready and set up to migrate our test suites. It’s time to run our existing REST Assured test suite. This execution will allow Keploy to capture the API requests and responses.
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729159715257/a93d6f7e-da22-4bf1-9d86-8955deaff5e3.png align="center")
    
    Each of the test cases generated by keploy is a REST Assured test case: -
    

```java
    @Test
    public void testGetEmployeeById() {
        get("/api/employees/" + testEmployee.getId())
            .then()
            .statusCode(200)
            .body("id", equalTo((int) testEmployee.getId()))  // Ensure correct ID
            .body("firstName", equalTo("John"))
            .body("lastName", equalTo("Doe"))
            .body("email", equalTo("john.doe@example.com"));
    }

    @Test
    public void testUpdateEmployee() {
        String updatedEmployeeJson = "{\"firstName\":\"Jane\",\"lastName\":\"Doe\",\"email\":\"jane.doe@example.com\"}";

        given()
            .contentType(ContentType.JSON)
            .body(updatedEmployeeJson)
        .when()
            .put("/api/employees/" + testEmployee.getId())
        .then()
            .statusCode(200)
            .body("firstName", equalTo("Jane"))
            .body("lastName", equalTo("Doe"))
            .body("email", equalTo("jane.doe@example.com"));
    }

    @Test
    public void testDeleteEmployee() {
        delete("/api/employees/" + testEmployee.getId())
            .then()
            .statusCode(200);
        
        // Verify that employee no longer exists
        get("/api/employees/" + testEmployee.getId())
            .then()
            .statusCode(404);
    }
```

### Step 4: Next Steps

We have successfully migrated REST Assured test cases to the keploy test suite. Below is one of the such keploy test case: -

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729160151219/5cc8ea66-df94-4627-9001-e013289014f8.png align="center")

So, let’s run our keploy test suite by running - `keploy test -c "java -jar target/springbootapp-0.0.1-SNAPSHOT.jar" --delay 10`: -

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729161693620/d798a2eb-3d63-4f39-bcda-d289aeb41446.png align="center")

Since we have noise, that’s why one test failed, and we got around `~70.5%` of coverage with keploy.

## Conclusion

Migrating from REST Assured to Keploy offered various advantages, such as zero-code testing, real-time feedback, and streamlined CI/CD integration. By following the steps outlined in this guide, you can ensure a smooth transition while maximising the benefits of Keploy for your API testing needs.

By adopting Keploy, your development team can focus more on delivering high-quality software with reduced engineering effort, ultimately leading to improved productivity and software quality.

## References:

1. CI/CD - [https://keploy.io/docs/ci-cd/jenkins/](https://keploy.io/docs/ci-cd/jenkins/)
    
2. Get the Cloud trial - [https://keploy.io/docs/keploy-cloud/cloud-installation/](https://keploy.io/docs/keploy-cloud/cloud-installation/)