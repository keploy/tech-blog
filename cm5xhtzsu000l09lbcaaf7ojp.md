---
title: "Mastering NYC: Enhance JavaScript & TypeScript Test Coverage"
seoTitle: "Mastering NYC: Enhance JavaScript & TypeScript Test Coverage"
seoDescription: "Use NYC, Istanbul's CLI, to improve JavaScript/TypeScript test coverage with Mocha, Jest, and Ava. Generates detailed reports"
datePublished: Wed Jan 15 2025 06:00:58 GMT+0000 (Coordinated Universal Time)
cuid: cm5xhtzsu000l09lbcaaf7ojp
slug: mastering-nyc-enhance-javascript-typescript-test-coverage
canonical: https://keploy.io/blog/technology/mastering-nyc-enhance-javascript-typescript-test-coverage
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1736920848553/a0b404cc-0c8b-49bf-a819-07f6c9aecc7c.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1736920843822/ef7c8d21-38ba-42ff-a4dd-65c06afcbe40.jpeg
tags: javascript, testing, typescript, node, keploy, nyc

---

NYC, often referred to as Istanbul's command-line interface (CLI), is a powerful code coverage tool designed specifically for JavaScript testing. It works seamlessly with testing frameworks like Mocha, Jest, and Ava, making it an invaluable resource for developers looking to measure and improve the coverage of their tests. NYC not only tracks how much of your code is covered by your existing test suite but also provides detailed reports that help identify untested parts of your codebase.

NYC offers a variety of output formats for its reports, including text summaries, detailed HTML pages, and JSON summaries, each providing different levels of detail and visual representation to suit various development and review processes. This flexibility makes NYC not just a tool for tracking test coverage, but also a vital part of continuous integration pipelines, ensuring that new code meets quality standards before it is merged.

Users can configure NYC extensively via `.nycrc` or package.json files, allowing them to tailor coverage criteria, reporting formats, and instrumenter options to fit their project's needs.

## Collecting Coverage for test scripts built using mocha

