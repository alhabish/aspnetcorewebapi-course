---
layout: post
title: "8. Consuming the Service"
date: 2020-05-23
lang: ar-SA
index: 8
comments: true
---

# إستدعاء الخدمة من Javascript

## إستدعاء GetEmployee

إحفظ الملف التالي بإسم get-employee.html ثم قم بعرضه في أحد المتصفحات:

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

ستلاحظ أن الصفحة فاضية، ولكن لو عرضت الـ developer tools بإستخدام F12 سترى التالي:

{% include image.html url="assets/files/article_08/err-cert-authority-invalid.png" border="1" %}

ولو حاولنا فتح المتصفح مباشرة على الرابط:

https://localhost:5001/api/employees/1

ستظهر لنا الشاشة التالية. والمشكلة هنا أن المتصفح لم يعتمد ويتعرف على الشهادة certificate التي أنشأتها بيئة عمل dot net core للتطوير development. ولإعتمادها مؤقتاً نضغط على Advanced:

{% include image.html url="assets/files/article_08/cert-issue-browser-advanced.png" border="1" %}

ثم Proceed to localhost:

{% include image.html url="assets/files/article_08/proceed-to-localhost.png" border="1" %}

بإمكاننا بعد ذلك أن نرى نتيجة الإستدعاء ظاهرة في المتصفح:

{% include image.html url="assets/files/article_08/get-employee-browser-response.png" border="1" %}

عند تنفيذ الملف get-employee.html مرة أخرى ستلاحظ أننا الآن نستقبل رسالة خطأ مختلفة:

{% include image.html url="assets/files/article_08/err-connection-refused.png" border="1" %}

وسبب ذلك يعود الى مبدأ Cross-Origin Resource Sharing - CORS حيث أن المتصفح لن يسمح لنا بإستدعاء خدمة عن طريق الـ javascript ليستا في نفس الدومين طلما لم تصرح الخدمة بالسماح بذلك.

## تمكين CORS

وللسماح لأي مستفيد من الوصول الى الخدمة نقوم بالتالي:

### 1. التعديل على ConfigureServices في Startup.cs

Open Startup.cs and in ConfigureServices add:

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

Also in Startup.cs and in Configure add:

app.UseCors();

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

### 2. التعديل على EmployeesController

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
وبإستطاعتنا أن نرى أنه تم إضافة المستخدم الجديد:

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
وبإستطاعتنا أن نرى أنه تم إضافة المستخدم الجديد:

{% include image.html url="assets/files/article_08/delete-employee-browser-response.png" border="1" %}

# إستدعاء الخدمة من NET.

أسهل طريقة للتعامل مع خدمة من نوع REST عن طريق الـ NET. هي بإستخدام مكتبة RestSharp. 

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


### 3. إضافة مكتبة RestSharp

لإضافة هذه المكتبة البرمجية نستخد الأمر التالي:

```bash
dotnet add package RestSharp
```

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
