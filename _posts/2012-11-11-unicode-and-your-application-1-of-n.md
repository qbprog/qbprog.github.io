---
title: "Unicode and your application (1 of n)"
date: 2012-11-11T13:59:00.002Z
slug: unicode-and-your-application-1-of-n
draft: false
layout: post
---

If you look around in the open source ecosystem, and also in the non-opensource one, you will find a good amount of applications that don't support unicode correctly.  

I also find that many programmers don't know exactly what "supporting unicode" means, so they end up not supporting it at all.  

Handling unicode in a correct way doesn't mean knowing all of the standard and/or transformation rules, but instead means making some choices and following them congruently across the application.  

Ignoring it will cause frustration for your users, specially when you are dealing with textual input and output.  

Some of the fundamental points of unicode support include:  

* Having a basic knowledge of what unicode means and of the commonly used encodings
* Choosing a proper string representation to use inside your application.
* Correctly handling textual input.
* Correctly handling textual output.

I won't write about unicode advanced text handling and transformations. At the end of this series you won't to be able to write an unicode text editor; instead your application will stop corrupting the user text files, at least :)  

Many concept here apply generically to any programming language, anyway I'll focus a bit more on C++ and eventually on the new C++11 unicode features.  
Also, I'll threat the thing from an high level point of view, so I won't discuss about the specific encoding representations, etc...  

### **Understanding UNICODE**

This is a really huge and confusing topic. But it's also a fundamental requisite for every programmer,nowadays.  

