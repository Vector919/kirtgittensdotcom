---
layout: post
title: "How It Works: Node.JS"
---

Since I've been working with node.js quite a bit lately, to build a better mental model of how it works, I wanted to spend some time walking through it's source, and learning a bit about how it works.

##### High Level Architecture 
First, at a high level, a quick review of the Node.JS architecture  might be helpful:

Node embeds v8, a javascript engine, and libuv, a library used for asynchronous I/O.

v8 is used to parse and execute javascript and libuv is used to implement node's event loop and handle asynchronous operations.

###### Getting Started
Since nodeJS is a C++ Project, the first thing I looked for was the main function, it's in `src/node_main.cc`

There's some differences between windows and linux, although it looks like in both cases, the important part seems to be this call:

```c++
node::Start(argc, argv);
``` 

`Start()` is defined in `node.cc`:

First, `Start()` calls `InitializeOnePerProcess`, which appears to be responsible for setting up the options we pass to node, both the command line flags and the environment variables.

The two main pieces of interest in `Start()` are the creation of a `NodeMainInstance` object, and then calling `Run()` on it.
  
```c++
NodeMainInstance main_instance(&params,
                                   uv_default_loop(),
                                   per_process::v8_platform.Platform(),
                                   result.args,
                                   result.exec_args,
                                   indexes);
result.exit_code = main_instance.Run();
```

The `uv_default_loop()` that gets passed in the creation of the node instance, is a function from Libuv, that just returns a global event loop. 

The libuv docs say:

> uv_default_loop
Returns the initialized default loop. It may return NULL in case of allocation failure. This function is just a convenient way for having a global loop throughout an application.

So anything that needs to access the global event loop that nodeJS uses, can simply call this function, and gain access to it.

The node main instance also is passed some other arguments that come from `InitializeOncePerProcess()`, that look like they're related to passing around command line arguments to node.

So to recap:
```
int main()
  ----> node::Start()
    ----> NodeMainInstance (constructor) // Create a NodeMainInstance object
    ----> NodeMainInstance.Run() // Call Run() on it
```

`Run()` contains the most interesting parts of the source. The 2 main things that node does are:

1. Parse and run the Javascript Code
2. Run an event loop for handling async operations. 

`Run()` is responsible for both of these.

`LoadEnvironment()` is mostly responsible for parsing and executing your javascript. It's defined in [node.cc.](https://github.com/nodejs/node/blob/master/src/node.cc#L469) It calls [`StartMainThreadExecution()`](https://github.com/nodejs/node/blob/master/src/node.cc#L420). That function is responsible for executing a javascript file (which file it executes is different depending on what configuration options you've started node with).

I'm going to look at a really basic case for running node. When you want to run a single script file. That would look like this:

```bash
$ node myscript.js
```

The condition that matches in `StartMainThreadExecution()` is this:

```c++
if (!first_argv.empty() && first_argv != "-") {
    return StartExecution(env, "internal/main/run_main_module");
}
```

first_argv refers to the first "real" item in argv. argv[0] will always be the name of the executable being run from the command line. So in our example case first_argv should equal `myscript.js`

In that case, we call `StartExecution` with the string `internal/main/run_main_module`. That function goes into a pretty deep tree of calls that I don\'t think are necessarily 100% important for understanding what happens, but the basic idea is, that, we compile and run a script called: `internal/main/run_main_module.js` via v8.

At this point, `run_main_module.js` calls a function called `runMain` on `process.argv[1]` (which should be the name of your javascript file):
 
```javascript
require('internal/modules/cjs/loader').Module.runMain(process.argv[1]);
```

There's a lot of internal details in cjs/loader.js (This file basically implements the CommonJS module system, as well as allows passing off to es6 modules). In the simple case with CommonJS this essentially boils down to loading and running the file passed in, the same way that you would load a module when it is loaded via `require()`:

[`runMain` is patched on to the cjs loader,](https://github.com/nodejs/node/blob/master/lib/internal/bootstrap/pre_execution.js#L398) underneath, it's essentially a call to: `executeUserEntryPoint()` which, if you're using CommonJS, just runs `module._load` which runs `module._compile`. These functions take care of using v8 to compile and run your javascript code.

It looks like they work by calling into a c++ function from node in:
[`node_contextify.cc`](https://github.com/nodejs/node/blob/master/src/node_contextify.cc)

So, for executing javascript, Node:
1. Runs a javascript file (run_main_module.js) that calls into the commonJS loader.
2. This file uses the common JS module loader to load your JS code.
3. Common JS effectively uses v8 to compile your module and run the function it's compiled.

That looks like it's basically it for the implementation of loading and executing your javascript code! There's a lot of details that I've glossed over, but this is the core of how it works.

For implementing the event loop, there's a very straightforward `do`/`while` loop in [node_main_instance.cc in the `Run()` function](https://github.com/nodejs/node/blob/25447d82d34afb8c7f0f7e8fea10942586ef2d7c/src/node_main_instance.cc#L136):

```c++
do {
  // Runs the event loop until there  are no more handles or requests
  uv_run(env->event_loop(), UV_RUN_DEFAULT);

  per_process::v8_platform.DrainVMTasks(isolate_);

  more = uv_loop_alive(env->event_loop());
  if (more && !env->is_stopping()) continue;
} while (more == true && !env->is_stopping());
```

The important part here is `uv_run()`, the libuv docs say that: 

> this function runs the event loop. It will act differently depending on the specified mode:

> **UV_RUN_DEFAULT**: Runs the event loop until there are no more active and referenced handles or requests.

If you aren't already familiar  with libuv terminology, there's two important structures for dealing with asynchronous operations: Handles, and Requests.

**Handles** are basically long lived objects that will "wake up" the event loop when events are available for that particular object. An example of this might be a TCP Server. Events will come in when the TCP server has accepted a connection.

**Requests** are basically the representations of an individual operation that might trigger a call back. For example, one request might be sending data to a TCP socket, and waiting for a response.

So this loop will run the event loop until there are no more callbacks to process, and run it until either there is nothing left to process, or it is explicitly stopped.

Handles and requests are created by your javascript application code, through the available functions from the node.js standard library.

For example, if you want to create a TCP connection, nodeJS defines a [TCPWrap object](https://github.com/nodejs/node/blob/master/src/tcp_wrap.cc) in the C++ code, which then is available to any javascript code that wants to use it. The C++ tcp socket methods, use the libuv functions for TCP connections underneath, so connections are handled asynchronously by the event loop.

That's really it for the event loop! In a future post, I'd like to dive into the functionality of libuv a little more, and see how it's used.

That is, at a very high level, the core functionality that nodeJS provides, and how it's implemented in C++.

For More Reading:
[User guide — libuv documentation](http://docs.libuv.org/en/v1.x/guide.html)
[Getting started with embedding V8 · V8](https://v8.dev/docs/embed)