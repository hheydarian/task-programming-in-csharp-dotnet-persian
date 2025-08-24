# فصل پنجم: 📌 **مدیریت لغو عملیات (Cancellations)**

لغو عملیات یک مکانیزم حیاتی در برنامه‌نویسی با تسک‌ها است. این ویژگی در موارد زیر بسیار مفید است:
• متوقف کردن یک تسک در حال اجرا به‌طور ایمن وقتی دیگر مورد نیاز نیست ⏹️
• آزادسازی منابع حیاتی 🗄️
• بهبود پاسخ‌دهی برنامه ⚡

به همین دلیل، یک تسک طولانی‌مدت ممکن است به‌طور مرتب بررسی کند که آیا درخواست لغو ارسال شده است یا خیر. اگر چنین درخواستی وجود داشته باشد، تسک باید مطابق با آن واکنش نشان دهد.

با این حال، داشتن قابلیت لغو تسک به این معنا نیست که باید تسک را به‌طور ناگهانی متوقف کنید، زیرا این کار می‌تواند برنامه را در وضعیت ناپایدار قرار دهد. در عوض، شما یک مدل **همکاری‌کننده (cooperative)** ایجاد می‌کنید که در آن تسک و کدی که لغو را آغاز می‌کند می‌توانند با هم کار کنند.

این فصل به بررسی این موضوع می‌پردازد.

پیش‌نیازها 📚

برای مدیریت لغو تسک‌ها در C#، باید با موارد زیر آشنا باشید:

• **CancellationTokenSource**: این کلاس مسئول **اعلام درخواست لغو** است. این کلاس یک **CancellationToken** تولید می‌کند که به تسک داده می‌شود تا وضعیت درخواست لغو را بررسی کند. ⏳

• **CancellationToken**: این یک ساختار (struct) است که به تسک داده می‌شود و راهی برای بررسی اینکه آیا لغو درخواست شده وجود دارد یا خیر فراهم می‌کند. این توکن برای **انتشار اطلاعیه لغو تسک** استفاده می‌شود.

بیایید ببینیم چگونه از این‌ها در برنامه استفاده کنیم. ابتدا از کد زیر استفاده می‌کنیم:

```csharp
CancellationTokenSource tokenSource = new();
CancellationToken token = tokenSource.Token;
```

البته، با استفاده از کلیدواژه `var` می‌توانید کد معادل زیر را بنویسید:

```csharp
var tokenSource = new CancellationTokenSource();
var token = tokenSource.Token;
```

سپس این توکن را به تسک موردنظر می‌دهیم. همان‌طور که قبلاً در فصل ۲ و شکل ۲-۱ دیدید، سازنده Task چندین نسخه **overload** دارد و برخی از آن‌ها یک نمونه **CancellationToken** را به‌عنوان پارامتر می‌پذیرند. مثال:

```csharp
public Task(Action action, CancellationToken cancellationToken);
```

همچنین، متدهای `StartNew` در کلاس `TaskFactory` و `Run` در کلاس `Task` نیز overloadهای مشابهی دارند. چند نمونه دیگر:

```csharp
public Task StartNew(Action action, CancellationToken cancellationToken)
public static Task Run(Action action, CancellationToken cancellationToken)
public static Task<TResult> Run<TResult>(Func<TResult> function, CancellationToken cancellationToken)
```

این ساختارها به شما ایده می‌دهند که چگونه یک **توکن لغو** را به تسک منتقل کنید. برای مثال، کد زیر در نمونه بعدی استفاده می‌شود:

```csharp
var printTask = Task.Run(
    () =>
    {
        // برخی کدها که نشان داده نشده
    }, token
);
```

اما باید نکات زیر از مایکروسافت را به خاطر بسپارید:

> **رشته فراخوانی‌کننده (calling thread)** تسک را به‌صورت اجباری نمی‌بندد؛ تنها **اعلام می‌کند که لغو درخواست شده است**. اگر تسک در حال اجرا باشد، بر عهده کاربر است که این درخواست را مشاهده کرده و به آن پاسخ دهد.
> منبع: [Cancel a task and its children](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-cancel-a-task-and-its-children)

این نکته نشان می‌دهد که ممکن است وقتی رشته فراخوانی‌کننده درخواست لغو را اعلام می‌کند، تسک در حال اجرا قبلاً به پایان رسیده باشد. بنابراین، اگر می‌خواهید یک تسک در حال اجرا را لغو کنید، **باید درخواست لغو را هر چه سریع‌تر ارسال کنید**. ⚡

### لغو توسط کاربر (User-Initiated Cancellations)

در اغلب موارد، **درخواست لغو توسط کاربران** ایجاد می‌شود. همچنین می‌توان لغو را به‌صورت **خودکار بعد از یک بازه زمانی مشخص** انجام داد.
بیایید بحث را با **لغو توسط کاربر** شروع کنیم. 🧑‍💻
رویکرد اولیه 🔹

