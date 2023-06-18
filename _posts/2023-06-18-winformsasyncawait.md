--- 
layout: post
title: "A Brief Overview on Backgrounding Tasks in a WinForms Application"
author: "Seth Hendrick"
comments: true
category: "Coding Tips"
description: "How to background tasks using async/await to prevent your UI from locking-up."
tags: [async, await, background, button, C#, csharp, dotnet, event, forms, GUI, hot, key, threading, UI, winforms, windows]
code: true
---

At my job, recently, we ran into a problem I didn't know how to solve right away.  We have a C# application written in [WinForms](https://en.wikipedia.org/wiki/Windows_Forms) where we wanted to have it so when a user either clicked a button on the UI or hit a hot-key, the application would do something.  This "something" would take time to perform.  We didn't want the application's UI to lock-up when performing this long-running task, so we wanted to move this task to a background thread.  We got the task running on a background thread with a button click easily enough, but we couldn't figure out how to background the task when the hot-key was pressed.

I finally found a potential solution, and wanted to share in case anyone else runs into a similar problem.

Some things to mention up-front:

* The examples below are purposefully simple.  A real application would need a lot more logic and error handling.
* While I do know enough about async/await and WinForms to write reliable applications using them, I would not call myself an expert with them.  This is a very tip-of-the-iceberg overview of these concepts, and some descriptions may not be 100% correct.

## The UI Thread

The UI thread is the thread where all the UI updates happen.  In fact, _all_ UI changes must happen on the UI thread, or your application may crash.  That is, you can't have a different thread other than the UI thread disable or enable buttons, change text, change colors, or do anything else to the UI.  If you want to have another thread change an UI element, you need to call the [BeginInvoke](https://learn.microsoft.com/en-us/dotnet/api/system.windows.forms.control.begininvoke?view=windowsdesktop-7.0) method on the UI object.

BeginInvoke is a method that takes in a delegate to run asynchronously on the thread that the control was created on.  To put it another way, think of the UI thread as an event queue.  Every single mouse click, mouse drag, key press, scroll wheel turn, etc. adds an event to this event queue that the UI thread then pulls from and runs the events on itself.  The BeginInvoke method simply says "Hey, I have this task I want do.  Please run it on the UI thread."

However, the fact that the UI thread behaves like an event queue opens the door to a problem that seems difficult to solve.  What happens if your application needs to perform a task that will take a long time to execute when you click a button on the UI?  Tasks such as writing to a file, doing a database query, or a web request can take some time.  If this work is done on the UI thread, it means the UI thread stops processing other events queued up while its waiting for this long-running task to finish.  This means that the UI can't be updated.  If a user presses a cancel button, tries to drag the UI around, or anything else, it won't get processed until this task finishes running.  This gives the appearance that your application locked up, which is not good for your application's reputation.

Ideally, you want to try to do as little work on the UI thread as possible to keep your UI as responsive as possible.  This means that any long-running tasks should be run on a background thread and notify the UI when its done running.  How does one go about doing this?

## Async/Await

Async/Await can be a complex and confusing concept in C#.  To be honest, I don't know all of the theory about how it _actually_ works under the hood, and I'm not going to pretend that I do.  But, I can at least give a high-level description about how it _appears_ to work with WinForms.

First, let's start off with an example of a simple WinForm GUI.  This GUI contains a single button that does some kind of task.  When the application is performing the task, the button will be disabled and will display text that says it is "Doing Work".

```C#
namespace WinFormsTest
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        private void button1_Click( object sender, EventArgs e )
        {
            ChangeGuiAndDoWork();
        }

        private void ChangeGuiAndDoWork()
        {
            try
            {
                this.button1.Enabled = false;
                this.button1.Text = "Doing Work...";
                DoWork();
            }
            finally
            {
                this.button1.Text = "Do Work";
                this.button1.Enabled = true;
            }
        }

        private static void DoWork()
        {
            // Emulate work being done by sleeping.
            Thread.Sleep( TimeSpan.FromSeconds( 5 ) );
        }
    }
}
```

If you ran the above code, what you would see is when you click the button, the UI will lock-up for 5 seconds.  You won't be able to drag it around or anything else until DoWork() finishes running.  The only way to prevent the UI from locking up is to have DoWork() run in a background thread.  While one _could_ have the button click event handler create a Thread and run DoWork inside of it, there's a much easier syntax that can be used instead with async/await.

```C#
        private async void button1_Click( object sender, EventArgs e )
        {
            await ChangeGuiAndDoWork();
        }

        private async Task ChangeGuiAndDoWork()
        {
            try
            {
                this.button1.Enabled = false;
                this.button1.Text = "Doing Work...";
                await Task.Run( () => DoWork() );
            }
            finally
            {
                this.button1.Text = "Do Work";
                this.button1.Enabled = true;
            }
        }

        private static void DoWork()
        {
            // Emulate work being done by sleeping.
            Thread.Sleep( TimeSpan.FromSeconds( 5 ) );
        }
```

Now when one clicks the button on the UI, they'll see the UI's button become disabled and the text changed to "Doing Work...".  The UI also no longer locks up, as the DoWork() method is being run on a thread that is not the UI thread.  But, what does the syntax in ChangeGuiAndDoWork mean?

* The ChangeGuiAndDoWork method is called on the UI thread via the button click event, so we are able to modify button1 inside of this method with no problem.
* The Task.Run more-or-less means "Please run the method passed inside of me on a different thread please."
* When we hit the "await" keyword, ChangeGuiAndDoWork actually returns once the Task begins running in the background, thus unblocking the UI thread.
* Though, before it returns, it does something to make it so when the method passed into Task.Run completes, any line in the method that comes _after_ the "await" keyword gets magically (that is, I don't fully understand how) enqueued back to the UI's event queue.
* Since we are back on the UI thread, we are able to enable the button and change the text inside of the finally block safely.

You can think of the following code as _similar_ behavior to the above.  Its _not_ identical behavior under-the-hood; there are things Task.Run and await do that I don't fully understand (for example, if an exception happens on the background thread, it is magically able to fall into the finally block on the UI thread).  But, it is good enough to show how one could do this behavior without async/await using raw threads.

```C#
namespace WinFormsTest
{
    public partial class Form1 : Form
    {
        private Thread? workerThread;

        public Form1()
        {
            InitializeComponent();
        }

        private void button1_Click( object sender, EventArgs e )
        {
            ChangeGuiAndDoWork();
        }

        private void ChangeGuiAndDoWork()
        {
            this.workerThread = new Thread(
                () =>
                {
                    // Performing the work in a background thread.
                    try
                    {
                        DoWork();
                    }
                    finally
                    {
                        // Work is completed, tell the UI thread
                        // to enable itself by adding an event to its event queue.
                        BeginInvoke(
                            () =>
                            {
                                // Can not change this in the background thread,
                                // must call BeginInvoke so it is done on the UI thread
                                // instead.
                                this.button1.Text = "Do Work";
                                this.button1.Enabled = true;
                            }
                        );

                        this.workerThread = null;
                    }
                }
            );

            // Still on the UI thread, safe to modify here.
            this.button1.Enabled = false;
            this.button1.Text = "Doing Work...";

            // Start the thread in the background, this method
            // returns right away to keep the UI thread running and un-blocked.
            this.workerThread.Start();
        }

        private static void DoWork()
        {
            // Emulate work being done by sleeping.
            Thread.Sleep( TimeSpan.FromSeconds( 5 ) );
        }
    }
}
```

## Overriding Protected Methods

One of the requirements of the application is if someone presses the F5 key, it needs to behave like someone pressed the button on the UI.  To do this, one can override the ProcessCmdKey like so:

```C#
        protected override bool ProcessCmdKey( ref Message msg, Keys keyData )
        {
            if( keyData == Keys.F5 )
            {
                ChangeGuiAndDoWork();

                // Return true to signal that the button event
                // was processed by this control, and to stop processing it.
                return true;
            }

            // Return false to signal that we did not handle the button
            // event.
            return false;
        }
```

Being able to background tasks with async/await for the button click event was as easy as marking the button click event handler as "async" and tossing in an "await" in the method body.  But, if one tries to mark the overridden ProcessCmdKey as async, you'll get a compile time error.  This is because there is no async version of this method to override.  This was _almost_ a road blocker for us at my job, as we didn't know how best to handle this situation.  We needed to background DoWork, but we couldn't call await in ProcessCmdKey.  We searched StackOverflow, and even asked ChatGPT out of desperation, but they all said the same thing of "sorry, can't make an overridden method async if the base method isn't as well".  Which means we started to look towards using raw Threads.

The solution came to me when I was lying in bed trying to sleep.  If we can't make ProcessCmdKey async, why don't we just have its job be as simple as adding an asynchronous action to the UI's event queue for us via BeginInvoke and then returning?  Well, that's exactly what we tried, and it seemed to work!

```C#
        protected override bool ProcessCmdKey( ref Message msg, Keys keyData )
        {
            if( keyData == Keys.F5 )
            {
                this.BeginInvoke( async () => await ChangeGuiAndDoWork() );
                return true;
            }

            return false;
        }
```

With the final code being this:

```C#
namespace WinFormsTest
{
    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();
        }

        protected override bool ProcessCmdKey( ref Message msg, Keys keyData )
        {
            if( keyData == Keys.F5 )
            {
                this.BeginInvoke( async () => await ChangeGuiAndDoWork() );
                return true;
            }

            return false;
        }

        private async void button1_Click( object sender, EventArgs e )
        {
            await ChangeGuiAndDoWork();
        }

        private async Task ChangeGuiAndDoWork()
        {
            try
            {
                this.button1.Enabled = false;
                this.button1.Text = "Doing Work...";
                await Task.Run( () => DoWork() );
            }
            finally
            {
                this.button1.Text = "Do Work";
                this.button1.Enabled = true;
            }
        }

        private static void DoWork()
        {
            // Emulate work being done by sleeping.
            Thread.Sleep( TimeSpan.FromSeconds( 5 ) );
        }
    }
}
```

Now, the UI no longer locks up when the button is clicked, or when the hot-key is pressed!

This example could use some more improvements.  For example, one is able to hit the hot-key multiple times to spawn multiple calls to DoWork().  But, its good enough to get the idea across.

## Conclusion

I'm sure some WinForms veterans may have already thought of this solution (or even have a better one).  But, since I couldn't find one when searching the internet, I wanted to write this down somewhere in case anyone out there ever runs into the same problem we did.

Thanks for reading!  If you have any ideas for improvements, please drop them in the comments below.
