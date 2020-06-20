---
layout: post
title: "12. Securing the API - Bearer (Token) Authentication - JWT"
date: 2020-06-21
lang: ar-SA
index: 12
comments: true
---

في النوع السابق من المصادقة basic authentication كان يتوجب علينا إرسال إسم المستخدم وكلمة المرور عند تنفيذ أي عملية. وفي هذا الدرس سنتعلم نوع آخر من المصادقة مبني على إستخدام التوكن. وهو عبارة عن نص يحتوي على معلومات خاصة بالمستخدم الذي يحاول تنفيذ العملية من دون إحتواءه على كلمة المرور الخاصة بالمستخدم. هذا التوكن يتم إنشاؤه بواسطة private key موجود على الـ server ولذلك وطالما أخذنا بالإعتبار الإحتياطات الأمنية اللازمة فإنه يصعب تزوير هذا التوكن.

آلية إنشاء التوكن تكون بالشكل التالي: 

1)  يستدعي المستخدم العملية auth / login ممرراً إسم المستخدم وكلمة المرور الخاصة به

2)  يتم التحقق من صحة معلومات المستخدم وبعد ذلك سنعيد اليه توكن يمكن إستخدامه في تنفيذ باقي العمليات

3)  في نفس الوقت سيعطى المستخدم توكن للتحديث refresh token بحيث يمرره الى العملية auth / refresh لتحديث الـ jwt token من دون الحاجة الى إرسال إسم المستخدم وكلمة المرور مرة أخرى

سيتم تمرير التوكن في الـ Authorization header بالشكل التالي:

```bash
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjEiLCJuYW1laWQiOiIxIiwidW5pcXVlX25hOcGaL6YuL7nk
```

وهنا بشكل مختصر شرح للتوكن وللعملية بشكل عام:

{% include image.html url="assets/files/article_12/jwt.jpeg" border="1" %}

### العمل في branch مستقل

ننشئ branch جديد ونتحول اليه:

```bash
git branch jwt
git checkout jwt
```

### إضافة المكتبات البرمجية

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

## إنشاء الـ JWT Token

### إدارة المستفيدين من الخدمة

ننشئ ملف جديد بإسم User.cs يمثل المستفيد من الخدمة التي نقدمها داخل المجلد Entities:

```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace aspnetcorewebapiproject.Entities
{
    public class User : IHasCreatedAndLastModifiedDates
    {
        [Key]
        public int Id { get; set; }

        [Required]
        [StringLength(100)]
        public string Username { get; set; }

        [Required]
        [StringLength(100)]
        public string Password { get; set; }

        [Required]
        [EmailAddress]
        public string Email { get; set; }

        public DateTime LastModified { get; set; }
        public DateTime CreatedDate { get; set; }
    }
}
``` 

### إضافته الى الـ db context

نقوم بإضافة الـ entity الجديد User على MainDbContext كما يلي:

```csharp
public class MainDbContext : DbContext
{
	public DbSet<Entities.Employee> Employees { get; set; }

	public DbSet<Entities.User> Users { get; set; }

	protected override void OnModelCreating(ModelBuilder modelBuilder)
	{
		modelBuilder.Entity<Entities.User>().HasData(
			new Entities.User { Id = 1, Username = "ahmad", Password = "abc@123", Email = "ahmad@email.com" },
			new Entities.User { Id = 2, Username = "ali", Password = "def@456", Email = "ali@email.com" },
			new Entities.User { Id = 3, Username = "yousef", Password = "ghi@789", Email = "yousef@email.com" }
		);
	}

	...
```

في OnModelCreating طلبنا إضافة بعض البيانات في قاعدة البيانات.

نضيف أوامر التعديل على قاعدة البيانات:

```csharp
dotnet ef migrations add "Adds_Users_and_Sample_Data"
```

نقوم الآن بتنفيذ الأمر:

```bash
dotnet ef database update
```

نستخدم الآن Azure Data Studio لرؤية التعديلات التي تم تنفيذها:


{% include image.html url="assets/files/article_12/ads-users.png" border="1" %}

### إضاقة إعدادات في appsettings.json

نقوم بإضافة الـ key المستخدم للتشفير ومدة صلاحية الـ jwt token وهي 5 دقائق:

```javascript
  ... ,
  "JwtSettings": {
    "SecretKey": "SecretSecurityKey",
    "TokenLifeTime": "00:05:00"
  }    
}
```

### إضافة DTO

في المجلد Models / v1 / Auth  نضيف الملف LoginDto.cs والذي يحتوي على معلومات المستخدم المطلوبة عند تسجيل الدخول:

```csharp
namespace aspnetcorewebapiproject.Models.Auth.v1
{
    public class LoginDto
    {
        public string Username { get; set; }
        public string Password { get; set; }
    }
}
```

### إضافة AuthController

نقوم الآن بإضافة الـ controller المسؤول عن إنشاء التوكن token وذلك في المجلد Controllers / v1 كما يلي:

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.AspNetCore.Cors;
using System.Security.Claims;
using System.IdentityModel.Tokens.Jwt;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using aspnetcorewebapiproject.Models.Auth.v1;
using aspnetcorewebapiproject.DbContexts;
using System.Linq;

namespace aspnetcorewebapiproject.Controllers.v1
{
    /// <summary>
    /// Authenticates users and generates a jwt token
    /// </summary> 
    [ApiVersion("1.0")]
    [ApiVersion("1.1")]
    [Route("api/v{version:apiVersion}/[controller]")]
    [EnableCors("CorsPolicy")]
    [ApiController] 
    public class AuthController : ControllerBase
    {
        private readonly IConfiguration _config;
        private readonly MainDbContext _dbContext;

        public AuthController(IConfiguration config, MainDbContext dbContext)
        {
            _config = config;
            _dbContext = dbContext;
        }

        [HttpPost("login")]
        public async Task<IActionResult> GenerateToken([FromBody] LoginDto data)
        {
            if( string.IsNullOrWhiteSpace(data.Username) || string.IsNullOrWhiteSpace(data.Password) )
                return BadRequest();

            var user = await _dbContext.Users.SingleOrDefaultAsync(u => u.Username == data.Username && u.Password == data.Password);
          
            if( user != null )
            {
                TimeSpan tokenLifeTime = _config.GetValue<TimeSpan>("JwtSettings:TokenLifeTime");

                // User found. Create ticket for user
                var claims = new List<Claim> {
                    new Claim("id", user.Id.ToString()),
                    new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
                    new Claim(ClaimTypes.Name, user.Username),
                    new Claim(ClaimTypes.Email, user.Email),

                    new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),

                    // not before, this token is not valid before a certain date and time
                    // in our case we want it to be valid right away
                    new Claim(JwtRegisteredClaimNames.Nbf, new DateTimeOffset(DateTime.UtcNow).ToUnixTimeSeconds().ToString()),

                    // when will it expire
                    new Claim(JwtRegisteredClaimNames.Exp, new DateTimeOffset(DateTime.UtcNow).Add(tokenLifeTime).ToUnixTimeSeconds().ToString())
                };

                var creds = new SigningCredentials(
                        new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config.GetValue<string>("JwtSettings:SecretKey"))), 
                        SecurityAlgorithms.HmacSha256);

                var tokenDescriptor = new SecurityTokenDescriptor
                {
                    Subject = new ClaimsIdentity(claims),
                    Expires = DateTime.UtcNow.Add(tokenLifeTime),
                    SigningCredentials = creds
                };

                var tokenHandler = new JwtSecurityTokenHandler();
                SecurityToken token = tokenHandler.CreateToken(tokenDescriptor);

                return Ok( new { 
                    token = tokenHandler.WriteToken(token),
                });                
            }

            return BadRequest();
        }
    }
}
```

### التعديل على Startup.cs

لتفعيل ما قمنا به نقوم بالتالي:

#### التعديل على الدالة ()Configure

علينا إضافة ()app.UseAuthentication  قبل ()app.UseAuthorization:

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILogger<Startup> logger, IApiVersionDescriptionProvider provider)
{
	...

	app.UseAuthentication();

	app.UseAuthorization();

	...     
}
```

