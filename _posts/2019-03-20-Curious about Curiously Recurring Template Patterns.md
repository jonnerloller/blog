---
layout: post
title: Curious about Curiously Recurring Template Pattern
published: true
tags: ["CRTP","C++","templates"]
---
So today a junior in my company asked me what would I use Curiously Recurring Template Patterns (CRTP) for. I was a bit ashamed when I couldn't really give him a good reason why I would use CRTP. I have used CRTP before in the past, but there weren't clear use cases where I felt that It was a clear win for CRTP. Usually there would be a way to design around it, and it could be argued either way.

So I decided to go read up, research, think, about what I would use CRTP for.

The first could think of was static polymorphism.

What is static polymorphism?
We'll take the following code for example.

{% highlight cpp %}
struct Base
{
    void non_virtual_interface()
    {
        static_cast<T*>(this)->implementation();
    }
    virtual void implementation() = 0;
    virtual int get_type_hash() = 0;
};

struct Derived : Base
{
    void implementation() override;
    virtual int get_type_hash() = override;
};
{% endhighlight %}

Here I want to call 2 different functions.
1. ```get_type_hash``` - which returns the hash code of the class that we are. In theory, this function could be a static function, but because we want to be able to
get the hash depending on what derived class we have, we need to override it using a virtual function.
2. ```non_virtual_interface``` - which calls a ```virtual implementation``` function behind the scenes. derived classes need to override this.

The cost of polymorphism isn't cheap. A few things happen under the hood.
1. The size of your object increases due to the need to store a virtual pointer.
2. There is a hidden cost to the calling a function, since we need to check the virtual pointer against the virtaul table.

Let's say we want to have the same behavior, but without paying the additional cost.

We could do something like this.
{% highlight cpp %}
template <typename T>
struct Base
{
    void non_virtual_interface()
    {
        static_cast<T*>(this)->implementation();
    }

    // This was my most common use.
    static int get_type_hash()
    {
        return T::get_type_hash();
    }
};

struct Derived : Base<Derived>
{
    void implementation();
    static int get_type_hash();
};
{% endhighlight %}

In this way, we can create a Derived of Base, that would still be able to share certain members from Base, call Base functions, but not pay for the cost of virtual pointers, or virtual lookup. This is better known as static polymorphism.

Now while static polymorphism may seem like a great idea, there are a few issues with it.
1. Because of the templated base class, we now lose the ability to take advantage of what polymorphism is best used for, being able to have a homogenous container of an entire family of types. Here, because the base class requires the child class info at compile time, we can no longer have 2-3 different derived classes in the same container.

2. Honestly, compilers have come a long way. In almost anywhere that you can use static polymorphism, using virtual functions and using the ```final``` keyword usually has the around the same results. On top of that, you don't have to deal with the class in any special way. Just use it the way you would normally. When the compiler can detect what was the intended type, it can remove the virtualization.

#So what other possible ways can we use CRTP then?
Well lets think about it for a second.

{% highlight cpp %}
template <typename T>
struct Base
{
    // What can we do with "T" here?
};

struct Derived : Base<Derived>
{

};
{% endhighlight %}

Essentially, all that happens is that we give the Base class access to a type "T". That this means is that we can use "T" to do something useful.

## Return Type
We can use T for the return type of something. 1 good example is with Singletons.
{% highlight cpp %}
template <typename T>
struct Singleton
{
    T& get_instance()
    {
        static T instance;
        return instance;
    }
};

struct Derived : Singleton<Derived>
{

};
{% endhighlight %}
In this way, we can call ```Singleton<MyClass>::get_instance()```.

We can further expand this use case if we feel like we would need the ability to chain calls.

{% highlight cpp %}
template <typename T>
struct Renderer
{
    // What can we do with "T" here?
    T& draw_object(std::string object_name);
};

struct Derived : Renderer<Derived>
{
    Derived& change_mode(int num);
    Derived& set_property(std::string property, std::string value);
};

void some_function()
{
    Derived test;
    test.draw_object("ball").change_mode(50).set_property("color","red");
}
{% endhighlight %}

## Static Settings

Another possible use case of CRTP is to set certain compile time settings.

{% highlight cpp %}
template <IsMySpecialClassPolicy T>
struct Base
{
    // What can we do with "T" here?

    std::array<T,T::Size> blah;
    void func()
    {
        if constexpr(T::mode == some_mode)
        {

        }
        // ... 
    }
};

struct Derived : Base<Derived>
{

};
{% endhighlight %}

Well that was mainly all I could think of.

All in all, I think that the best usage of CRTP that I have used for myself is the ability to generate the return type of a function. It really opens up the door to doing lots of cool stuff. 
But then, after thinking about it a little more, I'm just thinking that the coolest feature of template classes in the first place is the ability to change class signature.

On a side note, I got side tracked and started reading about Data, Context, Interactions.
So I'll probably talk about that in the near future.