در اولین رویکرد، قبل از ارسال درخواست لغو، یک شرط **if** بررسی می‌شود. در صورت نیاز، می‌توانید قبل از لغو تسک، کارهای اضافی انجام دهید. برای مثال، می‌توانید پیامی چاپ کنید که نشان دهد این تسک قرار است لغو شود. همچنین می‌توانید منابع لازم را قبل از لغو تسک **پاکسازی (cleanup)** کنید. در نهایت، با استفاده از **break** یا **return** از بلوک کد مربوطه خارج می‌شوید. احتمالاً اکثر ما با این نوع مکانیزم **خروج نرم (soft exit)** آشنا هستیم. بیایید یک مثال ببینیم. 🛑

### نمونه عملی – Demonstration 1 🖥️

در این نمونه، یک تسک ایجاد شده که می‌تواند اعداد ۰ تا ۹۹ را چاپ کند. برای اینکه امکان لغو وجود داشته باشد، یک **CancellationTokenSource** ساخته شده تا **توکن لغو** ایجاد کرده و به تسک منتقل شود تا در صورت نیاز درخواست لغو ارسال شود.

**توجه:** امروزه پردازنده‌های کامپیوتر بسیار سریع هستند، بنابراین این تسک ممکن است خیلی سریع اجرا شود. برای جلوگیری از این موضوع، پس از چاپ هر عدد، یک تأخیر کوتاه اعمال شده است.

```csharp
using static System.Console;

WriteLine("Simple cancellation demonstration.");

var tokenSource = new CancellationTokenSource();
var token = tokenSource.Token;

var printTask = Task.Run(
    () =>
    {
        // حلقه‌ای که 100 بار اجرا می‌شود
        for (int i = 0; i < 100; i++)
        {
            // رویکرد شماره 1
            if (token.IsCancellationRequested)
            {
                WriteLine("Cancelling the print activity.");
                // انجام برخی پاکسازی‌ها، در صورت نیاز
                return;
            }
            WriteLine($"{i}");
            // ایجاد تأخیر کوتاه بعد از چاپ هر عدد
            Thread.Sleep(500);
        }
    }, token
);

WriteLine("Enter c to cancel the task.");
char ch = ReadKey().KeyChar;

if (ch.Equals('c'))
{
    WriteLine("\nRaising the cancellation request.");
    tokenSource.Cancel();
}

try
{
    printTask.Wait();
    //printTask.Wait(token); // این خط در ادامه استفاده خواهد شد
}
catch (OperationCanceledException oce)
{
    WriteLine($"Operation canceled. Message: {oce.Message}");
}
catch (AggregateException ae)
{
    foreach (Exception e in ae.InnerExceptions)
    {
        WriteLine($"Caught: {e.GetType()}, Message: {e.Message}");
    }
}

WriteLine($"The final status of printTask is: {printTask.Status}");
WriteLine("End of the main thread.");
```

در این برنامه، شما مشاهده می‌کنید که تسک به‌صورت **همکارانه (cooperative)** لغو می‌شود و وضعیت نهایی آن در پایان گزارش می‌شود. ✅
### توضیح خروجی و Q\&A 📝

در **Demonstration 1**، حتی وقتی تسک لغو شد، وضعیت نهایی آن **RanToCompletion** بود و نه **Canceled**. این به این دلیل است که ما تسک را با **return** ساده خاتمه دادیم، نه با پرتاب **OperationCanceledException**.

Microsoft در مستندات خود توضیح می‌دهد:

* **راه اول:** بازگشت از delegate کافی است، اما در این حالت تسک به وضعیت `RanToCompletion` می‌رود.
* **راه دوم (ترجیحی):** پرتاب **OperationCanceledException** با پاس دادن توکن لغو. در این حالت تسک به وضعیت **Canceled** می‌رود و می‌توان از آن در کد فراخوان برای بررسی پاسخ‌دهی تسک به درخواست لغو استفاده کرد.

### Alternative Approach – Demonstration 2 🔄

در این روش، داخل تسک، به جای return ساده، یک **OperationCanceledException** پرتاب می‌کنیم:

```csharp
var printTask = Task.Run(
    () =>
    {
        for (int i = 0; i < 100; i++)
        {
            // Approach-2
            if (token.IsCancellationRequested)
            {
                WriteLine("Cancelling the print activity.");
                // انجام پاکسازی‌های لازم
                throw new OperationCanceledException(token);
            }
            WriteLine($"{i}");
            Thread.Sleep(500);
        }
    }, token
);
```

### نمونه خروجی با این رویکرد

```
Simple cancellation demonstration.
Enter c to cancel the task.
0
1
2
c
Raising the cancellation request.
Cancelling the print activity.
Caught: System.Threading.Tasks.TaskCanceledException, Message: A task was canceled.
The final status of printTask is: Canceled
End of the main thread.
```

