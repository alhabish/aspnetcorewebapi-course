---
layout: post
title: "6. Logging and Global Error Handling"
date: 2020-04-29
lang: ar-SA
index: 6
comments: true
---

## Logging

سنقوم بإضافة مكتبة NLog والكتابة الى ملف عن طريق إتباع خطوات مبسطة مشروحة في الرابط التالي:

<https://github.com/NLog/NLog/wiki/Getting-started-with-ASP.NET-Core-3>

### 1. إضافة مكتبة NLog الى المشروع

وذلك عن طريق تنفيذ الأمر التالي:

```bash
dotnet add package NLog.Web.AspNetCore
```

### 2. إنشاء ملف nlog.config

يجب أن يكون إسم الملف بالأحرف الصغيرة وننشئه في المجلد الرئيسي للمشروع. وفيما يلي محتواه:

```xml
<?xml version="1.0" encoding="utf-8"?>
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <extensions>
        <add assembly="NLog.Web.AspNetCore" />
    </extensions>
    <targets>
        <target name="logfile" xsi:type="File" fileName="c:\temp-logs\${shortdate}.log" />
        <target name="logconsole" xsi:type="Console" />
    </targets>
    <rules>
        <logger name="*" minlevel="Info" writeTo="logconsole" />
        <logger name="*" minlevel="Debug" writeTo="logfile" />
    </rules>
</nlog>
```

### 3. نسخ nlog.config الى الـ output directory

هنا نتأكد من أن الملف سيتم نشره مع المشروع وذلك عن طريق فتح ملف المشروع وإضافة التالي:
 
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  ...
  
  <ItemGroup>
      <Content Update="nlog.config" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>

</Project>
```

### 4. التعديل على ملف Program.cs

تأكد من أن هذا محتوى الملف:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using NLog.Web;

namespace aspnetcorewebapiproject
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var logger = NLog.Web.NLogBuilder.ConfigureNLog("nlog.config").GetCurrentClassLogger();
            try
            {
                logger.Debug("init main");
                CreateHostBuilder(args).Build().Run();
            }
            catch (Exception exception)
            {
                //NLog: catch setup errors
                logger.Error(exception, "Stopped program because of exception");
                throw;
            }
            finally
            {
                // Ensure to flush and stop internal timers/threads before application-exit (Avoid segmentation fault on Linux)
                NLog.LogManager.Shutdown();
            }
        }

        public static IHostBuilder CreateHostBuilder(string[] args) =>
            Host.CreateDefaultBuilder(args)
                .ConfigureWebHostDefaults(webBuilder =>
                {
                    webBuilder.UseStartup<Startup>();
                })
                .ConfigureLogging(logging =>
                {
                    logging.ClearProviders();
                    logging.SetMinimumLevel(Microsoft.Extensions.Logging.LogLevel.Trace);
                })
                .UseNLog();  // NLog: Setup NLog for Dependency injection
    }
}
```

### 5. إضافة ILogger للـ Controller

نعدل على EmployeesControllers كالتالي:

```csharp
using Microsoft.Extensions.Logging;
...

public class EmployeesController : ControllerBase
{
	...
	private ILogger<EmployeesController> _logger;

	public EmployeesController(ILogger<EmployeesController> logger, IConfiguration config, IRepository<Employee> repo, IMapper mapper)
	{
		_logger = logger;
		...
	}
}
```

### 6. إستخدام الـ logger في العمليات 

الآن بإمكاننا إستخدام الـ logger في أي من العمليات التي لدينا. أسهل شيئ بإمكاننا عمله هو كتابة إسم العملية التي تم طلبها:

```csharp
public async Task<ActionResult<EmployeesResponse<PaginatedList<EmployeeDetailsDto>>>> GetEmployees([FromQuery] EmployeeGetDto employeeGetDto)
{
	_logger.LogInformation("GetEmployees requested");
	
	...
}

public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployee(int id)
{
	_logger.LogInformation("GetEmployee requested");

	...
}

public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> PutEmployee(int id, EmployeeUpdateDto employeeUpdateDto)
{
	_logger.LogInformation("PutEmployee requested");

	...          
}

public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> PostEmployee(EmployeeInsertDto employeeInsertDto)
{
	_logger.LogInformation("PostEmployee requested");

	...
}

public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> DeleteEmployee(int id)
{
	_logger.LogInformation("DeleteEmployee requested");

	...
}
```

### 7. نفذ بعض العمليات بواسطة Postman

عن طريق Postman قم بإستدعاء بعض العمليات ثم إفتح الجلد temp-logs لتري ملف log لكل تاريخ جديد:

{% include image.html url="assets/files/article_06/temp-logs.png" border="1" %}

وعند فتح الملف سترى ملومات كالتالي:

```bash
2020-04-30 22:37:04.9184|DEBUG|aspnetcorewebapiproject.Program|init main
2020-04-30 22:37:10.3272|INFO|Microsoft.Hosting.Lifetime|Now listening on: https://localhost:5001
2020-04-30 22:37:10.3439|INFO|Microsoft.Hosting.Lifetime|Now listening on: http://localhost:5000
2020-04-30 22:37:10.3654|INFO|Microsoft.Hosting.Lifetime|Application started. Press Ctrl+C to shut down.
2020-04-30 22:37:10.3727|INFO|Microsoft.Hosting.Lifetime|Hosting environment: Development
2020-04-30 22:37:10.3973|INFO|Microsoft.Hosting.Lifetime|Content root path: C:\repos\aspnetcorewebapiproject
2020-04-30 22:49:27.4039|INFO|aspnetcorewebapiproject.Controllers.EmployeesController|GetEmployee requested
2020-04-30 22:49:50.4418|INFO|aspnetcorewebapiproject.Controllers.EmployeesController|GetEmployees requested
2020-04-30 22:50:44.0396|INFO|aspnetcorewebapiproject.Controllers.EmployeesController|PostEmployee requested
```

