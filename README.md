# lttng-linux-trace-toolkit
Trace the Linux kernel, user applications using LTTng. 

### Installation LTTNG

**On Debian and Ubuntu:**

**stable release using package manager**


```
sudo apt-add-repository ppa:lttng/stable-2.10
sudo apt update
sudo apt install lttng-modules-dkms lttng-tools python3-lttnganalyses
```


### Installation Babeltrace 2

Babeltrace 2: A rich, flexible trace manipulation toolkit which includes a versatile command-line interface

**On Debian and Ubuntu:**

**stable release using package manager**


```
apt-add-repository ppa:lttng/ppa
apt-get update
apt-get install babeltrace2
```


### Trace a user application

This section steps you through a simple example to trace a Hello world program written in C.

### Step 1

Create the tracepoint provider header file, which defines the tracepoints and the events they can generate:


```
mkidr ~/lttng-user-app-traces
cd lttng-user-app-traces
vim hello-tp.h
```

Copy the following code into `hello-tp.h`, and save it.

```
#undef TRACEPOINT_PROVIDER
#define TRACEPOINT_PROVIDER hello_world

#undef TRACEPOINT_INCLUDE
#define TRACEPOINT_INCLUDE "./hello-tp.h"

#if !defined(_HELLO_TP_H) || defined(TRACEPOINT_HEADER_MULTI_READ)
#define _HELLO_TP_H

#include <lttng/tracepoint.h>

TRACEPOINT_EVENT(
    hello_world,
    my_first_tracepoint,
    TP_ARGS(
        int, my_integer_arg,
        char*, my_string_arg
    ),
    TP_FIELDS(
        ctf_string(my_string_field, my_string_arg)
        ctf_integer(int, my_integer_field, my_integer_arg)
    )
)

#endif /* _HELLO_TP_H */

#include <lttng/tracepoint-event.h>
```

### Step 2: 
#### Create the tracepoint provider package source file:

```
cd lttng-user-app-traces
vim hello-tp.c
```
Copy the following code into `hello-tp.c`, and save it.

```
#define TRACEPOINT_CREATE_PROBES
#define TRACEPOINT_DEFINE

#include "hello-tp.h"
```

### Step 3
#### Build the tracepoint provider package:


```
sudo gcc -c -I /usr/src/lttng-modules-2.12.x+stable+git1320+202007022118~ubuntu16.04.1 hello-tp.c -I /home/arslan/lttng-user-app-traces 
```

### Step 4

#### Create the Hello World application source file:


```
cd lttng-user-app-traces
vim hello.c

```
Copy the following code into `hello.c`, and save it.

```
#include <stdio.h>
#include "hello-tp.h"

int main(int argc, char *argv[])
{
    int x;

    puts("Hello, World!\nPress Enter to continue...");

    /*
     * The following getchar() call is only placed here for the purpose
     * of this demonstration, to pause the application in order for
     * you to have time to list its tracepoints. It's not needed
     * otherwise.
     */
    getchar();

    /*
     * A tracepoint() call.
     *
     * Arguments, as defined in hello-tp.h:
     *
     * 1. Tracepoint provider name   (required)
     * 2. Tracepoint name            (required)
     * 3. my_integer_arg             (first user-defined argument)
     * 4. my_string_arg              (second user-defined argument)
     *
     * Notice the tracepoint provider and tracepoint names are
     * NOT strings: they are in fact parts of variables that the
     * macros in hello-tp.h create.
     */
    tracepoint(hello_world, my_first_tracepoint, 23, "hi there!");

    for (x = 0; x < argc; ++x) {
        tracepoint(hello_world, my_first_tracepoint, x, argv[x]);
    }

    puts("Quitting now!");
    tracepoint(hello_world, my_first_tracepoint, x * x, "x^2");

    return 0;
}
```


### Step5

#### Build the application:

```
gcc -c hello.c
```


### Step 6

#### Link the application with the tracepoint provider package, liblttng-ust, and libdl:

```
gcc -o hello hello.o hello-tp.o -llttng-ust -ldl
```


# To trace the user application:

### Step: 1

Run the application with a few arguments:

```
./hello world and beyond
```

You will see following on your terminal:

```
Hello, World!
Press Enter to continue...
```

Open new terminal, and leave current terminal to be open.


### Step: 2

#### Start an LTTng session daemon:

```
lttng-sessiond --daemonize
```

### Step: 3

#### List the available user space tracepoints:

```
lttng list --userspace
```

You see the `hello_world:my_first_tracepoint` tracepoint listed under the `./hello process`.

### Step: 4 

#### Create a tracing session:

```
lttng create my-user-space-session
```

### Step: 5

### Create an event rule which matches the hello_world:my_first_tracepoint event name:

```
lttng enable-event --userspace hello_world:my_first_tracepoint
```

### Step: 6

#### Start tracing:

```
lttng start
```

Go back to the running hello application and press Enter. The program executes all tracepoint() instrumentation points and exits.

### Step: 7

#### Destroy the current tracing session:

```
lttng destroy
```


# View and analyze the recorded events:

Use the babeltrace2 command-line tool
The simplest way to list all the recorded events of an LTTng trace is to pass its path to babeltrace2 without options:

```
babeltrace2 ~/lttng-traces/my-user-space-session*
```

Here is the output on terminal:


```
arslan@arslan-HP-ProBook-450-G5:~/arslan-data/software-development/new-tools$ babeltrace2 ~/lttng-traces/my-user-space-session*
[12:51:30.604663755] (+?.?????????) arslan-HP-ProBook-450-G5 hello_world:my_first_tracepoint: { cpu_id = 2 }, { my_string_field = "hi there!", my_integer_field = 23 }
[12:51:30.604665853] (+0.000002098) arslan-HP-ProBook-450-G5 hello_world:my_first_tracepoint: { cpu_id = 2 }, { my_string_field = "./hello", my_integer_field = 0 }
[12:51:30.604666239] (+0.000000386) arslan-HP-ProBook-450-G5 hello_world:my_first_tracepoint: { cpu_id = 2 }, { my_string_field = "world", my_integer_field = 1 }
[12:51:30.604666550] (+0.000000311) arslan-HP-ProBook-450-G5 hello_world:my_first_tracepoint: { cpu_id = 2 }, { my_string_field = "and", my_integer_field = 2 }
[12:51:30.604666851] (+0.000000301) arslan-HP-ProBook-450-G5 hello_world:my_first_tracepoint: { cpu_id = 2 }, { my_string_field = "beyond", my_integer_field = 3 }
[12:51:30.604675714] (+0.000008863) arslan-HP-ProBook-450-G5 hello_world:my_first_tracepoint: { cpu_id = 2 }, { my_string_field = "x^2", my_integer_field = 16 }

```


