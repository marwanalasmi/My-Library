---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 33 Maintenance

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Maintenance Fundamentals|Maintenance Fundamentals]]
- [[#What to Monitor|What to Monitor]]
- [[#Maintenance Routine|Maintenance Routine]]
- [[#Practical Maintenance Review|Practical Maintenance Review]]
- [[#Incident Response|Incident Response]]
- [[#Updates & Change Management|Updates & Change Management]]
- [[#Operational Documentation|Operational Documentation]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

بعد نشر Odoo، تبدأ مسؤولية المحافظة على استقرار النظام وصحة بياناته واستمرار العمليات التي تعتمد عليه.

قد تكون الواجهة متاحة للمستخدمين، بينما توجد مشكلة لا تظهر مباشرة:

- Scheduled Action لم تعد تنفذ العمل المطلوب.
- Integration تراكمت فيها العمليات المعلقة.
- Backup تعمل، لكن النسخ لا تحتوي الملفات اللازمة للاستعادة.
- مساحة Filestore تنمو أسرع من المتوقع.
- صلاحيات مستخدم سابق ما زالت فعالة.
- تحديث جديد تسبب في مشكلة داخل Custom Module.

لذلك لا تقتصر **Maintenance** على إصلاح الأخطاء بعد حدوثها.

هي عملية مستمرة تجمع بين:

```text
المراقبة
    ↓
اكتشاف التغير أو المشكلة
    ↓
فهم السبب والأثر
    ↓
اتخاذ إجراء مناسب
    ↓
التحقق من النتيجة وتوثيقها
```

في هذا الشابتر سنربط الصيانة بما بنيناه في `equipment_training`، ونضع طريقة عملية لمراجعة النظام والتعامل مع الأعطال.

---

## Maintenance Fundamentals

### ما المقصود بـMaintenance؟

تشمل الصيانة متابعة النظام بعد Deployment، والتأكد من استمرار قدرته على تنفيذ متطلبات العمل.

من أمثلتها:

| المجال | ما نريد التأكد منه |
|---|---|
| Availability | يستطيع المستخدمون الوصول إلى الوظائف المطلوبة |
| Business Operations | العمليات تنتهي بالنتائج المتوقعة |
| Data Integrity | البيانات والعلاقات ما زالت صحيحة |
| Background Jobs | الأعمال المجدولة تعالج العمل المستحق |
| Recovery | يمكن استعادة النظام ضمن أهداف الاستعادة |
| Security | الوصول والأسرار ما زالا مناسبين |
| Updates | التغييرات تصل إلى Production بعد التحقق منها |

هذه المجالات مترابطة.

مثلًا: امتلاء Disk قد يؤثر على PostgreSQL أو Logs أو Backups، ثم يظهر للمستخدم على شكل فشل في حفظ Record.

---

### Monitoring وAlerting وIncident Response

هذه مصطلحات مختلفة:

| المصطلح | وظيفته |
|---|---|
| Monitoring | جمع مؤشرات عن حالة النظام ومتابعتها |
| Alerting | تنبيه المسؤول عند تحقق شرط يحتاج متابعة |
| Incident Response | التعامل المنظم مع عطل يؤثر على الخدمة أو البيانات |

وجود Dashboard لا يكفي إذا لم يراجعها أحد.

ووجود Alert لا يكفي إذا لم يكن واضحًا:

- من المسؤول عنها؟
- ما أثر المشكلة؟
- ما أول إجراء للفحص؟
- متى تحتاج إلى تصعيد؟

> [!tip] صمم التنبيه حول إجراء
> التنبيه المفيد يساعد المسؤول على معرفة ما يحتاج إلى فحصه، وكيف يتحقق من عودة الخدمة إلى حالتها المطلوبة.

---

### راقب الخدمة ونتيجة العمل

تحتاج إلى مستويين من المراقبة:

```text
Technical Health
→ هل المكونات تعمل؟

Business Health
→ هل العمل المطلوب يحدث؟
```

في مشروعنا:

- فتح Dashboard مؤشر على إمكانية الوصول إلى الواجهة.
- ظهور عدد الأجهزة المتاحة مؤشر على نجاح القراءة في سياق المستخدم.
- معالجة طلبات المراجعة المستحقة مؤشر على نتيجة Cron.

كل مؤشر يثبت جزءًا مختلفًا من عمل النظام.

---

## What to Monitor

### 1. Availability والوظائف الأساسية

تابع إمكانية الوصول إلى النظام وتنفيذ العمليات المهمة.

قد يشمل الفحص:

- الوصول إلى صفحة الدخول.
- تسجيل الدخول بحساب مناسب.
- فتح قائمة أساسية.
- قراءة Record معروفة.
- تنزيل Attachment معروفة.

نجاح HTTP Request واحدة لا يثبت سلامة جميع هذه الوظائف.

أي فحص آلي ينشئ أو يعدّل بيانات يجب أن يكون مصممًا لهذا الغرض، وأن يستخدم بيانات وحسابات مخصصة.

راجع [[29 Deployment]] لفحوص ما بعد النشر.

---

### 2. Resources وStorage

راقب موارد البيئة واتجاه تغيرها بمرور الوقت:

| المؤشر | ما الذي يساعد على كشفه؟ |
|---|---|
| CPU | ارتفاع الحمل أو العمل المتكرر |
| Memory | ضغط الذاكرة أو تزايد الاستهلاك |
| Disk Space | اقتراب مساحة التخزين من الامتلاء |
| Database Size | نمو البيانات داخل PostgreSQL |
| Filestore Size | نمو Attachments والملفات |
| Response Time | تباطؤ العمليات التي يستخدمها الأشخاص |

لا تعتمد على حجم الملف وحده؛ راقب المساحة الحرة في وحدات التخزين التي يستخدمها Odoo وPostgreSQL وBackups.

اختر حدود التنبيه حسب سرعة النمو ووقت الاستجابة المتاح، بدل نسخ نسبة ثابتة من بيئة أخرى.

> [!warning] لا تحذف Filestore لمعالجة امتلاء Disk
> ملفات Filestore مرتبطة ببيانات داخل Database.
> حذفها يدويًا قد يجعل Attachments موجودة في الواجهة لكن محتواها غير قابل للفتح.

عند وجود بطء، استخدم طريقة القياس في [[28 Performance]] قبل تغيير Workers أو Indexes.

---

### 3. Errors وLogs

راجع الأخطاء المتكررة والأخطاء الجديدة، واربطها بالعملية التي فشلت.

من الأمثلة:

- Python Tracebacks.
- فشل Scheduled Actions.
- أخطاء Integrations.
- فشل إرسال Email.
- أخطاء Frontend.
- فشل الاتصال بـPostgreSQL.

وجود `AccessError` قد يعني منع وصول صحيحًا، أو صلاحيات غير مناسبة لمتطلب العمل، أو محاولة وصول غير متوقعة.

لذلك افحص السياق قبل تعديل الصلاحيات.

احتفظ بالحد اللازم للتحليل، مثل:

```text
وقت الخطأ مع Time Zone
البيئة المتأثرة
العملية المطلوبة
نوع Exception
مرجع تقني مناسب للعملية
الإصدار المنشور
```

تجنب تسجيل Passwords وAPI Keys وPayloads حساسة.

وراجع Log Rotation وRetention حتى لا تصبح Logs نفسها سببًا في امتلاء Disk.

راجع [[27 Debugging]] لطريقة قراءة الخطأ وعزل سببه.

---

### 4. Scheduled Actions

في Odoo تستخدم Scheduled Actions الـModel التالية:

```text
ir.cron
```

عند مراجعة Cron، افحص:

- هل يفترض أن تكون مفعلة؟
- ما Scheduler User؟
- ما موعد التنفيذ التالي؟
- هل توجد أخطاء في التنفيذ؟
- هل تتزايد الأعمال المستحقة؟
- هل النتيجة المطلوبة تظهر على Business Records؟

يوفر Odoo معلومات مثل `lastcall` و`failure_count`، لكن لا تفترض أن كل هذه الحقول ظاهرة في Form الافتراضية.

كذلك قد تؤدي الإخفاقات المتكررة إلى تعطيل المهمة تلقائيًا وفق آلية Odoo.

> [!important] التنفيذ والنتيجة شيئان مختلفان
> انتهاء Cron دون Exception لا يثبت أنها عالجت كل العمل المطلوب.
>
> قد تكون Domain غير مناسبة، أو Company Context مختلفة، أو حجم العمل أكبر من قدرة المعالجة الحالية.

في `equipment_training`، المهمة التي بنيناها في [[22 Automation & Scheduled Actions]] تبدأ غير مفعلة عمدًا.

وجودها غير مفعلة أثناء التدريب لا يعني وجود عطل.

---

### 5. Integrations

راجع حالة الاتصال ونتائج العمليات المرسلة أو المستقبلة.

قد تشمل المؤشرات:

- عدد العمليات الفاشلة.
- عمر أقدم عملية معلقة.
- نمو Backlog.
- تكرار Timeout.
- انتهاء صلاحية بيانات Authentication أو إلغاؤها.
- الأحداث المكررة.
- اختلاف النتيجة بين النظامين.

إذا كانت Integration تستخدم Queue أو Outbox، فراجع بياناتها وفق تصميمها الفعلي.

هذه ليست شاشة موحدة تنشأ تلقائيًا لكل Custom Integration في Odoo.

> [!warning] افحص نتيجة العملية قبل Retry
> Timeout لا يثبت أن العملية فشلت على النظام الآخر.
> إعادة إرسال عملية كتابة دون Idempotency قد تنشئ نتيجة مكررة.

راجع [[25 APIs & Integrations]] لتصميم Retries وIdempotency وOutbox.

---

### 6. Email

تابع الرسائل التي فشل إرسالها، مع التمييز بين:

```text
إنشاء Message
    ↓
تجهيز Notification أو Email
    ↓
محاولة الإرسال
    ↓
نتيجة التسليم المتاحة
```

هذه مراحل مختلفة.

يمكن للمستخدم التقني المناسب، بعد تفعيل Developer Mode، مراجعة:

```text
Settings → Technical → Email → Emails
```

افحص سبب الفشل قبل اختيار **Retry** أو **Send Now**.

قد تحتاج المشكلة إلى تعديل Recipient أو إعدادات الإرسال أو معالجة رفض من مزود البريد.

ولا تستخدم عدد Emails المتبقية وحده كمقياس لإجمالي الرسائل المرسلة؛ بعض سجلات الإرسال تُنظف وفق إعداداتها وآليات Odoo.

راجع [[21 Mail, Chatter & Activities]].

---

### 7. Backups وRestore

راجع أدلة نجاح النسخ والاستعادة:

- وقت آخر Backup مكتملة.
- وجود نسخة خارج Server الأصلية.
- اشتمال النسخة على Database وFilestore المطلوبة.
- توافق النسخة مع الكود والإعدادات اللازمة لتشغيلها.
- تطبيق Retention.
- تاريخ آخر Restore Test.
- نتيجة فحص Records وAttachments بعد الاستعادة.
- زمن الاستعادة الفعلي مقارنة بهدف RTO.

تغير حجم Backup بشكل غير معتاد يحتاج إلى تفسير، لكنه لا يثبت وحده أن النسخة سليمة أو تالفة.

راجع [[30 Backup & Restore]] للخطوات العملية.

---

### 8. Users وAccess وSecrets

راجع دوريًا:

- الحسابات التي لم تعد تحتاج إلى الوصول.
- Membership في Groups الحساسة.
- حسابات Integrations.
- الحسابات التي تنفذ Scheduled Actions.
- API Keys غير المستخدمة.
- الأسرار التي تحتاج إلى استبدال أو إلغاء.

قبل تعطيل حساب، افحص ارتباطه بعمليات خلفية أو Integration قائمة.

وعند استبدال Secret، تحقق من نجاح الجهة التي تستخدمها، ثم ألغِ القديمة وفق خطة التغيير.

راجع [[11 Security & Multi-Company]].

---

## Maintenance Routine

### توزيع العمل حسب التكرار

الجدول التالي نقطة بداية، ويُعدّل حسب أهمية النظام وحجم العمل:

| التكرار | أمثلة على المراجعة |
|---|---|
| مستمر أو متكرر خلال اليوم | Availability، أخطاء حرجة، مساحة التخزين، تأخر أعمال أساسية |
| يومي | نتائج Backups، Cron، Integrations، Email Failures |
| أسبوعي | اتجاه نمو Storage، الأخطاء المتكررة، Backlog، التغييرات الأخيرة |
| دوري حسب خطة الاستعادة | Restore Test وقياس RTO |
| دوري حسب خطة الوصول | Users وGroups وAPI Keys |
| قبل Release | Tests، Upgrade Rehearsal، Backup، خطة الاستعادة |
| بعد Release | Smoke Tests، الأخطاء الجديدة، نتائج العمليات الأساسية |

لا تنتظر المراجعة الشهرية لمشكلة قد تسبب أثرًا خلال ساعات.

وفي المقابل، لا تجعل كل Warning تنبيهًا عاجلًا؛ كثرة التنبيهات التي لا تحتاج إجراء تجعل متابعة التنبيهات المهمة أصعب.

---

### سجل النتيجة والدليل

بدل الاكتفاء بـ:

```text
Backup checked
```

اكتب نتيجة يمكن الرجوع إليها:

```text
Backup مكتملة.
نسخة خارج Server موجودة.
آخر Restore Test نجحت مع فحص Record وAttachment.
مرجع الاختبار محفوظ في سجل الصيانة.
```

السجل الجيد يوضح ما تم فحصه، وما بقي دون تحقق، وما يحتاج إجراءً لاحقًا.

---

## Practical Maintenance Review

سنراجع مشروع `equipment_training` دون إضافة Fields أو Models جديدة.

الهدف هو التحقق من حالة البيئة ونتائج العمليات التي بنيناها.

---

### 1. تحديد البيئة ونطاق المراجعة

قبل الفحص، سجل:

- هل البيئة Development أم Staging أم Production؟
- ما Database المستخدمة؟
- ما إصدار Odoo ومرجع الكود المنشور؟
- ما Config المستخدمة؟
- هل Cron وIntegrations مفعلة أم معطلة عمدًا؟

أمثلة Databases التدريبية في هذا المرجع:

```text
odoo20_training
odoo20_tests
odoo20_restore_test
```

لكل واحدة غرض مختلف.

عدم تشغيل Cron داخل بيئة Restore معزولة قد يكون نتيجة الإعداد المقصود للبيئة.

---

### 2. فحص الوظائف الأساسية من الواجهة

داخل Database التدريب:

1. سجل الدخول بحساب مناسب.
2. افتح **Equipment → Assets**.
3. افتح جهازًا معروفًا.
4. تحقق من قراءة بياناته.
5. افتح **Equipment → Overview**.
6. تحقق من ظهور نتيجة أو رسالة خطأ مفهومة.
7. إذا كانت لديك Attachment تدريبية معروفة، افتح محتواها.

راجع النتيجة وفق صلاحيات الحساب والشركات المفعلة.

اختلاف Count بين حسابين قد يكون صحيحًا بسبب اختلاف الوصول إلى Records.

---

### 3. مراجعة Cron الخاصة بطلبات المراجعة

فعّل Developer Mode، ثم افتح:

```text
Settings → Technical → Automation → Scheduled Actions
```

ابحث عن المهمة التي تستدعي:

```python
model._cron_schedule_reviews()
```

راجع:

- حالة التفعيل المتوقعة.
- Scheduler User.
- الفترة بين مرات التنفيذ.
- موعد التنفيذ التالي.
- الأخطاء المرتبطة بالمهمة.

في مثالنا، صُممت المعالجة ضمن Company حساب التنفيذ كما شرحنا في شابتر Automation.

لذلك لا تفترض أنها تعالج كل Companies.

> [!note] Run Manually تنفذ العمل فعليًا
> الزر يشغّل المهمة، وقد يغير Records أو ينشئ Activities.
> استخدمه عندما يكون التشغيل مقصودًا وبعد فهم نطاق البيانات المتأثرة.

للمراجعة الروتينية، ابدأ بقراءة الإعدادات والنتائج والسجلات.

---

### 4. التحقق من نتيجة المعالجة

إذا كانت المهمة قد شُغلت على بيانات مستحقة تحقق شروطها:

1. افتح الجهاز المستهدف.
2. تحقق من وجود Review Activity المناسبة.
3. راجع المستخدم المسؤول وDeadline.
4. تحقق من معالجة `review_request_date` وفق السلوك الذي بنيناه.
5. راجع وجود Activities مفتوحة مكررة دون سبب متوقع.

إذا لم تظهر النتيجة، افحص شروط الاختيار أولًا:

- هل الجهاز Active؟
- هل حالته `available`؟
- هل تاريخ الطلب مستحق؟
- هل Company ضمن نطاق التنفيذ؟
- هل حساب التنفيذ يملك الوصول المطلوب؟

ثم راجع Logs.

هذا يربط حالة Cron بالنتيجة التي يفترض أن تحققها.

---

### 5. مراجعة Backups والاستعادة

افتح سجل Backups وتحقق من آخر نسخة ناجحة ومرجع آخر Restore Test.

إذا حان موعد الاختبار، نفّذ الإجراء الكامل في [[30 Backup & Restore]] داخل بيئة معزولة.

يشمل التحقق:

- إمكانية تسجيل الدخول.
- وجود Record معروفة.
- إمكانية فتح محتوى Attachment.
- عزل الإرسال الخارجي والعمليات التي قد تؤثر على الأنظمة الحقيقية.

وجود ملف ZIP وحده لا يكمل هذا الفحص.

---

### 6. تسجيل نتيجة المراجعة

يمكن إنشاء Note منفصلة في Obsidian لكل مراجعة باستخدام القالب التالي:

```text
Maintenance Review

Environment:
Database:
Odoo revision:
Custom code revision:
Reviewed at:
Time zone:
Reviewer:

Checks:
- Core screens:
- Known record:
- Known attachment:
- Scheduled actions:
- Integrations:
- Email failures:
- Storage:
- Latest backup:
- Latest restore test:
- Access and secrets:

Findings:
- Observation:
- Business impact:
- Evidence:

Follow-up:
- Required action:
- Owner:
- Due date:
- Verification result:
```

استخدم وصفًا واضحًا مثل:

```text
Cron غير مفعلة عمدًا في بيئة Restore.
لم يتم تنفيذ Run Manually.
لا توجد نتيجة تشغيل مطلوب التحقق منها في هذه المراجعة.
```

هذه النتيجة أدق من وضع علامة نجاح دون توضيح ما تم فحصه.

---

## Incident Response

### 1. تحديد الأثر

ابدأ بتحديد نطاق المشكلة:

- هل النظام كله متوقف؟
- هل المشكلة تخص وظيفة معينة؟
- هل توجد بيانات غير صحيحة؟
- هل توجد عمليات مكررة؟
- هل هناك خطر وصول غير مسموح؟
- من المتأثر، ومنذ متى؟

الأثر يساعد على تحديد أولوية الاستجابة.

---

### 2. احتواء الضرر

اختر إجراءً يناسب المشكلة.

قد يكون:

- إيقاف Cron معينة مؤقتًا.
- تعليق Integration ترسل عمليات مكررة.
- منع عملية كتابة متأثرة.
- تقييد حساب أو مفتاح ثبت أنه سبب المشكلة.

سجل وقت الإيقاف وسببه، لأن الأعمال قد تتراكم أثناء التعليق وتحتاج إلى معالجة لاحقة.

---

### 3. حفظ الأدلة وتشخيص السبب

احتفظ بالمعلومات اللازمة للتحليل قبل حذف سجلات أو تغيير إعدادات:

- وقت المشكلة وTime Zone.
- أول خطأ مفيد مرتبط بها.
- مرجع Release.
- آخر تغيير ذي صلة.
- مراجع العمليات المتأثرة.

لا تحذف Failed Records فقط لإزالة مؤشر الخطأ.

ولا تبدأ بتكرار Restart دون معرفة ما سيحققه؛ فقد تختفي الأعراض مؤقتًا بينما يبقى السبب.

استخدم خطوات [[27 Debugging]].

---

### 4. الإصلاح والتحقق من التعافي

عند تطبيق الإصلاح، تحقق من:

- نجاح العملية التي فشلت.
- صحة البيانات المتأثرة.
- عدم تكرار العملية الخارجية دون قصد.
- معالجة Backlog.
- عودة Cron أو Integration المعلقة عند الحاجة.
- عدم ظهور أخطاء جديدة.

إذا احتاج الإصلاح إلى تعديل بيانات، فحدد Records المستهدفة وطريقة التحقق قبل التنفيذ.

إصلاح الكود وحده قد لا يصحح نتائج العمليات السابقة.

---

### 5. توثيق الحادثة

اكتب سجلًا مختصرًا يتضمن:

```text
الأثر
التسلسل الزمني
السبب
إجراء الاحتواء
الإصلاح
طريقة التحقق
الإجراء الذي يقلل تكرار المشكلة
```

قد يكون الإجراء اللاحق Test جديدة، أو Alert، أو تحسينًا في Retry، أو تعديل Runbook.

---

## Updates & Change Management

### أنواع التغيير

ميّز بين:

| التغيير | مثال |
|---|---|
| Odoo Bugfix Update | تحديث كود Odoo داخل نفس Major Version |
| Custom Module Change | تعديل كود `equipment_training` |
| Module Upgrade | تطبيق تغييرات Module على Database |
| Major Version Upgrade | الانتقال من Odoo 19 إلى Odoo 20 |
| Infrastructure Update | تحديث نظام التشغيل أو PostgreSQL أو Dependencies |

هذه العمليات قد تتطلب إجراءات تحقق مختلفة.

تحديث ملفات Python لا يعني تلقائيًا تطبيق تغييرات Fields وViews وData على Database.

راجع [[31 Upgrades & Data Migration]].

---

### مسار التغيير

جهّز تغييرًا محددًا يمكن اختباره والرجوع إلى مرجعه:

```text
تحديد التغيير والإصدارات
    ↓
Tests مناسبة
    ↓
تجربة على Staging ببيانات ممثلة ومعزولة
    ↓
تجهيز Backup وخطة استعادة
    ↓
تنفيذ التغيير
    ↓
Smoke Tests
    ↓
مراقبة النتائج
```

في Self-hosted، تشمل مسؤولية التشغيل متابعة تحديثات Odoo والبنية المحيطة بها.

في البيئات المُدارة، تختلف مسؤولية تحديث البنية حسب الخدمة، وتبقى الحاجة إلى التحقق من أثر التحديثات على Custom Modules وعمليات العمل.

---

### Git ومرجع الإصدار

احتفظ بمرجع واضح للكود المستخدم في كل Release:

- Odoo revision.
- Enterprise revision عند استخدامها.
- Custom Modules revision.
- Dependencies المطلوبة.
- التغييرات المهمة في الإعدادات.

اجعل Commits قابلة للمراجعة، واشرح سبب التغيير عندما لا يكون واضحًا من الكود.

ولا تجعل سحب آخر تغييرات Repository مباشرة على Production هو إجراء الصيانة المعتاد؛ اختبر Release محددة قبل نشرها.

> [!warning] Code Rollback لا تعيد كل الآثار
> إرجاع الكود لا يعكس تلقائيًا تغييرات Schema أو Data Migration أو العمليات التي نُفذت على نظام خارجي.
> جهّز خطة الاستعادة حسب نوع التغيير.

---

## Operational Documentation

### Runbook

Runbook هي خطوات تنفيذ عملية تشغيلية أو التعامل مع حالة محددة.

من الأمثلة:

- Deployment.
- Backup.
- Restore.
- Cron Failure.
- Integration Failure.
- Disk Space Alert.

ينبغي أن توضح:

| العنصر | الغرض |
|---|---|
| Scope | البيئة أو الخدمة التي ينطبق عليها الإجراء |
| Owner | المسؤول عن التنفيذ والمتابعة |
| Preconditions | ما يجب التحقق منه قبل البدء |
| Steps | الخطوات الفعلية |
| Expected Result | النتيجة التي تثبت النجاح |
| Recovery | ما الذي يحدث إذا فشل الإجراء؟ |
| Evidence | ما الذي يسجل بعد التنفيذ؟ |

لا تحفظ قيمة Secret داخل Runbook؛ وثق مكان إدارتها وطريقة الوصول المصرح بها.

---

### Maintenance Checklist

القائمة التالية مناسبة كنقطة بداية داخل Obsidian:

- [ ] تحديد البيئة ومرجع الإصدار.
- [ ] مراجعة الوظائف الأساسية.
- [ ] مراجعة الأخطاء الجديدة والمتكررة.
- [ ] مراجعة Cron ونتائجها الفعلية.
- [ ] مراجعة Integration Failures وBacklog.
- [ ] مراجعة Email Failures عند استخدام الإرسال.
- [ ] مراجعة المساحة الحرة واتجاه نمو Storage.
- [ ] التحقق من آخر Backup ونسختها خارج Server.
- [ ] مراجعة موعد ونتيجة آخر Restore Test.
- [ ] تنفيذ مراجعة Access وSecrets المستحقة.
- [ ] مراجعة التحديثات المطلوبة وخطة اختبارها.
- [ ] تسجيل النتائج وتحديد Owner لكل متابعة.

عدّل القائمة حسب النظام؛ لا تجعل وضع العلامات بديلًا عن التحقق.

---

## Important Notes

- Maintenance تشمل استمرارية العمل وصحة البيانات وإمكانية الاستعادة.
- راقب Business Results إلى جانب الموارد والأخطاء.
- نجاح Cron دون Exception لا يثبت اكتمال العمل المطلوب.
- تعطيل Cron قد يكون مقصودًا؛ تحقق من إعداد البيئة.
- Run Manually قد تنفذ تغييرات فعلية.
- افحص نتيجة العملية الخارجية قبل Retry بعد Timeout.
- لا تحذف Filestore أو Failed Records لمعالجة الأعراض.
- Backup تحتاج إلى Restore Test وأدلة تحقق.
- راجع Access وSecrets مع فهم العمليات التي تعتمد عليها.
- ميّز تحديث كود Odoo عن Module Upgrade وMajor Version Upgrade.
- احتفظ بمرجع الإصدار وخطة استعادة تناسب التغيير.
- لكل Alert مهمة مسؤول وإجراء متابعة واضح.

---

> [!tip] مراجعة المرجع
> بهذا يكتمل مسار الشابترات الأساسية.
>
> ارجع إلى [[00 Overview]] لمراجعة الخريطة، واستخدم مشروع التدريب لربط التطوير والاختبار والنشر والصيانة في دورة عمل واحدة.

---

## Sources

- [Odoo 20 Documentation — System Configuration](https://www.odoo.com/documentation/20.0/administration/on_premise/deploy.html)
- [Odoo 20 Documentation — Scheduled Actions](https://www.odoo.com/documentation/20.0/developer/reference/backend/actions.html#scheduled-actions-ir-cron)
- [Odoo 20 Source — Scheduled Actions Implementation](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_cron.py)
- [Odoo 20 Source — Scheduled Actions Views](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/views/ir_cron_views.xml)
- [Odoo 20 Documentation — Common Emailing Issues and Solutions](https://www.odoo.com/documentation/20.0/applications/general/email_communication/faq.html)
- [Odoo 20 Documentation — Bugfix Updates](https://www.odoo.com/documentation/20.0/administration/on_premise/update.html)
- [Odoo 20 Documentation — Upgrade](https://www.odoo.com/documentation/20.0/administration/upgrade.html)