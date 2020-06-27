---
layout: post
title: "13. Deploy to Azure"
date: 2020-06-27
lang: ar-SA
index: 13
comments: true
--- 

في هذا الدرس سوف نتعلم كيف نرفع deploy مشروعنا لأحد الخدمات السحابية، Azure بالتحديد، وذلك بإتباع الخطوات التالية: 

## إنشاء حساب في Azure

إتجه الى الرابط التالي:

[Azure](https://azure.microsoft.com/en-us/free/)

ثم أختر Start free:

{% include image.html url="assets/files/article_13/azure-start-free.png" border="1" %}

قم بتعبئة البيانات اللازمة:

{% include image.html url="assets/files/article_13/azure-sign-up.png" border="1" %}

إضغط على Go to the portal:

{% include image.html url="assets/files/article_13/azure-go-to-the-portal.png" border="1" %}


## إنشاء سيرفر قاعدة البيانات

إبحث عن sql servers:

{% include image.html url="assets/files/article_13/azure-sql-servers.png" border="1" %}

إضغط على Add لإضافة سيرفر قاعدة بيانات جديد:

{% include image.html url="assets/files/article_13/azure-add-sql-server.png" border="1" %}

أدخل بيانات السيرفر: 

{% include image.html url="assets/files/article_13/azure-sql-server-project-details.png" border="1" %}

إضغط على Next : Networking وإختر Allow Azure services and resources to access this server

{% include image.html url="assets/files/article_13/azure-sql-server-networking.png" border="1" %}

إضغط على Review + create:

{% include image.html url="assets/files/article_13/azure-sql-server-review-create.png" border="1" %}

أخيراً، إضغط على Create لإنشاء السيرفر.

## إعداد الـ Firewall

لن تتمكن من الوصول لهذا السيرفر لأن الـ firewall سيمنع هذا الإتصال، ولذلك نطلب إستثاء الـ IP الخاص بك من هذا الحظر عن طريق الضغط على Show firewall settings:

{% include image.html url="assets/files/article_13/azure-show-firewall-settings.png" border="1" %}

ثم Add Client IP ثم Save:

{% include image.html url="assets/files/article_13/azure-firewall-add-client-ip.png" border="1" %}

## رفع قاعدة البيانات MainDb الى Azure

إتصل بقاعدة البيانات على جهازك عن طريق MS SQL Server Management Studio:

{% include image.html url="assets/files/article_13/ssms-connect-to-db.png" border="1" %}

إضغط باليمين على إسم قاعدة البيانات ثم Tasks وبعد ذلك على Deploy Database to Microsoft Azure SQL Database:

{% include image.html url="assets/files/article_13/ssms-deploy-db.png" border="1" %}

إختر Next:

{% include image.html url="assets/files/article_13/ssms-db-deploy-1.png" border="1" %}

إذهب الى صفحة الـ Overview لسيرفر قاعدة البيانات وأنسخ إسم السيرفر:

{% include image.html url="assets/files/article_13/azure-db-server-name.png" border="1" %}

ألصقها في نافذة Connect to server مع تغيير كلمة المرور:

{% include image.html url="assets/files/article_13/ssms-connect-to-azure-db.png" border="1" %}

إضغط على Connect:

{% include image.html url="assets/files/article_13/ssms-deployment-settings.png" border="1" %}

ثم Finish:

{% include image.html url="assets/files/article_13/ssms-db-deploy-result.png" border="1" %}

وبعد ذلك على Close.

عُد مرة أخرى الى سيرفر قاعدة البيانات في Azure وسترى بأنه تم إضافة قاعدة البيانات الجديدة:

{% include image.html url="assets/files/article_13/azure-db-added.png" border="1" %}

بإمكانك الآن الإتصال بقاعدة البيانات في Azure للتأكد من ذلك:

{% include image.html url="assets/files/article_13/ssms-connected-to-azure-db.png" border="1" %}

# تهيئة الخدمة على Azure

إبحث عن App Services:

{% include image.html url="assets/files/article_13/azure-resource-app-services.png" border="1" %}

إضغط على على Add أو Create app service:

{% include image.html url="assets/files/article_13/azure-add-app-service.png" border="1" %}

قم بتعبئة البيانات المطلوبة:

{% include image.html url="assets/files/article_13/azure-create-web-app-basics.png" border="1" %}

في Sku and size إضغط على Change size.

إبحث عن الخيار المجاني ثم إضغط على Apply:

{% include image.html url="assets/files/article_13/azure-create-web-app-spec-picker.png" border="1" %}

والآن إضغط على Review + create:

{% include image.html url="assets/files/article_13/azure-create-web-app-summary.png" border="1" %}

إضغط على Create.

إذهب الى صفحة تفاصيل الـ app service وإنسخ العنوان وأفتحه في صفحة جديدة في المتصفح:

{% include image.html url="assets/files/article_13/azure-newly-created-app-service.png" border="1" %}

سترى هذه الصفحة:

{% include image.html url="assets/files/article_13/browser-service-initial.png" border="1" %}

إذهب الى قاعدة البيانات MainDb لنسخ الـ connection string:

{% include image.html url="assets/files/article_13/azure-show-database-connection-strings.png" border="1" %}

والآن قم بنسخه:

{% include image.html url="assets/files/article_13/azure-connection-string.png" border="1" %}

عد مرة أخرى الى app service وأضغط على Configuration ثم New connection string:

{% include image.html url="assets/files/article_13/azure-add-connection-string.png" border="1" %}

أكتب MainDbContext والصق الـ connection string الذي نسخته سابقاً مع إستبدال {your_password} بكلمة المرور الخاصة بك:

إحفظ التعديلات الجديدة.

## رفع الخدمة الى Azure

في VS Code، ننفذ أولاً الأمر publish كالآتي:

{% include image.html url="assets/files/article_13/vscode-publish.png" border="1" %}

ثم نقوم بإضافة الـ extension الذي يحمل إسم Azure App Service:

{% include image.html url="assets/files/article_13/vscode-azure-app-service-ext.png" border="1" %}

نضغط بعد ذلك على أيقونة Azure ثم نسجل دخول الى Azure. بعد ذلك نضغط باليمين على إسم المشروع ومن ثم Deploy to Web App:

{% include image.html url="assets/files/article_13/vscode-log-in-to-azure.png" border="1" %}

سنختار المجلد الذي سنقوم برفعه وسوف يكون bin / Release / netcoreapp3.1 :

{% include image.html url="assets/files/article_13/vscode-select-folder-to-deploy.png" border="1" %}

إختر المشروع الذي أنشأناه في Azure:

{% include image.html url="assets/files/article_13/vscode-select-web-app.png" border="1" %}

عندما ترى النافذه التالية إختر Deploy:

{% include image.html url="assets/files/article_13/vscode-deploy-confirmation.png" border="1" %}

بإمكانك متابعة إجراءات الرفع deployment من نافذة الـ Output:

{% include image.html url="assets/files/article_13/vscode-output-window-deployment-status.png" border="1" %}

وهنا نرى بأن العملية قد تمت:

{% include image.html url="assets/files/article_13/vscode-deployment-complete.png" border="1" %}

والآن بإمكاننا الذهاب الى عنوان الخدمة في المتصفح:

{% include image.html url="assets/files/article_13/browser-project-url.png" border="1" %}

## التجربة في Postman

سجل دخول أولاً، ولا تنسى تغيير العنوان ليشير الى الخدمة في Azure:

{% include image.html url="assets/files/article_13/postman-login.png" border="1" %}

ثم نفذ عملية GetEmployee مع تمرير الـ jwt token:

{% include image.html url="assets/files/article_13/postman-get-employee.png" border="1" %}

