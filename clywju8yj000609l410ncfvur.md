---
title: "How to Use covdata for Better Code Coverage in Go"
datePublished: Mon Jul 22 2024 05:31:27 GMT+0000 (Coordinated Universal Time)
cuid: clywju8yj000609l410ncfvur
slug: how-to-use-covdata-for-better-code-coverage-in-go
tags: golang, code-coverage, integration-test, covdata

---

When you run your Go programs or integration tests, numbers of raw coverage files are typically generated and dumped into a directory specified by the `GOCOVERDIR` environment variable. These files contain valuable data about which parts of your code were executed during tests, offering a glimpse into your code's effectiveness and robustness. However, sifting through these raw files to extract actionable insights can be daunting and unclear for many developers.

This is where `covdata` comes into play—a powerful tool designed specifically to address the complexities of analysing raw coverage files in Go. `covdata` simplifies the process by providing a suite of subcommands, each tailored to transform these intricate data sets into more digestible formats. Whether you're looking to generate a straightforward text-formatted output that summaries coverage metrics or an elaborate HTML file that provides a detailed, line-by-line view of code coverage, `covdata` is equipped to meet your needs.

In this blog post, we'll explore the various capabilities of `covdata`, starting with its basic setup and moving through to its more advanced features. We'll discuss each subcommand in detail, demonstrating how they can be used to derive meaningful conclusions from your coverage data.

## Delving into the capabilities of covdata

To effectively showcase the functionality of the `covdata` subcommands, I had my raw coverage dumped in subdirectories of `coverage-reports` by changing the `GOCOVERDIR` environment variable before each test set run.

```bash
coverage-reports
├── test-set-0
│   ├── covcounters.e6d14d876bdf4f3fb4e0a31953521510.35239.1720802735879395086
│   └── covmeta.e6d14d876bdf4f3fb4e0a31953521510
└── test-set-1
    ├── covcounters.e6d14d876bdf4f3fb4e0a31953521510.35331.1720802740927496918
    └── covmeta.e6d14d876bdf4f3fb4e0a31953521510
```

