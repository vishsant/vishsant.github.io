+++
date = "2026-06-28"
draft = false
title = "strncpy() was never a string function. So Linux deleted it."
+++

Every C programmer learns the same lesson, in the same order. *strcpy()* is dangerous because it has no bounds - hand it a source longer than the destination and it writes straight off the end of your buffer. Then someone points at *strncpy()*, the one that takes a size argument, and the lesson lands: this is the safe one. Use the *n* version. The number is your bounds check.

That lesson is half right, and the missing half is where the problem is. *strncpy()* does bound the write. But on June 2026, the Linux kernel finished a six-year campaign to remove every last call to it. To understand why a function this old and this universally taught got deleted from the most-deployed codebase on earth, you have to go back to what the *n* actually meant.

## The Field

In early Unix, a directory was a flat file of fixed-size records. Each entry was sixteen bytes: a two-byte inode number followed by a fourteen-byte filename. Fourteen bytes, exactly - not "up to fourteen." The slot was a fixed-width field on disk, and a filename shorter than fourteen characters had to be padded so the record layout stayed predictable.

`strncpy()`'s behavior fits this job exactly. Read its real behavior against that purpose and every "quirk" turns out to be a feature:

```c
/* Copy src into a fixed-width dest field of n bytes. */
char *strncpy(char *dest, const char *src, size_t n);
```

It copies at most *n* bytes from *src*. If *src* is shorter than *n*, it pads the entire remainder of *dest *with \0 bytes - so the fourteen-byte slot is fully defined, every time. And if *src* is exactly *n* bytes or longer, it copies *n* bytes and stops. No terminator is added, because the field is fourteen bytes whether or not the name fills it; there is no room reserved for a NUL and none was ever promised.

This is a disk-format tool. It writes fixed-width records. It was doing its job perfectly in 1975. The mistake was ours: a decade later, C programmers looking for a "safer strcpy" saw a copy function that took a length, assumed the length meant safety, and started using it as a general string copier. The function never changed. Our expectations did.

## The Lie

The gap between what we think `strncpy()` promises and what it does fits in three lines:

```c
char buf[5];

/* copies 'h','e','l','l','o' - fills all 5 bytes */
strncpy(buf, "hello", 5);

/* there is no \0 - printf keeps reading */
printf("%s\\n", buf);
```

"hello" is five characters. *buf *is five bytes. *strncpy() *faithfully copies all five and now there is no sixth byte to hold the terminator. *buf* is a character array that is not a string, because a C string is defined as bytes followed by a \0, and this one has no \0. The copy reported no error. Nothing crashed. You have a live landmine.

The danger moved downstream, to the next read. *printf("%s"), strlen(), strcat()*, every standard string operation finds the end of a string by scanning for \0. With no terminator inside buf, the scan does not stop at byte five. It continues into whatever sits next on the stack, reading and printing memory that was never yours to read, until it touches a zero byte somewhere downstream. On a good day that is garbage on your terminal. On a bad day it is an information leak, reading adjacent secrets into your output. This is the classic out-of-bounds read, and it is not a misuse of* strncpy().* It is *strncpy() *behaving exactly as specified. The function traded a loud, local write overflow for a quiet, deferred read overflow that detonates in someone else's code path.

Compile that snippet with the address sanitizer and the tool catches the read the language won't:

```console
$ gcc -fsanitize=address -g overread.c -o overread
$ ./overread
==...==ERROR: AddressSanitizer: stack-buffer-overflow ...
    READ of size 7 at 0x... thread T0
    ...
    [32, 37) 'buf' <== Memory access at offset 37 overflows this variable
```

The READ past a 5-byte buffer is the whole story: the program tried to read off the end because the terminator *strncpy()* "should" have written was never there.

The padding behavior hides a second issue. Because `strncpy()` zero-fills all unused bytes, copying a short string into a large field is not cheap:

```c
char line[4096];

/* copies 2 bytes, then zeroes 4094 more */
strncpy(line, "ok", sizeof(line));
```

Two useful bytes, and the function dutifully writes four thousand and ninety-four zeros after them, every call. You reach for *strncpy() *believing it is the careful, defensive choice. Often it manages to be both unsafe and slow at once: unsafe when the source is long, wasteful when it is short.

