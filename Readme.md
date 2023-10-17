# Interprocess Communication: Strategies and Best Practices

<p align="center">
  <img src="https://github.com/Nice3point/InterprocessCommunication/assets/20504884/21d38cc0-9dfe-46af-959d-8deffaf91b3c" />
</p>

Все мы знаем как сложно поддерживать крупные программы, и успевать за прогрессом. Разработчики плагинов для Revit это понимают как никто лучше.
Нам приходится писать свои программы на .NET Framework 4.8. Нам приходится отказываться от современных и быстрых библиотек.
Это в конечном сказывается и на пользователях, которые вынуждены пользоваться устаревшим программным обеспечением.

В таких сценариях разделение приложения на несколько процессов с использованием Named Pipes представляется превосходным решением благодаря своей производительности и надежности.
В этой статье мы рассмотрим, как создать и использовать Named Pipes для взаимодействия между приложением Revit, работающим на .NET 4.8 и его плагина, работающим на .NET 7.

# Введение в использование Named Pipes для общения между приложениями на разных версиях .NET

В мире разработки приложений часто требуется обеспечить обмен данными между разными приложениями, особенно в случаях, когда они работают на разных версиях .NET или разных языках.
Разделение одного приложения на несколько процессов должно быть обоснованным. Что проще, вызвать функцию напрямую, или обменяться сообщениями? Очевидно первое.

Тогда какие преимущества в том чтобы это делать?

- Решение конфликта зависимостей

  С каждым годом размер плагинов для Revit все больше и больше растет, а зависимости растут в геометрической прогрессии.
  Плагины могут использовать несовместимые версии одной библиотеки, что вызовет краш программы. Изоляция процессов решает эту проблему.

- Производительность

    Ниже приведены замеры производительности сортировки и математических вычислений на разных версиях .NET

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

  Разница в 68 раз в скорости при нахождении минимального значение, и полное отсутствие выделения памяти, впечатляет.

Тогда как написать программу на последней версии .NET, которая будет взаимодействовать с несовместимым .NET framework?
Создать два приложения, Server и Client, не добавляя зависимостей между друг другом и настроить взаимодействие между ними по настроенному протоколу.

Ниже приведены некоторые из возможных вариантов взаимодействия двух приложений:

1. Использование WCF (Windows Communication Foundation)
2. Использование сокетов (TCP или UDP)
3. Использование Named Pipes
4. Использование сигналов операционной системы (например, сигналов Windows):

   Пример кода компании Autodesk, взаимодействие плагина Project Browser с бекендом Revit посредством сообщений

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

У каждого варианта есть свои достоинства и недостатки, самым удобным на мой взгляд, для взаимодействия на одной локальной машине является Named Pipes. Его мы и рассмотрим.

# Что такое Named Pipes?

Named Pipes представляют собой механизм межпроцессного взаимодействия (Inter-Process Communication, IPC), который позволяет процессам обмениваться данными через именованные каналы.
Они обеспечивают однонаправленное или двунаправленное соединение между процессами.
Помимо высокой производительности, Named Pipes также предлагают различные уровни безопасности, что делает их привлекательным решением для многих сценариев взаимодействия между процессами.

# Взаимодействие между приложениями на .NET 4.8 и .NET 7

Рассмотрим два приложения, одно из которых содержит бизнес-логику (сервер), а другое - пользовательский интерфейс (клиент).
Для обеспечения связи между этими двумя процессами используется NamedPipe.

Принцип работы NamedPipe включает в себя следующие шаги:

1. **Создание и конфигурирование NamedPipe**: Сервер создает и конфигурирует
   NamedPipe с определенным именем, которое будет доступно клиенту. Клиенту необходимо
   знать это имя, чтобы подключиться к трубе.
2. **Ожидание подключения**: Сервер начинает ожидать подключения клиента к трубе.
   Это блокирующая операция, и сервер остается в подвешенном состоянии до тех пор, пока клиент не подключится.
3. **Подключение к NamedPipe**: Клиент инициирует подключение к NamedPipe, указывая имя трубы, к которой он хочет подключиться.
4. **Обмен данными**: После успешного соединения клиент и сервер могут обмениваться
   данными в виде байтовых потоков. Клиент отправляет запросы на выполнение бизнес-логики, а сервер обрабатывает эти запросы и отсылает результаты.
