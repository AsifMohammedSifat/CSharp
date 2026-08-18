# TCP দিয়ে HTTP Server বানানো — .NET, Kestrel, TCP, HTTP Fundamentals

# Program.cs
```csharp

  var server = new TcpServer(5000);
  await server.StartAsync();

```

# TcpServer.cs
```csharp

  using System.Net;
  using System.Net.Sockets;
  using System.Text;
  
  public class RequestContext
  {
      public string Method { get; set; } = string.Empty;
      public string Path { get; set; } = string.Empty;
  }
  
  public class TcpServer
  {
      private readonly int _port;
      public TcpServer(int port)
      {
          _port = port;
      }
  
      public async Task StartAsync()
      {
          var listener = new TcpListener(IPAddress.Any, _port); // connection or socket open korci
          listener.Start(); // server listening শুরু করলাম।
  
          while (true)
          {
              var client = await listener.AcceptTcpClientAsync(); // socket e ekjon connect hoite asche. then tar sathe conneciton establish korchi
              _ = Task.Run(() => HandleClient(client)); // multi thread e kaz hobe
              // await HandleClient(client); // only ekta request niye kaz korbe.
          }
  
      }
  
      private async Task HandleClient(TcpClient client)
      {
          using var stream = client.GetStream();
          var buffer = new byte[1024];
          var byteCount = await stream.ReadAsync(buffer); // stream theke ei buffer e value load koro
          var requestText = Encoding.UTF8.GetString(buffer, 0, byteCount);
  
          var lines = requestText.Split("\r\n");
          var requestLine = lines[0].Split(' ');
  
          var context = new RequestContext
          {
              Method = requestLine[0],
              Path = requestLine[1]
          };
  
  
          // Build and send a simple response
          var responseText = $"You requested {context.Path}";
          var responseBytes = Encoding.UTF8.GetBytes(
              "HTTP/1.1 200 OK\r\n" +
              "Content-Length: " + responseText.Length + "\r\n\r\n" +
              responseText
          );
  
          await stream.WriteAsync(responseBytes);
          client.Close();
      }
  }
  
  
  // GET / HTTP/1.1\r\n
  // Host: localhost:5000\r\n
  // Connection: keep-alive\r\n
  // Cache-Control: max-age=0\r\n
  // sec-ch-ua: "Not=A?Brand";v="99", "Google Chrome";v="151", "Chromium";v="151"\r\n
  // sec-ch-ua-mobile: ?0\r\n
  // sec-ch-ua-platform: "Windows"\r\n
  // Upgrade-Insecure-Requests: 1\r\n
  // User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36\r\n
  // Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7\r\n
  // Sec-Fetch-Site: cross-site\r\n
  // Sec-Fetch-Mode: navigate\r\n
  // Sec-Fetch-User: ?1\r\n
  // Sec-Fetch-Dest: document\r\n
  // Accept-Encoding: gzip, deflate, br, zstd\r\n
  // Accept-Language: en-BD,en-GB;q=0.9,en-US;q=0.8,en;q=0.7,bn;q=0.6,zh-TW;q=0.5,zh;q=0.4\r\n
  // \r\n
```
---

## 1. .NET কীভাবে তৈরি হলো — সংক্ষিপ্ত ইতিহাস

Microsoft চেয়েছিল এমন একটা platform, যেখানে Windows app, web app, database app, network app, enterprise app — সব ধরনের application বানানো যাবে। এর জন্যই **.NET Framework** তৈরি হয়।

```text
2002  → .NET Framework
        - Windows-only
        - বেশিরভাগ closed-source

2014-16 → .NET Core
        - সম্পূর্ণ rewrite
        - Cross-platform (Windows/Linux/macOS)
        - Open-source, lightweight, দ্রুত

2020  → .NET 5
        - .NET Framework ও .NET Core একীভূত (unified)
        - নাম থেকে "Core" বাদ
        - এরপর থেকে প্রতি বছর নতুন major version

.NET 5 → 6 → 7 → 8 → 9 → 10 ...
```

**মূল কারণ:** পুরনো .NET Framework শুধু Windows-এ চলত, Linux server বা Docker container-এ deploy করা যেত না। Cloud (Azure/AWS) আর cross-platform deployment-এর চাহিদা থেকেই .NET Core আসে।

### C# আর .NET কি একই জিনিস?

না।

```text
.NET = Platform / Runtime / Libraries / Infrastructure
C#   = Programming Language (এই platform-এর উপর লেখা একটা ভাষা)
```

`Console.WriteLine("Hello")` — এটা C# code, কিন্তু এটা চালানোর জন্য .NET runtime ও libraries দরকার হয়।

---

## 2. Kestrel কী?

**Kestrel** হলো ASP.NET Core-এর নিজস্ব built-in **cross-platform HTTP web server**, C# দিয়েই লেখা।

