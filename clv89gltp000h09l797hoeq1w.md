---
title: "Managing Go Processes"
seoTitle: "Managing Go Processes"
seoDescription: "While working on an application that required executing a command to run a blocking program, such as a TCP server, I encountered an interesting challenge"
datePublished: Fri Apr 19 2024 18:30:00 GMT+0000 (Coordinated Universal Time)
cuid: clv89gltp000h09l797hoeq1w
slug: managing-go-processes
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712074478334/96116a5c-7352-40f9-a3da-ecb99349f2a9.jpeg
tags: linux, go, command-line, shell, process, context

---

## Introduction: The Challenge of Managing Blocking Processes

While working on an application that required executing a command to run a blocking program, such as a ***TCP/HTTP*** server, I encountered an interesting challenge. I needed a way to stop the application and its child processes when a signal was sent to the main program, such as `SIGINT` (Ctrl+C) or `SIGTERM`. This blog post shares my journey and the solutions I found to manage processes effectively in a Go application, focusing on ***Linux*** environments.

## Executing Commands with the "os/exec" package

In my experience with the `os/exec` package in Go, I used `exec.Command` to run an external command, similar to how others use this package to execute external commands. However, when I tried to stop the command using `cmd.Process.Kill()`, I faced an issue. For eg: when running a command like `watch date > date.txt`, I noticed that the process continued to run in the background even after calling `cmd.Process.Kill()`. You can verify this by checking the active processes using cmd `ps -j`.

To address this issue, I realized, it was necessary to terminate the entire **process group** associated with the command to ensure a complete shutdown.

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"time"
)

func runCmd(userCmd string) error {
	cmd := exec.Command("sh", "-c", userCmd)

	// Set the output of the command
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	fmt.Println("executing cli: ", cmd.String())

	err := cmd.Start()
	if err != nil {
		return fmt.Errorf("failed to start the app: %w", err)
	}

	time.AfterFunc(5*time.Second, func() {
		cmd.Process.Kill()
	})

	err = cmd.Wait()
	if err != nil {
		return fmt.Errorf("unexpected error while waiting for the app to exit: %w", err)
	}
	return nil
}

func main() {
	runCmd("watch date > date.txt")
}
```

In the code snippet above, the use of `cmd.Process.Kill()` attempts to terminate the command after *5 seconds*, but as we've seen, it doesn't completely stop the background process.

Here is the output:

```bash
➜  cmd-graceful-exit-demo git:(main) ✗ go run process.go
executing cli:  /usr/bin/sh -c watch date > date.txt
                                                    %                                                                               
➜  cmd-graceful-exit-demo git:(main) ✗ ps -j
   PID     PGID    SID    TTY          TIME   CMD
