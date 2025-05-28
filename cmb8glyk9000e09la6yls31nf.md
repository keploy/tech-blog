---
title: "Building a CLI Tool in Go with Cobra and Viper"
seoTitle: "Build a CLI Tool in Go Using Cobra and Viper: Step-by-Step Guide"
seoDescription: "Learn how to build a powerful command-line interface (CLI) tool in Go using Cobra and Viper. Follow this step-by-step guide for easy configuration."
datePublished: Wed May 28 2025 21:30:43 GMT+0000 (Coordinated Universal Time)
cuid: cmb8glyk9000e09la6yls31nf
slug: building-a-cli-tool-in-go-with-cobra-and-viper
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1748466686712/8dde2282-b05d-47c3-8379-30bc5d1f7a52.jpeg
tags: go, tools, cli, cobra

---

> "The elegance of the command line lies in its ability to combine simple tools to accomplish complex tasks effortlessly." - Co-author of Unix

Go, with its simplicity and strong concurrency model, is a popular choice for building [CLI tools](https://keploy.io/blog/community/ai-and-cli-tools-a-new-era-of-developer-productivity-and-testing). Cobra and Viper are two powerful libraries in Go for building command-line interfaces and managing configuration, respectively. They are designed to work seamlessly together, offering a robust solution for developing feature-rich CLI applications. To understand how they work internally and complement each other, let's dive deeper into their architecture and how they interact.

### What are Cobra and Viper?

**Cobra** is a library for creating powerful modern CLI applications in Go. It's easy to use and provides a simple interface for creating commands, subcommands, and flags.

**Viper** is a complete configuration solution for Go applications. It can read configuration from different file formats (JSON, TOML, YAML, etc.), environment variables, command-line flags\*\*(cobra)\*\*, and more.

### Why Use Cobra and Viper Together?

Using Cobra and Viper together gives you the flexibility to handle both command-line flags and configuration files seamlessly. This combination is ideal for building CLI tools where you want to provide a rich set of features and options to the user. For example you know that you are going to use some flag every time with same value, It will be very inconvenient to mention that flag always in the command, If we are using viper along with cobra then these flags can be mentioned in the config file and the application reads out of that configuration file. There are many such usages.

### How Cobra Works?

```go
package main

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
)

func main() {
    // Root command
    var rootCmd = &cobra.Command{
        Use:   "cobracli",
        Short: "CobraCLI is a simple example CLI tool using Cobra",
        Long:  `CobraCLI is a simple example CLI tool to demonstrate how Cobra works with commands and subcommands.`,
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Println("Welcome to CobraCLI! Use --help to see available commands.")
        },
    }

    // Variables to store flag values
    var name string
    var excited bool

    // Subcommand: hello
    var helloCmd = &cobra.Command{
        Use:   "hello",
        Short: "Prints Hello with a name",
        Long:  `A simple subcommand that prints Hello followed by a name.`,
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Printf("Hello %s!\n", name)
        },
    }

    // Adding a flag to the hello command
    helloCmd.Flags().StringVarP(&name, "name", "n", "World", "name to greet")

    // Subcommand: goodbye
    var goodbyeCmd = &cobra.Command{
        Use:   "goodbye",
        Short: "Prints Goodbye with or without excitement",
        Long:  `A simple subcommand that prints Goodbye with optional excitement.`,
        Run: func(cmd *cobra.Command, args []string) {
            if excited {
                fmt.Println("Goodbye!")
            } else {
                fmt.Println("Goodbye.")
            }
        },
    }

    // Adding a flag to the goodbye command
    goodbyeCmd.Flags().BoolVarP(&excited, "excited", "e", false, "say goodbye with excitement")

    // Add subcommands to the root command
    rootCmd.AddCommand(helloCmd)
    rootCmd.AddCommand(goodbyeCmd)

    // Execute the root command
    if err := rootCmd.Execute(); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
}
```

**Root Command**:

* The **root command** is the base command of a CLI application. It's the entry point from which all subcommands are invoked. When you run the CLI tool without specifying any subcommands, the root command's functionality is executed.
    
* In our example, `rootCmd` is the root command.
    

**Subcommands**:

* **Subcommands** are commands that fall under a root command (or another subcommand) and provide additional functionality. Each subcommand has its own set of flags and can also have its own subcommands, forming a tree-like structure of commands.
    
* In the example, `helloCmd` and `goodbyeCmd` are subcommands of `rootCmd`.
    

**Key Functions and Fields of Cobra Commands**:

* `Use`: A string that specifies how the command should be used in the CLI. This is essentially the name of the command and how it's invoked from the command line. For example, `Use: "hello"` means you can invoke the command by typing `hello`.
    
* `Short`: A brief description of the command. This is used in the auto-generated help message to provide a quick summary of what the command does.
    
* `Long`: A more detailed description of the command. This is useful for providing a more comprehensive explanation in the help message.
    
* `Run`: A function that defines what happens when the command is executed. This is where you put the main logic of the command. The `Run` function is executed when the command is called from the command line.
    
* `Flags()`: A method that returns a flag set for the command. This is used to define flags that the command can accept.
    

**Adding Flags to Commands**:

* **Flags for** `helloCmd`:
    
    ```go
    helloCmd.Flags().StringVarP(&name, "name", "n", "World", "name to greet")
    ```
    
    * `Flags()` returns the flag set associated with the `helloCmd` command.
        
    * `StringVarP` defines a string flag (`--name` or `-n`) for the command, storing the value in the `name` variable. The default value is `"World"`, and the description is `"name to greet"`.
        
* **Flags for** `goodbyeCmd`:
    
    ```go
    goodbyeCmd.Flags().BoolVarP(&excited, "excited", "e", false, "say goodbye with excitement")
    ```
    
    * `BoolVarP` defines a boolean flag (`--excited` or `-e`) for the command, storing the value in the `excited` variable. The default value is `false`, and the description is `"say goodbye with excitement"`.
        

**Adding Subcommands to the Root Command**:

```go
rootCmd.AddCommand(helloCmd)
rootCmd.AddCommand(goodbyeCmd)
```

* `AddCommand()` attaches subcommands (`helloCmd` and `goodbyeCmd`) to the root command (`rootCmd`). This sets up the command hierarchy and ensures that Cobra knows how to handle and dispatch the subcommands when the CLI is invoked.
    

**How Cobra Processes Commands Internally**

1. **Initialisation**:
    
    * When the program starts, Cobra initialises the root command (`rootCmd`) and its associated subcommands (`helloCmd` and `goodbyeCmd`). Each command's flags and metadata are also initialised.
        
2. **Argument Parsing**:
    
    * When `Execute()` is called, Cobra starts parsing the command-line arguments provided by the user. It matches these arguments against the defined commands (`rootCmd`, `helloCmd`, `goodbyeCmd`) and their flags.
        
    * For example, if the user runs `cobracli hello --name=keploy`, Cobra recognises `hello` as a subcommand of `cobracli` and then parses the `--name` flag.
        
3. **Command Execution**:
    
    * After parsing, Cobra executes the corresponding `Run` function for the matched command.
        
    * If the user types `cobracli hello --name=keploy`, Cobra executes the `Run` function of `helloCmd`, which uses the flag value to print `Hello keploy!`.
        
4. **Help and Error Handling**:
    
    * Cobra automatically generates help messages and handles errors. If the user inputs an invalid command or requests help (`--help`), Cobra displays the appropriate help or error message based on the command hierarchy and flag definitions.
        

```bash
go build -o cobracli
```

cmds:

```bash
./cobracli
./cobracli --help
./cobracli hello
./cobracli hello --name=keploy
./cobracli hello -n keploy
./cobracli goodbye
./cobracli goodbye --excited
./cobracli goodbye -e
```

### Difficulties with cobra

* Maintaining flag variables in code
    
* Repetitive flag usage
    
* Too many flags in the cli command
    

Lets see if these problems can be solved by viper

### How Viper Works?

*config.yaml*

```yaml
app:
  name: "MyApp"
  version: "1.0"

database:
  host: "localhost"
  port: 5432
  user: "admin"
  password: "secret"
```

```go
package main

import (
    "fmt"
    "github.com/spf13/viper"
)

// Config struct to hold the configuration values
type Config struct {
    Host     string `mapstructure:"host"`
    Port     int    `mapstructure:"port"`
    User     string `mapstructure:"user"`
    Password string `mapstructure:"password"`
}

func main() {
    var config Config

    // Set the file name of the configurations file
    viper.SetConfigName("config")
    
    // Set the path to look for the configurations file
    viper.AddConfigPath(".") // optionally look for config in the working directory

    // Read the configuration file
    if err := viper.ReadInConfig(); err != nil {
        fmt.Printf("Error reading config file, %s", err)
        return
    }

    // Unmarshal the configuration file into the Config struct
    if err := viper.Unmarshal(&config); err != nil {
        fmt.Printf("Unable to decode into struct, %v", err)
        return
    }

    // Print the configuration values
    fmt.Println("Database Host:", config.Host)
    fmt.Println("Database Port:", config.Port)
    fmt.Println("Database User:", config.User)
    fmt.Println("Database Password:", config.Password)
}
```

run the binary

```bash
go build -o configreader
./configreader
```

**Define the Config Struct**:

* A `Config` struct is defined to hold all the configuration data. It includes nested structs for different sections (e.g., `App`, `Database`) of the configuration file.
    
* `mapstructure` Tags: These struct tags tell Viper how to map the configuration file keys to struct fields. The tags ensure that the configuration values are correctly assigned to the appropriate fields in the struct.
    

**Set Configuration File Name and Path**:

* `viper.SetConfigName("config")`: Specifies the base name of the configuration file without the extension.
    
* `viper.AddConfigPath(".")`: Adds the current directory as a path to look for the configuration file.
    

**Read Configuration File**:

* `viper.ReadInConfig()`: Reads the configuration file specified. If the file is not found or there’s an error reading it, the program prints an error message and exits.
    

**Unmarshal Configuration into Struct**:

* `viper.Unmarshal(&config)`: Viper unmarshals (i.e., decodes) the configuration data into the `Config` struct. If there’s an error during unmarshalling, the program prints an error message and exits.
    

**Print Configuration Values**: The program prints the configuration values stored in the struct.

**what else can viper do..?**

It can take input from environment variables, flags etc and combine them into one struct and later that struct can be used all over the application.

Lets see how that is done

```go
package main

import (
    "fmt"
    "os"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"
)

// Config struct to hold the configuration values
type Config struct {
    Host     string `mapstructure:"host"`
    Port     int    `mapstructure:"port"`
    User     string `mapstructure:"user"`
    Password string `mapstructure:"password"`
}

var config Config

func main() {
    // Initialize Cobra root command
    var rootCmd = &cobra.Command{
        Use:   "myapp",
        Short: "MyApp is an example application to demonstrate Viper and Cobra integration",
        Run: func(cmd *cobra.Command, args []string) {
            // Unmarshal the configuration into the Config struct
            if err := viper.Unmarshal(&config); err != nil {
                fmt.Printf("Unable to decode into struct, %v\n", err)
                return
            }

            // Print the configuration values
            fmt.Println("Database Host:", config.Host)
            fmt.Println("Database Port:", config.Port)
            fmt.Println("Database User:", config.User)
            fmt.Println("Database Password:", config.Password)
        },
    }

    // Define flags
    rootCmd.Flags().String("host", "localhost", "Database host")
    rootCmd.Flags().Int("port", 5432, "Database port")
    rootCmd.Flags().String("user", "admin", "Database user")
    rootCmd.Flags().String("password", "secret", "Database password")

    // Bind flags with Viper
    viper.BindPFlags(rootCmd.Flags())

    // Set environment variable prefix to avoid conflicts with other applications
    viper.SetEnvPrefix("myapp")
    viper.AutomaticEnv()

    // Bind environment variables
    viper.BindEnv("host")
    viper.BindEnv("port")
    viper.BindEnv("user")
    viper.BindEnv("password")

    // Set the configuration file name and path
    viper.SetConfigName("config")
    viper.AddConfigPath(".") // Search in the working directory

    // Read the configuration file
    if err := viper.ReadInConfig(); err != nil {
        fmt.Printf("Error reading config file, %s\n", err)
    }

    // Execute the root command
    if err := rootCmd.Execute(); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
}
```

1. **Define the Configuration Struct**: Create a struct (`Config`) to hold your configuration values. This struct will map to your configuration file and environment variables.
    
2. **Initialize Cobra Commands**: Use Cobra to define your CLI's root command and any subcommands, along with their respective flags.
    
3. **Bind Cobra Flags with Viper**: Use `viper.BindPFlags()` to bind Cobra command flags to Viper. This allows Viper to recognize the flags defined by Cobra, so you can manage them centrally.
    
4. **Set Up Environment Variables**: Use `viper.SetEnvPrefix()` to set a prefix for environment variables related to your application. Then, use `viper.BindEnv()` to bind specific environment variables to configuration keys.
    
5. **Read Configuration Files**: Use `viper.SetConfigName()` and `viper.AddConfigPath()` to specify the name and location of your configuration file. Call `viper.ReadInConfig()` to read the configuration file into memory.
    
6. **Unmarshal Configuration Data**: Use `viper.Unmarshal()` to decode the configuration data into your predefined struct (`Config`).
    

## **Conclusion**

Cobra and Viper are powerful libraries that create a strong base to build an amazing command line tool in Go. Cobra takes the pain out of organizing CLI applications through the commander and create commands and flags very easily. Viper creates flexibility in your applications as it allows you to manage configurations from files, environment variables, and flags.

The combination of the two libraries helps you solve some of the most common CLI pain points, such as many flags and configuration entry points, and helps get your CLI clean and easy to maintain, while providing a great user experience if default configurations are supported. Building utilities may seem simple, but building robust and extensible CLI applications is not. You easily add features without worrying about never-ending code maintainability.

## **FAQ’s**

### **1\. What is Cobra and why is it used for CLI development in Go?**

Cobra is a powerful Go library for creating modern CLI applications. It supports subcommands, flags, and intelligent help documentation. It’s used by tools like Kubernetes kubectl for its structure and flexibility.  
Cobra makes it easy to build user-friendly and extensible CLI tools.

### **2\. What role does Viper play alongside Cobra in a CLI tool?**

Viper handles configuration management in Go applications. It supports JSON, YAML, ENV variables, flags, and more. When paired with Cobra, Viper simplifies config loading and overrides. This makes it ideal for dynamic and portable CLI tools.

### **3\. How do Cobra and Viper work together in a CLI tool?**

Cobra manages the CLI structure and arguments, while Viper handles configuration values. You can bind Cobra flags directly to Viper variables. This allows users to configure your tool via flags, config files, or env vars. Together, they create a seamless command and config experience.

### **4\. What are the key steps to building a CLI with Cobra and Viper?**

First, initialize your Go project and install Cobra and Viper packages. Define your root command and add subcommands with Cobra. Use Viper to read config files and bind them to flags. Then, test and build your CLI binary for distribution.

### **5\. How can you structure a scalable CLI project with Cobra?**

Organize commands into separate files and directories for modularity. Use a cmd/ folder with one file per subcommand for clean code. Maintain a config/ package if needed to manage Viper settings. Follow Go module conventions for easy testing and maintenance.