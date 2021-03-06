# ESP32-Fast-external-IRQs
## How to receive more than 3.000.000 Interrupts per Second
## How to shape Pulses with a Granularity of 50 ns

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
In the espressif folder esp-id/fexamples you can find  the ***gpio_example***<br>
Here we have (selfmade) interrupts. The interrupt service routine then unblocks an RTOS task with ***xQueueSendFromISR***.<br>
( well, it could be done faster, but I don't want to change the example)

As allways  I want to see the timings on a scope, so I added pulses to show:
1) When is the interrupt generated
2) When does the interrupt service routine gets the interrupt
3) When does the RTOS-task gets the signal.

The pulses (high,low) are done at register level and need about 60 ns (nanoseconds)
 
This is the result:
 ![image1](./image1.jpg?raw=true "gpio example")

1.74 us from interrupt (source) to interrupt routine<br>
7.20 us from interrupt (source) to RTOS-tasks<br>
(sorry for the bad quality of the pictures, I am using a very old HP1653B logic analyzer as scope)

***That was disappointing !***

----------------
If you are poor and wants to buy a car maybe every €/$ counts.<br>
If you are rich you maybe spend a lot of money because you want a Countach - just for ***SPEED***<br>

Yes, I am rich and have 2 cores ;)  - and I am willing to sacrify one of them - just for ***SPEED***

The idea:

On core 0 following tasks are running:

1) A WebServer to show the results
2) A task to send some data to the monitor
3) A task, which produces a square wave to feed it to an interrupt pin:
```
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
```      
No vTaskDelay, no taskYIELD. But it is suspended when the end of the timeslice (1 ms) is reached. 


On core 1 there is only 1 task:
```
       while(1) {                                                       // the superloop
         level=GPIO_IN_Read(PinB);                                      
         
         if (level != oldLevel) {
            GPIO_Set(PinC);
            GPIO_Clear(PinC);
            Irqs++;
         }
         oldLevel=level;
       }
```

This is an RTOS-task, but no taskswitches, even no interrupts are allowed. It is a brute force polling !<br>

Advantages:
1) Superfast
2) I am ***not*** in an interrupt routine. Therefore I am able to use all functions an RTOS-task can do.

### How fast is superfast ?

After sending a  HIGH pulse  of 116 ns, the  superloop  reacts within 170 ns to each edge (rising and falling).<br>
This is repeated every 532 ns, giving a theoretical value of 3759398 interrupts per second.

 ![image3](./image3.jpg?raw=true "Detect an external interrupt in 176 ns")
 ***Detect an external interrupt in 176 ns***

But since the interrupt source task (RTOS2) has to share its time with WiFi and console output the measured value is<br>
a little bit lower. And it changes from second to second a little bit.

But how can I be sure not missing interrupts?<br>
It's easy: every time I generate an interrupt I incement "Sended Interrupts".<br>
The total independent ***superloop*** counts every detected pin change "interrupt".<br>
They should have exactly the same value - and they do !<br>
Errors are counted - and Errors are 0 for hours and hours.

***The webpage is automatically refreshed every 2-3 seconds***

 ![image1](./image2.jpg?raw=true "3.6 Million external Interrupts per Second:")


#  Conclusion
 
It is possible to count and react on more than ***3.600.000*** external pinchanges per second (with at least<br>
120 ns HIGH-Length) at both edges! That is really ***Superfast***.<br>
You are even able to detect such pulses at different pins. (mask the REG_IN)<br>

### Where to use it?

- Fast Handshake. Send an ACK to an external sender in less than 200 ns

- Build a high speed protocol

- Detect speed of motors (rpm) which are running very fast (turbo jet)

- Measure speed of a gun bullet

- Detect changes at multiple pins in 200 ns and react fast

- Sending marks from RTOS tasks to analyze timings

