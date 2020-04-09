---
layout: post
title: "2. Entities and Models"
date: 2020-04-07
lang: ar-SA
index: 1
comments: true
---

الخدمة service التي سنقدمها تتعلق بالموظفين وكل ما يتغلق بالإضافة/ التعديل / الإستعلام عن بياناتهم. 

قاعدة البيانات المستخدمة هي Microsoft SQL Server والنسخة التي سنقوم بتحميلها SQL Server 2019 Developer وذلك من الرابط التالي:

<https://www.microsoft.com/en-us/sql-server/sql-server-downloads>

ولإدارة قاعدة البيانات هذه والتعامل معها سنستخدم Azure Data Studio والتي يمكن تحميلها من الرابط التالي:

<https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio>

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

وفي ملف appsettings.json نضيف إعدادات الإتصال بقاعدة البيانت ConnectionStrings بإسم MainDbContext:

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
    "MainDbContext": "Data Source=(localdb)\\MSSQLSERVER;Initial Catalog=Employees;Integrated Security=True;"
  }    
}

```

نضيف الآن أدة dotnet-ef

```csharp
dotnet tool install --global dotnet-ef
```

نضيف أوامر التعديل على قاعدة البيانات ثم ننفذ الأمر:

```csharp
dotnet ef migrations add InitialCreate
```


```csharp
dotnet ef database update
```
