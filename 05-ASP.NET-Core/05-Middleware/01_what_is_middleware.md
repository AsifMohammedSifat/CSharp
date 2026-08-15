# C# (ASP.NET Core) Middleware — বাংলা নোট

## ১. Middleware কী? (Introduction)

**Middleware** হলো এমন একটা সফটওয়্যার কম্পোনেন্ট (একটা ছোট class বা function) যা **HTTP request এবং response** কে হ্যান্ডেল করার জন্য একটার পর একটা **pipeline** বা লাইনে সাজানো থাকে।

সহজ ভাষায় বলতে গেলে —
- Browser থেকে একটা **request** আসে।
- সেই request টা কয়েকটা ধাপ (middleware) পার হয়ে **response** হিসেবে আবার Browser-এ ফিরে যায়।
- প্রতিটা middleware তার নিজের কাজ করে — যেমন: Authentication চেক করা, Logging করা, Error handle করা, Routing করা ইত্যাদি — এবং তারপর পরবর্তী middleware-এ request টা পাঠিয়ে দেয়।

> **এক কথায়:** Middleware হলো request/response pipeline-এর একটা component, যেটা request-কে process করে এবং প্রয়োজনে পরের middleware-কে call করে অথবা নিজেই response তৈরি করে ফেরত পাঠিয়ে দেয়।

---

## ২. ছবি অনুযায়ী পুরো ফ্লো (Architecture Explanation)
<img width="573" height="168" alt="image" src="https://github.com/user-attachments/assets/f2e46cfe-6cd3-4d1f-bce6-c9e7f5c4e9ab" />


ছবিতে তিনটা ধাপে পুরো concept টা দেখানো হয়েছে:

### ধাপ ১: Browser ও Kestrel-এর সম্পর্ক
```
Browser  --request-->   Kestrel
Browser  <--response--  Kestrel
```
- **Browser** ইউজারের পক্ষ থেকে একটা **request** পাঠায়।
- **Kestrel** হলো ASP.NET Core-এর নিজস্ব **web server**, যেটা এই request-টা রিসিভ করে।
- কাজ শেষে Kestrel সেই request-এর ভিত্তিতে তৈরি হওয়া **response** আবার Browser-এ ফেরত পাঠায়।
- Kestrel প্রতিটা request-কে একটা **HttpContext** অবজেক্টে রূপান্তর করে।

### ধাপ ২: Middleware Pipeline
```
Kestrel --HttpContext--> [Middleware 1] --> [Middleware 2] --> [Middleware 3] --HttpContext--> Kestrel
```
- Kestrel, request-কে **HttpContext** আকারে Middleware Pipeline-এ পাঠিয়ে দেয়।
- **HttpContext** অবজেক্টের মধ্যে request ও response সংক্রান্ত সব তথ্য থাকে (headers, body, status code, user info ইত্যাদি)।
- Pipeline-এ একের পর এক middleware (এখানে 1, 2, 3) সাজানো থাকে, এবং একই HttpContext অবজেক্টটাই সবার মধ্য দিয়ে পাস হতে থাকে।
- সব middleware কাজ শেষ করার পর, আবার সেই HttpContext (এখন response সহ) ফেরত আসে Kestrel-এ, এবং Kestrel সেটা Browser-এ পাঠিয়ে দেয়।

### ধাপ ৩: প্রতিটা Middleware-এর ভেতরে কী হয় (`next()` এর কাজ)
ছবির শেষ অংশে Middleware 2-এর ভেতরে দেখানো হয়েছে:
```
[ Process Request ]
[     next()      ]
[ Process Response ]
```
এটাই মূল **key concept**। প্রতিটা middleware আসলে ৩টা অংশে কাজ করে:

1. **Process Request** — request আসলে middleware প্রথমে তার নিজের কাজ করে (যেমন: log লেখা, header চেক করা, authentication করা)।
2. **`next()`** — নিজের কাজ শেষে middleware `next()` নামে একটা delegate/function call করে, যেটা request-টাকে **পরবর্তী middleware**-এর কাছে পাঠিয়ে দেয়। (`next()` না ডাকলে pipeline এখানেই থেমে যাবে — একে বলে **short-circuiting**)।
3. **Process Response** — পরের middleware(গুলো) তাদের কাজ শেষ করে response ফেরত পাঠালে, সেই response আবার এই middleware-এর মধ্য দিয়ে ফিরে আসার সময় middleware আরেকবার কাজ করার সুযোগ পায় (যেমন: response-এ কিছু header যোগ করা)।

এই কারণে middleware pipeline-কে অনেকটা **"U" shape** বা nested function-এর মতো কল্পনা করা হয় — request pipeline-এ ভেতরের দিকে যায়, আবার response pipeline থেকে বাইরের দিকে বের হয়।

---

## ৩. Middleware কীভাবে কাজ করে (How it Works — Step by Step)

1. Browser থেকে HTTP request পাঠানো হয়।
2. **Kestrel** (web server) request-টা গ্রহণ করে এবং একটা **HttpContext** তৈরি করে।
3. HttpContext টা **Middleware Pipeline**-এ ঢোকে।
4. **Middleware 1** প্রথমে তার কাজ করে (Process Request অংশ), তারপর `next()` কল করে Middleware 2-কে request পাঠায়।
5. **Middleware 2** একইভাবে তার কাজ করে এবং `next()` দিয়ে Middleware 3-কে পাঠায়।
6. **Middleware 3** (সাধারণত এটাই শেষ middleware বা Endpoint/Controller) request প্রসেস করে একটা **response** তৈরি করে।
7. সেই response এবার উল্টো দিকে ফিরে আসে — Middleware 3 → Middleware 2 (Process Response অংশ কাজ করে) → Middleware 1 (Process Response অংশ কাজ করে)।
8. সবশেষে সম্পূর্ণ response HttpContext সহ **Kestrel**-এ পৌঁছায়, এবং Kestrel সেটা **Browser**-এ পাঠিয়ে দেয়।

---

## ৪. C# কোডে Middleware কেমন দেখতে হয়

### Program.cs-এ Built-in Middleware ব্যবহার
```csharp
var app = builder.Build();

app.UseExceptionHandler("/Error");   // Middleware 1
app.UseAuthentication();             // Middleware 2
app.UseAuthorization();              // Middleware 3
app.UseRouting();                    // Middleware 4

app.Run();
```
> এখানে `Use...()` মেথডগুলোর order (ক্রম) খুবই গুরুত্বপূর্ণ, কারণ pipeline ঠিক এই ক্রম অনুযায়ীই চলে।

### Custom Middleware (Inline)
```csharp
app.Use(async (context, next) =>
{
    // Process Request
    Console.WriteLine("Request আসছে: " + context.Request.Path);

    await next(); // পরের middleware-কে কল করা হচ্ছে

    // Process Response
    Console.WriteLine("Response পাঠানো হচ্ছে: " + context.Response.StatusCode);
});
```

### Custom Middleware (Class হিসেবে — বাস্তব প্রজেক্টে যেভাবে করা হয়)
```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;

    public LoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Process Request
        Console.WriteLine($"Request শুরু: {context.Request.Path}");

        await _next(context); // next middleware কল হচ্ছে

        // Process Response
        Console.WriteLine($"Response শেষ: {context.Response.StatusCode}");
    }
}
```
এবং `Program.cs`-এ এটাকে যুক্ত করতে হয়:
```csharp
app.UseMiddleware<LoggingMiddleware>();
```

---

## ৫. গুরুত্বপূর্ণ পয়েন্ট (Key Points — মনে রাখার মতো)

