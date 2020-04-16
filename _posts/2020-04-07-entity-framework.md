---
layout: post
title: "2. Entity Framework"
date: 2020-04-07
lang: ar-SA
index: 2
comments: true
---

الخدمة web service التي سنقدمها تتعلق بالموظفين وكل ما يخص إضافة / تعديل / الإستعلام عن بياناتهم. قاعدة البيانات المستخدمة هي Microsoft SQL Server والنسخة التي سنقوم بتحميلها هي SQL Server 2019 Developer وذلك من الرابط التالي:

[SQL Server 2019 Developer Edition](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)

عند الإنتهاء من تثبيت البرنامج إحفظ النص الموجود في الـ CONNECTION STRING جانباً حيث سنحتاج اليه فيما بعد:

{% include image.html url="assets/files/article_02/sql-server-installation-config.png" border="1" %}

```csharp
Server=localhost;Database=master;Trusted_Connection=True;
```

ولإدارة قاعدة البيانات هذه والتعامل معها سنستخدم Azure Data Studio والتي يمكن تحميلها من الرابط التالي:

[Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio)

بعد تثبيت هذه البرامج نقوم بإنشاء مجلد جديد بإسم Entities وبداخله ملف Employee.cs:

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

ولإضافة Entity Framework ننفذ الأوامر التالية:

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

نقوم الآن بإنشاء database context للتعامل مع قاعدة البيانات. ننشئ أولاً مجلد جديد بإسم DbContexts وبداخله الملف MainDbContext.cs ومحتواه:

```csharp
using System;
using Microsoft.EntityFrameworkCore;

namespace aspnetcorewebapiproject.DbContexts
{
    public class MainDbContext : DbContext
    {
        public DbSet<Entities.Employee> Employees { get; set; }

        public MainDbContext(DbContextOptions<MainDbContext> options)
            : base(options)
        {
        }
    }
}
```

ثم نضيف التالي في آخر الدالة ConfigureServices في ملف Startup.cs:

```csharp
services.AddDbContext<DbContexts.MainDbContext>(
    options => options.UseSqlServer(Configuration.GetConnectionString("MainDbContext"))
);
```

وفي ملف appsettings.json نضيف إعدادات الإتصال بقاعدة البيانت ConnectionStrings بإسم MainDbContext وهنا نستفيد من معلومات الإتصال بقاعدة البيانات التي نسخناها سابقاً ولكننا سنستبدل قاعدة البيانات master بـ MainDb:

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

نضيف الآن أدة dotnet-ef:

```csharp
dotnet tool install --global dotnet-ef
```

ثم نضيف أوامر التعديل على قاعدة البيانات:

```csharp
dotnet ef migrations add InitialCreate
```

نلاحظ إنه تم إنشاء مجلد جديد بإسم Migrations وبه الأوامر التي سيتم تنفيذها للتعديل على قاعدة البيانات. نقوم الآن بتنفيذ الأمر:

```csharp
dotnet ef database update
```

نستخدم الآن Azure Data Studio لرؤية التعديلات التي تم تنفيذها. من القائمة في اليسار نختار Connection ثم New Connection:

{% include image.html url="assets/files/article_02/ads-new-connection.png" border="1" %}

سنقوم بعد ذلك بتعبئة معلومات الإتصال فى النافذة الجديدة:

{% include image.html url="assets/files/article_02/ads-new-connection-details.png" border="1" %}

والمعلومات هذه تم أخذها من التالي:

```json
"ConnectionStrings": {
    "MainDbContext": "Server=localhost;Database=MainDb;Trusted_Connection=True;"
}  
```

نضغط بعد ذلك على زر Connect ثم يظهر لنا أنه تم إنشاء جدولين جديدين في قاعدة البيانات Employees:

{% include image.html url="assets/files/article_02/ads-maindb-created.png" border="1" %}

* **EFMigrationsHistory__** وتحتفظ بكل التغييرات التي تمت على قاعدة البيانات هذه من Entity Framework
* **Employees** الجدول الذي يقابل الـ  Employee entity الذي أنشأناه في الكود

بإمكاننا أن نرى أن جدول Employees خالي ولم يتم تعبئته بالبيانات:

{% include image.html url="assets/files/article_02/ads-employees-select-top-1000.png" border="1" %}

أضاف التغييرات الى git:

```bash
git add .
git commit -m "adds main database context"
```

## إضافة كنترولر جديد EmployeesController

سنستفيد من أداة الـ dotnet في توليد generate كنترولر جديد بإسم EmployeesController. وللقيام بذلك علينا أولاً إضافة الخاصية لهذه الأداة بتنفيذ الأوامر التالية:

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
| **name** | EmployeesController |  إسم الكنترولر المراد إنشاؤه |
| **async** |   | لجعل الأوامر غير متزامنه asynchronous |
| **api** |   | لعدم إنشاء Views |
| **m** | Employee | إسم الـ model الذي سيستخدم |
| **dc** | MainDbContext | الـ database context المستخدم |
| **outDir** | Controllers | المجلد الذي سيتم وضع الكنترولر فيه |

