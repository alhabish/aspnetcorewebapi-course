---
layout: post
title: "8. Consuming the Service"
date: 2020-05-23
lang: ar-SA
index: 8
comments: true
---

كنا فيما سبق نستخدم Postman لتجربة الخدمة التي بنيناها، ولكن عندما نريد الإستفادة من هذه الخدمة بالفعل سيكون ذلك إما عن طريق نداء الخدمة من الـ frontend وسنأخذ مثال على الـ javascript للقيام بذلك أو يكون في الـ backend وسنستخدم NET. في هذه الحالة.

# إستدعاء الخدمة من Javascript

سنبدأ من الـ frontend وسنوضح كيف يمكن لـ javascript أن تستدعي جميع العمليات التي قمنا بإنشائها في EmployeesController:

## إستدعاء GetEmployee

إحفظ الكود التالي في ملف بإسم get-employee.html ثم قم بعرضه في أحد المتصفحات:

```html
<html>
 <body>
  <script>
   var baseUrl = 'https://localhost:5001/api/employees';
   var employeeId = 1;
   var url = baseUrl + '/' + employeeId;

   fetch(url)
    .then(response => response.json())
    .then(json => console.log(json))
  </script>
 </body>
</html>
```

ستلاحظ أن الصفحة لا تحتوي على شيئ، ولكن لو عرضت الـ developer tools بإستخدام F12 سترى التالي:

{% include image.html url="assets/files/article_08/err-cert-authority-invalid.png" border="1" %}

ولو حاولنا فتح المتصفح مباشرة على الرابط:

<https://localhost:5001/api/employees/1>

ستظهر لنا الشاشة التالية:

{% include image.html url="assets/files/article_08/cert-issue-browser-advanced.png" border="1" %}

 والمشكلة هنا أن المتصفح لم يتعرف على الشهادة certificate التي أنشأتها بيئة عمل dot net core للتطوير development. ولإعتمادها مؤقتاً نضغط على Advanced ثم Proceed to localhost:

{% include image.html url="assets/files/article_08/proceed-to-localhost.png" border="1" %}

بإمكاننا بعد ذلك أن نرى نتيجة الإستدعاء ظاهرة في المتصفح:

{% include image.html url="assets/files/article_08/get-employee-browser-response.png" border="1" %}

عند تنفيذ الملف get-employee.html مرة أخرى ستلاحظ أننا الآن نستقبل رسالة خطأ مختلفة:

{% include image.html url="assets/files/article_08/err-connection-refused.png" border="1" %}

وسبب ذلك يعود الى مبدأ Cross-Origin Resource Sharing - CORS حيث أن المتصفح لن يسمح لنا بإستدعاء خدمة عن طريق الـ javascript ليست في نفس دومين المستدعي طلما لم تصرح الخدمة بالسماح بذلك.

## تمكين CORS

للسماح لأي مستفيد من الوصول الى الخدمة نقوم بالتالي:

### 1. التعديل على ConfigureServices في Startup.cs

نقوم بالتعديل التالي:

```csharp
public void ConfigureServices(IServiceCollection services)
{
	...

	services.AddCors(o => o.AddPolicy("CorsPolicy", 
		builder => { builder.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader(); }
	));
}
```
### 2. التعديل على Configure في Startup.cs

نضيف ما يلي:

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILogger<Startup> logger)
{
	...

	app.UseRouting();

	app.UseCors();

	app.UseAuthorization();

	...
}
```

يجب أن يكون النداء لـ UseCors بعد ()UseRouting وقبل ()UseAuthorization حسب الترتيب المذكور في الصفحة التالية:

[Middleware order](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/middleware/?view=aspnetcore-3.1#middleware-order)

### 3. التعديل على EmployeesController

في EmployeesController نضيف التعديلات التالية:

```csharp
...
using Microsoft.AspNetCore.Cors;

namespace aspnetcorewebapiproject.Controllers
{
    [EnableCors("CorsPolicy")]
    [Route("api/[controller]")]
    [ApiController]
    public class EmployeesController : ControllerBase
    {
		...
```

والآن عند عرض الملف في المتصفح مرة أخرى سوف يكون بإمكاننا رؤية نتيجة الإستدعاء بشكل صحيح:

{% include image.html url="assets/files/article_08/get-employee-browser-response-correct.png" border="1" %}

## إستدعاء GetEmployees

وهي قريبة لما سبق. ننشئ ملف جديد بإسم get-employees.html ثم نستعرضه في المتصفح:

```html
<html>
 <body>
  <script>
   var url = 'https://localhost:5001/api/employees';

   fetch(url)
    .then(response => response.json())
    .then(json => console.log(json))
  </script>
 </body>
</html>
```

{% include image.html url="assets/files/article_08/get-employees-browser-response.png" border="1" %}

## إستدعاء PostEmployee

ننشئ ملف جديد بإسم post-employee.html ثم نستعرضه في المتصفح:

```html
<html>
 <body>
  <script>
   var url = 'https://localhost:5001/api/employees';

   fetch(url, {
    method: 'POST',
    body: JSON.stringify({
	 firstName: "Abdulrahman",
	 lastName: "Saud",
	 isManager: false,
	 salary: 500,
	 enrollmentDate: "2016-03-12"
    }),
    headers: {
     'Content-type': 'application/json; charset=UTF-8'
    }
   })
  .then(response => response.json())
  .then(json => console.log(json))
  </script>
 </body>
</html>
```
وبإستطاعتنا أن نرى أنه تم إضافة المستخدم الجديد:

{% include image.html url="assets/files/article_08/post-employee-browser-response.png" border="1" %}

## إستدعاء PutEmployee

ننشئ ملف جديد بإسم put-employee.html ثم نستعرضه في المتصفح:

```html
<html>
 <body>
  <script>
   var baseUrl = 'https://localhost:5001/api/employees';
   var employeeId = 5;
   var url = baseUrl + '/' + employeeId;

