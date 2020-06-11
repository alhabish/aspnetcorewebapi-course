---
layout: post
title: "11. Securing the API - Basic Authentication"
date: 2020-06-11
lang: ar-SA
index: 11
comments: true
---

يعتبر الـ Basic Authentication من أبسط أنواع المصادقة authentication التي يمكن تطبيقها في مشروعك ولكنها ليست آمنه بالقدر الكافي حيث أنه يتم تمرير إسم المستخدم وكلمة المرور في كل طلب request بطريقة مكشوفة ولكن مشفرة الى base64 والتي يمكن فك تشفيرها ببساطة.

وهي تعتبر حل سريع ومقبول في الشبكات الداخلية بشرط أن تكون قناة الإتصال HTTPS. والفكرة تتمحور حول إرسال header بإسم Authorization متبوع بمسافة ثم كلمة Basic ثم مسافة وبعد ذلك إسم المستخدم وكلمة المرور وبينهما علامة : وجميعها محول الى نص من نوع base64. أي بالشكل التالي:

```
Authorization: Basic base64("username:password")
```

هنالك طريقتان لتنفيذ الـ basic authentication، إما عن طريق الـ middleware أو عن طريق الـ handler وهو الإسلوب الذي سنتبعة حيث إنه يعتبر أكثر مرونه.

## العمل في branch مستقل

ننشئ branch جديد ونتحول اليه:

```bash
git branch basic-auth
git checkout basic-auth
```

## إدارة المستفيدين من الخدمة

ننشئ مجلد جديد بإسم Users وبداخله ملف بإسم User.cs يمثل المستفيد من الخدمة التي نقدمها:

```csharp
namespace aspnetcorewebapiproject.Users
{
    public class User
    {
        public int Id { get; set; }
        public string Username { get; set; }
        public string Password { get; set; }
    }
}
``` 

أيضاً ننشئ interface يمثل عملية التحقق من المستخدم والتي ممكن أن تكون من قاعدة بيانات أو Active Directory أو خدمة أخرى على سبيل المثال. سيكون إسم الملف IAuthenticationService.cs وبه عمليه وحدة:

```csharp
using System.Threading.Tasks;

namespace aspnetcorewebapiproject.Users
{
    public interface IAuthenticationService
    {
        Task<User> Authenticate(string username, string password);
    }
}
```

بإمكاننا هنا إضافة جميع العمليات المتعلقة بإدارة المستخدمين من إضافة وتعديل بيانات ورقم سري ... الخ.

ننشئ الآن ملف جديد يعتبر تتطبيق implementation للـ interface السابق بإسم AuthenticationService.cs:

```csharp
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace aspnetcorewebapiproject.Users
{
    public class AuthenticationService : IAuthenticationService
    {
        private List<User> _users = new List<User>
        {
            new User { Id = 1, Username = "ahmad", Password = "abc@123" },
            new User { Id = 2, Username = "ali", Password = "def@456" },
            new User { Id = 3, Username = "yousef", Password = "ghi@789" }
        };

        public async Task<User> Authenticate(string username, string password)
        {
            var user = await Task.Run(() => _users.SingleOrDefault(x => x.Username == username && x.Password == password));

            // user not found
            if (user == null)
                return null;

            // return user details without password
            user.Password = string.Empty;
            return user;
        }
    }
}
```
أخترنا هنا أن نحفظ بيانات المستخدمين في قائمة list في الذاكرة لتسهيل العملية ولكن عملياً يجب أن تكون بيانات المستخدمين محفوظة في نظام آخر.

والآن جاء دور إخبار بيئة العمل بأنه متى ما تم طلب IAuthenticationService أعد AuthenticationService وذلك في الدالة ()ConfigureServices في Startup.cs:

```csharp
public void ConfigureServices(IServiceCollection services)
{
	...           

	services.AddScoped<Users.IAuthenticationService, Users.AuthenticationService>();
}
```

## إضافة الـ Handler

ننشئ مجلد جديد بإسم Handlers وبداخله ملف بإسم BasicAuthenticationHandler.cs وهي التي ستحدد طريقة المصادقة:

```csharp

using System;
using System.Net.Http.Headers;
using System.Security.Claims;
using System.Text;
using System.Text.Encodings.Web;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using aspnetcorewebapiproject.Users;

namespace aspnetcorewebapiproject.Handlers
{
    public class BasicAuthenticationHandler : AuthenticationHandler<AuthenticationSchemeOptions>
    {
        private readonly Users.IAuthenticationService _authService;

        public BasicAuthenticationHandler(
            IOptionsMonitor<AuthenticationSchemeOptions> options,
            ILoggerFactory logger,
            UrlEncoder encoder,
            ISystemClock clock,
            Users.IAuthenticationService authService)
            : base(options, logger, encoder, clock)
        {
            // 1. Pass auth service implementation
            _authService = authService;
        }

        protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
        {
            // 2. Make sure Authorization key exists
            if (!Request.Headers.ContainsKey("Authorization"))
                return AuthenticateResult.Fail("Missing Authorization Header");

            User user = null;
            try
            {
                // 3. Key exists. Extract username and password and authenticate the pair
                var authHeader = AuthenticationHeaderValue.Parse(Request.Headers["Authorization"]);

                if(!"Basic".Equals(authHeader.Scheme, StringComparison.OrdinalIgnoreCase))
                {
                    // 4. Not Basic authentication header
                    return AuthenticateResult.Fail("Not Basic authentication header");
                }

                var credentialBytes = Convert.FromBase64String(authHeader.Parameter);
                var credentials = Encoding.UTF8.GetString(credentialBytes).Split(new[] { ':' }, 2);
                var username = credentials[0];
                var password = credentials[1];
                user = await _authService.Authenticate(username, password);
            }
            catch
            {
                // 5. Key exists but there was an error with the extraction or authentication
                return AuthenticateResult.Fail("Invalid Authorization Header");
            }

            // 6. Could not find user with username/password
            if (user == null)
                return AuthenticateResult.Fail("Invalid Username or Password");

            // 7. User found. Create ticket for user
            var claims = new[] {
                new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
                new Claim(ClaimTypes.Name, user.Username),
            };
            var identity = new ClaimsIdentity(claims, Scheme.Name);
            var principal = new ClaimsPrincipal(identity);
            var ticket = new AuthenticationTicket(principal, Scheme.Name);

            return AuthenticateResult.Success(ticket);
        }
    }
}
```

