# Dependency Injection (DI) — Zero to Hero বাংলা নোটস (C#)
---

## ভূমিকা: Dependency Injection কী এবং কেন দরকার?

**Dependency** মানে হলো — একটা ক্লাস চালাতে গেলে আরেকটা ক্লাস/সার্ভিস লাগে। যেমন `NotificationService` কাজ করতে গেলে তার `EmailService` লাগে। এই `EmailService`-টাই হলো `NotificationService`-এর **dependency**।

**Dependency Injection (DI)** হলো একটা ডিজাইন প্যাটার্ন, যেখানে একটা ক্লাস নিজে তার dependency তৈরি করে না (`new` keyword দিয়ে) — বরং বাইরে থেকে সেই dependency-টা তাকে "inject" (সরবরাহ) করা হয়।

### কেন দরকার?
- **Tight Coupling কমানো**: একটা ক্লাস অন্য ক্লাসের সাথে সরাসরি বাঁধা থাকলে (hard-coded `new`), পরে সেটা পরিবর্তন করা কঠিন হয়ে যায়।
- **Testability**: DI থাকলে Unit Test করার সময় real service-এর বদলে fake/mock service বসানো যায়।
- **Maintainability**: কোড পরিবর্তন করা সহজ হয় — একটা জায়গায় পরিবর্তন করলেই পুরো সিস্টেমে প্রভাব পড়ে।
- **SOLID Principle-এর "D"**: Dependency Inversion Principle — high-level module কখনো low-level module-এর উপর সরাসরি নির্ভর করবে না, বরং দুটোই abstraction (interface)-এর উপর নির্ভর করবে।

এই নোটসে আমরা মোট **৮টি ধাপ (Step)** এ শিখব — একদম raw `new` keyword থেকে শুরু করে নিজের হাতে একটা mini DI Container বানানো পর্যন্ত।

---

## Chapter 1: Manual Implementation — সবচেয়ে খারাপ (Tightly Coupled) উপায়

### কোড
```csharp
// Program.cs
public class Program
{
    static void Main(string[] args)
    {
        NotificationService notificationService = new NotificationService();
        notificationService.NotifyUser("a@gmail.com", "Hello");
    }
}

// NotificationService.cs
class NotificationService
{
    private readonly EmailService _emailService;

    public NotificationService()
    {
        _emailService = new EmailService();
    }

    public void NotifyUser(string to, string message)
    {
        _emailService.sendEmail(to, message);
    }
}

class EmailService
{
    public void sendEmail(string to, string message)
    {
        Console.WriteLine($"Sending {message} to {to}");
    }
}
```

### সমস্যা কী?
- `NotificationService`-এর ভেতরেই `new EmailService()` করা হচ্ছে। মানে `NotificationService` নিজেই সিদ্ধান্ত নিচ্ছে কোন `EmailService` ব্যবহার করবে।
- ভবিষ্যতে যদি SMS বা Push Notification দিয়ে নোটিফাই করতে চাও, তাহলে `NotificationService`-এর ভেতরের কোড পরিবর্তন করতে হবে।
- Unit Test করা প্রায় অসম্ভব — কারণ real `EmailService` ছাড়া `NotificationService` টেস্ট করা যাবে না।
- এটাকে বলে **Tight Coupling**।

---

## Chapter 2: বাইরে থেকে Dependency ইনজেক্ট করা (Constructor Injection)

এখানে মূল আইডিয়া হলো — `NotificationService` নিজে `EmailService` তৈরি করবে না, বরং **Constructor**-এর মাধ্যমে বাইরে থেকে সেটা পাঠিয়ে দেওয়া হবে।

### কোড
```csharp
// Program.cs
public class Program
{
    static void Main(string[] args)
    {
        EmailService emailService = new EmailService();
        NotificationService notificationService = new NotificationService(emailService);
        notificationService.NotifyUser("a@gmail.com", "Hello");
    }
}

// NotificationService.cs
class NotificationService
{
    private readonly EmailService _emailService;

    public NotificationService(EmailService emailService)
    {
        _emailService = emailService;
    }

    public void NotifyUser(string to, string message)
    {
        _emailService.sendEmail(to, message);
    }
}

class EmailService
{
    public void sendEmail(string to, string message)
    {
        Console.WriteLine($"Sending {message} to {to}");
    }
}
```

### C# 12+ Primary Constructor দিয়ে (আরও ছোট করে)
```csharp
class NotificationService(EmailService emailService)
{
    public void NotifyUser(string to, string message)
    {
        emailService.sendEmail(to, message);
    }
}
```

