---
title: 1. Exporting EventSource logs to CSV
images: []
---

# 1. Exporting EventSource logs to CSV

Here's a scenario - you're building a service and you're emitting logs as you should be.
But your service is configured to send those logs to a log collector.
You don't have a lot of budget and you're using a log collector in production but locally you only write out to console (or Debug window in Visual Studio).
Maybe you want to persist your logs over a few sessions, maybe you have an automated test environment which hosts your service without console output being saved anywhere.
Sure you might implement a file sink for your logs.

Or you might decide to capture the data from the EventSource your service might be already using.

## EventSource?

This post is for .NET devs. What is an EventSource?
It's an old API designed in .NET Framework to pipe events into the ETW (Event Tracing for Windows) system.
It has been revamped in .NET Core to be multi-platform and it's the provider of data into `dotnet-trace` tool.
Many internal components emit EventSource events (like GC for example).

And some logging libraries do as well. For example `Microsoft.Extensions.Logging` is able to export logs to an event source.
See [AddEventSourceLogger](https://github.com/dotnet/runtime/blob/main/src/libraries/Microsoft.Extensions.Logging.EventSource/src/EventSourceLoggerFactoryExtensions.cs) method.
It's on by default with the default Host builder.

References:

* [EventSource - Getting started](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/eventsource-getting-started)
* [EventSource provider](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging-providers#event-source)

## PerfView library

So you can capture events using `dotnet-trace` and view them in standalone [PerfView](https://github.com/microsoft/perfview/releases) application.
It's powerful but not the most comfortable way of dealing with text logs.
Not that CSV is much better, but maybe a little.

PerfView ships either as a GUI app or as a NuGet package `Microsoft.Diagnostics.Tracing.TraceEvent`.
You can use the library to either:

* open a file produced by `dotnet-trace` - using `new EventPipeEventSource(inputFileName)`.
* listen to any events from a provider - using `new TraceEventSession(sessionName)` with `session.EnableProvider(providerName)`.

I opted for the second one, because it allows us to start capturing events as soon as the application starts
and it can simultaneously process events from multiple processes.

For example to listen to events from the `Microsoft-Extensions-Logging` EventSource
([source](https://github.com/dotnet/runtime/blob/main/src/libraries/Microsoft.Extensions.Logging.EventSource/src/LoggingEventSource.cs)):

```csharp
using (TraceEventSession session = new TraceEventSession("MyTraceSession"))
{
    // Enable the ETW session to listen for events from an EventSource
    session.EnableProvider("Microsoft-Extensions-Logging");

    // filter events to only a specific type and process with callback
    // we might only listen for MessageJson events for example
    session.Source.Dynamic.AddCallbackForProviderEvents(
        (string providerName, string eventName) =>
            providerName == "Microsoft-Extensions-Logging" && eventName == "MessageJson"
                ? EventFilterResponse.AcceptEvent
                : EventFilterResponse.RejectEvent;,
        (TraceEvent data) => {/* do stuff with the event */});

    session.Source.Process(); // Listen to events and invoke callback for events in the source
}
```

The `AddCallbackForProviderEvents` takes two functions:

* event filter - which allows you to specify which events your processor supports,
* event callback - which allows you to process events one by one.

So for me the callback read data from `TraceEvent` object and emitted them to CSV files.
The event object contains info about which process emitted it and the payload with data.

You can either process event with typed handlers (which I didn't manage to do because you need to generate a C# class from ETW manifest)
or a dynamic handler. In this case I'm reading dynamic properties from payload like this:

```csharp
TimeStamp = data.TimeStamp.ToString("s"),
ThreadId = data.ThreadID.ToString("X"),
TagId = new EventId((int)data.PayloadByName("EventId")).ToTagId(),
Level = ((LogLevel)data.PayloadByName("Level")).ToString(),
LoggerName = data.PayloadStringByName("LoggerName"),
Message = data.PayloadStringByName("FormattedMessage"),
ExceptionDetails = data.PayloadStringByName("ExceptionJson"),
```

If your event source emits structures, you can do this `(IDictionary<string, object>)data.PayloadByName("ComplexColumn")`.

So I wrote a little bit more code to use the `CsvHelper` library and write out the CSV files.
I'm not gonna post all of the code but it goes a little bit like this:

On each event:

1. Get CSV file name (process name + PID),
2. Create (or get from cache) an instance of the `CsvWriter` for the file name:
    * if the file didn't exist we create it and write the header, otherwise just append to it
3. Extract a row of data from event,
4. Write a row to the CSV file.

References:

* [PerfView application tutorials](https://learn.microsoft.com/en-us/shows/perfview-tutorial/)
* [The Microsoft.Diagnostics.Tracing.TraceEvent Library](https://github.com/microsoft/perfview/blob/main/documentation/TraceEvent/TraceEventLibrary.md)
* [The TraceEvent Library Programmers Guide](https://github.com/microsoft/perfview/blob/main/documentation/TraceEvent/TraceEventProgrammersGuide.md)
* [dotnet-trace tool](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)
* [CsvHelper website](https://joshclose.github.io/CsvHelper/)
* [Ways to query a CSV file with SQL](https://superuser.com/questions/7169/querying-a-csv-file)

## Building my tool

I decided to build my tool (which was like 4 C# files) into a single standalone executable,
because I needed to deploy it into our test environment to collect logs from automated testing.

An alternative is to create your tool in a portable way as a [dotnet tool](https://learn.microsoft.com/en-us/dotnet/core/tools/global-tools)
but to use it the machine needs to have .NET SDK installed.

So here's my configuration to publish a single file:

```xml
<DebugType>Embedded</DebugType>
<SelfContained>true</SelfContained>
<PublishSingleFile>true</PublishSingleFile>
<GenerateDocumentationFile>false</GenerateDocumentationFile>
<EnableCompressionInSingleFile>true</EnableCompressionInSingleFile>
<IncludeAllContentForSelfExtract>true</IncludeAllContentForSelfExtract>
```

I was able to call `dotnet publish` and get a nice executable.

But I wanted to package it into a NuGet package which would contain just a `tools` folder with the executable.
This took me two days to figure out fully.

First, we want to enabling packing:

```xml
<IsPackable>true</IsPackable>
```

Next, since we don't want to include build outputs, but instead publish output, we will disable that:

```xml
<!-- don't include DLL files produced by this project because we're manually adding publish output -->
<IncludeBuildOutput>false</IncludeBuildOutput>
```

Now, I'm going to add two targets - first one triggers the publish action after building and before packing,
second one includes the publish output in the package:

```xml
<Target Name="PublishBeforePack" BeforeTargets="GenerateNuspec" AfterTargets="Build">
  <CallTarget Targets="Publish" />
</Target>

<Target Name="UpdatePackOutput" BeforeTargets="GenerateNuspec" AfterTargets="Publish">
  <ItemGroup>
    <_PackageFiles Include="@(PublishItemsOutputGroupOutputs)">
      <FinalOutputPath>%(PublishItemsOutputGroupOutputs.OutputPath)</FinalOutputPath>
      <PackagePath>tools</PackagePath>
    </_PackageFiles>
  </ItemGroup>
</Target>
```

Finally, since the package has just a tool, it doesn't need any dependencies.
We can read online that property `PrivateAssets="all"` removes the dependency from nuspec.
But it also removes it from publish. After a while I found some issue on GitHub that mentioned
there's another property called `Publish` and setting it to true preserves the desired behavior for publishing.

```xml
<ItemGroup>
  <PackageReference Include="CsvHelper" Publish="True" PrivateAssets="all" />
  <PackageReference Include="Microsoft.Diagnostics.Tracing.TraceEvent" Publish="True" PrivateAssets="all" />
</ItemGroup>
```

But now that we've gotten rid of library dependencies we need to do remove the framework dependency as well:

```xml
<!-- suppress framework dependency when there's no library dependencies due to PrivateAssets=all -->
<SuppressDependenciesWhenPacking>true</SuppressDependenciesWhenPacking>
```

That's it. Well, there's one thing you need to do if you're still getting errors -
disable `IncludeSymbols` property which would generate the `snupkg` file.

With the package published to a NuGet source, you can use the following to extract the tool to
a projects output:

```xml
<ItemGroup>
  <PackageReference Include="LocalLogsExport" Version="1.0.0" ExcludeAssets="all" GeneratePathProperty="true" />
  <None Include="$(PkgLocalLogsExport)\tools\*.exe" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

References:

* [Create dotnet tool](https://learn.microsoft.com/en-us/dotnet/core/tools/global-tools-how-to-create)
* [Publish single file](https://learn.microsoft.com/en-us/dotnet/core/deploying/single-file/overview)
* [MSBuild debugging tool](https://msbuildlog.com/)
    * this is a life saver
* [MSBuild targets documentation](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-targets?view=vs-2022)
* [CallTarget task documentation](https://learn.microsoft.com/en-us/visualstudio/msbuild/calltarget-task?view=vs-2022)
* [PackageReference documentation](https://learn.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files)
* [Nuspec documentation](https://learn.microsoft.com/en-us/nuget/reference/nuspec)
    * I was inspecting the nuspec generate in the obj folder
* [Suppress the &lt;dependencies&gt; element when packing a project #5132](https://github.com/NuGet/Home/issues/5132)
* [NU5017 reported for symbol packages without clarifying this #10372](https://github.com/NuGet/Home/issues/10372)
    * that was the issue bugging me a long time

## Graceful stopping

The way I wrote the tool the CSV file wasn't being flushed on each event.
So if you killed it, you'd loose some rows only collected in memory.
Instead I subscribed to the console interrupt event to try to enable disposal of the CSV writer.

```csharp
private static void ConfigureProcesTermination(TraceEventSession session)
{
    Console.CancelKeyPress += (_, args) =>
    {
        args.Cancel = true; // prevent process from being terminated before cleanup
        session.Source.StopProcessing(); // allow call to session.Source.Process() to return
                                         // and nicely exit the using block
    };

    // if we are being stopped more forcefully let's try to at least clean up the session from the system
    AppDomain.CurrentDomain.ProcessExit += (_, _) => session.Stop(noThrow: true);
}
```

And then I was automating execution of my tool with Powershell and had to write this
monstrosity to gracefully stop the process:

```powershell
# Stop trace capture gracefully (so that any buffers are flushed to disk) by sending it a CTRL+C signal
# Apparently there's no native way to do it in Powershell, so we're calling a kernel function
# see https://stackoverflow.com/a/64930077
# The FreeConsole and AttachConsole isolates the process from the current Powershell process (otherwise the Ctrl+C kills the script itself as well)
$MemberDefinition = '
    [DllImport("kernel32.dll")]public static extern bool FreeConsole();
    [DllImport("kernel32.dll")]public static extern bool AttachConsole(uint p);
    [DllImport("kernel32.dll")]public static extern bool GenerateConsoleCtrlEvent(uint e, uint p);
    public static void SendCtrlC(uint p) {
        FreeConsole();
        if (AttachConsole(p)) {
            GenerateConsoleCtrlEvent(0, p);
            FreeConsole();
        }
        AttachConsole(uint.MaxValue);
    }'
Add-Type -Name 'Console' -Namespace 'Process' -MemberDefinition $MemberDefinition

$(Get-Process -Name 'LocalLogsExport').Id | foreach { [Process.Console]::SendCtrlC($_) }

Wait-Process -Name 'LocalLogsExport' -Timeout 30 -ErrorAction Ignore
```
