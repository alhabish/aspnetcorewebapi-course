---
layout: post
title: "9. Versioning"
date: 2020-06-04
lang: ar-SA
index: 9
comments: true
---

الخدمات services، كباقي البرمجيات، يطرأ عليها الحاجة الى التعديل والتطوير ويجب علينا كمطورين لهذه الخدمة أن نقوم بهذه التعديلات بطريقة لا تؤثر على المستفيدين من هذه الخدمة.

هنا ثلاث طرق رئيسية لإصدار نسخ versions من الخدمة، نذكر منها:

| الطريقة | مثال |
|---:|---:|
| HTTP Header | X-API-Version: 2 |
| Url | /v2/employees |
| Query String | /employees?api-version=2 |

في هذا الشرح سوف نعتمد طريقة تحديد النسخة في الـ Url.


### التعديل على الـ Controller

إنشاء مجلد جديد بإسم v1 بداخل Controllers والذي سنقوم بوضع جميع الـ controllers في النسخة 1 version بداخله:

{% include image.html url="assets/files/article_09/vscode-controllers-v1-employees.png" border="1" %}

تعديل الـ namespace الخاص بـ EmployeesController الى:

```csharp
namespace aspnetcorewebapiproject.Controllers.v1
{
```

### التعديل على الـ Models

نقوم أيضاً بإنشاء مجلد جديد بإسم v1 داخل المجلد Models ونضع المجلد Employees الموجود سابقاً بداخله:

{% include image.html url="assets/files/article_09/vscode-models-v1-employees.png" border="1" %}

نعدل الـ namespace لجميع الـ classes لتصبح كما يلي:

```csharp
namespace aspnetcorewebapiproject.Models.Employees.v1
{
```

### تصحيح الأخطاء

عند بناء المشروع ستجد أن هنالك الكثير من الأخطاء موجودة في الملفات التالية:

* EmployeesController.cs 
* EmployeesProfile.cs
* MiddlewareExtensions.cs

وذلك لأن الـ EmployeesController لم يعد بإمكانه إيجاد الـ classes السابقة وذلك لأننا قمنا بتغيير الـ namespace وكل ما عليك فعله هو كتابة الـ namespace الصحيح.

ملاحظة: في الملف MiddlewareExtensions.cs نعيد EmployeResponse من النسخة 1 دائماً حتى وإن طلبت نسخة مختلفة. قد يكون هذا ما تريده وبإمكانك أيضاً أن تعيد Response خاص بالنسخة المطلوبة.

قم الآن ببناء المشروع ومن المفترض أن تختفي جميع الأخطاء.

## إضافة مكتبة الـ Versioning

نقوم الآن بإضافة المكتبتين التالية والتي ستساعدنا على تطبيق الـ versioning في الخدمة المقدمة:

```csharp
dotnet add package Microsoft.AspNetCore.Mvc.Versioning
dotnet add package Microsoft.AspNetCore.Mvc.Versioning.ApiExplorer
```

## التعديل على الـ EmployeesController

أضف ApiVersion و عدل Route لتصبح كالتالي:

```csharp
...

namespace aspnetcorewebapiproject.Controllers.v1
{
    [ApiVersion("1.0")]
    [Route("api/v{version:apiVersion}/[controller]")]
    [EnableCors("CorsPolicy")]
    [ApiController]
    public class EmployeesController : ControllerBase
    {
		...
```

الـ ApiVersion attribute تحدد نسخة الـ controller وعدلنا على Route ليأخذ النسخة في العنوان url.

## التعديل على ()ConfigureServices في Setup.cs

نضيف على هذه الدالة ما يلي:

```csharp
services.AddApiVersioning(o => {
	o.DefaultApiVersion = new ApiVersion(1, 0);                
	o.AssumeDefaultVersionWhenUnspecified = true;
	o.ReportApiVersions = true;      
});
```

### إختبار ما قمنا به

للتأكد من أن ما قمنا به سينفذ بشكل صحيح وسيعود الينا رقم النسخة الصحيح فإنه بإمكاننا إضافة العملية التالية:

