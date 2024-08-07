![RIOS logo](rios.jpg)
## Overview

This is a copy of [rios by Frank Vahid and Bailey Miller](https://www.cs.ucr.edu/~vahid/rios/)

RIOS is a cooperative multitask scheduler written entirely in C that:

- Is simple and understandable for the beginning embedded programmer
- Can provide basic non-preemptive or preemptive cooperative multitasking capabilities for cooperative tasks
- Requires only a few dozen lines of C code. Reduces need for RTOS (real-time operating system).

## Versions

RIOS comes in three versions. A non-preemptive version with either simple or state-machine based cooperative tasks. A cooperative task when invoked executes quickly to completion, with no surrounding infinite loop and never waiting on external events. Simple examples for each version are available below for you to copy/paste. Each example targets the RIMS software [Programming Embedded Systems](http://www.programmingembeddedsystems.com/RITools). By copying/pasting, downloading, or using the code in any way, you are agreeing to the [EULA](eula.txt).

- [Non-preemptive, simple tasks](#)
  ```c
  #include "RIMS.h"

  /*
     Copyright (c) 2013 Frank Vahid, Tony Givargis, and
     Bailey Miller. Univ. of California, Riverside and Irvine.
     RIOS version 1.2
  */
  typedef struct task {
     unsigned long period;      // Rate at which the task should tick
     unsigned long elapsedTime; // Time since task's last tick
     void (*TickFct)(void);     // Function to call for task's tick
  } task;

  task tasks[2];
  const unsigned char tasksNum = 2;
  const unsigned long tasksPeriodGCD = 200; // Timer tick rate
  const unsigned long periodToggle = 1000;
  const unsigned long periodSequence = 200;

  void TickFct_Toggle(void);
  void TickFct_Sequence(void);

  unsigned char TimerFlag = 0;
  void TimerISR(void) {
     if (TimerFlag) {
        printf("Timer ticked before task processing done.\n");
     } else {
        TimerFlag = 1;
     }
     return;
  }

  int main(void) {
     // Priority assigned to lower position tasks in array
     unsigned char i = 0;
     tasks[i].period = periodSequence;
     tasks[i].elapsedTime = tasks[i].period;
     tasks[i].TickFct = &TickFct_Sequence;
     ++i;
     tasks[i].period = periodToggle;
     tasks[i].elapsedTime = tasks[i].period;
     tasks[i].TickFct = &TickFct_Toggle;

     TimerSet(tasksPeriodGCD);
     TimerOn();

     while (1) {
        // Heart of the scheduler code
        for (i = 0; i < tasksNum; ++i) {
           if (tasks[i].elapsedTime >= tasks[i].period) { // Ready
              tasks[i].TickFct(); // Execute task tick
              tasks[i].elapsedTime = 0;
           }
           tasks[i].elapsedTime += tasksPeriodGCD;
        }
        TimerFlag = 0;
        while (!TimerFlag) {
           Sleep();
        }
     }
  }

  // Task: Toggle an output
  void TickFct_Toggle(void) {
     static unsigned char init = 1;
     if (init) { // Initialization behavior
        B0 = 0;
        init = 0;
     } else { // Normal behavior
        B0 = !B0;
     }
  }

  // Task: Sequence a 1 across 3 outputs
  void TickFct_Sequence(void) {
     static unsigned char init = 1;
     unsigned char tmp = 0;
     if (init) { // Initialization behavior
        init = 0;
        B2 = 1;
        B3 = 0;
        B4 = 0;
     } else { // Normal behavior
        tmp = B4;
        B4 = B3;
        B3 = B2;
        B2 = tmp;
     }
  }
  ```
- Non-preemptive, state machine tasks
  ```c
  #include "RIMS.h"
  /*
     Copyright (c) 2013 Frank Vahid, Tony Givargis, and
     Bailey Miller. Univ. of California, Riverside and Irvine.
     RIOS version 1.2
  */
  typedef struct task {
     unsigned long period;      // Rate at which the task should tick
     unsigned long elapsedTime; // Time since task's last tick
     int state;                 // Current state
     int (*TickFct)(int);       // Function to call for task's tick
  } task;

  enum TG_States { TG_None, TG_s1 };
  int TickFct_Toggle(int);

  enum SQ_States { SQ_None, SQ_s1, SQ_s2, SQ_s3 };
  int TickFct_Sequence(int);

  unsigned char TimerFlag = 0;
  void TimerISR(void) {
     if (TimerFlag) {
        printf("Timer ticked before task processing done.\n");
     } else {
        TimerFlag = 1;
     }
     return;
  }

  int main(void) {
     task tasks[2];
     const unsigned char tasksNum = 2;
     const unsigned long tasksPeriodGCD = 200; // Timer tick rate
     const unsigned long periodToggle = 1000;
     const unsigned long periodSequence = 200;

     unsigned char i = 0;
     tasks[i].period = periodToggle;
     tasks[i].elapsedTime = tasks[i].period;
     tasks[i].TickFct = &TickFct_Toggle;
     tasks[i].state = TG_None;
     i++;
     tasks[i].period = periodSequence;
     tasks[i].elapsedTime = tasks[i].period;
     tasks[i].TickFct = &TickFct_Sequence;
     tasks[i].state = SQ_None;

     TimerSet(tasksPeriodGCD);
     TimerOn();

     while (1) {
        // Heart of the scheduler code
        for (i = 0; i < tasksNum; ++i) {
           if (tasks[i].elapsedTime >= tasks[i].period) { // Ready
              tasks[i].state = tasks[i].TickFct(tasks[i].state);
              tasks[i].elapsedTime = 0;
           }
           tasks[i].elapsedTime += tasksPeriodGCD;
        }
        TimerFlag = 0;
        while (!TimerFlag) {
           Sleep();
        }
     }
     return 0;
  }

  int TickFct_Toggle(int state) {
     switch (state) { // Transitions
        case TG_None: // Initial transition
           B0 = 0; // Initialization behavior
           state = TG_s1;
           break;
        case TG_s1:
           state = TG_s1;
           break;
        default:
           state = TG_None;
     }
     switch (state) { // State actions
        case TG_s1:
           B0 = !B0;
           break;
        default:
           break;
     }
     return state;
  }
  int TickFct_Sequence(int state) {
     switch (state) { // Transitions
        case SQ_None: // Initial transition
           state = SQ_s1;
           break;
        case SQ_s1:
           state = SQ_s2;
           break;
        case SQ_s2:
           state = SQ_s3;
           break;
        case SQ_s3:
           state = SQ_s1;
           break;
        default:
           state = SQ_None;
     }
     switch (state) { // State actions
        case SQ_s1:
           B2 = 1;
           B3 = 0;
           B4 = 0;
           break;
        case SQ_s2:
           B2 = 0;
           B3 = 1;
           B4 = 0;
           break;
        case SQ_s3:
           B2 = 0;
           B3 = 0;
           B4 = 1;
           break;
        default:
           break;
     }
     return state;
  }
  ```

## Porting to AVR microcontrollers

For more information on running RIOS on an AVR, check [here](rios_avr.md).

## Student Projects

- [All projects (unedited)](http://www.youtube.com/watch?v=zGjtY21McCA)
- [Highlights](http://www.youtube.com/watch?v=VkCYbA6kXng")

## About

RIOS stands for Riverside/Irvine Operating System, developed at the University of California (UC) Riverside and UC Irvine. RIOS is not really an operating system but rather a simple task scheduler, the key element of a real-time operating system in embedded systems.

For information, contact eslucr@gmail.com.

Developed by: [Frank Vahid](http://www.cs.ucr.edu/~vahid) and [Bailey Miller](http://www.cs.ucr.edu/~bmiller), University of California Riverside; and [Tony Givargis](http://givargis.com), University of California Irvine.

This material is based upon work supported by the National Science Foundation under Grants 0614957, 1016792, 1136146, 1563652. Any opinions, findings, and conclusions or recommendations expressed in this material are those of the author(s) and do not necessarily reflect the views of the National Science Foundation.