#### التعديل على ()ConfigureServices

نقوم بإضافة عملية التحقق ونعتمد التطبيق الموجود في BasicAuthenticationHandler:

```csharp
public void ConfigureServices(IServiceCollection services)
{
	...
	
	services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
	.AddJwtBearer(options => 
	{
		options.TokenValidationParameters = new TokenValidationParameters()
		{
			IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Configuration.GetValue<string>("JwtSettings:SecretKey"))),
			ValidateIssuer = false,
			ValidateAudience = false,
			ValidateLifetime = true,
			LifetimeValidator = LifetimeValidator
		};
	});
}
```

```csharp
private bool LifetimeValidator(DateTime? notBefore, DateTime? expires, SecurityToken token, TokenValidationParameters @params)
{
	if (expires != null)
		return expires > DateTime.UtcNow;
	
	return false;
}
```

والآن نخبر Swagger بأن الـ API تعتمد على JWT:

```csharp
public void ConfigureServices(IServiceCollection services)
{
	...
	            
	services.AddSwaggerGen(o =>
	{
		// add a custom operation filter which sets default values
		o.OperationFilter<SwaggerDefaultValues>();

		o.ResolveConflictingActions( apiDescriptions => apiDescriptions.First() );

		// Set the comments path for the Swagger JSON and UI.
		var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
		var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
		o.IncludeXmlComments(xmlPath);  
		
		// Swagger needs to know about JWT
		o.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
		{
			Description = "JWT Authorization header using the Bearer scheme.",
			Name = "Authorization",
			In = ParameterLocation.Header,
			Type = SecuritySchemeType.ApiKey,
			Scheme = "Bearer",
			BearerFormat = "JWT"
		});

		o.AddSecurityRequirement(new OpenApiSecurityRequirement(){
			{
				new OpenApiSecurityScheme
				{
					Reference = new OpenApiReference
					{
						Type = ReferenceType.SecurityScheme,
						Id = "Bearer"
					}
				},
				new List<string>()
			}
		});

	});            
}
```

### التعديل على EmployeesController

لجعل EmployeesController تتطلب التحقق من المستخدم قبل إمكانية الإستفادة منها فإنه يتوجب علينا إضافة الـ [Authorize] attribute:

```csharp
[Authorize]
...
public class EmployeesController : ControllerBase
{
	...
```

### التجربة في Postman

سنقوم بطلب الخدمة هذه المرة بدون تمرير الـ Authorization header:

{% include image.html url="assets/files/article_12/postman-unauthorized.png" border="1" %}

نلاحظ أن الخدمة رفضت هذه العملية وأعادت الـ HTTP Status Code التالي: Unauthorized 401.

نطلب الآن العنوان التالي ونقوم بتمرير إسم المستخدم وكلمة المرور لأحد المستخدمين المسجلين في AuthenticationService في الـ request body:

```html
https://localhost:5001/api/v1.0/auth/login
```

{% include image.html url="assets/files/article_12/postman-token.png" border="1" %}

والآن إنسخ التوكن ولو فتحت الـ console في المتصفح وكتبت التالي لأتضحت لك محتويات التوكن:


```javascript
JSON.stringify(JSON.parse(atob("eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1laWQiOiIxIiwidW5pcXVlX25hbWUiOiJhaG1hZCIsIm5iZiI6MTU5MjA4MDcxMSwiZXhwIjoxNTkyMTY3MTExLCJpYXQiOjE1OTIwODA3MTF9.luIluXkuSnd8YrKzsnyiboOopi37Eh5iixvCiPokmgM".split('.')[1])), null, 2);
```

{% include image.html url="assets/files/article_12/browser-token-parsed.png" border="1" %}