### কী উন্নতি হলো?
- এখন `NotificationService` আর নিজে `EmailService` তৈরি করছে না — বাইরে থেকে (Program.cs) সেটা দেওয়া হচ্ছে।
- একে বলে **Constructor Injection** — DI-এর সবচেয়ে জনপ্রিয় পদ্ধতি।

### এখনো সমস্যা কী?
- `NotificationService` এখনো `EmailService`-এর concrete (নির্দিষ্ট) ক্লাসের উপর নির্ভরশীল, কোনো abstraction/interface নেই।
- মানে ভবিষ্যতে `SmsService` দিয়ে replace করতে চাইলে constructor-এর প্যারামিটার টাইপ বদলাতে হবে — যেটা আবার একটা Breaking Change।

---

## Chapter 3: Interface দিয়ে Loose Coupling তৈরি করা

এবার আমরা **Dependency Inversion Principle** প্রয়োগ করব — `NotificationService` একটা concrete `EmailService`-এর উপর নয়, বরং একটা **interface**-এর উপর নির্ভর করবে।

### কোড
```csharp
class NotificationService(INotificationService service)
{
    public void NotifyUser(string to, string message)
    {
        service.sendMessage(to, message);
    }
}

public interface INotificationService
{
    void sendMessage(string to, string message);
}

class EmailService : INotificationService
{
    public void sendMessage(string to, string message)
    {
        Console.WriteLine($"Sending {message} to {to}");
    }
}
```

তবে `Program.cs`-এ এখনো `new` keyword ব্যবহার করতে হচ্ছে:
```csharp
public class Program
{
    static void Main(string[] args)
    {
        EmailService emailService = new EmailService();
        NotificationService notificationService = new NotificationService(emailService);
        notificationService.NotifyUser("a@gmail.com", "Hello");
    }
}
```

### কী উন্নতি হলো?
- এখন `NotificationService` শুধু `INotificationService` interface চেনে, `EmailService`-এর concrete implementation চেনে না।
- ভবিষ্যতে `SmsService : INotificationService` বানিয়ে সহজেই swap করা যাবে, `NotificationService`-এর কোনো কোড পরিবর্তন ছাড়াই।

### এখনো সমস্যা কী?
- `Program.cs`-এ এখনো ম্যানুয়ালি `new EmailService()` এবং `new NotificationService(emailService)` লিখতে হচ্ছে।
- ছোট প্রজেক্টে এটা সমস্যা না, কিন্তু বড় প্রজেক্টে যেখানে শত শত dependency আছে, সেখানে প্রতিটা জায়গায় ম্যানুয়ালি অবজেক্ট তৈরি করা কষ্টকর ও ভুল-প্রবণ (error-prone) হয়ে যায়।
- এই সমস্যা সমাধানের জন্যই পরের ধাপে আমরা **Factory Pattern**-এ যাব, যাতে `new` keyword ব্যবহারকারীর (caller) থেকে সম্পূর্ণ লুকিয়ে ফেলা যায়।

---

## Chapter 4: Factory Pattern দিয়ে Object তৈরির দায়িত্ব আলাদা করা

এখন আমরা `new` keyword পুরোপুরি `Program.cs` থেকে সরিয়ে একটা কেন্দ্রীয় (centralized) `ObjectFactory` ক্লাসে নিয়ে যাব।

### কোড
```csharp
// Program.cs
public class Program
{
    static void Main(string[] args)
    {
        NotificationService notificationService =
            ObjectFactory<NotificationService>.Get("notification-service");
        notificationService.NotifyUser("a@gmail.com", "Hello");
    }
}

// ObjectFactory.cs
class ObjectFactory<T>
{
    public static T Get(string service)
    {
        if (service.Equals("email-service", StringComparison.CurrentCultureIgnoreCase))
        {
            return (T)(object)new EmailService();
        }
        if (service.Equals("notification-service", StringComparison.CurrentCultureIgnoreCase))
        {
            INotificationService emailService =
                ObjectFactory<INotificationService>.Get("email-service");
            return (T)(object)new NotificationService(emailService);
        }
        throw new Exception("Unknown service");
    }
}
```

### কী উন্নতি হলো?
- এখন `Program.cs`-এ কোনো `new` নেই। `ObjectFactory` জানে কোন সার্ভিসের জন্য কোন implementation তৈরি করতে হবে।
- Object তৈরির সব লজিক একটা জায়গায় (Single Responsibility)।