✅ همان‌طور که مشاهده می‌کنید، حالا وضعیت نهایی تسک **Canceled** است و استثناء `TaskCanceledException` دریافت می‌شود.

این روش برای مواقعی مناسب است که می‌خواهید مطمئن شوید تسک به صورت رسمی به حالت **لغو شده** منتقل شده و کد فراخوان بتواند آن را تشخیص دهد.
### کوتاه‌سازی کد با `ThrowIfCancellationRequested` ⏱️

Microsoft توصیه می‌کند که به جای نوشتن دستی این خطوط:

```csharp
if (token.IsCancellationRequested)
    throw new OperationCanceledException(token);
```

می‌توانید از متد **`ThrowIfCancellationRequested`** استفاده کنید که معادل عملکرد بالا است و هم بررسی می‌کند که آیا درخواست لغو شده و هم در صورت نیاز استثناء مناسب را پرتاب می‌کند.

### مثال کوتاه‌شده – Approach-3

```csharp
var printTask = Task.Run(
    () =>
    {
        for (int i = 0; i < 100; i++)
        {
            // Approach-3
            token.ThrowIfCancellationRequested();
            WriteLine($"{i}");
            Thread.Sleep(500);
        }
    }, token
);
```

این روش نه تنها کوتاه‌تر و تمیزتر است، بلکه استاندارد **پرکاربرد** در برنامه‌های واقعی محسوب می‌شود. ✅

---

### نکات مهم Q\&A

**Q5.2 – تفاوت RanToCompletion و Canceled:**

* اگر فقط از `return` برای خروج استفاده کنید، وضعیت تسک **RanToCompletion** می‌شود.
* اگر `OperationCanceledException` پرتاب کنید (Approach-2 یا Approach-3)، وضعیت تسک **Canceled** می‌شود.
* در برنامه‌های سازمانی، حالت دوم ترجیح داده می‌شود چون امکان بررسی و ثبت لاگ تسک‌های لغو شده فراهم می‌شود.

**Q5.3 – انجام پاکسازی قبل از لغو تسک:**
می‌توانید از ترکیب بررسی `IsCancellationRequested` و سپس `ThrowIfCancellationRequested` استفاده کنید:

```csharp
if (token.IsCancellationRequested)
{
    // انجام پاکسازی‌های لازم
    token.ThrowIfCancellationRequested();
}
```

---

### نکته منابع و Dispose 💡

* همیشه پس از پایان کار با `CancellationTokenSource`، باید **`Dispose()`** را فراخوانی کنید تا منابع آزاد شوند:

```csharp
tokenSource.Dispose();
```

* این کلاس `IDisposable` را پیاده‌سازی می‌کند و در غیر این صورت منابع تا فراخوانی Garbage Collector آزاد نمی‌شوند.

---

### نمونه‌های دیگر `OperationCanceledException`

| Constructor                                     | توضیح                                           |
| ----------------------------------------------- | ----------------------------------------------- |
| `OperationCanceledException(CancellationToken)` | استفاده همراه با توکن لغو (مثل Demonstration 2) |
| `OperationCanceledException(String)`            | ایجاد استثناء با پیام دلخواه                    |
| `OperationCanceledException()`                  | پیام پیش‌فرض سیستم                              |

این انعطاف به شما امکان می‌دهد هنگام لغو تسک، جزئیات دلخواه را ثبت کنید و مدیریت بهتری روی عملیات لغو داشته باشید.
### بررسی تأثیر تغییر نحوه پرتاب و مدیریت استثناء لغو

---

### **Case Study 1 – تغییر تعریف تسک**

در Demonstration 2 (Approach-2)، اگر `OperationCanceledException` را بدون پاس دادن **CancellationToken** بسازید، مانند این:

```csharp
if (token.IsCancellationRequested)
{
    WriteLine("Cancelling the print activity.");
    // Do some cleanups, if required
    throw new OperationCanceledException("The operation is canceled.");
}
```

**خروجی نمونه:**

```
Simple cancellation demonstration.
Enter c to cancel the task.
0
1
2
c
Raising the cancellation request.
Cancelling the print activity.
Caught: System.OperationCanceledException, Message: The operation is canceled.
The final status of printTask is: Faulted
End of the main thread.
```

✅ **توضیح:**

* وقتی `OperationCanceledException` بدون توکن ایجاد شود، یا توکن آن با تسک مطابقت نداشته باشد، **تسک به جای حالت Canceled، به حالت Faulted می‌رود**.
* این رفتار توسط طراحی .NET تعیین شده است. منبع رسمی: [Task cancellation](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/task-cancellation)

> خلاصه: برای داشتن وضعیت `Canceled`، باید **توکن همان تسک** به استثناء داده شود.

---

### **Case Study 2 – تغییر نحوه صدا زدن Wait**

اگر از `Wait(token)` به جای `Wait()` استفاده کنید:

