---
title: "Unicode and your application (2 of n)"
date: 2012-11-14T20:38:00.001Z
slug: unicode-and-your-application-2-of-n
draft: false
layout: post
---

In this part I will write some speculations about the C++ string encodings and the possibly required conversions.  

_Post-editing note : after re-reading this many times, I see that it probably ended way more theoretical and demagogic than practical; instead of throwing it away, I'll publish it anyway since it poses some real issues.  
Part 3, that I'm editing right now, is much more practical :)_  

## Some clarifications

After writing the first article of this series, I got some feedback on reddit.  
I'd like to clarify some points and make a quick resume:  

* ASCII is the original commonly used 7-bit encoding with 128 characters.
* Extended-ASCII encodings define a maximum of 256 characters. ISO 8859-1 is an extended ASCII encoding, also called Latin1. The common Windows-1252 encoding defines 32 unused values in Latin1, but it's almost Latin1.
* Specially on Windows, the ANSI term is often referred as 8 bit single-byte encodings,even if this is not totally correct.
* A Code-page is used to specify the set of available characters and a way to encode them using a binary representation. "Character encoding" is a synonym for code-page. Each encoding or code-page has an identifier: UTF8, Windows-1252, OEM , ASCII are all encoding names.
* A system default code page is used to interpret 8-bit character data. Each system had a default code-page which is used to interpret and encode text files, user textual input, and any other form of text data. Applications can eventually change the default encoding used. C locales have a similar setting and behavior.
* UTF8 is backwards compatible with 7-bit ASCII data. It is not backwards compatible with any other 8 bit encoding, even the Latin1 one.
* UTF16 is a variable length encoding. It requires a single 16-bit word for characters is the BMP (Basic Multilingual Plane), but it requires two 16-bit words for characters in other Unicode planes.
* Characters and code-points are the same thing in Unicode. Multiple code-points can be combined to form a "glyph", i.e. a "character displayed to the user", also called "grapheme cluster".  
  For instance, a letter code-point followed by an accent code-point , will result in an accented letter glyph.
* I didn't mention normalization forms at all. That's another important concept, worth a read.

Thanks to everyone who commented.  

## The compatibility problem

In C/C++ the historical data-type for string characters is **char**  
The char data type is defined by the standard as a 1-byte type. It may be signed or unsigned depending on the compilation settings.  
As we have seen in the previous post, a 8-bit data type is used by various character encodings:  

* UTF8
* ASCII and extended-ASCII encodings
* (less common) multi-byte non-unicode

Note that the "extended-ASCII" item in reality groups a lot of 8-bit encodings.  

In a C/C++ software, if you see code a piece of code like this:  

const char \* mybuf = getstring ();

how do you tell the effective encoding of the string?  

**You can't**, unless the getstring () function has an explicit documentation, somewhere else.  

In fact a char \* or std::string string can hold any text in ANY 8-bit encoding you want. And there are plenty of 8-bit encodings. Moreover,you can have a std::string with an UTF8 text, while another with an windows-1252 text, and another with a 7 bit ASCII encoding.  

Let's consider two simple implementations of the _getstring_ function  

```cpp
const char * getstring () 
{
   return "English!";
}

const char * getstring()
{
   return "Français!"
}
```

What is the encoding used in the returned strings?  

The English version string is composed by ASCII characters, so it's representation would be the same in any ASCII compatible encoding. The french version could result in an UTF8 encoded string or in a extended-ASCII encoded string, maybe with the latin1 encoding.  

At least, in this case, you can take a look at the functions, and deduce the argument type (eventually looking in the compiler documentation).  

Another example:  

```cpp
const char * getstring ()
{
    return external_lib_function();
}
```

In this case, you have to read the documentation and hope that it contains the encoding used by the function.  
One can deduce that is uses the standard encoding used by the C/C++ runtime.  

In C++11 you can find  

```cpp
const char * getstring ()
{
    return u8"捯橎 びゅぎゃうク";
}
```

in this case , it is sure that the returned string is encoded with UTF8.  

But, what if the external\_lib\_function would have returned a string like that?  

The point is that char strings can represent a wide variety of encodings. For each instance of std::string in your program, you need to know exactly what encoding is using that exact instance.  

Note: this holds true even for wide chars, even if it's mitigated by the relatively small number of 16-bit and 32-bit encodings.  

## Do these issues arise in reality?

At this point you may ask why bother with these issues at all. While the effective encoding may be clear to identify in a relatively simple software, when you have a good mix of libraries and components, things can get more subtle.  
In addition, utf8 literals will effectively make this worse. What one of your libraries starts using UTF8 literals, and the rest of your system does not? You could end having UTF8 std::strings and 8-bit std::strings in the same software. (note : this heavily depends on compiler and compilation settings)  

Just to name a few cases:  

