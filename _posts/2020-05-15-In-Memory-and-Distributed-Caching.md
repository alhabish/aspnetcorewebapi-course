---
layout: post
title: "7. In-Memory and Distributed Caching"
date: 2020-05-15
lang: ar-SA
index: 7
comments: true
---

خاصية الـ cashing تفيدنا في حفظ البيانات التي يكثر الطلب عليها في الذاكرة مما يقلل من الحاجة الى الذهاب الى مصدر المعلومة (قاعدة بيانات أو خدمة خارجية أو غيرها) ومعالجة هذه البيانات ثم إعادة الى المستخدم مما يساعدنا في تحسين أداء الخدمة التي نقدمها. 

في هذا الدرس سنتعلم طريقتين في الـ caching. الأولى in-memory وتعتمد على ذاكرة الجهاز أو السيرفر الذي تعمل عليه الخدمة، والطريقة الثانية distributed وسنستفيد من Redis فيها.

## In-Memory Caching

كما ذكرنا سابقاً، سنستفيد من الذاكرة كمخزن مؤقت للبيانات التي يكثر إستدعائها وسنلاحظ تحسن في الأداء حيث لم يعد هنالك داعي لجلب البيانات من مصدرها في كل مرة. ولعمل ذلك نقوم بالتالي:

### 1. العمل في branch جديد

سنطور هذه الخاصية خارج الـ master branch، ولعمل ذلك ننفذ الأمر التالي:

```bash
git checkout -b inmemory-cache
```

{% include image.html url="assets/files/article_07/vscode-inmemory-cache-branch.png" border="1" %}

### 2. التعديل على Startup.cs

نضيف السطر التالي على الدالة ()ConfigureServices في الملف Startup.cs:

```csharp
public void ConfigureServices(IServiceCollection services)
{
	...

	services.AddMemoryCache();
}
```

### 3. التعديل على EmployeesController.cs

نضيف namespace جديد ونعدل على الـ constructor ليستقبل متغير من نوع IMemoryCache:

```csharp
...
using Microsoft.Extensions.Caching.Memory;


public class EmployeesController : ControllerBase
{
	private IMemoryCache _cache;
	...

	public EmployeesController(IMemoryCache cache, ...)
	{
		_cache = cache;
		...
	}
}
```