- Kestrel-ই সেই component যেটা socket খুলে, TCP connection accept করে, raw bytes থেকে HTTP request parse করে — ঠিক যে কাজটা তুমি `TcpServer.cs`-এ হাতে করার চেষ্টা করছো, প্রোডাকশনে Kestrel সেটাই করে, শুধু অনেক বেশি robust ও optimized ভাবে (buffering, connection management, HTTP/1.1-HTTP/2-HTTP/3 support ইত্যাদিসহ)।
- Kestrel-এর নিচের স্তরে OS-এর networking API ও native socket capability ব্যবহৃত হয়।
- Production-এ সাধারণত Kestrel-এর সামনে একটা reverse proxy (Nginx, IIS, Apache) থাকে — TLS termination, load balancing ইত্যাদির জন্য। Kestrel নিজে সরাসরি internet-facing হতে পারলেও, বেশিরভাগ setup-এ এটা reverse proxy-এর পেছনে থাকে।

> সংক্ষেপে: **তুমি এখন যা বানাচ্ছো, সেটা Kestrel-এর একটা mini, শেখার-উদ্দেশ্যে বানানো ভার্সন।**

```text
Browser
   ↓
Operating System
   ↓
TCP/IP Networking
   ↓
Kestrel
   ↓
ASP.NET Core
   ↓
Middleware
   ↓
Endpoint
```

---

## 3. Operating System কীভাবে request receive করে?

Browser সরাসরি তোমার C# application-এর সাথে কথা বলে না — মাঝে পুরো একটা layer stack থাকে।

```text
┌─────────────────────────────┐
│  Application (তোমার C# কোড)   │  ← TcpListener / socket API ব্যবহার করছে
├─────────────────────────────┤
│  Operating System (Kernel)    │  ← TCP/IP network stack ম্যানেজ করে
├─────────────────────────────┤
│  Network Card / Driver        │  ← raw bytes আসছে/যাচ্ছে
├─────────────────────────────┤
│  Network Cable / WiFi         │
└─────────────────────────────┘
```

- Network card-এ raw signal হিসেবে data আসে।
- OS-এর ভেতরের **TCP/IP stack** সেই bytes থেকে packet বের করে, IP address ও port দেখে বোঝে এটা কোন application-এর জন্য।
- OS সেই data একটা **socket**-এর মাধ্যমে application-কে পৌঁছে দেয়। `TcpListener`/`Socket` ক্লাসগুলো আসলে OS-এর socket system call-এর wrapper মাত্র।
- যদি request `localhost`-এ যায়, physical network card-এর বদলে **loopback interface** ব্যবহার হয়।

```text
Chrome → HTTP → TCP → IP → Network Card/Loopback → OS → Port 5000 → তোমার App
```

---

## 4. Port, Socket এবং Protocol

- **Port** একটা সংখ্যা (0–65535), যা দিয়ে বোঝানো হয় একটা machine-এর কোন নির্দিষ্ট application/service-এর কথা বলা হচ্ছে (80 = HTTP, 443 = HTTPS, 5000 = তোমার dev server, 3306 = MySQL)।

  ```text
  IP Address = Building address
  Port       = Room number
  ```

  IP বলে *কোন machine-এর সাথে যোগাযোগ*, Port বলে *সেই machine-এর কোন service-এর সাথে যোগাযোগ*।

- Socket তৈরির সময় বলে দিতে হয় এটা কোন **transport protocol** ব্যবহার করবে:
  - **TCP** — connection-oriented, reliable (HTTP এটাই ব্যবহার করে)
  - **UDP** — connectionless, দ্রুত কিন্তু unreliable (streaming, gaming, DNS)

- **HTTP কোনো transport protocol না** — এটা transport-এর উপরে চলা একটা **application-layer protocol**।

  ```text
  Application Layer → HTTP
  Transport Layer   → TCP
  Internet Layer    → IP
  ```

  ক্রম এভাবে হয়:

  ```text
  প্রথমে TCP connection তৈরি হয়
          ↓
  তারপর সেই connection-এর ভেতর দিয়ে HTTP request/response text পাঠানো হয়
  ```

### Bind কী?

`IP Address + Port`-এর সাথে server socket-কে associate করাকে **bind** বলে। যেমন `0.0.0.0:5000` মানে — এই machine-এর সব available network interface-এ port 5000-এ connection গ্রহণ করব।

```csharp
var listener = new TcpListener(IPAddress.Any, _port);
listener.Start();
```

`IPAddress.Any` → সব interface-এ listen করার জন্য প্রস্তুত। `Start()` → listener-কে **listening state**-এ নিয়ে যায় (bind + listen), যদিও এখনো কোনো connection accept হয়নি।

---

## 5. TCP কী?