This method of polling in a very fast superloop is not only for pulses.
You may use it as:
- A superfast Watchdog timer
- A smart pulse creator
If for instance an RTOS-task has to produce 6.8 us pulses very often, but you don't want to use a blocking delay.
Now you can start a pulse like (RTOS-task):
```
        portDISABLE_INTERRUPTS();
        GPIO_Set(PinA);      // + 30 ns
        SetFlagForSuperLoop = 1;
        portENABLE_INTERRUPTS();
```
and in the ***superloop***:

```
   while(1) {
     if (SetFlagForSuperLoop) cnt++;
     if (cnt==143) {          // this is for 6.8 us, 147 for 7.0 µs
       GPIO_Clear(PinA);      // + 30 ns
       SetFlagForSuperLoop = 0;
       cnt=0;
     }
   }
```       
With this you just have to initiate the pulse in the RTOS-task<br>
***superloop*** will end the pulse with high precision !<br>

### You will get a granularity of 50 ns !<br>

<br> 
# What else can we do ?
With the methode shown you get an independent pocessor like a superfast Arduino in Core1.<br> 
Exception: No Interrupts allowed.<br> 
But you can communicate with Interrupts (Timer, External) which are installed at Core0!<br> 

You should NOT use vTaskDelay or taskYIELD, but you may use delayMicroseconds ane even delayClocks.<br> 
You should NOT use printf. Instead use sprintf into a buffer and the set a flag.<br> 
An RTOS task can test this flag (for instance 10 times a second) to do the output of the buffer.<br> 

This is cool for jobs where you want to do high speed bit banging (WS2812 NeoPxeL).<br> 

For instance: you want a ***pulse*** of ***100 ns***, repeated every ***500ns***<br>
Global:
```
    volatile uint32_t ClockTau = (120-9);                               // -9:: subtract GPIO
    volatile uint32_t ClockLen = (24-9);
```
and ***superloop***
```
       uint32_t strt=clocks();
       while(1) {                       
          while ((clocks()-strt) < ClockTau);
          strt=clocks();                                                // the superloop
          GPIO_Set(PinC);
          delayClocks(ClockLen);
          GPIO_Clear(PinC);         
       }
```
and you are able to change ClockTau and ClockLen on the fly !

# CoopOS
Use RTOS? or cooperative multitasking ***CoopOS***? which I explained in other repositories.<br>
Here you can have both!<br>
RTOS on Core0, including Wifi, multiple RTOS tasks, Timer- an External interrupts.<br>
And the fast ***CoopOS*** dealing with microsseconds on Core 1.<br>
I will show it in another repository - stay tuned ;)<br>

<br><br><br><br><br>

# Installation (for esp-idf Version 4.1)

1) You should have managed to copy some examples into your **esp folder**. That is the folder where you can find **esp-idf**
2) The esp/esp-idf/exmples folder should be intact, because **CMakeList.txt** needs<br>
   **set(EXTRA_COMPONENT_DIRS $ENV{IDF_PATH}/examples/common_components/protocol_examples_common)**
3) Unzip the **fastIRQ2.zip** to **esp**
4) Run **ifd.py menuconfig**
   
      You should see:<br>
             Component_config/FreeRTOS **(1000)** Tick rate (Hz)<br>
             Component_config/Common_ESP_related:<br>
             &nbsp;&nbsp;**Unmarked** Interrupt watchdog<br>
             &nbsp;&nbsp;**Unmarked** Initialize Task Watchdog Timer<br>
             Component_config/ESP32_specific/CPU frequency **240 MHz**<br>

      Put your **WiFi data** into
             Example Connection Configuration
     
5) Now connect your ESP32 an start **ifd.py flash** in terminal in **esp/irqFast2**<br>
   The program should be compiled and uploaded.<br>
   
6) Look at the output to detect **192.168.xxx.yyy** (or whatever your network-id is for this device)

7) Open a browser and look at&nbsp;&nbsp; **http://192.168.xxx.yyy/info**   &nbsp;&nbsp;&nbsp;&nbsp;**Do not forget the .../info !!!**

You should see a Website like the last image which is updated every 2-3 seconds.

# Have fun ;)


   
