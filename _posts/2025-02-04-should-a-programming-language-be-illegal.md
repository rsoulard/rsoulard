---
layout: post
title: "Should a Programming Language Be Illegal?"
date:   2025-02-26 10:44:00 -0500
---

# Should A Programming Language be Illegal?
A shallow dive into software exploits and memory safety.

<!--more-->

Last year, the United States government published a document titled *Back to the Building Blocks: A Path Toward Secure and Measurable Software*[^1]. Soon after its publication the internet was a buzz with rumors and jokes that the United States government would put a ban on the C programming language and its offshoot C++. While both languages are mentioned by name, the document does not outline any plans to make their use illegal. The document's purpose is more in line with encouraging the government, and contractors, to use programming languages that feature better memory safety by default. But why mention these programming languages specifically by name? What makes C and C++ unsafe in terms of memory and should we follow the government's choice and reduce our usage of them?

# Memory Safety
On the evening of November 2nd, 1988 the Morris Worm began its crawl across the internet. One of the handful of exploits that the worm took advantage of was a buffer overflow in the fingerd service.[^2] Finger, and its service fingerd, were written in the C programming language for the Unix operating system, similar to many other programs at that time. The purpose of finger was to provide information about users on a host including their name, contact and log in status. The Worm exploited the fact that the fingerd service would move string input from a client to a 512 byte buffer in memory, specifically on the stack, for further processing. The routine it used to move input on to the buffer was the C Standard Library's `gets()`[^3] function, which was common practice at the time. However, `gets()`does not check the size of the buffer it is moving data to when called. It will simply keep writing data until it reads in a newline character, even beyond the bounds of the buffer. Someone could craft an input string on a client machine that overwrites the return address to point to code that opens a remote shell. The remote shell provides further control of the machine running fingerd, allowing for the attack to be run again from a new client. This is what Robert Morris, creator of the Morris Worm, did to exploit finger.

One might assume the maintainers of the C Standard Library, which provides the `gets()` function, would push an update that fixes that oversight. After all that is what happens nowadays when an exploit is found. And eventually, the function was removed outright in 2011[^4]. A whole 23 years later. The very same exploit that Robert Morris used in 1988 could still be usable without issue; unless someone knew to look out for it when coding. Why keep a known, exploitable function available in your library for so long? The committee that sets the standards for the C programming language did not want to invalidate existing code and chose to deprecate it rather than remove it. This decision was revisited for the C11 standard in 2011 where it was finally removed [^4].

This is not the only place in the C programming language that is unsafe with memory. In fact, there are little to no safety features to stand in your way when it comes to memory in C and C++. And some developers prefer it that way. Other languages practice memory safety by default; some even go as far as checking all operations at compile time and refusing to build if any issues are perceived to be found. As an example, let us compare how we would create and read from a basic buffer on the heap in C#, a language with strong memory safety, and then once again in C.

