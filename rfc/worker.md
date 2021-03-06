- Start Date: 05.02.2015

# Summary

Stabilize Framework Worker API

# Motivation

To prevent spontaneous API changes just because someone wants it.

# Detailed design

Worker API provides minimal functionality to be able to develop Cocaine applications. This includes:

 - Handling all internal work (with network and the Protocol) transparently to the user.
 - Providing convenient and simple to understand API for handling incoming requests.

## Worker

Worker is designed to be noncopyable class, which instances should be created on the main thread's stack.

```cpp
int main(int argc, char** argv) {
    worker_t worker(options_t(argc, argv));
    ...
    // Register all event handlers required.
    ...
    return worker.run();
}
```

It requires an instance of `options_t` class, which is configured automatically via command-line arguments (which is provided by the Runtime).

There are also several optiona, which can be configured manually, like worker threads count.

Internally worker manages with single IO thread, which is unavailable to users, and with isolated thread pool with event loops run in each of them, but you shouldn't care about this.
After `worker.run()` is invoked, the current control flow blocks until the worker is forced stopped via the Protocol's terminate message. Any other cases, in which the event loop stopps are treated as unexpected termination (and probably will generate core dumps). It can be:

 - Uncaught user exceptions, which reached some thread's event loop.
 - Designed to be uncaught exceptions, when the Runtime forces to stop the worker.
 - Any other way to stop the application (such as `std::terminate`, `std::exit` etc.).

The worker API:

```cpp
class worker_t {
public:
    /*!
     * Construct worker object using given arguments.
     *
     * \throw std::argument_error on any invalid input data, such as non-existing command-line arguments.
     */
    worker_t(options_t options);

    /*!
     * Returns an instance of service manager, which shares threads with the worker's one.
     */
    service_manager_t& manager() noexcept;

    /*!
     * Registers the given event handler with the worker.
     */
    template<typename F>
    void on(std::string event, F fn);

    /*!
     * Registers the given tagged event handler with the worker. Tagged handler (like http)
     * provides additional information about handler, like its arguments.
     */
    template<class T>
    void on(std::string, typename handler_traits<T>::function_type fn);

    /*!
     * Enters the main event loop and waits until all asynchronous operations completes.
     * Returns the value that indicates on the error occurred or 0 if everything is okay.
     * Can throw.
     */
    int run();
};
```

## Event Handlers

Event handlers is a functional objects, that handle incoming events, obviously. Depending on handler type its arguments may vary.

### Common Event Handler

The commonnest event handler is a functional object, that accepts sender and receiver pair and returns nothing.

```
worker.on("ping", [](worker::sender tx, worker::receiver rx){
    tx.write(*rx.recv().get()).get();
});
```

If the sender has gone out of the scope, and there aren't any pending futures chained with it, a close event will be automatically sent.

Any uncaught exceptions in the event handler are threated as programming error and will be propagated down the stack until reached worker thread, which calls `std::terminate` and generate core dump eventually. This allows to save stacktrace clean, which is useful to the further debugging.
If you want to keep the worker alive no matter what, you can always provide your own functional object as a handler, which wraps internally all potentially dangerous calls with `try/catch` blocks.

### HTTP Event Handler

The second kind of handlers - are HTTP handlers.

As you can see, the sender and receiver arguments are completely different from the common event handler's arguments. After calling the `rx.recv()` you will get a future with a pair of fetched http information, like url, headers, but not the body and a streaming receiver. Througn the receiver obtained you can fetch body chunks until `boost::none` received.

On the other side there is sender, which works completely specularly.

The following code should make it clear:

```cpp
typedef http::event<> http_t;
worker.on<http_t>("http", [](http_t::fresh_sender tx, http_t::fresh_receiver){
    // This is the place for initialization.
    ...

    http_request rq;
    receiver<streaming_tag<std::string>> rx;
    std::tie(rq, rx) = fresh_rx.recv().get();

    // Work with the request object obtained.
    auto url = rq.url();
    auto headers = rq.headers();
    ...

    // Here we can fetch the body at last until boost::none received.
    boost::optional<std::string> chunk = rx.recv().get();

    http_response rs;
    rs.code(200);
    rs.headers().set(...);
    ...
    // Fill the response.
    ...

    // Send the initial response chunk, obtain streaming sender and send chunks until close.
    sender<streaming_tag<std::string>> tx = ftx.send(response).get();
    tx.send("le body").get();
});
```

The distinction between fresh and streaming sender/receiver was made intentionally to make impossible to fetch headers while the stream is in streaming state and vice versa.

# Drawbacks

Why should we *not* do this?

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

What parts of the design are still TBD?