## The Fix That Wasn't

The systems world noticed the* strncpy()* trap decades ago, and in 1998 Todd Miller and Theo de Raadt shipped a replacement in OpenBSD: *strlcpy()*. Its contract is the one people wanted all along - always leave the destination a valid, NUL-terminated string, and tell the caller how much room they needed.

This is a genuine improvement, and for twenty-five years it was the answer most experienced C programmers gave. But the return value hides the flaw. *strlcpy()* returns the length of the source,* strlen(src)*, so the caller can detect truncation. To compute that length, *strlcpy()* has to scan the entire source string, all the way to its terminator, even when the destination is tiny and most of those bytes will never be copied.

That scan is itself an unbounded read. If *src* is not actually NUL-terminated, say a fixed-width field copied by, of all things,* strncpy()*, then *strlcpy()*'s *strlen()* runs off the end of the source looking for a zero that is not there. The function written to cure the* strncpy()* overread can commit the same overread, just on the source side instead of the destination. The fix had a blind spot exactly the size of the problem it was solving.

Each "safe" string function in C fixed the previous one's visible failure and introduced a new, subtler one. *strcpy()* had no bounds. *strncpy() *added a bound but dropped the terminator. *strlcpy()* restored the terminator but reintroduced an overread to compute its return value.

The bug was never in any one function. The bug is that a C string does not know its own length.

## The Function That Listens

The Linux kernel could not ship a string copier with a known overread, a missing terminator, or a hidden full-source scan. So in 2015, kernel 4.3 introduced a fourth function, designed by reading the failures of the first three and refusing all of them:

```c
ssize_t strscpy(char *dest, const char *src, size_t count);
```

`strscpy()` copies bytes until it hits the source's \0 or it has filled all but the last slot, then it always writes the \0 into that reserved last slot. So the destination is always a real string. That closes *strncpy()*'s failure. It stops at count and never runs a full* strlen()* of the source, so it cannot walk off the end of the source the way *strlcpy() *can. That closes *strlcpy()*'s failure too. (To be precise: for speed, *strscpy() *reads the source a machine word at a time, so it may touch a few bytes past the copied content within the same aligned word - bounded, never across a page boundary, and never the unbounded source scan *strlcpy()* performs. KASAN once flagged exactly this, which is how I got to know the boundary is real.) And it returns the number of bytes copied on success, or the negative error code -E2BIG when the source did not fit. Truncation stops being a silent event the caller must chase with a second *strlen()*: *strscpy() *returns it as a value to check.

The kernel's own documentation spells out the hierarchy. From `Documentation/process/deprecated.rst`, the file that lists the functions you are no longer allowed to use:

```text
strcpy()    — deprecated; performs no bounds checking.
strncpy()   — deprecated on NUL-terminated strings; does not guarantee termination.
strlcpy()   — deprecated; reads the source in full (may read past the buffer).
Use strscpy() instead.
```

## The Six-Year Delete

Knowing the right function is the easy part. The work is removing tens of thousands of existing calls from a codebase the size of Linux without breaking the millions of devices that run it.

*strcpy() *went first. Then *strlcpy()*: four years of coordinated conversion, finishing in the 6.8 cycle. The removal immediately broke out-of-tree code. DPDK, VirtualBox, and other drivers that still called strlcpy() failed to build against 6.8 and had to be patched, which is exactly the cost of a real deprecation, paid in the open. *strncpy()* was last and largest: roughly 362 patches from around seventy contributors over six years, landing in the Linux 7.2 merge window on June 2026. Much of it driven by Coccinelle scripts but a great deal of it requiring a human to look at each use and decide which of the purpose-built replacements actually matched the intent:* strscpy(), strscpy_pad(), strtomem()*, or just *memcpy()*.

One function took six years to delete because strncpy() was doing several different jobs at its various call sites. Sometimes it copied a string. Sometimes it filled a fixed field. Sometimes it deliberately left a hardware structure unterminated. A safe removal had to recover the original intent at each one. You cannot mechanically replace a function whose meaning depends on context. That made the campaign six years of archaeology rather than a find-and-replace.

That's what software maintenance looks like when it's done right.
