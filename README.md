## [ABA Problem](https://en.wikipedia.org/wiki/ABA_problem)

The ABA problem is a subtle challenge that can arise in multithreaded programming when dealing with shared memory. It occurs during synchronization, specifically when relying solely on a variable's current value to determine if data has been modified. 

The ABA problem occurs when multiple threads (or processes) accessing shared data interleave. Below is a sequence of events that illustrates the ABA problem:

1. Process $P_1$ reads value A from some shared memory location.

2. $P_1$ is preempted, allowing process $P_2$ to run.

3. $P_2$ writes value B to the shared memory location.

4. $P_2$ writes value A to the shared memory location.

5. $P_2$ is preempted, allowing process $P_1$ to run.

6. $P_1$ reads value A from the shared memory location.

7. $P_1$ determines that the shared memory value has not changed and continues.

### Example

```cpp
#include <atomic>
#include <iostream>
#include <stdexcept>
#include <windows.h>

class __StackNode
{
public:
    const int value;
    bool valid = true;  // for demonstration purpose
    __StackNode *next;

    __StackNode(int value, __StackNode *next) : value(value), next(next) {}
    __StackNode(int value) : __StackNode(value, nullptr) {}
};

class Stack
{
private:
    std::atomic<__StackNode *> _head_ptr = nullptr;

public:
    /** Pop the head object and return its value */
    int pop(bool sleep)
    {
        while (true)
        {
            __StackNode *head_ptr = _head_ptr;
            if (head_ptr == nullptr)
            {
                throw std::out_of_range("Empty stack");
            }

            // For simplicity, suppose that we can ensure that this dereference is safe
            // (i.e., that no other thread has popped the stack in the meantime).
            __StackNode *next_ptr = head_ptr->next;

            if (sleep)
            {
                Sleep(200); // interleave
            }

            // If the head node is still head_ptr, then assume no one has changed the stack.
            // (That statement is not always true because of the ABA problem)
            // Atomically replace head node with next.
            if (_head_ptr.compare_exchange_weak(head_ptr, next_ptr))
            {
                int result = head_ptr->value;
                head_ptr->valid = false; // mark this memory as released
                delete head_ptr;
                return result;
            }
            // The stack has changed, start over.
        }
    }

    /** Push a value to the stack */
    void push(int value)
    {
        __StackNode *value_ptr = new __StackNode(value);
        while (true)
        {
            __StackNode *next_ptr = _head_ptr;
            value_ptr->next = next_ptr;
            // If the head node is still next, then assume no one has changed the stack.
            // (That statement is not always true because of the ABA problem)
            // Atomically replace head with value.
            if (_head_ptr.compare_exchange_weak(next_ptr, value_ptr))
            {
                return;
            }
            // The stack has changed, start over.
        }
    }

    void print()
    {
        __StackNode *current = _head_ptr;
        while (current != nullptr)
        {
            std::cout << current->value << " ";
            current = current->next;
        }
        std::cout << std::endl;
    }

    void assert_valid()
    {
        __StackNode *current = _head_ptr;
        while (current != nullptr)
        {
            if (!current->valid)
            {
                throw std::runtime_error("Invalid stack node");
            }
            current = current->next;
        }
    }
};

DWORD WINAPI thread_proc(void *_stack)
{
    Stack *stack = (Stack *)_stack;
    stack->pop(false);
    stack->pop(false);
    stack->push(40);

    return 0;
}

int main()
{
    Stack stack;
    stack.push(10);
    stack.push(20);
    stack.push(30);
    stack.push(40);

    HANDLE thread = CreateThread(
        NULL,         // lpThreadAttributes
        0,            // dwStackSize
        &thread_proc, // lpStartAddress
        &stack,       // lpParameter
        0,            // dwCreationFlags
        NULL          // lpThreadId
    );

    stack.pop(true);

    WaitForSingleObject(thread, INFINITE);
    CloseHandle(thread);

    stack.print();
    stack.assert_valid();  // randomly crash

    return 0;
}
```

In the example above, 2 threads are scheduled to run concurrently. To promote the occurrence of the ABA problem, the main thread calls `stack.pop(true)` with `Sleep(200)` to suspend its execution, allowing other threads to run. Before interleaving the main thread, the local pointer variable `head_ptr` and `next_ptr` points to `40` and `30`, respectively. We can assume that while the main thread is sleeping, the second one completes its operations (2 `pop`s and 1 `push`), transforms the stack into `[10, 20, 40]` and deallocates the memory of the node `30`. Afterwards, the main thread resumes its execution, where the method call `_head_ptr.compare_exchange_weak(head_ptr, next_ptr)` returns `true` (since the top of the stack is still `40`). It then sets `next_ptr` (which is a pointer to `30`) as the top of the stack. But since this memory is deallocated, it is unsafe to perform the assignment. Thus running the program above will randomly crash at the line `stack.assert_valid();`

It is worth noticing that since the memory pointed by `next_ptr` may not be touched anyway after deallocation, assigning `next_ptr` in the main thread still appears to work correctly (in this example, we have to clear a flag `valid` to mark the node as deallocated). Accessing freed memory is undefined behavior: this may result in crashes, data corruption or even just silently appear to work correctly.

### Real-world implications

### Addressing the ABA Problem

Several approaches can help mitigate the ABA problem:

* **Timestamps:** By associating a version number or timestamp with the data, threads can ensure the value hasn't changed between reads. An inconsistency arises if the version retrieved during the second read doesn't match the version associated with the initial value.
* **Locking mechanisms:** Utilizing synchronization primitives like mutexes or spinlocks ensures exclusive access to the shared memory location during critical sections. This prevents other threads from modifying the data while the first thread is performing its read-modify-write operation.
* **Atomic operations:** In specific scenarios, employing atomic operations that combine read and write into a single, indivisible step can eliminate the window of vulnerability between reads. However, atomic operations might not be suitable for all situations.