ولإستخدامها في الدالة ()GetEmployee نقوم بالتالي:

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployee(int id)
{
	_logger.LogInformation("GetEmployee requested");

	if( !_cache.TryGetValue(id, out EmployeeDetailsDto employeeDetailsDto) )
	{
		var employeeEntity = await _repo.GetAsync(id);

		if (employeeEntity != null)                
			employeeDetailsDto = _mapper.Map<EmployeeDetailsDto>(employeeEntity);                

		_cache.Set(
			id,
			employeeDetailsDto,
			new MemoryCacheEntryOptions {
				SlidingExpiration = TimeSpan.FromMinutes(1)
			}
		);
	}
	
	var response = new EmployeesResponse<EmployeeDetailsDto>()
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

نتأكد أولاً من وجود البيانات في الـ cache بواسطة ()TryGet وإذا كانت القيمة مخزنة سيتم إحتوائها في response وإعادتها للمستخدم. وفي حالة لم تكن القيمة موجودة، سيتم إرجاعها من قاعدة البيانات وتخزينها في الـ cache بواسطة الأمر ()Set.

أستخدمنا هنا SlidingExpiration لتحديد متى يعتبر الـ cache منتهي وهي تستخدم لتحديد كم المدة التي يمكن للقيمة الا تستدعى قبل تحديثها من جديد وستظل القيمة موجودة في الـ cache طالما كان هنالك طلب عليها خلال المدة الزمنية المحددة. أي أنه طالما لم تمر الفترة الزمنية المحددة (دقيقة في مثالنا هذا) ستظل القيمة موجودة في الـ cache.

وبالإمكان لإستخدام AbsoluteExpiration بدلاً من SlidingExpiration وذلك لتحديد مدة إنتهاء القيمة في الـ cache سواء تم الإستفادة منها أم لا.

والان، إذا جربنا عن طريق Postman نرى أنه أخذت عملية التنفيذ أول مرة 3 ثواني ونصف تقريباً:

{% include image.html url="assets/files/article_07/postman-inmemory-fresh-data.png" border="1" %}

وعند تنفيذ نفس الأمر بنفس الـ argument في مدة لا تتجاوز الدقيقة نرى أن العملية أستغرقت وقت أقل بكثير، تحديداً 18 جزء من الثانية:

{% include image.html url="assets/files/article_07/postman-inmemory-cached-data.png" border="1" %}

### 4. حفظ التعديلات

نضيف الآن هذه التعديلات الى git:

```bash
git add .
git commit -m "adds in-memory cache support to GetEmployees()"
```

## Distributed Caching

تعتبر الطريقة السابقة في الـ caching خيار جيد في حالة أن الخدمة مستضافة على خادم server واحد, ولكن في حالة إستضافة هذه الخدمة على أكثر من خادم أو في الـ cloud فإنه لا يمكن إستخدامها. هنا يأتي دور الـ distributed caching. وسنستفيد من redis كـ key value store للقيام بذلك.

{% include image.html url="assets/files/article_07/redis.png" border="1" %}

### 1. العمل في branch جديد

نعود الآن الى الـ master branch:

```bash
git branch master
```

ثم ننشئ branch جديد ونتحول عليه:

```bash
git branch distributed-cache
git checkout distributed-cache
```

### 2. تثبيت Docker

redis في الأساس تطبيق يعمل على Linux ومدعوم بشكل محدود على Windows ولذلك سنستخدم Docker للتعامل مع redis container.

{% include image.html url="assets/files/article_07/docker.png" border="1" %}

يمكن تحميل Docker من الرابط التالي ولكن يجب أن تكون نسخة وندوز Professional على الأقل:

https://hub.docker.com/editions/community/docker-ce-desktop-windows

في حالة لم يكن الوندوز Professional أو Enterprise، فبالإمكان تحميل Virtual Machine مثل Virtual Box أو VM Ware وتحميل أحد توزيعات Linux ثم تحميل Docker. والحل الثاني هو تحميل Docker هلى Windows Subsystem for Linux - WSL.

الآن في قائمة Turn Windows features on or off يجب علينا التأكد من إختيار Containers و Hyper-V:

{% include image.html url="assets/files/article_07/turn-windows-features-on-or-off.png" border="1" %}

بعد الإنتهاء من تثبيت Docker فإنه يمكن التأكد من أن عملية التثبيت تمت بشكل صحيح بكتابة الأمر التالي في الـ command prompt:

```bash
docker run hello-world
```

وإذا ظهرت نتائج كالتالي فإنه يعمل بشكل صحيح:

{% include image.html url="assets/files/article_07/docker-hello-world.png" border="1" %}

الآن نسحب الـ redis image من الـ docker registry:

```bash
docker pull redis
```

{% include image.html url="assets/files/article_07/docker-pull-redis.png" border="1" %}

ونقوم الآن بتشغيلها:

```bash
docker run --name redis-store -p 6379:6379 -d redis
```

{% include image.html url="assets/files/article_07/docker-run.png" border="1" %}

وللتأكد من أن الـ image تعمل نكتب الأمر التالي:

```bash
docker ps
```

{% include image.html url="assets/files/article_07/docker-ps.png" border="1" %}

سنقوم الآن بالتأكد من مقدرتنا للوصول الى الـ Redis Command Line Interface أو redis-cli. ونقوم بذلك عن طريق فتح الـ bash command line في الـ container الذي أنشأناه ثم فتح الـ redis-cli:

```bash
docker exec -it redis-store /bin/bash
```

ثم:

```bash
redis-cli
```

ثم نكتب ping وإذا أعاد الينا PONG فإننا تمكنا من التعامل مع redis-cli بشكل صحيح:


{% include image.html url="assets/files/article_07/redis-ping-pong.png" border="1" %}

وفيما يلي بعض الأوامر التي يمكن تنفيذها:

{% include image.html url="assets/files/article_07/redis-commands.png" border="1" %}

```bash
// لحفظ قيمة في الذاكرة بشكل دائم
set <key-name> <value>

// لجلب القيمة من الذاكرة
get <key-name>

// لحذف القيمة من الذاكرة
del <key-name>

// لحفظ قيمة في الذاكرة لعدد معين من الثواني
set <key-name> <value> EX <num-of-seconds>

// لطباعة جميع الـ keys
keys *
```

### 2. إضافة package جديد للمشروع

نضيف الآن الـ package المتعلق بالـ distributed caching:

```bash
dotnet add Microsoft.Extensions.Caching.StackExchangeRedis
```

### 3. التعديل على Startup.cs

الآن في Startup.cs نستدعي الـ namespace التالي:

```csharp
using Microsoft.Extensions.Caching.StackExchangeRedis;
```

وفي ()ConfigureServices نقوم بالتعديل اللذي يلي:

```csharp
...
using Microsoft.Extensions.Caching.StackExchangeRedis;

public void ConfigureServices(IServiceCollection services)
{
	...

	services.AddStackExchangeRedisCache(options =>
	{
		options.Configuration = "localhost";
	});
}

### 4. التعديل على EmployeesController.cs

في هذا الملف، نقوم بالتعديلات التالية:

```csharp
...
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public class EmployeesController : ControllerBase
{
	private IMemoryCache _cache;
	...

	public EmployeesController(IDistributedCache cache, ...)
	{
		_cache = cache;
		...
	}
}
```

ولإستخدامها في الدالة ()GetEmployee نقوم بالتالي:

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployee(int id)
{
	_logger.LogInformation("GetEmployee requested");

	var cachedResponse = await _cache.GetStringAsync(id.ToString());
	
	EmployeeDetailsDto employeeDetailsDto = null;

	if( string.IsNullOrEmpty(cachedResponse) )
	{                                 
		var employeeEntity = await _repo.GetAsync(id);

		if (employeeEntity != null)                
			employeeDetailsDto = _mapper.Map<EmployeeDetailsDto>(employeeEntity);                

		string serializedData = (employeeDetailsDto == null) ? string.Empty : JsonSerializer.Serialize(employeeDetailsDto);
		
		await _cache.SetStringAsync(
			id.ToString(),
			serializedData,
			new DistributedCacheEntryOptions {
				SlidingExpiration = TimeSpan.FromMinutes(1)
			}
		);
	}
	else
	{
		employeeDetailsDto = JsonSerializer.Deserialize<EmployeeDetailsDto>(cachedResponse);
	}
	
	var response = new EmployeesResponse<EmployeeDetailsDto>()
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

جرب ما قمنا بعملة بواسطة Postman ولاحظ كيف أنه عند تنفيذ العملية في المرة الثانية في مدة لا تتجاوز الدقيقة ستكون المدة أقل بشكل واضح. 

وبالإمكان أن نرى كيف أن Redis  حفظت القيمة لديها. فبعد التنفيذ في Postman بمدة لا تتجاوز الدقيقة (وهي المدة التي حددناها في الـ SlidingExpiration) نستطيع أن نرى الـ keys التي تم حفظها بالأمر `keys *` ثم بإمكاننا أن نستعرض القيمة المحفوظة بواسطة الأمر 

{% include image.html url="assets/files/article_07/docker-redis-hgetall.png" border="1" %}

### 5. إيقاف الـ image

بإمكاننا الآن إيقاف الـ docker image وحذفها بعد تطبيق هذا المثال:

```bash
docker stop redis-store
docker rm redis-store
```

### 5. حفظ التعديلات

نضيف الآن هذه التعديلات الى git:

```bash
git add .
git commit -m "adds distributed cache support to GetEmployees()"
```