There is a lot of material on the Internet, both generic and focused to a specific programming language.  
[Wikipedia](http://en.wikipedia.org/wiki/Unicode) gives a good quick start.  
Oh, of course the [unicode consortium site](http://www.unicode.org/) is the most reliable source of information about this topic.  

Also, this article [by "Joel on Software"](http://www.joelonsoftware.com/articles/Unicode.html) gives a good introduction on unicode and code points.  
I suggest reading it, as it is a bit redundant (and more complete) in respect of what I'm writing in this first part.  

### **A quick overview of encoding and formats**

This would be really long to explain exactly and in a complete way. I'll write a quick resume of the concepts, focusing on some facts that people usually tend to ignore.  

From wikipedia : _**Unicode** is a [computing](http://en.wikipedia.org/wiki/Computing "Computing") [industry standard](http://en.wikipedia.org/wiki/Technical_standard "Technical standard") for the consistent [encoding](http://en.wikipedia.org/wiki/Character_encoding "Character encoding"), representation and handling of [text](http://en.wikipedia.org/wiki/Character_%28computing%29 "Character (computing)") expressed in most of the world's [writing systems](http://en.wikipedia.org/wiki/Writing_system "Writing system"). Unicode is a standard ._  
_The standard defines a (big) set of characters that can be represented by a computer system. The number of characters is, as today, 10FFFF16_  

A **code-point** is any acceptable value in the unicode character space. I.e. is any value that is defined by the unicode standard. That is, the range of integers from 0 to 10FFFF16  

Unicode code-points and characters are **theoretical,** defined by a set of tables inside an international standard.   

Not every **unicode character** is composed by a single code-point**:** the standard defines code-point sequences that can result in a single character. For example, a code-point followed by an accent code-point will eventually result in an accented character.  

Anyway, you will find that in real-word usage most of code-points represent a single unicode character. But, please, **don't take this as an assumption**, or your program will fail the first time it encounters a code-point sequence.  

An **encoding** is defined as the way you choose to represent your set of characters inside computer memory.  
**An encoding defines how you physically represent the theoretical code-point**.  

Given that you have `10FFFF16 `code points to represent, you need to choose a way to store them using your preferred data type.  For instance, you may choose to use an unsigned 32 integer. In this case you will need just one word per character. Instead, by choosing a 16 data type you will theoretically need two words per character, using an 8 byte type, you'll need up to 4 bytes per character.  

Encoding can have a fixed-length per code-point or variable length per code point.  

**Fact #1 :** It doesn't matter what encoding you are choosing, the important thing to remember is that you can have more than one **byte/word per unicode code-point.**  

To resume:  
\- A character is fully determined by one or more code-points (theoretical level)  
\- A code-point can be composed of multiple bytes or words, depending on the encoding (physical level)  

This has big implications, specially for C/C++ users, because special care is needed when doing string manipulation.  

Let's take a quick overview of the commonly used unicode and non-unicode encodings.  

#### **Code-page**

The code page it's a number that defines how a specific text is encoded.  
You can see the code-page as a number that identifies the encoding.  
Code pages usually indicate single-bytes character sets, but also can indicate multi-byte character sets and UNICODE ones.  

A code-page can identify a single-byte encoding (ANSI/ASCII) , a non-unicode multi-byte encoding or an unicode encoding.  

#### **Single-byte non-unicode encodings (ASCII,ANSI)**

In the early days computer used to represent strings as 7/8bit characters. That meant that you had only 255 possible different characters. Usually the first 128 (7 bits) where defined as a common set of characters (common English alphabetical and numeric characters).  
The higher 128 characters are actually different between languages. For instance french computer systems have a different 128-255 character set from Italian ones, and from English ones.  

There's a lot confusion of terms about ASCII,ANSI, and code pages. From a practical standpoint, we can say that:  

**Fact #2** : As today, the complete set of 255 characters that are allowed by the ASCII/ANSI encoding is defined by the code page. A single-byte code page binds each of the 0-255 values to a specific "theoretical" character.  
Each system has one default code page.In windows you can see your system default code page in the international options of the language.   

Using an ASCII encoding will limit your software to use at most 255 different characters inside a string.This also means that you will not be able to represent all the unicode characters.  

As stated above, code pages can even indicate a multi-byte or unicode encoding: I will indicate single-byte encodings as **ASCII code-paged** encodings.  

**Fact #3 :** single-byte Code-paged encodings are often referred as **ANSI encoding or ASCII encoding**s. While these terms are not fully correct, can be usually can be interpreted as **"8 bit single byte encodings"**. Each character is fully represented as 8 bit value, i.e. you have 1-byte 1- character encoding.  

This kind of encoding is commonly used as it perfectly fits the C "char" data type and the stdlib functions.  

#### **Multi byte non-unicode encodings**

for completeness, some code pages point to multi-byte encodings not defined by the unicode standard. These were used for oriental languages , that required more than 255 symbols.  

### **Unicode encodings**

Unicode encodings allow to represent each unicode code-point with one or more words of defined size.  

#### [**UTF-8 encoding**](http://en.wikipedia.org/wiki/UTF-8) :

The encoding uses an 8bit word, and uses a variable length scheme to represent a single code-point. Each code-point can use up to 4 bytes, but common ANSI characters usually require 1 byte.  

#### [**UTF16 encoding:**](http://en.wikipedia.org/wiki/UTF-16)

it is still a variable-length encoding. The first 65535 characters (_Basic Multilingual Plane_) are represented exactly with a single 16 bit value. Code-points with value greater than 0xFFFF use a multi-word surrogate sequence.  
Windows uses UTF16 as it's internal representation  

#### [**UTF32 encoding**:](http://en.wikipedia.org/wiki/UTF-32)

Uses 32bit word (4 bytes) to exactly represent each unicode character. While it can be used as internal presentation, it's not commonly used in text storage.  

#### **Unicode Endianness**

In non-8 bit encodings, you can store each word as BIG-ENDIAN or LITTLE-ENDIAN. So, you have UTF16-BE and UTF16-LE . These define the same encoding with different storage rules in the computer memory.The same applies for UTF32 and other non-8bit word encodings.  
 --  
As you can see, there are many ways to encode unicode and non-unicode text. Each encoding has it's specific rules. This leads to a simple, yet fundamental fact:  

**Fact #4:** **each transition from an encoding to another requires a conversion.**  

I.e. you cannot interpret an ASCII text as UTF8 without doing a conversion. This also holds true for text encoded using two different code-pages, and also for endianness. A conversion is required to go from UTF8 to UTF16 (you cannot just stick an additional 0 in front of each value).  

So, even if you have an 8-bit ASCII-codepaged text, you cannot use it as UTF8.  

Let me repeat that: **UTF8 is different from ANSI/ASCII text,** and unless you are in the lucky case of using only the first 128 values of the character set, you need a conversion for this case too.  

All of this may appear banal, but ignoring **Fact #4** is the common source of many kind of encoding problems (and also the motivation behind this article).  

### **Unicode support in different operating systems**

Windows support two different set of APIs : ASCII and UNICODE. Internally, the storage of choice is UTF-16, and ASCII strings get converted on-the-fly to UTF-16 ones.  
In reality , the APIs support three sets of APIs:  ASCII code paged, multi byte (by defining special multi-byte code pages) and unicode UTF-16.  
Also, Microsoft compiler defined wchar\_t as 16 bit type, so application using that type to store unicode strings (std::wstring for instance), will gain the benefit of direct API usage.  

Many Linux and FOSS software, being strongly based on C and stdlib, usually support UTF8.  
This can be done, while maintaining most source-compatibility , by enabling the correct locale in the C stdlib.  

Unfortunately the level of support in applications is limited to it's internal representation. Input and output issues are usually ignored.  

### **The C confusion problem**

In C we have a _char_ type , that is commonly used as string character type. The standard says that is must be of size = 1 byte. The identity 1 byte = 1 char was true until the advent of unicode, and it's still used in many places.C/C++ programmers still think as 1-byte = 1-char , and eventually think that this is true even for wide-chars.  

I strongly suggest to use BYTE instead of char when handling UTF8 unicode strings. So it will be clear that 1 byte doesn't necessarily represent a character/code-point.  
edit: as this sentence is not clear (thanks zokier), I will go in depth about this in the next articles.  

If you want to support unicode, the C char type doesn't represent a single unicode character. "char" is in fact a misleading name nowadays.  

**Wide char**   
C/C++ supports the `wchar_t `type. It is defined as a wide character type. Unfortunately it's size is compiler dependent.  

For example,you have  
MSVC = 2 byte  
GCC = 4 byte  

this will eventually cause many troubles when choosing an internal encoding for your program.  
Many "C" stdlib string functions are now available in the relative wide format, but this is not true for no-so-older compilers.  

In C++11 ,there exists better suitable character types (`char16_t`) , but you will have troubles in finding string handling functions anyway.  
We'll discuss these points in the next articles, but on the meantime remember that :  

**Fact 5:** using wchar\_t in your program doesn't make it correctly support unicode.  

\--  

In the next article, I will discuss how to choose an internal string representation and how to correctly handle it in C++.  

I promise to include some source code too!   