نعود الآن الى Postman لتنفيذ GetEmployee حيث نختار التبويب Authorization، ثم في القائمة المنسدلة TYPE نختار Bearer Token وفي الصندوق النصي Token في اليمين نضع التوكن الذي نسخناه بدون علامتي التنصيص ثم نضغط على Send ,سترى أنه بإمكاننا الآن الوصول الى الخدمة: 

{% include image.html url="assets/files/article_12/postman-request-using-token.png" border="1" %}

ولترى الـ header الذي سيتم إرساله، إختر التبويب Headers ثم إضغط على hidden:

{% include image.html url="assets/files/article_12/postman-hidden-headers.png" border="1" %}

### لمعرفة المستخدم الذي قام بالإستدعاء

بإمكانك الآن إستخدام User.Identity للحصول على معلومات المستخدم الذي قام بالإستدعاء. وعلى سبيل المثال:

```csharp
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployee(int id)
{
	_logger.LogInformation($"GetEmployee requested by: {User.Identity.Name}");
	...
```

{% include image.html url="assets/files/article_12/vscode-terminal.png" border="1" %}

### لعدم إشتراط التحقق

هناك حالات لا تريد فيها من المستخدم أن يرسل بيانات التحقق كإسم المستخدم وكلمة المرور أو التوكن في حالتنا هذه، كأن يسجل في الخدمة لأول مرة، ففي هذه الحالة يمكنك إضافة الـ attribute التالي: [AllowAnonymous] على العملية التي لا تتطلب التحقق من المستخدم.

## تحديث الـ JWT Token

مشكلة التوكن أن صلاحيته تنتهي expires بعد مرور فترة من الزمن تقوم أنت بتحديدها. وما يمكننا فعله في هذه الحالية هو توفير خاصية لتحديث التوكن بدلاً من تسجيل الدخول بإستخدام الإسم وكلمة المرور من جديد.

الفرق بين توكن jwt وتوكن التحديث هو أن الأول فترة صلاحية أقل بكثير من الثاني حيث من الممكن أن تستمر صلاحية الثاني الى عدة أشهر. بالإضافة الى أن توكن التحديث لا يحتوي على معلومات المستخدم وإنما يعتبر كمعرف للأول لا أكثر.

سوف نستخدم توكن jwt المنتهي صلاحيته وتوكن التحديث refresh token لتوليد توكن jwt جديد.

### تحديد مدة صلاحية الـ refresh token

يكون ذلك في ملف appsettings.json حيث سنقوم بإضافة RefreshTokenLifeTimeInMonths وهي تحدد صلاحية الـ refresh token بالأشهر:

```csharp
  "JwtSettings": {
    "SecretKey": "SecretSecurityKey",
    "TokenLifeTime": "00:15:00",
    "RefreshTokenLifeTimeInMonths": 6
  }
```

### إنشاء entity جديدة

ننشئ entity بإسم UserRefreshToken في مجلد الـ Entities مسؤولة عن حفظ الـ refresh token الخاصة بالمستخدم والـ jwt id التي ترتبط بها:

```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace aspnetcorewebapiproject.Entities
{
    public class UserRefreshToken
    {
        [Key]
        public int Id { get; set; }

        [Required]
        public string JwtTokenId { get; set; }

        [Required]
        public string RefreshToken { get; set; }

        [Required]
        public DateTime RefreshTokenExpirationDate { get; set; }

        [Required]
        public User User { get; set; }

        [Required]
        public bool IsValid { get; set; } = true;

        public DateTime LastModified { get; set; }

        public DateTime CreatedDate { get; set; }
    }
}
```

### إضافته الى الـ db context

نقوم بإضافة الـ entity الجديد UserRefreshToken على MainDbContext كما يلي:

```csharp
public class MainDbContext : DbContext
{
	public DbSet<Entities.Employee> Employees { get; set; }

	public DbSet<Entities.User> Users { get; set; }

	public DbSet<Entities.UserRefreshToken> UserRefreshTokens { get; set; }
	
	...
```

نضيف أوامر التعديل على قاعدة البيانات:

```csharp
dotnet ef migrations add "Adds_UserRefreshTokens"
```

نقوم الآن بتنفيذ الأمر:

```bash
dotnet ef database update
```

### إضافة DTO

في المجلد Models / v1 / Auth  نضيف الملف RefreshTokenDto.cs والذي يحتوي على معلومات المستخدم المطلوبة عند تسجيل الدخول:

```csharp
namespace aspnetcorewebapiproject.Models.Auth.v1
{
    public class RefreshTokenDto
    {
        public string JwtToken { get; set; }
        public string RefreshToken { get; set; }
    }
}
```

### التعديل على AuthController

هنالك عدة تعديلات تمت على AuthController وذلك لأننا أضفنا دالة خاصة بتحديث التوكن فأصبح لدينا كود مشترك بين دالة التحديث ودالة الإنشاء مما توجب علينا إستخلاصها في دوال مشتركة:

```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.AspNetCore.Cors;
using System.Security.Claims;
using System.IdentityModel.Tokens.Jwt;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using aspnetcorewebapiproject.Models.Auth.v1;
using aspnetcorewebapiproject.DbContexts;
using System.Linq;
using System.Security.Cryptography;
using Microsoft.EntityFrameworkCore;
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.Logging;

namespace aspnetcorewebapiproject.Controllers.v1
{
    /// <summary>
    /// Authenticates users and generates a jwt token
    /// </summary> 
    [AllowAnonymous]
    [ApiVersion("1.0")]
    [ApiVersion("1.1")]
    [Route("api/v{version:apiVersion}/[controller]")]
    [EnableCors("CorsPolicy")]
    [ApiController] 
    public class AuthController : ControllerBase
    {
        //
        // Fields
        //

        private readonly IConfiguration _config;
        private readonly MainDbContext _dbContext;
        private readonly string _jwtSecretKey;
        private readonly TimeSpan _jwtTokenLifeTime;        
        private readonly int _refreshTokenLifeTimeInMonths;
        private readonly ILogger<AuthController> _logger;

        //
        // Constructor
        //
        
        public AuthController(IConfiguration config, MainDbContext dbContext, ILogger<AuthController> logger)
        {
            _config = config;            
            _dbContext = dbContext;
            _logger = logger;
            
            _jwtSecretKey = _config.GetValue<string>("JwtSettings:SecretKey");
            _jwtTokenLifeTime = _config.GetValue<TimeSpan>("JwtSettings:TokenLifeTime");            
            _refreshTokenLifeTimeInMonths = _config.GetValue<int>("JwtSettings:RefreshTokenLifeTimeInMonths");           
        }

        //
        // Action Methods
        //        

        [HttpPost("login")]        
        public async Task<IActionResult> GenerateToken([FromBody] LoginDto data)
        {
            if( string.IsNullOrWhiteSpace(data.Username) || string.IsNullOrWhiteSpace(data.Password) )
                return BadRequest();

            var user = await _dbContext.Users.SingleOrDefaultAsync(
                                    u => u.Username == data.Username && u.Password == data.Password);
          
            if( user != null )
            {
                string jwtToken = _GenerateJwtToken(user, out string jwtId);
                string refreshToken = _GenerateRefreshToken();

                var userRefreshToken = new Entities.UserRefreshToken {
                    JwtTokenId = jwtId,
                    RefreshToken = refreshToken,
                    RefreshTokenExpirationDate = DateTime.UtcNow.AddMonths(_refreshTokenLifeTimeInMonths),
                    User = user
                };

                // Save to DB
                _dbContext.UserRefreshTokens.Add(userRefreshToken);
                await _dbContext.SaveChangesAsync();

                // Return to user
                return Ok( new { 
                    jwtToken = jwtToken,
                    refreshToken = refreshToken
                });                
            }

            return BadRequest();
        }

        [HttpPost("refresh")]
        public async Task<IActionResult> RefreshToken([FromBody] RefreshTokenDto data)
        {
            // 1. Get user info
            ClaimsPrincipal principal = _GetPrincipalFromToken(data.JwtToken);
            string userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;

            // 2. Check if jwt token actually expired
            var expirationDateUnixTimestamp = 
                long.Parse( principal.Claims.Single(c => c.Type == JwtRegisteredClaimNames.Exp).Value );

            var expirationDateUtc = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc)
                        .AddSeconds(expirationDateUnixTimestamp);

            if( expirationDateUtc > DateTime.UtcNow )
                return BadRequest("Token has not expired yet");

            // 3. Get jwt token Id
            var jti = principal.Claims.Single(c => c.Type == JwtRegisteredClaimNames.Jti).Value;

            // 4. Check if refresh token actually exists in db
            var userRefreshTokenDb = await _dbContext.UserRefreshTokens.Include("User").SingleOrDefaultAsync(
                        ut => ut.JwtTokenId == jti && ut.RefreshToken == data.RefreshToken
                                && ut.User.Id == Convert.ToInt32(userId));

            if( userRefreshTokenDb == null )
                return BadRequest("Refresh token does not exist");

            // 5. Check if refresh token is valid
            if( !userRefreshTokenDb.IsValid )
                return BadRequest("Refresh token is invalid");                

            // 6. Check if refresh token has expired
            if( DateTime.UtcNow > userRefreshTokenDb.RefreshTokenExpirationDate )
                return BadRequest("Refresh token expired. Please login again");

            string jwtToken = _GenerateJwtToken(userRefreshTokenDb.User, out string jwtId);
            string refreshToken = _GenerateRefreshToken();

            var userRefreshToken = new Entities.UserRefreshToken {
                JwtTokenId = jti,
                RefreshToken = refreshToken,
                RefreshTokenExpirationDate = DateTime.UtcNow.AddMonths(_refreshTokenLifeTimeInMonths),
                User = userRefreshTokenDb.User
            };

            // Save to DB
            _dbContext.UserRefreshTokens.Add(userRefreshToken);
            await _dbContext.SaveChangesAsync();

            // Return to user
            return Ok( new { 
                jwtToken = jwtToken,
                refreshToken = refreshToken
            });              
        }

        //
        // Helper Methods
        //

        private string _GenerateJwtToken(Entities.User user, out string jwtId)
        {    
            string userId = user.Id.ToString();
            string username = user.Username;
            string userEmail = user.Email;
            jwtId = Guid.NewGuid().ToString();
            string notBefore = new DateTimeOffset(DateTime.UtcNow).ToUnixTimeSeconds().ToString();
            string expiration = new DateTimeOffset(DateTime.UtcNow).Add(_jwtTokenLifeTime).ToUnixTimeSeconds().ToString();

            // User found. Create ticket for user
            var claims = new List<Claim> {
                new Claim("id", userId),
                new Claim(ClaimTypes.NameIdentifier, userId),
                new Claim(ClaimTypes.Name, username),
                new Claim(JwtRegisteredClaimNames.Email, userEmail),
                new Claim(JwtRegisteredClaimNames.Jti, jwtId),
                new Claim(JwtRegisteredClaimNames.Sub, username),

                // not before, this token is not valid before a certain date and time
                // in our case we want it to be valid right away
                new Claim(JwtRegisteredClaimNames.Nbf, notBefore),

                // when will it expire
                new Claim(JwtRegisteredClaimNames.Exp, expiration)
            };

            var tokenDescriptor = new SecurityTokenDescriptor
            {
                Subject = new ClaimsIdentity(claims),
                Expires = DateTime.UtcNow.Add(_jwtTokenLifeTime),
                SigningCredentials = new SigningCredentials(
                            new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSecretKey)), 
                            SecurityAlgorithms.HmacSha256) 
            };

            var tokenHandler = new JwtSecurityTokenHandler();
            SecurityToken token = tokenHandler.CreateToken(tokenDescriptor);

            return tokenHandler.WriteToken(token);
        }
        
        private string _GenerateRefreshToken()
        {            
            using(var randNumGen = RandomNumberGenerator.Create())
            {
                var randNum = new byte[32];

                randNumGen.GetBytes(randNum);

                return Convert.ToBase64String(randNum);
            }
        }

        // Gets user from access jwt token
        private ClaimsPrincipal _GetPrincipalFromToken(string jwtToken)
        {
            // The same as the one in Startup.ConfigureServices().AddJwtBearer() except ValidateLifetime = false
            var tokenValidationParameters = new TokenValidationParameters()
            {
                IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSecretKey)),
                ValidateIssuer = false,
                ValidateAudience = false,
                ValidateLifetime = false
            };

            var tokenHandler = new JwtSecurityTokenHandler();

            try
            {
                var principal = tokenHandler.ValidateToken(jwtToken, tokenValidationParameters, out SecurityToken validatedSecurityToken);

                // Is JWT with valid security algorithm
                if( (validatedSecurityToken is JwtSecurityToken jwtSecurityToken) &&
                    (jwtSecurityToken.Header.Alg.Equals(SecurityAlgorithms.HmacSha256, StringComparison.InvariantCultureIgnoreCase)))
                    return principal;

                throw new SecurityTokenException("Invalid token");
            }
            catch
            {
                throw new SecurityTokenException("Invalid token");
            }
        }
    }
}
```

