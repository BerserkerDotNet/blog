---
layout: post
title: How to rethrow exception correctly in .Net
date: '2018-08-19T09:00:00.000-07:00'
author: Andrii Snihyr
tags:
- .Net
- C#
- exception
- exceptions
- rethrow
- throw
- catch
- stacktrace
- ExceptionDispatchInfo
modified_time: '2018-08-19T09:00:00.000-07:00'
---
A few days ago I stumbled across a C# code that was rethrowing the exception by means of passing captured exception object as an argument of the throw keyword (`throw ex;`).
Such a pattern of rethrowing the exception can turn an exercise of troubleshooting a production issue into a game of "Find where the exception happened.". I had to play this game a few times in my life and it is not the most fun game to play, especially if your codebase is quite large. So, here I want to take a minute and discuss what is the correct way to rethrow the exception in .Net.
<!--more-->

### Why not to `throw ex`
Suppose you have a code that catches the exception (line 21), logs the necessary bits (line 23), rethrows the exception (line 24) and then the stack trace from the rethrown exception is outputted to the console in the main method (line 11).
<p>
    <script src="https://gist.github.com/BerserkerDotNet/36c1477fb6767406259e9fde2893d861.js"></script>
    <noscript>
        <pre>
namespace CSharpException
{
    public class Program
    {
        public static void Main(string[] args)
        {
            try
            {
                DoSomeUsefulWork();
            }
            catch(Exception ex) { Console.WriteLine(ex.StackTrace); }
        }

        private static void DoSomeUsefulWork()
        {
            try
            {
                ICanThrowException();
                ICanThrowException();
            }
            catch (Exception ex)
            {
                Log(ex);
                throw ex;
            }
        }

        private static void ICanThrowException()
        {
            throw new Exception(&quot;Bad thing happened&quot;);
        }

        private static void Log(Exception ex)
        {
            // Intentionally left blank
        }
    }
}
        </pre>
    </noscript>
</p>
The output of this program would be:
```
at CSharpException.Program.DoSomeUsefulWork() in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 24
at CSharpException.Program.Main(String[] args) in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 9
```
This could be quite unexpected. By looking at this output it seems like the exception was thrown on line 24 of `DoSomeUsefulWork` method. There is no mentioning of the `ICanThrowException` method or any signs that it was ever invoked. In this simple example, it might be easy to figure out what actually happened, but imagine if this is a real product code with hundreds of lines of complicated logic, in that case, it would take a lot of effort or might not be even possible to restore the complete chain of events that lead to the exception.

But why does it happen like this? Well, in C# when you want to throw the exception, you have to pass the exception object itself, that is usually done by instantiating a new instance of the class that represents the exception `throw new Exception()` (line 30). The thing is that for C# compiler it looks the same if you instantiate a new exception object or if you pass an already existing object.
This can be verified by looking at the IL code generated from the example. 
Here is the IL code for `ICanThrowException` method:
<p>
    <script src="https://gist.github.com/BerserkerDotNet/0d40ecc7b4d6e23dc2a97f4e1fcef640.js"></script>
    <noscript>
        <pre>
.method private hidebysig static void  ICanThrowException() cil managed
{
  // Code size       12 (0xc)
  .maxstack  8
  IL_0000:  nop
  IL_0001:  ldstr      &quot;Bad thing happened&quot;
  IL_0006:  newobj     instance void [mscorlib]System.Exception::.ctor(string)
  IL_000b:  throw
} // end of method Program::ICanThrowException
        </pre>
    </noscript>
</p>
On line 7 a new instance of the Exception class is created and the reference to it is placed on the stack. On line 8 the exception is thrown by the call to `throw` command.

The code for the catch block of the `DoSomeUsefulWork` method looks almost the same in terms of throwing the exception:
<p>
    <script src="https://gist.github.com/BerserkerDotNet/a3aa2480441ddc3d7b26bc3a10d561d3.js"></script>
    <noscript>
        <pre>
catch [mscorlib]System.Exception 
  {
    IL_0011:  stloc.0
    IL_0012:  nop
    IL_0013:  ldloc.0
    IL_0014:  call       void CSharpException.Program::Log(class [mscorlib]System.Exception)
    IL_0019:  nop
    IL_001a:  ldloc.0
    IL_001b:  throw
  }  // end handler
        </pre>
    </noscript>
</p>
Here on line 3 `stloc.0` command stores the exception reference to the local variable, on line 8 `ldloc.0` pushes the reference to the top of the stack and then `throw` command actually throws the exception. In `ICanThrowException` a brand new instance of an exception object is created and of course, CLR auto-populates the stack trace property of that exception. But now in the catch block of the `DoSomeUsefulWork` method, the same `throw` command is used and CLR will re-initialize the stack trace property of the exception that was passed to the `throw` instruction as if it was a brand new exception instantiated right there and make it look like the exception is thrown in the catch block itself.

### Just "throw"

The easiest way to rethrow the exception without losing the original stack trace is to just do `throw` and don't pass the exception object. Here is the same `DoSomeUsefulWork` method that just does `throw`:
<p>
    <script src="https://gist.github.com/BerserkerDotNet/32c9f8c88a4ecc77fa8a5b9165387668.js"></script>
    <noscript>
        <pre>
private static void DoSomeUsefulWork()
{
    try
    {
        ICanThrowException();
        ICanThrowException();
    }
    catch (Exception ex)
    {
        Log(ex);
        throw;
    }
}
        </pre>
    </noscript>
