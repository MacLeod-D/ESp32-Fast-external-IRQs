# ESP32-Fast-external-IRQs
## How to receive more than 3 Million Interrupts per Second

I like the ESP8266. With 160 MHz it is fast! And you can do nearly all your Arduino stuff - but (at least) 10 times faster.

But when it comes to WiFi (which comes built in) you have a problem:<br>
When you implement a WebServer, it uses callbacks to handle the http-transfers.

asas
 