**TCP (Transmission Control Protocol)** — transport-layer (OSI Layer 4) প্রোটোকল, যা দুইটা machine-এর মধ্যে **connection-oriented, reliable, ordered byte-stream** communication দেয়।

মূল দায়িত্বগুলো:

1. **Connection establish করা** — data পাঠানোর আগে দুই পক্ষের মধ্যে handshake-এর মাধ্যমে connection তৈরি করা।
2. **Reliable delivery** — packet হারালে retransmission-এর মাধ্যমে recovery।
3. **Ordering** — যে ক্রমে পাঠানো হয়েছে, ঠিক সেই ক্রমেই receiver byte stream হিসেবে পায় (মাঝে এলোমেলো হয়ে এলেও TCP সাজিয়ে দেয়)।
4. **Flow control** — receiver কতটা handle করতে পারছে সেটা বিবেচনা করে transmission নিয়ন্ত্রণ।
5. **Congestion control** — network congestion হলে transmission rate কমানো।
6. **Error checking** — প্রতিটা packet-এ checksum থাকে, corruption ধরার জন্য।

### TCP কি HTTP request বোঝে?

**না।** TCP জানে না `GET / HTTP/1.1`-এর মানে কী — TCP শুধু byte stream transport করে। HTTP বোঝা HTTP layer-এর কাজ।

```text
HTTP text → bytes → TCP → Network → TCP → bytes → HTTP parser
```

---

## 6. TCP 3-Way Handshake

Actual data পাঠানোর আগে দুই machine-এর মধ্যে এই "পরিচয়/সম্মতি" পর্বটা হয় — একে **3-way handshake** বলে, কারণ এতে তিনটা ধাপ/packet exchange থাকে।

```text
Client                          Server
  │                                │
  │──────── 1) SYN ──────────────▶│   "আমি connect করতে চাই, আমার sequence number এটা"
  │                                │
  │◀─────── 2) SYN-ACK ───────────│   "ঠিক আছে, পেয়েছি; আমিও রাজি, আমার sequence number এটা"
  │                                │
  │──────── 3) ACK ───────────────▶│   "তোমার response পেয়েছি, শুরু করা যাক"
  │                                │
  │◀════ TCP Connection Established ════▶│
  │                                │
  │◀──── এখন দুই দিকেই data যাওয়া-আসা শুরু (duplex) ────▶│
```

> তোমার কোডে `listener.AcceptTcpClientAsync()` কল করার সময় .NET/OS ভেতরে ভেতরে পুরো handshake সামলে দেয় — manually SYN/ACK নিয়ে কিছু লিখতে হয় না। Handshake শেষ হলেই এটা একটা connected `TcpClient` return করে।

Connection establish হওয়ার পর দুই দিকে data flow করে:

```text
Client                         Server
   │────── data ───────────────▶│
   │◀────── data ───────────────│
```

---

## 7. RFC কী?

**RFC (Request For Comments)** হলো internet-এর কোনো technology/protocol-এর **আনুষ্ঠানিক specification document**, প্রকাশ করে **IETF (Internet Engineering Task Force)**।

- HTTP/1.1-এর স্পেসিফিকেশন → RFC 9110–9112
- TCP-এর স্পেসিফিকেশন → RFC 793 (ও পরবর্তী updates)

নাম "শুধু মন্তব্য" মনে হলেও ঐতিহাসিক কারণে এমন — বাস্তবে এগুলোই সেই official rulebook, যা মেনে সব browser/server/network device একে অপরের সাথে সঠিকভাবে communicate করে।

---

## 8. HTTP কী?

**HTTP (HyperText Transfer Protocol)** একটা application-layer প্রোটোকল, যা বলে দেয় client কীভাবে request পাঠাবে আর server কীভাবে response দেবে।

```text
TCP  বলে → bytes reliably transport করার ব্যবস্থা কীভাবে হবে
HTTP বলে → সেই bytes-এর মধ্যে request/response-এর structure কী হবে
```

### Raw HTTP request-এর গঠন

```text
┌─────────────────────────────────────────┐
│ Request Line: METHOD  PATH  VERSION       │  → GET / HTTP/1.1
├─────────────────────────────────────────┤
│ Headers (key: value, একটার পর একটা)         │  → Host: ..., Connection: ...
├─────────────────────────────────────────┤
│ blank line (headers শেষ বোঝাতে)             │  → \r\n\r\n
├─────────────────────────────────────────┤
│ Body (optional — GET-এ সাধারণত থাকে না)     │  → POST/PUT-এ JSON/form data
└─────────────────────────────────────────┘
```

উদাহরণ:

```text
GET / HTTP/1.1\r\n
Host: localhost:5000\r\n
Connection: keep-alive\r\n
User-Agent: Chrome...\r\n
Accept: text/html...\r\n
\r\n
```

