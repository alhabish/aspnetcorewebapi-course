---
layout: post
title: "10. Documenting the API"
date: 2020-06-06
lang: ar-SA
index: 10
comments: true
---

في هذا الدرس سنتعلم كيف نوثق document الـ API التي قمنا بتطويرها، وبدلاً من كتابتها يدوياً سنستخدم مكتبة Swashbuckle لمساعدتنا في ذلك.

## إعداد المشروع 

### إضافة المكتبة البرمجية

```bash
dotnet add package Swashbuckle.AspNetCore
```

### إنشاء classes للإعدادات

إنشئ مجلد جديد بإسم Swagger وبداخله ملف بإسم ConfigureSwaggerOptions.cs محتواه في الأساس من الصفحة التالية:

[ConfigureSwaggerOptions.cs](https://github.com/microsoft/aspnet-api-versioning/blob/master/samples/aspnetcore/SwaggerSample/ConfigureSwaggerOptions.cs)

وبعد القيام بالتعديلات المناسبة يكون محتوى الملف:

```csharp
using Microsoft.AspNetCore.Mvc.ApiExplorer;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;
using System;

namespace aspnetcorewebapiproject.Swagger
{
    // From: https://github.com/microsoft/aspnet-api-versioning/blob/master/samples/aspnetcore/SwaggerSample/ConfigureSwaggerOptions.cs

    /// <summary>
    /// Configures the Swagger generation options.
    /// </summary>
    /// <remarks>This allows API versioning to define a Swagger document per API version after the
    /// <see cref="IApiVersionDescriptionProvider"/> service has been resolved from the service container.</remarks>
    public class ConfigureSwaggerOptions : IConfigureOptions<SwaggerGenOptions>
    {
        readonly IApiVersionDescriptionProvider provider;

        /// <summary>
        /// Initializes a new instance of the <see cref="ConfigureSwaggerOptions"/> class.
        /// </summary>
        /// <param name="provider">The <see cref="IApiVersionDescriptionProvider">provider</see> used to generate Swagger documents.</param>
        public ConfigureSwaggerOptions( IApiVersionDescriptionProvider provider ) => this.provider = provider;

        /// <inheritdoc />
        public void Configure( SwaggerGenOptions options )
        {
            // add a swagger document for each discovered API version
            // note: you might choose to skip or document deprecated API versions differently
            foreach ( var description in provider.ApiVersionDescriptions )
            {
                options.SwaggerDoc( description.GroupName, CreateInfoForApiVersion( description ) );
            }
        }

        static OpenApiInfo CreateInfoForApiVersion( ApiVersionDescription description )
        {
            var info = new OpenApiInfo()
            {
                Title = "Employees API",
                Version = description.ApiVersion.ToString(),
                Description = "A sample ASP.NET Core Web API project",
                Contact = new OpenApiContact() { Name = "The API Dev Company", Email = "email@email.com" },
                // License = new OpenApiLicense() { Name = "MIT", Url = new Uri( "https://opensource.org/licenses/MIT" ) }
            };

            if ( description.IsDeprecated )
            {
                info.Description += " This API version has been deprecated.";
            }

            return info;
        }
    }
}
```

بعد ذلك نضيف الملف SwaggerDefaultValues.cs والذي يمكن إيجاده في الرابط التالي:

[SwaggerDefaultValues.cs](https://github.com/microsoft/aspnet-api-versioning/blob/master/samples/aspnetcore/SwaggerSample/SwaggerDefaultValues.cs)

```csharp
using Microsoft.AspNetCore.Mvc.ApiExplorer;
using Microsoft.OpenApi.Any;
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;
using System.Linq;

namespace aspnetcorewebapiproject.Swagger
{
    // From: https://github.com/microsoft/aspnet-api-versioning/blob/master/samples/aspnetcore/SwaggerSample/SwaggerDefaultValues.cs

    /// <summary>
    /// Represents the Swagger/Swashbuckle operation filter used to document the implicit API version parameter.
    /// </summary>
    /// <remarks>This <see cref="IOperationFilter"/> is only required due to bugs in the <see cref="SwaggerGenerator"/>.
    /// Once they are fixed and published, this class can be removed.</remarks>
    public class SwaggerDefaultValues : IOperationFilter
    {
        /// <summary>
        /// Applies the filter to the specified operation using the given context.
        /// </summary>
        /// <param name="operation">The operation to apply the filter to.</param>
        /// <param name="context">The current operation filter context.</param>
        public void Apply( OpenApiOperation operation, OperationFilterContext context )
        {
            var apiDescription = context.ApiDescription;

            operation.Deprecated |= apiDescription.IsDeprecated();

            if ( operation.Parameters == null )
            {
                return;
            }

            // REF: https://github.com/domaindrivendev/Swashbuckle.AspNetCore/issues/412
            // REF: https://github.com/domaindrivendev/Swashbuckle.AspNetCore/pull/413
            foreach ( var parameter in operation.Parameters )
            {
                var description = apiDescription.ParameterDescriptions.First( p => p.Name == parameter.Name );

                if ( parameter.Description == null )
                {
                    parameter.Description = description.ModelMetadata?.Description;
                }

                if ( parameter.Schema.Default == null && description.DefaultValue != null )
                {
                    parameter.Schema.Default = new OpenApiString( description.DefaultValue.ToString() );
                }

                parameter.Required |= description.IsRequired;
            }
        }
    }
}
```

### التعديل على ConfigureServices في Startup.cs

نقوم بالإضافات التالية في نهاية الدالة:

```csharp
public void ConfigureServices(IServiceCollection services)
{
	...

	services.AddApiVersioning(o => {
		o.DefaultApiVersion = new ApiVersion(1, 0);                
		o.AssumeDefaultVersionWhenUnspecified = true;
		// reporting api versions will return the headers "api-supported-versions" and "api-deprecated-versions"
		o.ReportApiVersions = true;      
	});
	
	services.AddVersionedApiExplorer(o => {
		// add the versioned api explorer, which also adds IApiVersionDescriptionProvider service
		// note: the specified format code will format the version as "'v'major[.minor][-status]"
		o.GroupNameFormat = "'v'VVV";

		// note: this option is only necessary when versioning by url segment. the SubstitutionFormat
		// can also be used to control the format of the API version in route templates
		o.SubstituteApiVersionInUrl = true;
	});

	services.AddTransient<IConfigureOptions<SwaggerGenOptions>, ConfigureSwaggerOptions>();

	// Register the Swagger generator, defining 1 or more Swagger documents
	services.AddSwaggerGen(o =>
	{
		// add a custom operation filter which sets default values
		o.OperationFilter<SwaggerDefaultValues>();

		o.ResolveConflictingActions( apiDescriptions => apiDescriptions.First() );

		// Set the comments path for the Swagger JSON and UI.
		var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
		var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
		o.IncludeXmlComments(xmlPath);                
	});        
}
```

ولا ننسى إضافة الـ namespaces التالية:

```csharp
using Microsoft.Extensions.Options;
using Swashbuckle.AspNetCore.SwaggerGen;
using aspnetcorewebapiproject.Swagger;
using System.Reflection; 
using System.IO;
```

### التعديل على ملف csproj

لتمكين الـ xml comments نضيف الأسطر التالية لملف الـ csproj:

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
  <NoWarn>$(NoWarn);1591</NoWarn>
</PropertyGroup>
```

### التعديل على Configure في Startup.cs

نضيف أولاً الـ namespace التالي:

```csharp
using Microsoft.AspNetCore.Mvc.ApiExplorer;
```

ثم نمرر argument من نوع IApiVersionDescriptionProvider للدالة ()Configure:

```csharp
public void Configure(..., IApiVersionDescriptionProvider provider)
{
	...
}
```

والتعديل الأخير سيكون في نهاية الدالة حيث نقوم بإضافة ما يلي:

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILogger<Startup> logger, IApiVersionDescriptionProvider provider)
{
	...

	// Enable middleware to serve generated Swagger as a JSON endpoint.
	app.UseSwagger();

	// Enable middleware to serve swagger-ui (HTML, JS, CSS, etc.),
	// specifying the Swagger JSON endpoint.
	app.UseSwaggerUI(c =>
	{
		c.RoutePrefix = string.Empty;

		// build a swagger endpoint for each discovered API version
		foreach ( var description in provider.ApiVersionDescriptions )
		{
			c.SwaggerEndpoint( $"/swagger/{description.GroupName}/swagger.json", description.GroupName.ToUpperInvariant() );
		}
	});      
}
```

## إضافة وصف للـ EmployeeController

سنكتب وصف لجميع العمليات ومدخلاتها ومخرجتها ليتم عرضها في واجهة Swagger بإتباع التعديلات التالية:

### إضافة وصف للـ EmployeesController class

```csharp
/// <summary>
/// Contains CRUD operations on the Employee entity
/// </summary> 
[ApiVersion("1.0")]
[ApiVersion("1.1")]
[Route("api/v{version:apiVersion}/[controller]")]
[EnableCors("CorsPolicy")]
[ApiController] 

public class EmployeesController : ControllerBase
{
	...
}
```

### إضافة وصف لـ GetVersion

```csharp
/// <summary>
/// Retrieve controller version
/// </summary>
/// <returns>The controller version</returns>
/// <response code="200">Returns the controller version</response>

[HttpGet("version")]
[ProducesResponseType(typeof(string), StatusCodes.Status200OK)]

public string GetVersion() => HttpContext.GetRequestedApiVersion().ToString();
```

### إضافة وصف لـ GetEmployees

```csharp
/// <summary>
/// Retrieve all Employee items
/// </summary>
/// <param name="employeeGetDto">Ordering and paging criteria</param>
/// <returns>A list of Employee items</returns>
/// <response code="200">Returns the items</response>

// GET: api/Employees
[HttpGet]
[ProducesResponseType(typeof(EmployeesResponse<PaginatedList<EmployeeDetailsDto>>), StatusCodes.Status200OK)]

public async Task<ActionResult<EmployeesResponse<PaginatedList<EmployeeDetailsDto>>>> GetEmployees([FromQuery] EmployeeGetDto employeeGetDto)
{            
	...
}
```

### إضافة وصف لـ GetEmployee

```csharp
/// <summary>
/// Retrieve a specific Employee item
/// </summary>
/// <param name="id">Id of item to retrieve</param>
/// <returns>An Employee item</returns>
/// <response code="200">Returns the item</response>
/// <response code="404">If the item is not found</response> 

// GET: api/Employees/5
[HttpGet("{id}")]
[ProducesResponseType(typeof(EmployeesResponse<EmployeeDetailsDto>), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]

public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployee(int id)
{
	...               
}
```

### إضافة وصف لـ GetEmployeeV1_1

```csharp
/// <summary>
/// Retrieve a specific Employee item
/// </summary>
/// <param name="id">Id of item to retrieve</param>
/// <returns>An Employee item</returns>
/// <response code="200">Returns the item</response>
/// <response code="404">If the item is not found</response> 

// GET: api/Employees/5
[HttpGet("{id}")]
[MapToApiVersion("1.1")]
[ProducesResponseType(typeof(EmployeesResponse<EmployeeDetailsDto>), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)] 
  
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployeeV1_1(int id)
{            
	...             
}
```

### إضافة وصف لـ PostEmployee

```csharp
/// <summary>
/// Creates an Employee item
/// </summary>
/// <param name="employeeInsertDto">Employee info to create</param>
/// <returns>A newly created Employee item</returns>
/// <response code="201">Returns the newly created item</response>
/// <response code="400">If the item is null</response> 

// POST: api/Employees
[HttpPost]
[ProducesResponseType(typeof(EmployeesResponse<EmployeeDetailsDto>), StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]

public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> PostEmployee(EmployeeInsertDto employeeInsertDto)
{
	...
}
```

### إضافة وصف لـ PutEmployee

```csharp
/// <summary>
/// Update an Employee item
/// </summary>
/// <param name="id">Id of item to update</param>        
/// <param name="employeeUpdateDto">Employee info to update</param>
/// <returns>An updated Employee item</returns>
/// <response code="200">Returns the newly updated item</response>
/// <response code="400">If the inputs are invalid</response> 
/// <response code="404">If the item is not found</response> 

// PUT: api/Employees/5
[HttpPut("{id}")]
[ProducesResponseType(typeof(EmployeesResponse<EmployeeDetailsDto>), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status400BadRequest)] 
[ProducesResponseType(StatusCodes.Status404NotFound)]  
             
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> PutEmployee(int id, EmployeeUpdateDto employeeUpdateDto)
{
	...
}
```

### إضافة وصف لـ DeleteEmployee

```csharp
/// <summary>
/// Deletes a specific Employee item
/// </summary>
/// <param name="id">Id of item to delete</param>   
/// <response code="200">Returns the deleted item</response>
/// <response code="404">If the item is not found</response> 

// DELETE: api/Employees/5
[HttpDelete("{id}")]
[ProducesResponseType(typeof(EmployeesResponse<EmployeeDetailsDto>), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]

public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> DeleteEmployee(int id)
{
	...
}
```

## إستعراض الـ Swagger Documentation

بإمكانك إستعراض المستند الذي تم إنشاؤه على الرابط التالي:

<https://localhost:5001>

سترى جميع الـ controllers بعملياتها ونسخها المختلفة:

{% include image.html url="assets/files/article_10/browser-swagger.png" border="1" %}
