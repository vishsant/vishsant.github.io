+++
date = "2026-01-11"
draft = false
title = "Endianness: The Art of Thinking in Bytes"
+++

When you write the number 123, you know exactly what you mean.

One hundred twenty-three. The digit 1 carries the weight. It sits in the hundreds place. If it changes to 2, you have doubled your number. If the 3 changes to 4, you have added one.

The direction is natural. Obvious. Built into how you learned to count.

But imagine showing this number to someone from a culture that reads right to left like in Arabic.

You point at the paper and say, "This is one hundred twenty-three." They look at the symbols and read: three hundred twenty-one. They are not wrong. They are using their own directional logic. Same symbols, different direction, different meaning.

This is not a metaphor. This is what happens inside a computer.

A number like 0x12345678 exists as four separate bytes in memory.

The question is: in which direction do they face?

If you store a 32-bit integer at address 1000, you now occupy addresses 1000, 1001, 1002, and 1003. Each address holds one byte. Which byte holds the heavyweight digit? Which one comes first?

In English, we write the most important digit first. The number 123 puts the 1 up front because the 1 matters most. This feels human. Natural. Expected.

A computer could store 0x12345678 in two ways.

First way: start with the big byte.

```text
Address 1000: 0x12
Address 1001: 0x34
Address 1002: 0x56
Address 1003: 0x78
```

Read the addresses left to right and the bytes appear in order. The heavyweight byte leads. This is called big-endian.

Second way: start with the small byte.

```text
Address 1000: 0x78
Address 1001: 0x56
Address 1002: 0x34
Address 1003: 0x12
```

Read the addresses left to right and the bytes appear flipped. The lightweight byte leads. This is called little-endian.

Both are correct. Both represent the same number. But they look completely different when you inspect memory. And if you write the number on one machine and read it on another without knowing which direction was chosen, you get garbage.

The terms "big-endian" and "little-endian" come from *Gulliver's Travels*. Two groups go to war over which end of a boiled egg to crack first. Jonathan Swift meant this as satire. The computing world adopted it as technical vocabulary. The metaphor fits. Both byte orders work. The choice is arbitrary at the level of mathematics. But once made, it becomes embedded in hardware, in protocols, in every file format that stores numbers.

What makes endianness interesting is not that two directions exist. What makes it interesting is what happens when you forget a direction was chosen. When you assume everyone means the same thing by a byte sequence. When your program crosses a boundary and the data arrives backward.

For now, hold this image:

***A number that reads two different ways, depending on which end you start from. Neither direction is more correct. Both are conventions. And conventions only work when everyone agrees to use the same one.***

## Shape of Memory

Before you can understand endianness, you need to understand something simpler. Memory has a shape.

Memory is not abstract. It is not a cloud where variables float. It is a long row of boxes, each with an address. Address 0, address 1, address 2, and so on. You can store exactly one byte in each box. If you need to store a 32-bit integer, it takes four boxes.

Most programming education treats memory as infinite and uniform. You declare an int, the compiler gives you space, and you move on. This works fine for learning loops and conditionals. It fails the moment you need to understand systems.

When you write this in C:

```c
int x = 1000;
```

The compiler does something specific. It finds four contiguous bytes. Let's say they start at address 1000. Now the integer 1000 must fit inside these four bytes.

In hexadecimal, 1000 is 0x000003e8. Four bytes: 0x00, 0x00, 0x03, 0xe8. The compiler must decide which byte goes where.

On a little-endian machine:

```text
0xe8 0x03 0x00 0x00
```

On a big-endian machine:

```text
0x00 0x00 0x03 0xe8
```

Same variable. Same value. But the CPU that executes your code has a preference of byte order while storing it into the memory, wired into silicon. Your code does not decide. The machine does.

This is why endianness matters before you ever touch a network protocol. It matters the moment you take the address of a variable and look at what is actually there.

Consider this code:

```c
int x = 0x12345678;
char *p = (char *)&x;
printf("%02x\n", *p);
```

You are asking: what byte lives at the address where x begins?

On a little-endian machine, you get 0x78. On a big-endian machine, you get 0x12. Same variable, same code, different output. The only difference is which direction the machine chose to arrange the bytes.

Most curricula skip this when they introduce you to C. They teach variables as atomic entities. Indivisible. The abstraction works until you cast a pointer or dump memory or write data to a file. Then the abstraction breaks, and you discover that integers have always been sequences, and sequences have always had a direction.

Once you see that memory has a shape, endianness stops being a quirk. It becomes inevitable. The machine must put the bytes somewhere.

The only doubt is, "But in what order?"

And the answer is:

***whichever order the hardware designer chose, decades ago, for reasons that are now historical footnotes.***

