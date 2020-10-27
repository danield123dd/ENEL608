# Parallel I/O - Week One
*This markdown file is a ~~quick~~ lengthy rundown of the Parallel I/O. This should just act as a getting started, most of this content will be explained much better in the lectures!*

## Parallel I/O
The 8-bit AT90 Microcontroller we will be using has 6 Parallel Ports, each comprising of 8-pins. Almost all these pins can be configured to bi-directional communication (as either an output or an input - but not both simultaneously). These ports are used to connect devices to your Microcontroller (the stuff that makes it interesting), such as buttons and switches, LED's, screens, sensors, etc. 

*Parallel* refers to the way in which data can be pushed through all 8 pins at the same time, unlike a *serial* interface, in which would need to complete the same task using only one pin. This allows data to theoretically be transferred faster than over a serial interface.

## Parallel I/O Registers
*Note: In most reference material, these 8 pins will be listed from Pin 0 to Pin 7 (So Pin 1 is actually the 2nd physical pin!).*

Each Parallel Port has a block letter which represents it's own set of 8-pins on the micro (Port A - F). Each Parallel Port has three 8-bit registers associated to it.

* **DDRx**: The Data Direction Register (DDR) is responsible for setting whether each pin on that port is an input (0) or an output (1). Pins *cannot* be used for both input and output simultaneously. This register is generally configured in the setup routine of our program.
* **PINx**: The Port INput register (PIN) is a read-only register which contains the state of each pin of a port. We can use these to detect the input of devices such as switches or digital sensors.
* **PORTx**: The PORT output register (PORT) is a read/write register which sets the state of each pin of a port. We can use this register to send either a High or Low signal out on each pin to turn on/off devices (such as LEDs, Buzzers, Fans, etc.)

For all registers, the highest bit represents Pin 7, and the lowest bit represents Pin 0. For the PORT and PIN registers, a '0' represents a low signal (off), and a '1' represents a high signal (on). You will never use the PORT and PIN registers on the same Parallel Port in one program.

```c
  DDRA = 0b11111111; // Make Parallel Port A an output
  PORTA = 0b10000000; // Turn on Pin 7
  PORTA = 0b00000001; // Turn on Pin 0
```

Note that when over-writing a register (like we have with `PORTA` above), we do not inherit the previous state of the register (So for above, Pin 7 is turned on, and then Pin 1 is turned on and Pin 7 is turned off). **This is an important point to consider for the next section.**

In the example below, we will create a flashing set of 8 LED's which switch between on/off every half of a second. We will connect all these LEDs to the 8-pins of Parallel Port A. We will setup the DDR, and then use the PORT register and a while loop to turn on and off the LED's indefinitely.

*This example is written like a final program - focus on the main() and setup() functions here for now, the rest will come as you do the lab tasks.*

```c
#include <avr/io.h>
#include <util/delay.h>

void setup(void);
int main(void);

int main()
{
  // Call our setup function to set the function of Parallel Port A
  setup();
  
  // Repeat the code inside the while loop indefinitely.
  while(1)
  {
    PORTA = 0b11111111; // 🔴🔴🔴🔴🔴🔴🔴🔴
    _delay_ms(500);
    PORTA = 0b00000000; // ⚫⚫⚫⚫⚫⚫⚫⚫
    _delay_ms(500);
  }
  
  return 0;
}

void setup()
{
  DDRA = 0b11111111; // This sets all 8 pins to be outputs
}
```

## Effectively Controlling the state of each pin
While it's relatively easy to set the function of all 8 pins at once, let's say we wanted to only modify the function of an individual pin. Let's elaborate on our example from above to explain.

In this example, each of the 8 LEDs are a status light to represent the function of each element of a photocopier machine 🖨 (I know... super interesting) - when an LED is on, that function of the photocopier works:

```c
  PORTA = 0b11111111; // Turn on all LED's on Parallel Port A
```

