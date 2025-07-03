# Interprocess Communication: Strategies and Best Practices

<p align="center">
  <img src="https://github.com/Nice3point/InterprocessCommunication/assets/20504884/21d38cc0-9dfe-46af-959d-8deffaf91b3c" />
</p>

We all know how difficult it is to maintain large programs and keep up with progress. Revit plugin developers understand this better than anyone.
We have to write our programs on .NET Framework 4.8. We have to give up on modern and fast libraries.
This ultimately affects the users, who are forced to use outdated software.

In such scenarios, splitting an application into multiple processes using Named Pipes is an excellent solution due to its performance and reliability.
In this article, we will look at how to create and use Named Pipes for communication between a Revit application running on .NET 4.8 and its plugin running on .NET 7.

# Table of Contents

* [Introduction to using Named Pipes for communication between applications on different .NET versions](#introduction-to-using-named-pipes-for-communication-between-applications-on-different-net-versions)
* [What are Named Pipes?](#what-are-named-pipes)
* [Interaction between applications on .NET 4.8 and .NET 7](#interaction-between-applications-on-net-48-and-net-7)
  * [Creating a server](#creating-a-server)
  * [Creating a client](#creating-a-client)
  * [Transfer protocol](#transfer-protocol)
  * [Connection management](#connection-management)
  * [Two-way communication](#two-way-communication)
  * [Implementing a Revit plugin](#implementing-a-revit-plugin)
* [Installing .NET Runtime during plugin installation](#installing-net-runtime-during-plugin-installation)
* [Conclusion](#conclusion)

# Introduction to using Named Pipes for communication between applications on different .NET versions

In the world of application development, it is often necessary to ensure data exchange between different applications, especially when they run on different versions of .NET or different languages.
Splitting a single application into multiple processes must be justified. What is easier, calling a function directly, or exchanging messages? Obviously the first one.

So what are the advantages of doing this?

- Resolving dependency conflicts

  With each passing year, the size of Revit plugins grows larger and larger, and dependencies grow exponentially.
  Plugins can use incompatible versions of the same library, which will cause the program to crash. Process isolation solves this problem.

- Performance

    Below are performance measurements for sorting and mathematical calculations on different versions of .NET.

    ```
    BenchmarkDotNet v0.13.9, Windows 11 (10.0.22621.1702/22H2/2022Update/SunValley2)
    AMD Ryzen 5 2600X, 1 CPU, 12 logical and 6 physical cores
    .NET 7.0           : .NET 7.0.9 (7.0.923.32018), X64 RyuJIT AVX2
    .NET Framework 4.8 : .NET Framework 4.8.1 (4.8.9139.0), X64 RyuJIT VectorSize=256
    ```
  | Method      | Runtime            | Mean           | Error        | StdDev       | Allocated |
  |------------ |------------------- |---------------:|-------------:|-------------:|----------:|
  | ListSort    | .NET 7.0           | 1,113,161.8 ns | 20,385.15 ns | 21,811.88 ns |  804753 B |
  | ListOrderBy | .NET 7.0           | 1,064,851.1 ns | 12,401.25 ns | 11,600.13 ns |  807054 B |
  | MinValue    | .NET 7.0           |       979.4 ns |      7.40 ns |      6.56 ns |         - |
  | MaxValue    | .NET 7.0           |       970.6 ns |      4.32 ns |      3.60 ns |         - |
  | ListSort    | .NET Framework 4.8 | 2,144,723.5 ns | 40,359.72 ns | 37,752.51 ns | 1101646 B |
  | ListOrderBy | .NET Framework 4.8 | 2,192,414.7 ns | 25,938.78 ns | 24,263.15 ns | 1105311 B |
  | MinValue    | .NET Framework 4.8 |    58,019.0 ns |    460.30 ns |    430.57 ns |      40 B |
  | MaxValue    | .NET Framework 4.8 |    66,053.4 ns |    610.28 ns |    541.00 ns |      41 B |

  A 68-fold speed difference in finding the minimum value, and a complete absence of memory allocation, is impressive.

So how do you write a program on the latest version of .NET that will interact with an incompatible .NET framework?
Create two applications, a Server and a Client, without adding dependencies between them, and set up communication between them according to a configured protocol.

Below are some of the possible options for interaction between two applications:

1. Using WCF (Windows Communication Foundation)
2. Using sockets (TCP or UDP)
3. Using Named Pipes
4. Using operating system signals (e.g., Windows messages):

   Example code from Autodesk, communication between the Project Browser plugin and the Revit backend via messages

    ```c#
    public class DataTransmitter : IEventObserver
    {
        private void PostMessageToMainWindow(int iCmd) => 
            this.HandleOnMainThread((Action) (() => 
                Win32Api.PostMessage(Application.UIApp.getUIApplication().MainWindowHandle, 273U, new IntPtr(iCmd), IntPtr.Zero)));
    
        public void HandleShortCut(string key, bool ctrlPressed)
        {
            string lower = key.ToLower();
            switch (PrivateImplementationDetails.ComputeStringHash(lower))
            {
            case 388133425:
              if (!(lower == "f2")) break;
              this.PostMessageToMainWindow(DataTransmitter.ID_RENAME);
              break;
            case 1740784714:
              if (!(lower == "delete")) break;
              this.PostMessageToMainWindow(DataTransmitter.ID_DELETE);
              break;
            case 3447633555:
              if (!(lower == "contextmenu")) break;
              this.PostMessageToMainWindow(DataTransmitter.ID_PROJECTBROWSER_CONTEXT_MENU_POP);
              break;
            case 3859557458:
              if (!(lower == "c") || !ctrlPressed) break;
              this.PostMessageToMainWindow(DataTransmitter.ID_COPY);
              break;
            case 4077666505:
              if (!(lower == "v") || !ctrlPressed) break;
              this.PostMessageToMainWindow(DataTransmitter.ID_PASTE);
              break;
            case 4228665076:
              if (!(lower == "y") || !ctrlPressed) break;
              this.PostMessageToMainWindow(DataTransmitter.ID_REDO);
              break;
            case 4278997933:
              if (!(lower == "z") || !ctrlPressed) break;
              this.PostMessageToMainWindow(DataTransmitter.ID_UNDO);
              break;
            }
        }
    }
    ```

Each option has its own advantages and disadvantages, the most convenient in my opinion, for interaction on a single local machine is Named Pipes. We will consider it.

# What are Named Pipes?

Named Pipes are an inter-process communication (IPC) mechanism that allows processes to exchange data through named channels.
They provide a one-way or two-way connection between processes.
In addition to high performance, Named Pipes also offer various levels of security, making them an attractive solution for many inter-process communication scenarios.

# Interaction between applications on .NET 4.8 and .NET 7

Let's consider two applications, one of which contains business logic (server), and the other - a user interface (client).
To ensure communication between these two processes, a NamedPipe is used.

The principle of operation of a NamedPipe includes the following steps:

1. **Creating and configuring a NamedPipe**: The server creates and configures
   a NamedPipe with a specific name that will be available to the client. The client needs
   to know this name to connect to the pipe.
2. **Waiting for a connection**: The server starts waiting for a client to connect to the pipe.
   This is a blocking operation, and the server remains in a suspended state until the client connects.
3. **Connecting to a NamedPipe**: The client initiates a connection to the NamedPipe, specifying the name of the pipe to which it wants to connect.
4. **Data exchange**: After a successful connection, the client and server can exchange
   data in the form of byte streams. The client sends requests to execute business logic, and the server processes these requests and sends the results.
5. **Session termination**: After the data exchange is complete, the client and server can close the connection to the NamedPipe.

## Creating a server

On the .NET platform, the server part is represented by the `NamedPipeServerStream` class. The implementation of the class provides asynchronous and synchronous methods for working with NamedPipe.
To avoid blocking the main thread, we will use asynchronous methods.

Example code for creating a NamedPipeServer:

```C#
public static class NamedPipeUtil
{
    /// <summary>
    /// Create a server for the current user only
    /// </summary>
    public static NamedPipeServerStream CreateServer(PipeDirection? pipeDirection = null)
    {
        const PipeOptions pipeOptions = PipeOptions.Asynchronous | PipeOptions.WriteThrough;
        return new NamedPipeServerStream(
            GetPipeName(),
            pipeDirection ?? PipeDirection.InOut,
            NamedPipeServerStream.MaxAllowedServerInstances,
            PipeTransmissionMode.Byte,
            pipeOptions);
    }
    
    private static string GetPipeName()
    {
        var serverDirectory = AppDomain.CurrentDomain.BaseDirectory.TrimEnd(Path.DirectorySeparatorChar);
        var pipeNameInput = $"{Environment.UserName}.{serverDirectory}";
        var hash = new SHA256Managed().ComputeHash(Encoding.UTF8.GetBytes(pipeNameInput));
    
        return Convert.ToBase64String(hash)
            .Replace("/", "_")
            .Replace("=", string.Empty);
    }
}
```

The server name must not contain special characters to avoid an exception.
To create the pipe name, we will use a hash created from the username and the current folder, which is unique enough for the client to use this specific server when connecting.
You can change this behavior or use any name within your project, especially if the client and server are in different directories.

This approach is used in the [Roslyn .NET compiler](https://github.com/dotnet/roslyn). For those who want to delve deeper into this topic, I recommend studying the source code of the project.

`PipeDirection` specifies the direction of the channel, `PipeDirection.In` indicates that the server will only receive messages, and `PipeDirection.InOut` will be able to both receive and send them.

## Creating a client

To create a client, we will use the `NamedPipeClientStream` class. The code is almost identical to the server, and may differ slightly depending on the .NET version.
For example, in .NET framework 4.8 the `PipeOptions.CurrentUserOnly` value does not exist, but it appeared in .NET 7.

```C#
/// <summary>
/// Create a client for the current user only
/// </summary>
public static NamedPipeClientStream CreateClient(PipeDirection? pipeDirection = null)
{
    const PipeOptions pipeOptions = PipeOptions.Asynchronous | PipeOptions.WriteThrough | PipeOptions.CurrentUserOnly;
    return new NamedPipeClientStream(".",
        GetPipeName(),
        pipeDirection ?? PipeDirection.Out,
        pipeOptions);
}

private static string GetPipeName()
{
    var clientDirectory = AppDomain.CurrentDomain.BaseDirectory.TrimEnd(Path.DirectorySeparatorChar);
    var pipeNameInput = $"{System.Environment.UserName}.{clientDirectory}";
    var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(pipeNameInput));

    return Convert.ToBase64String(bytes)
        .Replace("/", "_")
        .Replace("=", string.Empty);
}
```

## Transfer protocol

A NamedPipe is a Stream, which allows us to write any sequence of bytes to the stream.
However, working with bytes directly can be inconvenient, especially when we are dealing with complex data or structures.
To simplify interaction with data streams and structure information in a convenient format, transfer protocols are used.

Transfer protocols define the format and order of data transmission between applications.
They provide information structuring to ensure understanding and correct interpretation of data between the sender and receiver.

In cases where we need to send "A request to execute a specific command on the server" or "A request to update application settings",
the server must understand how to process it from the client.
Therefore, to facilitate request processing and data exchange management, we will create an Enum `RequestType`.

```C#
public enum RequestType
{
    PrintMessage,
    UpdateModel
}
```

The request itself will be a class that will contain all the information about the transmitted data.

```c#
public abstract class Request
{
    public abstract RequestType Type { get; }

    protected abstract void AddRequestBody(BinaryWriter writer);

    /// <summary>
    ///     Write a Request to the given stream.
    /// </summary>
    public async Task WriteAsync(Stream outStream)
    {
        using var memoryStream = new MemoryStream();
        using var writer = new BinaryWriter(memoryStream, Encoding.Unicode);

        writer.Write((int) Type);
        AddRequestBody(writer);
        writer.Flush();

        // Write the length of the request
        var length = checked((int) memoryStream.Length);
        
        // There is no way to know the number of bytes written to
        // the pipe stream. We just have to assume all of them are written
        await outStream.WriteAsync(BitConverter.GetBytes(length), 0, 4);
        memoryStream.Position = 0;
        await memoryStream.CopyToAsync(outStream, length);
    }

    /// <summary>
    /// Write a string to the Writer where the string is encoded
    /// as a length prefix (signed 32-bit integer) follows by
    /// a sequence of characters.
    /// </summary>
    protected static void WriteLengthPrefixedString(BinaryWriter writer, string value)
    {
        writer.Write(value.Length);
        writer.Write(value.ToCharArray());
    }
}
```

The class contains the basic code for writing data to the stream. `AddRequestBody()` is used by derived classes to write their own structured data.

Examples of derived classes:

```C#
/// <summary>
/// Represents a Request from the client. A Request is as follows.
/// 
///  Field Name         Type            Size (bytes)
/// --------------------------------------------------
///  RequestType        Integer         4
///  Message            String          Variable
/// 
/// Strings are encoded via a character count prefix as a 
/// 32-bit integer, followed by an array of characters.
/// 
/// </summary>
public class PrintMessageRequest : Request
{
    public string Message { get; }

    public override RequestType Type => RequestType.PrintMessage;

    public PrintMessageRequest(string message)
    {
        Message = message;
    }

    protected override void AddRequestBody(BinaryWriter writer)
    {
        WriteLengthPrefixedString(writer, Message);
    }
}

/// <summary>
/// Represents a Request from the client. A Request is as follows.
/// 
///  Field Name         Type            Size (bytes)
/// --------------------------------------------------
///  ResponseType       Integer         4
///  Iterations         Integer         4
///  ForceUpdate        Boolean         1
///  ModelName          String          Variable
/// 
/// Strings are encoded via a character count prefix as a 
/// 32-bit integer, followed by an array of characters.
/// 
/// </summary>
public class UpdateModelRequest : Request
{
    public int Iterations { get; }
    public bool ForceUpdate { get; }
    public string ModelName { get; }

    public override RequestType Type => RequestType.UpdateModel;

    public UpdateModelRequest(string modelName, int iterations, bool forceUpdate)
    {
        Iterations = iterations;
        ForceUpdate = forceUpdate;
        ModelName = modelName;
    }

    protected override void AddRequestBody(BinaryWriter writer)
    {
        writer.Write(Iterations);
        writer.Write(ForceUpdate);
        WriteLengthPrefixedString(writer, ModelName);
    }
}
```

Using this structure, clients can create requests of various types, each of which defines its own logic for processing data and parameters.
The `PrintMessageRequest` and `UpdateModelRequest` classes provide examples of requests that can be sent to the server to perform specific tasks.

On the server side, it is necessary to develop the appropriate logic for processing incoming requests.
To do this, the server must read data from the stream and use the received parameters to perform the necessary operations.

Example of a received request on the server side:

```c#
/// <summary>
/// Represents a request from the client. A request is as follows.
/// 
///  Field Name         Type                Size (bytes)
/// ----------------------------------------------------
///  RequestType       enum RequestType   4
///  RequestBody       Request subclass   variable
/// 
/// </summary>
public abstract class Request
{
    public enum RequestType
    {
        PrintMessage,
        UpdateModel
    }
    
    public abstract RequestType Type { get; }

    /// <summary>
    ///     Read a Request from the given stream.
    /// </summary>
    public static async Task<Request> ReadAsync(Stream stream)
    {
        var lengthBuffer = new byte[4];
        await ReadAllAsync(stream, lengthBuffer, 4).ConfigureAwait(false);
        var length = BitConverter.ToUInt32(lengthBuffer, 0);

        var requestBuffer = new byte[length];
        await ReadAllAsync(stream, requestBuffer, requestBuffer.Length);

        using var reader = new BinaryReader(new MemoryStream(requestBuffer), Encoding.Unicode);

        var requestType = (RequestType) reader.ReadInt32();
        return requestType switch
        {
            RequestType.PrintMessage => PrintMessageRequest.Create(reader),
            RequestType.UpdateModel => UpdateModelRequest.Create(reader),
            _ => throw new ArgumentOutOfRangeException()
        };
    }
    
    /// <summary>
    /// This task does not complete until we are completely done reading.
    /// </summary>
    private static async Task ReadAllAsync(Stream stream, byte[] buffer, int count)
    {
        var totalBytesRead = 0;
        do
        {
            var bytesRead = await stream.ReadAsync(buffer, totalBytesRead, count - totalBytesRead);
            if (bytesRead == 0) throw new EndOfStreamException("Reached end of stream before end of read.");
            totalBytesRead += bytesRead;
        } while (totalBytesRead < count);
    }

    /// <summary>
    /// Read a string from the Reader where the string is encoded
    /// as a length prefix (signed 32-bit integer) followed by
    /// a sequence of characters.
    /// </summary>
    protected static string ReadLengthPrefixedString(BinaryReader reader)
    {
        var length = reader.ReadInt32();
        return length < 0 ? null : new string(reader.ReadChars(length));
    }
}

/// <summary>
/// Represents a Request from the client. A Request is as follows.
/// 
///  Field Name         Type            Size (bytes)
/// --------------------------------------------------
///  RequestType        Integer         4
///  Message            String          Variable
/// 
/// Strings are encoded via a character count prefix as a 
/// 32-bit integer, followed by an array of characters.
/// 
/// </summary>
public class PrintMessageRequest : Request
{
    public string Message { get; }

    public override RequestType Type => RequestType.PrintMessage;

    public PrintMessageRequest(string message)
    {
        Message = message;
    }

    protected override void AddRequestBody(BinaryWriter writer)
    {
        WriteLengthPrefixedString(writer, Message);
    }
}

/// <summary>
/// Represents a Request from the client. A Request is as follows.
/// 
///  Field Name         Type            Size (bytes)
/// --------------------------------------------------
///  RequestType        Integer         4
///  Iterations         Integer         4
///  ForceUpdate        Boolean         1
///  ModelName          String          Variable
/// 
/// Strings are encoded via a character count prefix as a 
/// 32-bit integer, followed by an array of characters.
/// 
/// </summary>
public class UpdateModelRequest : Request
{
    public int Iterations { get; }
    public bool ForceUpdate { get; }
    public string ModelName { get; }

    public override RequestType Type => RequestType.UpdateModel;

    public UpdateModelRequest(string modelName, int iterations, bool forceUpdate)
    {
        Iterations = iterations;
        ForceUpdate = forceUpdate;
        ModelName = modelName;
    }

    protected override void AddRequestBody(BinaryWriter writer)
    {
        writer.Write(Iterations);
        writer.Write(ForceUpdate);
        WriteLengthPrefixedString(writer, ModelName);
    }
}
```

The `ReadAsync()` method reads the request type from the stream, and then, depending on the type, reads the corresponding data and creates an object of the corresponding request.

Implementing a data transfer protocol and structuring requests in the form of classes allow you to effectively manage the exchange of information between the client and the server, while ensuring
a structured and understandable interaction between the two parties.
However, when designing such protocols, it is necessary to consider possible security risks, as well as to make sure that both ends of the interaction correctly handle all possible cases.

## Connection management

To send messages from the UI client to the server, we will create a `ClientDispatcher` class that will handle connections,
timeouts, and request scheduling, providing an interface for the client to interact with the server through named pipes.

```C#
/// <summary>
///     This class manages the connections, timeout and general scheduling of requests to the server.
/// </summary>
public class ClientDispatcher
{
    private const int TimeOutNewProcess = 10000;

    private Task _connectionTask;
    private readonly NamedPipeClientStream _client = NamedPipeUtil.CreateClient(PipeDirection.Out);

    /// <summary>
    ///     Connects to server without awaiting
    /// </summary>
    public void ConnectToServer()
    {
        _connectionTask = _client.ConnectAsync(TimeOutNewProcess);
    }

    /// <summary>
    ///     Write a Request to the server.
    /// </summary>
    public async Task WriteRequestAsync(Request request)
    {
        await _connectionTask;
        await request.WriteAsync(_client);
    }
}
```

Principle of operation:

1. **Initialization:** The constructor of the class initializes a `NamedPipeClientStream`, which is used to create a client stream with a named pipe.
2. **Establishing a connection:** The `ConnectToServer` method initiates an asynchronous connection to the server. The result of the operation is stored in a `Task`.
   `TimeOutNewProcess` is used to disconnect the client in case of unexpected exceptions.
3. **Sending requests:** The `WriteRequestAsync` method is designed for asynchronous sending of a `Request` object through an established connection. The request will be sent only after the connection is established.

To receive messages by the server, we will create a `ServerDispatcher` class that will manage the connection and read requests.

```C#
/// <summary>
///     This class manages the connections, timeout and general scheduling of the client requests.
/// </summary>
public class ServerDispatcher
{
    private readonly NamedPipeServerStream _server = NamedPipeUtil.CreateServer(PipeDirection.In);

    /// <summary>
    ///     This function will accept and process new requests until the client disconnects from the server
    /// </summary>
    public async Task ListenAndDispatchConnections()
    {
        try
        {
            await _server.WaitForConnectionAsync();
            await ListenAndDispatchConnectionsCoreAsync();
        }
        finally
        {
            _server.Close();
        }
    }

    private async Task ListenAndDispatchConnectionsCoreAsync()
    {
        while (_server.IsConnected)
        {
            try
            {
                var request = await Request.ReadAsync(_server);
                if (request.Type == Request.RequestType.PrintMessage)
                {
                    var printRequest = (PrintMessageRequest) request;
                    Console.WriteLine($"Message from client: {printRequest.Message}");
                }
                else if (request.Type == Request.RequestType.UpdateModel)
                {
                    var printRequest = (UpdateModelRequest) request;
                    Console.WriteLine($"The {printRequest.ModelName} model has been {(printRequest.ForceUpdate ? "forcibly" : string.Empty)} updated {printRequest.Iterations} times");
                }
            }
            catch (EndOfStreamException)
            {
                return; //Pipe disconnected
            }
        }
    }
}
```

Principle of operation:

1. **Initialization:** The constructor of the class initializes a `NamedPipeServerStream`, which is used to create a server stream with a named pipe.
2. **Listening for connections:** The `ListenAndDispatchConnections` method asynchronously waits for a client connection, after which it closes the named pipe and releases
   resources.
3. **Processing requests:** The `ListenAndDispatchConnectionsCoreAsync` method processes requests until the client disconnects.
   Depending on the type of request, the corresponding data is processed, for example, outputting the message content to the console or updating the model.

Example of sending a request from the UI to the server:

```C#

/// <summary>
///     Programme entry point
/// </summary>
public sealed partial class App
{
    public static ClientDispatcher ClientDispatcher { get; }

    static App()
    {
        ClientDispatcher = new ClientDispatcher();
        ClientDispatcher.ConnectToServer();
    }
}

/// <summary>
///     WPF view business logic 
/// </summary>
public partial class MainViewModel : ObservableObject
{
    [ObservableProperty] private string _message = string.Empty;

    [RelayCommand]
    private async Task SendMessageAsync()
    {
        var request = new PrintMessageRequest(Message);
        await App.ClientDispatcher.WriteRequestAsync(request);
    }

    [RelayCommand]
    private async Task UpdateModelAsync()
    {
        var request = new UpdateModelRequest(AppDomain.CurrentDomain.FriendlyName, 666, true);
        await App.ClientDispatcher.WriteRequestAsync(request);
    }
}
```

The example code is fully available in the repository, you can run it on your machine by following a few steps:

- Run "Build Solution"
- Run "Run OneWay\Backend"

The application will automatically start the Server and Client, and you will see the full output of messages transmitted over the NamedPipe in the IDE console.

## Two-way communication

Situations often arise where the usual one-way data transfer from the client to the server is insufficient.
In such cases, it is necessary to handle errors or send results in response. To ensure a more complex interaction between the client and the server, developers have to resort to
the use of two-way data transfer, which allows information to be exchanged in both directions.

As with requests, to effectively process responses, it is also necessary to define an enumeration for response types.
This will allow the client to correctly interpret the received data.

```C#
public enum ResponseType
{
    // The update request completed on the server and the results are contained in the message. 
    UpdateCompleted,
    
    // The request was rejected by the server.
    Rejected
}
```

To effectively process responses, you will need to create a new class called `Response`.
In terms of functionality, it is no different from the `Request` class, but unlike `Request`, which can be read on the server, `Response` will be written to the stream.

```C#
/// <summary>
/// Base class for all possible responses to a request.
/// The ResponseType enum should list all possible response types
/// and ReadResponse creates the appropriate response subclass based
/// on the response type sent by the client.
/// The format of a response is:
///
/// Field Name       Field Type          Size (bytes)
/// -------------------------------------------------
/// ResponseType     enum ResponseType   4
/// ResponseBody     Response subclass   variable
/// </summary>
public abstract class Response
{
    public enum ResponseType
    {
        // The update request completed on the server and the results are contained in the message. 
        UpdateCompleted,
    
        // The request was rejected by the server.
        Rejected
    }

    public abstract ResponseType Type { get; }

    protected abstract void AddResponseBody(BinaryWriter writer);

    /// <summary>
    ///     Write a Response to the stream.
    /// </summary>
    public async Task WriteAsync(Stream outStream)
    {
        // Same as request class from client
    }

    /// <summary>
    /// Write a string to the Writer where the string is encoded
    /// as a length prefix (signed 32-bit integer) follows by
    /// a sequence of characters.
    /// </summary>
    protected static void WriteLengthPrefixedString(BinaryWriter writer, string value)
    {
        // Same as request class from client
    }
}
```

You can find the derived classes in the project repository: [PipeProtocol](https://github.com/Nice3point/InterprocessCommunication/blob/main/TwoWay/Backend/Server/PipeProtocol.cs)

In order for the server to be able to send responses to the client, we must modify the `ServerDispatcher` class.
This will allow writing responses to the Stream after the task is completed.

We will also change the pipe direction to bidirectional:

```C#
_server = NamedPipeUtil.CreateServer(PipeDirection.InOut);

/// <summary>
///     Write a Response to the client.
/// </summary>
public async Task WriteResponseAsync(Response response) => await response.WriteAsync(_server);
```

To demonstrate the operation, we will add a 2-second delay, emulating a heavy task, in the `ListenAndDispatchConnectionsCoreAsync` method:

```C#
private async Task ListenAndDispatchConnectionsCoreAsync()
{
    while (_server.IsConnected)
    {
        try
        {
            var request = await Request.ReadAsync(_server);
            
            // ...
            if (request.Type == Request.RequestType.UpdateModel)
            {
                var printRequest = (UpdateModelRequest) request;

                await Task.Delay(TimeSpan.FromSeconds(2));
                await WriteResponseAsync(new UpdateCompletedResponse(changes: 69, version: "2.1.7"));
            }
        }
        catch (EndOfStreamException)
        {
            return; //Pipe disconnected
        }
    }
}
```

At the moment, the client does not process responses from the server.
Let's do this.
Let's create a `Response` class in the client that will process received responses.

```C#
/// <summary>
/// Base class for all possible responses to a request.
/// The ResponseType enum should list all possible response types
/// and ReadResponse creates the appropriate response subclass based
/// on the response type sent by the client.
/// The format of a response is:
///
/// Field Name       Field Type          Size (bytes)
/// -------------------------------------------------
/// ResponseType     enum ResponseType   4
/// ResponseBody     Response subclass   variable
/// 
/// </summary>
public abstract class Response
{
    public enum ResponseType
    {
        // The update request completed on the server and the results are contained in the message. 
        UpdateCompleted,

        // The request was rejected by the server.
        Rejected
    }

    public abstract ResponseType Type { get; }

    /// <summary>
    ///     Read a Request from the given stream.
    /// </summary>
    public static async Task<Response> ReadAsync(Stream stream)
    {
        // Same as request class from server
    }

    /// <summary>
    /// This task does not complete until we are completely done reading.
    /// </summary>
    private static async Task ReadAllAsync(Stream stream, byte[] buffer, int count)
    {
        // Same as request class from server
    }

    /// <summary>
    /// Read a string from the Reader where the string is encoded
    /// as a length prefix (signed 32-bit integer) followed by
    /// a sequence of characters.
    /// </summary>
    protected static string ReadLengthPrefixedString(BinaryReader reader)
    {
        // Same as request class from server
    }
}
```

Next, we will update the `ClientDispatcher` class so that it can process responses from the server. To do this, we will add a new method and change the direction to bidirectional.

```C#
_client = NamedPipeUtil.CreateClient(PipeDirection.InOut);

/// <summary>
///     Read a Response from the server.
/// </summary>
public async Task<Response> ReadResponseAsync() => await Response.ReadAsync(_client);
```

We will also add response handling to the ViewModel, where we will simply display it as a message.

```C#
[RelayCommand]
private async Task UpdateModelAsync()
{
    var request = new UpdateModelRequest(AppDomain.CurrentDomain.FriendlyName, 666, true);
    await App.ClientDispatcher.WriteRequestAsync(request);

    var response = await App.ClientDispatcher.ReadResponseAsync();
    if (response.Type == Response.ResponseType.UpdateCompleted)
    {
        var completedResponse = (UpdateCompletedResponse) response;

        MessageBox.Show($"{completedResponse.Changes} elements successfully updated to version {completedResponse.Version}");
    }
    else if (response.Type == Response.ResponseType.Rejected)
    {
        MessageBox.Show("Update failed");
    }
}
```

These changes will allow for more effective organization of the interaction between the client and the server, ensuring more complete and reliable processing of requests and responses.

## Implementing a Revit plugin

<p align="center">
  <img src="https://github.com/Nice3point/InterprocessCommunication/assets/20504884/09e0dee3-d4bd-4858-87eb-6bf6766b8dde" />
</p>

<p align="center">Technologies evolve, but Revit doesn't change © Confucius</p>

Currently, Revit uses .NET Framework 4.8.
However, to improve the user interface of plugins, let's consider moving to .NET 7.
It is important to note that the plugin backend will only interact with Revit on the outdated Framework, and will act as a server.

Let's create an interaction mechanism that will allow the client to send requests to delete model elements, and then receive responses about the result of the deletion.
To implement this functionality, we will use two-way data transfer between the server and the client.

The first step in our development process will be to teach the plugin to automatically close when Revit closes.
To do this, we wrote a method that sends the ID of the current process to the client.
This will help the client to automatically close its process when the parent Revit process closes.

Code for sending the current process ID to the client:

```C#
private static void RunClient(string clientName)
{
    var startInfo = new ProcessStartInfo
    {
        FileName = Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location)!.AppendPath(clientName),
        Arguments = Process.GetCurrentProcess().Id.ToString()
    };

    Process.Start(startInfo);
}
```

And here is the code for the client, which closes its process when the parent Revit process closes:

```C#
protected override void OnStartup(StartupEventArgs args)
{
    ParseCommandArguments(args.Args);
}

private void ParseCommandArguments(string[] args)
{
    var ownerPid = args[0];
    var ownerProcess = Process.GetProcessById(int.Parse(ownerPid));
    ownerProcess.EnableRaisingEvents = true;
    ownerProcess.Exited += (_, _) => Shutdown();
}
```

In addition, we need a method that will be responsible for deleting the selected model elements:

```C#
public static ICollection<ElementId> DeleteSelectedElements()
{
    var transaction = new Transaction(Document);
    transaction.Start("Delete elements");
    
    var selectedIds = UiDocument.Selection.GetElementIds();
    var deletedIds = Document.Delete(selectedIds);

    transaction.Commit();
    return deletedIds;
}
```

We will also update the `ListenAndDispatchConnectionsCoreAsync` method to handle incoming connections:

```C#
private async Task ListenAndDispatchConnectionsCoreAsync()
{
    while (_server.IsConnected)
    {
        try
        {
            var request = await Request.ReadAsync(_server);
            if (request.Type == Request.RequestType.DeleteElements)
            {
                await ProcessDeleteElementsAsync();
            }
        }
        catch (EndOfStreamException)
        {
            return; //Pipe disconnected
        }
    }
}

private async Task ProcessDeleteElementsAsync()
{
    try
    {
        var deletedIds = await Application.AsyncEventHandler.RaiseAsync(_ => RevitApi.DeleteSelectedElements());
        await WriteResponseAsync(new DeletionCompletedResponse(deletedIds.Count));
    }
    catch (Exception exception)
    {
        await WriteResponseAsync(new RejectedResponse(exception.Message));
    }
}
```

And finally, the updated ViewModel code:

```C#
[RelayCommand]
private async Task DeleteElementsAsync()
{
    var request = new DeleteElementsRequest();
    await App.ClientDispatcher.WriteRequestAsync(request);

    var response = await App.ClientDispatcher.ReadResponseAsync();
    if (response.Type == Response.ResponseType.Success)
    {
        var completedResponse = (DeletionCompletedResponse) response;
        MessageBox.Show($"{completedResponse.Changes} elements successfully deleted");
    }
    else if (response.Type == Response.ResponseType.Rejected)
    {
        var rejectedResponse = (RejectedResponse) response;
        MessageBox.Show($"Deletion failed\n{rejectedResponse.Reason}");
    }
}
```

# Installing .NET Runtime during plugin installation

Not every user may have the latest version of the .NET Runtime installed on their local machine, so we need to make changes to the plugin installer.

If you are using [Nice3point.RevitTemplates](https://github.com/Nice3point/RevitTemplates) templates, then making changes will not be difficult.
The templates use the WixSharp library, which allows you to create .msi files directly in C#.

To add custom actions and install the .NET Runtime, we will create a `CustomAction`

```C#
public static class RuntimeActions
{
    /// <summary>
    ///     Add-in client .NET version
    /// </summary>
    private const string DotnetRuntimeVersion = "7";

    /// <summary>
    ///     Direct download link
    /// </summary>
    private const string DotnetRuntimeUrl = $"https://aka.ms/dotnet/{DotnetRuntimeVersion}.0/windowsdesktop-runtime-win-x64.exe";

    /// <summary>
    ///     Installing the .NET runtime after installing software
    /// </summary>
    [CustomAction]
    public static ActionResult InstallDotnet(Session session)
    {
        try
        {
            var isRuntimeInstalled = CheckDotnetInstallation();
            if (isRuntimeInstalled) return ActionResult.Success;

            var destinationPath = Path.Combine(Path.GetTempPath(), "windowsdesktop-runtime-win-x64.exe");

            UpdateStatus(session, "Downloading .NET runtime");
            DownloadRuntime(destinationPath);

            UpdateStatus(session, "Installing .NET runtime");
            var status = InstallRuntime(destinationPath);

            var result = status switch
            {
                0 => ActionResult.Success,
                1602 => ActionResult.UserExit,
                1618 => ActionResult.Success,
                _ => ActionResult.Failure
            };

            File.Delete(destinationPath);
            return result;
        }
        catch (Exception exception)
        {
            session.Log("Error downloading and installing DotNet: " + exception.Message);
            return ActionResult.Failure;
        }
    }

    private static int InstallRuntime(string destinationPath)
    {
        var startInfo = new ProcessStartInfo(destinationPath)
        {
            Arguments = "/q",
            UseShellExecute = false
        };

        var installProcess = Process.Start(startInfo)!;
        installProcess.WaitForExit();
        return installProcess.ExitCode;
    }

    private static void DownloadRuntime(string destinationPath)
    {
        ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12 | SecurityProtocolType.Tls11 | SecurityProtocolType.Tls;

        using var httpClient = new HttpClient();
        var responseBytes = httpClient.GetByteArrayAsync(DotnetRuntimeUrl).Result;

        File.WriteAllBytes(destinationPath, responseBytes);
    }

    private static bool CheckDotnetInstallation()
    {
        var startInfo = new ProcessStartInfo
        {
            FileName = "dotnet",
            Arguments = "--list-runtimes",
            RedirectStandardOutput = true,
            UseShellExecute = false,
            CreateNoWindow = true
        };

        try
        {
            var process = Process.Start(startInfo)!;
            var output = process.StandardOutput.ReadToEnd();
            process.WaitForExit();

            return output.Split('\n')
                .Where(line => line.Contains("Microsoft.WindowsDesktop.App"))
                .Any(line => line.Contains($"{DotnetRuntimeVersion}."));
        }
        catch
        {
            return false;
        }
    }

    private static void UpdateStatus(Session session, string message)
    {
        var record = new Record(3);
        record[2] = message;

        session.Message(InstallMessage.ActionStart, record);
    }
}
```

This code checks if the required version of .NET is installed on the local machine, and if not, it downloads and installs it.
During installation, the status of the current progress of downloading and unpacking the Runtime is updated.

All that remains is to connect the `CustomAction` to the WixSharp project, for this we will initialize the `Actions` property:

```C#
var project = new Project
{
    Name = "Wix Installer",
    UI = WUI.WixUI_FeatureTree,
    GUID = new Guid("8F2926C8-3C6C-4D12-9E3C-7DF611CD6DDF"),
    Actions = new Action[]
    {
        new ManagedAction(RuntimeActions.InstallDotnet, 
            Return.check,
            When.Before,
            Step.InstallFinalize,
            Condition.NOT_Installed)
    }
};
```

# Conclusion

In this article, we looked at how Named Pipes, primarily used for communication between different processes, can be used in scenarios where data exchange between applications on different versions of .NET is required.
When dealing with code that needs to be maintained across multiple versions, a well-thought-out Inter-Process Communication (IPC) strategy can be useful and provide key benefits, such as:

- Resolving dependency conflicts
- Improving performance
- Functional flexibility

We discussed the process of creating a server and a client that interact with each other through a predefined protocol, as well as various ways to manage connections.

We looked at an example of server responses and a demonstration of the operation of both sides of the interaction.

Finally, we highlighted how Named Pipes are used in the development of a Revit plugin to ensure communication between the backend, running on the outdated .NET 4.8 platform, and the user interface, running on the newer .NET 7 version.

Demonstration code for each part of this article is available on GitHub.

In certain cases, splitting applications into separate processes can not only reduce dependencies in the program, but also speed up its execution.
But let's not forget that the choice of approach requires analysis and should be based on the real requirements and limitations of your project.

We hope that this article will help you find the optimal solution for your inter-process communication scenarios and give you an understanding of how to apply IPC approaches in practice.