### সমস্যা কী?
- সার্ভিসের নাম **string** ("notification-service") দিয়ে দেওয়া হচ্ছে — এটা টাইপ-সেফ (type-safe) না। যদি কেউ ভুল বানানে "notifcation-service" লেখে, তাহলে Compile-time-এ ধরা পড়বে না, শুধু Runtime-এ Exception আসবে।
- Magic string ব্যবহার করা সবসময় ভালো প্র্যাকটিস না।

---

## Chapter 5: Factory Pattern-কে আরও Type-Safe করা (Generic দিয়ে)

Magic string সমস্যা সমাধানের জন্য এখন আমরা `typeof(T)` ব্যবহার করব, string-এর বদলে।

### কোড
```csharp
// Program.cs
public class Program
{
    static void Main(string[] args)
    {
        NotificationService notificationService = ObjectFactory<NotificationService>.Get();
        notificationService.NotifyUser("a@gmail.com", "Hello");
    }
}

// ObjectFactory.cs
class ObjectFactory<T>
{
    public static T Get()
    {
        if (typeof(T) == typeof(NotificationService))
        {
            var emailService = ObjectFactory<INotificationService>.Get();
            return (T)(object)new NotificationService(emailService);
        }

        if (typeof(T) == typeof(INotificationService))
        {
            return (T)(object)new EmailService();
        }

        throw new Exception("Unknown service");
    }
}
```

### কী উন্নতি হলো?
- এখন আর string লিখতে হয় না, বরং Generic Type (`<NotificationService>`) দিয়ে সার্ভিস চাওয়া হয় — যা টাইপ-সেফ।
- IntelliSense সাপোর্ট পাওয়া যায়, টাইপো করলে Compile-time-এ error ধরা পড়ে।

### সমস্যা কী?
- `ObjectFactory`-এর ভেতরে প্রতিটা টাইপের জন্য `if` কন্ডিশন হার্ডকোড করা আছে। নতুন সার্ভিস যোগ করলে `ObjectFactory`-এর কোড বদলাতে হয় — যা **Open/Closed Principle** ভঙ্গ করে।
- এই সমস্যার প্রকৃত সমাধান হলো একটা প্রপার **DI Container/Framework** ব্যবহার করা, যেখানে সার্ভিস "register" করা যায় এবং framework নিজেই বুঝে নেয় কী দিয়ে কী তৈরি করতে হবে।

---

## Chapter 6: Microsoft.Extensions.DependencyInjection — Real World DI Container

এবার আমরা নিজে হাতে বানানো `ObjectFactory`-এর বদলে .NET-এর অফিসিয়াল, Industry-Standard DI Container ব্যবহার করব।

### Package Install
```bash
dotnet add package Microsoft.Extensions.DependencyInjection
```

### কোড
```csharp
using Microsoft.Extensions.DependencyInjection;

public class Program
{
    static void Main(string[] args)
    {
        var services = new ServiceCollection();
        services.AddTransient<NotificationService>();
        services.AddTransient<INotificationService, EmailService>();

        var serviceProvider = services.BuildServiceProvider();

        var notificationService = serviceProvider.GetRequiredService<NotificationService>();
        notificationService.NotifyUser("a@gmail.com", "hello");
    }
}

class NotificationService(INotificationService service)
{
    public void NotifyUser(string to, string message)
    {
        service.sendMessage(to, message);
    }
}

public interface INotificationService
{
    void sendMessage(string to, string message);
}

class EmailService : INotificationService
{
    public void sendMessage(string to, string message)
    {
        Console.WriteLine($"Sending {message} to {to}");
    }
}
```

# How DI Works | WorkFlow

```csharp
var notificationService =
    serviceProvider.GetRequiredService<NotificationService>();
```

দেখতে মনে হয় শুধু একটা object চাওয়া হচ্ছে। কিন্তু **behind the scene অনেকগুলো step হচ্ছে**।

তোমার দেওয়া code অনুযায়ী `NotificationService` registered আছে এবং তার constructor-এ `INotificationService` লাগে। 

---

# প্রথমে পুরো picture

তোমার code:

```csharp
var services = new ServiceCollection();

services.AddTransient<NotificationService>();
services.AddTransient<INotificationService, EmailService>();

var serviceProvider = services.BuildServiceProvider();

var notificationService =
    serviceProvider.GetRequiredService<NotificationService>();
```

এখানে **তুমি `new NotificationService(...)` লিখোনি**।

তাহলে প্রশ্ন:

> তাহলে `NotificationService` object বানালো কে?

**Answer: `serviceProvider` / DI Container।**

---

# Step 1 — Registration-এর সময় object তৈরি হচ্ছে না

এই লাইন:

```csharp
services.AddTransient<NotificationService>();
```

মানে কিন্তু:

```csharp
new NotificationService();
```

