# Writing Universal Code

In a server-rendered app, most of our code will be "universal" - that is, it runs on both the server and the client. However, the two environments are quite different, so the behavior of our code will not be exactly the same. Here we will go over some differences that you need to be aware of.

## Data Reactivity on the Server

In a client-only app, every user will be using a fresh instance of the app in their browser. For server-side rendering we want the same: each request should have a fresh, isolated app instance so that there is no cross-request state pollution. Since each app is rendered only once, this means data reactivity is unnecessary on the server, so it is disabled by default. Disabling data reactivity also avoids the performance cost of converting data into reactive objects.

## Component Lifecycle Hooks

Since there are no dynamic updates, of all the lifecycle hooks, only `beforeCreate` and `created` will be called during SSR. This means any code inside other lifecycle hooks such as `beforeMount` or `mounted` will only be executed on the client.

## Access to Environment-specific APIs

Universal code cannot assume access to environment-specific APIs, so if your code directly uses browser-only globals like `window` or `document`, they will throw errors when executed in Node.js, and vice-versa.

For tasks shared between server and client but use different platform APIs, it's recommended to wrap the platform-specific implementations inside a universal API, or use libraries that do this for you. For example, [axios](https://github.com/mzabriskie/axios) is an HTTP client that exposes the same API for both server and client.

For browser-only APIs, the common approach is to lazily access them inside client-only lifecycle hooks.

Note that if a 3rd party library is not written with universal code in mind, it could be tricky to use it in an server-rendered app. You *might* be able to get it working by mocking some of the globals, but it would be hacky and may interfere with the environment detection code of other libraries.