## Global Error Handling

يجب علينا التعامل مع الأخطاء التي قد تحدث في الكود. ونحن بين خيارين: إما التعامل معها على مستو العملية ذاتها، وهذا يعني تكرار كبير للأكواد التي نكتبها. أو، وهو الحل الذي سنتبعه هنا، أن يكون لنا مكان موحد للتعامل مع الأخطاء.

ولإتباع الطريقة الثانية، نتعامل مع UseExceptionHandler وهو middleware يمكننا من الوصول الى الخطأ الذي حدث والتعامل معه قبل إعادة response. والآن نتبع الخطوات التالية:

ننشئ مجلد جديد بإسم Extensions وبداخله ملف بإسم MiddlewareExtensions.cs:

```csharp
using System;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Http;
using System.Net;
using Microsoft.Extensions.Logging;
using aspnetcorewebapiproject.Models.Employees;
using System.Text.Json;
using System.Text.Json.Serialization;

namespace aspnetcorewebapiproject.Extensions
{
    public static class MiddlewareExtensions
    {
        public static void ConfigureGlobalExceptionHandler(this IApplicationBuilder app, ILogger logger)
        {
            app.UseExceptionHandler(appError =>
            {
                appError.Run(async context =>
                {
                    context.Response.StatusCode = (int)HttpStatusCode.InternalServerError; // 500
                    context.Response.ContentType = "application/json";

                    var contextFeature = context.Features.Get<IExceptionHandlerFeature>();
                    if(contextFeature != null)
                    { 
                        var error = contextFeature.Error;

                        logger.LogError($"Internal Server Error: {error}");

                        var response = JsonSerializer.Serialize( new EmployeesResponse<object>(){
                            IsSuccessful = false,
                            Status = context.Response.StatusCode,
                            Message = "Internal Server Error",
                        });

                        await context.Response.WriteAsync(response);
                    }
                });
            });
        }
    }
}
```

من المهم أن تلاحظ بأننا لم نعط للمستخدم تفاصيل الخطأ بل أعدنا له رسالة عامة لما قد يسبب ذلك من مخاطر أمنية في كشف تفاصيل النظام وإنما نقوم وبإستخدام الـ logger بكتابة تفاصيل الخطأ في ملف لمعالجتها فيما بعد.

الآن في ملف Startup.cs نضيف الـ namespace التالي:

```csharp
using aspnetcorewebapiproject.Extensions;
```

ثم نعدل على الدالة ()Configure لتستقبل متغير من نوع <ILogger<Startup ونستدعي الدالة ()ConfigureGlobalExceptionHandler التي أنشأناها:

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILogger<Startup> logger)
{
	if (env.IsDevelopment())
	{
		app.UseDeveloperExceptionPage();
	}

	...
}
```

والآن لتجربة ما قمنا به، قم بإحداث خطأ في العملية ()GetEmployee كالتالي ولا تنسى حذفها بعد التجربة:

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployee(int id)
{
	throw new Exception ("A problem occurred!");
	
	...
}
```

وعند تجربتها في Postman نلاحظ الرد التالي:

{% include image.html url="assets/files/article_06/general-error-handling.png" border="1" %}

وبما أننا أصبحنا نتعامل مع الأخطاء في مكان موحد، فبإمكاننا الآن العودة الى العملية ()PutEmployee والتعديل عليها لكي لا ترجع خطأ من نوع Status500InternalServerError بل نجعلها مسؤولية الـ middleware الجديد الذي أنشأناه ()ConfigureGlobalExceptionHandler:

```csharp
[HttpPut("{id}")]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> PutEmployee(int id, EmployeeUpdateDto employeeUpdateDto)
{
	_logger.LogInformation("PutEmployee requested");

	var response = new EmployeesResponse<EmployeeDetailsDto>();
	
	if ( (!ModelState.IsValid) || (id != employeeUpdateDto.Id) )
	{
		response.IsSuccessful = false;                
		response.Status = 400;
		response.Message = "Inputs are invalid";

		return BadRequest(response);
	}

	try
	{
		var employeeEntity = _mapper.Map<Employee>(employeeUpdateDto);
		await _repo.UpdateAsync(employeeEntity);

		var employeeDetailsDto = _mapper.Map<EmployeeDetailsDto>(employeeEntity);

		response.IsSuccessful = true;
		response.Status = 200;
		response.Data = employeeDetailsDto;

		return Ok( response );    
	}
	catch 
	{
		if (await _repo.GetAsync(id) == null)     
		{
			response.IsSuccessful = false;
			response.Status = 404;
			response.Message = "Employee not found";

			return NotFound(response);
		}

		// A general error occurred
		throw;               
	}
}
```

وبذلك لن نستخدم عبارة `try {} catch {}` في العمليات وإنما سنجعل الـ middleware الجديد يتحمل مسؤولية التعامل مع الأخطاء.
