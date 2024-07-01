---
title: "My Testing Journey with Jasmine and Mocha"
datePublished: Mon Jul 01 2024 09:27:56 GMT+0000 (Coordinated Universal Time)
cuid: cly2s1hvn00040al6e5ydcav3
slug: my-testing-journey-with-jasmine-and-mocha
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711952341080/ca9b4154-7a2e-48f9-bbef-0bf13124ac0a.png

---

We all know the why it's important to write clean, reliable code. But let's face it, catching bugs before they wreak havoc on our applications can feel like chasing after a greased weasel. That's where testing frameworks come in, acting as our trusty bug-hunting companions.

In this blog, we'll be putting two popular frameworks – Jasmine and Mocha – head-to-head. We'll explore their strengths, weaknesses, and quirks, all through the lens of a developer's experience. Buckle up, grab a metaphorical cup of debugging coffee, and let's get started!

## **Introduction: What is Jasmine?**

Imagine explaining your code's behavior like telling a captivating story. That's the essence of Jasmine, it's a Behavior-Driven Development (BDD) framework. It encourages us to write tests that mimic real-world scenarios, making them easier to understand for both developers and non-technical stakeholders.

## How to write Tests with Jasmine?

Here's a glimpse into Jasmine's world:

### **1\. Setting the Stage: describe() and it()**

Jasmine uses `describe()` and `it()` functions to structure our tests. Think of `describe()` as the act of setting the scene for a particular functionality. We give it a clear and concise description (like "***Calculator Functionality***"). Inside this scene, each `it()` block represents a specific test case, detailing the expected behavior (e.g., "it should add two numbers correctly").

```javascript
describe("Calculator Functionality", function() {
  it("should add two numbers correctly", function() {
  });
});
```

**here:**

* `describe` defines a test suite (like a category).
    
* `it` defines an individual test case (what should happen when we add 2 and 3).
    
* `expect` is Jasmine's built-in assertion library, checking if the result (5) matches our expectation.
    

### **2\. The Assertive Voice: expect()**

Every good story needs a strong conclusion, *right!!*. That's why, in Jasmine, we use the `expect()` function to make assertions about the outcome of our tests. This is where we verify if the actual results match our expectations. Jasmine provides a rich set of matchers like `toEqual()`, `toBeGreaterThan()`, and `toBeLessThan()` to express various conditions.

```javascript
it("should add two numbers correctly", function() {
  const calculator = new Calculator();
  const result = calculator.add(2, 3);
  expect(result).toEqual(5); // Asserting the expected sum
});
```

### **3\. Hooks**

Jasmine offers optional hooks like `beforeEach()`, `afterEach()`, and `beforeAll()`, `afterAll()` for setting up and tearing down test environments or shared data across tests.

## Understanding Mocha

If you prefer a more streamlined and lightweight approach, Mocha might be your champion. It focuses on providing a flexible testing framework without imposing a specific BDD structure. This allows for greater customization and control over your tests.

## How does Mocha work?

Unlike Jasmine, Mocha comes bundled with a test runner, making it easier to execute your tests directly from the command line. This simplifies the testing workflow for developers who value efficiency.

### 1\. **describe() and it()**

Similar to Jasmine, Mocha uses `describe()` and `it()` functions for test organization. However, the focus leans more towards describing the test itself rather than the functionality being tested.

```javascript
describe("Calculator Addition Test", function() {
  it("adds 2 and 3 to equal 5", function() {
  });
});
```

### **2\. Assertions à la Carte:**

Mocha doesn't come with its own built-in assertions. This might seem like a limitation, but it actually allows you to choose the assertion library that best suits your needs. For example, most popular options include Chai:

```javascript
const chai = require('chai');
const expect = chai.expect;

describe("Calculator Addition Test", function() {
  it("adds 2 and 3 to equal 5", function() {
    const calculator = new Calculator();
    const result = calculator.add(2, 3);
    expect(result).to.equal(5); // Assertion using Chai
  });
});
```

## **Jasmine vs Mocha which to use?**

Honestly, there's no single victor. The choice between Jasmine and Mocha boils down to your development style and project needs.

**Jasmine's Pros:**

* Easy to learn and read, thanks to its BDD approach.
    
* Built-in assertions make writing tests concise.
    
* Extensive community and plenty of resources available.
    

**Jasmine's Cons:**

* Lacks a built-in test runner; requires additional tools like Karma.
    
* Can become cluttered for complex test suites.
    

---

**Mocha's Pros:**

* Lightweight and flexible framework with no imposed structure.
    
* Built-in test runner simplifies test execution.
    
* Freedom to choose assertion libraries based on preference.
    

**Mocha's Cons:**

* Requires additional setup for assertions.
    
* Less intuitive for beginners compared to Jasmine's BDD style.
    

Although, The beauty of these frameworks lies in their adaptability. You can even leverage the strengths of both! Here's how:

* **Jasmine with Chai:** Combine Jasmine's BDD structure with Chai's rich set of assertions for a powerful testing experience.
    