না।

বরং Container-কে বলা হচ্ছে:

> "ভবিষ্যতে যদি কেউ `NotificationService` চায়, তখন তুমি এটা তৈরি করে দেবে।"

আর:

```csharp
services.AddTransient<INotificationService, EmailService>();
```

মানে:

> "কেউ যদি `INotificationService` চায়, তাকে `EmailService` implementation দেবে।"

তোমার notes-এও `ServiceCollection`-কে registration list হিসেবে এবং `AddTransient`-কে registration হিসেবে দেখানো হয়েছে। 

তখন Container-এর ভিতরে roughly এমন information আছে:

```text
DI Container
│
├── NotificationService
│      └── Lifetime = Transient
│
└── INotificationService
       └── Implementation = EmailService
       └── Lifetime = Transient
```

এখনো **কোনো object তৈরি হয়নি**।

---

# Step 2 — `BuildServiceProvider()`

এই লাইন:

```csharp
var serviceProvider = services.BuildServiceProvider();
```

এর মাধ্যমে registration list থেকে একটা **ServiceProvider/Container** তৈরি হলো।

তখন roughly:

```text
ServiceProvider
      │
      ├── NotificationService → Transient
      │
      └── INotificationService → EmailService → Transient
```

তোমার code-এর ভাষায় `BuildServiceProvider()` registration list থেকে এমন একটা container তৈরি করে যেটা পরে object supply করতে পারে। 

---

# Step 3 — এখন আসল magic 😄

তুমি লিখলে:

```csharp
serviceProvider.GetRequiredService<NotificationService>();
```

এখানে Container-কে তুমি বলছো:

> **"আমাকে একটা `NotificationService` object দাও।"**

এখন Container প্রথমে দেখে:

```text
আমি কি NotificationService-এর registration জানি?
```

উত্তর:

```text
YES ✅
```

কারণ আগে লিখেছিলে:

```csharp
services.AddTransient<NotificationService>();
```

---

# Step 4 — Container `NotificationService`-এর constructor দেখে

তোমার class:

```csharp
class NotificationService(INotificationService service)
{
    public void NotifyUser(string to, string message)
    {
        service.sendMessage(to, message);
    }
}
```

এটার constructor logically এমন:

```csharp
public NotificationService(INotificationService service)
{
    this.service = service;
}
```

Container Reflection ব্যবহার করে constructor দেখে:

```text
NotificationService
        │
        ↓
Constructor
        │
        ↓
Needs INotificationService
```

তোমার custom DI container-এ ঠিক এই concept দেখানো আছে—`GetConstructors()` দিয়ে constructor বের করে এবং `GetParameters()` দিয়ে constructor-এর dependency বের করে। 

---

# Step 5 — এখন Container-এর নতুন প্রশ্ন

Container বলবে:

> "`NotificationService` বানাতে আমার `INotificationService` লাগবে।"

তাই সে internally আবার resolve করবে:

```csharp
GetRequiredService<INotificationService>()
```

অর্থাৎ:

```text
GetRequiredService<NotificationService>()
             │
             ↓
NotificationService-এর constructor দেখল
             │
             ↓
প্রয়োজন: INotificationService
             │
             ↓
Get INotificationService
```

---

# Step 6 — `INotificationService` এর জন্য কী আছে?

তুমি registration করেছিলে:

```csharp
services.AddTransient<INotificationService, EmailService>();
```

Container জানে:

```text
INotificationService
        ↓
EmailService
```

অর্থাৎ:

> "`INotificationService` চাইলে `EmailService` বানিয়ে দাও।"

---

# Step 7 — এখন EmailService তৈরি হবে

Container internally conceptually করে:

```csharp
var emailService = new EmailService();
```

এটা **তুমি লেখোনি**।

DI Container internally object creation করছে।

তারপর:

```text
emailService
      │
      ↓
INotificationService
```

কারণ:

```csharp
EmailService : INotificationService
```

---

# Step 8 — এবার NotificationService তৈরি

এখন Container-এর কাছে dependency ready:

```text
emailService
```

তাই logically সে করে:

```csharp
var notificationService =
    new NotificationService(emailService);
```

🔥 **এই জায়গাটাই তোমার সবচেয়ে important visualization।**

তুমি লিখেছ:

```csharp
serviceProvider.GetRequiredService<NotificationService>();
```

কিন্তু internally conceptually হচ্ছে:

```csharp
var emailService = new EmailService();

var notificationService =
    new NotificationService(emailService);

return notificationService;
```

---

# পুরো process একসাথে দেখো

