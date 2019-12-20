# Four ways to Blinky

## Loop, Ticker, Threads & EventQueues

In this tutorial, we will take the well-known example from the Mbed community, the [mbed-os-example-blinky](https://github.com/ARMmbed/mbed-os-example-blinky) and we will gradually modify it to demonstrate the different concepts provided by Mbed to enable multitasking, that is the ability to run multiple tasks to run concurrently. The tasks can either be scheduled to run due to a system event (e.g. on a system interrupt) or manually started by the user. The concepts should be pretty familiar for any developer who has previously done any concurrency work in some programming language, but we will highlight the specific idiosyncrasies when those concepts are applied to a constrained embedded environment such as Mbed.


> NOTE: In our examples we are using [STMicroelectronics DISCO-L475VG-IOT01A](https://os.mbed.com/platforms/ST-Discovery-L475E-IOT01A/) board, but you can use any board supported by Mbed, albeit with some modifications on `LED` and `BUTTONS` constants to correctly point to yout specific board characteristics. The [Mbed boards page](https://os.mbed.com/platforms/) provides detailed descriptions and architectural diagrams for each board so you can use it as a reference.

Let's start first with the most simplistic example that blinks an `LED`:

## Loop

Using your favourite Mbed tool, either [mbed-cli](https://os.mbed.com/docs/mbed-os/v5.15/quick-start/compiling-the-code.html) or our recommended approach, [Mbed studio](https://os.mbed.com/studio/) to import the code from the [blinky-loop](https://github.com/cvasilak/mbed-os-example-blinky-loop) repository and flash in on the board.

Here is a snippet of the code that actually does the blinking:

```
#include "mbed.h"

// Blinking rate in milliseconds
#define BLINKING_RATE_MS             500

// define the Serial object
Serial pc(USBTX, USBRX);

int main()
{
    // Initialise the digital pin LED1 as an output
    DigitalOut led(LED1);
    
    while (true) {
        led = !led;
        pc.printf("Blink! LED1 is now %d\r\n", led.read());
        thread_sleep_for(BLINKING_RATE_MS);
    }
}
```

The code should be self-explanatory and it's basic enough to introduce the threading model of Mbed OS. As a multitasking operating system there are many threads running in the background in order to support system services as well as run the actual user code. A partial list of threads can be summerized in the following list:

- The `Main thread` that executes the application `void main()` function.
- The `Idle thread` that is scheduled to run when there are no other threads available to run. The job of this thread is to ensure the device is put into sleep and save processor cycles with the ultimate goal to preserve battery life.
- The `Timer thread` that schedules repetive (or one-shot) functions to be called.

Notice the call of the `thread_sleep_for()` function in our main code. It is executed within the context of the `Main   thread` and instructs the OS to suspend the execution of this thread, release the resources it currently holds and give control back to the OS. If there are no other threads, the OS will put the device to sleep to preserve battery life.

Let's now see a different example of `blinky` this time using a `Ticker` object that setups a function to be called repeatedly and at a specified rate. This example will help us highlight one important concept in Mbed, which is usually a source of confusion and results in runtime issues when programming with Mbed. 
 

## Ticker

Import the code from the [blinky-ticker](https://github.com/cvasilak/mbed-os-example-blinky-ticker) repository and flash it on the device.

Here is a snippet of the code:

```
#include "mbed.h"

// Blinking rate in milliseconds
#define BLINKING_RATE_MS             500

Ticker flipper;
DigitalOut led1(LED1);
DigitalOut led2(LED2);
 
 // define the Serial object
Serial pc(USBTX, USBRX);

void flip() {
    led2 = !led2;
    pc.printf("Blink! LED2 is now %d\r\n", led2.read());
}
 
int main() {
    led2 = 1;
    flipper.attach(&flip, 2.0); // the address of the function to be attached (flip) and the interval (2 seconds)
 
    // spin in a main loop. flipper will interrupt it to call flip
    while(1) {
        led1 = !led1;
        pc.printf("Blink! LED1 is now %d\r\n", led1.read());
        thread_sleep_for(BLINKING_RATE_MS);
    }
}
```

The code is similar to the previous example but this time we use the Ticker object to schedule a repetive function `flip()` to blink a second led (`LED2`) after two seconds. The program starts and after blinking of `LED1`, execution halts when trying to blink `LED2`.

Let's examine the error message:

```
Blink! LED1 is now 1
Blink! LED1 is now 0
Blink! LED1 is now 1
Blink! LED1 is now 0

++ MbedOS Error Info ++
Error Status: 0x80010133 Code: 307 Module: 1
Error Message: Mutex: 0x2000038C, Not allowed in ISR context
Location: 0x800A3E5
Error Value: 0x2000038C
Current Thread: rtx_idle  Id: 0x100018D0 Entry: 0x8007F81 StackSize: 0x380 StackMem: 0x10001958 SP: 0x20017EF0
For more info, visit: https://mbed.com/s/error?error=0x80010133&tgt=DISCO_L475VG_IOT01A
-- MbedOS Error Info --

= System will be rebooted due to a fatal error =
= Reboot count(=5) reached maximum, system will halt after rebooting
```

The reason of this can be traced on the fact that any code that runs in Mbed can be in two different context modes, either run within the context of `Interrupt mode` or in user `Thread mode` and there are specific restrictions on what code is allowed to do in each one. In the first case, the `Interrupt mode`, the developer should cater for:
	
-  the code should execute as fast as possible (without blocking) and return to `Thread mode`.
-  the code should not call library fuctions that are not designed to be called in `Interrupt mode`.

> The [Thread safety page](https://os.mbed.com/docs/mbed-os/v5.15/reference/thread-safety.html) in our documentation highlights the different API's that are  allowed be called in each mode and can serve as a reference during your development.

Coming back to the example above, you need to pay attention that the Ticker object schedules a function to be called in `Interrupt mode` and as such certain API fucntion won't work and will cause a system error. In our case the call to the Serial object method `printf` is not allowed in `Interrupt` mode and thus the system error we receive.

The concepts of `Interrupt` mode or `Thread` mode in which your code runs and what is allowed to call is crucial to understand and you should always consult the documention to avoid unexpected runtime errors such as the one we faced in our sample code above.

Let's now rewrite the example above but this time utilizing Mbed's Thread API to schedule user Threads (which run in `Thread` mode) to achieve the blinking of both LED's.


## Threads

Import the code from the [blinky-thread](https://github.com/cvasilak/mbed-os-example-blinky-thread) repository and flash it on the device:

Here is a snippet of the code:

```
#include "mbed.h"

// Blinking rate in milliseconds
#define BLINKING_RATE_LED1_MS             500
#define BLINKING_RATE_LED2_MS            1000

DigitalOut led1(LED1);
DigitalOut led2(LED2);
Thread thread;

// define the Serial object
Serial pc(USBTX, USBRX);

void led2_thread() {
    while (true) {
        led2 = !led2;
        pc.printf("Blink! LED2 is now %d\r\n", led1.read());
        thread_sleep_for(BLINKING_RATE_LED2_MS);
    }
}
 
int main() {
    thread.start(led2_thread);
    
    while (true) {
        led1 = !led1;
        pc.printf("Blink! LED1 is now %d\r\n", led1.read());
        thread_sleep_for(BLINKING_RATE_LED1_MS);
    }
}
```

Here we have two threads running, one is the `Main` thread which within an infinite loop blinks repeatedly the first `LED1` and second using Mbed Thread API we create a second thread that blinks the second `LED2`. Notice, that since we are running in `Thread mode` our call to Serials `printf` function succeeds this time and no system error is generated.

Browsing the [Thread class](https://os.mbed.com/docs/mbed-os/v5.15/apis/thread.html) API documentation you will notice the similarities in method names with similar functionality offered by other programming languages threading constructs. This is deliberately done such as a newcomer developer to Mbed would feel right at home with the Threading API.
	 
Although dealing with thread creation and management can be powerfull, still requires a lot of carefully written code to cope with thread management and synchronization. In some use cases, a more simplistic API offering similar functionality with Threads but in a way that is more developer friendly and more familiar to recent advances in asyncronous programming can be preferrable. 

Welcome to the wonderful world of EventQueue's !


## EventQueue

The API documentation of [EventQueue class](https://os.mbed.com/docs/mbed-os/v5.15/apis/eventqueue.html) describes it with the following:

> The EventQueue class provides a flexible queue for scheduling events. You can use the EventQueue class for synchronization between multiple threads, or to move events out of interrupt context (deferred execution of time consuming or non-ISR safe operations).

In simplest terms, the EventQueue allows the developer to queue events (e.g. functions) to be run at a later time and importantly outside of the `Interrupt` mode we descibed earlier. All events are scheduled to run in user `Thread` mode.

Here is a snippet of code that schedules to call `printf` once, in two seconds and every second. Notice that since we are running in user `Thread mode` we are allowed to call those library functions. Calling those functions in `Interrupt` mode would have resulted in a system error as we show previously.
 

```
#include "mbed_events.h"
#include <stdio.h>

int main() {
    // creates a queue with the default size
    EventQueue queue;

    // events are simple callbacks
    queue.call(printf, "called immediately\n");
    queue.call_in(2000, printf, "called in 2 seconds\n");
    queue.call_every(1000, printf, "called every 1 seconds\n");

    // events are executed by the dispatch method
    queue.dispatch();
}
```

> The [EventQueue API documentation](https://os.mbed.com/docs/mbed-os/v5.15/tutorials/the-eventqueue-api.html) page and the [class reference page](https://os.mbed.com/docs/mbed-os/v5.15/apis/eventqueue.html) host more in-depth details of the internal operation of EventQueue as well as providing several examples of its usage.

Let's return to our blinky example and see how we can rewrite it to use the `EventQueue` class instead. Import the code from the [blinky-eventqueue](https://github.com/cvasilak/mbed-os-example-blinky-eventqueue) repository and flash it on the device:

```
#include "mbed.h"
#include "mbed_events.h"

DigitalOut led1(LED1);
InterruptIn sw(USER_BUTTON);
EventQueue queue(32 * EVENTS_EVENT_SIZE);
Thread t;

// define the Serial object
Serial pc(USBTX, USBRX);

void rise_handler_user_context(void) {
    pc.printf("rise_handler_user_context in context %p\r\n", osThreadGetId());
    pc.printf("Blink! LED1 is now %d\r\n", led1.read());
}

void fall_handler_user_context(void) {
    pc.printf("fall_handler_user_context in context %p\r\n", osThreadGetId());
    pc.printf("Blink! LED1 is now %d\r\n", led1.read());
}

void rise_handler(void) {
    // Execute the time critical part first
    led1 = !led1;
    // The rest can execute later in user context (and can contain code that's not interrupt safe).
    // We use the 'queue.call' function to add an event (the call to 'rise_handler_user_context') to the queue.
    queue.call(rise_handler_user_context);
}

void fall_handler(void) {
    // Execute the time critical part first
    led1 = !led1;
    // The rest can execute later in user context (and can contain code that's not interrupt safe).
    // We use the 'queue.call' function to add an event (the call to 'fall_handler_user_context') to the queue.
    queue.call(fall_handler_user_context);
}

int main() {
    // Start the event queue
    t.start(callback(&queue, &EventQueue::dispatch_forever));
    pc.printf("Starting in context %p\r\n", osThreadGetId());
    // The 'rise' handler will execute in interrupt context
    sw.rise(rise_handler);
    // The 'fall' handler will execute in interrupt context
    sw.fall(fall_handler);
}
```

In the `void main()` function we start the Event Dispatch Queue thread and we attach two interrupt handlers, `fall_handler()` and `rise_handler()` on a button press and release status respectively. Notice that both interrupt handlers will run in `Interrupt mode` and as such we need to be as fast as possible (critical path). For that reason a call to Serials `printf()` method is prohibited in this mode as we described in the previous section. For that reason, we split the critical and non-critical path to separate functions and with the aid of the `EventQueue` `call()` method we schedule the invocation of the non-critical path at user context (``Thread mode`` of the event dispatch queue thread itself).

## Conclusion

In this tutorial we demonstrated four different ways to blink a led and in the process, we exposed the different concepts provided by Mbed to support multitasking, `Threads` and `EventQueues`. Further we highlighted the critical concept of `Interrupt` and `Thread` context modes and how it can lead to confusion and runtime errors if the user is not aware in which context the code runs on. We suggest the interesting reader to visit the [Mbed OS fundamentals](https://os.mbed.com/docs/mbed-os/v5.15/reference/mbed-os-fundamentals.html) section on the Mbed documentation web site where the concepts of Threads and Thread safety is discussed in more detail, as well as visit the [Thread](https://os.mbed.com/docs/mbed-os/v5.15/apis/thread.html) and [EventQueue](https://os.mbed.com/docs/mbed-os/v5.15/apis/eventqueue.html) API documentation for a thorough description of those classes together with useful code snippets that demonstrate various features of the API.