211477  211477  211477 pts/33   00:00:01   zsh
274396  274328  211477 pts/33   00:00:00   watch
274709  274644  211477 pts/33   00:00:00   watch
274825  274825  211477 pts/33   00:00:00   ps
```

***Note on Using***`sh -c`***:***

Using `sh -c` in `exec.Command("sh", "-c", userCmd)` is important for some reasons:

**Pros:**

1. **Shell Features**: Allows the use of shell features like redirection, pipes, and expansions.
    
2. **Command Grouping**: Treats the entire command as a single unit, simplifying process management.
    

**Cons:**

1. **Dynamic Dependency**: Introduces a dependency on the shell, which could affect portability and consistency.
    
2. **Overhead**: Adds some overhead due to the additional shell process, which might impact performance in resource-constrained environments.
    

Using `sh -c` provides flexibility and control, but it's important to be aware of the potential drawbacks.

You might wonder why we should send a signal to the process group instead of individually signaling each child's PID ?

Sending signals to individual child processes may not be the most efficient approach for several reasons:

1. **Reduced Overhead**: Sending a signal to a process group requires fewer system calls compared to sending signals to each child process individually. This reduces the overhead and improves the efficiency of the signal-sending operation.
    
2. **Handling Dynamic Process Creation**: A child process might spawn new processes between the time you enumerate the PIDs and the time you send the signals. This could result in newly created processes that don't receive the intended signal. By signaling the entire process group, you ensure that all current and future processes in the group are affected, providing a more reliable way to manage process termination.
    
3. **Probability of Short-lived Processes**: While there is a low probability of short-lived process groups being created, individual child processes can be spawned and terminated quickly. Signaling the process group ensures that even these short-lived processes are accounted for, reducing the chances of leaving orphaned processes.
    

Now, let's understand the process groups concept.

## **Understanding Process Groups**

In Unix-like operating systems, a process group is a collection of one or more processes that can be treated as a single unit for signaling. Each process in a system belongs to a process group, and each process group has a unique process group ID (***PGID***). When a signal is sent to a ***PGID***, it is delivered to all processes in the process group.

After understanding the concept of process groups, we can now ensure that terminating our process doesn't leave any leftovers running in the background. By setting the `Setpgid` attribute to `true` in `SysProcAttr`, we create a new process group for our command. This allows us to use the negative PID (`-`[`cmd.Process.Pid`](http://cmd.Process.Pid)) with `syscall.Kill` to send the signal to the entire process group, ensuring a complete termination of the process and any of its child processes. *See ref \[2\] and \[3\] to know why we have used the -ve sign.*

Here's the updated code snippet that demonstrates this approach:

```go
func run(logger *zap.Logger, userCmd string) error {
	cmd := exec.Command("sh", "-c", userCmd)

	cmd.SysProcAttr = &syscall.SysProcAttr{
		Setpgid: true,
	}

	// Set the output of the command
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	logger.Debug("", zap.Any("executing cli", cmd.String()))

	err := cmd.Start()
	if err != nil {
		return fmt.Errorf("failed to start the app: %w", err)
	}

	time.AfterFunc(5*time.Second, func() {		
		syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL)
	})

	err = cmd.Wait()
	if err != nil {
		return fmt.Errorf("unexpected error while waiting for the app to exit: %w", err)
	}
	return nil
}
```

Now, You could be questioning as to why we used `Setpgid: true` in the code:

```go
cmd.SysProcAttr = &syscall.SysProcAttr{
    Setpgid: true,
}
```

The reason is simple: to avoid "suicide". Without `Setpgid: true`, the parent and child processes share the same process group. If we kill this group, it's like hitting a self-destruct button, The parent process goes down too. So, we set `Setpgid: true` to give the child its own process group. This way, when we send a kill signal to the child's group, the parent stays safe and running.

> *Setpgid sets the process group ID of the child to Pgid, or, if Pgid == 0, to the new child's process ID.*

Inside the [`forkAndExecInChild1`](https://github.com/golang/go/blob/db6097f8cbaceaed02051850d2411c88b763a0c3/src/syscall/exec_linux.go#L216), after forking a child process, the process group ID (***PGID***) is set using the following code snippet:

```go
	// Set process group
	if sys.Setpgid || sys.Foreground {
		// Place child in process group.
		_, _, err1 = RawSyscall(SYS_SETPGID, 0, uintptr(sys.Pgid), 0)
		if err1 != 0 {
			goto childerror
		}
	}