فيما يلي شرح لأهم النقاط المذكورة في الكود:

* **GenerateToken**: هذه العملية مسؤولة عن إنشاء الـ jwt token والخاصية الجديدة refresh token وإعادتها للمستخدم. يجب التنويه أنه كلما أنشأنا jwt token ننشئ في مقابله refresh token ونحفظها في قاعدة البيانات. ولكن لا نحفظ الـ jwt token ذاته وإنما معرفه ID

* **RefreshToken**: وهي مسؤولة عن إنشاء الـ refresh token ولكنها تقوم بعدة عمليات مهمة للتأكد من صحة الـ jwt token والـ refresh token الممر اليها. 

* **GenerateJwtToken_**: وهي المسؤولة فعلاً عن إنشاء الـ jwt token ويتم إستخدامها في كلاً من GenerateToken و RefreshToken.

* **GenerateRefreshToken_**: وهي المسؤولة فعلاً عن إنشاء الـ refresh token ويتم إستخدامها في كلاً من GenerateToken و RefreshToken.

* **GetPrincipalFromToken_**: دورها إستخلاص معلومات المستخدم الحامل للـ jwt token. من المهم جعل `ValidateLifetime = false` وإلا لن يستطيع المستخدم تحديث الـ jwt token الذي لديه بسبب إنتهاء صلاحيته.