* Your function (or compiler) generates UTF8 literals, while your locale is set to use a different encoding.
* Your function (or compiler) generates UTF8 literals and are using these as ANSI (say, calling an ANSI windows API).
* A library you use doesn't correctly handle ANSI file input, and returns them in ANSI format while your locale uses UTF8(and vice-versa)
* You write functions to handle UTF8 strings, but the strings are in a different encoding because of the system settings.

## Theoretical string handling and conversions

Theoretically, in function calls, each input string argument would require a conversion to the required input encoding, and each output string argument would require a conversion to the required output encoding.  
To keep things congruent, you will need to keep in mind :  

* The string encoding used by your compiler.
* The string encoding used by the libraries and external functions you are using.
* The string encoding used by the system APIs you are compiling for.
* The string encoding used by your implementation.

The conceptually correct way to move strings around would be using something like this:  

```cpp
typedef std::pair<encoding_t,std::string> enc_string_t;
```

i.e. bring the string encoding along with the string. The conversions would be done accordingly using a (conceptual) conversion like this:  

```cpp
enc_string_t convert (const enc_string_t & , encoding_t dest_encoding);
```

Fortunately you don't have to do all of these conversions every time, nor to bring along the string encoding. Anyway,tracking this aspect when writing code is useful to avoid errors.  

```cpp
const char * provider_returning_windows1252();
void consumer(const char * string_utf8);
```

Proper string handling would require a conversion (pseudo code):  

```cpp
const char * ansiStr = provider_returning_windows1252();
consumer(convert(ansiStr,UTF8));
```

of course, if you are sure that both the provider and the consumer use the same encoding, the conversion becomes an identity and could be removed.  

```cpp
const char * ansiStr = provider_returning_utf8();
consumer(ansiStr);
```

unfortunately, many times the conversion is omitted even if it is not an identity.  

```cpp
const char * ansiStr = provider_returning_windows1252();
consumer(ansiStr);
```

in this case, the consumer is called with a different type of string and will probably lead to corruption, unless the ansiStr contains only characters in the range 0-127.  
So, how can we avoid this kind of problems?  
Unfortunately the language doesn't help you.  

```cpp
std::string utf8 = convert_to_utf8(std::string & ascii);
std::string ascii = convert_from_utf8(std::string & utf8);
```

as we have seen the data-type is shared.Maybe this would be better to understand:  

```cpp
std::basic_string<utf8char_t> utf8 = convert_to_utf8(std::string & ascii);
std::string ascii = convert_from_utf8(std::basic_string<utf8char_t> & ascii);
```

for the same reason, people tend to ignore the UTF8->ASCII conversion, which in the worst case is being LOSSY.  
I would really have liked to see an utf8\_t type in the new C++, at least to give programmers a choice to explicitly mark utf8 strings. Anyway, nothing prevents you to define and use such a type.  
Despite language support, to detect the conversions need one has to look and edit the code  

* Review all the library usage and calls, and make sure of the string type and encoding it should use. This include external libraries and operating system APIs.
* Place appropriate conversions where needed or missing.
* Enforce a single encoding in your application source code.

this latter aspect is important to remove the ambiguities at least inside the controlled part of the source code and make it more readable.  

## Mitigation factors

All of these potential conventions may appear a bit scaring and heavy to handle, specially for existing code. If you look in open source software (or even closed one), you will hardly find any explicit kind of handling.  
This is because there exists some mitigation factors that in fact remove the need of the conversions.  

* If all the libraries use the same encoding and rules there's no need for conversions at all. This is very common, for instance, in C/C programs. Modern linux systems use UTF8 as default locale encoding, so if you accept this, you just to make sure that your input/output is UTF8 too, and that the libraries you use follow the same conventions.  
  Unfortunately this heavily depends on the compiler and eventually on the system settings.

* If you use an application framework, which already choose a specific encoding and does everything for you, the need of conversions is limited.

* If you don't explicitly generate or manipulate the strings of your program, these aspects could be ignored.  
  For instance,
  
  ```cpp
  int main (int argc, const char [] * argv)
  {
  
      FILE * f = fopen(argv[1],"r");
  
  ...
  ```
  
  in this case, you are just passing the input argv\[1\] string to the fopen function. So, knowing that both the arguments use the "C" locale setting, these already match and no conversion is needed.  

In the end you'll probably simplify the 90% of the possibly required conversions.  

## Conclusions

In writing this article , to illustrate the points, I took the "always convert" approach, excluding later the identity conversions.  
When coding, you usually take the opposite approach, because most of the conversions are not needed in practice. Anyway, considering and thinking about them is always important, specially when doing inter-library or API calls.  
In part 3 I will examine some of the peculiar aspects about unicode string handling in C/C++ , and some big differences across linux and windows compilers. You'll see that there are many cases where the encoding is not obvious, and eventually not coherent between compilers and systems.