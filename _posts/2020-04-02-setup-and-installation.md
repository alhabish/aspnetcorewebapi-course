---
layout: post
title: "1. تهيئة البيئة البرمجية"
date: 2020-04-02
lang: ar-SA
index: 1
comments: true
---

نعتمد في هذه الدورة على عدة برامج وتطبيقات يرجى تحميلها من الروابط التالية:

* [Visual Studio Code](https://code.visualstudio.com)
* [Git](https://git-scm.com)
* [Postman](https://www.postman.com)
* [.NET Core SDK](https://dotnet.microsoft.com/download)

بالنسبة لـ .NET Core SDK يرجى إختيار التالي:

{% include image.html url="assets/files/article_01/download-dotnetcore-sdk.jpg" border="1" %}

نقوم بعد ذلك بتنصيب Setup جميع هذه البرامج.


### []()التأكد من صحة تنصيب البرامج

تأكد من أنه تم تنصيب Git بشكل صحيح عن طريق فتح الـ Command Prompt وكتابة:

```bash
git --version
```

فإن ظهرت رسالة تبين نسخة git المستخدمة فإن عملية التنصيب تمت بشكل صحيح:

{% include image.html url="assets/files/article_01/git-version.jpg" border="1" %}

ونقوم بنفس الشيئ مع الـ dotNET Core Framework:


```bash
dotnet --version
```

ومن المفترض أن يظهر لنا:

{% include image.html url="assets/files/article_01/dotnetcore-version.jpg" border="1" %}


### []()تهيئة Git

لنقم بتعديل بعض الإعدادات المتعلقة بـ git:


```bash
git config --global user.name "Khalid Alghamidi"
git config --global user.email "kalhabish@gmail.com"
git config --global apply.whitespace nowarn
```

السطر الأول والثاني تقوم بتغير الإسم والبريد الإلكتروني حيث يقوم git بربط هذه المعلومات بكل commit تقوم به.
السطر الثاث لكي لا يظهر لك git بأن هنالك مشكلة في نهاية الأسطر بين الأنظمة المختلفة.

بإمكانك رؤية جميع الإعدادات التي يستخدمها git بكتابة الأمر التالي:

```bash
git config --list
```

{% include image.html url="assets/files/article_01/git-config-list.png" border="1" %}


### []()تهيئة VS Code

قم بتحميل الإضافات extensions التالية:

{% include image.html url="assets/files/article_01/vscode-ext-csharp.png" border="1" %}

{% include image.html url="assets/files/article_01/vscode-ext-intellicode.png" border="1" %}

{% include image.html url="assets/files/article_01/vscode-ext-gitlens.png" border="1" %}

{% include image.html url="assets/files/article_01/vscode-ext-githistory.png" border="1" %}

{% include image.html url="assets/files/article_01/vscode-ext-prettier.png" border="1" %}

{% include image.html url="assets/files/article_01/vscode-ext-vscodeicons.png" border="1" %}

{% include image.html url="assets/files/article_01/vscode-ext-bracketpair.png" border="1" %}

وذلك عن طريق الخطوات التالية:

* إختيار Extensions من القائمة اليسرى
* كتابة إسم الإضافة extension في صندوق البحث
* إختيار الإضافة
* الضغط على زر Install

{% include image.html url="assets/files/article_01/vscode-how-to-install-an-extension.png" border="1" %}

## إنشاء المشروع

يكون لدي في العادة مجلد إسمه repos أحفظ فيه جميع المشاريع البرمجية. فإن لم يكن موجود قم بالتالي:

```bash
cd /
mkdir repos
cd repos
```

نقوم بعد ذلك بإنشاء مجلد جديد بإسم `aspnetcorewebapiproject` يحتوي على مشروعنا:


```bash
mkdir aspnetcorewebapiproject
cd aspnetcorewebapiproject
```

وبذلك سنكون داخل المجلد الجديد الذي أنشأناه:

{% include image.html url="assets/files/article_01/pwd.png" border="1" %}

نقوم الآن بإنشاء git repository جديد في هذا المجلد:

```bash
git init
```

نلاحظ أن git أنشأ مجلد جديد بإسم `.git` لإدارة المحتوى:

{% include image.html url="assets/files/article_01/git-init.png" border="1" %}

سنستخدم الآن الأداة `dotnet` لإنشاء المشروع. ولرؤية أنواع المشاريع التي بإمكان هذه الأداة إنشائها إستخدم الأمر التالي:

```bash
dotnet new -h
```

{% include image.html url="assets/files/article_01/dotnet-new-h.png" border="1" %}

نحن الآن بحاجة الى ملف `.gitignore` والتى توضح لـ git ماهي الملفات التي يجب عليه عدم متابعتها. ولنقوم بذلك نستخدم الأمر التالي:

```bash
dotnet new gitignore
```

المشروع الذي نرغب في إنشاؤه من نوع ASP.NET Core Web API ولذلك ننفذ الأمر التالي:

```bash
dotnet new webapi
```

{% include image.html url="assets/files/article_01/dotnet-new-webapi.png" border="1" %}

بإمكاننا أن نرى الملفات الجديدة التي تم إنشاؤها:

{% include image.html url="assets/files/article_01/ls-after-creating-project.png" border="1" %}

ولو أستخدمت الأمر التالي لرأيت بأن git يعلم عن الملفات الجديدة ولكنه لا يراقبها:

```bash
git status
```

{% include image.html url="assets/files/article_01/git-status-before-add.png" border="1" %}

ثم ننفذ الأمر التالي لإضافة الملفات الجديدة الى منطقة الـ staging الخاصة بـ git:

```bash
git add .
```

{% include image.html url="assets/files/article_01/git-status-after-add.png" border="1" %}

والآن نضيفها للـ local repository بإستخدام الأمر التالي:

```bash
git commit -m "initial commit"
```

{% include image.html url="assets/files/article_01/git-commit.png" border="1" %}

نفتح المشروع في vs code بإستخدام الأمر التالي:

```bash
code .
```

إذا ظهرت لك النافذة التالية، إختر Yes حيث سيقوم vs code بإضافة بعض الأدوات الضرورية لبناء وتشغيل المشروع:

{% include image.html url="assets/files/article_01/vscode-required-assets.png" border="1" %}

وبدلاً من إستخدام الـ Command Prompt فإنه بإمكاننا إستخدام الـ Terminal الموجود في vs code بإستخدام الإختصار:


```bash
Ctrl+Shift+`
```

{% include image.html url="assets/files/article_01/vscode-terminal.png" border="1" %}

لنقم الآببناء المشروع ثم تشغيله:

```bash
dotnet build
dotnet run
```

نتيجة تنفيذ الأمر `dotnet run` هي:

{% include image.html url="assets/files/article_01/dotnet-run.png" border="1" %}

نلاحظ انه بإمكاننا الوصول الى الخدمة عبر البروتوكول http أو https. سنستخدم عنوان الـ https وسنضيف اليه WeatherForecast لنصل الى هذا الـ Controller:

```bash
https://localhost:5001/WeatherForecast
```

{% include image.html url="assets/files/article_01/weatherforcast.png" border="1" %}

يمكن إيقاف المشروع من Terminal في VS Code بالأوامر التالية:

`Ctrl+C`

بذلك أنشأنا مشروع جديد وتأكدنا من أنه يعمل بشكل صحيح.