```text
serviceProvider
       │
       │
       │ GetRequiredService<NotificationService>()
       ↓
┌───────────────────────────────┐
│ DI Container                  │
│                               │
│ NotificationService registered│
└───────────────┬───────────────┘
                │
                ↓
   Constructor inspect করে
                │
                ↓
       Needs INotificationService
                │
                ↓
┌───────────────────────────────┐
│ Registration                  │
│                               │
│ INotificationService          │
│          ↓                    │
│      EmailService             │
└───────────────┬───────────────┘
                │
                ↓
      new EmailService()
                │
                ↓
       EmailService object
                │
                ↓
new NotificationService(emailService)
                │
                ↓
   NotificationService object
                │
                ↓
         return object
                │
                ↓
var notificationService = ...
```

---

# তাহলে তোমার variable-এ কী আসে?

এই লাইন:

```csharp
var notificationService =
    serviceProvider.GetRequiredService<NotificationService>();
```

শেষ হওয়ার পরে memory-তে roughly:

```text
notificationService
       │
       ▼
┌──────────────────────────┐
│ NotificationService      │
│                          │
│ service ─────────────┐   │
└──────────────────────┼───┘
                       │
                       ▼
              ┌─────────────────┐
              │ EmailService    │
              │                 │
              │ sendMessage()   │
              └─────────────────┘
```

অর্থাৎ:

```csharp
notificationService
```

এর ভিতরে `INotificationService service` হিসেবে একটা **EmailService object** আছে।

---

# এরপর তুমি যখন করো

```csharp
notificationService.NotifyUser(
    "a@gmail.com",
    "hello"
);
```

তখন:

```text
notificationService
        │
        ↓
NotifyUser()
        │
        ↓
service.sendMessage()
        │
        ↓
EmailService.sendMessage()
        │
        ↓
Console.WriteLine()
```

অর্থাৎ final output:

```text
Sending hello to a@gmail.com
```

---

# সবচেয়ে গুরুত্বপূর্ণ: এটা Magic না

DI Container আসলে মোটামুটি এই কাজটাই করছে:

### তুমি manually করলে:

```csharp
var emailService = new EmailService();

var notificationService =
    new NotificationService(emailService);
```

### DI Container ব্যবহার করলে:

```csharp
var notificationService =
    serviceProvider.GetRequiredService<NotificationService>();
```

**Difference হলো:** দ্বিতীয় ক্ষেত্রে `new` করার দায়িত্ব তোমার বদলে **DI Container নিয়েছে**।

তোমার নিজের বানানো Mini DI Container-এ এই কাজটা খুব সুন্দরভাবে দেখা যায়:

```csharp
var ctor = implType.GetConstructors().First();

var deps = ctor.GetParameters()
    .Select(p => GetService(p.ParameterType))
    .ToArray();

return Activator.CreateInstance(implType, deps)!;
```

এই অংশটাই basically বলে:

> **Constructor খুঁজো → constructor-এর dependency খুঁজো → dependency তৈরি করো → তারপর মূল object তৈরি করো।** 

### একটা sentence মনে রাখো:

> **`GetRequiredService<T>()` মানে শুধু "object বের করে দাও" না; DI Container-এর কাছে এটা হলো "T কীভাবে তৈরি করতে হবে সেটা বের করো, তার সব dependency recursively তৈরি করো, তারপর T-এর object আমাকে দাও।"**

এটাই `GetRequiredService<NotificationService>()`-এর **behind the scene**।


### এখানে যা যা শেখা হলো
| ধাপ | ব্যাখ্যা |
|---|---|
| `ServiceCollection` | সব সার্ভিসের "রেজিস্ট্রেশন লিস্ট" — এখানে বলা হয় কোন Interface-এর জন্য কোন Implementation ব্যবহার হবে। |
| `AddTransient<T>()` | সার্ভিস register করা হচ্ছে "Transient" lifetime দিয়ে (বিস্তারিত পরের চ্যাপ্টারে)। |
| `BuildServiceProvider()` | রেজিস্ট্রেশন লিস্ট থেকে একটা আসল "Container" তৈরি হয়, যেটা অবজেক্ট সাপ্লাই করতে পারবে। |
| `GetRequiredService<T>()` | Container থেকে চাওয়া অবজেক্ট বের করে আনা — না পেলে Exception ছুড়বে। |

### কী উন্নতি হলো?
- নিজে হাতে `ObjectFactory` মেইনটেইন করার দরকার নেই। Microsoft-এর টেস্টেড, প্রোডাকশন-রেডি লাইব্রেরি ব্যবহার হচ্ছে।
- নতুন সার্ভিস যোগ করতে চাইলে শুধু একটা `services.AddXxx<...>()` লাইন লিখলেই হবে, `ObjectFactory`-এর ভেতরের কোড বদলানো লাগবে না।

