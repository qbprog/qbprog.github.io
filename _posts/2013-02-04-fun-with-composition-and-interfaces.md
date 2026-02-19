---
title: "Fun with composition and interfaces"
date: 2013-02-04T20:59:00Z
slug: fun-with-composition-and-interfaces
draft: false
layout: post
---

As one of most important programming concepts is code reuse, let's see some nice ways to use C++ templates and make code-reuse simple and efficient. Here it follows a bunch of techniques that have proven useful in my real life programming.  

Suppose you have a set of unrelated classes, and you want to add a common behavior to each of these.  

```cpp
class A : public A_base
{

};

class B : public B_base
{

}
...
```

you want to add a set of common data and functions to each of these class. These can be extension functions (like loggers,benchmarking classes, etc...), or algorithm traits (see below).  

```cpp
class printer
{
public:
    void print ()
    {
      std::cout << "Print " << endl;
    }
};
```

The simplest way to accomplish this is using multiple inheritance:  

```cpp
class A : public A_base , public printer
{

}
```

That's easy , but the printer class won't be able to access any of the A' members. In this way it is possible to compose only classes which are independent from each other.  

## Composition by hierarchy insertion

Suppose we are willing to access a class member to perform an additional operation. A possible way is to insert the extension class in the hierarchy.  

```cpp
 template <class T>
 class printer : public T
 {
  public:
    void print ()
    {
         std::cout << "Class name " << this->name() << endl;
    }
 }
```

The class printer now relies on the "std::string name()" function being available on it's base class. This kind of requirement is quite common on template classes, and until we get _concepts_ we must pay attention that the required methods exists on the extending classes.  
BTW, type traits could eventually be used in place of direct function calls.  
The class can be inserted in a class hierarchy and the derived classes can access the additional functionality.  

```cpp
class A_base
{
public:
    std::string name () { return "A\_base class";}
}  

class A : public printer<A_base>
{

} ;

int main ()
{
    A a;
    a.print();
    return 0;
}
```

this technique can be useful to compose algorithms that have different traits, and share code between similar classes that don't have a common base.  
This last examples is not a really good one for multiple reasons:  

* In case of complex A\_base constructor, it's arguments should be forwarded by the printer class. Using C++11 delegating constructors will make things easier, but in C++98 you'll have to manually forward each A\_base constructor , making the class less generic.
* inserting a class in a hierarchy can be unwanted.
* If you want to access A (not A\_base) members from the extension class, you need to add another derived class, deriving from printer<A>

Anyway, this specific kind of pattern can still be useful to reuse virtual function implementations:  

```cpp
class my_interface
{
public:
   virtual void functionA () = 0;
   virtual void functionB () = 0;
};
```

```cpp
template <class Base>
class some_impl : public Base
{
     void functionA () override;
};

class my_deriv1 : public some_impl<my_interface>
{
   void functionB() override;
};
```

In particular, if my\_base is an interface, some\_impl can be used to reuse a piece of the implementation.  

### Using the [Curiously recurring template pattern.](http://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)

Now it comes the nice part: to overcome the limitations of the previous samples,a nice pattern can be used: the **Curiously recurring template pattern**.  

```cpp
template <class T>
class printer
{
public:
   void print ()
   {
       std::cout << (static_cast<T*>(this))->name() << std::endl;
   }
};
```

```cpp
class A : public A_base, public printer<A>
{
public:
   std::string name ()
   {
       return "A class";
   }
};

int main ()
{
     A a;
     a.print ();
     return 0;
}
```

Let's analyze this a bit: the printer class is still a template, but it doesn't derive from T anymore.  
Instead, the printer implementation assumes that the printer class will be used in a context where it is convertible to T\*, and will have access to all **T** members.  
This is the reason of the static cast to (T\*) in the code.  

If you look at the code of the printer class alone, this question arises immediately:  
_how comes that a class unrelated to T can be statically casted to T\* ?_  

The answer is that you don’t have look at the printer class “alone” : template classes and functions are instantiated on the first usage.  
When the print() function is called, the template is instantiated. At that point the compiler already knows that **A** is derived from **printer<A>** so the static cast can be performed like any down cast.  

As you can see, with this idiom you can extend any class and even access it's members from the extending functions.  
You may have noticed that the extension class can only access public members of the extending class. To avoid this , the extension class must be made friend:  

```cpp
template <class T>

class printer
{
 T* thisClass() { return (static_cast<T*>(this)) };

public:

   void print ()
   {
       std::cout << thisClass()->name() << std::endl;
   }
};

class A : public A_base, public printer<A>
{
  friend class printer<A>;

private:

   std::string name ()
   {
       return "A class";
   }
};
```

I've also added a thisClass() utility function to simplify the code and place the cast in one place (const version left to the reader).  

## Algorithm composition

This specific kind of pattern can be used to create what I call “algorithm traits”, and it’s one of the ways I use it in real code.  

Suppose you have a generic algorithm which is composed by two or more parts. Suppose also that the data is shared and is eventually stored in another class (as usual endless combinations are possible). Here I’ll make a very simple example, but I’ve used it to successfully compose complex CAD algorithms:  