```csharp
// printTask.Wait();
printTask.Wait(token);
```

**تأثیر:**

* این بار، `OperationCanceledException` داخل `AggregateException` قرار نمی‌گیرد.
* بنابراین، لازم است catch مجزا برای `OperationCanceledException` داشته باشید تا آن را مدیریت کنید.

**خروجی نمونه:**

```
Simple cancellation demonstration.
Enter c to cancel the task.
0
1
2
c
Raising the cancellation request.
Operation canceled. Message: The operation was canceled.
The final status of printTask is: Running
End of the main thread.
```

✅ **نکات کلیدی:**

1. پرتاب استثناء با توکن مناسب → تسک وضعیت `Canceled` می‌گیرد.
2. پرتاب استثناء بدون توکن → تسک وضعیت `Faulted` می‌گیرد.
3. استفاده از `Wait(token)` → استثناء لغو مستقیماً مدیریت می‌شود و در `AggregateException` جمع‌آوری نمی‌شود.

---

این دو مطالعه موردی نشان می‌دهد که **نحوه پرتاب و مدیریت استثناء لغو و همچنین توکن مورد استفاده، تأثیر مستقیم روی وضعیت نهایی تسک دارند**.

### **Q5.5 – چرا وضعیت نهایی تسک Running نشان داده شد؟**

در مثال قبلی که از `Wait(token)` استفاده شد، خروجی نهایی تسک `Running` بود، نه `Canceled` یا `Faulted`.

✅ **توضیح:**

* وقتی از `Wait(token)` استفاده می‌کنید، انتظار اصلی تسک به وسیله **توکن لغو** کنترل می‌شود.
* اگر کاربر قبل از اتمام تسک، لغو را درخواست کند، **main thread سریعاً از Wait خارج می‌شود** و بنابراین هنوز تسک ممکن است در حال اجرا باشد.
* در نتیجه، اگر بلافاصله وضعیت تسک را بررسی کنید، مقدار `Running` دیده می‌شود چون تسک هنوز کامل نشده است.

💡 **راه حل برای دیدن وضعیت واقعی تسک:**

می‌توانید قبل از بررسی وضعیت، **منتظر شوید تا تسک واقعاً کامل شود**:

```csharp
// Wait till the task finishes the execution
while (!printTask.IsCompleted) { }

WriteLine($"The final status of printTask is: {printTask.Status}");
```

**خروجی نمونه پس از این تغییر:**

```
Simple cancellation demonstration.
Enter c to cancel the task.
0
1
2
c
Raising the cancellation request.
Operation canceled. Message: The operation was canceled.
Cancelling the print activity.
The final status of printTask is: Faulted
End of the main thread.
```

✅ **نکته کلیدی:**

* `Wait(token)` باعث خروج زودهنگام main thread می‌شود، اما تسک ممکن است هنوز کامل نشده باشد.
* برای مشاهده وضعیت نهایی دقیق، باید تا اتمام تسک صبر کنید (`IsCompleted`).
<div align="center">
    
![Conventions-UsedThis-Book](../../assets/image/05/Table%205-1.jpeg) 
</div>

نکات مهم 📝

می‌خواهم به نکات زیر توجه کنید:

1️⃣ با بررسی دقیق خواهید دید که متد `Wait()` تنها می‌تواند `AggregateException` ایجاد کند، در حالی که `Wait(CancellationToken cancellationToken)` قابل لغو است و می‌تواند `OperationCanceledException` را ایجاد کند. اگر علاقه‌مند هستید، می‌توانید بحث آنلاین ما در این مورد را در لینک زیر مشاهده کنید:
[StackOverflow Discussion](https://stackoverflow.com/questions/77833724/why-the-catch-block-of-aggregateexception-was-not-sufficient-to-handle-cancellat/77833858#77833858)

2️⃣ برای پاسخ به سؤال Q5.5، من از خط `while (!printTask.IsCompleted) { }` استفاده کردم. با این حال، مایکروسافت توصیه می‌کند (لینک آنلاین: [Microsoft Documentation](https://learn.microsoft.com/en-us/dotnet/standard/parallel-programming/exception-handling-task-parallel-library)) که از چنین polling در کدهای تولیدی خودداری کنید، زیرا بسیار ناکارآمد است.

3️⃣ در خروجی قبلی، وضعیت نهایی تسک (`task`) به صورت `Faulted` نمایش داده شد. دلیل آن این است که من در آن دمو از خط زیر استفاده کردم:

```csharp
throw new OperationCanceledException("The operation is canceled.");
```

با این حال، اگر این خط را با

```csharp
throw new OperationCanceledException(token);
```

جایگزین کنید، وضعیت نهایی `Canceled` خواهد بود و نه `Faulted`.

⏱️ لغو با تایم‌اوت (Timeout Cancellation)

شما می‌توانید یک درخواست لغو را پس از یک بازه زمانی مشخص ایجاد کنید. به عنوان مثال، در یک عملیات شبکه معمولی، ممکن است نخواهید بی‌نهایت منتظر بمانید. در چنین حالتی، برنامه شما می‌تواند به‌صورت خودکار درخواست لغو را صادر کند.

برای پیاده‌سازی این ایده، می‌توانید از خط زیر استفاده کنید:

```csharp
tokenSource.CancelAfter(2000);
```

در دمو قبلی (Demonstration 2) به شکل زیر:

```csharp
// هیچ تغییری در کد قبلی نیست
var tokenSource = new CancellationTokenSource();
var token = tokenSource.Token;
tokenSource.CancelAfter(2000);
// هیچ تغییر دیگری در کد باقی‌مانده نیست
```

اکنون با اجرای برنامه‌ی اصلاح‌شده، برنامه می‌تواند پس از ۲۰۰۰ میلی‌ثانیه به‌صورت خودکار درخواست لغو را صادر کند.

💡 توجه: از آنجا که من بقیه کد را تغییر نداده‌ام، این برنامه همچنان می‌تواند به لغوهای ایجادشده توسط کاربر نیز پاسخ دهد. در این حالت، کاربر باید قبل از فعال شدن لغو خودکار، درخواست لغو را صادر کند. در واقع، برنامه تا دریافت ورودی از کاربر منتظر می‌ماند قبل از اینکه بسته شود. می‌توانید پروژه `Chapter5_TimeoutCancellation` را از وب‌سایت Apress دانلود کنید تا برنامه کامل را ببینید.

🖥️ نظارت بر لغو تسک (Monitoring Task Cancellation)

در خروجی برخی از دموهای قبلی (مثلاً خروجی Demonstration2، Chapter5\_Demo2\_CaseStudy1، یا پاسخ به Q5.5)، شما خط زیر را مشاهده کردید:

```
Cancelling the print activity.
```

من از این خط برای نظارت بر تسک لغو شده قبل از عملیات لغو استفاده کردم. جالب است که روش‌های جایگزین دیگری نیز وجود دارد. بیایید برخی از آن‌ها را ببینیم.

🔔 استفاده از Register

شما می‌توانید در یک رویداد ثبت‌نام کنید. به عنوان مثال، در کد زیر یک delegate ثبت می‌کنیم که هنگام لغو شدن token فراخوانی می‌شود:

```csharp
token.Register(
    () =>
    {
        WriteLine("Cancelling the print activity. [Using event subscription]");
        // اگر خواستید کار دیگری انجام دهید
    }
);
```
استفاده از `WaitHandle.WaitOne` ⏳

اجازه دهید یک روش دیگر را نشان دهم که نسبت به روش قبلی کمی پیچیده‌تر است. با این حال، این روش نیز می‌تواند ایده‌ای درباره‌ی نظارت بر لغو تسک به شما بدهد. لینک آنلاین [Microsoft Documentation](https://learn.microsoft.com/en-us/dotnet/api/system.threading.waithandle.waitone?view=net-8.0) متد `WaitOne` کلاس `WaitHandle` را به صورت زیر توضیح می‌دهد:

> بلوک کردن ترد فعلی تا زمانی که `WaitHandle` فعلی سیگنالی دریافت کند.

متد `WaitOne` چندین اورلود دارد. در دمو پیش‌رو، ساده‌ترین شکل آن را نشان می‌دهم که نیازی به ارسال هیچ آرگومانی ندارد. ایده اصلی این است که ترد فعلی یک token را در نظر می‌گیرد و منتظر می‌ماند تا کسی این token را لغو کند. به محض اینکه کسی عملیات لغو را فراخوانی کند، تماس بلاک‌شده آزاد می‌شود. به همین دلیل می‌توانم یک تسک دیگر از ترد فراخواننده به شکل زیر اجرا کنم:

```csharp
Task.Run(
    () =>
    {
        token.WaitHandle.WaitOne();
        WriteLine("Cancelling the print activity. [Using WaitHandle]");
        // اگر خواستید کار دیگری انجام دهید
    }
);
```

توجه کنید که این روش بسیار شبیه به ثبت در رویداد (event subscription) است، زیرا در اینجا نیز منتظر وقوع لغو هستید. به همین دلیل من یک دستور مشابه در این بلاک کد نوشتم.

---

📌 **Demonstration 3**

وقت آن است که دمو دیگری را ببینیم که در آن روش‌های بحث‌شده برای نظارت بر عملیات لغو را نشان می‌دهد. تغییرات کلیدی با **بولد** مشخص شده‌اند:

```csharp
using static System.Console;

WriteLine("Monitoring the cancellation operation.");

var tokenSource = new CancellationTokenSource();
var token = tokenSource.Token;

token.Register(
    () =>
    {
        WriteLine("Cancelling the print activity.[Using event subscription]");
        // اگر خواستید کار دیگری انجام دهید
    }
);

var printTask = Task.Run
(
    () =>
    {
        // حلقه‌ای که 100 بار اجرا می‌شود
        for (int i = 0; i < 100; i++)
        {
            // Approach-3
            token.ThrowIfCancellationRequested();
            WriteLine($"{i}");
            // اضافه کردن تاخیر برای مشاهده بهتر
            Thread.Sleep(500);
        }
    }, token
);

Task.Run(
    () =>
    {
        token.WaitHandle.WaitOne();
        WriteLine("Cancelling the print activity.[Using WaitHandle]");
        // اگر خواستید کار دیگری انجام دهید
    }
);

WriteLine("Enter c to cancel the task.");
char ch = ReadKey().KeyChar;
if (ch.Equals('c'))
{
    WriteLine("\nTask cancellation requested.");
    tokenSource.Cancel();
}

// تا پایان اجرای تسک منتظر بمانید
while (!printTask.IsCompleted) { }
WriteLine($"The final status of printTask is: {printTask.Status}");
WriteLine("End of the main thread.");
```

---

📤 **خروجی نمونه**

توجه کنید تغییرات بولد شده‌اند:

```
Monitoring the cancellation operation.
Enter c to cancel the task.
0
1
2
c
Task cancellation requested.
Cancelling the print activity.[Using WaitHandle]
Cancelling the print activity.[Using event subscription]
The final status of printTask is: Canceled
End of the main thread.
```

---

🌐 **استفاده از چندین Cancellation Token**

یک برنامه می‌تواند به دلایل مختلف لغو شود. در چنین حالتی، می‌توانید از چندین token استفاده کنید و منطق لازم را اعمال کنید. در این زمینه می‌توانید از متد `CreateLinkedTokenSource` استفاده کنید.

در دمو زیر، دو token مختلف ایجاد می‌کنیم:

```csharp
var normalCancellation = new CancellationTokenSource();
var tokenNormal = normalCancellation.Token;

var unexpectedCancellation = new CancellationTokenSource();
var tokenUnexpected = unexpectedCancellation.Token;
```

پس از ایجاد، آن‌ها را به متد `CreateLinkedTokenSource` می‌دهیم:

```csharp
var compositeToken = CancellationTokenSource.CreateLinkedTokenSource(tokenNormal, tokenUnexpected);
```

ایده این است که شما می‌توانید با لغو هر یک از `normalCancellation` یا `unexpectedCancellation`، تسک نهایی را لغو کنید.

توجه داشته باشید که متد `CreateLinkedTokenSource` اورلودهای مختلفی دارد و در صورت نیاز می‌توانید تعداد بیشتری token ارسال کنید. نکته‌ی اصلی همان است: می‌توانید هر یک از این token‌ها را لغو کنید تا وضعیت نهایی تسک `Canceled` شود.
📌 **Demonstration 4 – دمو چهارم**

در برنامه‌ی زیر، کاربر می‌تواند یک لغو معمولی (normal cancellation) را فعال کند. با این حال، شما می‌توانید یک لغو غیرمنتظره/اضطراری (unexpected/emergency cancellation) را نیز مشاهده کنید.

برای شبیه‌سازی لغو اضطراری، من از یک تولیدکننده‌ی عدد تصادفی استفاده می‌کنم. اگر عدد تصادفی برابر ۵ باشد، لغو اضطراری فعال خواهد شد.

کد کامل برنامه برای نمایش این ایده به شکل زیر است:

```csharp
using static System.Console;

WriteLine("Monitoring the cancellation operation.");

var normalCancellation = new CancellationTokenSource();
var tokenNormal = normalCancellation.Token;

var unexpectedCancellation = new CancellationTokenSource();
var tokenUnexpected = unexpectedCancellation.Token;

tokenNormal.Register(
    () =>
    {
        WriteLine("Processing a normal cancellation.");
        // اگر خواستید کار دیگری انجام دهید
    }
);

tokenUnexpected.Register(
    () =>
    {
        WriteLine("Processing an unexpected cancellation.");
        // اگر خواستید کار دیگری انجام دهید
    }
);

var compositeToken = CancellationTokenSource.CreateLinkedTokenSource(tokenNormal, tokenUnexpected);

var printTask = Task.Run
(
    () =>
    {
        // حلقه‌ای که 100 بار اجرا می‌شود
        for (int i = 0; i < 100; i++)
        {
            compositeToken.Token.ThrowIfCancellationRequested();
            WriteLine($"{i}");
            // اضافه کردن تاخیر برای مشاهده بهتر
            Thread.Sleep(500);
        }
    }, compositeToken.Token
);

// منطق ساده برای شبیه‌سازی لغو اضطراری
int random = new Random().Next(1, 6);
if (random == 5)
    unexpectedCancellation.Cancel();

WriteLine("Enter a key (type c for a normal cancellation)");
char ch = ReadKey().KeyChar;
if (ch.Equals('c'))
{
    WriteLine("\nTask cancellation requested.");
    normalCancellation.Cancel();
}

// تا پایان اجرای تسک منتظر بمانید
while (!printTask.IsCompleted) { }

WriteLine($"The final status of printTask is: {printTask.Status}");
WriteLine("End of the main thread.");
```

---

📤 **خروجی نمونه – لغو معمولی**
زمانی که کاربر “c” را فشار می‌دهد:

```
Monitoring the cancellation operation.
Enter a key (type c for a normal cancellation)
0
1
2
c
Task cancellation requested.
Processing a normal cancellation.
The final status of printTask is: Canceled
End of the main thread.
```

📤 **خروجی نمونه – لغو اضطراری**
زمانی که لغو اضطراری به‌صورت خودکار فعال می‌شود:

```
Monitoring the cancellation operation.
Processing an unexpected cancellation.
The final status of printTask is: Canceled
End of the main thread.
```
📌 **خلاصه – Summary**

لغو (Cancellation) یک مکانیزم اساسی در برنامه‌نویسی Task است. با این حال، به جای متوقف کردن ناگهانی یک تسک، شما یک مدل همکاری ایجاد می‌کنید که در آن تسک و کدی که لغو را آغاز می‌کند (calling code) با هم کار می‌کنند تا سلامت برنامه شما حفظ شود.

این فصل به این موضوع پرداخت و به سؤالات زیر پاسخ داد:

* چگونه می‌توان لغوهای مبتنی بر کاربر (user-initiated cancellations) را پشتیبانی کرد؟ 🤚
* چگونه می‌توان لغوهای مبتنی بر زمان (timeout cancellations) را پشتیبانی کرد؟ ⏱️
* چگونه می‌توان لغوها را در برنامه خود مانیتور کرد؟ 👀
* چگونه می‌توان از چندین CancellationToken در برنامه خود استفاده کرد؟ 🔗
* 
📝 **تمرین‌ها – Exercises**

برای سنجش درک خود، تمرین‌های زیر را انجام دهید:
<div align="center">
    
![Conventions-UsedThis-Book](../../assets/image/05/Table%205-2.jpeg) 
</div>

🔔 **یادآوری – Reminder**

همان‌طور که قبلاً گفته شد، می‌توانید با اطمینان فرض کنید که همه‌ی namespaceهای لازم برای این قطعه‌های کد در دسترس هستند. این نکته برای همه‌ی تمرین‌های کتاب نیز صادق است.

---

📝 **تمرین‌ها – Exercises**

**E5.1**
اگر کد زیر را اجرا کنید، آیا می‌توانید خروجی آن را پیش‌بینی کنید؟

```csharp
using static System.Console;

var tokenSource = new CancellationTokenSource();
var token = tokenSource.Token;

var printTask = Task.Run(
    () =>
    {
        int i = 0;
        while (i != 10)
        {
            if (token.IsCancellationRequested)
            {
                WriteLine("Cancelling the print activity.");
                return;
            }
            // در صورت نیاز، انجام کارهای دیگر
            Thread.Sleep(1000);
            i++;
        }
    }, token
);

Thread.Sleep(500);
WriteLine("The cancellation is initiated.");
tokenSource.Cancel();

// منتظر بمانید تا Task کامل شود
while (!printTask.IsCompleted) { }

WriteLine($"The final status of printTask is: {printTask.Status}");
WriteLine("End of the main thread.");
```

---

**E5.2**
در تمرین قبلی، قطعه کد زیر:

```csharp
if (token.IsCancellationRequested)
{
    WriteLine("Cancelling the print activity.");
    return;
}
```

را با این خط جایگزین کنید:

```csharp
token.ThrowIfCancellationRequested();
```

آیا تغییر خروجی اتفاق می‌افتد؟ 🤔

---

**E5.3**
برنامه‌ی زیر یک Task والد و یک Task تو در تو (nested) ایجاد می‌کند. همچنین امکان لغو این Taskها با فشار دادن کلید “c” فراهم شده است. آیا می‌توانید خروجی آن را پیش‌بینی کنید؟

```csharp
using static System.Console;
WriteLine("Exercise 5.3");

var tokenSource = new CancellationTokenSource();
var token = tokenSource.Token;
Task child = null;

var parent = Task.Factory.StartNew(
    () =>
    {
        Thread.Sleep(1000);
        if (token.IsCancellationRequested)
        {
            WriteLine("The cancellation request is raised too early.");
            token.ThrowIfCancellationRequested();
        }

        WriteLine("The parent task is running.");
        // ایجاد Task تو در تو
        child = Task.Factory.StartNew(
            () =>
            {
                WriteLine("The child task has started.");
                for (int i = 0; i < 10; i++)
                {
                    token.ThrowIfCancellationRequested();
                    WriteLine($"\tThe nested task prints:{i}");
                    Thread.Sleep(200);
                }
                return "The child task has finished too";
            },
            token,
            TaskCreationOptions.AttachedToParent,
            TaskScheduler.Default
        );

        child.Wait(token);
    }, token
);

WriteLine("Enter c to cancel the nested task.");
char ch = ReadKey().KeyChar;

if (ch.Equals('c'))
{
    WriteLine("\nTask cancellation requested.");
    tokenSource.Cancel();
}

try
{
    parent.Wait();
}
catch (AggregateException ae)
{
    foreach (Exception e in ae.InnerExceptions)
    {
        WriteLine($"Caught error: {e.Message}");
    }
}

WriteLine($"The current state of the parent task: {parent.Status}");
string childStatus = child != null ? child.Status.ToString() : "not created";
WriteLine($"The current state of the child task: {childStatus}");
WriteLine("End of the main thread.");
```

---

**E5.4** – درست/نادرست:
i) کلاس `CancellationTokenSource` یک کلاس است که اینترفیس `IDisposable` را پیاده‌سازی می‌کند. ✅ / ❌
ii) ویژگی `Token` از کلاس `CancellationTokenSource` برای ایجاد نمونه‌ی `CancellationToken` استفاده می‌شود. ✅ / ❌

---

**E5.5**
در فصل ۳، شما تمرین E3.2 را حل کرده‌اید. اکنون با توجه به اینکه با سناریوهای استثنا (Exception) و لغو Task (Cancellation) آشنا شده‌اید، آیا می‌توانید همان تمرین را دوباره با در نظر گرفتن این سناریوها حل کنید؟ 💡

💡 **راه‌حل تمرین‌ها – Solutions to Exercises**

---

### **E5.1**

برنامه به‌طور خودکار لغو (cancellation) را اجرا می‌کند. یک خروجی احتمالی به شکل زیر است (توجه کنید که وضعیت نهایی Task برابر **RanToCompletion** است و نه **Canceled**):

```
The cancellation is initiated.
Cancelling the print activity.
The final status of printTask is: RanToCompletion
End of the main thread.
```

---

### **E5.2**

این بار وضعیت نهایی Task باید به صورت **Canceled** نمایش داده شود. نمونه خروجی:

```
The cancellation is initiated.
The final status of printTask is: Canceled
End of the main thread.
```

---

### **E5.3**

همان‌طور که می‌دانید، این برنامه یک Task والد و یک Task تو در تو (nested) ایجاد می‌کند و به شما اجازه می‌دهد با فشار دادن کلید “c” Task تو در تو را لغو کنید. بنابراین، بسته به زمان لغو، خروجی متفاوت خواهد بود:

**اگر لغو انجام نشود و در پایان Enter بزنید، خروجی ممکن است به شکل زیر باشد:**

```
Exercise 5.3
Enter c to cancel the nested task.
The parent task is running.
The child task has started.
The nested task prints:0
The nested task prints:1
The nested task prints:2
The nested task prints:3
The nested task prints:4
The nested task prints:5
The nested task prints:6
The nested task prints:7
The nested task prints:8
The nested task prints:9
The current state of the parent task: RanToCompletion
The current state of the child task: RanToCompletion
End of the main thread.
```

**اگر کلید “c” تقریباً در ابتدای برنامه فشار داده شود، خروجی ممکن است چنین باشد:**

```
Exercise 5.3
Enter c to cancel the nested task.
c
Task cancellation requested.
The cancellation request is raised too early.
Caught error: A task was canceled.
The current state of the parent task: Canceled
The current state of the child task: not created
End of the main thread.
```

**و یا یک لغو عادی که به شکل زیر است:**

```
Exercise 5.3
Enter c to cancel the nested task.
The parent task is running.
The child task has started.
The nested task prints:0
The nested task prints:1
c
Task cancellation requested.
Caught error: A task was canceled.
The current state of the parent task: Canceled
The current state of the child task: Canceled
End of the main thread.
```

---

### **E5.4** – درست/نادرست ✅❌

i) کلاس `CancellationTokenSource` یک کلاس است که اینترفیس `IDisposable` را پیاده‌سازی می‌کند. **\[True ✅]**
ii) ویژگی `Token` از کلاس `CancellationTokenSource` برای ایجاد نمونه‌ی `CancellationToken` استفاده می‌شود. **\[True ✅]**

---

### **E5.5**

این تمرین را هم‌اکنون به خودتان واگذار می‌کنم. موفق باشید! 🍀

💡 **توجه اضافی:** از این به بعد، هنگام حل تمرین‌ها، می‌توانید مکانیزم‌های لغو (cancellation) و مدیریت استثنا (exception) را هم اعمال کنید. همین نکته برای تمرین‌هایی که در فصل‌های قبلی حل کرده‌اید نیز صادق است.