```

This code checks if the `sys.Setpgid` flag is true or if the process should run in the foreground (`sys.Foreground` is true). If either condition is met, the child process's ***PGID*** is set using the `SYS_SETPGID` syscall. The syscall takes the desired ***PGID*** as an argument. If `sys.Pgid` is zero, the child's PID is used as the ***PGID***, creating a new process group. If `sys.Pgid` is non-zero, the child is added to an existing process group with that ***PGID***.

### **Sending Signals to Multiple Process Groups**

In a complex application, it's possible that a process group's child processes might create their own process groups. To ensure a clean shutdown, we need to find all unique process groups and send a signal to each of them. The following function, `interruptProcessTree`, achieves this by interrupting an entire process tree using a given signal:

```go
func interruptProcessTree(logger *zap.Logger, ppid int, sig syscall.Signal) error {
    // Find all descendant PIDs of the given PID and then signal them.
    // Any shell doesn't signal its children when it receives a signal.
    // Children may have their own process groups, so we need to signal them separately.
    children, err := findChildPIDs(ppid)
    if err != nil {
        return err
    }

    children = append(children, ppid)
    uniqueProcess, err := uniqueProcessGroups(children)
    if err != nil {
        logger.Error("failed to find unique process groups", zap.Int("pid", ppid), zap.Error(err))
        uniqueProcess = children
    }

    for _, pid := range uniqueProcess {
        err := syscall.Kill(-pid, sig)
        // ignore the ESRCH error as it means the process is already dead
        if errno, ok := err.(syscall.Errno); ok && err != nil && errno != syscall.ESRCH {
            logger.Error("failed to send signal to process", zap.Int("pid", pid), zap.Error(err))
        }
    }
    return nil
}
```

This function first finds all child PIDs of the given parent PID recursively present at that moment, and then determines the unique process groups among them. It sends the specified signal to each unique process group, ensuring that all processes in the tree are interrupted. This approach is crucial for managing complex process hierarchies and ensuring a consistent state during shutdown. You can replace `syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL)` with this `interruptProcessTree(logger, cmd.Process.Pid, syscall.SIGKILL) error` function.

*Note:****SIGKILL****can't be captured, so you may want to send****SIGINT****or****SIGTERM****for a graceful shutdown of your application.*

### **Platform-Specific Considerations: The Proc File System**

In the implementation of the `interruptProcessTree` function, we collect unique process group IDs by reading the ***proc*** file system. The ***proc*** file system is a special filesystem in Unix-based operating systems like *Linux*, that provides a mechanism for the kernel to provide information about processes to userspace. It is typically mounted at `/proc` and contains various files and directories that represent the state of running processes.

Since the ***proc*** file system is specific to Unix-based systems, the current implementation of the process group interruption is limited to *Linux*. If you need to support other operating systems, you would need to write your own custom interrupter that can handle the signaling to all the child processes.

## **Utilizing Context Cancellation for Process Control in Go with**`exec.CommandContext`

[Context cancellation](https://go.dev/doc/database/cancel-operations) is a mechanism in Go for signaling a process to stop. It's useful for managing long-running processes or ensuring clean resource cleanup.

When working with long-running processes in Go, it's common to use `context.Context` to manage their lifecycle. The `exec.CommandContext` function provides a convenient way to run a command with a context, allowing you to cancel the process based on the context's cancellation. Here's an example of how you might use it:

```go
func run(ctx context.Context, logger *zap.Logger, userCmd string) error {
	
	cmd := exec.CommandContext(ctx, "sh", "-c", userCmd)

	cmd.SysProcAttr = &syscall.SysProcAttr{
		Setpgid: true,
	}

	// Set the output of the command
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	logger.Debug("", zap.Any("executing cli", cmd.String()))

	err := cmd.Start()
	if err != nil {
		return fmt.Errorf("failed to start the app: %w", err)
	}

	err = cmd.Wait()
	select {
	case <-ctx.Done():
		logger.Debug("context cancelled, error while waiting for the app to exit", zap.Error(ctx.Err()))
		return ctx.Err()
	default:
		if err != nil {
			return fmt.Errorf("unexpected error while waiting for the app to exit: %w", err)
		}
		logger.Debug("app exited successfully")
		return nil
	}
}
```

In this implementation, `exec.CommandContext` create a command that is aware of the provided context. If the context is cancelled, the default behavior of `CommandContext` is to send a *kill* signal to the process. Since the *kill* signal can't be captured, the application will not exit gracefully. And this is evident from the `CommandContext` function's implementation, where the `Cancel` function is set to call `cmd.Process.Kill()`:

```go
func CommandContext(ctx context.Context, name string, arg ...string) *Cmd {
    if ctx == nil {
        panic("nil Context")
    }
    cmd := Command(name, arg...)
    cmd.ctx = ctx
    cmd.Cancel = func() error {
        return cmd.Process.Kill()
    }
    return cmd
}
```

## **Evolution of Cancellation Mechanism in Go**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1712074065566/b71b7588-33e2-4e63-9493-09ded86ed5ae.gif align="center")

Before ***Go 1.20,*** the `CommandContext` function did not provide a built-in [cancellation mechanism](https://github.com/golang/go/blob/go1.19.13/src/os/exec/exec.go#L296) for the running process. Developers had to implement custom methods to handle graceful termination of the process. For example, the [`skaffold`](https://github.com/GoogleContainerTools/skaffold) project used `CommandContext` and created a custom method to send *interrupt* or *terminate* signals for graceful termination ([source, grac](https://github.com/GoogleContainerTools/skaffold/blob/6bff1758fe154f5e8da756a76518c8d46b813436/pkg/skaffold/build/custom/script.go#L41)[eful termination](https://github.com/GoogleContainerTools/skaffold/blob/main/pkg/skaffold/build/misc/graceful.go#L35)).

However, the situation improved with the resolution of [issue #50436](https://github.com/golang/go/issues/50436) in Go 1.20, which added support for a proper cancellation function in `CommandContext`. While the default behavior of this cancellation function is to *kill* the process, it is now possible to override this function to achieve more graceful termination.

## **Enhanced Cancellation with Wait Delay and Cancel**

To improve the cancellation mechanism, we can modify the `cmd.Cancel` function to send a `SIGINT` signal instead of killing the process immediately. Additionally, we can introduce a `cmd.WaitDelay` to wait for a specified duration after sending the interrupt signal before sending the *kill* signal:

```go
func run(ctx context.Context, logger *zap.Logger, userCmd string) error {
    cmd := exec.CommandContext(ctx, "sh", "-c", userCmd)

    // to run child process in a new process group
    cmd.SysProcAttr = &syscall.SysProcAttr{
		Setpgid: true,
	}

    // Set the cancel function for the command
    cmd.Cancel = func() error {
        return interruptProcessTree(logger, cmd.Process.Pid, syscall.SIGINT)
    }
    // wait after sending the interrupt signal, before sending the kill signal
    cmd.WaitDelay = 10 * time.Second

    // Set the output of the command
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr

    logger.Debug("Executing command", zap.String("command", cmd.String()))

    err := cmd.Start()
    if err != nil {
        return fmt.Errorf("failed to start the app: %w", err)
    }

    err = cmd.Wait()
    select {
    case <-ctx.Done():
        logger.Debug("Context cancelled", zap.Error(ctx.Err()))
        return ctx.Err()
    default:
        if err != nil {
            return fmt.Errorf("unexpected error while waiting for the app to exit: %w", err)
        }
        logger.Debug("App exited successfully")
        return nil
    }
}
```

## Conclusion

*Github Repo: (*[*Source Code*](https://github.com/gouravkrosx/golang-cmd-exit-demo)*)*

This blog post has explored various techniques for managing long-running processes in Go, including the use of `context.Context` for cancellation, handling process groups for independent signaling, and addressing platform-specific considerations like the *proc* file system on Linux. And a few other questions as well.

Please feel free to comment below. Your feedback is valuable in ensuring the accuracy and usefulness of this content.

***References:***

* [*Killing a Child Process and All of its Children in Go*](https://medium.com/@felixge/killing-a-child-process-and-all-of-its-children-in-go-54079af94773)*\- (1)*
    
* [*Signal Propagation in Linux*](https://www.baeldung.com/linux/signal-propagation)*\- (2)*
    
* [*kill(2) - Linux man page*](https://man7.org/linux/man-pages/man2/kill.2.html)*\- (3)*
    
* [*Find child pids recursively*](https://medium.com/p/f16bf97c8c4e)*\- (4)*  
    

## FAQ's

### **What are the benefits of using** `exec.CommandContext` **over** `exec.Command` for running commands in Go?

`exec.CommandContext` allows you to associate a command with a context, which enables cancellation and timeout functionality. This is particularly useful for managing long-running processes and ensuring that they can be stopped gracefully when necessary. By contrast, `exec.Command` does not provide built-in support for context cancellation.

### **Why is it important to manage process groups when terminating processes?**

Managing process groups ensures that when you terminate a process, all its child processes are also terminated. This is crucial for cleaning up resources effectively and avoiding orphaned processes that could consume system resources unnecessarily.

### **How can containerization technologies like Docker be leveraged to manage long-running processes more effectively?**

Containers like Docker provide a lightweight and isolated environment for running applications, including long-running processes. By dockerizing your applications, you can encapsulate dependencies, manage resource allocation, and ensure consistent behavior across different environments.

Additionally, container orchestration platforms like Kubernetes offer features for deploying, scaling, and managing containerized applications at scale, making it easier to handle long-running processes in distributed systems.

### **What are the potential risks of not properly managing long-running processes, especially in production environments?**

Not properly managing long-running processes can lead to various issues such as resource leaks, degraded performance, and even system instability.

For example, failing to terminate processes cleanly can result in processes that consume system resources without performing any useful work. In production environments, these issues can impact reliability and scalability, leading to service disruptions or outages.