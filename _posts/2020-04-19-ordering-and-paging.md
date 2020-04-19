---
layout: post
title: "4. Ordering and Paging Results"
date: 2020-04-19
lang: ar-SA
index: 4
comments: true
---

في هذا الدرس سوف نتعلم بمشيئة الله كيف نرتب order النتائج وكيف نجزئها على صفحات paging.

## ترتيب النتائج Ordering Results

جميع تعديلاتنا في هذا الدرس سوف تكون على ()GetEmployees في EmployeesController وذلك لأنها هي الدالة التي ترجع لنا أكثر من قيمة وعليه يمكن ترتيبها وتتجزئتها. أول تعديل نقوم به هو جعلها تستقبل arguments تحدد أي خانه في الـ entity نرغب الترتيب بناءاً عليها (orderBy) وهل الترتيب هذا تصاعدي م تنازلي (orderType).

```charp
[HttpGet]
public async Task<ActionResult<IEnumerable<EmployeeDetailsDto>>> GetEmployees(string orderBy, string orderType)
{
	var employeeEntities = await _repo.GetAllAsync(orderBy ?? "FirstName", orderType ?? "Asc");

	var employeeDetailsDtos = _mapper.Map<List<EmployeeDetailsDto>>(employeeEntities);

	return Ok( employeeDetailsDtos );
}
```

في داخل هذه الدالة، نتأكد أولاً من أنه تم فعلاً تمرير القيمتين orderBy و orderType في الـ query string أو نمرر قيّم إفتراضية ثم نمرر هذه القيّم الى ()GetAllAsync.

نعدل الآن على ()GetAllAsync في IRepository لتستقبل orderBy و orderType:

```csharp
public interface IRepository<T> where T : class
{ 
	Task<List<T>> GetAllAsync(string orderBy, string orderType);
	...
}
```

والآن جاء دور التعديل على ()GetAllAsync في EfRepository لتستقبل هي أيضاً orderBy و orderType وتقوم بالمنطق logic الخاص بترتيب النتائج:

```csharp
public async Task<List<TEntity>> GetAllAsync(string orderBy, string orderType)
{
	var table = _table as IQueryable<Entities.Employee>;

	switch (orderBy)
	{
		case "FirstName":
			table = (orderType == "Asc") ? table.OrderBy(t => t.FirstName) : table.OrderByDescending(t => t.FirstName);
			break;

		case "LastName":
			table = (orderType == "Asc") ? table.OrderBy(t => t.LastName) : table.OrderByDescending(t => t.LastName);
			break;
			
		case "Id":
			table = (orderType == "Asc") ? table.OrderBy(t => t.Id) : table.OrderByDescending(t => t.Id);
			break;
			
		default:
			table = table.OrderBy(t => t.EnrollmentDate);
			break;
	}

	return await table.ToListAsync() as List<TEntity>;
}
```

لنقم الآن ببناء المشروع. وبما أننا لم نضف NuGet packages جديدة منذ آخر مرة قمنا فيها ببياء المشروع فإنه بإمكاننا إستخدام الأمر التالي لتسريع عملية ابناء:

```csharp
dotnet build --no-restore
```

نقوم بعد ذلك تشغيله:

```csharp
dotnet run
```

ولنجرب في Postman بعض العناوين التالية لنرى النتائج:

```html
https://localhost:5001/api/employees?orderBy=FirstName&orderType=Desc
```

الترتيب بالإسم الأول بشكل تنازلي

```html
https://localhost:5001/api/employees?orderBy=LastName
https://localhost:5001/api/employees?orderBy=LastName&orderType=Asc
```

ستعيدان لنا نفس النتجية مرتبة حسب الإسم الأخير وشكل تصاعدي

```html
https://localhost:5001/api/employees?orderBy=Id
```

سوف تكون النتائج مرتبة حسب رقم الموظف وبشكل تصاعدي

```html
https://localhost:5001/api/employees
```

النتائج المسترجعة ستكون بناء على تاريخ الإلتحاق بالوظيفة وبشكل تصاعدي.

## تجزئة النتائج وإرجاعها على صفحات Paging

حالياً، ()GetEmployees في EmployeesController تعيد سجلات جميع الموظفين في النظام. ولكن تخيل كيف سيكون أداء الخدمة web service في حالة وجود الآف الموظفين؟ فهل من المنطق أن نعيد جميع السجلات مرة واحدة بغض النظر عن ما إذا كان الـمستخدم client بحاجتها جميعاً أم لا؟ هنا تأتي أهمية تجزئة وتقسيم النتائج على صفحات يحدد المستخدم عدد السجلات التي يرغب في إظهارها في كل صفحة. ولعمل ذلك نقوم بالتالي:

ننشئ داخل المجلد Data الملف PaginatedList.cs وهو مأخوذ من الأساس من الرابط التالي مع بعض التعديل:

```html
https://docs.microsoft.com/en-us/aspnet/core/data/ef-mvc/sort-filter-page?view=aspnetcore-3.1#add-paging-to-students-index
```

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;

namespace aspnetcorewebapiproject.Data
{
    public class PaginatedList<T>
    {
        public int      PageIndex   { get; private set; }
        public int      PageSize    { get; private set; }
        public int      TotalPages  { get; private set; }
        public int      ItemsCount  { get; private set; }
        public List<T>  Items       { get; private set; } = new List<T>();

        public bool HasPreviousPage
        {
            get { return (PageIndex > 1); }
        }

        public bool HasNextPage
        {
            get { return (PageIndex < TotalPages); }
        }

        public PaginatedList(List<T> items, int count, int pageIndex, int pageSize)
        {
            Items.AddRange(items);

            ItemsCount  = count;
            PageIndex   = pageIndex;
            PageSize    = pageSize;
            TotalPages  = (int)Math.Ceiling(ItemsCount / (double)PageSize);            
        }

        public static async Task<PaginatedList<T>> CreateAsync(IQueryable<T> source, int pageIndex, int pageSize)
        {
            var count = await source.CountAsync();
            var items = await source.Skip((pageIndex - 1) * pageSize).Take(pageSize).ToListAsync();
            
            return new PaginatedList<T>(items, count, pageIndex, pageSize);
        }
    }
}
```

نعدل الآن على ()GetAllAsync في IRepository ونجعلها تستقبل arguments تحدد أي صفحة نود عرضها وكم عدد السجلات المعروضة في الصفحة الواحدة وتعيد PaginatedList بدلاً من List:

```csharp
public interface IRepository<T> where T : class
{ 
	Task<PaginatedList<T>> GetAllAsync(string orderBy, string orderType, int pageIndex, int pageSize);
	...
} 
```

والآن التعديل سوف يكون على ()GetAllAsync في EfRepository. ولكن قبل القيام بذلك يجب علينا القيام ببعض التحسينات على التطبيق implementation السابق وتلافي القصور الموجود فيه. ففي الكود السابق لنتمكن من إستخدام الكود:

```csharp
OrderBy(t => t.FirstName)
// أو
OrderByDescending(t => t.FirstName)
```

كان يجب علينا تحويل table_ الى IQueryable من نوع Entities.Employee وعدم الإكتفاء بالنوع العام generic:

```csharp
_table as IQueryable<Entities.Employee>
```

ثم في آخر الدالة أضطررنا من جديد لعمل casting وتحويل النتيجة الى `List<TEntity>`.

ولتلافي هذا القصور، سوف نضبف الباقة Dynamic Linq لنتمكن من تمرير نص string الى OrderBy وليس expression وذلك عن طريق الأمر:

```csharp
dotnet add package System.Linq.Dynamic.Core
```

ثم نقوم بالتعديل التالي:

```csharp
using System.Linq.Dynamic.Core;

...

public async Task<PaginatedList<TEntity>> GetAllAsync(string orderBy, string orderType, int pageIndex, int pageSize)
{
	var table = _table as IQueryable<TEntity>;

	string orderByCriteria = new List<string> 
				{"FirstName", "LastName", "Id"}
					.Contains(orderBy) ? orderBy : "EnrollmentDate";

	if(orderType != "Asc")        
		orderByCriteria += " DESC";            

	table = table.OrderBy(orderByCriteria);

	return await PaginatedList<TEntity>.CreateAsync(table.AsNoTracking(), pageIndex, pageSize); 
}
```

أهم النقاط في الكود:

* نتعامل الآن مع النسخة العامة generic من IQueryable<TEntity>
* OrderBy تأخذ نص string بإسم الـ property الذي سيبنى عليه الترتيب وليس lambda expresion
* Dynamic Linq لا تحتوي على دالة ()OrderByDescending إنما ()OrderBy فقط ونضيف نهاية إسم الـ property عبارة DESC للدلالة على أن الترتيب تنازلي

يتبقى لنا الآن التعديل على ()GetEmployees في EmployeesController      ونمرر القيم داخل ()GetAllAsync:

```csharp
// GET: api/Employees
[HttpGet]
public async Task<ActionResult<PaginatedList<EmployeeDetailsDto>>> GetEmployees(string orderBy, string orderType, int? pageIndex, int? pageSize)
{
	var pagedEntities = await _repo.GetAllAsync(orderBy ?? "FirstName", orderType ?? "Asc", pageIndex ?? 1, pageSize ?? 3);

	var employeeDetailsDtos = _mapper.Map<List<EmployeeDetailsDto>>(pagedEntities.Items);

	var pagedDtos = new PaginatedList<EmployeeDetailsDto>(employeeDetailsDtos, pagedEntities.ItemsCount, pagedEntities.PageIndex, pagedEntities.PageSize);

	return Ok( pagedDtos );
}
```