```cpp
template <class T>
class base1
{
protected:
    std::vector Data();
    void fillData ();
};

template <class T>
class phase1_A
{
protected:
    void phase1();
};

template <class T>
class phase1_B
{
protected:
    void phase1();
};

template <class T>
class phase2_A()
{
protected:
   void phase2();
};

template <class T>
class phase2_B()
{
protected:
   void phase2();
};

template <class T>
class algorithm
{
public:
    void run ()
    {
          fillData();
          phase1();
          phase2();
    }
};
// this would be the version using the “derived class” technique
//public class UserA : public Algorithm<phase2\_a<phase1\_a<base1>>>;class comb1 : public algorithm<comb1>,phase1\_A<comb1>, phase2\_A<comb1>,base1<comb1> {};
class comb2 : public algorithm<comb2>,phase1_B<comb2>, phase2_A<comb2>,base1<comb2> {};
class comb3 : public algorithm<comb3>,phase1_A<comb3>, phase2_B<comb3>,base1<comb3> {};
class comb4 : public algorithm<comb4>,phase1_B<comb4>, phase2_B<comb4>,base1<comb4> {};
...
```

This technique is useful when the algorithms heavily manipulate member data, and functional style programming could not be efficient (input->copy->output). Anyway.... it's just another way to combine things.  
A small note on performance: the static\_casts in the algorithm pieces will usually require a displacement operation on this (i.e a subtraction), while using a hierarchy usually results in a NOP.  
This technique can also be mixed with virtual functions and eventually the algorithm implemented in a base class while the function overrides composed in the way I just exposed.  

## Interfaces

As we saw, extension methods allow to reuse specific code in unrelated extending classes. In high level C++, the same thing is often accomplished with interfaces:  

```cpp
class IPrinter
{
public:
  virtual void print () = 0;
};

class A : public A_base , public IPrinter
{
public:
 void print () override { std::cout << "A class" << std::endl; }
};
```

in this case each class that implements an interface requires to re-implement the code in a specific way. The (obvious) advantage of using interfaces is that instances can be used in an unique way, independently from the implementing class.  

```cpp
void use_interface(IPrinter * i)
{
  i->print();
}

A a;
B b; // unrelated to a
use_interface(&a);
use_interface(&b);
```

this it is not possible with the techniques of the previous sections, since even if the template class is the same, the instantiated classes are completely unrelated.  
Of course one could make use\_interface a template function too. Surely it can be a way to go, specially if you are writing code heavily based on templates. In this case I would like to find an high-level way, and reduce template usage (consequently code bloat and compilation times too).  

```cpp
class IPrinter
{
public:
  virtual void print () = 0;
};

template <class T>
class Implementor : public IPrinter
{
    T* thisClass() { return (static_cast<T*>(this)) };

public:
  void print () override
  {
      return thisClass()->name ();
  }
};
```

the **Implementor** class implements the **IPrinter** interface using the composition technique explained before and expects the **name()** function to be present in the user class.  
It can be used in this simple way:  

```cpp
class A : public A_Base, public Implementor
{
    std::string name () {return "A";}
}

int main ()
{
    A a;
    IPrinter * intf = &a;
    intf->print();
    use_interface(intf);
    return 0;
}
```

Some notes apply:  

* This kind of pattern is useful when you have a common implementation of an interface, that depends only in part on the combined class ( A in this case )

* Since A derives from Implementor, it's also true that A impelemnts IInterface; up-casting from A to IInterface is allowed.

* Even if Implementor doesn't have data members, A size is increased due to presence of the IPrinter VTable.

* Using interfaces allows to reduce code bloat in the class consumers, since every Implementor derived class can be used as (IPrinter \*)  
  There's a small performance penalty though, caused by virtual function calls and increased class size.

* The benefit is that virtual function calls will be used only when using the print function thought an IPrinter pointer. If called directly, static binding will be used instead. This can be true even for references, if the C++11 final modified is added to the Implementor definition.
  
  void print () final override;

```cpp
A a;
A & ref = a;
IPrinter * intf = &a;
a.print ();        // static binding
ref.print();       // dynamic binding (static if declared with final)
intf->print (); // dynamic binding;

This kind of composition doesnt have limits on the number of interfaces.  

class mega_composited : public Base , public Implementor<mega_composited>, public OtherImplementor<mega_composited>.....
{

};
```

## Adapters

These implementors can be seen as a sort of [adapters](http://en.wikipedia.org/wiki/Adapter_pattern) between your class and the implemented interfaces. This means that the adaptor can also be an external class. In this case you will need to pass the pointer to the original class in the constructor.  

```cpp
template <class T>
class Implementor : public IPrinter
{
    T* thisClass;

public:
  Implementor (T * original): thisClass(cls)
  {
  }

  void print () override
  {
      return thisClass->name ();
  }
};
```

note that thisClass is now a member and is initialized in the constructor.  

```cpp
 ...
 A a;
 Implementor impl(&a);
 use_interface(&impl);
```

as you see , the implementor is used as an adapter between A and IPrinter. In this way class A won't contain the additional member functions.  
**Note:** memory management has been totally omitted from this article. Be careful with these pointers in real code!  
Eventually one can make the object convertible to the interface but keeping it as object member (sort of COM aggregation).  

```cpp
class A
{
public:

   Implementor<A> printer_impl;
   /*explicit */ operator IPrinter * { return &printer_impl; }

   A () : printer_impl(this) {};
};
```

or even lazier...  

```cpp
class A
{
public:
     unique_ptr<Implementor<A>> printer_impl;

     /* explicit */ operator IPrinter * () {
       if (printer_impl.get() != nullptr)
          printer_impl.reset(new Implementor(this));
       return &printer_impl;
     }    

};
```

I will stop here for now. C++ let programmers compose things in many interesting ways, and obtain an high degree of code reuse without loosing performance.  
This kind of composition is near the original intent of templates, i.e. a generic piece of code that can be reused without having to use copy-and-paste! Nice :)