To follow along, download the coverage-reports directory from [https://github.com/AkashKumar7902/covdata-demostration](https://github.com/AkashKumar7902/covdata-demostration)

1. ### go tool covdata textfmt
    

The `textfmt` subcommand is designed to convert raw coverage data files into a machine-readable, text-based and traditional coverage format. The output file generated can be used to generate analytics in CI/CD pipelines.

Lets run this on coverage-reports/test-set-0:

```bash
go tool covdata textfmt -i="coverage-reports/test-set-0" -o="coverage-reports/test-set-0/test-set-0.txt"
```

`test-set-0.txt` would look something like this:

```bash
mode: set
test-app-url-shortener/handler.go:27.56,31.16 4 1
test-app-url-shortener/handler.go:31.16,33.3 1 0
test-app-url-shortener/handler.go:99.16,102.3 2 0
test-app-url-shortener/main.go:70.76,72.3 1 0
test-app-url-shortener/main.go:73.2,73.31 1 1
```

2. ### go tool covdata percent
    

The `percent` subcommand calculates the total percentage of lines covered. This command simplifies understanding test effectiveness by providing a quick numerical summary of coverage.

Lets run this on coverage-reports/test-set-1:

```bash
go tool covdata percent -i=coverage-reports/test-set-1
```

You would see an output like:  
`test-app-url-shortener coverage: 38.6% of statements`

### go tool covdata func

The `func` subcommand outputs detailed coverage profile information for each function within your Go codebase. This feature enables developers to assess the test coverage at a function-level granularity, providing insights into which parts of the application are well-tested and which may need more attention.

Running this on coverage-reports/test-set-1:

```bash
go tool covdata func -i=coverage-reports/test-set-1
```

It would generate an output like:

```bash
test-app-url-shortener/handler.go:27:   Get                     0.0%
test-app-url-shortener/handler.go:37:   Upsert                  0.0%
test-app-url-shortener/handler.go:52:   getJoke                 0.0%
test-app-url-shortener/handler.go:56:   getExcuse               100.0%
test-app-url-shortener/handler.go:60:   getURL                  0.0%
test-app-url-shortener/handler.go:76:   putURL                  0.0%
test-app-url-shortener/handler.go:109:  New                     100.0%
test-app-url-shortener/handler.go:119:  GenerateShortLink       0.0%
test-app-url-shortener/handler.go:126:  sha256Of                0.0%
test-app-url-shortener/handler.go:132:  base58Encoded           0.0%
test-app-url-shortener/main.go:23:      main                    90.3%
total                                   (statements)            38.6%
```

### go tool covdata merge

The `merge` subcommand is designed to consolidate multiple coverage data files into a single, comprehensive coverage report. This functionality is crucial for projects where tests are run in parallel or across different environments, as it allows for an aggregated view of overall code coverage, ensuring a complete and unified analysis of test effectiveness.

Let's merge data files in test-set-0 and test-set-1:

```bash
mkdir coverage-reports/merged-coverage
go tool covdata merge -i=coverage-reports/test-set-0,coverage-reports/test-set-1 -o=coverage-reports/merged-coverage
```

The merged data files gets dumped in merged-coverage directory after successful execution of the above command.

To see the merged-coverage percentage, we will run `percent` subcommand with merged-coverage as input directory:

```bash
 go tool covdata percent -i=coverage-reports/merged-coverage
```

It would generate an output like:  
`test-app-url-shortener coverage: 80.7% of statements`

### go tool covdata subtract

The `subtract` subcommand is designed to compare and subtract coverage data from two different sets of coverage files. This feature is particularly useful for identifying changes in code coverage between different test runs or configurations, enabling developers to pinpoint exactly which parts of the code have experienced decreases in test coverage. This helps focus improvement efforts on areas where coverage has regressed.

Running this on merged-coverage and test-set-0:

```bash
mkdir coverage-reports/sub-coverage
go tool covdata subtract -i=coverage-reports/merged-coverage,coverage-reports/test-set-1 -o=coverage-reports/sub-coverage
```

To see the coverage percentage of profile generated by `subtract` subcommand, Run:

```bash
go tool covdata percent -i=coverage-reports/sub-coverage
```

An output like this would be generated:  
`test-app-url-shortener coverage: 42.0% of statements`

### go tool covdata intersect

The `intersect` subcommand is used to identify and generate a coverage report based on the common coverage data found in multiple coverage files. This command is particularly valuable in scenarios where you want to determine the overlap in test coverage across different test suites or conditions, ensuring that key code paths are consistently tested under varying environments or setups.

Let's run this on test-set-0 and test-set-1 directories:

```bash
mkdir coverage-reports/intersect-coverage
go tool covdata intersect -i=coverage-reports/test-set-0,coverage-reports/test-set-1 -o=coverage-reports/intersect-cov
erage
```

Run `percent` subcomand to see the coverage percentage of profile generated by `intersect`:

```bash
go tool covdata percent -i=coverage-reports/intersect-coverage
```

It would generate an output like:  
`test-app-url-shortener coverage: 36.4% of statements`

## Conclusion

In conclusion, the `covdata` tool offers essential support for Go developers focused on delivering high-quality software. It simplifies the analysis of test coverage by converting raw coverage files into comprehensive, easy-to-understand reports, `covdata` not only enhances your testing strategy but also aids in continuous improvement by pinpointing areas where code can be refined and optimised.

With its straightforward commands and ability to convert complex data into easy-to-understand reports, `covdata` is more than just a tool—it’s a critical component in the ongoing effort to refine and perfect software projects. Using `covdata`, developers can ensure comprehensive coverage, reduce bugs, and maintain software excellence, ultimately leading to more robust and reliable applications in the competitive tech landscape.

#### References:

1. [https://pkg.go.dev/cmd/covdata](https://pkg.go.dev/cmd/covdata)
    
2. [https://go.dev/doc/build-cover](https://go.dev/doc/build-cover)