```csharp
[HttpGet("version")]
public string GetVersion() => HttpContext.GetRequestedApiVersion().ToString();
```

في Postman قم بإستدعاء العناوين التالية:

```html
<!-- 1 -->
https://localhost:5001/api/v1/employees/version 

<!-- 1.0 -->
https://localhost:5001/api/v1.0/employees/version
```

في المرة الأولى ستعود الينا القيمة `1` وفي الثانية `1.0` ولكنها فعلياً تشير الى نفس الـ controller.

### التعديل على عملية GetEmployees

لنفترض أنه لدينا الآن متطلب جديد للعملية GetEmployees وهي أن تعيد لنا تاريخ الإنضمام بالتاريخ الميلادي بالإضافة الى الهجري. بإمكاننا أن الآن أن نعدل على هذه العملية وسيكتشف المستفيدين من الخدمة بأنه تم إضافة property جديد في القيمة المسترجعة. في مثالنا هذا الموضوع لا يشكل مسألة كبيرة ولكن لو أفترضنا أنا حذفنا property أو دمجنا أكثر من property معاً (مثل أن تصبح FirstName و LastName الى FullName في التعديل الجديد). هذه الأمور ستسبب بلا شك مشكلة للمستفيد من الخدمة حيث أنه بنى برنامجه على شكل معين للقيمة المسترجعة ثم يكتشف بأنه شكل آخر.

ولذلك سننشى action جديد وسنجعله يعيد DTO جديد يحمل التاريخ الهجري وذلك بإتباع ما يلي:

### التعديل على الـ Models

نقوم بإنشاء مجلد جديد بإسم v1_1 داخل المجلد Models وننشئ مجلد آخر جديد بداخله بإسم Employees والذي بداخله أيضاً الملف EmployeeDetailsDto.cs:

```csharp

using System;

namespace aspnetcorewebapiproject.Models.Employees.v1_1
{
    public class EmployeeDetailsDto
    {
        public int Id { get; set; }
        public string FirstName { get; set; }        
        public string LastName { get; set; }
        public bool IsManager { get; set; }
        public DateTime EnrollmentDate { get; set; }
        public string EnrollmentDateHijri { get; set; }
    }
}
```

نلاحظ أنه مشابه للـ class في النسخة 1 ماعدا في الـ namespace والـ property الأخيرة EnrollmentDateHijri.

{% include image.html url="assets/files/article_09/vscode-models-v1_1-employees.png" border="1" %}

### التعديل على EmployeesProfile

يجب علينا أن نوضح لـ AutoMapper كيف يقوم بالتحويل من Entities.Employee الى Models.Employees.v1_1.EmployeeDetailsDto حيث أن جميع التحويلات السابقة كانت سهلة لأن القيمة المحول منها source والمحول اليها destination كانتا متطابقتان في الـ properties ولكن في حالتنا هذه يوجد property إضافي في المحول اليه ليس موجود في المحول منه.

```csharp
using System;
using System.Globalization;
using AutoMapper;

namespace aspnetcorewebapiproject.Profiles
{
    public class EmployeesProfile : Profile
    {
        public EmployeesProfile()
        {
            // v 1.0
            //            

            // Map from Employee to EmployeeDetailsDto
            CreateMap<Entities.Employee, Models.Employees.v1.EmployeeDetailsDto>();

            // Map from EmployeeInsertDto to Employee
            CreateMap<Models.Employees.v1.EmployeeInsertDto, Entities.Employee>();

            // Map from EmployeeUpdateDto to Employee
            CreateMap<Models.Employees.v1.EmployeeUpdateDto, Entities.Employee>();

            // v 1.1
            // 

            // Map from Employee to EmployeeDetailsDto
            Calendar umAlQura = new UmAlQuraCalendar();

            CreateMap<Entities.Employee, Models.Employees.v1_1.EmployeeDetailsDto>()
                .ForMember( dest => dest.EnrollmentDateHijri,
                    map => map.MapFrom(
                        src => 
                            new String($"{umAlQura.GetDayOfMonth(src.EnrollmentDate)}/{umAlQura.GetMonth(src.EnrollmentDate)}/{umAlQura.GetYear(src.EnrollmentDate)}")
                    ));
        }
    }
}
```

