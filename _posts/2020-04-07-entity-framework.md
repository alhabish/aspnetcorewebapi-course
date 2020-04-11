---
layout: post
title: "2. Entity Framework"
date: 2020-04-07
lang: ar-SA
index: 1
comments: true
---

الخدمة service التي سنقدمها تتعلق بالموظفين وكل ما يتغلق بالإضافة/ التعديل / الإستعلام عن بياناتهم. 

قاعدة البيانات المستخدمة هي Microsoft SQL Server والنسخة التي سنقوم بتحميلها SQL Server 2019 Developer وذلك من الرابط التالي:

[SQL Server 2019 Developer Edition](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)

عند الإنتهاء من ثبيت البرنامج إحفظ النص الموجود في الـ `CONNECTION STRING` جانباً حيث سنحتاج اليه فيما بعد:

{% include image.html url="assets/files/article_02/sql-server-installation-config.png" border="1" %}

`Server=localhost;Database=master;Trusted_Connection=True;`

ولإدارة قاعدة البيانات هذه والتعامل معها سنستخدم `Azure Data Studio` والتي يمكن تحميلها من الرابط التالي:

[Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio)

بعد تنصيب هذه البرامج نقوم بإنشاء مجلد جديد بإسم `Entities` وبداخله ملف Employee.cs:

```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace aspnetcorewebapiproject.Entities
{
    public class Employee
    {
        [Key]
        public int Id { get; set; }

        [Required]
        [StringLength(100)]
        public string FirstName { get; set; }
        
        [Required]
        [StringLength(100)]
        public string LastName { get; set; }
        
        public bool IsManager { get; set; }
        
        [DataType(DataType.Date)]
        public DateTime EnrollmentDate { get; set; }
        
        public DateTime CreatedDate { get; set; }
    }
}
```

لإضافة Entity Framework ننفذ الأوامر التالية:

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

نقوم الآن بإنشاء database context للتعامل مع قاعدة البيانات. ننشئ أولاً مجلد جديد بإسم `DbContexts` وبداخله الملف `MainDbContext` ومحتواه:

ثم نضيف التالي في آخر الدالة `ConfigureServices` في ملف `Startup.cs`:

```csharp
services.AddDbContext<DbContexts.MainDbContext>(options => options.UseSqlServer(Configuration.GetConnectionString("MainDbContext")));
```

وفي ملف appsettings.json نضيف إعدادات الإتصال بقاعدة البيانت ConnectionStrings بإسم MainDbContext وهنا نستفيد من معلومات الإتصال بقاعدة البيانات التي نسخناها سابقاً ولكننا سنستبدل قاعدة البيانات `master` بـ `MainDb`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "MainDbContext": "Server=localhost;Database=MainDb;Trusted_Connection=True;"
  }    
}
```

نضيف الآن أدة dotnet-ef

```csharp
dotnet tool install --global dotnet-ef
```

نضيف أوامر التعديل على قاعدة البيانات:

```csharp
dotnet ef migrations add InitialCreate
```

نلاحظ إنه تم إنشاء مجلد جديد بإسم `Migrations` وبه الأوامر التي سيتم تنفيذها للتعديل على قاعدة البيانات. نقوم الآن بتنفيذ الأومر:

```csharp
dotnet ef database update
```

نستخدم الآن `Azure Data Studio` لرؤية التعديلات التي تم تنفيذها. من القائمة في اليسار نختار `Connection` ثم `New Connection`:

{% include image.html url="assets/files/article_02/ads-new-connection.png" border="1" %}

سنقوم بعد ذلك بتعبئة معلومات الإتصال فى النافذة الجديدة:

{% include image.html url="assets/files/article_02/ads-new-connection-details.png" border="1" %}

والمعلومات هذه تم أخذها من التالي:

```json
  "ConnectionStrings": {
    "MainDbContext": "Server=localhost;Database=MainDb;Trusted_Connection=True;"
  }   
