# ESP32-Fast-external-IRQs
## How to receive more than 3 Million Interrupts per Second

I like the ESP8266. With 160 MHz it is fast! And you can do nearly all your Arduino stuff - but (at least) 10 times faster.

But when it comes to WiFi (which comes built in) you have a problem:<br>
When you implement a WebServer, it uses callbacks to handle the http-transfers - and they can use 4 ms or more.
No more high speed tasks with WiFi enabled :(

Maybe the reasons behind the scene are different, but if I was the inventor of the ESP8266, I would think:.<br>
"It must be great, if we have a 2. core. The first core can do all the WiFi stuff and some RTOS-tasks.<br>
In the second core we can do some high speed functions. If it is possible to stop taskswitching for the second core only<br>
we have a superfast Arduino - using all the libraries, which are not done for an RTOS.<br>

Then the ESP32 came out - and well, you can use it in exactly this manner!<br>

But lets start with the beginning:<br>
In the examples you can find  the ***gpio_exmple***<br>
Here we have (selfmade) interrupts. The interrupt service routine then unblocks an RTOS task with ***xQueueSendFromISR***.<br>

As allways  I want to see the timings on a scope, so I added pulses to show:
1) When is the interrupt generated
2) When does the interrupts service routine gets the interrupt
3) When does the RTOS-task gets the signal.

The pulses (high,low) are done at register level and need about 60 ns (nanosecends)
 
This is the result:
 ![image1](./image1.jpg?raw=true "gpio example")
