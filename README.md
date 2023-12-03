# P3_ControladorMaquinaExpendedora

# Introduction

As indicated in the repo name, this practice consists of creating a controller for a vending machine that is based on Arduino UNO and the sensors/actuators provided in the Arduino kit.
The practice of integrating the following hardware components:
* Arduino UNO
* LCD
* Joystick
* Temperature/Humidity Sensor DHT11
* Ultrasonic sensor
* Boton
* 2 Normal LEDs (LED1, LED2)

Below is the circuit diagram:

![Esquema_P3](https://github.com/ana-martinezal2021/P3_ControladorMaquinaExpendedora/assets/92941166/84b3f019-f0ee-4765-a8e8-32b520b854ad)

# Development

## Needs
* Since the joystick has two potentiometers (X and Y) that generate analog signals, the X and Y axis pins of the joystick must be connected to the analog inputs. 
* To change the intensity of an LED using pulse width modulation (PWM), you must connect one of the LEDs to a digital input (pins 1 to 13). 
* To declare a button as a drawn input, you must connect the joystick button to a digital input.
* The button should be connected to pin 2 or 3, which is the interrupt.

## Techniques employed

### Libraries
For the development some libraries have been used necessary for the control and management of sensors or software methods:

```c++
#include <LiquidCrystal.h>  // LCD control
#include "DHT.h" // Interaction with the DHT sensor 

// Implementation and management of threads
#include <Thread.h> 
#include <StaticThreadController.h> 
#include <ThreadController.h>

#include <TimerOne.h> // Timers set and control
#include <avr/wdt.h> // Watchdog set and control 
```

### Methods
The controller code is based on the use of switch(case) as it is a project where the status is constantly changed and the use of these makes it easy to move around them. But looking at the most significant, below I will explain the use of different methods and why.

#### Threads
In my case, I have made use of a thread that is responsible for obtaining the distances from the ultrasound sensor every 200 milliseconds. I do this so that it is constantly checked if there is someone within 1 meter of the sensor without cutting the code execution. It could also have been used for temperature and humidity measurements or for the meter, but these are things that can be implemented directly.

```c++
ThreadController controller = ThreadController(); 
Thread myThread = Thread();

void setup() {
  myThread.enabled = true;  
  myThread.setInterval(2000);    
  myThread.onRun(callback_thread1);    
  controller.add(&myThread);
}

void loop() {
controller.run();

}

void callback_thread1() {  
  digitalWrite(pinTrig, HIGH);
  delayMicroseconds(10);
  digitalWrite(pinTrig, LOW);
  
  long tiempo = pulseIn(pinEcho, HIGH);
  distance = tiempo/59;

  delay(100);
}
```

#### Interruptions

One of the conditions that this controller has is to be able to enter and exit the administrator mode, as well as resetting the program to the service mode by pressing a button for a certain time. For this it is necessary to use an interruption that allows us to act in a certain way when a change in the variable of the status of the button is detected. In this case, it is also necessary to use a timer to be able to know how long the button has been pressed since within the millis() interrupt is not actauliza. That is, it is necessary to perform a hardware interruption with the button and a software interruption with the timer, all of which is then combined in a function where it acts depending on the time the button has been pressed.

* Hardware Interruption:
  
  ```c++
  void setup(){
    pinMode(pinButton, INPUT_PULLUP);
    attachInterrupt(digitalPinToInterrupt(pinButton), button_pressed, CHANGE);
  }
  ```
  As you can see in the code the interrupt is activated when there is a change in the status of the button, either from HIGH to LOW or vice versa. When this change occurs, that is, the     button is pressed or held, the function that appears in the code where the response to that change is indicated is called.
  
* Software Interruption:

  ```c++
  void setup(){
    Timer1.initialize(1000000); // Initializes Timer1 with one interrupt every second (in microseconds)
    Timer1.attachInterrupt(timerCallback); // Associates the callback function to the Timer1
  }

  void timerCallback() {
  noInterrupts();
  pressing_time++;
  }
  ```
  The timer what it does is to make an interruption every second in which we call a function that will be used as a timer to know the time we are pressing the button.
  
* Combine function:

   ```c++
    void button_pressed() {
    // Enable interruptions
    interrupts();
  
    // Enable the watchdog (8 seconds)
    wdt_enable(WDTO_8S);
  
    // If the button is not pressed
    if (digitalRead(pinButton) == HIGH) {
      wdt_disable();
  
      // Stops the timer
      Timer1.stop();
  
      // Depending on how long the button is pressed and the state we were in, we act one way or another
      if (pressing_time >= 2 && pressing_time <= 3) {
        // Reset pressing_time
        pressing_time = 0;
        if (serviceMode == true){
          lcd_1.clear();
          var = 1;
        }
      } 
      else if (pressing_time >= 5) {
        // Reset pressing_time
        pressing_time = 0;
        if (adminMode == false){
          serviceMode = false;
          adminMode = true;
          lcd_1.clear();
          admin_var = 1;
        }
        else if (adminMode == true){
          analogWrite(PIN_LED2, 0);
          digitalWrite(PIN_LED1, LOW);
          adminMode = false;
          serviceMode = true;
          lcd_1.clear();
          var = 1;
        }
      }
      // Reset pressing_time
      pressing_time = 0;
  
    } 
    else {
      // Enable interruptions
      interrupts();
  
      // Enable watchdog(8 seconds)
      wdt_enable(WDTO_8S);
  
      // Start the timer
      Timer1.start();
    }
  }
    ```

  Keep in mind that to modify a variable within a break these must be declared as volatiles.

#### Watchdog

The watchdog must be used in order to perform a plate reset in case the controller execution is blocked for any reason or does not respond properly. Therefore, it is a safety mechanism that we must employ in the controller as follows:

```c++
// Disable the watchdog
  wdt_disable();

// Enable the watchdog (8sec)
  wdt_enable(WDTO_8S);
```

# Video
https://urjc-my.sharepoint.com/:v:/g/personal/a_martinezal_2021_alumnos_urjc_es/EYRCXvJrZT5KgUSARmJbgU8B1t921e2RBUF-L4YJAYyISw
  
