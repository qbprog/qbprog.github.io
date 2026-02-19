---
title: "Unicode and your application (5 of 5)"
date: 2013-01-06T15:22:00Z
slug: unicode-and-your-application-5-of-5
draft: false
layout: post
---

Here it comes the last installment on this series: a quick discussion on output files, then a resume on the possible choices using the C++ language.  

## Output files

What we actually saw for input files applies as well to output files.  
Whenever your application needs to save a text output file for the user, the rules of thumb are the following:  

* Allow the user to select an output encoding the files he's going to save.
* Consider a default encoding in the program's configuration options, and allow the user to change it using the output phase.
* If your program loads and saves the same text file, save the original encoding and propose it as default.
* Warn the user if the conversion is lossy(e.g. when converting from UTF8 to ANSI)

Of course this doesn't apply for files stored in the internal application format. In this case it's up to you to choose the preferred encoding.  

The steps of text-file saving are the mirrored steps of text-file loading:  

1. Handle the user interaction and encoding selection (with the appropriate warnings)
2. Convert the buffer from your internal encoding to the output encoding
3. Send/save the byte buffer to the output device

I won't bother with the pseudo-code to do these specific steps, instead I have updated the unicode samples [on github](http://github.com/QbProg/unicode-samples/) with "uniconv" , an utility that is an evolution of the unicat one: a tool that let's you convert a text file to a different encoding.  

```batch
  uniconv input_file output_file [--in_enc=xxxx\] --out_enc=xxxx [--detect-bom] [--no-write-bom\]
```

it basically let's you choose a different encoding both for input and output files.  

#### To BOM or not to BOM?

When saving an UTF text file, it's really appropriate to save the BOM in the first bytes of the text file. This will allow other programs to automatically detect the encoding of the file (see previous part).  
The unicode standard discourages the usage of the BOM in UTF8 text files. Anyway, at least in Windows, using an UTF8 BOM is the only way to automatically detect an UTF8 text file from an "ANSI" one.  
So, the choice it's up to you, depending on how and where the generated files will be used.  
Personally, I tend to prefer the presence of BOM, to leave all the ambiguities behind.  
**Edit**: since the statements above seem to be a personal opinion, and I don't have enough arguments neither pro or against storing an UTF8 BOM, I let the reader find a good answer on his own! I promise I'll come back to this topic.  

## You have to make a choice

As we have seen in the previous article, there are a lot of differences between systems, encodings and compilers. Anyway, you still need to handle strings and manipulate them. So, what's the most appropriate choice for character and string types?  
Posing such a question in a forum or StackOverflow, could easily generate a flame war :) This is one of the decisions that depends on a wide quantity of factors and there's not a definitive choice valid for everyone.  

Choosing a string and encoding type doesn't mean you have to use an UNICODE encoding at all, also it doesn't mean that you have to use a single type of encoding for all the platforms you are porting your application in.  

**The important thing is that this single choice is propagated coherently across your program**.  

Basically you have to decide about three fundamental aspects:  

1. Whether using standard C++ types and functions or an existing framework
2. The character and string type you will be using
3. The internal encoding for your strings

Choosing an existing framework will often force choices 2) and 3).  
For instance, by choosing Qt you will automatically forced to use **QChar** and **UTF-16**.  
Choices 2) and 3) are strictly related, and can be inverted. Indeed, choosing a data-type will force the encoding you use, depending on the word size. In alternative one can choose a specific encoding and the data-type will be choosed by consequence.  

Depending on how much portable your software needs to be, you can choose between:  

* a narrow character type (char) and the relative std::string type
* a wide character type (wchar) and the relative std::wstring type
* a new C++11 character type
* a character type depending on the compilation system

Here it follows a quick comparison between these various choices.  

#### Choosing a narrow character type

This means choosing the `char/std::string` pair.  
Pros:  

* it's the most supported at library level, and widely used in existing programs and libraries.
* the data type can be adapted to a variety of encodings, both with fixed and variable length (eg Latin1 or UTF8).

Cons:  

* In Windows you can only use fixed length encodings , UTF8 is not supported unless you do explicit conversions.
* In linux the internal encoding can vary between systems, and UTF8 is just one of the choices. Indeed you can have a fixed and variable encoding depending on the locale.

Let me mark once again that choosing char in a Windows platform is a BAD choice, since your program will not support unicode at all.  

#### Choosing a wide character type

This means using `wchar_t `and `std::wstring`.  

Pros:  

* wide character types are actually more unicode-friendly and less ambiguous than the char data type.
* Windows works better( is built on!) with wide-character strings.

Cons:  

* the wchar\_t type has different sizes between systems and compilers , and the encoding will change accordingly.
* Library support is more limited , some functions are missing between standard library implementations.
* Wide characters are not really widespread, and existing libraries that choose the "char" data type can require character type conversions and cause troubles.
* Unixes "work better" with the "char" data type.

#### Choosing a character type depending on the compilation system

This means that the character type is #defined at compilation type and it varies between system.  
Pros:  

* the character type will adapt to the "preferred" one of the system. For instance, it can be wchar\_t on windows and char on unixes
* You are sure that the character type is well supported by the library functions too