**Request Line**-এর তিনটা অংশ (স্পেস দিয়ে আলাদা):

| অংশ | মানে | উদাহরণ |
|---|---|---|
| Method | কী ধরনের action চাওয়া হচ্ছে | `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| Path | কোন resource-এর জন্য request | `/`, `/products/10` |
| Version | কোন HTTP version | `HTTP/1.1` |

**Headers** browser/client সম্পর্কে extra metadata দেয় (কী browser, কোন language পছন্দ, connection keep রাখতে চায় কিনা ইত্যাদি) — browser নিজে থেকেই এগুলোর বেশিরভাগ পাঠায়।

### `\r\n` আসলে কী?

- `\r` = **Carriage Return (CR)** — ASCII 13
- `\n` = **Line Feed (LF)** — ASCII 10
- `\r\n` = **CRLF**, পুরনো teletype/typewriter terminology থেকে আসা নাম

HTTP/1.x স্পেসিফিকেশন অনুযায়ী প্রতিটা লাইনের শেষে **CRLF** ব্যবহার করতে হয় — শুধু `\n` (যেমন Linux ফাইলে হয়) যথেষ্ট নয়, HTTP নির্দিষ্টভাবে `\r\n` চায়।

শেষের একটা আলাদা `\r\n` (অর্থাৎ একটা খালি লাইন) headers শেষ হওয়ার সংকেত দেয়:

```text
GET / HTTP/1.1
Host: localhost:5000
Connection: keep-alive
User-Agent: Chrome...

<empty line>  ← headers শেষ
```

---

## 9. তোমার Code-এর Flow — লাইন ধরে ধরে

### `TcpServer.StartAsync()`

```csharp
var listener = new TcpListener(IPAddress.Any, _port);
listener.Start();

while (true)
{
    var client = await listener.AcceptTcpClientAsync();
    _ = Task.Run(() => HandleClient(client));
}
```

- `AcceptTcpClientAsync()` — নতুন client connect হওয়ার জন্য অপেক্ষা করে; এর ভেতরেই 3-way handshake সম্পন্ন হয়, শেষে একটা connected `TcpClient` রিটার্ন করে।
- `_ = Task.Run(() => HandleClient(client))` — accepted client-এর handling background execution-এ পাঠিয়ে দেওয়া হচ্ছে, যাতে একজন client process করার সময় main loop নতুন client accept করতে block না হয়:

  ```text
  Client A → Accept → HandleClient(A)     (background)
  Client B → Accept → HandleClient(B)     (একই সময়ে, background)
  ```

  ⚠️ তবে **`Task.Run()` মানেই প্রতিটা request-এর জন্য নতুন thread তৈরি হয় না** — .NET ThreadPool ব্যবহার করে task execute করে, এটা oversimplified ধারণা এড়িয়ে চলা ভালো।

  ⚠️ `_ =` দিয়ে fire-and-forget করা হচ্ছে — `HandleClient`-এ কোনো unhandled exception হলে সেটা silently swallow হতে পারে বা crash করতে পারে (.NET version ভেদে আচরণ ভিন্ন)। প্রোডাকশন-grade কোডে `HandleClient`-এর ভেতরে `try/catch` রাখা ভালো অভ্যাস।

  (যদি এই লাইনের জায়গায় `await HandleClient(client);` লেখা হতো, তাহলে এক client পুরোপুরি শেষ না হওয়া পর্যন্ত পরের client accept হতো না — অর্থাৎ concurrent handling-এর সুবিধা চলে যেত।)

### `HandleClient()`

```csharp
using var stream = client.GetStream();
var buffer = new byte[1024];
var byteCount = await stream.ReadAsync(buffer);
var requestText = Encoding.UTF8.GetString(buffer, 0, byteCount);
```

- `client.GetStream()` — established TCP connection-এর byte stream, যেখান থেকে read/write করা যায়।
- `buffer = new byte[1024]` — ১KB fixed buffer।
  - ⚠️ **সম্ভাব্য সমস্যা:** বাস্তব HTTP request headers অনেক সময় ১KB-এর বেশি হতে পারে (অনেক cookies, বড় User-Agent ইত্যাদি সহ)। তখন `ReadAsync` শুধু প্রথম ১KB পড়বে, বাকিটা বাদ পড়ে যাবে — parsing ভুল হতে পারে। শেখার প্রজেক্টে এটা মেনে নেওয়া যায়, কিন্তু ভবিষ্যতে `\r\n\r\n` না পাওয়া পর্যন্ত loop করে read চালিয়ে যাওয়া উচিত।
- `stream.ReadAsync(buffer)` — TCP connection-এর receive stream থেকে available bytes পড়ে buffer-এ রাখে; `byteCount` বলে কতগুলো byte আসলে পড়া হলো (buffer size 1024 হলেও actual received হতে পারে যেমন 450)।
- `Encoding.UTF8.GetString(...)` — raw bytes-কে readable string-এ decode করে।

> **গুরুত্বপূর্ণ concept:** TCP-তে "request"-এর কোনো নিজস্ব boundary নেই — এটা শুধু byte stream দেয়, এটা জানে না একটা সম্পূর্ণ HTTP request কোথায় শেষ। তাই `ReadAsync(buffer)`-এর একটা কলেই পুরো HTTP request চলে আসবে — এটা ধরে নেওয়া সবসময় safe নয়। Production-grade server-এ (যেমন Kestrel-এ) আরও sophisticated buffering/parsing logic থাকে।

```csharp
var lines = requestText.Split("\r\n");
var requestLine = lines[0].Split(' ');

