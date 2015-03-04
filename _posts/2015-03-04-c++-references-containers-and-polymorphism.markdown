---
layout: post
title: C++, References, Containers, and Polymorphism
categories:
- Programming
published: true
---
During my leave from school, I've been funneling my creativity and recuperative free time into making a autobiographical game. I've [tweeted about it](https://twitter.com/starla4444) quite a bit in the last month. Progress has been slow, due to the illness that I'm recovering from, but the progress made so far has been both fun and illuminating. I'm using C++ for this project, of course not for the first time, but still, using C++ always results in you learning something strange about the language that you probably didn't want to know.

One of the things I've been implementing for my game engine has been an entity-component system. In short, such a system consists of two fundamental objects: entities, and components. Entities are perfectly generic objects that have no properties besides being able to contain components. Components are specialized objects that provide specific functionality to their parent entities. Such a system allows one to decouple a lot of the inner workings of a game engine as well as provide a simple way to mix and match functionality.

So, to start implementing this system, I created an Entity class and a Component class. I wanted Entity to be able to contain Components, but I didn't want to have to go through the trouble of writing my own linked list implementation, so I used one of the STL containers, `list`, which provides the exact functionality I am expecting. Well, almost the exact functionality, but we'll get to that later. Component was implemented as a class with several `virtual` methods that could be overridden by subclasses to provide their specialized funtionalities.

Doesn't seem like too much of a complex system, right? Unfortunately, even after implementing two Component subclasses, it became clear that something wasn't working with my system. The overridden Component methods were never getting called--only the empty versions in the Component superclass were ever entered. Therein lies the tangled interaction between references, STL containers, and C++ polymorphism. This is what my original code looked like (simplified for brevity):

{% highlight c++ %}
#include <list>
#include <cstdio>

class Base {
  public:
    virtual void display() { printf("At the base of our issues.\n"); }
};

class Derived : public Base {
  public:
    virtual void display() { printf("You're so derivative.\n"); }
};

int main()
{
  Base b;
  Derived d;
  
  std::list<Base> l;
  l.push_back(b);
  l.push_back(d);
  
  for (std::list<Base>::iterator it = l.begin(); it != l.end(); it++)
  {
    it->display();
  }
  
  return 0;
}
{% endhighlight %}

This code *should* print "At the base of our issues." followed by "You're so derivative.", but instead it prints the first message twice. Why is this so? The confusion in this code stems mainly from C++ references. One of the main reasons references in C++ exist is to allow STL containers to appear seamless or "pointerless" in normal use, and therefore it is sometimes difficult to tell when references are being used because they appear syntactically similar to normal code.

{% highlight c++ %}
std::vector<int> numbers(4, 100);
for (int i=0; i<4; i++)
{
  numbers[i]++;
}
{% endhighlight %}

Despite appearing to behave exactly like an array, `std::vector<int>::operator[]` returns an `int&`. If it returned an `int`, you would not be able to reassign the value at that position in the array. If it returned an `int*`, you would have to dereference the return from `[]` before being able to use it or reassign it. In this way, references allow your code to look a bit cleaner, at the cost of some added confusion.

Because `(*it)->display()` is somewhat ugly, I wanted to make use of references in my code. Specifically, I intended `l` to be a list of references to `Base`, not just a list of `Base`. The confusion here is also perhaps attributable to the method signature of `std::list::push_back`, which takes a `const value_type&`. Due to the confusion around the references syntax, I assumed this meant that if I had a list of value types, it would actually operate on references to that value type. I was incorrect. It turns out that STL containers cannot contain references, because containers are implemented using references and C++ does not support references to references. Because of this, my above list `l` is actually a list of `Base` objects, not a list of addresses to `Base` objects in the guise of references.

Here's where the polymorphism complication comes in. When you assign a value type `Derived` object to a variable with type `Base`, it gets *cast* to a `Base` object. Any information identifying that object as actually being a `Derived` object is lost. Then, when you later call a virtual method on that object, there is no information telling the runtime to call the `Derived` overridden method rather than the `Base` method. Therein lies the crux: pointers or references are actually required for polymorphism to function properly. When you assign a pointer to a `Derived` object to a variable with type `Base*`, the address of the `Derived` object is stored at the requested pointer, just as expected. No information from the actual `Derived` object is modified at all. This works because `Derived*` is type-compatible with `Base*`, which is one of the fundamental reasons that polymorphism works at all. When you then call the virtual method on the pointer, the runtime knows that the object pointed to is actually of a derived subclass, and is able to enter the proper method.

Therefore, we have figured out two things. One, STL containers cannot contain reference types, only value types and pointers. Two, you must use either pointers or references for polymorphism to work properly. The solution to our problem is then simple, if a bit ugly: we must use a list of pointers to Base.

Here is how you fix the above code, at the cost of seven additional characters:

{% highlight c++ %}
#include <list>
#include <cstdio>

class Base {
  public:
    virtual void display() { printf("At the base of our issues.\n"); }
};

class Derived : public Base {
  public:
    virtual void display() { printf("You're so derivative.\n"); }
};

int main()
{
  Base b;
  Derived d;
  
  std::list<Base*> l;
  l.push_back(&b);
  l.push_back(&d);
  
  for (std::list<Base*>::iterator it = l.begin(); it != l.end(); it++)
  {
    (*it)->display();
  }
  
  return 0;
}
{% endhighlight %}

In writing up this post, I've done a bit of research into C++ and discovered that my understanding of it as "C with classes" is poor at best, and grossly flawed at worst. Despite being built on top of a very straight-to-the-point language, C++ has a lot of hidden intricacies that merit further study for one to be able to claim to be proficient in C++ programming. I believe my confusion regarding C++ can be best expressed in a sentence I found while reading [a report from the committe](http://isocpp.org/blog/2013/04/trip-report-iso-c-spring-2013-meeting) working on the C++14 standard:

> The reason `make_unique` has important impact is that now we can teach C++ developers to mostly never use explicit `new` again.

At this point, it is unclear whether or not I will continue writing my project in C++, or if I will switch to using C, which I am more familiar with. In either event, the journey to knowledge has been, and will continue to be, fun.
