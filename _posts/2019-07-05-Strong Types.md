---
layout: post
title: Strong Types
published: true
tags: ["Types","C++"]
---
I'm pretty sure that everyone has encountered the following code[in some form or another], in a codebase that they have worked on before.

{% highlight cpp %}
Rect a = Rect{0,0,10,10};

unsigned index = vertices[0]; 
{% endhighlight %}

So looking at the above code, we can instantly ask a few questions about the code.
1. In Rect, what do the parameters take? 
- Do the 4 parameters represent the 4 corners?
- Or is it anchor_x, anchor_y, width, height?
- Or is it center_x, center_y, half_width, half_height?

2. If a vertex index is of an unsigned type [Assume that Vertex index represents a index to a node on a graph]
- What does it mean to increment a index?
- Should I be comparable to an unsigned value ?
- If i have a triangle Index [Assume that Triangle index represents a index to 3 connected nodes on a graph]

## Creator functions

For the reader of the code, at that point, without refering to the interface of the class, there is basically no way to know what the class does at that point. Sure, at the point of time when some guy wrote the code, he is able to know exactly what to write, mainly because he read the interface code, but there is an additional lookup cost when debugging this kind of code without knowing exactly what each argument is. And often, I find myself double/triple checking, or even when explaining the code in a code review, that's probably a sign that we should consider refactoring the code in some way.

But how can we do something that handles this?

For example.
{% highlight cpp %}
struct Rect
{
    Rect(double left, double right, double top, double bottom);
    Rect(double topleft_x, double topleft_y, double width, double height);
    Rect(double center_x, double center_y, double half_width, double half_height);
};
{% endhighlight %}
Clearly, in the above example, all the declarations are considered identical.
So if we had no idea on how to handle this, we could do something like writing Create functions

{% highlight cpp %}
struct Rect
{
public:
    Rect CreateWithAnchor(double left, double right, double top, double bottom);
    Rect CreateWithTopLeft(double topleft_x, double topleft_y, double width, double height);
    Rect CreateWithCenter(double center_x, double center_y, double half_width, double half_height);
private:
    Rect(double left, double right, double top, double bottom);
};
{% endhighlight %}

So here, internally, we have a representation, and we provide an interface for the users to use. They can no longer instantiate a Rect directly, but they choose based on a "Create" function. All is great right? Well, not really. While it does solve the primary problem, it also adds an additional overhead to the user. By default, the expected behavior in c++ is that we use the Constructor to ... well ... construct things. Why then do we need an additional "Create" function just to disambiguate between Constructors?

So we're back to square one. Lets see how we can attempt to solve this problem again.

## Argument Clarification
Perhaps, we can make a generic "StrongType" class.
{% highlight cpp %}

template<typename T>
class StrongType
{
public:
    explicit StrongType(const T& val): val_(val){}
    explicit StrongType(T&& val): val_(std::move(val)){}

    [[nodiscard]] T& get() {return val_;}
    [[nodiscard]] const T& get() const {return val_;} 

private:
    T val_;
};
{% endhighlight %}

So. Here we have a mechanism to now declare a strong type, which has explicit constructors, meaning that it isn't going to by default convert from a type T. And with that, we can change our original class to.

{% highlight cpp %}
using Width = StrongType<double>;
using Height = StrongType<double>;

using HalfWidth = StrongType<double>;
using HalfHeight = StrongType<double>;

using Left = StrongType<double>;
using Right = StrongType<double>;
using Top = StrongType<double>;
using Bottom = StrongType<double>;

using TopLeftX = StrongType<double>;
using TopLeftY = StrongType<double>;

using CenterX = StrongType<double>;
using CenterY = StrongType<double>;
struct Rect
{
    Rect(Left left, Right right, Top top, Bottom bottom);
    Rect(TopLeftX topleft_x, TopLeftY topleft_y, Width width, Height height);
    Rect(CenterX center_x, CenterY center_y, HalfWidth half_width, HalfHeight half_height);
};
{% endhighlight %}

## Prevent wrong arguments from being passed in.

Now as you can see, clearly, there are issues with trying to strong type things.
1. It can get very tedious, because now simple types need to be named, 
2. It does not solve problem #2 as stipulated above. Which is that now, StrongType<double>'s don't distinguish between each other.
- This means that I can accidentally input a Width argument as a parameter for a function which is expecting a Height parameter.

So to get around that, we can use a tag variable, similar to that of the classic C-style.

{% highlight cpp %}
template<typename T,typename Tag>
class StrongType
{
public:
    explicit StrongType(const T& val): val_(val){}
    explicit StrongType(T&& val): val_(std::move(val)){}

    [[nodiscard]] T& get() {return val_;}
    [[nodiscard]] const T& get() const {return val_;} 

private:
    T val_;
};
{% endhighlight %}

Note that the second template parameter "Tag" doesn't actually do anything. It is there solely to help disambiguate the 2 things.
Our class can now be rewritten as such:

{% highlight cpp %}

using Width = StrongType<double, struct WidthTag>;
using Height = StrongType<double, struct HeightTag>;

using HalfWidth = StrongType<double, struct HalfWidthTag>;
using HalfHeight = StrongType<double, struct HalfHeightTag>;

using Left = StrongType<double, struct LeftTag>;
using Right = StrongType<double, struct RightTag>;
using Top = StrongType<double, struct TopTag>;
using Bottom = StrongType<double, struct BottomTag>;

using TopLeftX = StrongType<double, struct TopLeftXTag>;
using TopLeftY = StrongType<double, struct TopLeftYTag>;

using CenterX = StrongType<double, struct CenterXTag>;
using CenterY = StrongType<double, struct CenterYTag>;
struct Rect
{
    Rect(Left left, Right right, Top top, Bottom bottom);
    Rect(TopLeftX topleft_x, TopLeftY topleft_y, Width width, Height height);
    Rect(CenterX center_x, CenterY center_y, HalfWidth half_width, HalfHeight half_height);
};
{% endhighlight %}

And now, well the class works the way we expect it to work, and constructors can be used again.
Well, there are still some issues though.
The first, is the issue we haven't previously addressed, which is that for something to simple, we managed to complicate the problem quite significantly. It is so much harder to write code if we are trying to "StrongType" everything.

That being said, the example I gave was definitely far too dumb to showcase the usefulness of Strong types.

The issues that can be easily resolved are
1. Better grouping of arguments.
- In this case, topleft_x,topleft_y, could have been a point2 or a vec2, that represent position, that may help to reduce the number of StrongTypes declared.
2. Generally speaking, the types that you declare can be reused in multiple places. For example the example I gave at the start ```using VertexIndex = StrongType<unsigned,struct VertexIndexTag>; using TriangleIndex = StrongType<unsigned,struct TriangleIndex>;```. These Strong types can be declared once, and then used throughout the program.
