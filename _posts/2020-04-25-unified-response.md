---
layout: post
title: "5. Unified Response"
date: 2020-04-25
lang: ar-SA
index: 5
comments: true
---

في هذا الدرس سنقوم بتوحيد النتائج المسترجعة response من الخدمة. نلاححظ أن بعظ العمليات تعيد لنا قائمة (GetEmployees) وبعضها تعيد لنا قيمة واحدة (GetEmployee و PostEmployee و DeleteEmployee) بينما العملية الأخير لا تعيد أي قيمة على الإطلاق (PutEmployee).

## إنشاء الـ Class الخاص بالـ Response

داخل المجلد Models\Employees نقوم بإنشاء ملف جديد بإسم EmployeesResponse.cs ويحتوي على الكود التالي:

```csharp
using System;

namespace aspnetcorewebapiproject.Models.Employees
{
    public class EmployeesResponse<T>
    {
        public bool IsSuccessful { get; set; }
        public int Status { get; set; }
        public string Message { get; set; } 
        public T Data { get; set; }
    }
}
```

وفيما يلي شرح للـ properties المستخدمة:


| Property | الوصف |
|---:|---:|
| **IsSuccessful** | هل تم تنفيذ العملية بنجاح؟ |
| **Status** | الـ Http Status Code المسترجع من تنفيذ العملية |
| **Message** | في حالة لم ينجح التنفيذ فإنه يتم حفظ الرسالة هنا |
| **Data** | البيانات المسترجعة |

ونبدأ الآن إستخدامها في العمليات الخاصة بـ EmployeesController.

## GetEmployees

نعدل على العملية لتصبح بالشكل التالي:

```csharp
public async Task<ActionResult<EmployeesResponse<PaginatedList<EmployeeDetailsDto>>>> GetEmployees(string orderBy, string orderType, int? pageIndex, int? pageSize)
{
	var pagedEntities = await _repo.GetAllAsync(
			orderBy     ?? "FirstName", 
			orderType   ?? _config.GetValue<string>("ResponseDefaults:OrderType"),
			pageIndex   ?? _config.GetValue<int>("ResponseDefaults:PageIndex"), 
			pageSize    ?? _config.GetValue<int>("ResponseDefaults:PageSize")
		);

	var employeeDetailsDtos = _mapper.Map<List<EmployeeDetailsDto>>(pagedEntities.Items);

	var pagedDtos = new PaginatedList<EmployeeDetailsDto>(employeeDetailsDtos, pagedEntities.ItemsCount, pagedEntities.PageIndex, pagedEntities.PageSize);

	var response = new EmployeesResponse<PaginatedList<EmployeeDetailsDto>>
	{
		IsSuccessful = true,
		Status = 200,
		Message = string.Empty,
		Data = pagedDtos
	};

	return Ok( response );
}
```

ونلاحظ أن نوع القيمة المعادة أصبحت على الشكل التال:

```csharp
Task<ActionResult<EmployeesResponse<PaginatedList<EmployeeDetailsDto>>>>
```

وفيما يلي مثال للقيمة المسترجعة بعد التنفيذ بواسطة Postman:

```json
{
    "isSuccessful": true,
    "status": 200,
    "message": "",
    "data": {
        "pageIndex": 2,
        "pageSize": 3,
        "totalPages": 2,
        "itemsCount": 5,
        "items": [
            {
                "id": 2005,
                "firstName": "Ali",
                "lastName": "Saleh",
                "isManager": false,
                "enrollmentDate": "2012-10-03T00:00:00"
            },
            {
                "id": 2006,
                "firstName": "Alaa",
                "lastName": "Saleh",
                "isManager": false,
                "enrollmentDate": "2012-10-03T00:00:00"
            }
        ],
        "hasPreviousPage": true,
        "hasNextPage": false
    }
}
```

### GetEmployee

