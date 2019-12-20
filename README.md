# ESP32-Fast-external-IRQs
## How to receive more than 3.000.000 Interrupts per Second

I like the ESP8266. With 160 MHz it is fast! And you can do nearly all your Arduino stuff - but (at least) 10 times faster.

But when it comes to WiFi (which comes built in) you have a problem:<br>
When you implement a WebServer, it uses callbacks to handle the http-transfers - and they can use 4 ms or more.
No more high speed tasks with WiFi enabled :(

Maybe the reasons behind the scene are different, but if I was the inventor of the ESP8266, I would think:.<br>
"It must be great, if we have a 2. core. The first core can do all the WiFi stuff and some RTOS-tasks.<br>
In the second core we can do some high speed functions. If it is possible to stop taskswitching for the second core only<br>
we have a superfast Arduino - using all the libraries, which are not done for an RTOS.<br>

***Then the ESP32 came out - and well, you can use it in exactly this manner!***<br>

But lets start with the beginning:<br>
In the espressif folder esp-id/fexamples you can find  the ***gpio_exmple***<br>
Here we have (selfmade) interrupts. The interrupt service routine then unblocks an RTOS task with ***xQueueSendFromISR***.<br>
( well, it could be done faster, but I don't want to change the example)

As allways  I want to see the timings on a scope, so I added pulses to show:
1) When is the interrupt generated
2) When does the interrupts service routine gets the interrupt
3) When does the RTOS-task gets the signal.

The pulses (high,low) are done at register level and need about 60 ns (nanoseconds)
 
This is the result:
 ![image1](./image1.jpg?raw=true "gpio example")

1.74 us from interrupt (source) to interrupt routine
7.20 us from interrupt (source) to RTOS-tasks

***That was disappointing***

----------------
If you are poor and wants to buy a car maybe every €/$ counts.<br>
If you are rich you maybe maybe spend a lot of money because you want a Countach - just for ***SPEED***<br>

Yes, I am rich and have 2 cores - and I am willing to sacrify one of them - just for ***SPEED***

The idea:

On core 0 following tasks are running:

1) A WebServer to show the results
2) A task to send some data to the monitor
3) A task, which produces a square wave to feed it to an interrupt pin:

      while(1) {
        portDISABLE_INTERRUPTS();
        GPIO_Set(PinA);      // + 30 ns
        Irqs2++;
        delayClocks(40);                                                // 60 of 240 is  1/4 µs = 250 ns
        GPIO_Clear(PinA);    // + 30 ns
        Irqs2++;
        delayClocks(40);
        portENABLE_INTERRUPTS();
      }
      
No vTaskDelay, no taskYIELD. But it is suspended after end of the timeslice (1 ms) is reached. 


On core 1 there is only 1 task:
       while(1) {                                                       // the superloop
         level=GPIO_IN_Read(PinB);                                      
         
         if (level != oldLevel) {
            GPIO_Set(PinC);
            GPIO_Clear(PinC);
            Irqs++;
         }
         oldLevel=level;
       }

This is an RTOS-task, but no taskswitches, even no interrupts are allowed. It is a brute force polling !<br>

Advantages:
1) Superfast
2) I am not in an interrupt routine. Therefore I am able to use all functions an RTOS-task can use.

### How fast is superfast ?

After sending a  HIGH pulse  of 116 ns, the  superloop  reacts within 170 ns to each edge (rising and falling).<br>
This is repeated every 532 ns, giving a theoretical value of 3759398 interrupts per second.

But since the interrupt source task (RTOS2) has to share its time with WiFi an d console output the measured value is<br>
a little bit lower. And it changes from second to second a little bit.

But how can I be sure not missing interrupts?<br>
It's easy: every time I generate an interrupt I count "Sended Interrupts".<br>
The total independent ***superloop*** counts every detected pin change "interrupt".<br>
They should have exactly the same value - and they do !<br>
Errors are counted - and Errors are 0 for hours and hours.

***The webpage is automatically refreshed every 2-3 seconds***

 ![image1](./image2.jpg?raw=true "3.6 Million external Interrupts per Second:")