we will now create a sample test to demonstrate how to generate coverage data for tests made using mocha. Refer [https://istanbul.js.org/docs/tutorials/](https://istanbul.js.org/docs/tutorials/) for tutorials to use nyc with different testing frameworks.

The example project can be found on [https://github.com/AkashKumar7902/nyc-mocha](https://github.com/AkashKumar7902/nyc-mocha)

here is the test file for a simple `arithmetic.js` :

```javascript
const assert = require('assert');
const Arithmetic = require('./arithmetic');

describe('Arithmetic', function () {
    describe('#add()', function () {
        it('should return the sum of two numbers', function () {
            assert.strictEqual(Arithmetic.add(1, 1), 2);
        });
    });

    describe('#subtract()', function () {
        it('should return the difference of two numbers', function () {
            assert.strictEqual(Arithmetic.subtract(5, 3), 2);
        });
    });

    describe('#multiply()', function () {
        it('should return the product of two numbers', function () {
            assert.strictEqual(Arithmetic.multiply(4, 3), 12);
        });
    });
});
```

To run the tests with mocha change "test" script in package.json to:

```json
  "scripts": {
    "test": "nyc --reporter=html --reporter=text mocha"
  },
```

*here,*`--reporter=html`*flag creates a folder named*`coverage`*in the current working directory. To view the coverage data in html format, open coverage/index.html in your web browser.*

*and,*`--reporter=text`*displays the coverage data in a pretty format.*

Execute `npm test` to initiate the predefined tests and produce a coverage report. Here's what the output will look like:

```bash
> nyc-mocha@1.0.0 test
> nyc --reporter=html --reporter=text mocha

  Arithmetic
    #add()
      ✔ should return the sum of two numbers
    #subtract()
      ✔ should return the difference of two numbers
    #multiply()
      ✔ should return the product of two numbers

  3 passing (3ms)

---------------|---------|----------|---------|---------|-------------------
File           | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
---------------|---------|----------|---------|---------|-------------------
All files      |   57.14 |        0 |      75 |   57.14 |                   
 arithmetic.js |   57.14 |        0 |      75 |   57.14 | 15-18             
---------------|---------|----------|---------|---------|-------------------
```

## Collecting coverage data for each request coming to a server

Coverage data for each server run gets saved in the `.nyc_output` directory by default. In some e2e testing scenarios where server is restarted after every testset run, we can analyze the raw coverage files for each run and calculate overall coverage percentage.

Replaying test sets recorded with keploy starts the application each time and the coverage data for each run can be viewed from json files in `.nyc_output` . But not all json files in the nyc folder is useful (most of them are empty), to determine the useful json's we have to analyze the corresponding files in `.nyc_output/processinfo`. The files in this directory lists the process metadata and the files which are instrumented during the process run. we can mark the filename's as useful by checking if the `files` array in the processinfo json contains any element or not. Once we have this information, we can process the data in the useful json's and calculate the coverage percentage.

Below is the implementation of the above approach in go:

```go
func CalTypescriptCoverage() (models.TestCoverage, error) {
	testCov := models.TestCoverage{
		FileCov:  make(map[string]string),
		TotalCov: "",
	}

	coverageFilePaths, err := getCoverageFilePathsTypescript(filepath.Join(".", ".nyc_output", "processinfo"))
	if err != nil {
		return testCov, err
	}
	if len(coverageFilePaths) == 0 {
		return testCov, fmt.Errorf("no coverage files found")
	}

	// coverage is calculated as: (no of statements covered / total no of statements) * 100
	// no of statements covered is the no of entries in S which has a value greater than 0
	// Total no of statements is len of S

	linesCoveredPerFile := make(map[string]map[string]bool) // filename -> line -> covered/not covered

	for _, coverageFilePath := range coverageFilePaths {

		coverageData, err := os.ReadFile(coverageFilePath)
		if err != nil {
			return testCov, err
		}
		var cov TypescriptCoverage
		err = json.Unmarshal(coverageData, &cov)
		if err != nil {
			return testCov, err
		}

		for filename, file := range cov {
			if _, ok := linesCoveredPerFile[filename]; !ok {
				linesCoveredPerFile[filename] = make(map[string]bool)
			}
			for line, isStatementCovered := range file.S {
				if _, ok := linesCoveredPerFile[filename][line]; !ok {
					linesCoveredPerFile[filename][line] = false
				}
				if isStatementCovered.(float64) > 0 {
					linesCoveredPerFile[filename][line] = true
				}
			}
		}
	}

	totalLines := 0
	totalCoveredLines := 0
	coveredLinesPerFile := make(map[string]int) // filename -> no of covered lines
	for filename, lines := range linesCoveredPerFile {
		for _, isCovered := range lines {
			totalLines++
			if isCovered {
				totalCoveredLines++
				coveredLinesPerFile[filename]++
			}
		}
	}

	for filename, lines := range linesCoveredPerFile {
		testCov.FileCov[filename] = strconv.FormatFloat(float64(coveredLinesPerFile[filename]*100)/float64(len(lines)), 'f', 2, 64) + "%"
	}
	testCov.TotalCov = strconv.FormatFloat(float64(totalCoveredLines*100)/float64(totalLines), 'f', 2, 64) + "%"
	return testCov, nil
}
```

*here,*`TypescriptCoverage`*is a struct which depicts the structure of the json in .nyc\_output folder.*

and, `getCoverageFilePathsTypescript` function is used to get the file paths of the useful json. Here is the implementation:

```go
func getCoverageFilePathsTypescript(path string) ([]string, error) {
	filePaths := []string{}
	walkfn := func(path string, info os.FileInfo, err error) error {
		if !info.IsDir() && !strings.HasSuffix(path, "index.json") {
			fileData, err := os.ReadFile(path)
			if err != nil {
				return err
			}
			var processInfo ProcessInfo
			err = json.Unmarshal(fileData, &processInfo)
			if err != nil {
				return err
			}
			if len(processInfo.Files) > 0 {
				filePaths = append(filePaths, processInfo.CoverageFilename)
			}
		}
		return nil
	}
	err := filepath.Walk(path, walkfn)
	if err != nil {
		return nil, err
	}
	return filePaths, nil
}
```

## Conclusion

To conclude, NYC serves as a comprehensive tool for monitoring and improving code coverage across JavaScript and TypeScript projects. By integrating with popular testing frameworks like Mocha, it facilitates the generation of detailed coverage reports in various formats, enhancing the quality assurance process. These reports not only highlight the covered sections of code but also pinpoint areas lacking tests, guiding developers towards more robust software development.

Additionally, with the ability to replay test sets using Keploy and analyze coverage data through generated JSON files in the `.nyc_output` directory, developers can effectively manage and optimize test coverage.

## FAQ’s

### **What is NYC in JavaScript testing?**

NYC, Istanbul's CLI, is a powerful code coverage tool for JavaScript testing frameworks like Mocha, Jest, and Ava. It tracks and reports test coverage, identifying untested parts of the codebase.

### **How do I configure NYC for a Mocha test suite?**

Add `"test": "nyc --reporter=html --reporter=text mocha"` to the `scripts` section in your `package.json`. Run `npm test` to generate coverage reports in text and HTML formats.

### **What output formats does NYC support for coverage reports?**

NYC provides various report formats, including text summaries, detailed HTML pages, and JSON summaries, allowing flexibility for development and review workflows.

### **Can NYC analyze server-side test coverage?**

Yes, NYC can collect server-side test coverage data. The `.nyc_output` directory stores coverage files for each server run, which can be analyzed to calculate overall test coverage.

### **How does Keploy integrate with NYC for test coverage?**

Keploy replays test sets, restarting the server for each run. NYC captures coverage data, saved as JSON files in `.nyc_output`, which can be processed to calculate test coverage percentages effectively.