نعدل على العملية لتصبح بالشكل التالي:

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployee(int id)
{
	var response = new EmployeesResponse<EmployeeDetailsDto>();
	response.IsSuccessful = true;

	var employeeEntity = await _repo.GetAsync(id);

	if (employeeEntity == null)
	{
		response.Status = 404;
		response.Message = "Employee not found";

		return NotFound( response );
	}

	var employeeDetailsDto = _mapper.Map<EmployeeDetailsDto>(employeeEntity);

	response.Status = 200;
	response.Data = employeeDetailsDto;

	return Ok( response );
}
```

ونلاحظ أن نوع القيمة المعادة أصبحت على الشكل التال:

```csharp
Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>>
```

وفيما يلي مثال للقيمة المسترجعة بعد التنفيذ بواسطة Postman:

```json
{
    "isSuccessful": true,
    "status": 200,
    "message": null,
    "data": {
        "id": 2003,
        "firstName": "Khalid",
        "lastName": "Alhabish",
        "isManager": false,
        "enrollmentDate": "2014-01-19T00:00:00"
    }
}
```

### PutEmployee

سابقاً كتنت هذه العملية لا تعيد لنا قيمة حيث أن Http Status Code كان 204 أي No Content. ولكن لو عدنا الى الـ method definitions في w3.org لرأينا أنه بإمكان الـ Put أن تعيد إما:

* 204 No Content أو
* Ok 200

<https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html#sec9.6>

وعلى ذلك سوف نعدل على القيمة المسترجعه في حالة النجاح لتصبح Ok 200 في حالة نجاح العملية:

```csharp
[HttpPut("{id}")]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> PutEmployee(int id, EmployeeUpdateDto employeeUpdateDto)
{
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
	catch (Exception)
	{
		if (await _repo.GetAsync(id) == null)     
		{
			response.IsSuccessful = false;
			response.Status = 404;
			response.Message = "Employee not found";

			return NotFound(response);
		}

		response.IsSuccessful = false;
		response.Status = 500;
		response.Message = "Internal server error";

		return StatusCode(StatusCodes.Status500InternalServerError, response);                
	}            
}
```

ونلاحظ أن نوع القيمة المعادة أصبحت على الشكل التالي:

```csharp
Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>>
```

### PostEmployee

نعدل على العملية لتصبح بالشكل التالي:

```csharp
[HttpPost]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> PostEmployee(EmployeeInsertDto employeeInsertDto)
{
	var employeeEntity = _mapper.Map<Employee>(employeeInsertDto);

	await _repo.InsertAsync(employeeEntity);

	var employeeDetailsDto = _mapper.Map<EmployeeDetailsDto>(employeeEntity);

	var response = new EmployeesResponse<EmployeeDetailsDto>
	{
		IsSuccessful = true,
		Status = 201,
		Message = string.Empty,
		Data = employeeDetailsDto
	};

	return CreatedAtAction(nameof(GetEmployee), new { id = employeeDetailsDto.Id }, response);
}
```

ونلاحظ أن نوع القيمة المعادة أصبحت على الشكل التالي:

```csharp
Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>>
```

### DeleteEmployee

نعدل على العملية لتصبح بالشكل التالي:

```csharp
[HttpDelete("{id}")]
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> DeleteEmployee(int id)
{
	var response = new EmployeesResponse<EmployeeDetailsDto>();

	Employee employeeEntity = await _repo.DeleteAsync(id);

	if (employeeEntity == null)
	{
		response.IsSuccessful = false;
		response.Status = 404;
		response.Message = "Employee not found";

		return NotFound(response);            
	}
	
	var employeeDetailsDto = _mapper.Map<EmployeeDetailsDto>(employeeEntity);

	response.IsSuccessful = true;
	response.Status = 200;
	response.Data = employeeDetailsDto;

	return Ok( response );
}
```

ونلاحظ أن نوع القيمة المعادة أصبحت على الشكل التالي:

```csharp
Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>>
```
