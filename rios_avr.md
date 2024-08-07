The given example code targets RIMS, a microcontroller simulator with its own interface to a timer peripheral, and simple I/O ports A and B. To use an AVR microcontroller, the following functions must be replaced with functions that target the AVR you are using:

- `TimerOn()`: Turns the timer peripheral on.
- `TimerSet(int)`: Set the number of milliseconds between timer interrupts.
- `TimerISR(int)`: The timer interrupt handler.

The following sections provide code for implementing RIOS on an AVR ATMega324P.

## TimerOn()
```c
void TimerOn(void) { 
   TCCR1B = (1<<WGM12)|(1<<CS12); // Clear timer on compare. Prescaler = 256
   TIMSK1 = (1<<OCIE1A); // Enables compare match interrupt
   SREG |= 0x80; // Enable global interrupts 
}
```

## TimerSet()
```c
void TimerSet(int milliseconds) { 
   TCNT1 = 0; 
   OCR1A = milliseconds*31.25; // 8 MHz AVR, prescalar == 256 -> 31.25 ticks per ms
}
```

## TimerISR()
AVRs require specific, predefined function names when calling an ISR. Therefore we can simply place the TimerISR function into the appropriate interrupt function, which in this case is the timer 1 compare interrupt. To save a few cycles, the RIOS kernel code can be inlined into the AVR-specific interrupt function instead.
```c
void TimerISR(void) { 
   // RIOS scheduler code 
} 

ISR(TIMER1_COMPA_vect) { // Timer compare match interrupt service routine
   TimerISR(); 
}
```

## Complete AVR example
The following is complete template code for the preemptive version of RIOS implemented on an AVR. Simply replace the 3 empty task functions with your own code.
```c
#include <avr/io.h>
#include <avr/interrupt.h>
#include <math.h>
#define F_CPU 8000000UL  
#include <util/delay.h>

typedef struct task {
   unsigned char running; // 1 indicates task is running
   int state;             // Current state of state machine
   unsigned long period;  // Rate at which the task should tick
   unsigned long elapsedTime; // Time since task's previous tick
   int (*TickFct)(int);   // Function to call for task's tick
} task;

task tasks[3];

const unsigned char tasksNum = 3;
const unsigned long tasksPeriodGCD = 25;
const unsigned long period1 = 25;
const unsigned long period2 = 50;
const unsigned long period3 = 100;

int TickFct_1(int state);
int TickFct_2(int state);
int TickFct_3(int state);

unsigned char runningTasks[4] = {255}; // Track running tasks, [0] always idleTask
const unsigned long idleTask = 255; // 0 highest priority, 255 lowest
unsigned char currentTask = 0; // Index of highest priority task in runningTasks

unsigned schedule_time = 0;
ISR(TIMER1_COMPA_vect) {
   unsigned char i;
   for (i=0; i < tasksNum; ++i) { // Heart of scheduler code
      if ((tasks[i].elapsedTime >= tasks[i].period) // Task ready
          && (runningTasks[currentTask] > i) // Task priority > current task priority
          && (!tasks[i].running)) { // Task not already running (no self-preemption)
         SREG &= 0x7F;
         tasks[i].elapsedTime = 0; // Reset time since last tick
         tasks[i].running = 1; // Mark as running
         currentTask += 1;
         runningTasks[currentTask] = i; // Add to runningTasks
         SREG |= 0x80;
         
         tasks[i].state = tasks[i].TickFct(tasks[i].state); // Execute tick
         
         SREG &= 0x7F;
         tasks[i].running = 0; // Mark as not running
         runningTasks[currentTask] = idleTask; // Remove from runningTasks
         currentTask -= 1;
         SREG |= 0x80;
      }
      tasks[i].elapsedTime += tasksPeriodGCD;
   }
}

void init_processor(void) {
    /* Set up SPI */
    PORTB = 0xff;
    
    /* Set up timer */
    TCCR1B = (1<<WGM12)|(1<<CS11); // CTC mode (clear timer on compare). Prescaler=8
    OCR1A = 25000; // AVR output compare register OCR0.
    TIMSK1 = (1<<OCIE1A); // Enables compare match interrupt
    TCNT1 = 0;
        
    /* Enable global interrupts */
    SREG |= 0x80;
}

int main(void) {
    init_processor();

   // Priority assigned to lower position tasks in array
   unsigned char i = 0;
   tasks[i].state = -1;
   tasks[i].period = period1;
   tasks[i].elapsedTime = tasks[i].period;
   tasks[i].running = 0;
   tasks[i].TickFct = &TickFct_1;
   ++i;
   tasks[i].state = -1;
   tasks[i].period = period2;
   tasks[i].elapsedTime = tasks[i].period;
   tasks[i].running = 0;
   tasks[i].TickFct = &TickFct_2;
   ++i;
   tasks[i].state = -1;
   tasks[i].period = period3;
   tasks[i].elapsedTime = tasks[i].period;
   tasks[i].running = 0;
   tasks[i].TickFct = &TickFct_3;
    
    while (1) {
    }
}

int TickFct_1(int state) {
    _delay_us(1000);
    return 0;
}

int TickFct_2(int state) {
    _delay_us(5000);
    return 0;
}

int TickFct_3(int state) {
    _delay_us(25000);
    return 0;
}
```