### التجربة في Postman

1) إطلب auth / login :

{% include image.html url="assets/files/article_12/postman-auth-login.png" border="1" %}

2) إنسخ الـ jwt token  وإستخدمه في إستدعاء GetEmployee:

{% include image.html url="assets/files/article_12/postman-employees-getemployee.png" border="1" %}

3) نفذ الإستدعاء السابق بعد مرور دقيقة واحدة:

{% include image.html url="assets/files/article_12/postman-employees-getemployee-unauthorized.png" border="1" %}
	
4) إنشئ request جديد في Postman يستدعي العملية auth / refresh ويمرر اليها الـ jwt & refresh tokens التي حصلنا عليها في الخطوة رقم 1:

{% include image.html url="assets/files/article_12/postman-auth-refresh.png" border="1" %}

5) إستخدم الـ jwt token الجديد وسترى بأن الإستدعاء سيتم بشكل صحيح


###  حفظ التعديلات

نضيف الآن هذه التعديلات الى git:

```bash
git add .
git commit -m "adds bearer token authentication support"
```

### إعتماد خاصية الـ JWT tokens

سنعتمد الآن ما قمنا به في الـ branch الذي أنشأناه وسنقوم بإضافته على الـ master.

نعود الآن الى الـ master branch:

```bash
git checkout master
```

ثم ندمج التعديلات التي تمت في الـ jwt branch:

```bash
git merge jwt
```