## Big-Endian, Little-Endian, and the Human Bias

***Big-endian stores the most significant byte first.*** When you dump memory in hexadecimal, big-endian integers look the way you expect. Read the bytes left to right and they form a number you recognize. This makes debugging easier. You see 0x00 0x00 0x03 0xe8 and you know immediately it is a small number, probably around 1000. The leading zeros give it away.

***Little-endian stores the least significant byte first.*** The same number appears as 0xe8 0x03 0x00 0x00. Read left to right and it looks backward. You see 0xe8 first and think the number is large. You must reverse the bytes in your head to understand what you are looking at.

Here is a simple way to see this difference:

```c
uint32_t x = 0x12345678;
uint8_t *p = (uint8_t *)&x;
for (int i = 0; i < 4; i++) printf("%02x ", p[i]);
```

On a big-endian machine, you see: 12 34 56 78. The number reads naturally, just as you wrote it. When you look at a memory dump, the value 0x12345678 appears exactly as expected. This is why debugging on big-endian systems feels more intuitive. The hex dump matches your mental model.

On a little-endian machine, you see: 78 56 34 12. The bytes are reversed. Every time you examine memory, you must mentally flip the sequence to understand the value. This creates cognitive friction during debugging sessions.

Humans have a bias toward big-endian, because it matches how we write.

When you see 0x12345678 in a debugger, you expect the byte at the lowest address to be 0x12. On a little-endian machine, it is 0x78. This mismatch is small, but it compounds. After hours of debugging, the constant mental translation wears on you.

Yet most modern hardware is little-endian. Intel processors use it. AMD processors use it. ARM processors default to it. Why?

Little-endian has a technical advantage.

It makes certain operations faster. When a processor performs arithmetic, it starts with the least significant byte. On a little-endian machine, that byte is already at the lowest address. The processor can begin work while still loading the rest of the number. This saves cycles in critical paths.

Little-endian also simplifies pointer casting in subtle ways:

```c
uint32_t x = 255;
uint16_t y = *(uint16_t *)&x;
uint8_t z = *(uint8_t *)&x;
```

On a ***little-endian*** machine, y is 255 and z is 255. The least significant bytes are at the start, so truncating by casting just works. On a big-endian machine, y would be 0 and z would be 0, because you are reading the high-order zeros.

This is not necessarily better. It is just different. But it explains why certain low-level operations feel more natural on little-endian architectures. ***The address of a value and the address of its least significant portion are the same.***

***Big-endian*** has no such technical wins. Its ***advantage is purely human readability***. When you debug a network packet or examine a binary file, big-endian lets you see the structure immediately. You do not need to translate. The bytes appear in the order you expect.

Consider reading a file header:

```c
uint32_t magic, size;
fread(&magic, 4, 1, fp);
fread(&size, 4, 1, fp);
```

If the file uses big-endian and you examine it in a hex editor, you see the magic number and size exactly as specified. If the file uses little-endian, every multibyte value appears reversed. Documentation might say the magic number is 0x89504E47, but in the file you see 47 4E 50 89. You must constantly verify you are reading the bytes in the correct direction.

The internet chose big-endian for exactly this reason.

RFC 1700 declared it network byte order. This was partly because it is human-readable and partly because several influential architectures of the time used it. The decision was made in the 1980s. It is now permanent.

What is interesting is that this choice creates a permanent translation layer.

Every little-endian machine on the internet must convert to big-endian when sending data and convert back when receiving:

```c
uint16_t port = 8080;
uint16_t network_port = htons(port);
send(sock, &network_port, 2, 0);
```

This happens billions of times per second, on every device connected to a network. The cost is negligible in CPU cycles.

But it is a constant reminder that two worlds exist, and they do not naturally align.

It exists because two equally valid choices were made by different groups of engineers, and now every program that crosses the boundary between those choices must pay the conversion tax.

This is the human bias at work.

We want computers to think like we do. We want numbers to appear in memory the way we write them on paper. Big-endian gives us that. But computers do not care about human intuition. They care about efficiency. Little-endian gives them that.

So we live in a world where both exist, and the programmer must know which one applies in which context.

Once you understand this, endianness stops being arbitrary.

It becomes a design choice with clear tradeoffs. Readability versus efficiency. Human intuition versus machine optimization. Neither is wrong. Both are valid.

The only mistake is forgetting that a choice was made.

## When the Choice Is Made

A common question haunts programmers when they first encounter endianness: "*when does the machine decide which byte order to use?"*

Is it at compile time, when your code becomes binary? Or at runtime, when the operating system loads your program into memory?

The answer matters more than it seems. It reveals something about the nature of compiled code and the contract between compiler and machine.

The compiler decides endianness.