5. **Завершение сеанса**: После завершения обмена данными клиент и сервер могут закрыть соединение с NamedPipe.

## Создание Сервера

На платформе .NET серверная часть представлена классом `NamedPipeServerStream`. Реализация класса предоставляет асинхронные и синхронные методы для работы с NamedPipe.
Во избежание блокировки основного потока, мы будем использовать асинхронные методы.

Пример кода для создания NamedPipeServer:

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

Имя сервера не должно содержать специальные символы во избежание исключения.
Для создания имени трубы мы будем использовать хеш, созданный из имени пользователя и текущей папки, достаточно уникально чтобы клиент при подключении использовал именно этот сервер.
Вы можете изменить это поведение или использовать любое имя в рамках своего проекта, особенно если клиент и сервер находятся в разных директориях.

Данный подход используется в [Roslyn .NET compiler](https://github.com/dotnet/roslyn). Для тех кто сильнее хочет углубиться в эту тему, рекомендую изучить исходный код проекта

`PipeDirection` указывает направления канала, `PipeDirection.In` говорит о том что сервер будет только принимать сообщения, а `PipeDirection.InOut` сможет как принимать, так и отправлять их.

## Создание клиента

Для создания клиента воспользуемся классом NamedPipeClientStream. Код практически аналогичен с сервером, и может немного отличаться в зависимости от версий .NET.
Например, в .NET framework 4.8 значения `PipeOptions.CurrentUserOnly` нет, но появилось в .NET 7.

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

## Протокол передачи

NamedPipe представляет собой Stream, что позволяет нам записывать любую последовательность байтов в поток.
Однако, работать с байтами напрямую может быть не очень удобно, особенно когда мы имеем дело со сложными данными или структурами.
Для упрощения взаимодействия с потоками данных и структурирования информации в удобном формате используются протоколы передачи.

Протоколы передачи определяют формат и порядок передачи данных между приложениями.
Они обеспечивают структурирование информации, чтобы обеспечить понимание и правильную интерпретацию данных между отправителем и получателем.

В случая когда нам нужно отправить "Запрос на выполнение определенной команды на сервере" или "Запрос на обновление настроек приложения",
сервер должен понимать как его обрабатывать от клиента.
Поэтому для облегчения обработки запросов и управлением обмена данными, создадим Enum `RequestType`.

```C#
public enum RequestType
{
    PrintMessage,
    UpdateModel
}
```

Сам заброс будет представлять класс, который будет содержать всю информацию о передаваемых данных.

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

Класс содержит базовый код для записи данных в поток. `AddRequestBody()` используется производными классами, для записи для записи собственных структурированных данных.

Примеры производных классов:

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

Используя данную структуру, клиенты могут создавать запросы различных типов, каждый из которых определяет собственную логику обработки данных и параметров.
Классы `PrintMessageRequest` и `UpdateModelRequest` предоставляют примеры запросов, которые можно отправить серверу для выполнения конкретных задач.

На стороне сервера, необходимо разработать соответствующую логику обработки входящих запросов.
Для этого сервер должен читать данные из потока и использовать полученные параметры для выполнения нужных операций.

Пример полученного запроса на стороне сервера:

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

Метод `ReadAsync()` считывает тип запроса из потока, а затем, в зависимости от типа, считывает соответствующие данные и создает объект соответствующего запроса.

Реализация протокола передачи данных и структурирование запросов в виде классов позволяют эффективно управлять обменом информацией между клиентом и сервером, обеспечивая при этом
структурированное и понятное взаимодействие между двумя сторонами.
Однако, при проектировании подобных протоколов необходимо учитывать возможные риски безопасности, а также убедиться, что оба конца взаимодействия правильно обрабатывают все возможные случаи.

## Управление соединениями

Для отправки сообщений с UI клиента на сервер, создадим класс ClientDispatcher который будет обеспечивать обработку соединений,
тайм-аутов и планирование запросов, предоставляя интерфейс для взаимодействия клиента с сервером через именованные трубы.

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

Принцип работы:

1. **Инициализация:** В конструкторе класса инициализируется `NamedPipeClientStream`, используемый для создания клиентского потока с именованным каналом.
2. **Установка подключения:** Метод `ConnectToServer` инициирует асинхронное подключение к серверу. Результат операции сохраняется в `Task`.
   `TimeOutNewProcess` используется для отключения клиента в случае возникновения непредвиденных исключений.
3. **Отправка запросов:** Метод `WriteRequestAsync` предназначен для асинхронной отправки объекта Request через установленное соединение. Запрос отправится только после установки соединения.

Для приема сообщений сервером, создадим класс ServerDispatcher который управлять соединением и читать запросы.

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

Принцип работы:

1. **Инициализация:** В конструкторе класса инициализируется `NamedPipeServerStream`, используемый для создания серверного потока с именованным каналом.
2. **Прослушивание подключений:** Метод `ListenAndDispatchConnections` асинхронного ожидает подключения клиента, после завершения обработки запросов закрывает именованный канал и освобождает
   ресурсы.
3. **Обработка запросов:** Метод `ListenAndDispatchConnectionsCoreAsync` обрабатывает запросы, до момента отключения клиента.
   В зависимости от типа запроса происходит соответствующая обработка данных, например, вывод в консоль содержания сообщения или обновление модели.

Пример отправки запроса из UI на сервер:

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

Пример кода полностью доступен в репозитории, вы можете запустить его на своей машине выполнив несколько шагов:

- Запустите "Build Solution"
- Запустите "Run OneWay\Backend"

Приложение автоматически запустит Server и Client, а полный вывод сообщений передающихся по NamedPipe вы увидите в консоли IDE.

## Двусторонняя передача

Часто возникают ситуации, когда обычная однонаправленная передача данных от клиента к серверу недостаточна.
В таких случаях необходимо обрабатывать ошибки или отправлять результаты в ответ. Чтобы обеспечить более сложное взаимодействие между клиентом и сервером, разработчикам приходится прибегать
к применению двухсторонней передачи данных, которая позволяет обмениваться информацией в обоих направлениях.

Как и в случае с запросами, для эффективной обработки ответов также необходимо определить перечисление для типов ответов.
Это позволит клиенту правильно интерпретировать полученные данные.

```C#
public enum ResponseType
{
    // The update request completed on the server and the results are contained in the message. 
    UpdateCompleted,
    
    // The request was rejected by the server.
    Rejected
}
```

Для эффективной обработки ответов потребуется создать новый класс, названный Response.
По функционалу он ничем не отличается от класса Request, однако в отличие от Request, который может читаться на сервере, Response будет записываться в поток.

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

Производные классы вы можете найти в репозитории проекта: [PipeProtocol](https://github.com/Nice3point/InterprocessCommunication/blob/main/TwoWay/Backend/Server/PipeProtocol.cs)

Для того чтобы сервер мог отправлять ответы клиенту, мы должны модифицировать класс `ServerDispatcher`.
Это позволит записывать ответы в Stream после выполнения задачи.

Так же изменим направление трубы на двунаправленное:

```C#
_server = NamedPipeUtil.CreateServer(PipeDirection.InOut);

/// <summary>
///     Write a Response to the client.
/// </summary>
public async Task WriteResponseAsync(Response response) => await response.WriteAsync(_server);
```

Для демонстрации работы добавим задержку на 2 секунды, эмулируя тяжелую задачу, в методе ListenAndDispatchConnectionsCoreAsync:

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

В настоящий момент клиент не обрабатывает ответы от сервера. 
Давайте сделаем это. 
Создадим в клиенте класс Response, который будет обрабатывать полученные ответы.

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

Далее обновим класс ClientDispatcher, чтобы он мог обрабатывать ответы от сервера. Для этого добавим новый метод и изменим направление на двунаправленное.

```C#
_client = NamedPipeUtil.CreateClient(PipeDirection.InOut);

/// <summary>
///     Read a Response from the server.
/// </summary>
public async Task<Response> ReadResponseAsync() => await Response.ReadAsync(_client);
```

Также добавим обработку ответа во ViewModel, где будем просто выводить его как сообщение.

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

Эти изменения позволят более эффективно организовать взаимодействие между клиентом и сервером, обеспечивая более полную и надежную обработку запросов и ответов.

## Реализация плагина для Revit

<p align="center">
  <img src="https://github.com/Nice3point/InterprocessCommunication/assets/20504884/09e0dee3-d4bd-4858-87eb-6bf6766b8dde" />
</p>

<p align="center">Технологии развиваются, а Revit не меняется © Конфуций</p>

В настоящее время Revit использует .NET Framework 4.8. 
Однако для улучшения пользовательского интерфейса плагинов, рассмотрим переход на .NET 7. 
Важно отметить, что бэкэнд плагина будет взаимодействовать только с Revit на устаревшем Framework, и будет выступать в качестве сервера.

Давайте создадим механизм взаимодействия, который позволит клиенту отправлять запросы на удаление элементов модели, а затем получать ответы о результате удаления. 
Для реализации этой функциональности мы будем использовать двустороннюю передачу данных между сервером и клиентом.

Первым шагом в нашем процессе разработки будет научить плагин автоматически закрываться при закрытии Revit. 
Для этого мы написали метод, который отправляет ID текущего процесса клиенту. 
Это поможет клиенту осуществить автоматическое закрытие своего процесса при закрытии родительского процесса Revit.

Код для отправки ID текущего процесса клиенту:

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

А вот код для клиента, который осуществляет закрытие своего процесса при закрытии родительского процесса Revit:

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

Кроме того, нам необходим метод, который будет отвечать за удаление выбранных элементов модели:

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

Также обновим метод ListenAndDispatchConnectionsCoreAsync для обработки входящих соединений:

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

И, наконец, обновленный код ViewModel:

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

# Установка .NET Runtime во время установки

Не у каждого пользователя может быть установлена последняя версия .NET Runtime на локальной машине, нам необходимо внести изменения в установщик плагина.

Если вы используете шаблоны [Nice3point.RevitTemplates](https://github.com/Nice3point/RevitTemplates), то внести изменения не составит труда.
В шаблонах используется библиотека WixSharp, которая позволяет создавать .msi файлы прямо на C#.

Для добавления пользовательских действий, и установки .NET Runtime создадим `CustomAction`

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

Этот код проверяет, установлена ли требуемая версия .NET на локальной машине, и если нет, то скачивает и устанавливает ее.
Во время установки обновляется Status текущего хода выполнения скачивания и распаковки Runtime.

Осталось подключить CustomAction в проект WixSharp, для этого инициализируем свойство `Actions`:

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

# Заключение

В данной статье мы рассмотрели как Named Pipes, преимущественно используемые для взаимодействия между разными процессами, могут быть использованы в сценариях, где требуется обмен данными между приложениями на разных версиях .NET. 
Имея дело с кодом, который необходимо поддерживать в нескольких версиях, выверенная стратегия межпроцессного взаимодействия (Inter-Process Communication, IPC) может быть полезной и обеспечивать ключевые преимущества, такие как:

- Решение конфликтов зависимостей
- Улучшение производительности
- Функциональная гибкость

Мы обсудили процесс создания сервера и клиента, которые взаимодействуют друг с другом через заранее определенный протокол, а также различные способы управления соединениями.

Рассмотрели пример ответов сервера и демонстрацию работы обеих сторон взаимодействия.

Наконец, мы подчеркнули как Named Pipes используются в разработке плагина для Revit для обеспечения взаимодействия между бекендом, работающим на устаревшей платформе .NET 4.8, и пользовательским интерфейсом, работающим на более новой версии .NET 7.

Демонстрационный код для каждой части этой статьи доступен на GitHub.

В определенных случаях разделение приложений на отдельные процессы может не только уменьшить зависимости в программе, но и ускорить его выполнение.
Но давайте не забывать, что выбор подхода требует анализа и должен основываться на реальных требованиях и ограничениях вашего проекта.

Мы надеемся, что эта статья поможет вам найти оптимальное решение для ваших сценариев межпроцессного взаимодействия и даст понимание, как применять подходы IPC на практике.