var context = new RequestContext
{
    Method = requestLine[0],
    Path = requestLine[1]
};
```

- `Split("\r\n")` পুরো raw text-কে লাইন-বাই-লাইন ভাগ করছে (request line + প্রতিটা header)।
- `GET / HTTP/1.1` → `["GET", "/", "HTTP/1.1"]` → `Method = "GET"`, `Path = "/"`।
- ⚠️ **সম্ভাব্য সমস্যা:** malformed বা খালি request এলে (যেমন কোনো port-scanner শুধু connect করে বন্ধ করে দিলো), `lines[0]` খালি হতে পারে আর `requestLine[1]` অ্যাক্সেস করতে গেলে `IndexOutOfRangeException` হতে পারে — validation/try-catch দরকার।
- `Version` (`HTTP/1.1`) এখনো parse হচ্ছে না — চাইলে `RequestContext`-এ যোগ করা যায়।

```csharp
var responseText = $"You requested {context.Path}";
var responseBytes = Encoding.UTF8.GetBytes(
    "HTTP/1.1 200 OK\r\n" +
    "Content-Length: " + responseText.Length + "\r\n\r\n" +
    responseText
);

await stream.WriteAsync(responseBytes);
client.Close();
```

- একটা সঠিক গঠনের minimal HTTP response: status line (`HTTP/1.1 200 OK`) → header (`Content-Length`) → blank line → body।
- `client.Close()` response পাঠানোর পর connection বন্ধ করে দেয়। Browser `Connection: keep-alive` চেয়েছিল, কিন্তু server বন্ধ করে দিচ্ছে — কার্যকরীভাবে ঠিক আছে (browser নতুন request-এ নতুন connection বানাবে), তবে explicitly `Connection: close` header যোগ করলে এটা আরও "সঠিক" ও স্পষ্ট হয়।

### `Program.cs`

```csharp
var server = new TcpServer(5000);
await server.StartAsync();
```

Port 5000-এ server চালু হয়, `StartAsync()`-এর ভেতরের `while (true)` loop অ্যাপ্লিকেশন চলা পর্যন্ত সবসময় request শুনতে থাকে।

---

## 10. পুরো Request → Response Flow — একনজরে

```text
Browser: http://localhost:5000 এ request পাঠায়
        │
        ▼
TCP 3-Way Handshake (OS-এর মাধ্যমে)
        │
        ▼
listener.AcceptTcpClientAsync() → connected TcpClient
        │
        ▼
Task.Run(() => HandleClient(client))  → আলাদা background task
        │
        ▼
stream.ReadAsync(buffer) → raw bytes পড়া হয়
        │
        ▼
UTF8 decode → string
        │
        ▼
Split("\r\n") → লাইন আলাদা
        │
        ▼
Request Line parse → Method, Path
        │
        ▼
RequestContext তৈরি
        │
        ▼
Response string বানানো (status line + headers + body)
        │
        ▼
UTF8 bytes → stream.WriteAsync()
        │
        ▼
TCP → OS → Browser
        │
        ▼
