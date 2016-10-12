---
title: Multithreading Model
description: Learn how to offload heavy work on a non-UI thread to create a responsive UI without slowing down rendering.
position: 110
---

# Multithreading Model

One of NativeScript's benefits is that it allows fast and efficient access to all native platform (Android/Objective-C) APIs through JavaScript, without using (de)serialization or reflection. This however comes with a tradeoff - all JavaScript executes on the main thread (AKA the `UI Thread`). That means that operations that potentially take longer can lag the rendering of the UI and make the application look and feel slow.

To tackle issues with slowness where UI sharpness and high performance are critical, developers can use NativeScript's solution to multithreading - Worker Threads. Workers are scripts executing in an absolutely isolated context, separate from the Main Thread. Tasks that could take long to execute should be offloaded on to a worker thread. 

Workers API in NativeScript is loosely based on the [Dedicated Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) and the [Web Workers Specification](https://www.w3.org/TR/workers/)

* [Workers API](#workers-api)
    * [Worker Object](#worker-object-prototype)
    * [Worker Global Scope](#worker-global-scope)
* [Sample Usage](#sample-usage)
* [Limitations](#limitations)

## Workers API

### Worker Object prototype
 - `new Worker(path)` - creates an instance of a Worker and spawns a new OS Thread, where the script pointed to by the `path` parameter is executed.
 - `postMessage(message)` - sends a JSON-serializable message to the associated script's `onmessage` event handler.
 - `terminate()` - terminates the execution of the Worker Thread on the first available tick

**Worker** Object event handlers
 - `onmessage(message)` - handle incoming messages sent from the associated Worker Thread. The message object exposes the following properties:
    - `message.data` - the message's content, as sent in the Worker Thread's `postMessage`
 - `onerror(error)` - handle uncaught errors from the Worker Thread. The error object exposes the following properties:
    - `error.message` - the uncaught error, and a stacktrace, if applicable
    - `error.filename` - the file where the uncaught error was thrown
    - `error.lineno` - the line where the uncaught error was thrown
 
### Worker Global Scope
 - `postMessage(message)` - sends a JSON-serializable message to the Worker instance's `onmessage` event handler on the Main Thread.
 - `close()` - terminates the execution of the Worker Thread on the first available tick

**Worker** Global Scope event handlers
 - `onmessage(message)` - handle incoming messages sent from the Main Thread. The message object exposes the following properties:
    - `message.data` - the message's content, as sent in the Worker Thread's `postMessage`
 - `onerror(error)` - handle uncaught errors occurring during execution of functions inside the Worker Scope (Worker Thread). The `error` parameter contains the uncaught error message. If the handler returns a true-like value, the message will not propagate to the Worker instance's `onerror` handler on the Main Thread.
 - `onclose()` - handle any "clean-up" work; suitable for freeing up resources, closing streams and sockets.

## Sample Usage

 main-view-model.js
 ```JavaScript
    ...

    var worker = new Worker('./workers/image-processor');
    worker.postMessage({ src: imageSource, mode: 'scale', options: options });

    worker.onmessage = function(msg) {
        if (msg.data.success) {
            // Stop idle animation
            // Update Image View
            // Terminate worker or send another message

            worker.terminate();
        } else {
            // Stop idle animation
            // Display meaningful message
            // Terminate worker or send message with different parameters
        }
    }

    worker.onerror = function(err) {
        console.log('An unhandled error occurred in worker: ${err.filename}, line: ${err.lineno} :');
        console.log(err.message);
    }

    ...
 ```

 workers/image-processor.js
 ```JavaScript
    onmessage = function(msg) {
        var request = msg.data;
        var src = request.src;
        var mode = request.mode || 'noop'
        var options = request.options;

        var result = processImage(src, mode, options);

        var msg = result !== undefined ? { success: true, src: result } : { }

        postMessage(msg);
    }

    function processImage(src, mode, options) {
        // image processing logic

        // save image, retrieve location

        // return source to processed image
        return updatedImgSrc;
    }

    // does not handle errors with an `onerror` handler
    // errors will propagate directly to the Main Thread Worker instance
 ```

## General Guidelines

 For optimal results when using the Workers API, follow these guidelines:
  - Always make sure you close the Worker Threads, using the appropriate API, when the worker's finished its job. If Worker instances become unreachable in the scope you are working in before you are able to terminate it, you will be able to close it only from inside the script.
  - Workers are not a general solution for all performance-related problems. Starting a Worker has an overhead of its own, and may sometimes be slower than just processing a quick task. Optimize DB queries, or rethink complex application logic before resorting to Workers.

## Limitations

 There are certain limitations to keep in mind when working with Threads:
  - You have **no direct access** to objects and variables from other Threads. Data is sent back and forth only with `postMessage`.
  - You can **ONLY** send **JSON-serializable** objects from one Thread to another. That means that native objects (Android, Obj-C) in JavaScript wrappers will be empty, even if you try to use them on another Thread.
  - You **cannot access or change UI Views** on a Worker Thread. Attempting to do so will cause an exception to be thrown inside the invoking Thread.
  - Worker Scripts are **not debuggable** as of NativeScript 2.4.
  - Workers **cannot** be created inside other Worker Threads.   
