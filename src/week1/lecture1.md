# What is an Operating System?

|||objectives
After this lecture, you should be able to answer the following:
- What is this course all about?
- What is an Operating System and why is it important?
- What is Linux and where is it being used?
|||

### Examples of Operating Systems

An Operating System is software that manages hardware resources and facilitates the communication between software and hardware.

* Windows
* Android
* IOS
* FreeBSD
* OpenBSD
* **Linux**
* Real-time OSs

### Why do we need an operating system?

Let us take a look at this code example. This code opens a file and writes the text `Hello` to it.

```C
#include <stdio.h>
#include <stdlib.h>

int main(){
   FILE *file;

   file = fopen("file.txt", "w");

   fprintf(file, "Hello");
   fclose(file);

   return 0;
}
```

After running this code, a file will be created with the text `Hello`, but how does the code write the file to the hard disk? There is nothing in the code above that communicates with the hard disk. It all seems like magic. How does software communicate with hardware such as the memory?

Basically, the operating system is like a manager that controls who accesses resources and how.

![OS jobs](./week1/osjobs.png)
![OS jobs](./week1/cars.png)

### Why learning about Operating systems is important?

* Because they are everywhere.
* Knowing how the operating system works will be useful in your career. Many jobs require deep knowledge of how an operating system works.
* Operating systems knowledge is essential for many roles, such as security-related roles.
* Knowledge of how an operating system works will make you a better engineer.

### Why focus on Linux?

Linux is everywhere; it is used in servers, cars, embedded systems, IoT devices, and phones (Android is actually Linux). Linux is open source, meaning that we can see its source code and examine how it is implemented. This makes it easier to study. Linux is also free and you can modify it however you would like to (we will do that in this course!). You can create your own version of Linux if you want.

### How are we going to study this course?

* 1 recorded lecture per week.
* 1 optional live session every two weeks for Q/A.
* The course will focus on the practical and application aspects rather than pure theory.

### What do you need to study this course?

* Basic understanding of Operating Systems.
* Basic understanding of Linux commands.
* Having a Linux VM installed.

|||quiz
- What is the job of Operating systems?
- Can you give examples of operating systems?
- What is Linux?
- Make sure to have a Linux VM installed and ready for the next lectures.
|||

<div style="text-align: center; font-size: 0.8em; color: gray; margin-top: 50px;">Maysara Alhindi -- 2026</div>