In C#, we declare an array of 512 integers on the heap using the `new` keyword and specifying the size in the square brackets after the declared type.
```csharp
var array = new int[512];
Console.WriteLine(array[511]);

// Output
// 0
```
When we try to access the 512th element at index 511 (as C# and C use zero based indices), we get the value `0` back. C# will default all values in an array to the type's default value. If we attempt to go beyond the 512th element, at index 512 we get:
```csharp
var array = new int[512];
Console.WriteLine(array[512]);

// Output
// System.IndexOutOfRangeException: 'Index was outside the bounds of the array.'
```
C# prevents us from reading outside the bounds of the array by throwing an exception. Now let us look at these same examples but in C. In C, we declare an array of 512 integers on the heap using `stdlib.h`'s `malloc()` function. Malloc requires the size, in bytes, of the space to be allocated so `512 * sizeof(int)` is passed in to indicate we need 512 integers worth of memory. `malloc()` returns a pointer to the first address in memory that was allocated, so we store that as an integer pointer. It is generally a good idea to make sure the allocation was successful and did not return `NULL` using `assert()` from `assert.h` and then free the allocation after we are done with it using `free()`.
```c
int* array = malloc(512 * sizeof(int));
assert(array);
printf("%d\n", array[511]);
free(array);

// Output (May change dependent on operating system)
// -842150451
```
We were able to access the 512th element, but what we got back was not zero. In this case, we got back a value that Windows uses to denote memory that has been cleared but not yet used by the application. Imagine if windows had not cleared it! We could have potentially had a buffer that contains passwords or personal data from an earlier state of the program. And when we try to read past the 512th element:
```c
int* array = malloc(512 * sizeof(int));
assert(array);
printf("%d\n", array[512]);
free(array);

// Output (May change dependent on operating system)
// -33686019
```
We can access its value without any issue. This time, the value we got back is used by Windows to denote "fence" memory that exists around the bounds of allocated memory (to prevent reads and writes outside of the bounds of the array!). We can even push it even further:
```c
int* array = malloc(512 * sizeof(int));
assert(array);
printf("%d\n", array[600]);
free(array);

// Output
// 0
```
We can read well beyond the bounds of the array, up until the bounds in memory the operating system has set aside for our program. When we do read past that boundary, we finally get an exception:
```c
int* array = malloc(512 * sizeof(int));
assert(array);
printf("%d\n", array[4096]);
free(array);

// Output
// Exception thrown: read access violation.
```
But who knows what we could have read or even overwritten within the bounds of our program by that point. Hopefully, we are not giving users of our program direct control of how we index and write to our buffer.
# To C or Not to C
Given its history with allowing developers to make mistakes with memory, should developers consider stopping their use of C and other memory "unsafe" languages? In my opinion, the answer is no. You can still make unsafe memory choices in memory safe languages. Here is an example in C#:
```csharp
// Example adapted from https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/unsafe-code

var array = new int[512];

unsafe
{
    fixed (int* p = &array[511])
    {
        int* p2 = p;
        p2 += 1;
        Console.WriteLine(*p2);
    }
}

// Output
// 0
```
Granted, you have to jump through hoops to get to that point. But you can still do it and the application will not stop you until the operating system does. Just like C.

And on the other side of things, there are things developers can do in C and C++ that make it safer. `gets()` may have stuck around for 23 years, but `fgets()`, a version of the function that accounts for buffer size and allows reading from any file stream, was introduced in C99[^5] and was heavily pushed as a replacement for `gets()`. NASA notoriously has many rigorous standards for how their C code is written[^6]. Even with lives on the line, they still use C and C++ while sticking to their standards and practices on how to write safe and reliable code. The latest versions of C++ have also pushed new memory safety features such as smart pointers including `std::unique_ptr` which constricts ownership of a pointer to one owner at a time. Developers have the tools and the responsibility to make "unsafe" programming languages safe.

There will always be a place for "unsafe" languages. Whether its embedded solutions where every byte matters, games where latency is the biggest concern or legacy systems that would be too much of hassle to replace, C and C++ will continue to be employed. It is the responsibility of the developers, in any language, to make sure memory safety is upheld. Some languages make safety implicit like C#. Others require the developer to take an active role in memory safety. As for the future legality of "unsafe" languages, do not read too much into this post's title. It is just click bait after all.

[^1]: *Back to the Building Blocks: A Path Toward Secure and Measurable Software*. PDF file. February 23rd, 2024. https://commons.wikimedia.org/wiki/File:Back_to_the_Building_Blocks_-_A_Path_Toward_Secure_and_Measurable_Software.pdf
[^2]: Spafford, Eugene H. *The Internet Worm Program: An Analysis*. PDF File.  December 8th, 1998. https://spaf.cerias.purdue.edu/tech-reps/823
[^3]: Wang, Mengzhi. PDF file. February 18th, 2001. https://www.cs.cmu.edu/afs/cs/academic/class/15213-s02/www/applications/recitation/recitation4/A/r04-A-4up.pdf
[^4]: *Programming languages - C Technical Corrigendum 3*, ISO/IEC 9899:1999. https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1235.pdf
[^5]: *Programming languages — C*, ISO/IEC 9899:1999. https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1256.pdf
[^6]: *JPL Institutional Coding Standard for the C Programming Language*. PDF file. March 3rd, 2009. https://web.archive.org/web/20111015064908/http://lars-lab.jpl.nasa.gov/JPL_Coding_Standard_C.pdf