| বিষয় | ব্যাখ্যা |
|---|---|
| **HttpContext** | প্রতিটা request/response সম্পর্কিত তথ্য বহন করে; এটাই pipeline-এর মধ্য দিয়ে ঘুরে বেড়ায় |
| **next()** | পরের middleware-কে call করার জন্য ব্যবহৃত delegate |
| **Short-circuit** | কোনো middleware যদি `next()` কল না করে, তাহলে pipeline সেখানেই থেমে যায় (যেমন: Authentication ব্যর্থ হলে) |
| **Order matters** | Middleware যোগ করার ক্রম অনুযায়ীই তারা execute হয় — ক্রম ভুল হলে অ্যাপ ভুলভাবে কাজ করতে পারে |
| **Pipeline shape** | Request যাওয়ার সময় এক দিকে, response ফেরার সময় উল্টো দিকে — অনেকটা nested (U-shaped) স্ট্রাকচার |
| **Kestrel** | ASP.NET Core-এর built-in, cross-platform web server, যেটা Browser আর Middleware Pipeline-এর মাঝে কাজ করে |

---

## ৬. Middleware-এর সাধারণ উদাহরণ (Real-life Examples)

- **Authentication Middleware** — ইউজার লগইন করা কি না, চেক করে।
- **Authorization Middleware** — ইউজারের নির্দিষ্ট রিসোর্স access করার permission আছে কি না, দেখে।
- **Exception Handling Middleware** — কোনো error হলে সেটা catch করে সুন্দর error page/response দেখায়।
- **Logging Middleware** — প্রতিটা request/response এর log রাখে।
- **Routing Middleware** — কোন URL-এ কোন Controller/Action কল হবে, তা ঠিক করে।
- **Static Files Middleware** — CSS, JS, image ইত্যাদি ফাইল সরাসরি সার্ভ করে।

---

## ৭. সারাংশ (Summary)

- Middleware হলো ASP.NET Core-এর একটা **pipeline-based architecture**, যেখানে request একটার পর একটা component এর মধ্য দিয়ে পার হয়।
- প্রতিটা middleware **request process** করে, তারপর `next()` দিয়ে পরের middleware-কে call করে, এবং সবশেষে **response process** করে ফেরত পাঠায়।
- **Kestrel** সার্ভার Browser আর Middleware Pipeline-এর মধ্যে সেতুবন্ধন (bridge) হিসেবে কাজ করে, এবং **HttpContext** পুরো প্রসেসের central object।

## Visualization
<img width="626" height="164" alt="image" src="https://github.com/user-attachments/assets/da8bd595-3d0b-4523-8b54-9900e494e167" />
এবার একটা middleware-এর ভেতরে ঢুকে দেখা যাক — Process Request → next() → Process Response এই তিনটা ধাপ ঠিক কীভাবে কাজ করে:
<img width="1440" height="760" alt="image" src="https://github.com/user-attachments/assets/29871005-2cff-4eed-95fb-e9cf1acc7426" />

সংক্ষেপে ছবি দুটোর মানে:
- প্রথম ডায়াগ্রামে দেখা যাচ্ছে — Browser থেকে request Kestrel-এ যায়, Kestrel সেটা HttpContext আকারে Middleware Pipeline-এ পাঠায়, যেখানে 1 → 2 → 3 ক্রমে middleware-গুলো কাজ করে।
- দ্বিতীয় ডায়াগ্রামে একটা মাত্র middleware (এখানে Middleware 2) জুম করে দেখানো হয়েছে — এটা আগে request প্রসেস করে, তারপর `next()` কল করে পরের middleware-কে (Middleware 3) কাজ চালিয়ে যেতে দেয়, আর সেটা response ফেরত দিলে middleware আবার response প্রসেস করে (dashed তীর দিয়ে দেখানো হয়েছে ফেরার পথ)।

কোনো বক্সে ক্লিক করলে সেই অংশ নিয়ে আরও ব্যাখ্যা চাইতে পারেন। আগের markdown নোট ফাইলটাতেও এই একই ফ্লো লেখায় ব্যাখ্যা করা আছে, চাইলে দুটো একসাথে মিলিয়ে পড়তে পারেন।