---

## Chapter 7: Service Lifetime — Transient vs Scoped vs Singleton

DI Container-এ অবজেক্ট রেজিস্ট্রেশনের সময় ৩ ধরনের **Lifetime (জীবনচক্র)** ঠিক করা যায়। এটা DI শেখার সবচেয়ে গুরুত্বপূর্ণ অংশগুলোর একটা।

| Lifetime | ব্যাখ্যা | কবে ব্যবহার করবে |
|---|---|---|
| **Transient** | প্রতিবার `GetService` কল করলেই নতুন অবজেক্ট তৈরি হয়। | হালকা, স্টেটলেস (stateless) সার্ভিসের জন্য। |
| **Scoped** | একই "scope"-এর ভেতরে একই অবজেক্ট রিইউজ হয়, কিন্তু নতুন scope-এ নতুন অবজেক্ট তৈরি হয়। | সাধারণত এক HTTP Request-এর মধ্যে একই DB Context ব্যবহার করার জন্য। |
| **Singleton** | পুরো অ্যাপ্লিকেশনের লাইফটাইমে মাত্র একবারই অবজেক্ট তৈরি হয়, সবাই সেটাই শেয়ার করে। | Configuration, Caching, Logging-এর মতো গ্লোবাল সার্ভিসের জন্য। |

### কোড
```csharp
public interface ITransientService { Guid Id { get; } }
public class TransientService : ITransientService
{
    public Guid Id { get; } = Guid.NewGuid();
}

public interface IScopedService { Guid Id { get; } }
public class ScopedService : IScopedService
{
    public Guid Id { get; } = Guid.NewGuid();
}

public interface ISingletonService { Guid Id { get; } }
public class SingletonService : ISingletonService
{
    public Guid Id { get; } = Guid.NewGuid();
}
```

```csharp
using Microsoft.Extensions.DependencyInjection;

public class Program
{
    static void Main(string[] args)
    {
        var services = new ServiceCollection();

        services.AddTransient<ITransientService, TransientService>();
        services.AddScoped<IScopedService, ScopedService>();
        services.AddSingleton<ISingletonService, SingletonService>();

        var serviceProvider = services.BuildServiceProvider();

        // Transient: প্রতিবার আলাদা Id
        var t1 = serviceProvider.GetRequiredService<ITransientService>();
        var t2 = serviceProvider.GetRequiredService<ITransientService>();
        Console.WriteLine($"Transient: {t1.Id} vs {t2.Id}"); // আলাদা

        using (var scope = serviceProvider.CreateScope())
        {
            var s1 = scope.ServiceProvider.GetRequiredService<IScopedService>();
            var s2 = scope.ServiceProvider.GetRequiredService<IScopedService>();
            Console.WriteLine($"Scoped (same scope): {s1.Id} vs {s2.Id}"); // একই
        }

        using (var scope = serviceProvider.CreateScope())
        {
            var s1 = scope.ServiceProvider.GetRequiredService<IScopedService>();
            Console.WriteLine($"Scoped (new scope): {s1.Id}"); // আগের scope থেকে আলাদা
        }

        var sg1 = serviceProvider.GetRequiredService<ISingletonService>();
        var sg2 = serviceProvider.GetRequiredService<ISingletonService>();
        Console.WriteLine($"Singleton: {sg1.Id} vs {sg2.Id}"); // সবসময় একই
    }
}
```

### মনে রাখার সহজ Trick
- **Transient** = প্রতিবার নতুন বাচ্চা 👶 (কখনো আগের বাচ্চার সাথে সম্পর্ক নেই)
- **Scoped** = একই পরিবারের ভেতরে (scope) সবাই একই জিনিস শেয়ার করে, কিন্তু নতুন পরিবার (নতুন scope) মানেই নতুন জিনিস
- **Singleton** = পুরো দেশে (Application) একটাই কপি, সবাই সেই একটাই ব্যবহার করে

### সতর্কতা (Best Practice)
- **Singleton-এর ভেতরে কখনো Scoped সার্ভিস inject করা ঠিক না** — একে বলে "Captive Dependency" সমস্যা, কারণ Singleton একবারই তৈরি হয় কিন্তু Scoped সার্ভিস Request-ভিত্তিক হওয়ার কথা।

---

## Chapter 8: নিজের হাতে একটা Mini DI Container বানানো (Custom ServiceProvider)

