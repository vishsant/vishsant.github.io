+++
date = "2026-05-31"
draft = false
title = "The Keys That Aren't Keys: What the Linux Line Discipline Does to Your Typing"
+++

There is a comfortable lie every developer carries about the terminal. The lie is that when you press a key, that key travels to your program. You type, bytes flow, your program reads them. Input is input.

For most of the keys you press, that is roughly true. For the most important ones, it is completely false. The keys that matter most - the password you type, the *Ctrl+C* that saves you from a runaway process, the *Ctrl+D* that ends a session, the backspace that fixes a typo - never reach your program as input at all. They are intercepted by a layer of kernel code that acts on them itself, and your program is simply handed the result.

That layer is the **line discipline**. The default implementation on Linux lives in a single file, ***drivers/tty/n_tty.c***, and by the end you will see that a terminal is not a channel that carries your keystrokes - it is an active agent that reads them, decides which ones are commands meant for it, and forwards only the leftovers to your program.

### The Password That Proves It

Run *sudo* anything. Run *ssh* to a host that wants a password. The prompt appears, you type, and the screen stays blank. No characters, no asterisks, no dots. Almost every developer has typed a password into that silence and assumed the program on the other end is being careful - reading each character and choosing not to print it.

That is not what happens, and the reason is the cleanest entry point into how the whole subsystem works. Echoing your keystrokes is not your program's job and never was. When your terminal is in its normal state, every printable character you type is written back to the screen by the line discipline itself - not by your shell, not by the program you are running. This is **what "echo" means** in a terminal: **the kernel layer takes the byte you typed and copies it to the output so you can see what you are doing.** In the source, that is the ***echo_char()*** path, and it ***runs only while a flag named ECHO*** is set. Your program is not involved in showing you your own typing.

So when a password prompt wants to hide your input, it does not "decline to print" anything. It reaches into the line discipline and clears that one flag. **ECHO **is** a single bit in a kernel structure **called** termios** that holds the entire configuration of the terminal. The program **reads the current settings** with **tcgetattr()**, clears the ECHO bit, and **writes** them **back with tcsetattr()** - in C it is almost literally *term.c_lflag &= ~ECHO*. From that moment the kernel stops copying your keystrokes to the screen. You type into the resulting silence, the program reads your password, and then it sets ECHO again to restore normal behavior. The asterisks you see in a browser password field are a courtesy the web platform chose to offer. The terminal, by deliberate design, shows you nothing - not even the count of how many characters you have typed.

This is the cleanest possible proof that the terminal is doing work on your behalf that you never asked for and rarely notice.

### The Keys That Are Really Commands

Once you accept that the line discipline is reading your keystrokes before your program does, a whole category of "keys" stops being keys. Press *Ctrl+C* and your program dies. You have done this ten thousand times and filed it under "*Ctrl+C* quits things." But *Ctrl+C* is not a quit command, and it is not even delivered to your program. What you sent was a single byte, 0x03. The line discipline caught that byte, recognized it as the configured interrupt character, and instead of placing it in the queue your program reads from, it did something far more violent: it looked up the foreground process group of the terminal and sent every process in it a SIGINT signal. In the kernel this is a helper called ***__isig()***, and it really is one line of consequence - ***kill_pgrp(tty_pgrp, SIGINT, 1)*** - the terminal driver reaching out and signalling a whole group of processes because you pressed one key.

The distinction matters because it explains every exception you have ever hit. A program that installs a SIGINT handler can catch that signal and refuse to die - which is why* Ctrl+C *sometimes does nothing.

### The Buffer Your Program Cannot See

There is one more thing the line discipline does that hides in plain sight every time you correct a typo. Type a long command, notice a mistake near the start, hold backspace, retype it, and press Enter. Your shell receives the corrected command. It never saw the mistake. It never saw the backspaces. It never saw any of the intermediate states of the line you were editing.

This is **canonical mode**, controlled by the** ICANON flag**, and it is the default. In canonical mode the **line discipline maintains a line buffer inside the kernel and does your line editing for you**. Backspace is the configured erase character; when the line discipline sees it, a function named eraser() removes the previous character from its internal buffer and rewrites the screen to match. Ctrl+U erases the whole line. Ctrl+W erases the previous word. None of these are sent to your program. **Your program is blocked in a read() that will not return until you press Enter**, at which point the line discipline hands over the single finished line and nothing else.

This is also the answer to why **interactive programs** feel different. A text editor, a pager, a REPL with arrow-key history - these cannot tolerate the kernel buffering and editing input on their behalf, because they need every keystroke the instant it happens. So they turn ICANON off and enter **raw mode**, and from that point** they receive each byte directly, do their own editing, draw their own screen, and take responsibility for everything** the line discipline used to handle. The reason a crashed editor leaves your terminal broken - no echo, Enter does nothing, output marches diagonally down the screen - is that it disabled ICANON and ECHO to do its job and died before restoring them. The termios state belongs to the terminal, not to the process that changed it, so a program that mangles it and forgets to change it back leaves the next program living in its mess.

### The Control Panel You Have Never Opened

Everything above is configuration, and you can see all of it at once. Run ***stty -a***. The output is the complete state of your line discipline: which character is interrupt, which is erase, which is end-of-file, and whether each flag - echo, icanon, isig, ixon - is on or off. A flag printed plainly means on; a flag with a leading minus means off. This single command is the dashboard for the kernel subsystem that has been silently editing your input your entire career.

Three flags carry most of the weight, and they are the mental model worth keeping. ECHO decides whether the kernel shows you your own typing - clearing it is how passwords hide. ICANON decides whether the kernel buffers and lets you edit a line before your program sees it - clearing it is raw mode, where every keystroke goes straight through. ISIG decides whether certain keystrokes become signals instead of bytes - clearing it is how editors keep Ctrl+C for themselves. Every confusing thing a terminal has ever done to you is some combination of these three flags being in a state you did not expect.

So the next time you type a password, remember what it actually means. And this is the second half of a story.

In the previous article I traced the physical path from your keyboard to your shell - the PTY, the master and slave ends, and the kernel sitting between them where I called the line discipline "one of the most quietly important pieces of code in the kernel." If you want the full context for where this layer lives, read it here: [**Your Terminal Is Not Your Shell In Linux**](/post/your-terminal-shell-linux/)
