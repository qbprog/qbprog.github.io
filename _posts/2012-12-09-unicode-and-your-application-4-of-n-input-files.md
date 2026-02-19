---
title: "Unicode and your application (4 of n) : input files"
date: 2012-12-09T16:02:00.003Z
slug: unicode-and-your-application-4-of-n-input-files
draft: false 
layout: post
---

I was already writing part 4 about choosing a good internal string encoding. Then I realized that a piece was missing: as a reader, I would like to know about all of the issues before making such a choice.  
So here it comes the most interesting part of all these disquisitions: input and output.  

This part is mostly generic, you won't find too much code inside.  
This is because the implementation varies between libraries and frameworks. Like in the previous parts, standard C++ is not of much help here.  

## Reading a text file

Let's suppose for a moment that you did choose an internal encoding for your strings and you are willing to read a user text file and display it.  

This is the example taken from cplusplus.com:  

```cpp
// reading a text file
#include <iostream>
#include <fstream>
#include <string>
using namespace std;

int main () {
  string line;
  ifstream myfile ("example.txt");
  if (myfile.is_open())
  {
    while ( myfile.good() )
    {
      getline (myfile,line);
      cout << line << endl;
    }
    myfile.close();
  }

  else cout << "Unable to open file"; 

  return 0;
}
```

Do you know what?  

**This is wrong. Really wrong!**  

Really, it is wrong:  

* The program ignores completely the text encoding of example.txt.
* The program assumed that example.txt file is in the same encoding as your C locale. Eventually this may be the same as your system default encoding.
* As we saw in previous parts, this program behaves in a completely different way between systems (Windows, modern linuxes, older linuxes).
* Even if the text file encoding matches the system and C locale one, you are implicitly forcing the user to use that specific encoding.

Given that one knows all the implications,this piece of code may be good to read internal configuration files.  
But absolutely not to read user text files.  
With _user file_ I mean every file that is not under direct control of the application,for instance any file opened via command line or common dialogs.  

## A matter of encodings

This leads to two different and fundamental questions:  

* What is the encoding of the user file?
* How do I read a text file once I know it's encoding?