client.Close()
```

---

## 11. TCP বনাম HTTP — সারাংশ

| বিষয় | TCP | HTTP |
|---|---|---|
| Layer | Transport | Application |
| কাজ | Reliable byte transport | Request/response structure |
| জানে `GET` কী? | ❌ | ✅ |
| জানে `/users` কী? | ❌ | ✅ |
| 3-way handshake | ✅ | ❌ |
| Port ব্যবহার | ✅ সরাসরি | ✅ TCP-এর মাধ্যমে |
| Header | ❌ | ✅ |
| `GET / HTTP/1.1` বোঝে | ❌ | ✅ |

> **ভুল ধারণা:** Port open করলে কী ধরনের request receive করব সেটা TCP-কে বলে দিতে হয়।
> **সঠিক ধারণা:** Port-এ তুমি TCP connection accept করো; তারপর সেই connection-এর byte stream-এর মধ্যে client যদি HTTP protocol অনুযায়ী data পাঠায়, তাহলে তোমার application সেই bytes-কে HTTP হিসেবে parse করতে পারে।

```text
Port → TCP → Bytes → HTTP Parser → HTTP Request
```

---

## 12. Learning Roadmap

```text
1. Operating System
2. Socket
3. IP Address + Port
4. TCP
5. TCP 3-Way Handshake
6. TcpListener
7. TcpClient
8. NetworkStream
9. Bytes
10. HTTP Protocol
11. HTTP Request
12. HTTP Request Parsing
13. HTTP Response
14. HTTP Response Writing
15. RequestContext
16. Middleware
17. Routing
18. ASP.NET Core
19. Kestrel
```

---

## 13. পরবর্তী ধাপ (Next Steps)

তোমার "port open + bind, HTTP receive + parse" অংশ ইতিমধ্যে হয়ে গেছে। এরপর যোগ করার মতো:

1. **Headers পার্স করা** — এখন শুধু request line পার্স হচ্ছে; `Host`, `Content-Length`, `Content-Type` ইত্যাদি header-ও `Dictionary<string, string>`-এ পার্স করে `RequestContext`-এ রাখা যায়।
2. **Body পার্স করা** — POST/PUT-এ body থাকে; `Content-Length` header দেখে ততটুকু আরও read করতে হবে (প্রথম `ReadAsync`-এ পুরো body নাও আসতে পারে)।
3. **বড়/loop-based reading** — `\r\n\r\n` না পাওয়া পর্যন্ত read চালিয়ে যাওয়া, কারণ headers ১KB-এর বেশি হতে পারে।
4. **Middleware Pipeline যুক্ত করা** — `RequestContext` বানানোর পর সেটা middleware pipeline-এ পাঠানো যায়, ঠিক যেভাবে Kestrel ASP.NET Core middleware pipeline-কে feed করে।
5. **Error handling** — malformed request, connection reset, empty request ইত্যাদির জন্য try/catch ও validation।


## Extra (Networking Fundamentals)
অবশ্যই। এখানে কয়েকটা concept একটু correction করে note করছি, কারণ **socket, HTTP connection, TCP connection, এবং epoll**—এগুলো একসাথে মিশে গেলে confusion হতে পারে।

# 1. Socket Connection — কী?

সহজভাবে বললে, network programming-এ **socket হলো communication endpoint**।

দুইটি application network-এর মাধ্যমে communicate করতে চাইলে দুই পাশে socket থাকতে পারে।

```text
Client Application
       │
     Socket
       │
       │ Network
       │
     Socket
       │
Server Application
```

আরও practicalভাবে:

```text
IP Address + Port
        ↓
      Socket
        ↓
   TCP Connection
```

উদাহরণ:

```text
Client: 192.168.1.10:50000
Server: 192.168.1.20:5000
```

এখানে এই endpoint-গুলোর মাধ্যমে TCP communication হতে পারে।

### গুরুত্বপূর্ণ correction

"দুনিয়ার যেকোনো connection-ই socket connection" — এভাবে বলা technically ঠিক নয়।

বরং বলা ভালো:

> **Network application-এ socket হলো communication-এর একটি endpoint/interface। TCP connection socket ব্যবহার করে তৈরি করা যায়।**

---

# 2. Socket কি TCP-এর সমান?

না।

```text
Socket ≠ TCP
```

Socket হলো programming interface/endpoint abstraction।

TCP হলো transport protocol।

উদাহরণ:

```text
Application
    ↓
Socket API
    ↓
TCP
    ↓
IP
    ↓
Network
```

C#-এ:

```csharp
TcpListener
TcpClient
Socket
```

এগুলো দিয়ে TCP networking করা যায়।

---

# 3. HTTP Connection কীভাবে কাজ করে?

এখানে তোমার কথাটা **কিছুটা ঠিক, কিন্তু একটু correction দরকার।**

তুমি লিখেছ:

> HTTP-তে connection one way করে দেওয়া হয় এবং TCP + HTTP মিলে socket connection close করে দেয়।

এটা পুরোপুরি ঠিক নয়।

HTTP request/response সাধারণত **client ↔ server দুই দিকেই communication** করে।

```text
Client
   │
   │ HTTP Request
   ▼
Server
   │
   │ HTTP Response
   ▼
Client
```

অর্থাৎ HTTP communication bidirectional হলেও **HTTP/1.0/HTTP/1.1-এর request-response model-এ client request করে, server response দেয়।**

---

# 4. TCP Connection বনাম HTTP Request

এই distinction খুব গুরুত্বপূর্ণ।

একটি TCP connection-এর উপর এক বা একাধিক HTTP request থাকতে পারে।

### HTTP/1.0-এর ক্ষেত্রে

অনেক ক্ষেত্রে:

```text
TCP Connection
      ↓
