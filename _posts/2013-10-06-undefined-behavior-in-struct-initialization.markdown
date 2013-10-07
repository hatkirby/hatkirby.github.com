---
layout: post
title: Undefined Behavior in struct Initialization
categories:
- Programming
published: true
---
Working on [rawr-ebooks](http://github.com/hatkirby/rawr-ebooks) recently has allowed me to tinker with the writhing mass of horror and love that is C++. You may not be able to tell from this that I am a massive fan of C++. The nitty gritty, say what you mean and EVERTHING you mean fifty times methodology of C++ is actually a great amount of fun for me, and what would be better than implementing a fun algorithm in a really fun, rigorious programming language?

One of the incredibly fun parts about C++ is that it's practically a well of undefined behavior. If you aren't doing things exactly right, there's really no telling what will happen. And how do we define "exactly right"? Well, we're not *always* completely sure about that. Here's an instance of that.

As you may know, there are two real ways to allocate memory for a struct in C++. One is to use `malloc` or `calloc`, pass it the `sizeof` your struct, and then cast the resulting pointer to a pointer of your struct. The other, fancier and C++-exclusive method is the use the `new` operator. While this is all fine and dandy, it turns out that these methods are not exactly equivalent. I'd like you to examine the following two blocks of code and tell me what they output, okay? Let's have some fun.

**Code Block A:** Using `calloc`, the C way
{% highlight c++ %}
#include <iostream>
#include <string>

using namespace::std;

typedef struct
{
    int collect;
    string t;
} ts;

int main(int argc, char** argv)
{
    ts* test = (ts*) calloc(1, sizeof(ts));    
    test->t = "howdy";
    test->collect++;
    
    cout << test->t << endl;
    cout << test->collect << endl;
	
    return 0;
}
{% endhighlight %}

**Code Block B:** Using `new`, the C++ way
{% highlight c++ %}
#include <iostream>
#include <string>

using namespace::std;

typedef struct
{
    int collect;
    string t;
} ts;

int main(int argc, char** argv)
{
    ts* test = new ts();
    test->t = "howdy";
    test->collect++;
    
    cout << test->t << endl;
    cout << test->collect << endl;
	
    return 0;
}
{% endhighlight %}

How do each of these code snippets behave? Let's examine code block A first. Now, `calloc` is very similar to `malloc` in that it allocates the amount of memory requested and returns a pointer to it, but it also initializes the block of memory so that every bit is zero. In the days of C when everything was either a primitive or a pointer (both of which were really just numbers), this worked perfectly because it would just set every field of the struct to 0. How does it act when mixed with C++ STL objects?

    Segmentation fault: 11
   
Well, it really doesn't behave at all. It just segfaults. Why? It turns out that "being" a bunch of zeros isn't really a valid state for a C++ STL string. Why does this matter, though, if we're reassigning the string anyway? In C++, setting an STL string equal to another string actually calls a method called `assign` on the original string, which replaces its contents with that of the new string. Why on earth would something this silly happen? Well, you have to remember that the field `t` of `ts` here is not a pointer. It is an instance of the class `string` and can be thought of in a way as a sort of primitive, except it is far more complex than a primitive. Essentially, though, a single `string` object is fixed into every instance of `ts`, and assigning to it just modifies the instance of `string`. Huh. Interesting.

So, if the C-way of allocating memory doesn't do too well with STL objects, then why not just use `new`? Let's examine code block B now. The `new` operator, when applied to a struct, allocates memory for the instance of the struct and initializes each field seperately. Sounds good, right?

    howdy
    -1719966546

That's certainly better than before, but I think we were all expecting the second line to be `1`, no? It turns out that "initializes each field" means "constructs each field", which works well for classes like `string`, but doesn't mean a whole lot for primitives like `int`. So, instead of setting each `int` in the struct to zero, `new` just leaves them be, which is the perfect formula for undefined behavior. Unlike the `calloc`/`string` conundrum, however, this one is a lot easier to fix. All you have to do is initialize your primitive fields on your own after calling `new`:

{% highlight c++ %}
ts* test = new ts();
ts->collect = 0;
ts->collect++;
{% endhighlight %}

This works as expected. Horray!

The reason I find this so interesting is that working on `rawr-ebooks`, I found myself in some mutated state between these two methods of allocating a struct. I cannot reproduce it anymore, but the behavior I had was so undefined it was unbelievable. There were three main outputs that I recieved, though:

1. Segfault immediately.
2. Raise an abort trap because something tried to malloc 980 terabytes of memory.
3. Summon Satan.

Don't believe me? Here's a screenshot of my terminal while debugging this error: (the madness at the top was only the tip of the iceberg, it got quite wild but didn't fit into the picture):

<a href="/assets/images/2013-10-06-undefined-behavior-in-struct-initialization/terminal.png"><img src="/assets/images/2013-10-06-undefined-behavior-in-struct-initialization/terminal.png"></a>

The nonsense at the top even says something about "religious themes." Don't tell me that doesn't scare you.

The lesson here is that C++ is a lot of fun even if you don't really know what you're doing half the time. Really, figuring out what's wrong (and crashing G++ a few times, yes, that happened, I really don't know how I managed it) is the fun part.
