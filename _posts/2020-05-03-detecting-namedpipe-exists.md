---
layout: post
title: Detecting that NamedPipe exists
date: '2020-05-03T12:00:00.000-07:00'
author: Andrii Snihyr
tags:
- .Net
- C#
- NamedPipe
- Exists
- Check
- Pipe
- File.Exists
- Directory.GetFiles
modified_time: '2020-05-03T12:00:00.000-07:00'
---
In my project at work we use NamedPipe as one of the telemetry transports when running the app locally. All seemed to work fine, until I decided to write a [BenchmarkDotNet][BDN] diagnoser for the application that was supposed to use NamedPipe telemetry transport to collect certain metrics.
<!--more-->

### TL;TR
Don't use `File.Exists` it is not reliable, use [Directory.GetFiles][Directory.GetFiles.API] with `searchPattern` parameter. `System.IO.Directory.GetFiles("\\\\.\\pipe\\", "testpipe*").Length > 1`.

### What is NamedPipe
[Official][PipeDef] definition states: "A named pipe is a named, one-way or duplex pipe for communication between the pipe server and one or more pipe clients. All instances of a named pipe share the same pipe name, but each instance has its own buffers and handles, and provides a separate conduit for client/server communication. The use of instances enables multiple pipe clients to use the same named pipe simultaneously.". In other words, NamedPipe is a way to communicate between to processes on the same machine. They way it works is by creating one server stream and one or many client streams. When creating a stream you suppose to provide a name for the pipe, hence NamedPipe 😊. For example:

<p>
<script src="https://gist.github.com/BerserkerDotNet/a2351d072bcc7a9d8ed18107b0f7b8b7.js"></script>
    <noscript>
        <pre>
// Server
using var pipeServer = new NamedPipeServerStream("testpipe", PipeDirection.InOut);
pipeServer.WaitForConnection();
// Do some useful stuff

// Client
 using var pipeClient = new NamedPipeClientStream(".", "testpipe", PipeDirection.In);
 pipeClient.Connect();
 // Read from pipe
        </pre>
    </noscript>
</p>

Despite the dedicated API for NamedPipes, under the hood Windows treats each pipe as a file object. There is no actual file exists, but File API such as `WriteFile` or `ReadFile` available in Windows will send data to the pipe. This information will be useful later.

### Metrics I get look incomplete
I mentioned before that the idea was to use NamedPipe telemetry transport to write a [BenchmarkDotNet][1] diagnoser that will collect metrics from the application. When I finished writing the diagnoser I noticed that the metrics look weird, they felt incomplete as if some of the telemetry messages didn't get through. After quick debugging sessions I have verified that this is actually the case, certain telemetry messages don't get to the diagnoser. Time to debug. Here is the code that sends the event:

<p>
<script src="https://gist.github.com/BerserkerDotNet/d1f938af9ae1b5e69a1b5c9cdd93745d.js"></script>
    <noscript>
        <pre>
public const int DefaultTimeout = 60000;

