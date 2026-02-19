---
title: "Unicode and your application (3 of n)"
date: 2012-11-18T14:43:00.001Z
slug: unicode-and-your-application-3-of-n
draft: false
layout: post
---

Let's go into the actual Unicode support among different compilers and systems. In these examples I mainly mention Visual C++ and GCC, just because I have them by hand.  
I'm interested in doing a Clang+MAC comparison too,but I don't have such a system avaiable. Comments are welcome in this regard :)  

## Character types

C++ defines two character types: **char** and **wchar\_t.**  

**char** is 1 byte as defined by the standard. wchar\_t size is implementation dependent.  
These two different types allow to make a distinction between different types of strings (and eventually encodings).  

The first important thing to remember : wchar\_t can have different size in different compilers or architectures.  

For instance, in Visual C++ it is defined as a 16-bit type, in GCC is defined as a 32-bit type. So, in Visual C++ wchar\_t can be used to store UTF16 text, in GCC can be used to store UTF32 text.  

C++11 defines two new character type: char16\_t , char32\_t , which have fixed sizes on all compilers. But at the moment there is no standard library function support for them. You can use these two types to write more portable string and character functions.  

## String literals

C++98 compilers support narrow character literals and wide strings.  

```cpp
const char * narrowStr = "Hello world!";
const wchar_t * wideStr = L"Hello world!";
```

* Question 1 : what is the encoding of the text inside string literals?
* Question 2 : what is the encoding of the text inside wide string literals?

Answer for 1 and 2 : **it's compiler specific.**  

The encoding may be an extended-ASCII one with a specific code-page, it can be UTF16 , it can be anything else.  

In Visual C++ (from MSDN) : _For the Microsoft C/C++ compiler, the source and execution character sets are both ASCII._ In reality , it seems that the resulting string encoding is based on the compiling system code page, even if the source file is saved in UTF8. So, it's **absolutely not UTF8.**  

In GCC, on modern systems, the default encoding for char string literals is UTF8. This can be changed with the compilation option -fexec-charset.  

```cpp
 std::string narrow = "This is a string.";
 std::string narrowWithUnicode = "This is a string with an unicode character : Ц";
```

The first string compiles fine in both compilers, the second one is not representable in Visual C++, unless your system uses the Windows:Cyrillic code page.  

Let's consider wide strings.  

```cpp
 std::wstring wide = L"This is a wide string"; 
```

In Visual C++ wide strings are encoded using UTF16, while in GCC the default encoding is UTF32. Other compilers may even define wide strings as 8bit type.  

Along with new data types C++11 introduces new unicode strings. This is the way to go if you have to represent unicode strings and characters in a portable way.  

```cpp
 const char * utf8literal = u8"This is an unicode UTF8 string! 剝Ц";
 const char16_t * utf16literal = u"This is an unicode UTF16 string! 剝Ц";
 const char32_t * utf32literal = U"This is an unicode UTF32 string! 剝Ц";
```

In this way you are sure to get the desired encoding in a portable way. But, as we are going to see, you will lack portable functions that handle these strings  

You may have noticed that there's is no specific type for utf8 string literals. char is already good for that as it's fixed size. char16\_t and char32\_t are the relative portable fixed-size types.  
Note: as today Visual C++ doesn't support UTF string literals.  

As you can see, using char and wchar\_t can cause portability problems. Just compiling for two different compilers can make your string encoding different, leading to different requirements when it comes to the conversions mentioned in the previous part.  

## Character literals

Like strings, we have different behavior between compilers.  
Since UTF8 (and also UTF16) is a variable length encoding, is it not possible to represent characters that require more than one byte with a single character item.  

```cpp
  char a = 'a'; // ASCII ,ok
  char b = '€'; // nope, in UTF8 the character is 3 bytes
```

Keep in mind that you can't store unicode character literals that are defined with a length more than "1" byte/word. The compiler will complain, and depending on what you are using, the char literal will be probably widened from char to int.  

```cpp
  std::cout << sizeof('€') << std::endl;  // 4 in GCC
  std::cout << sizeof("€") << std::endl;  // 3 in GCC 
```