HTTP Request
      ↓
HTTP Response
      ↓
TCP Connection Close
```

### HTTP/1.1-এর ক্ষেত্রে

Connection reuse করা যায়:

```text
TCP Connection
      ↓
HTTP Request 1
      ↓
HTTP Response 1
      ↓
HTTP Request 2
      ↓
HTTP Response 2
      ↓
HTTP Request 3
      ↓
HTTP Response 3
      ↓
Connection Close
```

এটাকে **persistent connection / keep-alive** বলা হয়।

তোমার browser request-এ:

```text
Connection: keep-alive
```

দেখেছ।

এর অর্থ connection এক request-এর পর অবশ্যই close করতে হবে—এমন নয়।

---

# 5. তোমার Code কেন Connection Close করছে?

তোমার code-এ:

```csharp
await stream.WriteAsync(responseBytes);
client.Close();
```

এখানে তুমি নিজেই TCP connection close করে দিচ্ছ।

তাই তোমার server-এর flow হচ্ছে:

```text
TCP Connection
      ↓
Receive Request
      ↓
Parse Request
      ↓
Send Response
      ↓
Close TCP Connection
```

এটা তোমার learning server-এর জন্য simple approach।

কিন্তু production HTTP server-এ connection handling আরও sophisticated।

---

# 6. TCP + HTTP কি মিলে Socket Connection Close করে?

না।

বরং layers এভাবে ভাবো:

```text
HTTP
──────
TCP
──────
IP
──────
Network
```

HTTP request শেষ হলেই TCP connection অবশ্যই close হবে—এটা নিয়ম নয়।

HTTP এবং TCP আলাদা layer-এর protocol।

---

# 7. HTTP Connection Close করলে কী হয়?

যদি TCP connection close করা হয়:

```text
Client
   │
   │ TCP Connection
   │
Server
   X
 Connection Closed
```

তাহলে সেই TCP connection-এর মাধ্যমে আর data পাঠানো যাবে না।

আবার request করতে হলে নতুন TCP connection তৈরি করতে হতে পারে।

---

# 8. এখন আসি `epoll`-এ

এটা খুব important networking concept।

Linux operating system-এ **epoll** হলো একটি I/O event notification mechanism।

এর মাধ্যমে একটি thread/process অনেকগুলো file descriptor/socket monitor করতে পারে।

সহজভাবে:

> **একটি thread অনেকগুলো socket-এর উপর কোনো I/O event ঘটেছে কি না সেটা efficiently monitor করতে পারে।**

---

# 9. সমস্যাটা কী?

ধরো server-এ:

```text
10,000 clients
```

connected আছে।

Naive approach হলে:

```text
Client 1 → Thread 1
Client 2 → Thread 2
Client 3 → Thread 3
...
Client 10000 → Thread 10000
```

এটা expensive হতে পারে।

কারণ thread-এর:

* memory লাগে
* scheduling overhead আছে
* context switching হয়

---

# 10. epoll কীভাবে সাহায্য করে?

ধরো:

```text
             ┌── Socket 1
             ├── Socket 2
             ├── Socket 3
             ├── Socket 4
Thread ──────┼── Socket 5
             ├── Socket 6
             ├── Socket 7
             └── Socket 10000
```

একটি thread অনেক socket monitor করতে পারে।

Thread প্রতিটি socket-এর জন্য বসে থাকে না।

বরং epoll-কে বলে:

> এই socket-গুলো monitor করো। কোনো socket-এ read/write করার মতো event হলে আমাকে জানাও।

---

# 11. epoll-এর Basic Idea

ধরো 5টা client:

```text
Socket A → কোনো data নেই
Socket B → কোনো data নেই
Socket C → data এসেছে
Socket D → কোনো data নেই
Socket E → data এসেছে
```

epoll বলতে পারে:

```text
Data available:
    Socket C
    Socket E
```

তারপর server শুধু C এবং E নিয়ে কাজ করবে।

```text
Thread
  │
  ▼
epoll_wait()
  │
  ├── Socket C ready
  └── Socket E ready
```

---

# 12. epoll কি Single Thread-এ Multiple Connection চালায়?

হ্যাঁ, **একটি event loop thread দিয়ে অনেক connection manage করা সম্ভব**।

তবে একটা গুরুত্বপূর্ণ distinction:

> epoll নিজে request process করে না; এটি socket-এর I/O events efficiently notify করে।

অর্থাৎ:

```text
epoll
  ↓
"Socket 5-এ data ready"
  ↓
Application
  ↓
read()
  ↓