* **Mocha with BDD features:** While Mocha isn't strictly BDD-focused, libraries like `mocha-describe` can bring a BDD-like syntax to your Mocha tests.
    

Let's use Mocha and Jasmine with an actual program. Here's a simple program that calculates the area of a rectangle:

```javascript
function calculateArea(length, width) {
  return length * width;
}
```

### Get Started with Mocha

Let's first create a test folder and install mocha using: -

```bash
mkdir test && npm install mocha
```

Since we need to make assertions, mocha alone won't be enough so we will be using **<mark>Chai.js</mark>** along side with Mocha for our tests.

```bash
npm install chai
```

Now it's time to create our test with Mocha and Chai. Let's create a `test.js` file in our test folder we created earlier: -

```javascript
// test/test.mjs

import { expect } from 'chai';

describe("Rectangle Area Calculation", function() {
  function calculateArea(length, width) {
    return length * width;
  }

  it("should calculate the area correctly", function() {
    const area = calculateArea(5, 4);
    // Positive Testcase
    expect(area).to.equal(20); // Using expect from Chai
  });
});

describe("Rectangle Area Calculation", function() {
  function calculateArea(length, width) {
    return length * width;
  }

  it("should calculate the area correctly", function() {
    const area = calculateArea(3, 15);
    // Negative testcase
    expect(area).to.equal(40); // Using expect from Chai
  });
});
```

Above we have created two test cases, first where width is 5 and height is 4 and verifying whether the area of rectangle would 20 or not, which will result in pass testcase. On the otherhand, in second testcase, we have our height and width defined as 3 and 15 respectively, and is checking whether the area would 40 or not, this a fail test case we have created to check if our `function calculateArea()` is working as we expect it to work or not.

Now, let's our testcases using `npx mocha` : -

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711949626685/24ecfeb1-0995-48db-9ad6-e681889dd0c6.png align="center")

From above we can refer that both the test result are what we have expected and our function is working fine as expected.

### First Testcase with Jasmine

You can install Jasmine using npm locally in your project:

```bash
npm install --save-dev jasmine
```

Once we have installed the jasmine as dependency, We need to initialize a project for Jasmine by creating a spec directory and configuration json:

```javascript
npx jasmine init
```

**Running Specs**

Once we have set up our `jasmine.json`, we can execute all our specs by running `jasmine` from the root of your project.

```bash
├── app
│   ├── spec
│   │   ├── support
│   │   │   ├── jasmine.json
│   │   ├── appSpec.js
```

Unlike Mocha, once we have intialized the jasmine project, unless you want more granular control over the configuration, we don't need to add Jasmine library explicitly.

```javascript
// spec/appSec.js

describe("Rectangle Area Calculation", function() { 
  function calculateArea(length, width) {
  return length * width;
}
    it("should calculate the area correctly", function() {
      const area = calculateArea(5, 4);
      expect(area).toEqual(20);
    });
});

describe("Rectangle Area Calculation", function() {
    function calculateArea(length, width) {
      return length * width;
    }
  
    it("should calculate the area correctly", function() {
      const area = calculateArea(3, 15);
      expect(area).toEqual(40);
    });
});
```

Now let's run our test cases with `npx jasmine`: -

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711951102640/6eda7ab5-2d40-4166-87ff-b23c1d288543.png align="center")

In Jasmine, you can provide `seed` as a runtime argument. If a test fails when executing a test suite, then you can use the seed to rerun the tests in the exact order it ran last time. Then, you can find out why that one test fails given the execution order.

## Conclusion

Testing is an indispensable part of the development process. By embracing frameworks like Jasmine and Mocha, you can write robust code, catch bugs early, and ultimately deliver high-quality applications. So, keep testing, keep learning, and have fun along the way!

Ultimately, the best testing framework is the one that empowers you to write clear, maintainable, and effective tests. Experiment with Jasmine and Mocha, find the approach that resonates with you and watch your codebase flourish with confidence.

In this part, we got to know what is jasmine and how is it different from existing frameworks like *Mocha,* in the next part we will deep dive into how to write complex test cases using Jasmine.

## FAQ's

### **What is the difference between Jasmine and Mocha?**

Jasmine is a BDD framework with built-in assertions, suitable for straightforward testing. Mocha, while offering flexibility, lacks built-in assertions but allows customization and integrates seamlessly with various assertion libraries.

### **Which framework has a larger community and resources available?**

Jasmine boasts an extensive community with abundant resources, making it easier to find support and guidance for testing challenges.

### **Which framework is easier for beginners?**

Jasmine's BDD structure and built-in assertions make it more beginner-friendly, providing clear guidelines for test organization and assertion syntax. Mocha itself doesn't enforce BDD, but libraries like `mocha-describe` enable BDD-style syntax within Mocha tests for those who prefer it.

### **Can I use both frameworks together?**

Yes, you can leverage the strengths of both frameworks. For example, you can use Jasmine's BDD structure with Chai's assertion library in Mocha for a powerful testing experience. Alternatively, you can use Mocha with plugins to achieve a BDD-like syntax.