এই ধাপে তুমি নিজেই বুঝেছ Microsoft-এর `IServiceProvider`-এর ভেতরে আসলে কী ঘটে — এবং নিজেই একটা সিম্পল ভার্সন বানিয়েছ। এটা DI বোঝার জন্য সবচেয়ে শক্তিশালী ব্যায়াম (exercise), কারণ এখানে "under the hood" কী হয় সেটা দেখা যায়।

### মূল Building Blocks

**১. ServiceDescriptor** — কোন সার্ভিসের জন্য কোন implementation ও কোন lifetime, সেই তথ্য রাখে।
```csharp
public enum ServiceLifetime { Singleton, Scoped, Transient }

public class ServiceDescriptor(Type serviceType, Type implementationType, ServiceLifetime lifetime)
{
    public Type ServiceType { get; } = serviceType;
    public Type ImplementationType { get; } = implementationType;
    public ServiceLifetime Lifetime { get; set; } = lifetime;
    internal object? SingletonInstance { get; set; }
    internal object SingletonLock { get; } = new();
}
```

**২. CustomServiceCollection** — রেজিস্ট্রেশনের লিস্ট রাখে (Microsoft-এর `ServiceCollection`-এর মতো)।
```csharp
public class CustomServiceCollection
{
    private readonly List<ServiceDescriptor> _services = [];

    public void AddTransient<TService, TImplementation>()
        => _services.Add(new ServiceDescriptor(typeof(TService), typeof(TImplementation), ServiceLifetime.Transient));

    public void AddSingleton<TService, TImplementation>()
        => _services.Add(new ServiceDescriptor(typeof(TService), typeof(TImplementation), ServiceLifetime.Singleton));

    public void AddScoped<TService, TImplementation>()
        => _services.Add(new ServiceDescriptor(typeof(TService), typeof(TImplementation), ServiceLifetime.Scoped));

    public ServiceProvider BuildServiceProvider() => new(_services.AsReadOnly());
}
```

**৩. ServiceProvider** — আসল ম্যাজিক এখানে হয়। এটা `Reflection` ব্যবহার করে রানটাইমে বুঝে নেয় একটা ক্লাসের constructor-এ কী কী dependency লাগবে, এবং সেগুলো recursively (একটার ভেতরে আরেকটা) resolve করে।

```csharp
public class ServiceProvider
{
    private readonly IReadOnlyList<ServiceDescriptor> _services;
    private readonly Dictionary<Type, object>? _scopedCache;
    private readonly Lock _lock = new();

    public ServiceProvider(IReadOnlyList<ServiceDescriptor> services)
    {
        _services = services;
    }

    private ServiceProvider(IReadOnlyList<ServiceDescriptor> services, bool isScope)
    {
        _services = services;
        if (isScope) _scopedCache = new Dictionary<Type, object>();
    }

    public T GetRequiredService<T>() => (T)GetService(typeof(T));

    public object GetService(Type serviceType)
    {
        var descriptor = _services.FirstOrDefault(x => x.ServiceType == serviceType)
            ?? throw new Exception($"Service {serviceType.Name} isn't registered");

        return descriptor.Lifetime switch
        {
            ServiceLifetime.Transient => CreateInstance(descriptor.ImplementationType),
            ServiceLifetime.Singleton => GetSingletonInstance(descriptor),
            ServiceLifetime.Scoped => GetScopedInstance(descriptor),
            _ => throw new Exception("Unknown lifetime")
        };
    }

    public IServiceScope CreateScope()
    {
        var scopeProvider = new ServiceProvider(_services, isScope: true);
        return new ServiceScope(scopeProvider);
    }

    private object GetSingletonInstance(ServiceDescriptor descriptor)
    {
        lock (descriptor.SingletonLock)
        {
            descriptor.SingletonInstance ??= CreateInstance(descriptor.ImplementationType);
            return descriptor.SingletonInstance;
        }
    }

    private object GetScopedInstance(ServiceDescriptor descriptor)
    {
        if (_scopedCache == null)
            throw new InvalidOperationException("Cannot resolve scoped service from root provider");

        lock (_lock)
        {
            if (_scopedCache.TryGetValue(descriptor.ServiceType, out var instance)) return instance;
            instance = CreateInstance(descriptor.ImplementationType);
            _scopedCache[descriptor.ServiceType] = instance;
            return instance;
        }
    }

    private object CreateInstance(Type implType)
    {
        var ctor = implType.GetConstructors().First();
        var deps = ctor.GetParameters()
            .Select(p => GetService(p.ParameterType))
            .ToArray();

        return Activator.CreateInstance(implType, deps)!;
    }

    internal void DisposeScopedInstances()
    {
        if (_scopedCache == null) return;

        lock (_lock)
        {
            foreach (var instance in _scopedCache.Values)
            {
                if (instance is IDisposable disposable) disposable.Dispose();
            }
            _scopedCache.Clear();
        }
    }
}

public interface IServiceScope : IDisposable
{
    ServiceProvider ServiceProvider { get; }
}

public class ServiceScope(ServiceProvider serviceProvider) : IServiceScope
{
    public ServiceProvider ServiceProvider { get; } = serviceProvider;

    public void Dispose() => ServiceProvider.DisposeScopedInstances();
}
```