So far, reasonably straight forward (also, our photocopier is working yay!). Now, let's pretend that one of our photocopier elements breaks, and that element is linked specifically to the LED on Pin 1. Easily enough, we just need to turn off the LED using the PORTA register:

```c
  // Turn off the LED on Pin1 - remember, Pin 1 is the second physical pin, hence why the 2nd bit has changed!
  PORTA = 0b11111101; // 🔴🔴🔴🔴🔴🔴⚫🔴
```

Seems easy enough, right? However, for us to have turned off the LED on Pin 1 as above, we've assumed that the rest of the other 7 elements of the photocopier are working correctly. In the real world, however, this assumption cannot be made - our photocopier could have multiple different faults, in which mean the LED's could be actively set in any number of combinations:

```c
  ...
  PORTA = 0b01101111;
  PORTA = 0b10011111;
  PORTA = 0b11110010;
  ...
```

As mentioned earlier, it is not possible to inherit the previous attributes by just writing to the `PORT` register alone. The question becomes - **How can we change the state of one specific LED, while preserving the existing state of the other 7?** Using our current method (and some very trivial thinking), we could try use an 'if' tree to sort through and compare each possible scenario, and if the scenario matches, we could change the LED's correctly based on its current state:

```c++
  ...
  else if (PORTA == 0b01101111)
  {
    PORTA = 0b01101101;
  }
  else if (PORTA == 0b10011111)
  {
    PORTA = 0b10011101;
  }
  else if (PORTA == 0b11110010)
  {
    PORTA = 0b11110000;
  }
  ...
```
To spare you doing the maths in your head, we'd need to create 256 different cases to correctly change the state of the LED on Pin 1, while keeping the current state of all the other LED's. Yikes 😳, that ain't gonna go over well...

Luckily for us, however, we're able to use some bit-manipulation techniques to create a command that is able to correctly preserve and change the state of bits!

## Bit-manipulation Techniques
*This is where things can get complicated, and it is highly recommended that you invest the time to understand Bit-masking as it will be essential as the course progresses!!* 🤯

### Bit-masking using bitwise operators

The technique of bit-masking is simply to produce an output based on the comparison of two inputs. Think of it as a more dumbed-down version of the way you used to do maths without a calculator in primary school:

```   
               1
      10      273            11001100
   +  10    +   9    --->    01010101     
   -----    -----          & --------
      20      282            01000100
        
```

#### AND (`&`) Operator
Using the AND operator, we can compare the values in two registers, and for each bit, produce a '1' if it is a '1' in both registers:

```
  R1  10110110
  R2  01111011
  ------------
  OP  00110010
```

#### OR (`|`) Operator
Using the OR operator, we can compare the values in two registers, and for each bit, produce a '1' if either or both bits in each register is a '1'

```
  R1  10110110
  R2  01111010
  ------------
  OP  11111110
```

#### EXCLUSIVE OR / XOR (`^`) Operator
This is very similar to the OR operator, except that a '1' is produced only if there is a '1' in *either* register (if both registers have a '1' a '0' results).

```
  R1  10110110
  R2  01111010
  ------------
  OP  11001100
```

### Shifting Bits using Bitwise operators

We will also be using a small degree of bit shifting to make things a bit more readable. For our course, shifting of bit's will be limited to manipulation of the decimal number '1'. Bit-shifting is just shifting the bits in a byte.

Let's take on an example below - here's an integer named 'value' assigned a number equal to `(1<<4)`:

```c
  int value = (1<<4);
```

Breaking it down, we will start by taking the decimal number '1' from above, and graphically represent it as a binary number:

```
  00000001
```

The `<<` means we will be shifting to the right, and the '4' tells us we will be moving 4 places:

```
  ORIGINAL VALUE:   00000001
                    00000010
                    00000100  <-- Notice the 1 shifting to the right?
                    00001000
  FINAL VALUE:      00010000
```
Numbers can also be shifted to the left with the `>>` operator, and any of the numbers above can be modified (just make sure your decimal number doesn't exceed 255)!

### Ones' Complement (~)
Finally, another bitwise operator we will be using is the Ones' complement (denoted by a `~`). This is simply to say to invert the state of each bit:

```
  BEFORE              10101110
  ONES' COMPLEMENT    01010001
```

## Using Bit-masking to solve complications with modifying pins

Going back to our photocopier scenario 🖨, we can use bit-masking to control the state of a single LED. To make code more readable, we generally use `#define` statements to define such functions:

```c++
  #define turnOnLED1  (PORTA |= (1<<1))
  #define turnOffLED1 (PORTA &= ~(1<<1))
  #define toggleLED1  (PORTA ^= (1<<1))
```

*These bit-masks are widely used throughout the course - it might be ideal to memorise the three masks above, but take the time to understand them too!*

### Going from an Off to an On state

```c++
  #define turnOnLED1  (PORTA |= (1<<1))

  PORTA = 0b01000000; // Assigned value to PORTA in this example
```

Firstly, we are shifting bits in the decimal number 1:

```
  (1<<1)

  ORIGINAL VALUE: 00000001
  SHIFTED VALUE:  00000010
```

Then, we are comparing this value to the value in the register `PORTA`:

```
  PORTA   01000000
  VALUE   00000010
      |   --------
          01000010
```

The `|=` operator means that after we compare the values with the OR operator, we will write the new value to the register `PORTA`.

### Going from an On to an Off state

```c++
  #define turnOffLED1 (PORTA &= ~(1<<1))

  PORTA = 0b00100010; // Assigned value to PORTA in this example
```

Firstly, we are shifting bits in the decimal number 1:

```
  (1<<1)

  ORIGINAL VALUE: 00000001
  SHIFTED VALUE:  00000010
```

We then perform the Ones' Complement on this number:

```
  SHIFTED VALUE:  00000010
  INV. VALUE:     11111101
```
Then, we are comparing this value to the value in the register `PORTA`:

```
  PORTA   00100010
  VALUE   11111101
      &   --------
          00100000
```
The `&=` operator means that after we compare the values with the AND operator, we will write the new value to the register `PORTA`.

### Toggling Between states
Toggling is to switch between either off or on, depending on the current state the pin is in at the time the command is executed.

```c++
  #define toggleLED1  (PORTA ^= (1<<1))

  PORTA = 0b01100010;
```
Firstly, we are shifting bits in the decimal number 1:

```
  (1<<1)

  ORIGINAL VALUE: 00000001
  SHIFTED VALUE:  00000010
```
Then, we are comparing this value to the value in the register `PORTA`:

```
  PORTA   01100010
  VALUE   00000010
      ^   --------
          01100010
```
The `^=` operator means that after we compare the values with the XOR operator, we will write the new value to the register `PORTA`.

### Complications Solved

Truly magical, right!? 🎩 Combining these bit-level operations, we can change the state of one specific LED, while preserving the existing state of the other 7!

## Using Bit-masking to read the state of a specific pin
Bit-masking is also used to determine whether or not an input is active when using pins as inputs. In the example below, we have a switch attached to Pin 2 on Parallel Port C:

```c++
  DDRC = 0b00000000; // All pins on Parallel Port C are inputs
  #define switchIsOn  PINC & (1<<2)
```
Let's expand on how the value for `switchIsOn` is determined. For the sake of this example, let's pretend that at the point we check the `switchIsOn` statement, that `PINC = 0b00000100`.

 We firstly start by shifting bits in the decimal number 1 over 2 places to the right:

```
  (1<<2)

  ORIGINAL VALUE: 00000001
  SHIFTED VALUE:  00000100
```
Then, we compare this value to the value in the register `PINC`:

```
  PORTA   01100010
  VALUE   00000100
      &   --------
          00000000
```
Using the `&` operator, we can return a binary number equivalent to the decimal '0'. A zero is always considered `false`, and any value other than '0' is considered `true`. This tells us that the switch is in fact, turned off.
