# Javascript Promises

Javascript is a single-threaded language. You can think of a thread as a car lane. Multi-threaded languages have multiple car lanes to handle their traffic (processes). Javascript only has one lane, meaning if a car (process) is stuck for a few seconds, it has to pass before any other car can. This lane is referred to as the **Call Stack**.

Sometimes, we need to perform operations which may clog up this Call Stack. Let's say we have a function called `MyTimeout`. `MyTimeout` takes a function `x` and a number `y` as arguments, and executes `x` after `y` seconds. Functions passed in as arguments are called **Callback Functions**. Whenever `MyTimeout` is called, it will block our Call Stack for `y` seconds.

The problem with this is that the Call Stack is used by the browser to render a new frame of the website every 1/60th of a second. That means the website should be rendering at 60 frames per second, which are 60 processes that need to resolve each second. If we throw a `MyTimeout` into the mix, we end up delaying the render queue. For example, if I click a button which executes the `MyTimeout` function for 5 seconds, our page will stop rendering for 5 seconds. During this time, you won't be able to interact with your screen at all (including scrolling and hover animations).

The asynchronous version of our `MyTimeout` function would be the `setTimeout` method in Javascript. Other asynchronous processes can include event handlers, timers, and file system operations. One of the most common ones is `fetch` used to execute HTTP requests.

Our `MyTimeout` example is very simple. Let's say we've built a collaborative document editor called "Boogle Docs." Our editor should handle multiple users on one document and show changes in real time. Whenever a user makes a change or moves their cursor, other users should see this. One way to achieve this (not the most efficient, but simple to understand) is by sending HTTP GET requests at intervals to fetch other users' data from a server. We’ll also need to send POST requests to update our own data. If HTTP requests blocked the Call Stack, our website would render at a painfully slow rate, and we'd lose all of our users!

Your browser provides the Javascript runtime with an additional thread (not necessarily just one thread, but we'll think of it that way for simplicity) called the **WebAPI** thread. We can use this thread to handle operations that would typically block the Call Stack. Whenever we encounter a function classified as asynchronous (like `setTimeout` or `fetch`), we send it to this new thread and continue processing the rest of our synchronous code. This frees up the Call Stack for high-priority processes, such as rendering frames.

The WebAPI thread resolves these asynchronous functions (like the 5-second timeout) and queues up their Callbacks into a **Callback Queue**. The Callback Queue constantly checks the Call Stack to see if it's empty. As soon as the Call Stack is empty, it pushes the next Callback function onto the stack. This is the **Event Loop**. The Event Loop prioritizes synchronous operations over asynchronous ones, as we've just demonstrated.

How can we use the results of an asynchronous operation? With `MyTimeout`, we used a Callback function. After a set amount of time, our Callback function will be executed. What if we need to execute another `MyTimeout` after the first one? Then we pass the next `MyTimeout` as a Callback to the first `MyTimeout`. If this pattern continues, with maybe 10 timeouts, we end up with a huge nested mess of code. This is referred to as **Callback Hell**. Promises resolve this issue in Javascript by providing a new way of handling Callbacks.

When you create a new Promise, you pass a Callback function into the constructor. This function has two arguments: `resolve` and `reject`, both are Callback functions. Within this Promise, we might use `setTimout`. We’ll pass a Callback function into the `setTimeout` that performs some operations after the time is done, then calls the `resolve` callback at the end. If you'd like to pass the result of these operations to the next task, you can pass it into the `resolve` function.

We can store our promise in a variable, say `promise`. We define the resolve Callback by passing it to `promise.then`. If we passed a result to `resolve`, we can access it as an argument in the Callback that we pass to `promise.then`. The `.then` method can be used multiple times on our `promise`. It stores each Callback in an array and the `resolve` Callback iterates through each one. This means each Callback will be executed sequentially as soon as `resolve` is called.

The `promise.then` method returns a Promise itself, enabling promise chaining. The promise which `.then` returns has `resolve` and `reject` Callbacks. You define their behavior by adding more `.then` calls to the chain.

If the return value of a `.then` call is not a Promise, the value is passed directly to the next `resolve` or `reject` Callback (assuming a value is defined, and assuming `resolve` or `reject` have been defined, meaning there's at least one more `.then` call remaining in the chain). If the return value of a `.then` call is a Promise, then the `resolve` and `reject` Callbacks are passed into that returned Promise’s `.then` method.

This is an example of a Promise chain:
```js
new Promise(function(resolve, reject) {
  setTimeout(function () {
    resolve(5);
  }, 1000);
}).then(function(res) {
  console.log(res);
  return new Promise(function(resolve, reject) {
    setTimeout(function () {
      resolve(res + 5);
    }, 1000);
  });
}).then(function(res) {
  console.log(res);
  return new Promise(function(resolve, reject) {
    setTimeout(function () {
      resolve(res + 5);
    }, 2000);
  });
}).then(function(res) {
  console.log(res);
  return new Promise(function(resolve, reject) {
    setTimeout(function () {
      resolve(res + 5);
    }, 3000);
  });
});
```

Here's an example implementation of the Promise class:
```js
class MyPromise {
  constructor(executor) {
    this.state = 'pending';
    this.value = undefined;
    this.reason = undefined;
    this.onFulfilledCallbacks = [];
    this.onRejectedCallbacks = [];

    const resolve = (value) => {
      if (this.state === 'pending') {
        this.state = 'fulfilled';
        this.value = value;
        this.onFulfilledCallbacks.forEach(callback => callback(value));
      }
    };

    const reject = (reason) => {
      if (this.state === 'pending') {
        this.state = 'rejected';
        this.reason = reason;
        this.onRejectedCallbacks.forEach(callback => callback(reason));
      }
    };

    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  then(onFulfilled, onRejected) {
    return new MyPromise((resolve, reject) => {
      const handleFulfilled = (value) => {
        try {
          const result = onFulfilled(value);
          if (result instanceof MyPromise) {
            result.then(resolve, reject);
          } else {
            resolve(result);
          }
        } catch (error) {
          reject(error);
        }
      };

      const handleRejected = (reason) => {
        try {
          const result = onRejected ? onRejected(reason) : reason;
          if (result instanceof MyPromise) {
            result.then(resolve, reject);
          } else {
            reject(result);
          }
        } catch (error) {
          reject(error);
        }
      };

      if (this.state === 'fulfilled') {
        handleFulfilled(this.value);
      } else if (this.state === 'rejected') {
        handleRejected(this.reason);
      } else {
        this.onFulfilledCallbacks.push(handleFulfilled);
        this.onRejectedCallbacks.push(handleRejected);
      }
    });
  }

  catch(onRejected) {
    return this.then(null, onRejected);
  }
}
```