</p>
Notice that on line 11 `throw` instruction does not have any arguments. Here is the output of the program:
```
   at CSharpException.Program.ICanThrowException() in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 30
   at CSharpException.Program.DoSomeUsefulWork() in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 24
   at CSharpException.Program.Main(String[] args) in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 9
```
This makes much more sense, now it is clearly visible that the exception happened on line 30 of `ICanThrowException` method and was rethrown on line 24 of the `DoSomeUsefulWork` method.
The change is visible in the IL code as well
<p>
    <script src="https://gist.github.com/BerserkerDotNet/bcb85eac019342dafde95458f0b5e2b9.js"></script>
    <noscript>
        <pre>
  catch [mscorlib]System.Exception 
  {
    IL_0011:  stloc.0
    IL_0012:  nop
    IL_0013:  ldloc.0
    IL_0014:  call       void CSharpException.Program::Log(class [mscorlib]System.Exception)
    IL_0019:  nop
    IL_001a:  rethrow
  }  // end handler
        </pre>
    </noscript>
</p>
On the line 8 `rethrow` instruction is used instead of the `throw`. This instruction preserves exception's place of origin but it still has its flaws.

### Capture and throw
While it is true that `throw` preserves the information on where the exception was thrown, but it still does not give a complete picture of what happened. A careful reader might have noticed that in `DoSomeUsefulWork` method there are actually two calls to `ICanThrowException` method. This is not a typo, imagine that `DoSomeUsefulWork` is a fairly complex method with few `if` statements each of which can call to `ICanThrowException`. Now, the question is which one led to the exception? Unfortunately, `throw` does not give an answer to that. In the previous example, the only mentioning of the `DoSomeUsefulWork` method was line 24 and this is where the exception was rethrown, so it is unclear if it occurs on line 18 or line 19 (see the first code sample). In .Net Framework 4.5 [ExceptionDispatchInfo][1] class was introduced that can solve this issue. It is intended to be used by the [TPL][2] library, specifically to support throwing non-aggregated exceptions with `await` keyword. But since the class is public it can be used to rethrow exception even without [TPL][2]. Here is how it looks in the code (line 11):
<p>
    <script src="https://gist.github.com/BerserkerDotNet/059a1fa7a443587c5ccf3b8d9782a0b3.js"></script>
    <noscript>
        <pre>
private static void DoSomeUsefulWork()
{
    try
    {
        ICanThrowException();
        ICanThrowException();
    }
    catch (Exception ex)
    {
        Log(ex);
        ExceptionDispatchInfo.Capture(ex).Throw();
    }
}
        </pre>
    </noscript>
</p>
It works by first capturing the exception and then throwing it again. Here is the output of the program:
```
   at CSharpException.Program.ICanThrowException() in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 30
   at CSharpException.Program.DoSomeUsefulWork() in C:\UsersSource\Repos\CSharpException\CSharpException\Program.cs:line 18
--- End of stack trace from previous location where exception was thrown ---
   at System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw()
   at CSharpException.Program.DoSomeUsefulWork() in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 24
   at CSharpException.Program.Main(String[] args) in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 8
```
This is much better, not only it shows where the exception originated and where it was rethrown, but it also shows the complete invocation chain that led to the exception, making it much easier to deduct from the logs where and why the issue happened. The internals of [ExceptionDispatchInfo][1] is quite simple when `Capture` is called the new instance of [ExceptionDispatchInfo][1] is created and it holds a reference to the exception object that is passed as a parameter. When `Throw` is called it restores the stack trace of the original exception and then basically does a `throw ex`.

### What about wrapping the exception in another exception
One other way to preserve the stack trace information of the original exception on rethrow is to wrap the original exception with another exception, like this: 
<p>
    <script src="https://gist.github.com/BerserkerDotNet/3ac199c906d03345dea5f8ca274fb0d3.js"></script>
    <noscript>
        <pre>
private static void DoSomeUsefulWork()
{
    try
    {
        ICanThrowException();
        ICanThrowException();
    }
    catch (Exception ex)
    {
        Log(ex);
        throw new Exception(&quot;Rethrown&quot;, ex);
    }
}
        </pre>
    </noscript>
</p>
The output would be somewhat similar to the [ExceptionDispatchInfo][1] usage
```
System.Exception: Rethrown ---> System.Exception: Bad thing happened
   at CSharpException.Program.ICanThrowException() in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 30
   at CSharpException.Program.DoSomeUsefulWork() in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 18
   --- End of inner exception stack trace ---
   at CSharpException.Program.DoSomeUsefulWork() in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 24
   at CSharpException.Program.Main(String[] args) in C:\Users\Source\Repos\CSharpException\CSharpException\Program.cs:line 9
```
But, for this to work the main method should also be changed to log the entire exception `Console.WriteLine(ex)` or to extract the stack trace from the inner exception in addition to the main exception stack trace. It can be an option if `ExceptionDispatchInfo` is not available, but generally, it is better to use `ExceptionDispatchInfo` when possible to rethrow.

### Summary
In general, in .Net I would strongly advise against using `throw ex` to just rethrow the exception in a catch block as it destroys the information about where the exception was thrown originally and will definitely cause the frustration when looking into the logs and trying to figure out what had happened. With .Net Framework 4.5 and higher I would always use [ExceptionDispatchInfo][1] to rethrow as it gives the most complete picture of the events that happened. On versions of the framework lower than 4.5, I would just use `throw` as the simplest way to rethrow and to keep the information on the origin of the exception. Wrapping the exception with another exception to keep the information on what method led to the exception is simply not worth it.

Happy coding and hope you'll not have to play "Find where the exception happened." game ever!

[1]:https://docs.microsoft.com/en-us/dotnet/api/system.runtime.exceptionservices.exceptiondispatchinfo?redirectedfrom=MSDN&view=netframework-4.7.2
[2]:https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl