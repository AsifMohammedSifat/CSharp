# ASP.NET Core-এ নিজের Middleware ও Pipeline বানানো

> এই নোটটা মূল ভার্সনের একটা পরিষ্কার, bug-fixed এবং একটু বেশি ব্যাখ্যা-সহ version। কোডের typo গুলো ঠিক করা হয়েছে (যেমন `MiddlwareDelegate` → `MiddlewareDelegate`), এবং শেষে ASP.NET Core-এর আসল pipeline-এর সাথে তুলনা যোগ করা হয়েছে।

---

## 1. প্রথমে Project তৈরি করা

```bash
dotnet new webapi -n MiddlewareDemo
cd MiddlewareDemo
dotnet run
```

`dotnet run` দিলে ASP.NET Core application চালু হবে (default port সাধারণত `http://localhost:5xxx`, terminal-এ exact URL দেখা যাবে)।

---

## 2. সাধারণ ASP.NET Core Application

```csharp
var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

app.UseHttpsRedirection();

app.Run();
```

এখানে দুইটা গুরুত্বপূর্ণ object আছে:

```text
builder  →  app
```

---

## 3. `WebApplication.CreateBuilder(args)` কী?

```csharp
var builder = WebApplication.CreateBuilder(args);
```

এখানে application-এর জন্য বিভিন্ন জিনিস **configure এবং register** করার foundation তৈরি হয়।

> `builder` ব্যবহার করে application-এর services এবং configuration setup করা হয় — এখনও application "তৈরি" হয়নি, শুধু প্রস্তুতি চলছে।

Dependency Injection-এর উদাহরণ:

```csharp
builder.Services.AddSingleton<MyService>();
builder.Services.AddScoped<UserService>();
```

মোট কথা এই জায়গায় সাধারণত থাকে:

```csharp
var builder = WebApplication.CreateBuilder(args);

// এখানে সাধারণত:
// - DI service register (AddSingleton, AddScoped, AddTransient)
// - configuration (appsettings.json load)
// - logging setup
// - অন্যান্য application setup
```

---

## 4. `builder.Build()` কী করে?

```csharp
var app = builder.Build();
```

`builder`-এ যা কিছু register করা হয়েছিল, সেই সব configuration অনুযায়ী এখন একটা **actual, runnable `WebApplication`** object তৈরি হয়।

```text
WebApplication.CreateBuilder()
        │
        ▼
     builder  (configuration + DI + logging জমা হচ্ছে)
        │
        ▼
   builder.Build()
        │
        ▼
       app   (runnable application)
```

> "আমার application-এর configuration অনুযায়ী actual application object তৈরি করো — এখন থেকে আমি request handle করতে প্রস্তুত।"

`builder.Build()`-এর পর নতুন কোনো service `builder.Services`-এ register করা যায় না — সেই পর্যায় শেষ।

---

## 5. ASP.NET Core Middleware কী?

Middleware হলো এমন একটি component যেটা প্রতিটা HTTP request-এর পথে (pipeline-এ) বসে কাজ করে।

```text
Browser
   │
   ▼  Request
Middleware 1
   │
   ▼
Middleware 2
   │
   ▼
Middleware 3
   │
   ▼  Response
Browser
```

একটি middleware চাইলে request পরের middleware-এ পাঠানোর **আগে** এবং **পরে**, দুই জায়গাতেই কাজ করতে পারে:

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");

    await next();

    Console.WriteLine("After");
});
```

```text
Request → Before → next() → পরের middleware → ... → After
```

---

## 6. আমরা কী করতে চাই?

এই টিউটোরিয়ালে আমরা ASP.NET Core-এর built-in middleware pipeline (`app.Use(...)`) সরাসরি ব্যবহার না করে, **নিজেরা একটা mini middleware pipeline সিস্টেম বানাব** — যাতে বোঝা যায় ভেতরে আসলে কী ঘটে।

```text
ASP.NET Core-এর built-in Pipeline   →  শুধু বোঝার জন্য পাশে রাখলাম
আমাদের নিজের তৈরি Custom Pipeline   →  এটা বানাবো
```

লক্ষ্য:

```text
Request → Our Middleware 1 → Our Middleware 2 → End → Response
```

> **মনে রাখা ভালো:** এটা শুধুই একটা শেখার (educational) প্রজেক্ট। আসল প্রোডাকশন অ্যাপে ASP.NET Core-এর নিজস্ব `IApplicationBuilder` / `app.Use()` ব্যবহার করাই উচিত — কারণ সেটা routing, exception handling, endpoint metadata ইত্যাদির সাথে ভালোভাবে integrated। নিচে সেকশন ১৩-এ তুলনাটা আছে।

---

## 7. কাজের ধাপগুলো

```text
Step 1 → Project তৈরি
Step 2 → Middleware-এর delegate/structure ঠিক করা
Step 3 → PipelineBuilder ক্লাস তৈরি
Step 4 → Middleware register (Use) করা
Step 5 → Pipeline build করা
Step 6 → ASP.NET Core-এর request-এর সাথে আমাদের pipeline connect করা
Step 7 → dotnet run করে টেস্ট করা
```

---

## 8. Step 1 — Project তৈরি

```bash
dotnet new webapi -n MiddlewareDemo
cd MiddlewareDemo
dotnet run
```

---

## 9. Step 2 — Middleware Delegate

```csharp
public delegate Task MiddlewareDelegate(
    HttpContext context,
    Func<HttpContext, Task> next
);
```

> মূল নোটে এই delegate-এর নাম ভুলবশত `MiddlwareDelegate` লেখা হয়েছিল (একটা `e` মিসিং)। কোড কম্পাইল হলেও এটা সব জায়গায় একই বানান হতে হবে — এখানে সঠিক বানান `MiddlewareDelegate` ব্যবহার করা হয়েছে।

দুইটা parameter:

### `context`

```csharp
HttpContext context
```

Current HTTP request এবং response-এর তথ্য এখানে থাকে:

```csharp
context.Request
context.Response
```

### `next`

```csharp
Func<HttpContext, Task> next
```

`next` মানে: **"আমার কাজ শেষ হলে পরের middleware-কে call করো।"**

```text
Middleware 1 → next(ctx) → Middleware 2
```

---

## 10. Step 3 — PipelineBuilder ক্লাস

Middleware রাখার জন্য একটা list দরকার:

```csharp
public class PipelineBuilder
{
    private readonly List<MiddlewareDelegate> _middlewares = new();
}
```

```text
_middlewares
  [0] → Middleware 1
  [1] → Middleware 2
  [2] → Middleware 3
```

---

## 11. Middleware Add করার জন্য `Use()`

```csharp
public PipelineBuilder Use(MiddlewareDelegate middleware)
{
    _middlewares.Add(middleware);
    return this;
}
```

`this` return করার কারণে **fluent/chainable API** পাওয়া যায়:

```csharp
new PipelineBuilder()
    .Use(Middleware1)
    .Use(Middleware2);
```

প্রথম `Use()`-এর পর:

```text
_middlewares
  [0] → Middleware1
```

দ্বিতীয় `Use()`-এর পর:

```text
_middlewares
  [0] → Middleware1
  [1] → Middleware2
```

---

## 12. Step 4 — Pipeline Build করা (মূল অংশ)

```csharp
public Func<HttpContext, Task> Build()
```

আমাদের কাছে middleware-এর একটা **list** আছে, কিন্তু execute করার জন্য দরকার একটা single, chained **function**:

```csharp
Func<HttpContext, Task> pipeline;

// ব্যবহার:
await pipeline(context);
```

### ধাপ ১ — Pipeline-এর "শেষ" (terminal) তৈরি করা

```csharp
Func<HttpContext, Task> pipeline = context =>
{
    Console.WriteLine("End");
    return Task.CompletedTask;
};
```

এটাই pipeline-এর একদম শেষ ধাপ — কোনো middleware বাকি না থাকলে এখানে এসে থামবে।

### ধাপ ২ — পেছন থেকে সামনে (reverse order) loop করা

```csharp
for (var i = _middlewares.Count - 1; i >= 0; i--)
{
    MiddlewareDelegate current = _middlewares[i];
    Func<HttpContext, Task> next = pipeline;

    pipeline = ctx => current(ctx, next);
}
```

**কেন reverse order-এ (শেষ থেকে শুরু) loop করা হচ্ছে?**

কারণ প্রতিটা middleware-কে তার "পরে কী হবে" (`next`) — সেটা আগে থেকেই জানতে হয়। তাই সবচেয়ে শেষের middleware দিয়ে শুরু করে, ধীরে ধীরে তার সামনে নতুন middleware "মুড়িয়ে" (wrap করে) দেওয়া হয় — অনেকটা পেঁয়াজের খোসার মতো, ভেতর থেকে বাইরের দিকে তৈরি হয়।

`Middleware1, Middleware2` — এই দুইটার জন্য ধাপে ধাপে কী ঘটে:

**i = 1 (Middleware2):**
```text
current = Middleware2
next    = End                 (এখনো পর্যন্ত pipeline যা ছিল)
pipeline = ctx => Middleware2(ctx, End)
```

**i = 0 (Middleware1):**
```text
current = Middleware1
next    = আগের pipeline = ctx => Middleware2(ctx, End)
pipeline = ctx => Middleware1(ctx, next)
```

শেষ ফলাফল:

```text
pipeline(ctx)
   = Middleware1(ctx, next)
   যেখানে next কল করলে চলে যায় Middleware2(ctx, End)-এ
```

অর্থাৎ চূড়ান্ত pipeline:

```text
Middleware1 → Middleware2 → End
```

এটাই একদম সঠিক ক্রম — যদিও আমরা list-টা **উল্টো** ঘুরেছিলাম, ফলাফল **সঠিক (স্বাভাবিক) ক্রমেই** আসে। এটাই এই প্যাটার্নের সবচেয়ে গুরুত্বপূর্ণ এবং একটু কনফিউজিং অংশ।

### সম্পূর্ণ `Build()` মেথড

```csharp
public Func<HttpContext, Task> Build()
{
    Func<HttpContext, Task> pipeline = context =>
    {
        Console.WriteLine("End");
        return Task.CompletedTask;
    };

    for (var i = _middlewares.Count - 1; i >= 0; i--)
    {
        MiddlewareDelegate current = _middlewares[i];
        Func<HttpContext, Task> next = pipeline;

        pipeline = ctx => current(ctx, next);
    }

    return pipeline;
}
```

---

## 13. পুরো Code

### `PipelineBuilder.cs`

```csharp
public delegate Task MiddlewareDelegate(
    HttpContext context,
    Func<HttpContext, Task> next
);

public class PipelineBuilder
{
    private readonly List<MiddlewareDelegate> _middlewares = new();

    public PipelineBuilder Use(MiddlewareDelegate middleware)
    {
        _middlewares.Add(middleware);
        return this;
    }

    public Func<HttpContext, Task> Build()
    {
        Func<HttpContext, Task> pipeline = context =>
        {
            Console.WriteLine("End");
            return Task.CompletedTask;
        };

        for (var i = _middlewares.Count - 1; i >= 0; i--)
        {
            MiddlewareDelegate current = _middlewares[i];
            Func<HttpContext, Task> next = pipeline;

            pipeline = ctx => current(ctx, next);
        }

        return pipeline;
    }
}
```

### `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var pipeline = new PipelineBuilder()
    .Use(async (context, next) =>
    {
        Console.WriteLine("Middleware 1 - Before");
        await next(context);
        Console.WriteLine("Middleware 1 - After");
    })
    .Use(async (context, next) =>
    {
        Console.WriteLine("Middleware 2 - Before");
        await next(context);
        Console.WriteLine("Middleware 2 - After");
    })
    .Build();

app.Run(async ctx =>
{
    await pipeline(ctx);
    await ctx.Response.WriteAsync("Hello from custom pipeline!");
});
```

---

## 14. Request করলে কী হবে?

Browser থেকে `http://localhost:5260`-এ request করলে console-এ দেখা যাবে:

```text
Middleware 1 - Before
Middleware 2 - Before
End
Middleware 2 - After
Middleware 1 - After
```

আর browser-এ দেখাবে:

```text
Hello from custom pipeline!
```

---

## 15. সম্পূর্ণ Flow ডায়াগ্রাম

```text
                ASP.NET Core
                     │
                     ▼
              app.Run(ctx)
                     │
                     ▼
             Custom Pipeline
                     │
                     ▼
              Middleware 1 (Before)
                     │
                 next(ctx)
                     │
                     ▼
              Middleware 2 (Before)
                     │
                 next(ctx)
                     │
                     ▼
                   End
                     │
                     ▼
              Middleware 2 (After)
                     │
                     ▼
              Middleware 1 (After)
                     │
                     ▼
     ctx.Response.WriteAsync("Hello...")
```

---

## 16. আসল ASP.NET Core Pipeline-এর সাথে তুলনা

আমরা যা বানালাম, সেটা আসলে ASP.NET Core নিজেই ভেতরে (conceptually) এভাবেই কাজ করে:

| আমাদের Custom সিস্টেম | ASP.NET Core-এর আসল সিস্টেম |
|---|---|
| `MiddlewareDelegate` | `RequestDelegate` |
| `PipelineBuilder` | `IApplicationBuilder` |
| `PipelineBuilder.Use()` | `app.Use()` |
| `PipelineBuilder.Build()` | `IApplicationBuilder.Build()` (framework ভেতরে-ভেতরে করে) |
| `app.Run(async ctx => ...)` (আমাদের terminal handler) | `app.Run(...)` বা endpoint-এ পৌঁছানো |

মূল পার্থক্য: ASP.NET Core-এর pipeline routing, endpoint dispatch, exception handling middleware, authentication/authorization ইত্যাদির সাথে গভীরভাবে integrated, আর আমাদের version শুধু concept বোঝানোর জন্য একটা সরল সংস্করণ।

---

## 17. এক লাইনে সারমর্ম

- **`Use()`** → middleware list-এ জমা রাখে (register করে, execute করে না)
- **`Build()`** → middleware list-কে একটা single, chained, executable pipeline function-এ রূপান্তর করে
- **reverse loop** → প্রতিটা middleware-কে তার `next` আগেই জানিয়ে দেওয়ার একটা কৌশল, যার ফলাফল সঠিক (normal) ক্রমে আসে
- **`next()`** → বর্তমান middleware-এর কাজ শেষে পরের middleware-এ control পাঠায়
- **`End`** → pipeline-এর একদম শেষ ধাপ, যখন আর কোনো middleware বাকি থাকে না
- **`app.Run()`** → ASP.NET Core-এর আসল HTTP request-কে আমাদের custom pipeline-এর সাথে connect করে

---

## 18. সম্ভাব্য ভুল / সতর্কতা (Common Pitfalls)

1. **`next()` কল করতে ভুলে যাওয়া** — যদি কোনো middleware-এ `await next(context);` না লেখা হয়, তাহলে pipeline সেখানেই থেমে যাবে, পরের middleware বা `End` কখনো চলবে না।
2. **একই middleware-এ `next()` একাধিকবার কল করা** — এটা logically ভুল এবং exception বা অস্বাভাবিক আচরণ ঘটাতে পারে।
3. **delegate-এর নামের বানান সব জায়গায় এক না রাখা** — যেমন মূল নোটে `MiddlwareDelegate` লেখা হয়েছিল, যা compile হলেও maintainability-এর জন্য খারাপ অভ্যাস।
4. **`builder.Services`-এ `builder.Build()`-এর পরে service যোগ করার চেষ্টা করা** — এটা কাজ করবে না, কারণ ততক্ষণে configuration "লক" হয়ে গেছে।
