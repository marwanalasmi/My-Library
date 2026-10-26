---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 30 Backup & Restore

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Backup Fundamentals|Backup Fundamentals]]
- [[#Recovery Planning|Recovery Planning]]
- [[#Practical Backup & Restore|Practical Backup & Restore]]
- [[#Restore Verification|Restore Verification]]
- [[#Production Recovery|Production Recovery]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

وجود Backup لا يثبت وحده أننا نستطيع استعادة Odoo.

قد تكون Database سليمة، لكن المرفقات مفقودة. وقد توجد البيانات والملفات، لكن Custom Modules أوDependencies اللازمة لتشغيلها غير متاحة.

لذلك ننظر إلى الاستعادة بوصفها إعادة تجهيز حالة تشغيل مترابطة:

```text
Database
+
Filestore
+
Odoo Source
+
Custom Modules
+
Dependencies
+
Configuration
+
إمكانية استعادة الأسرار المطلوبة
```

في هذا الشابتر سنأخذ نسخة من Database التدريب، ثم نستعيدها باسم جديد ونفحص التطبيق والمرفقات وعزل الاتصالات الخارجية.

---

## Backup Fundamentals

### ماذا نحتاج إلى حفظه؟

| الجزء | ما الذي يحتويه أويوفره؟ |
| --- | --- |
| Database | Business Records وتعريفات وإعدادات محفوظة داخل Odoo |
| Filestore | المرفقات التي تستخدم التخزين على القرص |
| Odoo Source | الإصدار الذي يشغّل Database |
| Custom Modules | الكود المخصص وتبعياته |
| Dependencies | مكتبات Python وتبعيات النظام |
| Configuration | الاتصال والمسارات وإعدادات التشغيل |
| Secrets Recovery | الوصول إلى المفاتيح وكلمات المرور اللازمة |
| Infrastructure Information | إعدادات الخدمة وProxy وDNS والتخزين |

قد تحفظ بعض هذه الأجزاء داخل Backup، وأجزاء أخرى داخل Git أوSecret Store أوتوثيق التشغيل.

المطلوب أن تكون متاحة ومترابطة عند الاستعادة.

---

### Database وFilestore

Database تحتوي معلومات عن Records والمرفقات.

عندما يكون محتوى المرفق محفوظًا داخل Filestore، تحتاج الاستعادة إلى الملف نفسه أيضًا.

```text
Attachment Record في Database
              ↓
ملف مرتبط داخل Filestore
```

وجود Record لا يضمن وجود محتوى الملف.

ويعتمد مسار Filestore المعتاد على:

```text
data_dir/filestore/<database_name>
```

لذلك يجب معرفة `data_dir` التي يستخدمها Server الفعلي.

---

### اتساق النسخة

نريد Database وFilestore من حالة متوافقة.

إذا استمرت عمليات إنشاء الملفات وحذفها أثناء النسخ، فقد نحصل على Database تشير إلى ملف لم يدخل النسخة.

تجميعهما داخل ZIP لا يعني وحده أن عملية التقاطهما كانت Snapshot ذرية.

في تجربة هذا الشابتر:

1. نحفظ بيانات التحقق.
2. نوقف عمليات الكتابة.
3. نوقف Server والأعمال الأخرى التي تعدل Database.
4. نأخذ Backup.

في Production، يمكن استخدام تصميم Backup آخر يدعم العمل المستمر، بشرط أن يضمن الاتساق ويُختبر عمليًا.

---

### ZIP وDatabase Dump

في أمر `db dump`، الصيغة الافتراضية هي ZIP، وتتضمن Filestore عند استخدام الإعداد المعتاد.

تحتوي النسخة عادةً على:

```text
dump.sql
manifest.json
filestore/
```

وجود `filestore/` يعتمد على وجود الملفات في المصدر وتضمينها.

أما:

```text
--no-filestore
```

فيستبعد Filestore من ZIP.

وصيغة:

```text
--format dump
```

تنتج PostgreSQL Dump، وتحتاج إلى خطة منفصلة للملفات وأداة الاستعادة المناسبة.

سنستخدم ZIP في المثال، لتكون تجربة Database والملفات مترابطة.

> [!important] تحقق من مسار المصدر
> إذا استخدمت Config أو`data_dir` مختلفة عن Server الأصلية، فقد لا يجد الأمر Filestore المطلوبة.
>
> نجاح إنشاء ZIP لا يثبت أن كل مرفقات التطبيق دخلت النسخة.

---

### الكود ومعلومات الإصدار

ملف Backup لا يغني عن حفظ Custom Module Source Code.

اربط كل نسخة بمعلومات مثل:

- Odoo Commit.
- Custom Modules Commit أوTag.
- Enterprise Commit عند استخدامها.
- إصدارات Dependencies.
- Config المناسبة.
- تاريخ النسخة واسم Database المصدر.
- مكان النسخة ونتيجة آخر Restore Test.

ابدأ الاستعادة بالكود المتوافق مع النسخة، ثم تعامل مع أي Upgrade لاحقة كعملية مستقلة.

راجع [[29 Deployment]].

---

## Recovery Planning

### RPO

**Recovery Point Objective** تحدد مقدار فقد البيانات المقبول زمنيًا.

مثلًا:

> نحتاج إلى استعادة حالة لا يزيد عمرها على ساعة.

هذا يتطلب وجود نقطة استعادة صالحة تحقق هذا الهدف.

جدولة Backup كل ساعة لا تكفي وحدها إذا كانت المهام تفشل أوالنسخ غير قابلة للاستعادة.

راقب عمر أحدث نسخة صالحة، وليس موعد المهمة فقط.

---

### RTO

**Recovery Time Objective** تحدد الزمن المقبول لاستعادة الخدمة.

يشمل ذلك:

```text
اكتشاف المشكلة واتخاذ القرار
             ↓
الوصول إلى النسخة والأسرار
             ↓
تجهيز البيئة والكود
             ↓
استعادة Database والملفات
             ↓
فحص التطبيق
             ↓
إعادة فتح الخدمة
```

مدة أمر Restore جزء من RTO، وليست المدة كاملةً.

---

### التخزين والحماية وRetention

حدد سياسة توضح:

| العنصر | القرار المطلوب |
| --- | --- |
| Frequency | متى تؤخذ النسخ؟ |
| Retention | كم نسخة نحتفظ بها، ولأي مدة؟ |
| Location | أين تحفظ النسخ؟ |
| Access | من يستطيع قراءتها أواستعادتها؟ |
| Protection | كيف تحمى أثناء النقل والتخزين؟ |
| Verification | متى نختبر الاستعادة؟ |
| Failure Handling | كيف نكتشف فشل Backup؟ |

احتفظ بنسخة خارج Server المصدر، وبحماية تناسب احتمال تعطل Server أوفقدها أوحذف ملفاتها.

Backups نفسها تحتوي بيانات حساسة، وقد تحتوي إعدادات أوأسرارًا محفوظة داخل Database.

---

## Practical Backup & Restore

### 1. تحديد المصدر والهدف

سنستخدم:

| العنصر | القيمة |
| --- | --- |
| Source Database | `odoo20_training` |
| Target Database | `odoo20_restore_test` |
| Backup File | `backups/odoo20-training-backup-001.zip` |
| Restore Config | `config/restore-test.conf` |
| Restore HTTP Port | `8071` |

نفذ الأوامر من مجلد:

```text
odoo20-dev
```

ويفترض المثال توفر أدوات PostgreSQL المطلوبة، ومنها `pg_dump` و`psql`، وصلاحيات إنشاء Database الهدف.

إذا كانت الأدوات خارج المسار المعتاد، راجع إعداد `pg_path` المناسب لبيئتك.

---

### 2. تجهيز بيانات للتحقق

داخل Database التدريب:

1. اختر جهازًا معروفًا، أوجهز جهاز تدريب باسم واضح.
2. سجل اسمه وحالته وCompany.
3. أرفق ملفًا تجريبيًا صغيرًا عبر Chatter.
4. افتح المرفق للتأكد من سلامته.
5. سجل إعارة معروفة إن كانت متاحة.

مثال لبيانات اصطناعية:

```text
Equipment:
Restore Practice Laptop

Attachment:
restore-check.txt
```

يمكن أن يحتوي الملف:

```text
Equipment restore verification.
```

هذه العلامات ستساعدنا على التحقق من أننا فتحنا النسخة الصحيحة وأن المرفق استعيد بالفعل.

---

### 3. تجهيز مجلد النسخ وإيقاف الكتابات

أنشئ مجلدًا باسم:

```text
backups
```

ثم:

1. احفظ التغييرات داخل Odoo.
2. أوقف Server التدريب.
3. تأكد من عدم وجود Script أوWorker أخرى تعدل المصدر.
4. أبقِ PostgreSQL تعمل.

اختر اسم ملف جديدًا لكل Backup؛ لا تعد استخدام اسم نسخة سابقة تريد الاحتفاظ بها.

---

### 4. إنشاء ZIP Backup

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin db \
  -c config/odoo.conf \
  dump \
  --format zip \
  odoo20_training \
  backups/odoo20-training-backup-001.zip
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin db `
  -c config\odoo.conf `
  dump `
  --format zip `
  odoo20_training `
  backups\odoo20-training-backup-001.zip
```

استخدمنا Config المصدر حتى يقرأ الأمر الاتصال و`data_dir` المناسبة.

حددنا ZIP صراحةً، رغم أنها الصيغة الافتراضية، لتكون نية المثال واضحة.

---

### 5. الفحص الأولي للنسخة

بعد انتهاء الأمر:

- راجع Logs ونتيجة العملية.
- تحقق من وجود الملف وحجمه.
- افتح قائمة محتويات ZIP.
- تحقق من وجود `dump.sql` و`manifest.json`.
- تحقق من وجود ملفات داخل `filestore/` عندما يحتوي المصدر مرفقات محفوظة على القرص.

يمكن تسجيل Checksum لاكتشاف تغير الملف أثناء النقل أوالتخزين.

لكن سلامة ZIP أوتطابق Checksum لا يثبتان أن التطبيق سيعمل بعد Restore؛ نكمل الاختبار بالاستعادة.

---

### 6. تجهيز Restore Config

انسخ Config التدريب إلى:

```text
config/restore-test.conf
```

احتفظ بإعدادات الاتصال ومسارات Addons و`data_dir` اللازمة، ثم اضبط القيم التالية:

```ini
db_name = odoo20_restore_test
dbfilter = ^odoo20_restore_test$

http_interface = 127.0.0.1
http_port = 8071

workers = 0
max_cron_threads = 0

proxy_mode = False
```

هذه تعديلات على ملف Config الحالية، وليست ملفًا كاملًا مستقلًا.

في هذا المثال:

- نختار Database الهدف بوضوح.
- نقيد HTTP بالاتصال المحلي.
- نستخدم Port منفصلة.
- نعطل Cron Workers أثناء الفحص.
- نتصل مباشرةً دون Reverse Proxy.

إذا كانت البيئة تسمح باتصالات خارجية، جهز عزل التكاملات قبل الاستعادة والتشغيل.

احتفظ بملفات Config التي تحتوي أسرارًا خارج Git.

---

### 7. الاستعادة باسم جديد مع Neutralization

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin db \
  -c config/restore-test.conf \
  load \
  --neutralize \
  odoo20_restore_test \
  backups/odoo20-training-backup-001.zip
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin db `
  -c config\restore-test.conf `
  load `
  --neutralize `
  odoo20_restore_test `
  backups\odoo20-training-backup-001.zip
```

ينشئ الأمر Database الهدف، ويستعيد البيانات والـFilestore المضمنة، ويطبق Neutralization.

استخدم اسم هدف غير موجود.

> [!warning] خيار `--force`
> يحذف هذا الخيار Database الهدف الموجودة قبل الاستعادة.
>
> لا نستخدمه في تجربة الشابتر. إذا كان الاسم مستخدمًا، اختر اسم تجربة جديدًا وحدّث Config والأوامر وفقه.

إذا فشلت المحاولة، افحص الخطأ وحالة الهدف قبل إعادة التنفيذ؛ لا تفترض أن كل أثر للمحاولة الفاشلة أزيل تلقائيًا.

---

### 8. فهم Neutralization وحدودها

تعطل Neutralization إعدادات تشغيلية معروفة، مثل إرسال البريد وبعض الأعمال المجدولة ومزودي الخدمات.

تجمع آليتها في الفرع الحالي SQL الخاصة بالـModules المثبتة من ملفات مثل:

```text
<module>/data/neutralize.sql
```

لكن Custom Integration قد تحتاج إلى معالجة إضافية.

بعد الاستعادة، راجع:

- SMTP وإرسال البريد.
- Scheduled Actions.
- Outgoing Webhooks.
- خدمات الدفع والشحن.
- API Credentials.
- Jobs أوWorkers خارج Odoo.

Neutralization لا تعني إخفاء البيانات الشخصية، ولا تحول النسخة تلقائيًا إلى بيانات اصطناعية.

---

### 9. تشغيل النسخة المستعادة

بعد مراجعة الاستعادة والعزل:

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/restore-test.conf \
  -d odoo20_restore_test
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\restore-test.conf `
  -d odoo20_restore_test
```

افتح المتصفح وانتقل إلى:

```text
http://127.0.0.1:8071
```

استخدم حساب تدريب معروفًا كان موجودًا وقت أخذ النسخة.

لا تضف `-i` أو`-u` إلى أول تشغيل لمجرد اختبار Restore؛ نريد أولًا التحقق من استعادة الحالة المحفوظة بكودها المتوافق.

---

## Restore Verification

### التحقق من Database والبيانات

تأكد من أنك داخل:

```text
odoo20_restore_test
```

ثم افحص:

| الفحص | النتيجة المطلوبة |
| --- | --- |
| Login | الدخول بالحساب المتوقع |
| Module التدريب | موجودة بالحالة المتوقعة |
| الجهاز المعروف | موجود بالقيم المسجلة |
| الإعارة المعروفة | موجودة بعلاقاتها وتواريخها |
| Company | النطاق والقيم صحيحان |
| Views | تفتح دون أخطاء مرتبطة بالكود أوFields |

إذا ظهرت Models أوFields مفقودة، تحقق من نسخة الكود وImports وAddons Paths قبل محاولة تعديل Database.

راجع [[27 Debugging]].

---

### التحقق من Filestore

افتح المرفق التجريبي وتحقق من محتواه.

ظهور اسم المرفق في Chatter لا يكفي؛ قد تكون Record موجودة بينما الملف مفقود.

افحص أيضًا عينة مناسبة من:

- Images.
- Documents.
- مرفقات Records أخرى.

إذا فشل فتح الملفات، راجع:

- تضمين Filestore داخل ZIP.
- `data_dir` المستخدمة عند النسخ.
- `data_dir` المستخدمة عند الاستعادة والتشغيل.
- اسم Database ومجلد Filestore.
- صلاحيات حساب تشغيل Odoo على الملفات.

---

### التحقق من العزل

تأكد أن النسخة لا تنفذ آثارًا تشغيلية غير مقصودة.

| العنصر | ما الذي نتحقق منه؟ |
| --- | --- |
| Cron | لا تعمل أعمال الإنتاج أثناء الفحص |
| Email | لا تصل رسائل إلى مستلمين فعليين |
| Webhooks | لا ترسل إلى Endpoints الإنتاج |
| Payment وShipping | تستخدم إعدادات اختبار أوتظل معطلة |
| External Workers | لا تعالج Jobs الخاصة بالنسخة |
| Access | النسخة متاحة للنطاق المقصود فقط |

اسم Database مختلف أوظهور علامة Neutralization لا يغنيان عن هذا الفحص.

---

### تسجيل نتيجة Restore Test

سجل نتيجة واضحة:

| المعلومة | ما نسجله؟ |
| --- | --- |
| Backup | اسم الملف وموقعه وتاريخه |
| Source | Database والإصدار المرتبط |
| Target | بيئة وDatabase التجربة |
| Duration | زمن النقل والتجهيز والاستعادة والفحص |
| Data Check | Records التي فحصناها |
| File Check | المرفقات التي فتحناها |
| Isolation | حالة الاتصالات والأعمال الخارجية |
| Problems | الأخطاء والإجراءات المطلوبة |
| Result | هل أصبحت النسخة قابلة للاستخدام؟ |

إذا كانت الاستعادة تحتاج إلى معرفة موجودة لدى شخص واحد فقط، أضف هذه الخطوات إلى Runbook.

---

## Production Recovery

### Restore Test وDisaster Recovery

تجربة الشابتر تنشئ نسخة معزولة للفحص.

أما استعادة Production فتحتاج إلى قرار تشغيل واضح يشمل:

1. إيقاف التغييرات والاتصالات المتعارضة.
2. اختيار نقطة الاستعادة المناسبة.
3. تجهيز الإصدار والـDependencies المتوافقة.
4. استعادة Database وFilestore.
5. إعادة إعداد Domain وProxy والخدمة عند الحاجة.
6. فحص التطبيق.
7. إعادة تفعيل الأعمال والتكاملات تدريجيًا.
8. مراقبة التشغيل.

لا تعِد تفعيل الاتصالات الخارجية قبل التأكد من هوية البيئة والبيانات المستعادة.

---

### البيانات الأحدث من Backup

استعادة نسخة قديمة تعيد البيانات إلى نقطة سابقة.

حدد أثر ذلك على:

- Records المنشأة بعد النسخة.
- عمليات نفذتها أنظمة خارجية.
- Emails أوWebhooks التي أرسلت.
- طلبات قد تعيد أنظمة أخرى إرسالها.

Restore لا تلغي أثرًا حدث في نظام خارجي.

وقد تحتاج إلى تسوية البيانات أوإعادة معالجة عمليات مع حماية من التكرار، وفق [[25 APIs & Integrations]].

---

### الاستعادة ليست Upgrade

استعادة Database لا تحولها تلقائيًا إلى إصدار Odoo أحدث.

المسار المعتاد:

```text
استعادة حالة متوافقة
         ↓
التحقق من سلامتها
         ↓
تجربة Upgrade أوMigration منفصلة
```

سنفصل هذا المسار في [[31 Upgrades & Data Migration]].

---

## Important Notes

- احفظ Database وFilestore بصورة متوافقة.
- ZIP تتضمن Filestore افتراضيًا في الأمر المستخدم، لكن تحقق من المصدر والمحتوى.
- `data_dir` الخاطئة قد تؤدي إلى فقد المرفقات من النسخة أوفشل الوصول إليها.
- وجود ملف ZIP لا يثبت نجاح الاستعادة.
- اربط Backup بالكود وDependencies والإعدادات المناسبة.
- استخدم Database هدف جديدة أثناء Restore Test.
- خيار `--force` يحذف الهدف الموجود؛ لا تستخدمه لتجاوز تشخيص محاولة سابقة.
- جهز عزل الاتصالات قبل تشغيل النسخة المستعادة.
- `--neutralize` لا تعالج تلقائيًا كل Custom Integration.
- Neutralization ليست Anonymization.
- افحص محتوى المرفق، وليس اسمه فقط.
- احتفظ بنسخ محمية خارج Server المصدر.
- ضع Retention Policy وراقب فشل النسخ وعمر أحدث نسخة صالحة.
- قس زمن الاستعادة الكامل مقابل RTO.
- راجع مقدار البيانات المعرضة للفقد مقابل RPO.
- اختبر Restore دوريًا، وحدّث Runbook عند تغير بيئة التشغيل.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[31 Upgrades & Data Migration]]

---

## Sources

- [Odoo 20 Official Source — Database CLI](https://github.com/odoo/odoo/blob/20.0/odoo/cli/db.py)
- [Odoo 20 Official Source — Database Dump and Restore](https://github.com/odoo/odoo/blob/20.0/odoo/modules/db.py)
- [Odoo 20 Documentation — Database CLI](https://www.odoo.com/documentation/20.0/developer/reference/cli.html#db-manage-a-database)
- [Odoo 20 Documentation — Neutralized Database](https://www.odoo.com/documentation/20.0/administration/neutralized_database.html)
- [Odoo 20 Official Source — Neutralization](https://github.com/odoo/odoo/blob/20.0/odoo/modules/neutralize.py)
- [Odoo 20 Official Source — Configuration and Filestore Paths](https://github.com/odoo/odoo/blob/20.0/odoo/tools/config.py)