Cons:  

* you have to think in a generic way and make sure that all the functions are available for both data types
* Not many library support a generic data type, and the usage is not widespread in unixes, more in windows
* You will have to map with a define all the functions you are using.

Have you ever met the infamous tchar type in Windows? it is defined as  

```cpp
#ifndef UNICODE
  #define TCHAR char
#else
  #define TCHAR wchar_t
#end
```

the visual C++ library also defines a set of generic "C" library functions, that map to the narrow or wide version.  
This technique can be used between different systems too , and indeed works good. The bad thing is that you will have to mark all your literals with a macro that also maps to char or wchar version.  

```cpp
#if !defined(WIN32) || !defined(UNICODE)
  #define tchar char
  #define tstring std::string
#else
  #define tchar wchar_t
  #define tstring std::wstring
#end
tstring t = _T("Hello world");
```

I never have seen using this approach outside Windows, but I have used it sometimes and it's a viable choice, specially if you don't do too much string manipulations.  

#### Choosing the new C++11 types

Hei, nice idea. This means using char16\_t and char32\_t (and the relative u16string and u32string).  
Pros:  

* You will have data-types that have fixed sizes between systems and compilers.
* You only write and test your code once.
* Library support will likely improve in this direction.

Cons:  

* As today, library support is lacking
* Operating systems don't support these data types, so you will need to do conversions anyway to make functions calls.
* The UTF8 strings still use the char data type and std::string, increasing ambiguity (see previous chapters).

## Making a choice,the opposite way

As we have seen in the part above, making the choice about the data type will lead you to different encodings in different runtimes and systems.  
The opposite way to take a decision is choosing an encoding and **then** infer the data-type from it. For instance , one can choose UTF8 and select the data type consequently.  
This kind of choice is "against" the common C/C++ design and usage,because as we have seen, encodings and data type tend to be varying between platforms.  
Still,I can say that is probably a really good choice, and the choice that many "existing frameworks" took.  
Pros:  

* You write and test code only once (!)

Cons:  

* Forget about the C/C++ libraries unless you are willing to do many conversions depending on the system (and loosing all the advantages).
* This kind of approach will likely require to use custom data types
* Libraries written using standard C++ will require many conversions between string types.

In this regard,let's consider three possible choices:  
**UTF8:**  

* You use the char data type and be compatible. I suggest not doing so to avoid ambiguity.
* In windows it is not supported, so you will require to convert to UTF16/wchar\_t inside your API calls
* In unixes that support UTF8 locales, it works out-of-the-box.

**UTF16:**  

* If you have a C++11 compiler you can use a predefined datatype, else you will have invent one of your own that is 16bits on all systems
* In windows , all the APIs work out of the BOX, in linux you will have to convert to the current locale (hopefully UTF8).

**UTF32:**  

* Again, if you have C++11 you can use a predefined datatype, else you will have invent one of your own that is 32bits on all systems
* On any system you will have to do conversions to make system calls.

This approach, taken by frameworks such as Qt, .NET , etc... requires a good amount of code to be written. Not only they choose an encoding for the strings, but also these contain a good number of supporting functions, streams, conversions,etc...  

## Finally choosing

All of this seems like a _[cul de sac](http://en.wikipedia.org/wiki/Cul-de-sac)_ :)  
To resume, one has either to choose a C++ way of doing things that is very variable between systems, or a "fixed encoding" way forced to use existing frameworks.  

Fortunately the C++ genericity can abstract the differences, and hopefully standard libraries will improve unicode support out of the box. I don't expect too much change from existing OS APIs though.  
Still, I'm not able to give you a rule and the "correct" solution, simply because it doesn't exists.  
Anyway, I hope to have pointed out the possible choices and scenarios that a programmer can face during the development of an application.  

Now that you have read all of this, I hope that you are asking yourself these questions:  

* Do my programs correctly handle compiler differences about encodings and function behavoir?
* Do my programs correctly handle variable length encodings?
* Do my programs do the all the required conversions when passing strings around?
* Do your programs correctly handle input and output encodings for text files?

## Personal choices

Personally, if the application I'm writing does any text manipulation, I prefer to use an existing framework,such as Qt, and forget about the problem  
I really prefer the maintainability and the reduced test-burden of this approach.  
Anyway,if the strings are just copied and passed around, I usually stick with the standard C/C++ strings, like std::string or "tstring" when necessary.In this way I keep the program portable with a reduced set of dependencies.  
Finally, when I write Windows-only programs I choose std::wstring, but then I use APIs to do everything.  

## C++ standard library status : again

As we have previously seen, standard C++ classes are not really unicode friendly, and the implementation level varies very much (too much!)between systems. Anyway if you are using a decent C++11 compiler you will found some utility classes that let you do many of these operations by only using standard classes.  
I will write an addendum soon on this blog, and I will update the two unicode examples using C++11 instead of Qt, trying writing them in the most portable way possible.  

## Conclusions

I hope to get some feedback about all of this: I'm still learning it, and I think that shared knowledge always leads to better decisions. So, feel free to write me down an email or a comment below!