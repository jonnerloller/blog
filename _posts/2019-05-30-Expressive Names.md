---
layout: post
title: Expressive Names
published: true
tags: ["Names","C++"]
---
So today at work, I encountered a problem that I think everyone who has coded for a while would have faced in their codebase one way or another.
I found a function. And it was called ``` unsigned GetID() const ```. Now, obviously, by looking at the class I was working on at that point of time,
It kind of made sense as to what ```ID``` was, based on the context of the class. But here is where it started to get interesting.

I stumble upon another class. It has a function called ```ObjectID GetID() const```. Now, although I had just recently talked about "Strong Types", there are still
plenty of places within the codebase which still rely on typedefs and usings. And that itself shouldn't have been too much of a problem, since these 2 classea are clearly 
different classes with different purposes right?

Well not exactly. What actually hapened was that in our engine, we have our own ```Handle<T>``` class. It's meant to help wrap around object similar to how one may use a smart pointer, but with more features powering it. And so, the function looked something like this.

{% highlight cpp %}
void SomeFunction()
{
    ...

    // myObject is a Handle
    auto id = myObject.GetID();

    // objectContainer is an array of handles.
    for(auto x : objectContainer)
    {
        if(id == x->GetID())
        {
            Do something
        }
        else 
        {
            Do something else
        }
    }

    ...
}
{% endhighlight %}

Now leaving aside how the code could have been rewritten, we can clearly see an issue here.
There are 2 instances of ``` GetID() ``` being called. 1 with the ```.``` operator, and the other with the ```->``` operator.
And because we don't have strong typing, there are no compiler errors being thrown.

So I investigate. What are these ```IDs ?``` Well, turns out, that for the ```Handle<T>``` class itself, ID just represented "The ID used by the Object allocator , i.e Was I the first created object, or Nth created object?" Whereas the ID  of the actual class itself, represented which object in a ring of N objects is it,

While in the process of refactoring away some of the code, since I was short on time, I asked myself: "How can I prevent myself from encountering more of these conflicts?"
I'm on a schedule, I dpn't have time to reactor the code yet, but I refuse to encounter this bug again when it comes to this variable at least.

So I renamed it. To something which made sense. I changed ```Handle<T>```'s ```GetID()``` to ```GetObjectAllocationID()``` and the other ```GetID()``` to ```GetRingID()```. It wasn't a great fix, just some duct tape that did the job, and the code is schedules to be refactored soon. Turns out, while renaming it, I found seveeral other places where this had occured, and it saved us from some potential crashes.

I guess the takeaway here is,
1. Try to use Strong Types so that your compiler can be the one helping you track and prevent these problems.
2. If you can't do that, at least Name your variables and functions more explicity and expressively.
- If I can know what the function/variable does without even looking at the class name (without embedding the entire class name), it will save us from lots of potential problems in the future.
