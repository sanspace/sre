# Understanding Linux Processes

A process is an instance of a running program. For an SRE, understanding how to manage and inspect processes is fundamental for troubleshooting, performance analysis, and ensuring system reliability.

---

## What is a Process?

When you execute a program, the operating system creates a process. Each process has its own address space, system resources (like file descriptors), and a unique Process ID (PID).

**Key Attributes of a Process:**

* **PID (Process ID):** A unique integer that identifies the process.
* **PPID (Parent Process ID):** The PID of the process that created this process. Most processes are children of `init` (PID 1) or `systemd` (PID 1).
* **UID/GID (User ID/Group ID):** The user and group that own the process, determining its permissions.
* **State:** Processes can be in various states:
    * **R (Running or Runnable):** Actively using the CPU or waiting for its turn.
    * **S (Interruptible Sleep):** Waiting for an event to complete (e.g., I/O, a signal).
    * **D (Uninterruptible Sleep):** Usually waiting for I/O (e.g., disk activity) and cannot be interrupted by signals. High numbers of these can indicate I/O problems.
    * **T (Stopped or Traced):** Stopped by a job control signal (e.g., Ctrl+Z) or because it's being debugged.
    * **Z (Zombie):** A process that has completed execution, but its entry still remains in the process table because its parent has not yet read its exit status. Too many zombies can (rarely) exhaust PIDs.
* **Priority & Niceness:** Values that influence how the OS schedules the process.

---

## Processes, Threads, and Multiprocessing

* **Process:** An independent program with its own memory space. Creating a new process (forking) is relatively expensive. Communication between processes (Inter-Process Communication - IPC) requires explicit mechanisms (pipes, sockets, shared memory).
* **Thread:** A lightweight unit of execution *within* a process. Threads share the same memory space as their parent process and other threads within that process. This makes communication between threads easier and creation/context-switching faster than processes.
* **Multiprocessing:** Using multiple processes (CPUs/cores) to execute parts of a program or multiple programs simultaneously.
* **Multithreading:** Using multiple threads within a single process to perform tasks concurrently, often improving responsiveness and throughput for I/O-bound or parallelizable tasks within that process.

**SRE Perspective:**
- Understanding if an application is single-threaded, multi-threaded, or uses multiple processes helps in diagnosing performance bottlenecks.
- A single-threaded application might not fully utilize a multi-core CPU for CPU-bound tasks.
- Issues in one thread can sometimes impact other threads in the same process.
- Knowing how an application scales (more processes vs. more threads) is crucial for capacity planning.

---

## Viewing and Inspecting Processes

Several commands help you view and inspect running processes:

### 1. `ps` (Process Status)

The `ps` command provides a snapshot of the currently running processes. It has many options:

* **`ps aux`**: Shows all processes (a=all, u=user-oriented format, x=processes without a controlling terminal). This is a very common and informative invocation.
    ```bash
    # Example output (truncated)
    # USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    # root         1  0.0  0.1 169560 13188 ?        Ss   Apr23   0:30 /sbin/init
    # root         2  0.0  0.0      0     0 ?        S    Apr23   0:00 [kthreadd]
    # user1     1234  0.5  2.5 876543 54321 ?        Sl   10:00   0:15 /usr/bin/my-application
    ```
* **`ps ef`**: Shows all processes (e=all, f=full-format listing, often showing process hierarchy).
    ```bash
    # Example output showing hierarchy (truncated)
    # UID        PID  PPID  C STIME TTY          TIME CMD
    # root         1     0  0 Apr23 ?        00:00:30 /sbin/init
    # root      1000     1  0 Apr23 ?        00:00:05  \_ /usr/lib/systemd/systemd-journald
    # user1     1234  1000  0 10:00 ?        00:00:15      \_ /usr/bin/my-application -c /etc/app.conf
    ```
* **`ps -p <PID>`**: Show information about a specific PID.
* **`ps -u <username>`**: Show processes owned by a specific user.
* **`ps -C <commandname>`**: Show processes by command name.

**SRE Perspective:**
- Identify which user is running a process.
- Find the PID of a misbehaving application.
- Check if expected daemons or services are running.
- View command-line arguments to understand how a process was started.