   fetch(url, {
    method: 'PUT',
    body: JSON.stringify({
	 id: 5,
	 firstName: "Abdulrahman",
	 lastName: "Saud",
	 isManager: true,
	 salary: 500,
	 enrollmentDate: "2016-03-12"
    }),
    headers: {
     'Content-type': 'application/json; charset=UTF-8'
    }
   })
  .then(response => response.json())
  .then(json => console.log(json))
  </script>
 </body>
</html>
```
وبإستطاعتنا أن نرى أنه تم فعلاً تعديل البيانات:

{% include image.html url="assets/files/article_08/put-employee-browser-response.png" border="1" %}

## إستدعاء DeleteEmployee

ننشئ ملف جديد بإسم delete-employee.html ثم نستعرضه في المتصفح:

```html
<html>
 <body>
  <script>
   var baseUrl = 'https://localhost:5001/api/employees';
   var employeeId = 5;
   var url = baseUrl + '/' + employeeId;

   fetch(url, {
    method: 'DELETE'
   })
  .then(response => response.json())
  .then(json => console.log(json))
  </script>
 </body>
</html>
```
وبإستطاعتنا أن نرى أن العملية تمت بشكل صحيح:

{% include image.html url="assets/files/article_08/delete-employee-browser-response.png" border="1" %}

# إستدعاء الخدمة من NET.

في الحالات التي نريد فيها إستدعاء الخدمة من الـ backend فإننا نستخدم هذه الإسلوب وأسهل طريقة للتعامل مع خدمة من نوع REST عن طريق الـ NET. هي بإستخدام مكتبة RestSharp. وللقيام بذلك نتبع الخطوات التالية:

### 1. أنشئ مشروع جديد من نوع Console

وذلك بتنفيذ الأمر التالي:

```bash
dotnet new console
```

ثم نفتح المشروع في VS Code:

```bash
code .
```

### 2. إعادة تعريف بعض الـ classes

يجب علينا إعادة تعريف الـ classes التالية للتعامل معها في المشروع:

* EmployeesResponse
* PaginatedList
* EmployeeDetailsDto

وبالنسبة لـ PaginatedList فنحن لسنا بحاجة لتعريف الـ methods وإنما الـ properties فقط.

### 3. إضافة مكتبة RestSharp

لإضافة هذه المكتبة البرمجية نستخدم الأمر التالي:

```bash
dotnet add package RestSharp
```

وسنقوم الآن بإستدعاء جميع العمليات بإستخدام هذه المكتبة.

## إستدعاء GetEmployee

```csharp
using System;
using RestSharp;

namespace ConsumeRestService
{
    class Program
    {
        static void Main(string[] args)
        {
            var client = new RestClient("https://localhost:5001/api");
            client.RemoteCertificateValidationCallback = (sender, certificate, chain, sslPolicyErrors) => true;

            var request = new RestRequest("employees/1", Method.GET);

            var response = client.Execute<EmployeesResponse<EmployeeDetailsDto>>(request);

            Console.WriteLine( response.Content );
        }
    }
}
```

أحتجنا الى إضافة السطر التالي لتجاوز مشكلة شهادة الـ SSL:

```csharp
client.RemoteCertificateValidationCallback = (sender, certificate, chain, sslPolicyErrors) => true;
```

## إستدعاء GetEmployees

```csharp
static void Main(string[] args)
{
	var client = new RestClient("https://localhost:5001/api");
	client.RemoteCertificateValidationCallback = (sender, certificate, chain, sslPolicyErrors) => true;

	var request = new RestRequest("employees", Method.GET);

	var response = client.Execute<EmployeesResponse<PaginatedList<EmployeeDetailsDto>>>(request);

	Console.WriteLine( response.Content );
}
```

## إستدعاء PostEmployee

```csharp
static void Main(string[] args)
{
	var client = new RestClient("https://localhost:5001/api");
	client.RemoteCertificateValidationCallback = (sender, certificate, chain, sslPolicyErrors) => true;

	var request = new RestRequest("employees", Method.POST);

	request.AddJsonBody( new {
		firstName = "Fahad",
		lastName = "Talal",
		isManager = false,
		salary = 800,
		enrollmentDate = "2013-02-11"
	});

	var response = client.Execute<EmployeesResponse<EmployeeDetailsDto>>(request);

	Console.WriteLine( response.Content );
}
```

## إستدعاء PutEmployee

```csharp
static void Main(string[] args)
{
	int employeeId = 6;

	var client = new RestClient("https://localhost:5001/api");
	client.RemoteCertificateValidationCallback = (sender, certificate, chain, sslPolicyErrors) => true;

	var request = new RestRequest($"employees/{employeeId}", Method.PUT);

	request.AddJsonBody( new {
		id = employeeId,
		firstName = "Faisal",
		lastName = "Talal",
		isManager = false,
		salary = 800,
		enrollmentDate = "2013-02-11"
	});

	var response = client.Execute<EmployeesResponse<EmployeeDetailsDto>>(request);

	Console.WriteLine( response.Content );
}
```

## إستدعاء DeleteEmployee

```csharp
static void Main(string[] args)
{
	var client = new RestClient("https://localhost:5001/api");
	client.RemoteCertificateValidationCallback = (sender, certificate, chain, sslPolicyErrors) => true;

	var request = new RestRequest("employees/6", Method.DELETE);

	var response = client.Execute<EmployeesResponse<EmployeeDetailsDto>>(request);

	Console.WriteLine( response.Content );
}
```
