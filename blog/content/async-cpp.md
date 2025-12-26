+++
title = "Asynchronous Multi-Thread Design Pattern with C++"
date = 2024-10-19
draft = false
[taxonomies]
  tags = ["Concurrent Programming", "C++"]
[extra]
  toc = true
    keywords = "Concurrent Programming, C++"
+++

This post does NOT discuss existing high-level async libraries or frameworks in C++. Instead, we explore the fundamental concepts behind the **Asynchronous Multi-Thread Design Pattern** and provide a raw implementation example.

Unlike Rust, C++ has no built-in language support for asynchronous channels. Consequently, developers new to C++ often struggle to plan their multithreaded architecture effectively.

## The Asynchronous Design Pattern

In traditional **synchronous programming**, parent and child threads are usually interlocked via tight synchronization mechanisms, such as state machines. They often share direct access to the same memory for communication.

The problem with the synchronous pattern is the requirement for meticulous step-by-step synchronization. One mistake can lead to data races or deadlocks. Moreover, as software grows, the synchronization complexity increases exponentially. Because of these serious scalability issues, most modern high-performance systems have moved away from this design.

The **asynchronous pattern** decouples the execution. The parent thread does not interrupt the child thread; instead, they interact strictly through I/O structures. A typical workflow looks like this:

1.  **Task Submission:** The parent thread sends a task data structure to one (or a group of) child threads.
2.  **Processing:** A child thread unblocks upon receiving data, processes the task, and generates a result.
3.  **Result Reporting:** The child thread sends the result back to the parent and waits for the next task.
4.  **Collection:** The parent thread continues its own work. When it requires the result, it checks the incoming queue (blocking if necessary) to retrieve it.

You might wonder: *Where are the synchronization steps?*

In this pattern, synchronization is handled implicitly by the data structures (queues). Adding complex state management to child threads is generally considered bad practice as it re-introduces unnecessary complexity. Typically, parent and child threads communicate through **FIFO (First-In-First-Out) queues**, allowing data to be buffered while the reader is busy.

## Example C++ Program

Let's look at a concrete example. The main thread wants to distribute math calculations to a group of worker threads (assuming a simple `square()` calculation).

First, we define a thread-safe FIFO queue:

```cpp
```cpp
template <class T>
struct WorkerChannel_T {
    std::queue<T> queue_;
    std::mutex mutex_;
    std::condition_variable cv_;

    // Thread safe push
    void push(T data) {
        std::unique_lock<std::mutex> lk(mutex_);
        queue_.push(data);
    }

    // Thread safe pop (blocking)
    T pop() {
        std::unique_lock<std::mutex> lk(mutex_);
        // Wait until queue is not empty
        cv_.wait(lk, [this] { return !this->queue_.empty(); });
        T result = queue_.front();
        queue_.pop();
        return result;
    }