### 2. `top`

An interactive, real-time view of system processes. Processes are typically sorted by CPU usage by default.

* **Key Information in `top`:** PID, USER, PR (Priority), NI (Nice value), VIRT (Virtual Memory), RES (Resident Memory), SHR (Shared Memory), S (Status), %CPU, %MEM, TIME+ (CPU Time), COMMAND.
* **Interactive Commands (while `top` is running):**
    * `k`: Kill a process (prompts for PID and signal).
    * `r`: Renice a process (change priority).
    * `M`: Sort by memory usage.
    * `P`: Sort by CPU usage (default).
    * `u`: Filter by user.
    * `q`: Quit.
    * `1`: Toggle display of individual CPU core statistics.

### 3. `htop`

An enhanced, more user-friendly interactive process viewer (often needs to be installed separately, e.g., `sudo apt install htop` or `sudo yum install htop`).

* **Features:** Colorized display, easier scrolling, tree view, direct F-key shortcuts for actions like kill, renice.
* Generally preferred over `top` for interactive use if available.

**SRE Perspective for `top`/`htop`:**
- Quickly identify processes consuming the most CPU or memory *right now*.
- Monitor system load and resource utilization in real-time.
- Interactively manage processes (kill, renice) during an incident if necessary (with caution).

---

## Sending Signals to Processes (`kill`, `pkill`, `killall`)

Signals are a way to communicate with processes, often to request them to terminate or change behavior.

* **`kill <PID>`**: Sends a signal to the specified PID. By default, sends `SIGTERM (15)`.
    ```bash
    kill 1234 # Sends SIGTERM to PID 1234
    kill -9 1234 # Sends SIGKILL to PID 1234 (force kill)
    kill -l # Lists all available signals
    ```
* **`pkill <process_name>`**: Sends a signal to processes matching a name or other criteria.
    ```bash
    pkill my-application # Sends SIGTERM to all processes named 'my-application'
    pkill -u user1 defunct-app # Sends SIGTERM to 'defunct-app' run by 'user1'
    ```
* **`killall <process_name>`**: Similar to `pkill`, sends a signal to all processes matching the exact command name. (Behavior can vary slightly from `pkill`).
    ```bash
    killall -s SIGINT my-webserver
    ```

**Common Signals for SREs:**

* **`1` or `SIGHUP` (Hang Up):** Often used to tell a daemon to reload its configuration file.
* **`2` or `SIGINT` (Interrupt):** Sent when you press `Ctrl+C`. Usually requests graceful termination.
* **`9` or `SIGKILL` (Kill):** The "unkillable" kill signal. Forces immediate termination. The process doesn't get a chance to clean up. **Use as a last resort** as it can lead to data corruption or unclean state.
* **`15` or `SIGTERM` (Terminate):** The default signal for `kill`. Requests graceful termination. The process can catch this signal and perform cleanup operations before exiting. **Always try this before `SIGKILL`**.

**SRE Perspective:**
- **Graceful Shutdown:** Use `SIGTERM` to allow applications to shut down cleanly (save state, close connections).
- **Force Termination:** Use `SIGKILL` for unresponsive processes that don't respond to `SIGTERM`, but understand the risks.
- **Configuration Reloads:** Use `SIGHUP` for services that support it to reload configurations without a full restart, minimizing downtime.
- Automating process restarts or cleanup often involves sending specific signals.

---

## SRE Scenarios Involving Processes:

* **High CPU/Memory Usage:** Identifying which process is consuming excessive resources using `top`, `htop`, or `ps`.
* **Unresponsive Application:** Determining if a process is hung, in an uninterruptible sleep (D state), or not responding to normal signals.
* **Zombie Processes:** Checking for an accumulation of zombie processes (`Z` state in `ps`) which might indicate a bug in a parent process not reaping its children.
* **Service Management:** Ensuring critical application processes are running and restarting them if they fail (often handled by `systemd` or other init systems, but manual checks are sometimes needed).
* **Resource Limits:** Understanding how process resource limits (ulimits) can affect application behavior.
* **Debugging:** Using process information as a starting point for deeper debugging with tools like `strace`, `lsof`, or debuggers.

---