بإمكاننا أن نرى بأنه تم إنشاء ملف جديد بإسم EmployeesController.cs داخل المجلد Controllers:

{% include image.html url="assets/files/article_02/vs-new-employeescontroller.png" border="1" %}

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using aspnetcorewebapiproject.DbContexts;
using aspnetcorewebapiproject.Entities;

namespace aspnetcorewebapiproject.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class EmployeesController : ControllerBase
    {
        private readonly MainDbContext _context;

        public EmployeesController(MainDbContext context)
        {
            _context = context;
        }

        // GET: api/Employees
        [HttpGet]
        public async Task<ActionResult<IEnumerable<Employee>>> GetEmployees()
        {
            return await _context.Employees.ToListAsync();
        }

        // GET: api/Employees/5
        [HttpGet("{id}")]
        public async Task<ActionResult<Employee>> GetEmployee(int id)
        {
            var employee = await _context.Employees.FindAsync(id);

            if (employee == null)
            {
                return NotFound();
            }

            return employee;
        }

        // PUT: api/Employees/5
        // To protect from overposting attacks, enable the specific properties you want to bind to, for
        // more details, see https://go.microsoft.com/fwlink/?linkid=2123754.
        [HttpPut("{id}")]
        public async Task<IActionResult> PutEmployee(int id, Employee employee)
        {
            if (id != employee.Id)
            {
                return BadRequest();
            }

            _context.Entry(employee).State = EntityState.Modified;

            try
            {
                await _context.SaveChangesAsync();
            }
            catch (DbUpdateConcurrencyException)
            {
                if (!EmployeeExists(id))
                {
                    return NotFound();
                }
                else
                {
                    throw;
                }
            }

            return NoContent();
        }

        // POST: api/Employees
        // To protect from overposting attacks, enable the specific properties you want to bind to, for
        // more details, see https://go.microsoft.com/fwlink/?linkid=2123754.
        [HttpPost]
        public async Task<ActionResult<Employee>> PostEmployee(Employee employee)
        {
            _context.Employees.Add(employee);
            await _context.SaveChangesAsync();

            return CreatedAtAction("GetEmployee", new { id = employee.Id }, employee);
        }

        // DELETE: api/Employees/5
        [HttpDelete("{id}")]
        public async Task<ActionResult<Employee>> DeleteEmployee(int id)
        {
            var employee = await _context.Employees.FindAsync(id);
            if (employee == null)
            {
                return NotFound();
            }

            _context.Employees.Remove(employee);
            await _context.SaveChangesAsync();

            return employee;
        }

        private bool EmployeeExists(int id)
        {
            return _context.Employees.Any(e => e.Id == id);
        }
    }
}
```

نلاحظ أنه تم إنشاء الـ actions التالية:

| Verb | Url| الوصف| Request Body | Response Body |
|---:|---:|---:|---:|---:|
| **GET** | *api/employees* | لإسترجاع قائمة بجميع الموظفين | - | قائمة بالموظفين |
| **GET** | *api/employees/{id}* | لإسترجاع معلومات موظف معين | - | معلومات موظف |
| **PUT** | *api/employees/{id}* | لتعديل معلومات موظف | معلومات موظف | - |
| **POST** | *api/employees* | لإضافة معلومات موظف جديد | معلومات موظف | معلومات موظف |
| **DELETE** | *api/employees/{id}* | لحذف معلومات موظف | - | معلومات موظف |

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

سنستخد Postman للقيام بذلك، ولكن أول ما نقوم به هو الذهاب الى File ثم Settings والتأكد من قيمة الإعداد التالي:

```bash
SSL certificate verification = OFF
```

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

```html
https://localhost:5001/api/employees/2
```

### حذف معلومات موظف معين

نرى أن الـ Http Status Code المسترجع هو  200 Ok أي أنه تم تنفيذ العملية بشكل صحيح:

{% include image.html url="assets/files/article_02/postman-delete-employee-info.png" border="1" %}

بإمكانك الإستعلام عن معلومات الموظف لتتأكد بأنه تم فعلاً حذف المعلومات المتعلقة به:

{% include image.html url="assets/files/article_02/postman-get-specific-employee-after-deletion.png" border="1" %}

نرى أن الـ Http Status Code المسترجع هو  404 Not Found أي أنه لم يتم إيجاد الموظف بهذا الرقم وهذا صحيح.

والـن لا تنسى إضافة التغييرات الى git:

```bash
git add .
git commit -m "adds EmployeeController"
```

وهكذا نكون قد أضفنا controller جديد مما يمكنا من بنائ باقي المشروع عليه بمشيئة الله.