the "4" value comes out because character constant is widened to an int (with a warning), while the value 3 is the effective size required by UTF8 to represent that "€" character.  

Note that in Visual C++ all of this is not even possible, since the literal string encoding is the "ANSI" one. You'll have to use wide literals to obtain similar results.  
Note also that C++11 didn't define a new specific UTF8 character type, probably because a single byte wouldn't be sufficient in most cases anyway.  

The safest and portable way to store a generic unicode code-point in C++ is by using uint32\_t (or char32\_t) and use it's hex representation (or \\uxxxx escapes)  

Wide literals have this kind of issue, but with a minor probability,because BMP characters require a single 16-bit word. If you have to store a non BMP character, you will require a surrogate pair (two 16-bit words).  

## Variable length encoding and substrings

All of this leads to an interesting rule about unicode strings: _You can't do substring and subscript handling like you did before with fixed-length encodings._  

```cpp
   std::string S = "Hello world";
   char first = S[0];
   std::cout << "The first character is : " << first << std::endl;
```

This code works correctly only with a fixed-length encoding.  
If your UTF8 character requires more than 1 byte to be represented, this code does not work as expected anymore.  

```cpp
   std::string S = u8"€";
   char first = S[0];
   std::cout << "The first character is : " << first << std::endl;
```

you won't see anything useful here. You are just displaying the first of the 3 byte sequence required to represent the character.  
The same applies if you are doing string validation and manipulation. Consider this example:  

```cpp
std::string cppstr = "1000€";
int len = cppstr.length();
```

in GCC _len_ is 7 (because the string encoding is UTF8), in MSVC is 5 because it's Latin1.  
This specific aspect can cause much trouble if you don't explicitly take care of it.  

```cpp
bool validate ()
{
std::string pass=getpass();
if (pass.length() < 5)
   {
      std::cerr << "invalid password length") << std::endl;
      return false;
   }
return true;
}
```

This function will fail with an UTF8 string, because it will check the "character length" with an inappropriate function.  
std::length (or strlen) is not UTF8 aware.We need to use an UTF8 aware function as replacement.  

So, how do we implement the pass.length replacement? Sorry to say this, but the standard library doesn't help here. We will need to get an additional library to do that (like boost or [UTFCpp](http://utfcpp.sourceforge.net/)).  

Another problematic issue arise when doing character insertion or replacement.  

```cpp
std::string charToRemove("€"), the_string;
std::string::size_type pos;

if ((pos=the_string.find(charToRemove)) != the_string.npos)
  the_string.remove(pos,1);
```

now , with variable length characters, you'll have to do:  

```cpp
if ((pos=the_string.find(charToRemove)) != the_string.npos)
     the_string.remove(pos,strlen(charToRemove));
```

because charToRemove is not long 1 anymore.  
Note three things:  

* This is true even for UTF16 strings, because non BMP characters can take up to 4 bytes.
* In UTF32 , you won't have surrogate pairs, but still code-points sequences.
* You should not use sizeof('€'),because the size of characters with length more than 1 word is compiler defined. In GCC sizeof('€') is 4, while strlen("€") is 3.

Generally, when dealing with unicode you need to think about substrings, instead of single characters.  

I will not discuss the correct UNICODE string handling and manipulation techniques, both because the topic is huge and because I'm not really an expert in this field.  
The important thing is knowing that these issues exists and cannot be ignored.  

More generally, it is much better to get a complete unicode string handle library and use it to handle all the problematic cases.  
Really.  

## Wide strings

UTF16 is often seen and used as a fixed-length Unicode encoding. This is not true. UTF16 strings can contain surrogate pairs. Both in UTF16 and UTF32 you can find code-point sequences.  

As today the probability of finding these two situations is not very high, unless you are writing a word-processor, or an application that deals with text.  
It's up to you to define a compromise between effort and unicode compatibility, but before converting your app to wide chars, at least consider to use specific unicode library functions(and eventually stay UTF8).  

## Conversions

When using unicode strings you will to perform four types of possible encoding conversions:

* From single-byte code-paged encodings to another single-byte code-paged encoding
* From single-byte code-paged encodings to UTF8/16/32.
* From UTF8/16/32 to single-byte code-paged encodings (lossy)
* From UTF8/16/32 to UTF8/16/32

Be careful with the third kind of conversions, as it is a lossy conversion. In that case you are converting from an UNICODE encoding (with a million possible characters) to an encoding that only supports 255 characters.  
This can end up corrupting your string,as the non-representable characters will be encoded with an "unknown" character (i.e "?"), usually chosen from one of the allowed characters.  
While knowing how to perform these conversions can surely be interesting, you are probably not willing to implement them :)  

Unfortunately, once again, the C++ standard libraries don't help you at all.  
You can use platform specific functions (such as MultiByteToWideString), but the better option is to find a portable library that does it for you.  

Even after C++11, the standard library seriously lacks unicode support. If you go that way, you'll find yourself with portability problems and missing functions.  
For instance, being a good choice or not, if you end up using wide strings you'll find incoherent support to the wide versions of the C functions, missing functions, incomplete or buggy stream support.  

Finally, if you choose to use the new char16\_t or char32\_t types , support is still nonexistent today.  

## Windows specific issues

As we have seen, Visual C++ defines wchar\_t as 16 bit, and wide strings encoding as UTF16. This is mostly because the Windows API is based and works with UTF16 strings.  
For compatibility reasons,there are two different versions for each API function, one that accepts wide strings, one that accepts narrow strings.  

Here comes the ambiguity again: what is the encoding required by the windows API for it's char strings?  

From MSDN : **All ANSI versions of API functions use the currently active code page.**  

The program active code page inherits from the system default code page, which can be changed by the user. The active code page can be changed by the program at runtime.  
So, even if you internally enforce the usage of UTF8 you will need to convert your UTF8 strings to an "ANSI" encoding, and probably loose your unicode data.  

Visual C++ narrow strings are ANSI strings. This is probably because they wanted to keep compatibility with the ANSI version of the API.  

**Note:** I don't really see this as argument of complain. If you look back, this a more compatible choice than converting the whole thing to UTF8, which is in fact a completely different representation.  

**This is a big portability issue, and probably the most ignored one.**  

In Windows, using wchar\_t could be a good choice for enabling correct Unicode support. The consecutive choice would be using wchar\_t strings, but keep in mind that other operating systems and compilers may have more limited widechar support.  

In Windows , if you want to correctly support a minimum set of unicode , I'd suggest to:  

* Compile your program with UNICODE defined. This doesn't mean that you are forced to use wide strings, but that you are using the UNICODE version of the APIs
* Use wide API functions, even if you are internally using narrow strings.
* Keep in mind that narrow strings are ANSI ones, not UTF8 ones. This is reflected in all the standard library functions available in Visual C++.
* If you need portability don't use wchar\_t as type string, or be prepared to switch at compilation time. (Ironically enough , the infamous TCHARs could ease code portability in this case).
* Be careful with UTF8 encoded strings, because the standard library functions may not support these.

As today, if you keep using ANSI functions and strings, you will limit the possibilities for your users.  

```cpp
int main (int argc, const char [] * argv)
{
    FILE * f = fopen(argv[1],"r");
}
```

if your user passes an unicode filename (i.e. outside the current code page), your program simply won't work, because Windows it will corrupt the unicode data by converting the argument to an 8bit "ANSI" string.  

**Note:** in Windows it is not possible to force the ANSI API version to use UTF8.  

## A quick resume

I hope to have made it clear that there are many perils and portability issues when dealing with strings.

* Take a look at your compiler settings and understand what is the exact encoding used.
* If you write portable code, be careful at the different compiler and system behavior.
* Using UTF8 narrow strings can cause issues with existing non-UTF8 string handling routines and code.
* Choose a specific unicode string handling and conversion library if you are going to do any kind of text manipulation.
* In a theoretical way, each string instance and argument passing could require a conversion.The number and type of conversions can change at compile time depending on the compiler, and at runtime depending on the user system settings.

**What unicode support library are you using in your projects?**