If you take a look inside the [wikipedia page of “character encodings”](http://en.wikipedia.org/wiki/Character_encoding#Common_character_encodings), you see that the lists is very long.  
Why should your program support all these encodings?  
Could we force the user to use a specific encoding for the files passed to our application?  

**No! This is wrong (again).**  

Please don't do it: **you have to let the user choose it's preferred encoding.** .  
Don't make assumptions and don't force it to use _your_ preferred encoding, just because you are not able to read the others! The user must have the freedom of choosing an encoding, mainly for two reasons:  

* The user **has** the right to choose it's preferred encoding. It may choose an encoding that looks wrong to you, but remember: you are writing a tool,the user is using it! :)
* Most users don't even know that encodings are, and they are not required to. I see that many programmers still don't know what encodings are, so why should end-users?

_rant-mode on_  
I see that this specific rule is often ignored in linux systems. At a certain point they decided to change the locale to UTF8, and by a strange transitive property, all the text files handed by command line utilities are required to be UTF8. Unfortunately, this is true even for commonly used tools, like diff (and consequently git). Bad, very bad. _rant-mode off_  

Again: please let you users to choose their preferred encoding whenever they need to pass a textual input to your program.  

When dealing with textual input files, consider these aspects:  

* Most text files in Windows systems are encoded with a fixed 8-bit encoding. So, these files look good only in systems with the same code-page.
* Most text files in Linux are encoded in UTF8 nowadays, but older systems used different encodings.
* Text files are likely to come from different systems, via Internet, so assumptions good 10-15 years ago, are not valid anymore. This may sound ridicoluous to say, but it's not so obvious for many. XML and HTML files already allow to specify any encoding, so do emails and many internet-related formats.

Of course, if you need to load/store internal data files, you can choose the encoding you like, probably one that matches your internal string encoding. At least, in this case, document it somewhere in the program documentation.  

## The input file encoding

Once you get convinced that the input encoding choice it's a user matter, we can go on with the first question: how to know the input file encoding?  
(if you are still not convinced , please go back to the previous section and convince yourself! Or just try to diff an UTF16 file using git and come back here.)  

There are two ways to obtain the input file encoding:  

* You let the user specify it
* Your program tries to auto detect it
* You assume that all the input files have a predefined-fixed encoding

To let the user specify it, you have to give it a chance to pass the encoding in some way; it can be a command line switch, an option in the common dialog, or a predefined encoding that can be changed later.  
Keep in mind that each text file can have a different encoding, so having a program default is good but you should allow changing it for the specific file.  

Let's see some examples:  

* LibreOffice let choose the encoding when opening the text file
* Qt Creator let you choose a default encoding in the program options, but let's you eventually change it later if it is not compatible with the contents of the file or if you see strange characters.
* GCC lets you choose the encoding of source files with a command line switch (-finput-charset=charset )

The number of character encodings is really high, and any of these programs uses a specific library or framework to support all of these. For instance, GCC uses libiconv, so the list of supported charset is defined [here.](http://www.gnu.org/software/libiconv/)  

## Auto-detecting the encoding

There are some ways to auto-detect the encoding of a text file: all of these are empirical, though they provide different levels of uncertainty.  

#### Byte order mask

There is a convention for UNICODE text files that allows to store the encoding used directly inside the text file: a predefined byte sequence is saved at the top of the text file.  
This predefined byte sequence is called **Byte Order Mask**, abbreviated **BOM**.  
Different byte sequences specify different encodings, following these rules:  

* Byte sequence :
  
  `0xEF 0xBB 0xBF`
  
  the file is encoded with UTF8

* Byte sequence :
  
  `0xFE 0xFF`
  
  the file is encoded with UTF16 , big endian

* Byte sequence :
  
  `0xFF 0xFE`
  
  the file is encoded with UTF32 , big endian

* Byte sequence :
  
  `0x00 0x00 0xFE 0xFF`
  
  the file is encoded with UTF32 , big endian

* Byte sequence :
  
  `0xFF 0xFE 0x00 0x00`
  
  the file is encoded with UTF32 , little endian

(there are other sequences to defined the remaining UNICODE encodings [here](http://en.wikipedia.org/wiki/Byte_order_mark#Representations_of_byte_order_marks_by_encoding) ).  

When you open a text file, you can try to detect one of these sequences on the top of it and read the rest of the file accordingly.  
If you don't find a BOM, just start again and fallback to a default encoding.  

If you look at the byte sequences, you see that the used values can be valid values of other encodings.  
For instance, the UTF8 BOM can be a good character sequence using CP1252 (þÿ);what if the user text file contained exactly these characters at the top?  
You are unlucky, and you will interpret the text file in a wrong way. This is unlikely, at least not much probable , but it is not impossible.  

This is why I referred as these method as uncertain: one can't be 100% sure of the result,so even if you choose this auto-detection method it is wise to let the user override the detected encoding.  

#### BOM Fallback

If you don't find a BOM inside the text file, you can eventually fallback to another detection method or use a default encoding. Keep in mind that the BOM is optional even for UTF16 and UTF32, and also is even discouraged for UTF8.  

* On windows system you can fallback to the system default code page, and read the file as ANSI.
* On linux systems, you can fallback to system “C” locale.
* You can use other autodetection methods.

#### Other autodetection methods

There are some other encoding detection methods.  
Some encodings have an unused range of values, so one could exclude these specific encodings if an invalid value is found.  
Unicode files can contain code-point sequences. Detecting an invalid code-point sequence, can let you exclude that specific encoding in the detection algorithm.  
Windows API has an infamous _IsTextUnicode_ function, that uses some heuristics to detect the possible file encoding.  
Other frameworks may have different detection functions.  

I advise against using these functions, since you are adding uncertainty to your input handling process. Nonetheless, the auto-detected encoding can be used as a proposed value in the user interface.  

Since none of these method is 100% safe (even the BOM one), always give more priority to the user choice.  

An interesting link in this regard is http://blogs.msdn.com/b/oldnewthing/archive/2007/04/17/2158334.aspx : how notepad tries to detect a text file encoding.  

## Conversion

So you got a possible encoding for your file. You have two ways to load it and convert it to your internal encoding.  

* A buffer based approach : you load the entire file as binary data, and perform a read.
* A stream based approach.

Here I propose a pseudo-code pieces that illustrates the steps to to handle textual input in your program, using a buffer based approach.  

```cpp
string_type read_text_file (std::string & filename)
{
 std::vector buffer = read_file_as_binary(filename);
 encoding_t  default_encoding = get_locale_encoding();
 encoding_t detected_encoding = try_detect_encoding(buffer);
 encoding_t user_encoding = get_user_choice ( /* det_encoding */ );
 encoding_t internal_encoding = /* get_locale_encoding () or UTF8 or ASCII */
 encoding_t input_encoding;

 if (user_encoding != UNK_ENCODING)
 {
  if (det_encoding == UNK_ENCODING)
   input_encoding = default_encoding;
  else 
   input_encoding = detected_encoding;

  if (is_unicode(detected_encoding) && has_bom(buffer))
   strip_bom(buffer);
 }
 else 
  input_encoding = user_encoding;

 string_type str = convert_encoding( input_encoding, internal_encoding,buffer);
 return ret;
}

string_type convert_encoding ( encoding_t input_encoding , encoding_t internal_encoding, std::vector buffer)
{
 adjust_endianness(input_encoding, buffer);
 if (input_encoding != internal_encoding)
  convert_cvt(input_encoding,internal_encoding,buffer);

 return string_type(reinterpret_cast< internal_char_t *>(buffer), buffer.size());
}
```

This code perform these operations:  

* Load the text file as untyped byte buffer.
* Try to detect the encoding, and eventually let the user choose it.
* If the buffer has a BOM (and the encoding is an UNICODE one), strip the BOM from the buffer.
* Call the convert function to obtain an internally-encoded string:
* If the endianness of the input file doesn't match the endianness of your system (when using a 16 or 32 bit encoding), adjust it and swap the bytes of the input buffer accordingly.
* If the input encoding doesn't match your internal encoding, convert it. This operation is complex to implement, and usually done by existing libraries.
* Finally, interpret the input buffer as your preferred character type, since it is converted to your internal encoding. You can construct a string from the buffer data.

Some notes:  

* In this case the default encoding is taken from the system. The program may have a configuration option to specify the default encoding.
* The detection routine may be simpler, and detect if the text is UNICODE or NOT. In that case you may treat the text as fixed 8-bit code page or UTF8.
* If you are using C++ data types and strings, keep in mind that the encoding used (internal encoding) can vary between system. As we saw in the previous part, in Visual C++ std::string is ANSI, in Linux is UTF8 nowadays. The same applies to wchar\_t.
* If you are using an 8bit data type which is UTF8, the text file can contain non-representable character, which your function will discard.

Some frameworks will do all of this for you, by letting you choose a text file and encoding, and resulting in a compatible string.  

## Detecting the BOM is easy.

As stated before, I won't propose an implementation for this pseudo code. Anyway, just to illustrate one of these function, here it comes a possible BOM detection routine, still buffer-based:  

```
/* Tryes to detect the BOM */
encoding_t try_to_detect_encoding ( const std::vector & buffer )
{
 /* tryes to detect the BOM */
 if (buffer.size() > 3 && buffer[0] == 0xEF && buffer[1] == 0xBB && buffer[2] == 0xBF) 
  return UTF8;
 else if (buffer.size() > 2 && buffer[0] == 0xFE && buffer[1] == 0xFF ) 
  return UTF16_BE;
 else if (buffer.size() > 2 && buffer[0] == 0xFF && buffer[1] == 0xFE ) 
  return UTF16_LE;
 ….. 
 else 
  return UNK_ENCODING
};
```

About the other functions, there are not many standard C++ utilities, at least not enough to handle all the required cases. In Windows, the convert functions may be implemented with **MultiByteToWideChar** , in Qt the **QTextCodec** class provides most of these. There are many toolkits, frameworks, and libraries to do this specific thing, so you just have to pickup your preferred one.  

## Using streams

Loading the entire file into a byte buffer can be unwanted and impossible in some cases, and you may want to use streams instead.  
C++ streams don't support unicode well. Some library implementation are better than others, but you'll end up with non-portable code.  
Also, remember that execution character set may be different between compilers, so streams potentially behave different with different data types. For instance, to write UTF8 data you still need to use wchar\_t in Windows, because char streams are ANSI anyway (even if your internal encoding is forced to UTF8).  
Since I don't usually use standard C++ streams to do textual input, I spent some time looking for some examples.I mostly found mostly platform specific code, workarounds, and custom implementations. I won't propose any of these in this post.  

## Using existing frameworks

By contrast, take a look how it is easy to read a text file using Qt.  

```cpp
int main()
{
QFile file("in.txt");
if (!file.open(QIODevice::ReadOnly | QIODevice::Text))
 return 1;

 QTextStream in(&file);
  in.setCodec(input\_encoding);  // "UTF-8" or any other...
     // in.setAutoDetectUnicode (true);
 while (!in.atEnd()) {
  QString line = in.readLine();
  process\_line(line);
    }
}
```

Qt also let's you do the auto-detection, with the setAutoDetectUnicode function.  

After many tests, implementation and experimentation , I've reached a conclusion: using existing frameworks to do textual input (and eventually output) is a really good choice.  
This can be really time saving, as you eventually call one function and get the result, or a specific stream that already works good with any text encoding.  
Usually I just need to set or detect the encoding and let the library do all the hard work.  
Why bother with something unportable and complex instead, like C++ iostreams ?  

## Conclusions

One may think that text files are the easier way to handle data, but it is clear that this is not the case anymore. Even reading a text file can be difficult and error-prone.  
Your users will thank you if you let them to make a choice (explicit or implicit).  
Existing frameworks will help you in this, Standard C++ support is still way behind. So what are you waiting for? Go on and replace your textual file input routines with better ones :)  

Next time: textual output.