### এখানে যা যা গুরুত্বপূর্ণ শেখার বিষয়
1. **Reflection দিয়ে Constructor Resolve করা** — `CreateInstance()` মেথড `GetConstructors().First()` দিয়ে constructor বের করে, তারপর প্রতিটা parameter-এর জন্য recursively `GetService()` কল করে dependency resolve করে। এটাই আসল DI Container-এর মূল কৌশল।
2. **Singleton-এর জন্য Thread-Safety** — `lock (descriptor.SingletonLock)` ব্যবহার করা হয়েছে যাতে Multi-threaded পরিবেশে দুইটা থ্রেড একসাথে দুইটা আলাদা Singleton instance তৈরি না করে ফেলে (Double-Checked Locking-এর মতো একটা প্যাটার্ন)।
3. **Scoped Cache প্রতিটা Scope-এর জন্য আলাদা** — `CreateScope()` কল করলে নতুন `ServiceProvider` তৈরি হয়, যার নিজস্ব `_scopedCache` থাকে। তাই একটা scope-এর ভেতরের ক্যাশ অন্য scope-কে প্রভাবিত করে না।
4. **IDisposable Cleanup** — `DisposeScopedInstances()` scope শেষ হলে (`using` ব্লক শেষে) সব Scoped instance dispose করে দেয়, যাতে Memory Leak না হয়।

### ⚠️ ছোট্ট একটা বাগ যা ঠিক করা দরকার
তোমার মূল কোডে root `ServiceProvider` constructor-এ (`isScope` ছাড়া overload-টায়) `_scopedCache` initialize করা হয় না — মানে সেটা `null` থাকে। এটা ইচ্ছাকৃতভাবেই করা হয়েছে যাতে root provider থেকে সরাসরি Scoped service resolve করতে গেলে Exception থ্রো হয় (Microsoft-এর আসল লাইব্রেরিও এমনই আচরণ করে) — এটা আসলে বাগ না, বরং সঠিক ডিজাইন। শুধু মনে রাখা দরকার:
> **Scoped সার্ভিস সবসময় `CreateScope()`-এর ভেতর থেকেই resolve করতে হবে, root `ServiceProvider` থেকে না।**

---

## সারসংক্ষেপ: পুরো যাত্রাটা একনজরে

| ধাপ | কী শেখা হলো | মূল সমস্যা যা সমাধান হলো |
|---|---|---|
| ১ | Manual `new` | — |
| ২ | Constructor Injection | Object বাইরে থেকে ইনজেক্ট করা |
| ৩ | Interface / Abstraction | Concrete ক্লাসের উপর নির্ভরতা কমানো |
| ৪ | Factory Pattern (string-based) | `new` keyword caller থেকে সরানো |
| ৫ | Factory Pattern (type-based) | Magic string সমস্যা সমাধান |
| ৬ | Microsoft.Extensions.DependencyInjection | নিজের হাতে factory মেইনটেইনের ঝামেলা দূর করা |
| ৭ | Service Lifetimes | Transient / Scoped / Singleton-এর পার্থক্য বোঝা |
| ৮ | Custom DI Container | DI Container ভেতরে ভেতরে কীভাবে কাজ করে সেটা বোঝা |

## পরবর্তীতে যা শেখা উচিত (Next Steps)
- **ASP.NET Core-এ DI**: `Program.cs`-এ `builder.Services.AddScoped<...>()` ইত্যাদি — Web API-তে DI কীভাবে কাজ করে।
- **Constructor Injection বনাম Property/Method Injection** — কখন কোনটা ব্যবহার করা উচিত।
- **Multiple Implementation Resolve করা** — যেমন একই interface-এর একাধিক implementation থাকলে কীভাবে নির্দিষ্ট একটা বেছে নেওয়া যায় (Keyed Services, .NET 8+ ফিচার)।
- **IOptions Pattern** — Configuration Injection।
- **Decorator Pattern with DI** — একটা সার্ভিসকে wrap করে নতুন behavior যোগ করা।
