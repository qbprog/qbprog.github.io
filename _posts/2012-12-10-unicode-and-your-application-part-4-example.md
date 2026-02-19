---
title: "Unicode and your application : part 4 example"
date: 2012-12-10T20:36:00.001Z
slug: unicode-and-your-application-part-4-example
draft: false
layout: post
---

I just wrote a simple example with Qt on how to read (and print-out) a text file. As I suggested in part 4, the program let the user choose the default encoding and eventually auto-detect it.  
In [this](https://github.com/QbProg/unicode-samples/) github repository I added the unicat application, which is a console application that can print a single text file.  
You can download the repository as ZIP file here: [https://github.com/QbProg/unicode-samples/archive/master.zip](https://github.com/QbProg/unicode-samples/archive/master.zip)  

## Usage

unicat \[--enc=<encoding\_name>\] \[--detect-bom\] <filename>

* If the application is launched without a filename, it prints a list of supported encodings.
* If --enc is not passed , it uses the system default encoding
* if --enc is passed , it uses the selected encoding.
* If --detect-bom is passed, the program tryes to detect the encoding from the BOM.If a BOM is not found, it uses the passed (or default) encoding.

The file is then printed to stdout.  

Note: on Windows, the console must have an Unicode font set, like Lucida Console, or you won't see anything displayed. Also , even with a console font , you won't see all the characters.  

The repository contains some text files encoded with various encodings. Feel free to play with these files and with the program options.  

## Building

To build the unicat sample, point open the .pro file with QtCreator or run qmake from the sources dir. To build it use make or nmake, or your preferred build command.  

## The code

The code is pretty simple. It first uses qt functions to parse the program options.  
It then reads the text file: Qt uses QTextCoded to abstract the coding and decoding of a binary stream to a specific string encoding.  

The function **QTextCoded::availableCodecs()** is used to enumerate all the possible encodings supported in the system.  

```cpp
Q_FOREACH(const QByteArray & B , QTextCodec::availableCodecs())
   {
       QString name(B);
       std::cout << name.toStdString() << std::endl;
   }
```

If the user passes the encoding, it tryes to load a specific QTextCoded from it, else it uses the default QTextCoded, which usually is set to "System"  

```cpp
if (encoding.isEmpty())
    userCodec = QTextCodec::codecForLocale();
else
    userCodec = QTextCodec::codecForName(encoding.toAscii());
```

After a file is open, the program uses a QTextStream to read it:  

```cpp
QTextStream stream(&file);
stream.setAutoDetectUnicode(detectBOM);
stream.setCodec(userCodec);
```

this is the initialization part, which specifies if a BOM should be auto-detected and the default encoding to be used.  
Reading the text and displaying it is just a matter of readline and wcout.  

## "Why are you using wcout?"

If you mind the previous parts, we have that the default std::string encoding is UTF8 on Linux, but ANSI (i.e. the active 8bit code page) on Windows. You can't do anything about this, since the CRT works that way.  
In addition, std::string filenames won't work on Windows for the same reason. So,in this example, the most portable way to print unicode strings is by using wcout + QString::toStdWString().  

There are also other problems with the Windows console: by default Visual C++ streams and windows console don't support UTF16 (or UTF8) output. To make it work you have to use this hack:  

` _setmode(_fileno(stdout), _O_U16TEXT);  `

this allows to print unicode strings to console. Keep in mind the considerations done in the previous chapter, since not all fonts will display correctly all the characters.  
BTW, I didn't find any problem in Linux.  

## Other frameworks

It would be nice to get an equivalent of this program (portable in the same way) using other frameworks or using only standard C++, to compare the complexity of each approach. Feedback is appreciated :)