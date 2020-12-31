--- 
layout: post
title: "My Nightmare of Trying to Switch Users on a Windows Jenkins Agent"
author: "Seth Hendrick"
comments: true
category: "DevOps Rants"
description: "How to switch users on a Windows Jenkins Agent."
tags: [Jenkins, DevOps, Windows, Agent, Worker, CI, Continuous Integration, CT, Continuous Test, Impersonation, runas, sudo, su, ssh]
code: true
---

In 2019, the buzzword at my job was "DevOps".  Every department was trying to introduce [Continuous Integration](https://en.wikipedia.org/wiki/Continuous_integration) (CI) and [Continuous Test](https://en.wikipedia.org/wiki/Continuous_testing) (CT) into their software processes.  Since I work in automated test development, I helped in bringing our continuous testing infrastructure online.  Honestly, although "DevOps" seemed like it was a shallow buzzword-of-the-year, I was excited!  I've seen what CI and CT can do if done correctly at a previous job.  At my first co-op, they were able to do a full regression test _nightly_ while doing it manually would take weeks.

After probably a total of roughly 2 man-months of building up our DevOps infrastructure, we were able to bring online an automated smoke test.  Since our smoke test was now automated, this saved about 8-10 man hours per week!

But this isn't a "DevOps success story" post.  Oh no.  When we first started bringing our DevOps infrastructure online, we used [Bamboo](https://www.atlassian.com/software/bamboo).  However, the higher-ups then demanded we switched to [Jenkins](https://www.jenkins.io/) instead.  Fortunately, our testing infrastructure was built to be CI platform-agnostic, so the switch wasn't too painful...

...Except for one thing...

Switching users in the middle of a Jenkins job automatically is **_painful_** on a Windows Jenkins agent.

## Why switch users?

The main reason is we need to run the automated tests as a specific shared account.  There are a few reasons for this.  First, it needs to be a shared account since no one wants to have Jenkins login as their own user accounts, for obvious reasons.  Second, this shared account has access to the builds that need to be tested.  And third, for better or for worse, our automated test software needs to have a one-time manual setup to work on an agent.  This means that once and (hopefully) only once, someone needs to login as the shared account on the Windows PC that will run the automated tests and configure the automated test software.  This is stuff like which serial ports to use, what files to test, etc.  At some point, we would like to be able to _not_ do this, and have our automated smoke test configure this stuff automatically on-the-fly, but we're not quite there yet.  Maybe someday if my group gets more staff :P.

How this worked on Bamboo agents, is Bamboo installs a Windows service.  We configured the Windows service to login as this shared account.  This was a poor decision for a couple of reasons.  For one, this meant that whenever the password changed, we would have to go in to each Bamboo agent, stop the service, update the password, and start the service.  If it were just one agent, that's one thing.  However, we have multiple, so it got really old really quickly.  Oh, and if we forgot to update the password on one, the agent would keep trying to login with the wrong password, and the account would get locked out.  Not fun.  The other reason why this was a poor decision was because it violated the [Principle of Least Security](https://en.wikipedia.org/wiki/Principle_of_least_privilege).  Because the Bamboo Windows service was logged in as this shared account, _any_ job that ran on this agent had access to anything the shared account had access to.  So, someone could theoretically create or modify a Bamboo job, run on this agent, archive things they didn't have access to but the shared account did, download them, and delete or revert the Bamboo job.  Unlikely to happen given other security measures in place, but it _could_ have if someone knew what they were doing and had time to do it.

Jenkins takes a different approach when it comes to configuring agents.  While Bamboo agents talk to the main node to form the connection, Jenkins does the opposite.  The main Jenkins node connects to agents by [SSH](https://en.wikipedia.org/wiki/SSH_(Secure_Shell))'ing into them.  This was great!  There was no configuration needed on the agents other than installing SSH Server.  This also meant we no longer have to login to the agents and start/stop Windows services when a password changed, as the password is stored on the main node.  Thus, we only have to update passwords in one spot, through the Jenkins UI.  One annoyance with Bamboo solved!

However, the user Jenkins logged in as via SSH was not our shared account, rather it was an account that was local to the PC, and had _no_ permissions at all.  Permissions had to be passed in from the Jenkins job with credentials configured on the main node.  This also solved another problem with Bamboo, as now we follow the Principle of Least Security with the accounts Jenkins runs jobs under.

## The Problem

However, this no-permission account opened up another whole can of worms.  See, we still had to run automated tests using the shared account.  The no-permission account that Jenkins ran under didn't have access to the builds we had to test.  We also couldn't login to them to do the one-time manual setup of our automated test software, as only IT had the passwords to these no-permission accounts.

But, we figured this wouldn't be too big of a deal.  We should just be able to switch users at the start of an automated smoke test.  I mean, this problem has been solved on Linux for years with the ```sudo -u -S``` and ```su -c``` commands.  The password could then be piped into STDIN.

Oh... how wrong I was!

## Failed Solutions

Windows did not make this easy.  I pulled my hair out for days trying to figure out how to switch users in a Jenkins environment.  Here are all of the things I tried that did not work.

### Runas

Windows does have something like Unix's ```sudo -u``` built in, and its called "runas".  Runas allows one to run a process as a different user.  One specifies the user they want to run the process as, followed by the process to run as the other user.  Runas would then prompt for a password through STDIN.  For example:

```
PS C:> runas /profile /user:jenkinsuser cmd.exe
Enter the password for jenkinsuser:
```

When doing this manually, it worked great!  We put in the path to our automated test software, and it loaded the shared user's profile.  However, when we tried to input the password automatically via STDIN, runas failed.  We double checked everything to make sure the password was correct, but, alas, it was still failing.

Turns out, this was a design decision by the authors of runas; the program demands you type the password manually.  According to Raymond Chen in [this](https://devblogs.microsoft.com/oldnewthing/20041129-00/?p=37183) blog post, there was a fear that people would start embedding passwords in batch files if runas allowed for this to happen, which is obviously insecure if one does it that way.  I _think_ I remember reading somewhere that if one pushes the enter button too quickly (such as a pipe via STDIN would), runas would fail, and if one were to add a couple second sleep before sending the password, it _could_ work.  However, I wasn't able to get this to work, so I'm not sure how true that is.

### Psexec

Another option was to use a tool called [Psexec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec).  This allows one to specify the password on the command line.  However, there is a security concern with using this tool.  While Psexec allows one to specify a password with the ```-p``` argument, well, it is on the command line.  This means any user can login to the agent and open [Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer) and see the command line arguments for any process on the machine.  This means a leaked password, which is bad.

We also opted not to use this tool because it was yet another thing we had to install on our CT agents.

### C# Process Object

In Raymond Chen's article, he suggested if one wanted to pass in a password via the command line, they would have to write their own tool, and use Window's [CreateProcessWithLogonW](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createprocesswithlogonw) function.  Turns out, C#'s [Process](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.process?view=net-5.0) object just does that (you can see the call in dotnet's source code [here](https://github.com/dotnet/runtime/blob/ef2a1878793e7e3fc3060396d3d2655ac53b1316/src/libraries/System.Diagnostics.Process/src/System/Diagnostics/Process.Windows.cs#L567))!

So, we did just that, but tweaked it slightly.  Instead of specifying the password directly through the command line, the command line would instead specify the name of the environment variable that would contain the password.  The process would then get the password from the environment variable and pass it into the C# Process object.  Since the raw password wasn't specified on the command line directly, if someone opened Process Explorer, they would only see the environment variable name, not the password itself unless they were an admin and looked at the process's environment variables.

The code looked somewhat like this, but more way more robust:

```C#
using System;
using System.Diagnostics;

namespace RunAsTest
{
    class Program
    {
        static int Main( string[] args )
        {
            ProcessStartInfo info = new ProcessStartInfo
            {
                CreateNoWindow = true,
                FileName = args[2],
                UserName = Environment.GetEnvironmentVariable( args[0] ),
                PasswordInClearText = Environment.GetEnvironmentVariable( args[1] ),
                LoadUserProfile = true,
                Domain = "thedomain"
            };

            using( Process process = new Process() )
            {
                process.StartInfo = info;
                process.Start();
                process.WaitForExit();

                return process.ExitCode;
            }
        }
    }
}
```

It would be executed in Windows BATCH by:
```batch
set USERNAME_ENV = "someuser";
set USERNAME_PASS = "somepass";
MyRunAs.exe USERNAME_ENV USERNAME_PASS c:\OurTestSoftware.exe
```

or in Jenkins:
```java
withCredentials(
    [usernamePassword(
        credentialsId: "CREDS_ID",
        passwordVariable: 'USERNAME_PASS',
        usernameVariable: 'USERNAME_ENV'
    )]
)
{
    bat "MyRunAs.exe USERNAME_ENV USERNAME_PASS c:\\OurTestSoftware.exe"
}
```

On my standard work PC, writing our own runas worked great!  We were able to launch our test software as our shared user account, and the password wasn't exposed via Process Explorer!

But, my soul was crushed just a few minutes later, when I tried it in a Jenkins Environment.  It didn't work.  I could not, for the life of me figure out why.  I then remembered that Jenkins logs into agents via SSH, so I said to myself, "I wonder if SSH is the problem?"  So, on my standard work PC, I SSH'ed into itself by connecting to "localhost".  I then changed into the directory where I ran our version of runas, and ran the exact same command that worked outside of an SSH environment.

It failed for the same reason it was failing on Jenkins.

As mentioned earlier, the C# process object calls CreateProcessWithLogonW.  Turns out, according to the [documentation](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createprocesswithlogonw#remarks), we can't call this function with the "SYSTEM" account.  We need to, instead, use the [CreateProcessAsUser](https://docs.microsoft.com/en-us/windows/desktop/api/processthreadsapi/nf-processthreadsapi-createprocessasusera) and [LogonUser](https://docs.microsoft.com/en-us/windows/desktop/api/winbase/nf-winbase-logonusera) function, which do not exist in C#.  So we would have to [P/Invoke](https://docs.microsoft.com/en-us/dotnet/standard/native-interop/pinvoke) the Windows API from C#.

The exact remarks from the documentation are:

>Windows XP with SP2,Windows Server 2003, or later:  You cannot call CreateProcessWithLogonW from a process that is running under the "LocalSystem" account, because the function uses the logon SID in the caller token, and the token for the "LocalSystem" account does not contain this SID. As an alternative, use the CreateProcessAsUser and LogonUser functions.

The SSH server service runs as the System account.  So any sub-processes will not be able to login as a different user using CreateProcessWithLogonW.

Ugh.  Back to the drawing board.

So before I went down the P/Invoke rabbit hole, I continued searching for something else that could work.

### Impersonation

I then found Impersonation in Windows.  According to Microsoft's [documentation](https://docs.microsoft.com/en-us/windows/win32/com/impersonation):

>Impersonation is the ability of a thread to execute in a security context that is different from the context of the process that owns the thread. When running in the client's security context, the server "is" the client, to some degree. The server thread uses an access token representing the client's credentials to obtain access to the objects to which the client has access.

Notice the phrase "a thread".  That's important.  But more on that later.

We actually had some success with impersonation.  We were actually able to login as the shared account and access files it had permission to read.  In fact, after we compile our builds, we impersonate to deploy them to file shares, where our testers can then download them.  Compiling and deploying happen within the same thread, so this works.  However, what happens if while we are impersonating a different user, we launched a sub-process?  Could this be the solution I have searched for?

Turns out, no.  A sub-process is an entirely different thread.  So impersonating fails.  Here is (a very hacky) sample program I wrote to demonstrate this.

```C#
// This must be run in a directory the impersonated user also has access to,
// or the sub-process will not launch since it can't access the .exe.

using System;
using System.Diagnostics;
using System.IO;
using System.Reflection;
using System.Runtime.InteropServices;

// Need the System.Security.Principal and System.Security.Principal.Windows
// NuGet packages installed for this to work, if dotnet core.
using System.Security.Principal;
using Microsoft.Win32.SafeHandles;

namespace RunAsTest
{
    class Program
    {
        [DllImport( "advapi32.dll", SetLastError = true, CharSet = CharSet.Unicode )]
        public static extern bool LogonUser(
            String lpszUsername,
            String lpszDomain,
            String lpszPassword,
            int dwLogonType,
            int dwLogonProvider,
            out SafeAccessTokenHandle phToken
        );

        const int LOGON32_PROVIDER_DEFAULT = 0;
        const int LOGON32_LOGON_INTERACTIVE = 2; // This parameter causes LogonUser to create a primary token.

        const string user = "jenkinsuser";
        const string password = "thepassword"; // Obviously, not the real password.
        const string domain = null;

        static void Main( string[] args )
        {
            string fileLocation = Assembly.GetExecutingAssembly().Location; // Returns .dll, not .exe.
            fileLocation = Path.GetFileNameWithoutExtension( fileLocation ) + ".exe";

            if( args.Length == 0 )
            {
                Console.WriteLine( "Starting User Info:" );
            }
            else
            {
                Console.WriteLine( "Sub-process while impersonated Info:" );
            }
            PrintEnv();

            // If this is the parent process, launch a sub-process.
            if( args.Length == 0 )
            {
                Console.WriteLine();

                SafeAccessTokenHandle handle;
                bool loggedIn = LogonUser(
                    user,
                    domain,
                    password,
                    LOGON32_LOGON_INTERACTIVE,
                    LOGON32_PROVIDER_DEFAULT,
                    out handle
                );

                if( loggedIn == false )
                {
                    handle?.Dispose();
                    Console.Error.WriteLine( "Did not login" );
                    return;
                }

                WindowsIdentity.RunImpersonated(
                    handle,
                    () =>
                    {
                        Console.WriteLine( "Impersonation Info:" );
                        PrintEnv();

                        try
                        {
                            ProcessStartInfo info = new ProcessStartInfo
                            {
                                CreateNoWindow = true,
                                FileName = fileLocation,
                                LoadUserProfile = true,

                                // Specify an argument so we don't fork bomb.
                                Arguments = "stop",
                                WindowStyle = ProcessWindowStyle.Hidden,
                                RedirectStandardOutput = true
                                // Username / Password purposefully not specified, so we don't
                                // call CreateProcessWithLogonW that specifies a username/password,
                                // which we know doesn't work.
                            };

                            using( Process process = new Process() )
                            {
                                process.StartInfo = info;
                                process.Start();

                                string stdout = process.StandardOutput.ReadToEnd();
                                Console.WriteLine( stdout );

                                process.WaitForExit();
                            }
                        }
                        catch( Exception e )
                        {
                            Console.Error.WriteLine( "Error when starting process: " );
                            Console.Error.WriteLine( e.Message );
                        }
                    }
                );

                handle?.Dispose();
            }  
        }

        private static void PrintEnv()
        {
            Console.WriteLine( "\tUser Name: " + Environment.UserName );
            Console.WriteLine( "\tMachine Name: " + Environment.MachineName );
            Console.WriteLine( "\tCurrent Directory: " + Environment.CurrentDirectory );
            Console.WriteLine( "\tMy Documents: " + Environment.GetFolderPath( Environment.SpecialFolder.MyDocuments ) );
            Console.WriteLine( "\tExe location: " + Assembly.GetExecutingAssembly().Location );
        }
    }
}
```

The resulting output is this:

```
Starting User Info:
        User Name: seth
        Machine Name: hostname
        Current Directory: C:\Jenkins\bin\Debug\netcoreapp3.1
        My Documents: C:\Users\seth\Documents
        Exe location: C:\Jenkins\bin\Debug\netcoreapp3.1\RunAsTest.dll

Impersonation Info:
        User Name: jenkinsuser
        Machine Name: hostname
        Current Directory: C:\Jenkins\bin\Debug\netcoreapp3.1
        My Documents: C:\Users\jenkinsuser\Documents
        Exe location: C:\Jenkins\bin\Debug\netcoreapp3.1\RunAsTest.dll

Sub-process while impersonated Info:
        User Name:
        Machine Name: hostname
        Current Directory: C:\Jenkins\bin\Debug\netcoreapp3.1
        My Documents:
        Exe location: C:\Jenkins\bin\Debug\netcoreapp3.1\RunAsTest.dll
```

So, we start the process as me.  The process then impersonates jenkinsuser, and it prints the information one would expect while logged in as this user.  However, the weirdness comes when when we try to launch a subprocess while impersonated.  Notice, the user name and the documents folder are empty strings.  I have no explanation as to why this is the case, but either way, it meant launching our test software while impersonated was NOT the right course of action.  If we wanted to run a sub-process as a different user, we would always have to specify the username and password in the [ProcessStartInfo](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo?view=net-5.0).  However, it was a worthwhile experiment since we were still able to use impersonation elsewhere.

## The Actual Solution

I don't remember how I came up with this idea.  But it was so simple, I can't believe I didn't think of it sooner.  There's an SSH server running on our CT agents.  Why don't we just SSH from the no-permission account into the same PC via localhost and login to the shared account?  We could then run the command to launch our test software?

This is the approach we use.  It does work.  However, we can not use the OpenSSH's "ssh" command itself, since there is no way to specify a password on the command line through that.  Piping the password via STDIN to SSH does not work since it does some weird terminal thing that I don't fully understand.  [SshPass](https://linux.die.net/man/1/sshpass) is a tool that can be used on Linux to pass a password in, but it does not work on Windows.

So, I created a front-end to the C# library [SSH.NET](https://github.com/sshnet/SSH.NET) that allows one to pass in environment variable names that contain the username and password via the command line, the server to connect to, and the command to run.  I called this tool SshRunAs, and it is on my GitHub [here](https://github.com/xforever1313/SshRunAs)

To use in BATCH, this is what to do:

```batch
set USER=me
set PASSWD=SuperSecretPassword
.\SshRunAs.exe -c "curl https://shendrick.net" -u USER -p PASSWD -s localhost
```

In Jenkins, meanwhile, withCredentials can be used:

```java
withCredentials(
    [usernamePassword(
        credentialsId: "CREDS_ID",
        passwordVariable: 'USER',
        usernameVariable: 'PASSWD'
    )]
)
{
    bat "SshRunAs.exe -c \"curl https://shendrick.net\" -u USER -p PASSWD -s localhost"
}
```

And... that's how we change users in a Jenkins Environment running on a Windows machine.  Its a bit hacky, but it works surprisingly well.  

## Conclusion

In an ideal scenario, we would not need to switch users to run our automated smoke tests.  We should just be able to use the no-permission Jenkins account.  Alas, we are not there quite yet, and may not be for quite some time.  However, SshRunAs is working well for us for the time being, and, honestly, maybe its good enough.  My group is still young in terms of experience in the realm of DevOps, and I'm sure those with more experience reading this are screaming "DO IT THIS WAY!".  I will say, if anyone has any suggestions or comments, please leave one below!  

In the end, I did learn more about how Windows and C# work under-the-hood.  I also learned more about how Jenkins works as well.  Although it was a frustrating few days at work, it was all worth it in the end.