When GCC compiles your code on an x86 machine, it knows that machine is little-endian. So it emits an instruction that loads the value 0x12345678 with bytes in little-endian order. The binary file itself contains 78 56 34 12 in the data section. When the CPU executes that instruction, it reads those bytes and reconstructs the value correctly, because it is also little-endian.

If you compile the same code on a big-endian machine, the compiler emits different bytes. It stores 12 34 56 78 in the binary. When that CPU loads the value, it reads the bytes in big-endian order and gets the same logical value: 0x12345678.

This is why cross-compilation is harder than it looks. If you compile on an x86 machine but target a PowerPC architecture, the compiler must flip its assumptions. It must emit big-endian bytes even though it is running on a little-endian host. The same source code produces different binary representations.

You can verify this yourself:

```bash
lscpu | grep -i endian
```

This has a surprising implication. An executable compiled for a little-endian machine will not run correctly on a big-endian machine, even if both machines use the same instruction set.

The operating system could load the program into memory, but every numeric constant would be interpreted backward. Every pointer offset would be wrong. The program would crash or produce garbage.

This is why binary compatibility matters. This is why you cannot take a Windows executable and run it on a Mac, even when both use x86. The instruction sets may overlap, but the assumptions about byte order, system calls, and file formats differ. The binary is not portable because it was compiled with specific assumptions about the target architecture.

So, Endianness is not a runtime decision. It is baked into the binary at compile time.

If you want to know the byte order of your machine, which compiler will be using, try the *lscpu* command:lscpu command

## The Moment of Collision

Endianness does not matter if your program lives alone on one machine. You store an integer one way. You read it the same way. The system is consistent. Confusing, perhaps, but consistent.

There is a classic example that captures this perfectly. It is called the ***NUXI problem***, and it shows exactly how byte order creates silent corruption.

Imagine you store the string "UNIX" as two 16-bit values. Each character is one byte. The string becomes two shorts: "UN" followed by "IX". You write this code:

```c
short values[2];
values[0] = ('U' << 8) | 'N';
values[1] = ('I' << 8) | 'X';
```

On the machine where you write this, it works perfectly. You store "UNIX" and when you read it back, you get "UNIX". The machine is internally consistent.

But now you write these two shorts to a file. The file contains four bytes in sequence. On a big-endian machine, memory looks like this:

```text
U N I X
```

This makes sense. In the value "UN", the letter U carries more weight (it is multiplied by 256). So U appears first. Same for "IX". The letter I appears before X.

On a little-endian machine, the same code produces different memory:

```text
N U X I
```

This also makes sense, from the machine's perspective. Little-endian stores the small byte first. In "UN", the letter N is the low byte. So N appears first. Same for "IX". The letter X appears before I.

Both machines are correct. Both will read back "UNIX" if you ask them to load the shorts. They know their own byte order and compensate automatically.

The problem appears when you move the file. You write it on a big-endian machine, which stores U N I X. You read it on a little-endian machine, which interprets those four bytes as two shorts and gets N U X I. The string "UNIX" has become "NUXI".

The bytes are identical. The interpretation changed.

This is exactly what happens when binary data crosses architecture boundaries. File formats that do not specify byte order. Network protocols that assume everyone uses the same endianness. Disk images moved between machines. Every case where raw bytes are stored and later interpreted without agreement on direction.

The moment endianness matters is the moment your program tries to talk to something else.

Consider another example.

Two computers. Your laptop runs x86-64 Linux. Little-endian. The other is an embedded device somewhere on a network. A router, a sensor, a legacy system. Big-endian. You have no idea which is which.

Your program needs to send a number. A file size, a port number, a record count. Let's say the number is 1000.

On your laptop, when stored as a 16-bit unsigned integer, the bytes are:

```text
0xe8 0x03
```

You send these bytes over a socket in order: 0xe8, then 0x03.

On the receiving device, the bytes arrive in the same order. But the device is big-endian. It reads multibyte values starting with the big byte. So it interprets the sequence as:

The device received 59,395 instead of 1,000.

Your program is now broken. The file it tries to open is the wrong size. The port is invalid. The buffer allocation fails. The crash happens downstream, in code that never touches networking, making the root cause invisible.

This scenario teaches endianness through pain.

But it teaches something deeper: the program did not fail because endianness is hard. It failed because the programmer did not know their code had assumptions. The code assumed bytes would be interpreted one way. The device assumed another. Neither assumption was communicated.

The ***solution*** is rarely to rewrite the program. It is to make the assumption explicit. To say: ***when I send data, it will be in network byte order, which is big-endian. On receipt, convert to host byte order***.

Functions like ***htons()*** and ***ntohs()*** exist only because this collision is inevitable. They convert between host byte order and network byte order. On a big-endian machine, they do nothing. The bytes are already correct. On a little-endian machine, they reverse the bytes.