By understanding the ABA problem and implementing appropriate synchronization techniques, developers can ensure the integrity of data in multithreaded environments.

## Sleeping Barber Problem

The Sleeping Barber problem is a classic synchronization problem that illustrates the challenges of process management, specifically around CPU scheduling, resource allocation, and deadlock prevention.

The scenario is as follows:
* A barber works in a barbershop with one barber chair and a waiting room with several chairs.
* If there are no customers, the barber goes to sleep.
* When a customer arrives, if the barber is asleep, the customer wakes up the barber.
* If the barber is busy, the customer sits in the waiting room.
* If the waiting room is full, the customer leaves.

### Example

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::mutex mtx;
std::condition_variable cv_barber;
std::condition_variable cv_customer;
std::queue<int> customers;
const int NUM_CHAIRS = 5;
bool done = false;

void barber()
{
    std::unique_lock<std::mutex> lk(mtx);
    while (!done)
    {
        while (customers.empty())
        {
            // Barber goes to sleep if no customers
            std::cout << "Barber is sleeping." << std::endl;
            cv_barber.wait(lk);
        }
        // Barber is cutting hair
        std::cout << "Barber is cutting hair of customer " << customers.front() << std::endl;
        customers.pop();
        // Notify waiting customers
        cv_customer.notify_one();
    }
}

void customer(int id)
{
    std::unique_lock<std::mutex> lk(mtx);
    if (customers.size() < NUM_CHAIRS)
    {
        // Customer sits in waiting room
        customers.push(id);
        std::cout << "Customer " << id << " is waiting." << std::endl;
        // Wake up the barber if sleeping
        cv_barber.notify_one();
        // Wait for the barber to be ready
        cv_customer.wait(lk, [&]{ return customers.front() == id; });
    }
    else
    {
        // Waiting room is full
        std::cout << "Customer " << id << " leaves as waiting room is full." << std::endl;
    }
}

int main()
{
    std::thread barber_thread(barber);
    std::thread customers_thread[NUM_CHAIRS];

    // Create customer threads
    for (int i = 0; i < NUM_CHAIRS; ++i)
    {
        customers_thread[i] = std::thread(customer, i+1);
    }

    // Join customer threads
    for (auto& th : customers_thread)
    {
        th.join();
    }

    // End of the day
    done = true;
    cv_barber.notify_one();
    barber_thread.join();

    return 0;
}
```

In the example above:
* The barber thread waits for customers and cuts their hair.
* Customers arrive, check if there’s space in the waiting room, and either wait or leave.
* Semaphores (`cv_barber` and `cv_customer`) are used for synchronization.

### Real-world implications

* **Customer Service Queues:** Similar to the barbershop scenario, customer service centers often have a limited number of representatives to handle a fluctuating number of customer calls or requests. The challenge is to efficiently manage the queue so that customers are served in a timely manner without overwhelming the service reps.
* **Computer Operating Systems:** In operating systems, managing multiple processes that need access to limited CPU time or memory resources is akin to the Sleeping Barber problem. The OS must ensure that each process gets a fair chance to execute without causing deadlock or starvation.
* **Database Access:** Multiple applications or users trying to access and modify a database can lead to concurrency issues. The Sleeping Barber problem helps in designing systems that prevent conflicts and ensure data integrity when multiple transactions occur simultaneously.
* **Airport Runway Scheduling:** Air traffic controllers must manage the takeoffs and landings of aircraft on a limited number of runways. The principles derived from the Sleeping Barber problem can help in creating schedules that maximize runway usage while maintaining safety.
* **Hospital Emergency Rooms:** In healthcare, particularly in emergency rooms, patients must be attended based on the severity of their condition. The Sleeping Barber problem’s solutions can inform the design of triage systems that manage patient flow effectively.
* **Web Server Management:** Web servers handling requests for web pages or services must manage their threads to respond to simultaneous requests. The Sleeping Barber problem provides insights into how to balance load without causing long wait times or server crashes.

### Addressing the Sleeping Barber Problem

Several approaches can help mitigate the Sleeping Barber problem:
* **Semaphores:** The most common solution involves using semaphores to coordinate access to resources. Semaphores can be used to signal the availability of the barber (or resource) and the waiting chairs (queue space). This ensures that customers (or tasks) are served in the order they arrive and that the barber (or resource) is not overwhelmed.
* **Mutexes:** Mutexes are used to ensure mutual exclusion, particularly in critical sections of the code where variables are accessed by multiple threads. This prevents race conditions and ensures that the system’s state remains consistent.
* **Condition Variables:** These are used in conjunction with mutexes to block a thread until a particular condition is met. For example, a barber thread might wait on a condition variable until there is a customer in the waiting room.
* **Monitor Constructs:** Monitors are higher-level synchronization constructs that provide a mechanism for threads to temporarily give up exclusive access in order to wait for some condition to be met, without risking deadlock or busy-waiting.
* **Message Passing:** In distributed systems, message passing can be used to synchronize processes. This involves sending messages between processes to signal the availability of resources or the need for service.
* **Event Counters:** These can be used to keep track of the number of waiting customers and to signal the barber when a customer arrives. This helps in managing the queue and ensuring that no customer is missed.
* **Ticket Algorithms:** Ticket algorithms can be used to assign a unique number to each customer, ensuring that service is provided in the correct order. This is similar to taking a number at a deli counter.

By understanding the Sleeping Barber problem and implementing appropriate synchronization techniques, developers  learn how to efficiently manage limited resources in concurrent systems. 