Process data
```

---

# 13. Traditional Thread Model বনাম epoll

### Thread-per-connection

```text
Client 1 ── Thread 1
Client 2 ── Thread 2
Client 3 ── Thread 3
Client 4 ── Thread 4
```

### Event-driven model

```text
             Socket 1
             Socket 2
             Socket 3
             Socket 4
                 │
                 ▼
               epoll
                 │
                 ▼
            Event Loop
                 │
                 ▼
             Process
```

এখানে একই thread অনেক connection-এর event handle করতে পারে।

---

# 14. epoll-এর তিনটি গুরুত্বপূর্ণ ধারণা

### 1. Register

Server বলে:

```text
epoll:
"এই socket-গুলো monitor করো।"
```

### 2. Wait

Server অপেক্ষা করে:

```text
epoll_wait()
```

### 3. Event পাওয়া

যখন কোনো socket ready:

```text
Socket 10 → readable
Socket 25 → readable
```

তখন application তাদের process করে।

---

# 15. `epoll` কি শুধু Linux-এর?

হ্যাঁ।

`epoll` হলো Linux-specific mechanism।

Operating system অনুযায়ী একই ধরনের বিভিন্ন mechanism আছে:

```text
Linux   → epoll
BSD/macOS → kqueue
Windows → IOCP
```

এই concepts-এর goal একই ধরনের:

> অনেকগুলো I/O operation efficiently manage করা।

---

# 16. তাহলে Kestrel কীভাবে অনেক connection handle করে?

এখানেই তোমার শেখার সঙ্গে ASP.NET Core/Kestrel-এর connection তৈরি হচ্ছে।

তুমি এখন করছ:

```text
TcpListener
    ↓
AcceptTcpClientAsync
    ↓
HandleClient
```

Production server-এ networking এবং I/O handling অনেক বেশি sophisticated।

Operating system-এর async I/O capabilities ব্যবহার করে অনেক connection efficiently manage করা যায়।

Conceptually:

```text
Many Clients
     ↓
Operating System I/O
     ↓
Kestrel
     ↓
.NET async APIs
     ↓
HTTP Parser
     ↓
Middleware Pipeline
```

তাই Kestrel-এর মতো server-এ **"প্রতিটি connection = একটি dedicated thread"** এই simple model ধরে নেওয়া ঠিক নয়।

---

# 17. Merkle — Extra Topic

তুমি `Merkle` লিখেছ, সম্ভবত **Merkle Tree** বোঝাচ্ছ।

Merkle Tree মূলত অনেকগুলো data-এর integrity efficiently verify করার একটি tree-based structure।

Basic idea:

```text
              Root Hash
             /         \
         Hash AB       Hash CD
         /    \        /    \
      Hash A Hash B  Hash C Hash D
        │      │       │      │
       Data   Data    Data   Data
```

প্রতিটি leaf-এর data থেকে hash তৈরি হয়।

তারপর দুইটা hash combine করে আবার hash:

```text
Hash(A) + Hash(B)
       ↓
    Hash(AB)
```

শেষ পর্যন্ত:

```text
Hash(AB) + Hash(CD)
          ↓
      Root Hash
```

এই root hash পুরো data structure-এর একটি fingerprint-এর মতো কাজ করে।

---

# 18. Merkle Tree কোথায় ব্যবহার হয়?

Merkle Tree গুরুত্বপূর্ণ কারণ বড় dataset-এর integrity efficiently verify করা যায়।

এটি বিভিন্ন distributed/data-integrity systems-এ ব্যবহৃত হয়েছে, যেমন:

* Git-এর object hashing model-এ related tree/hash concepts
* Distributed systems
* Blockchain systems
* Peer-to-peer systems
* কিছু database/storage systems

তবে মনে রাখবে:

> **Merkle Tree networking-এর জন্য প্রয়োজনীয় নয়।**

এটা TCP বা HTTP-এর অংশ নয়।

---

# 19. পুরো Concept একসাথে

এখন তোমার পুরো learning chain:

```text
                 Application
                     │
                    HTTP
                     │
                     TCP
                     │
                   Socket
                     │
                     IP
                     │
              Operating System
                     │
          ┌──────────┴──────────┐
          │                     │
       epoll                  IOCP
       Linux                 Windows
          │                     │
          └──────────┬──────────┘
                     │
                  Kestrel
                     │
              HTTP Processing
                     │
                Middleware
                     │
                 Endpoint
```

আর connection-এর ক্ষেত্রে:

```text
Client
  │
  │ TCP 3-Way Handshake
  ▼
TCP Connection
  │
  │ HTTP Request
  ▼
Server
  │
  │ HTTP Response
  ▼
Client
```

**সবচেয়ে গুরুত্বপূর্ণ ৪টা distinction মনে রাখো:**

```text
Socket → communication endpoint/interface

TCP → reliable byte-stream transport protocol

HTTP → application-level request/response protocol

epoll → Linux-এর I/O event notification mechanism
```