protected override void SendEvent(string eventName, Dictionary<string, string> eventData)
{
    try
    {
        eventData.Add(nameof(eventName), eventName);
        if (!File.Exists($"\\\\.\\pipe\\{this.pipeaddress.AsPipeName()}"))
        {
            return;
        }

        using (var client = this.CreateClientStream())
        {
            var dataArray = Encoding.UTF8.GetBytes(JsonConvert.SerializeObject(eventData));
            client.Connect(DefaultTimeout);
            client.Write(dataArray, 0, dataArray.Length);
            client.Flush();
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Couldn't send event. {ex.Message}");
    }
}
        </pre>
    </noscript>
</p>

At first glance, nothing suspicious, the code checks if the pipe exists, connects, and sends the telemetry message. My first suspicion was that maybe there is an exception when sending the message. I have put a breakpoint in the catch block and waited for it to light up, but nothing happened, there are no exceptions happening.

Next stop was the check for if pipe exists. That would be weird, cause I know that the pipe is there, but I mean when did things start to make sense 😊 I have put a break point on the `return` statement and "Voila!" it light up. Strange, I know for sure that pipe is there and some messages are getting through how come one moment it thinks it is there and a few milliseconds later it thinks it is not. Schrödinger's pipe.
I quickly wrote a code that would run an infinite loop `while(true)` with `File.Exists` inside to detect at what point `File.Exists` check does not work. It looks like that when the pipe is being written to `File.Exists` check thinks that pipe is gone.

### Collapsing the wave function
[Under the hood][File.Exists.Code] `File.Exists` checks for `Read` permission on the file. Technically, it might be the case that you cannot read from the pipe at the same time you're writing to it. Quick search on if you can do simultaneous Read and Write operations revealed that it is somewhat possible, but would require some ["Dark Magic"][OverlappedReadWrite] that is not exposed via .Net API and definingly out of the question for a simple task of transmitting telemetry when running application locally.

First question in my head was, is there a better way to check if the named pipe exists. Browsing documentation and StackOverflow revealed the sad answer, there is no API exposed to tell you if the pipe is there or no. There are few articles on StackOverflow that suggest to use Mutex or global EventWaitHandle. None of these sounded appealing to me.

Next question I asked was, what if I don't do this check at all? Well, there is `client.Connect(DefaultTimeout);` where `DefaultTimeout` is set to a minute. This means that if the pipe is not there each time we try to send a message the code will wait for a whole minute before failing with `TimeoutException`, this is not acceptable. Even reducing `DefaultTimeout` to something manageable would not be a good option as the `TimeoutException` would be there.

So, how do I reliably check that the pipe is there? Since I know that pipe is treated as a file by Windows, another way to phrase this question would be, how do I reliably check that file is there? `Directory.GetFiles`, this should not check for permissions, it should actually enumerate files. Quick look at the [source code][Directory.GetFiles], and yes, it does create an iterator. Plug it in `Directory.GetFiles(@"\\.\\pipe\\").Any(f => f.Contains("testpipe"));` to the program that checks if a pipe is there in the infinite loop, and "Bingo!", it works, not a single false negative. Also, the second I stop the pipe server it correctly detects that the pipe is gone.

### What would it cost me
Checking for `Read` permissions on the file is fast, how much it will cost to use `Directory.GetFiles`. Additionally, is there a difference between filtering myself or using `searchPattern` parameter of the `Directory.GetFiles`. [BenchmarkDotNet][BDN] to the rescue:
<p>
<script src="https://gist.github.com/BerserkerDotNet/594c64519975fb14112e589e1fa1dd7b.js"></script>
    <noscript>
        <pre>
[MemoryDiagnoser]
public class NamedPipeSearchBenchmark
{
    [Params(1, 100, 1000, 10_000)]
    public int Iterations;

    private NamedPipeServerStream namedPipeServer;

    [GlobalSetup]
    public void Setup()
    {
        namedPipeServer = new NamedPipeServerStream("testpipe", PipeDirection.InOut, 10, PipeTransmissionMode.Message, PipeOptions.Asynchronous, 0, 0);
    }

    [GlobalCleanup]
    public void Cleanup()
    {
        namedPipeServer.Dispose();
    }

    [Benchmark(Baseline = true)]
    public bool File()
    {
        var result = false;
        for (int i = 0; i < Iterations; i++)
        {
            result &= System.IO.File.Exists("\\\\.\\pipe\\testpipe");
        }
        return result;
    }

    [Benchmark]
    public bool Directory()
    {
        var result = false;
        for (int i = 0; i < Iterations; i++)
        {
            result &= System.IO.Directory.GetFiles("\\\\.\\pipe\\").Any(f => f.Contains("testpipe"));
        }
        return result;
    }

    [Benchmark]
    public bool DirectoryFilter()
    {
        var result = false;
        for (int i = 0; i < Iterations; i++)
        {
            result &= System.IO.Directory.GetFiles("\\\\.\\pipe\\", "testpipe*").Length == 1;
        }
        return result;
    }
}
        </pre>
    </noscript>
</p>

Results are somewhat expected:

```
BenchmarkDotNet=v0.12.1, OS=Windows 10.0.18363.778 (1909/November2018Update/19H2)
Intel Core i7-9700K CPU 3.60GHz (Coffee Lake), 1 CPU, 8 logical and 8 physical cores
.NET Core SDK=3.1.300-preview-015135
  [Host]     : .NET Core 3.1.2 (CoreCLR 4.700.20.6602, CoreFX 4.700.20.6702), X64 RyuJIT
  DefaultJob : .NET Core 3.1.2 (CoreCLR 4.700.20.6602, CoreFX 4.700.20.6702), X64 RyuJIT

```

|          Method | Iterations |          Mean |         Error |        StdDev | Ratio | RatioSD |      Gen 0 | Gen 1 | Gen 2 |   Allocated |
|---------------- |----------- |--------------:|--------------:|--------------:|------:|--------:|-----------:|------:|------:|------------:|
|            **File** |          **1** |      **22.14 μs** |      **0.265 μs** |      **0.248 μs** |  **1.00** |    **0.00** |          **-** |     **-** |     **-** |        **32 B** |
|       Directory |          1 |      62.93 μs |      1.145 μs |      1.071 μs |  2.84 |    0.05 |     1.8311 |     - |     - |     11713 B |
| DirectoryFilter |          1 |      54.23 μs |      0.984 μs |      0.921 μs |  2.45 |    0.05 |     0.0610 |     - |     - |       440 B |
|                 |            |               |               |               |       |         |            |       |       |             |
|            **File** |        **100** |   **2,236.88 μs** |     **44.168 μs** |     **55.859 μs** |  **1.00** |    **0.00** |          **-** |     **-** |     **-** |      **3200 B** |
|       Directory |        100 |   7,797.07 μs |     75.978 μs |     63.445 μs |  3.46 |    0.10 |   187.5000 |     - |     - |   1186420 B |
| DirectoryFilter |        100 |   6,961.27 μs |     53.226 μs |     44.446 μs |  3.09 |    0.08 |          - |     - |     - |     44004 B |
|                 |            |               |               |               |       |         |            |       |       |             |
|            **File** |       **1000** |  **23,092.60 μs** |    **448.595 μs** |    **498.612 μs** |  **1.00** |    **0.00** |          **-** |     **-** |     **-** |     **32000 B** |
|       Directory |       1000 |  77,776.36 μs |  1,464.263 μs |  1,438.102 μs |  3.36 |    0.10 |  1857.1429 |     - |     - |  11864162 B |
| DirectoryFilter |       1000 |  54,087.57 μs |    728.358 μs |    608.212 μs |  2.33 |    0.06 |          - |     - |     - |    440136 B |
|                 |            |               |               |               |       |         |            |       |       |             |
|            **File** |      **10000** | **220,174.95 μs** |  **1,432.402 μs** |  **1,118.325 μs** |  **1.00** |    **0.00** |          **-** |     **-** |     **-** |    **320000 B** |
|       Directory |      10000 | 616,890.04 μs | 11,154.419 μs | 10,433.850 μs |  2.80 |    0.05 | 17000.0000 |     - |     - | 112560000 B |
| DirectoryFilter |      10000 | 522,347.14 μs |  8,473.153 μs |  7,511.230 μs |  2.37 |    0.04 |          - |     - |     - |   4400000 B |


Using `Directory.GetFiles` with `searchPattern` is the best option, it is about 2.3 to 3.0 times slower than `File.Exists` and allocates ~13x more bytes, but it actually works.

### Summary
At the end I went with using [Directory.GetFiles][Directory.GetFiles.API] with `searchPattern` parameter. Additionally, I moved the check for pipe's existence to the class constructor to avoid calling it multiple time. It worked for me as I know that our telemetry emitter is registered as a `InstancePerLifetimeScope`. In general I would recommend to reduce number of times this check is performed if possible.

Happy hosting!

[BDN]: https://benchmarkdotnet.org/
[PipeDef]: https://docs.microsoft.com/en-us/windows/win32/ipc/named-pipes
[File.Exists.Code]: https://referencesource.microsoft.com/#mscorlib/system/io/file.cs,3360368484a9f131
[OverlappedReadWrite]: https://docs.microsoft.com/en-us/windows/win32/ipc/named-pipe-server-using-overlapped-i-o
[Directory.GetFiles]: https://referencesource.microsoft.com/#mscorlib/system/io/directory.cs,f9704790d3b23471
[Directory.GetFiles.API]: https://docs.microsoft.com/en-us/dotnet/api/system.io.directory.getfiles?view=netcore-3.1#System_IO_Directory_GetFiles_System_String_System_String_