سيتم إستدعاء HandleAuthenticateAsync قبل كل عملية إستدعاء request للخدمة.

وفيما يلي شرح لأهم النقاط التي وردت في الكود السابق:

1. نمرر Users.IAuthenticationService في الـ constructor والتي سيتم إستبدالها بالتطبيق AuthenticationService

2. في حالة لم يتم تمرير Authorization في الـ HTTP Header فإن عملية التحقق ستفشل

3. نستخلص قيمة الـ Authorization header

4. نتأكد أن نوع عملية التحقق هي Basic

5. إذا حدث خطأ في إستخلاص الإسم وكلمة المرور فإن العملية ستفشل

6. إذا لم نجد المستخدم الذي يحمل هذا الإسم وكلمة المرور فإن التحقق سيفشل

7. في حالة أننا وجدنا المستخدم سنقوم بحفظ معلوماته في ما يسمى بالـ claims وهي عبارة عن key-value pair تمثل بيانات المستخدم


## التعديل على Startup.cs

لتفعيل ما قمنا به نقوم بالتالي:

### التعديل على الدالة ()Configure

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

### التعديل على ()ConfigureServices

نقوم بإضافة عملية التحقق ونعتمد التطبيق الموجود في BasicAuthenticationHandler:

```csharp
public void ConfigureServices(IServiceCollection services)
{
	...           

	// configure DI for application services
	services.AddScoped<Users.IAuthenticationService, Users.AuthenticationService>();

	// configure basic authentication 
	services.AddAuthentication("BasicAuthentication")
		.AddScheme<AuthenticationSchemeOptions, BasicAuthenticationHandler>("BasicAuthentication", null);
}
```

## التعديل على EmployeesController

لجعل EmployeesController تتطلب التحقق من المستخدم قبل إمكانية الإستفادة منها فإنه يتوجب علينا إضافة الـ [Authorize] attribute:

```csharp
[Authorize]
...
public class EmployeesController : ControllerBase
{
	...
```

## التجربة في Postman

سنقوم بطلب الخدمة هذه المرة بدون تمرير الـ Authorization header:

{% include image.html url="assets/files/article_11/postman-unauthorized.png" border="1" %}

نلاحظ أن الخدمة رفضت هذه العملية وأعادت الـ HTTP Status Code التالي: Unauthorized 401.

والآن إختر التبويب Authorization، ثم في القائمة المنسدلة TYPE إختر Basic Auth وفي Username/Password في اليمين ضع إسم المستخدم وكلمة المرور لأحد المستخدمين المسجلين في AuthenticationService:

{% include image.html url="assets/files/article_11/postman-auth-type.png" border="1" %}

ولترى الـ header الذي سيتم إرساله، إختر التبويب Headers ثم إضغط على hidden:

{% include image.html url="assets/files/article_11/postman-hidden-headers.png" border="1" %}

ستلاحظ أن القيمة سيتم إرسالها بالطريقة التي ذكرناها سابقاً:

{% include image.html url="assets/files/article_11/postman-authorization-header.png" border="1" %}

وعند التنفيذ سترى أنه بإمكاننا الآن الوصول الى الخدمة:

{% include image.html url="assets/files/article_11/postman-ok.png" border="1" %}

## لمعرفة المستخدم الذي قام بالإستدعاء

بإمكانك الآن إستخدام User.Identity للحصول على معلومات المستخدم الذي قام بالإستدعاء. وعلى سبيل المثال:

```csharp
public async Task<ActionResult<EmployeesResponse<EmployeeDetailsDto>>> GetEmployee(int id)
{
	_logger.LogInformation($"GetEmployee requested by: {User.Identity.Name}");
	...
```

{% include image.html url="assets/files/article_11/vscode-terminal.png" border="1" %}

## لعدم إشتراط التحقق

هناك حالات لا تريد فيها المستخدم أن يرسل إسم مستخدم وكلمة المرور، كأن يسجل في الخدمة لأول مرة، ففي هذه الحالة يمكنك إضافة الـ attribute التالي: [AllowAnonymous] على العملية التي لا تتطلب التحقق من المستخدم.

###  حفظ التعديلات

نضيف الآن هذه التعديلات الى git:

```bash
git add .
git commit -m "adds basic authentication support"

سوف نبقي هذه الخاصية في هذا الـ branch وسوف نعود الى master من دونها لنتمكن من تكملة باقي الدروس:

```bash
git checkout master
```

### ~~~ مصادر ~~~
[ASP.NET Core 3.1 - Basic Authentication Tutorial with Example API
](https://jasonwatmore.com/post/2019/10/21/aspnet-core-3-basic-authentication-tutorial-with-example-api)
[ASP.NET Core Web API + Entity Framework Core : Basic Authentication Explained - EP07](https://www.youtube.com/watch?v=6X6iONXhz2w)