```c
uint16_t value = 1000;
uint16_t network_value = htons(value);
send(sock, &network_value, sizeof(network_value), 0);
```

The names are intuitive. Host to network short. Network to host short. Once you know the pattern, you never forget it.

What is interesting is that once a protocol defines byte order, endianness becomes almost invisible. TCP/IP says: network byte order is big-endian. Every system that joins the network agrees to this. Convert on send, convert on receive. The collision is prevented by agreement.

But the agreement had to be made.

## What Pointer Arithmetic Actually Reveals

Most programmers learn pointer arithmetic as a mechanism. You have a pointer. You add a number. It moves. Add 1 to an int* and it moves by four bytes. Add 1 to a char* and it moves by one byte.

This is taught mechanically, without surprise. But surprise is hiding inside.

Pointer arithmetic is how endianness reveals itself. It is the tool that lets you see the bytes as they actually sit in memory.

Consider a structure:

```c
struct Header {
    uint16_t magic;
    uint32_t size;
};
```

When you serialize this structure to a file, endianness affects every multibyte field. If you write it on a little-endian machine and read it on a big-endian machine, both fields arrive backward.

The structure itself does not change. The bytes do not disappear. But their interpretation flips. What was 1000 becomes 59,395. What was a valid magic number becomes garbage.

This is the design problem that endianness creates.

If you write code that reads or writes raw binary data, you must think in terms of byte sequences, not types. You must be explicit about order. Code that assumes a particular byte order is code that will break when ported or when integrated with external systems.

For this reason, endianness should be taught alongside pointer arithmetic. Not as a separate topic, but as an inseparable part of understanding what the address-of operator means. You cannot truly understand &x until you understand that x is not an indivisible thing. It is a sequence of bytes with a direction.

## Why We Teach It Too Late

Endianness appears late in most curricula because it is categorized as systems programming. It shows up in courses on operating systems, networking, or embedded development. By that time, students have been writing code for months or years. They have built a mental model of how computers work. That model does not include byte order.

This timing creates a problem.

When you encounter a concept late, you do not integrate it. You append it. Endianness becomes a special case, a detail to remember when working with networks or files. It does not reshape your understanding of memory, because your understanding of memory is already formed.

The result is that most programmers think of endianness as niche. They know it exists, but they do not think about it unless forced. When forced, they look up the solution, apply it, and move on. The deeper insight never appear.

The point to remember is: memory is not transparent.

When you write a value, the machine makes decisions about how to store it. Those decisions are invisible when you read the value back on the same machine. But they become visible the moment you serialize data, send it over a network, or move it to different hardware.

Endianness is not the only such decision. Alignment, padding, struct layout, and word size are others. But endianness is the simplest and most common. It is the clearest example of the gap between abstraction and reality.

If you learn endianness early, you develop different intuition. You stop thinking of integers as atomic. You start thinking of them as byte sequences. You become comfortable with the idea that the machine has its own perspective on your data, and that perspective does not always match yours.

## Building Intuition Without Memorization

Many approach endianness by memorizing rules.

They memorize the below snippet to answer the question, "***What is endianness?***"

***Endianness is the byte-order convention used to store a multi-byte numeric value in memory. In big-endian systems, the most significant byte is stored at the lowest memory address. In little-endian systems, the least significant byte is stored at the lowest memory address.***

They repeat this until it sticks, then apply it to exam questions.

But real understanding comes from intuition. Intuition is the ability to predict what will happen in a new context without being told.

To build intuition for endianness, you must see it in action.

Write code that prints memory in hexadecimal. Trace through pointer arithmetic by hand. Use a debugger to examine memory and see the bytes as they actually sit in RAM. Serialize a structure to a file, then open that file in a hex editor and observe which bytes appear where.

The ***intuition*** you want to build is: ***whenever data crosses a boundary, byte order must be explicit. ***

This applies to networks, file formats, inter-process communication, anything.

It is universal.

Here is a small exercise that reveals endianness without memorization:

```c
uint32_t x = 1;
if (*(uint8_t *)&x == 1)
    printf("little-endian\n");
else
    printf("big-endian\n");
```

Once you write experimental code like this, to understand what resides in memory, endianness stops being abstract. It becomes something you can see and touch. And once you can see it, you cannot unsee it.

And once you cannot unsee it, you write code that respects what the machine is actually doing, instead of what you wish it were doing.

“*The first principle is that you must not fool yourself and you are the easiest person to fool..*"

                                                                                                               - Richard Feynman

Now here’s your challenge:

Pick one piece of code you’ve worked on that crosses a boundary - file, network, or IPC, and ask:

“Where am I assuming byte order instead of making it explicit?”
