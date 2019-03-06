---
layout: post
title: Template Deduction Woes!
published: true
tags: ["short","templates","rant"]
---

This is just going to be short rant about template deduction.

Today I encountered a simple template problem that took me way longer than I should to solve.

Simply put, it was something like this

{% highlight cpp %}
#include <cstddef>
template<int N,typename T>
struct my_struct
{
    T blah [N];
};

template<typename T , 
         //some sfinae to disable if is of type my_struct.
        >
int func(T obj)
{
    return 0;
}

template <std::size_t N,typename ... Args>
int func(my_struct<N, Args ...> obj)
{
  return 1;
}

int my_caller()
{
    my_struct<10,float> test;
    return func(test);
}

{% endhighlight %}

The code was slightly more complicated than this, with a couple of sfinaes, but you'll get the point.
So basically, I have a Base template function that handles all cases, and I have a narrower template to handle specific cases.
For the longest of time, I didn't know why my code path never entered the narrow path, and always took the more generic option.
Well when I lay the code in front of me now, its obvious. 

There is a type mismatch. ```std::size_t``` is not equal to ```int```.

So i fixed the problem, and all was good in the world. Well yes, but at the same time no.

I started to play around with more narrow templates.

{% highlight cpp %}
#include <cstddef>
template<int N,typename T>
struct my_struct
{
    T blah [N];
};

template<typename T , 
         //some sfinae to disable if is of type my_struct.
        >
int func(T obj)
{
    return 0;
}

template <std::size_t N,typename ... Args>
int func(my_struct<N, Args ...> obj)
{
  return 1;
}

template <std::size_t N>
int func(MyStruct<N, float> obj)
{
  return 2;
}

template <typename T, std::size_t N>
int func (MyStruct<N,float> obj)
{
   return 3;
}

int my_caller()
{
    my_struct<static_cast<std::size_t>(10),float> test;
    return func(test);
}

{% endhighlight %}

Even after all this extra work, it still didn't work.

**Casting it to a ```std::size_t``` didn't work either.```

So what is going on behind the scenes. Simple.

Firstly, because I defined ```MyStruct``` as a template struct that takes in a ```int```, no matter what,
it can never be converted automatically to that of a ```MyStruct<std::size_t,...>```. during template resolution.

I can however, do 

{% highlight cpp %}
#include <cstddef>
template<int N,typename T>
struct my_struct
{
    T blah [N];
};

template<typename T , 
         //some sfinae to disable if is of type my_struct.
        >
int func(T obj)
{
    return 0;
}

template <std::size_t N,typename ... Args>
int func(my_struct<N, Args ...> obj)
{
  return 1;
}

template <std::size_t N>
int func(MyStruct<N, float> obj)
{
  return 2;
}

int my_caller()
{
    my_struct<10,float> test;
    return func<10>(test);
}

{% endhighlight %}


This allows me to call the code which I want.

But this time, I'm still "technically" getting an dangerously formed converted template, because it is converting my ``` std::size_t ``` template parameter to a ``` int ```.

Now initially when I began to write this post, I wrote it with the intention of being upset because the template did not auto convert
a ``` std::size_t ``` into a ``` int ``` similar to that of plain old functions, like

{% highlight cpp %}
#include <cstddef>
#include <iostream>
int func2 (int obj)
{
    return 4;
}

int func2 (std::size_t obj)
{
    return 5;
}

int main()
{
    std::cout << func2(5) << std::endl; // prints 4.
    std::cout << func2(static_cast<std::size_t>(5)) << std::endl; // prints 4.
    // 5 will only ever be printed if func2(int) does not exist.
}

{% endhighlight %}

But after thinking about it long enough, I realized why that would be dangerous, since template arguments can be used to do so much more than just the arguments of plain old functions, it made sense to me for it to work that way.

**What I am slightly upset about**,is that I never received a single warning from this. Clearly, my template was meant to take in a ```int N```, and yes, I made the mistake of supplying the wrong argument in my own function, but surely this should be a warning of some kind?

The only reason I figured out what was going on was when i removed my "catch all" template, that the compiler told me that my other function would never ever be called unless explicitly instantiated, but it still let me instantiate it.

Well, that's it. rant over. I'm tired so I probably didn't make much sense typing this, but hopefully someday I come back to fix all the bad grammar errors and mistakes I made in this post.

**tldr;**
Compiler probably needs better warning messages for template deduction failures due to impossible type placements.



