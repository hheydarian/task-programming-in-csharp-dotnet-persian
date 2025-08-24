# فصل ششم: موارد اضافی 📝

در این فصل، چند موضوع تکمیلی درباره **برنامه‌نویسی Task** بررسی می‌شود.

---

### گزارش پیشرفت (Progress Reporting) 📊

حتماً دیده‌اید هنگام **به‌روزرسانی سیستم‌عامل** یا **نصب نسخه جدید Visual Studio** روی یک کامپیوتر، وضعیت پیشرفت کار نمایش داده می‌شود. حالا بیایید ببینیم آیا می‌توانید با استفاده از **برنامه‌نویسی Task** یک برنامه با ویژگی مشابه بسازید؟

---

### درک نیاز (Understanding the Need) 🤔

فرض کنید در حال پردازش تعداد زیادی **رکورد (Record)** مختلف هستید. برای شبیه‌سازی شرایط واقعی، تصور کنید که **زمان پردازش هر رکورد متفاوت است**. از آن‌جایی که این عملیات ممکن است **زمان‌بر** باشد، نمی‌خواهید **رشته اصلی (Main Thread)** مسدود شود. در عوض تصمیم می‌گیرید این کار را از طریق یک **Task پس‌زمینه** انجام دهید.

---

### نمایش اولیه (Demonstration 1) 🖥️

با این حال، اگر هنگام پردازش این رکوردها **وضعیت پیشرفت را نمایش ندهید**، کاربر ممکن است **سردرگم شود**. برای ساده‌تر کردن موضوع، بیایید یک نمونه آزمایشی ببینیم که فقط با **۵ رکورد** کار می‌کند:

```csharp
using static System.Console;

WriteLine("The main thread is initiating a task to process some records.");
var recordProcessingTask = Task.Run(ProcessRecords);

WriteLine("The main thread is doing other work now.");
recordProcessingTask.Wait();

static void ProcessRecords()
{
    WriteLine($"Starts processing the records...");
    for (int i = 1; i <= 5; i++)
    {
        // Varying the delay
        Thread.Sleep(i * 500);
    }
    WriteLine("All the records are processed.");
}
```

---

### خروجی (Output) 🖨️

نمونه‌ای از خروجی که جای تعجبی ندارد به این صورت است:

```
The main thread is initiating a task to process some records.
The main thread is doing other work now.
Starts processing the records...
All the records are processed.
```

---

### تحلیل (Analysis) 🔍

همان‌طور که مشاهده می‌کنید، خط **"All the records are processed."** بعد از **مدتی قابل توجه** نمایش داده می‌شود. از دید کاربر این رفتار **گیج‌کننده** است؛ چون بعد از دیدن جمله **"Starts processing the records..."** نمی‌دانید در پس‌زمینه چه اتفاقی در حال رخ دادن است.

اینجاست که متوجه می‌شوید **گزارش پیشرفت (Progress Reporting)** می‌تواند در چنین شرایطی بسیار مفید باشد، به‌ویژه هنگام اجرای **یک Task طولانی‌مدت**.

---

### چگونه پیشرفت را گزارش کنیم؟

این موضوع کاملاً به شما بستگی دارد. به‌عنوان مثال:

* می‌توانید به‌سادگی چاپ کنید که **کدام رکورد در حال پردازش است**.
* یا می‌توانید پیشرفت را به‌صورت **درصدی** نمایش دهید، مثل:

```
completed processing: 60%
```

در مثال بعدی، می‌خواهم یک **نمونه عملی** با استفاده از **ساختارهای داخلی برای گزارش پیشرفت** به شما نشان دهم. بیایید بحث را شروع کنیم.

---

### رابط IProgress و کلاس Progress<T> 🛠️

یک رابط (**Interface**) به نام **IProgress** در فضای نام **System** وجود دارد. این رابط یک متد به نام **Report** دارد که می‌تواند به شما کمک کند پیشرفت را نمایش دهید. نمای این رابط در Visual Studio به شکل زیر است:

```csharp
public interface IProgress<in T>
{
    //
    // Summary:
    //     Reports a progress update.
    //
    // Parameters:
    //   value:
    //     The value of the updated progress.
    void Report(T value);
}
```

با استفاده از این رابط، می‌توانید یک **پیاده‌سازی سفارشی** برای گزارش پیشرفت ایجاد کنید.

همچنین یک کلاس داخلی به نام **Progress<T>** وجود دارد که **رابط IProgress** را از قبل پیاده‌سازی کرده است. بنابراین می‌توانید از این کلاس برای رسیدن به هدف خود استفاده کنید.

این کلاس یک **رویداد (Event)** به نام **ProgressChanged** ارائه می‌دهد که می‌تواند با هر به‌روزرسانی پیشرفت از کد ناهمزمان **فعال شود**.

---

### استفاده از Delegate در Progress

یک نکته جالب دیگر این است که این کلاس **Progress** دارای دو **سازنده (Constructor)** است که یکی از آن‌ها به شکل زیر است:

```csharp
public Progress(Action<T> handler);
```

این نکته به شما سرنخی می‌دهد که می‌توانید یک **Delegate** را به عنوان **پارامتر سازنده** کلاس Progress ارسال کنید. این Delegate عملاً نقش یک **Event Handler** را دارد که می‌توانید از آن برای **گزارش پیشرفت** استفاده کنید.

بیایید **کد قبلی را به‌روزرسانی کنیم** تا ببینیم این کار چگونه انجام می‌شود.

ابتدا تابع **ProcessRecords** را تغییر دادم تا بتواند **IProgress<int>** را به عنوان یک **پارامتر** دریافت کند. بیایید نسخه اصلاح‌شده این تابع را ببینیم که تغییرات کلیدی آن **پررنگ** شده است.

<div align="center">
    
![Conventions-UsedThis-Book](../../assets/image/06/Table%206-1.jpeg) 
</div>