```

نضغط بعد ذلك على زر `Connect` ثم يظهر لنا أنه تم إنشاء جدولين جديدين في قاعدة البيانات `Employees`:

{% include image.html url="assets/files/article_02/ads-maindb-created.png" border="1" %}

* `_EFMigrationsHistory` وتحتفظ بكل التغييرات التي تمت على قاعدة البيانات هذه من Entity Framework
* `Employees` الجدول الذي يقابل الـ `class Employee` الذي أنشأناه في الكود

بإمكاننا أن نرى أن جدول `Employees` خالي ولم يتم تعبئته بالبيانات:

{% include image.html url="assets/files/article_02/ads-employees-select-top-1000.png" border="1" %}

### إضافة كنترولر جديد EmployeesController

سنستفيد من أداة الـ dotnet في توليد generate كنترولر جديد بإسم EmployeesController. وللقيام بذلك علينا أولاً إضافة الخاصية لهذه الأداة بتنفيذ الأوامر التالي:

```bash
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet tool install --global dotnet-aspnet-codegenerator
```

والآن سنقوم بإنشاء الكنترولر الجديد:

```bash
dotnet aspnet-codegenerator controller -name EmployeesController -async -api -m Employee -dc MainDbContext -outDir Controllers
```

وفيما يلي تفصيل للأجزاء المختلفة للأمر السابق:

| الأمر | القيمة | الوصف|
|---:|---:|---:|
| _name_ | EmployeesController |  إسم الكنترولر المراد إنشاؤه |
| _async_ |   | جعل الأوامر غير متزامنه asynchronous |
| _api_ |   | لعدم إنشاء Views |
| _m_ | Employee | إسم الـ model الذي سيستخدم |
| _dc_ | MainDbContext | الـ database context المستخدم |
| _outDir_ | Controllers | المجلد الذي سيتم وضع الكنترولر فيه |

بإمكاننا أن نرى بأنه تم إنشاء ملف جديد بإسم EmployeesController.cs داخل المجلد Controllers:

{% include image.html url="assets/files/article_02/vs-new-employeescontroller.png" border="1" %}

نلاحظ أنه تم إنشاء الـ actions التالية:

| Verb | Url| الوصف|
|---:|---:|---:|
| GET | api/employees | لإسترجاع قائمة بجميع الموظفين |
| GET | api/employees/{id} | لإسترجاع معلومات موظف معين |
| PUT | api/employees/{id} | لتعديل معلومات موظف |
| POST | api/employees | لإضافة معلومات موظف جديد |
| DELETE | api/employees/{id} | لحذف معلومات موظف |

هتالك تعديل بسيط يستحسن القيام به على الملف EmployeesController.cs في السطر 86 للإعتماد على الـ concrete types وليس على نص hard coded:

```csharp
return CreatedAtAction("GetEmployee", new { id = employee.Id }, employee);
```

لتصبح:

```csharp
return CreatedAtAction(nameof(GetEmployee), new { id = employee.Id }, employee);
```

لنقم الآن ببناء المشروع ثم تشغيله:

```bash
dotnet build
dotnet run
```

## تجربة الكنترولر الجديد EmployeesController

سنستخد postman للقيام بذلك، ولكن أول ما نقوم به هو الذهاب الى File ثم Settings والتأكد من قيمة الإعداد التالي:

`SSL certificate verification = OFF`

{% include image.html url="assets/files/article_02/postman-disable-ssl-certificate-verification.png" border="1" %}

### إضافة معلومات موظف جديد

عند القيام بالتجربة نرى أن الـ Http Status Code المسترجع هو 201 Created أي أنه تم إنشاء معلومات موظف جديد بشكل صحيح:

{% include image.html url="assets/files/article_02/postman-post-employees.png" border="1" %}

قم بإضافة معلومات موظف جديد لنتمكن من تكملة باقي الإختبارات:

{% include image.html url="assets/files/article_02/postman-post-employees-additional-employee.png" border="1" %}

### إسترجاع معلومات جميع الموظفين

نرى أن الـ Http Status Code المسترجع هو 200 Ok أي أنه تم تنفيذ العملية بشكل صحيح:

{% include image.html url="assets/files/article_02/postman-get-all-employees.png" border="1" %}

### إسترجاع معلومات موظف معين

نرى أن الـ Http Status Code المسترجع هو 200 Ok أي أنه تم تنفيذ العملية بشكل صحيح:

{% include image.html url="assets/files/article_02/postman-get-specific-employee.png" border="1" %}

### تعديل معلومات موظف معين

نرى أن الـ Http Status Code المسترجع هو  204 No Content أي أنه تم تنفيذ العملية بشكل صحيح:

{% include image.html url="assets/files/article_02/postman-update-employee-info.png" border="1" %}

بإمكانك الإستعلام عن معلومات الموظف لتتأكد بأنه تم تعديل المعلومات:

`https://localhost:5001/api/employees/2`




### حذف معلومات موظف معين

نرى أن الـ Http Status Code المسترجع هو  200 Ok أي أنه تم تنفيذ العملية بشكل صحيح:

{% include image.html url="assets/files/article_02/postman-delete-employee-info.png" border="1" %}

بإمكانك الإستعلام عن معلومات الموظف لتتأكد بأنه تم فعلاً حذف المعلومات المتعلقة به:

{% include image.html url="assets/files/article_02/postman-get-specific-employee-after-deletion.png" border="1" %}

نرى أن الـ Http Status Code المسترجع هو  404 Not Found أي أنه لم يتم إيجاد الموظف بهذا الرقم وهذا صحيح.