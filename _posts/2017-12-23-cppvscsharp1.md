--- 
layout: post
title: "C# vs. C++ - Part 1: Introduction"
author: "Seth Hendrick"
comments: true
category: "Coding Rants"
description: "An introduction to why C# replaced C++ as my favorite language."
tags: [C#, csharp, C++, introduction, vs, RIT, HVCC, co-op, job, Harris Corporation]
code: true
---

I am still a novice in the world of Software Engineering.  I am currently, as of writing, 26 years old.  My first exposure to the world of software engineering was only 8 years ago, during my first year at Hudson Valley Community College.  The class was ENGR-110: "Engineering Tools".  In that class we learned about Excel, CAD, and an introduction to programming.  Despite the fact that I was considered the "computer nerd" in high school, I didn't know any programming.  Yeahhh, I knew HTML, but HTML is not a programming language.  This was the first time I ever did anything with if statements, while loops, and arrays.

I'll never forget the first time I hit the "compile" button followed by the "run" button and saw "Hello, World!" appear just over 8 years ago.  We were working in a little-known IDE we downloaded from [Bloodshet.net](http://bloodshed.net/).  The language we were working in?  C++.

That was my first exposure to C++.  It wouldn't be until after I graduated HVCC and started my second quarter at RIT would I learn about it again.  That class was Computer Science 4.  A year later, I got my first co-op, which lasted for 9 months.  The language the company used? C++.  When I returned to RIT, I minored in Game Design and Development.  Their introduction classes to OpenGL was all C++.  I wrote my [senior design](https://ctsn.shendrick.net) project in C++.  The startup I was briefly involved in had all of our code written in C++.

All during this timed, I learned about the various compilers, which ones were great, which ones were... less-than-stellar.  I learned which IDEs worked decently and which were just okay.  I learned various build environments such as make, and SCons.  I learned how to compile libraries myself on both Windows and Linux.

C++ was my favorite language!  It had its problems, but it was powerful and the syntax made sense to me (more on that later).  It got to the point where I would write everything in C++, because it was that flexible of a language!

Time passed, and I was ready for my last co-op while at RIT.  I accepted an offer with Harris Corporation.  I was going to be working on C/C++ code for some of the tactical radios there.  My first day there, the HR people went around the room and told us who our managers would be.  When they got to me, they revealed that my manager would not be who I interviewed with.  My friend who  co-oped there prior told me that the department did acceptance testing.

Uh-oh.

I was... upset...  ACCEPTANCE TESTING?  That's not what I signed up for!  But maybe I would do something cool?

My supervisor came by to pick be up.  I'll never forget the hallway chat we had.

* Him: "So what experience do you have?"
* Me: "Well, I do a lot of C++ programming.  I also have some embedded experience."
* Him: "We don't do embedded development in our department.  We mostly work in C#".

I was disheartened.  I not only was not going to work on an embedded system, but I was also going to work in what I thought was an inferior language at the time.  "It is only 3 months," I thought to myself.

But, it being a co-op, I took the opportunity to try to learn something new.  I programmed in a similar language (Java) before, and C# once.  Maybe it would be cool?

My biased was showing early.  My time there, there were multiple times where I had a thought such as: "C++ is clearly a better choice than C#. C# doesn't even have destructors (they're finalizers! those don't count!), or const references (we have to use interfaces), and templates (generics in C#) need to implement a common interface!?  And what's the deal with .Equals and operator==?"

However, I did acknowledge the C# had some nice features about it.  Properties are a godsend.  Syntax is considerably nicer than C++.  C# actually had filesystem, sockets, threading, and XML stuff build into its "standard library".  C++11 was *just* going mainstream then, so having threading built into the language without using a third-party library (e.g. Boost) was wonderful.

Well, over 3.5 years after that dreaded walk down the hallway, my supervisor from my co-op is now my current, full-time, supervisor.  I work full-time in the department I co-oped in, and I look forward to going into work everyday.  I program in C# almost every single day, and I rarely touch C++ anymore.

Why?  What caused me to change my favorite programming language?

In this series of posts, I'll be describing what I like about C++ and C#, what I dislike about both, and why I switched.  I programmed in both of them of roughly the same amount of time (3 years each).  I worked with both in a professional environment (though my time with C++ was a co-op, it was still an actual company with a complex codebase).  And hopefully, we all learn something from these rants!