### إنشاء العملية Action الجديدة

ننسخ العملية GetEmployee ونلصقها بالتعديلات التالية:

* نغير إسم الدالة الجديدة الى GetEmployeeV1_1 
* نضيف الـ attribute الجديد MapToApiVersion والذي يحدد في أي نسخه يتم إستدعاء هذه العملية
* نستخدم EmployeeDetailsDto بنسخة  1.1
* نحفظ القيمة في الـ cache بإسم يدل على النسخة ونسترجعها بالإسم الجديد حيث لو أعتمدنا على الـ id فقط سيكون هنالك تضارب بين القيمة في النسخة 1.0 والنسخة 1.1

```csharp
[HttpGet("{id}")]
[MapToApiVersion("1.1")]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployeeV1_1(int id)
{            
	_logger.LogInformation("GetEmployee requested");

	string key = "v1.1__" + id;

	if( !_cache.TryGetValue(key, out Models.Employees.v1_1.EmployeeDetailsDto employeeDetailsDto) )
	{
		var employeeEntity = await _repo.GetAsync(id);

		if (employeeEntity != null)                
			employeeDetailsDto = _mapper.Map<Models.Employees.v1_1.EmployeeDetailsDto>(employeeEntity);                

		_cache.Set(
			key,
			employeeDetailsDto,
			new MemoryCacheEntryOptions {
				SlidingExpiration = TimeSpan.FromMinutes(1)
			}
		);
	}

	var response = new EmployeesResponse<Models.Employees.v1_1.EmployeeDetailsDto>()
	{
		IsSuccessful = true,
		Status = (employeeDetailsDto == null) ? 404 : 200,
		Message = (employeeDetailsDto == null) ? "Employee not found" : string.Empty,
		Data = employeeDetailsDto
	};

	if (response.Status == 404) 
		return NotFound( response );
	
	return Ok( response );                
}
```

### التعديل على الـ EmployeesController

نضيف الآن ApiVersion إضافي الى EmployeesController يشير الى أن الـ controller يدعم نسخة 1.1 أيضاً لتصبح كالتالي:

```csharp
...

namespace aspnetcorewebapiproject.Controllers.v1
{
    [ApiVersion("1.0")]
    [ApiVersion("1.1")]
    [Route("api/v{version:apiVersion}/[controller]")]
    [EnableCors("CorsPolicy")]
    [ApiController]
    public class EmployeesController : ControllerBase
    {
		...
```

### التجربة في Postman

نقوم أولاً بتجربة النسخة الأصلية 1.0 من GetEmployees:

{% include image.html url="assets/files/article_09/postman-getemployee-v1.png" border="1" %}

ثم نجرب النسخة الجديدة 1.1:

{% include image.html url="assets/files/article_09/postman-getemployee-v1_1.png" border="1" %}

نلاحظ أن القيمة المسترجعة أخذت بالإعتبار التعديلات الجديدة.

ملاحظة مهمة، ما قمنا به فعلياً هو توفير نسختين من EmployeesController الأولى بالنسخة 1 أو 1.0 والثانية بالنسخة 1.1 ومعنى ذلك أنه بإمكاني إستدعاء جميع العمليات بأحد النسختين وسيعيد الينا نفس القيمة ولكن GetEmployee هي الوحيدة التي تختلف بين النسختين.

فلو قمنا بتجربة GetEmployees بالنسخة 1.0:

{% include image.html url="assets/files/article_09/postman-getemployees-v1.png" border="1" %}

هي نفسها لو أستدعينا بالنسخة 1.1 حيث أننا لم نقم بإضافة تطبيق جديد للعملية في نسخة 1.1:

{% include image.html url="assets/files/article_09/postman-getemployees-v1_1.png" border="1" %}