    void notify() { cv_.notify_all(); }
};
```

> Note: Both `push()` and `pop()` are protected by mutex_, making instances of this struct safe to share between parent and child threads.

Next, we define the worker thread function:

```cpp
void square_worker(WorkerChannel_T<float>& in_ch,
                   WorkerChannel_T<float>& out_ch) {
    while (true) {
        float in_num = in_ch.pop();
        
        // Check for sentinel value to stop the thread
        if (std::isnan(in_num)) {
            break; 
        } else {
            // Perform calculation
            float result = std::pow(in_num, 2);
            
            // Simulate heavy work (100ms)
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            
            out_ch.push(result);
            out_ch.notify();
        }
    }
}
```

Finally, the main thread orchestrates the workers:

```cpp
int main() {
    WorkerChannel_T<float> in_ch;
    WorkerChannel_T<float> out_ch;
    std::vector<std::thread> worker_vec;

    // 1. Create 8 worker threads
    for (int i = 0; i < 8; i++) {
        worker_vec.push_back(
            std::thread(square_worker, std::ref(in_ch), std::ref(out_ch)));
    }

    // 2. Post tasks
    for (int i = 0; i < 100; i++) {
        std::cout << "Post number: " << float(i) << std::endl;
        in_ch.push(i);
    }
    
    // Notify workers that data is available
    in_ch.notify();

    // 3. Retrieve results
    for (int i = 0; i < 100; i++) {
        float result = out_ch.pop();
        std::cout << "Get result: " << result << std::endl;
    }

    // 4. Stop all worker threads using NAN as a sentinel
    for (size_t i = 0; i < worker_vec.size(); i++) {
        in_ch.push(NAN);
    }
    in_ch.notify();

    // 5. Join all threads
    for (size_t i = 0; i < worker_vec.size(); i++) {
        std::cout << "Join worker[" << i << "]" << std::endl;
        worker_vec[i].join();
    }

    return 0;
}
```

We share the same `WorkerChannel_T<float>` instance (the input channel) among multiple workers. Because the queue is locked during access, only one worker can `pop()` a specific task, ensuring tasks are distributed evenly without duplication.

You can find the full executable code on [Godbolt.org](https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1DIApACYAQuYukl9ZATwDKjdAGFUtAK4sGIAMxcpK4AMngMmAByPgBGmMQgAGyJpAAOqAqETgwe3r4BQemZjgJhEdEscQnJtpj2JQxCBEzEBLk%2BfoG19dlNLQRlUbHxSSkKza3t%2BV3j/YMVVaMAlLaoXsTI7Bzm/uHI3lgA1Cb%2BbsgsTAQIJ9gmGgCCO3sHmMenaAz4DQD6AG4teCYMXoNzujzMuwY%2By8RxObic42ImFYoIeTyhLzebhYXgImFUqPBkOhsNOAEcvJhKYT0STXnCrkimOgaRDnjD6adfpgHCQaQ88SxUgY8Vj9kwFApDgAVQmIrwOQ4AdRIAGt4m4EIYIrRvtLjgB2KwPQ6mw7jdAgEAUqmYOGy/zYQ42ynfE7G%2B5m80ES0gHF41SHf34t3%2BD1ei1Wj5fbJ/AFA%2BiHZC/UPhs0AenTMoQTPQ5qYVDtJrNv1QeDzqS8CgQEH16EuTCWhrTXtNkZAXgYeBt31oqGQqsOtFVEGDqm%2BS3dYNbZpdmG%2BADpK9WIPXmpOw9OzSYDQARLemzPZ3P5wsHmWHdKpCBNnct1vtzvd119gdDkdjidT4sz5OLgDuTCEBAw6kMcACsFhXHgCgmOBu7NocSIEOsDCHGAYDQQoAC0NxzoumBCgQACeN7uoau4bveXr6kiCheLQBBvAh%2BELlQxCyGRm4/q2rFXlx1FmshqFIZg9GMd%2Bnrbnu56luWhwMKgjhUKRt5GkmKYLopynEd8Yi0AJFFgju%2B7cY8DxyXmCgUi087/mq8QQCqxDqsQmranUepwlQfaXKCZiJIc4TfMgCCkOeM6Raazmue5DA6l5pw%2BagfmOuYgVrAQIUIGp97/ggdCvBABDEJSuURWayWXEFDDfAwPjMTV2VLqg15URVpp4FQhwQO2MHMAwEDBfVLBLOVPFRTETKquRhxHtgnyHIyyIshN0kIXUSjNh1rbDQ1JwIe26T/kNtUjWBZjtWtM5HkIeA4iKrxcBoGgsFKjhsEmAj0UR2Q7RGPpWlh3zLcyVoKPQmCpN8/DEL1gMoDmAioFaLB0LQME8gI6AKBAz0aGNklRV6mXNcuNZ7aNRPE6apOhVpSldap1MziZ55s2iMlog84RMRc4Q3ttPExRqWrxZ5DpuFVBA3E1oUs8qDluWLCWS9Lst09cZlekwuKoIc9kufEfw8o17bcryxBwu2oMso6AnnkebhMqKAAcBtK0tOYrbBPGwz1vNBY1GjkXgWKHK7ofWNY41STOhuuSbyBLlWCDfDETADhA/1mjb3vMr1NlIt8CfxGB7ZIlQp3ZSs3q%2BpXECa2NV1x0ZPFHsoGRMc0Ciqr7rf%2B6dTFhwdhwh2GQdwoc%2BNR5YMdC63AO%2BmguJYlP5hmJ34wKcMCTHGYZhr6chzS0Nt6nFP7auLQCtesF9Pk2fCsc63R6RIzKmHG4ABq573wgDMdIOz9iQAOghJ7%2BAQuPCwEC3DTxerPKwlhY5RWlocbQjVNYtTarfXOCMV5MThOvA%2BABxTATE6IMQICAfeh8iHH1Ptoc%2BbhL4I2vs/Lmr8sxNFaocfSHsjbEC9rmfuXpB6BxHpAseodw6l2IEnBcmQABemBDJ4GjsgheUV/4pxXJEe4kQW5ehfnfWq9NtJM2AVww4AApMsaF%2BG2wNggRgBtXjoAEEWAeoCh4QKgTIqeciFHKNUVRIKGiLAoMiu2AhR84EbzseEARrk4JILoRfY%2BkiWHHw3nBfcB84l1ytOw7WkUgkW1Seo%2BCC5tD2KscYzhXphLEDQtA4ye4OArFoJwcCvA/DcF4CjDgLC56WHNGsDY9IIQ8FINQjgWgxqkFVAEfwC5/DrI2ZszZKRukcEkH0zQgzOC8AUCADQszDkrDgLAJAaAhSFTIBQCAdzUgPJAMAKQZg%2BB0DxMQU5EAYiHNIDEcILRiKcBmSC5gxBiIAHkYjaB5HMmZdy2CCFhQwWg4L5m8CwDELwwA3D6VOQM0gWALhGHEDisleAkQODwNyElWhgiqB5LiLYzLeZ1CBRjKaYKPBYCBSVe6ELeDcmIDEDImBdyEUMMADGRhLl8AMMABQ388CYH/LC1IjBRUyEECIMQ7ApD6vkEoNQQLdBBAMIq0woybC8tOZAFYrUGgksGeK4g5YxLwAgMwNgIALTitIP8bw7A7VIIsGYAmKw7BIuyC4T4Uw/BBFCOEIYlQRiFAyFkAQya9BFFzQweYu89BxvpQIPokxPAdDLXUeNlaJgDHTQsLNtgm35qCLMVoJbM0JC4LGiZmwJBdJ6Qc6lQzDiqFdokbCiRJCHGAMgZA09JALkPhAXAhBQE7AHbwOZCyVjLK4OBBckgACcGgzCu2jZIF6d7z1cANPoTg%2BzSD9OZUMk5ZyLk4quTARAIBMqVgIOQSgLyHmRFYFsads752LuXau9dvBMBfBIOWPQ/ADWiHECarDZqVDqGpVa0g/5iBMFSKK0dHBenvqBUM2FuIQOHFQN1WDc6F1LpXVIddPUPD3PoEI3dSx92XKPSAcCXAFxPsSP4V2khEgBUU%2BBA00hdlvo/Ucjg37zkHs6dRsw47P3HN/YekN8RMjOEkEAA).

### Comparison to Async Frameworks
If you are familiar with standard async frameworks, you will notice similarities:

* `in_ch.push()` followed by `notify()` behaves similarly to creating a Promise or Future.
* `out_ch.pop()` behaves like an Await. The parent thread blocks on this function call, waiting for results to become available.

We effectively created a tiny async framework using only `std::thread` and standard library synchronization primitives! In production frameworks, the executioner is often a thread pool or coroutines rather than raw system threads, but the logic remains the same.

### Scaling the Pattern

As shown above, sharing a FIFO channel among multiple workers is straightforward. Since the queue is mutex-protected, it is safe for all workers to block on one input channel.

However, complexity arises on the consumer side. In our example, the parent thread blocks on a single output channel. What if the parent needs to watch multiple channels simultaneously?

#### Strategy 1: Serialized Wait (Scatter-Gather)

The most straightforward approach is for the parent to wait for results one by one. While this may sound inefficient, it is actually quite powerful. The parent only waits as long as the slowest task takes to complete.

```cpp
result1 = out1_ch.pop();
result2 = out2_ch.pop();
result3 = out3_ch.pop();
```

Once this sequence finishes, the parent is guaranteed to have all data ready for the next processing step. This is ideal when the parent needs a complete set of data before proceeding (often called a "Barrier").

#### Strategy 2: Wait for First Available

Sometimes, you want the parent thread to react immediately when any result is available, regardless of the order.

The simplest solution is to use a single shared output channel that supports multiple data types (using `std::variant`) or a common base structure. All child threads push to this single "Result Channel." The parent calls `pop()` on this single channel and uses `std::visit` to determine which worker replied and how to process the data.

Alternatively, you can implement a dedicated "Notification Channel." When a child thread finishes, it pushes its ID to the Notification Channel. The parent watches this channel to know which specific data channel is ready to be read.
