---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 31 Upgrades & Data Migration

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Upgrade Fundamentals|Upgrade Fundamentals]]
- [[#Upgrade Scripts|Upgrade Scripts]]
- [[#Planning Data Changes|Planning Data Changes]]
- [[#Practical Example|Practical Example]]
- [[#Verification|Verification]]
- [[#Major Version Upgrade|Major Version Upgrade]]
- [[#Rollback & Recovery|Rollback & Recovery]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

قد نحتاج إلى تعديل Module مثبتة، أوتحويل بيانات قديمة، أوالانتقال إلى إصدار أحدث من Odoo.

هذه عمليات مترابطة، لكنها ليست العملية نفسها.

مثلًا:

- تعديل Python Method قد يحتاج إلى Restart.
- إضافة Field وView تحتاج إلى Module Upgrade مناسبة.
- تعبئة Field جديدة للـRecords القديمة قد تحتاج إلى Data Migration.
- الانتقال من Odoo 19 إلى Odoo 20 يحتاج إلى مسار Major Version Upgrade متكامل.

في هذا الشابتر سنوضح الفرق، ثم نضيف Field إلى مشروع الأجهزة ونهيئ قيمتها للبيانات الموجودة باستخدام Upgrade Script.

---

## Upgrade Fundamentals

### أنواع التغيير

| العملية | ما الذي تفعله؟ |
| --- | --- |
| Restart | إعادة تشغيل Process وتحميل الكود في بيئة تشغيل جديدة |
| Module Upgrade | تطبيق تحديث Module مثبتة داخل Database |
| Data Migration | تحويل بيانات أوعلاقات أوتعريفات من شكل إلى آخر |
| Major Version Upgrade | نقل Database وتخصيصاتها إلى إصدار Odoo أحدث |

قد يتضمن إصدار واحد أكثر من عملية.

مثلًا:

```text
كود جديد
   ↓
Module Upgrade
   ├── تحديث Schema
   ├── تحميل Data وViews
   └── تنفيذ Upgrade Scripts المناسبة
```

---

### Module Upgrade

نستخدم Module Upgrade لتطبيق تغييرات Module المثبتة، مثل:

- Fields.
- Views وActions.
- Security Data.
- ملفات XML وCSV.
- تحويلات البيانات المرتبطة بالإصدار.

يمكن تنفيذها من Apps أوCLI حسب بيئة العمل وخطة التشغيل.

لكن Upgrade لا تستطيع استنتاج المعنى التجاري لكل تغيير.

إذا أصبحت قيمة مطلوبة في Records قديمة، فنحتاج إلى تحديد مصدرها الصحيح.

---

### Major Version Upgrade

تشغيل:

```text
-u equipment_training
```

باستخدام Odoo 20 ليس خطة لتحويل Database من Odoo 19 إلى Odoo 20.

ترقية الإصدار الأساسي تحتاج إلى معالجة توافق:

```text
Standard Odoo Database
+
Custom Module Code
+
Custom Data
+
Dependencies
+
Integrations
```

سنعرض مسارها لاحقًا، بينما المثال العملي هنا يحدّث Module داخل Odoo 20.

---

## Upgrade Scripts

### مكان الملفات

يمكن وضع Upgrade Scripts داخل:

```text
equipment_training/
└── upgrades/
    └── 20.0.3.0.0/
        └── post-10-fill-reference.py
```

ويدعم Odoo أيضًا اسم المجلد:

```text
migrations
```

سنستخدم `upgrades` في مشروعنا.

اسم المجلد يحدد Version المرتبطة بالتحويل، وبداية اسم الملف تحدد مرحلة التنفيذ.

لا تضف هذه الملفات إلى `data` في Manifest، ولا تستوردها من `__init__.py`.

يتولى Upgrade Loader اكتشافها.

---

### Entry Point

يحتوي الملف على:

```python
def migrate(cr, version):
    ...
```

| Parameter | المعنى |
| --- | --- |
| `cr` | Cursor الخاصة بعملية التحديث |
| `version` | Version المثبتة التي بدأ منها تحديث Module |

ليست `version` هي Version الهدف الموجودة في اسم المجلد.

دع Framework يدير Transaction؛ لا تضف `commit()` يدويًا داخل Script المعتادة.

---

### شروط اختيار Script

في مسار Module Upgrade المعتاد، يجب أن تكون Version الخاصة بمجلد Script:

```text
أعلى من Version المثبتة
          و
أقل من أو تساوي Version الهدف
```

مثال:

```text
Installed:
20.0.2.0.0

Target:
20.0.3.0.0

Script:
upgrades/20.0.3.0.0/post-10-fill-reference.py
```

إذا بقيت Version الهدف مساوية للمثبتة، فلا تفترض أن مجرد تنفيذ `-u` سيعيد Script الخاصة بتلك Version.

كما أن هذه Scripts لا تعمل تلقائيًا عند Installation جديدة في المسار المعتاد.

---

### مراحل التنفيذ

| المرحلة | توقيتها | الاستخدام المعتاد |
| --- | --- | --- |
| `pre` | قبل تحميل وتحديث Module | تجهيز البنية القديمة أوإعادة تسمية مطلوبة قبل تحميل التعريف الجديد |
| `post` | بعد تحميل وتحديث Module واعتمادياتها | تعبئة Field جديدة أوتحويل يحتاج إلى التعريف الجديد |
| `end` | بعد تحميل وتحديث جميع Modules المعنية | أعمال تحتاج إلى اكتمال التحديثات المرتبطة |

داخل المرحلة نفسها، يحدد الترتيب المعجمي لأسماء الملفات ترتيب التنفيذ.

مثلًا:

```text
post-10-fill-reference.py
post-20-verify-reference.py
```

اختيار المرحلة يعتمد على البيانات والبنية التي يجب أن تكون متاحة عند تنفيذ التحويل.

---

### Scripts جزء من Upgrade

المسار الصحيح:

```text
بدء Module Upgrade
        ↓
pre scripts
        ↓
تحميل وتحديث Module
        ↓
post scripts
        ↓
استكمال تحديث Modules
        ↓
end scripts
```

لا تشغّل ملفات التحويل يدويًا كبرامج Python مستقلة لمجرد أن أسماءها تنتهي بـ`.py`.

---

## Planning Data Changes

### تحديد قاعدة التحويل

قبل كتابة Script، حدد:

- ما بيانات المصدر؟
- ما القيمة المستهدفة؟
- أي Records تدخل في التحويل؟
- هل تشمل المؤرشفة وكل الشركات؟
- ماذا نفعل بالقيم الفارغة؟
- هل نحافظ على قيمة موجودة في الهدف؟
- كيف نتحقق من اكتمال التحويل؟
- هل توجد قواعد أوتكاملات تعتمد على الحقل القديم؟

اكتب القاعدة بصورة يمكن اختبارها.

لا تجعل Script تخمن معنى البيانات.

---

### Defaults والبيانات التاريخية

افترض أننا نريد جعل `company_id` مطلوبة، بينما توجد Records قديمة بلا Company.

إضافة:

```python
default=lambda self: self.env.company
```

ليست وحدها قاعدة تجارية صحيحة لتوزيع كل Records التاريخية.

يجب تحديد الشركة المناسبة لكل Record، ثم معالجة البيانات قبل فرض القاعدة التي تحتاج إليها.

وينطبق الأمر على:

- إضافة قيمة مطلوبة.
- تشديد Constraint.
- تغيير معنى Selection.
- تحويل علاقة بين Models.
- فصل Field واحدة إلى أكثر من Field.

---

### تغيير الاسم وتغيير المعنى

| التغيير | ما الذي يجب مراجعته؟ |
| --- | --- |
| Rename | البيانات والمراجع التي تستخدم الاسم القديم |
| تغيير Type | قابلية تحويل القيم القديمة |
| إضافة Constraint | البيانات المخالفة الموجودة |
| حذف Field | البيانات والمراجع التي ستفقد |
| تغيير معنى القيمة | قواعد العمل والتقارير والتكاملات |

تغيير اسم Attribute في Python لا يضمن وحده نقل بيانات Column القديمة إلى الاسم الجديد.

لعمليات Renaming المعقدة، راجع أدوات Odoo الرسمية في Upgrade Utils.

هذه مكتبة تحتاج إلى تجهيز ضمن بيئة الترقية؛ لا تفترض توفرها تلقائيًا في كل تثبيت.

---

## Practical Example

### 1. تحديد المطلوب

سنضيف إلى الجهاز Field اختيارية باسم:

```text
reference
```

وستكون مرجعًا داخليًا مستقلًا.

قاعدة تهيئة البيانات الموجودة:

| الحالة وقت التحويل | النتيجة |
| --- | --- |
| Reference فارغة وSerial Number موجودة | نسخ الرقم التسلسلي إلى Reference |
| Serial Number فارغة | إبقاء Reference فارغة |
| Reference تحتوي قيمة | الحفاظ عليها |

سنشمل الأجهزة المؤرشفة وكل الشركات.

بعد التحويل، لا تتزامن `reference` و`serial_number` تلقائيًا؛ هذه تهيئة للبيانات الموجودة، وليست Related Field.

---

### 2. تجهيز نقطة البداية

قبل تعديل الكود:

1. تحقق من Version المثبتة.
2. جهز أجهزة تدريب ببيانات معروفة.
3. سجل الأعداد والقيم التي ستفحصها.
4. خذ Backup مع الكود المتوافق.
5. استعدها في بيئة معزولة وفق [[30 Backup & Restore]].

أرقام Version في المثال توضيحية:

```text
20.0.2.0.0 → 20.0.3.0.0
```

إذا كانت Version المثبتة في مشروعك مختلفة، اختر Version هدف أعلى منها، واجعل Manifest ومجلد Script متوافقين معها.

لا تعدل Version المسجلة داخل Database لإجبار Script على العمل.

---

### 3. فحص Version المثبتة

من Odoo Shell المناسبة لنسخة التجربة:

```python
module = env["ir.module.module"].search(
    [("name", "=", "equipment_training")],
    limit=1,
)

module.ensure_one()

module.latest_version
```

في هذا Model، الاسم التقني `latest_version` يمثل Version المثبتة داخل Database.

أما `installed_version` في الكود الحالي فيعكس Version المتاحة من ملفات Module.

الأسماء هنا قد تكون مربكة؛ تحقق من المعلومة المطلوبة بدل الاعتماد على الاسم وحده.

---

### 4. إضافة Field

داخل Class الحالية الخاصة بـ`equipment.asset`، أضف:

```python
reference = fields.Char(
    string="Reference",
    copy=False,
)
```

احتفظ بتعريف `serial_number` وبقية Model.

أبقينا Field اختيارية؛ لأن بعض الأجهزة قد لا تحتوي رقمًا تسلسليًا ننسخه منها.

وفي Form الجهاز داخل:

```text
views/equipment_views.xml
```

أضف بعد Field الرقم التسلسلي:

```xml
<field name="reference"/>
```

لا تضف تعريف Form جديدة إذا كان المطلوب تعديل Form الموجودة.

---

### 5. تحديث Manifest Version

داخل `__manifest__.py`، عدل قيمة `version` الموجودة:

```python
"version": "20.0.3.0.0",
```

استخدم Version الهدف التي اخترتها وفق نقطة البداية الفعلية.

رفع Version وحده لا ينفذ التحديث داخل Database؛ يجب تطبيق Module Upgrade.

---

### 6. إنشاء Script

أضف:

```text
equipment_training/
└── upgrades/
    └── 20.0.3.0.0/
        └── post-10-fill-reference.py
```

اخترنا `post` لأننا نحتاج إلى Column الجديدة بعد تحديث تعريف Model.

داخل الملف:

```python
import logging

from odoo import SUPERUSER_ID, api


_logger = logging.getLogger(__name__)


def migrate(cr, version):
    env = api.Environment(cr, SUPERUSER_ID, {})

    Asset = env["equipment.asset"]

    Asset.flush_model([
        "reference",
        "serial_number",
    ])

    cr.execute("""
        UPDATE equipment_asset
           SET reference = serial_number
         WHERE reference IS NULL
           AND serial_number IS NOT NULL
           AND serial_number <> ''
        RETURNING id
    """)

    updated_ids = [
        row[0]
        for row in cr.fetchall()
    ]

    updated_assets = Asset.browse(updated_ids)

    updated_assets.invalidate_recordset([
        "reference",
    ])

    updated_assets.modified([
        "reference",
    ])

    _logger.info(
        "Filled reference for %s equipment assets; "
        "previous module version: %s",
        len(updated_ids),
        version,
    )
```

---

### 7. فهم اختيار SQL

هذا تحويل بيانات إداري محدد أثناء Upgrade، ويستهدف كل صف مطابق داخل جدول الأجهزة.

لذلك يشمل المؤرشف والشركات المختلفة، ولا يعتمد على Filters الخاصة بمستخدم الواجهة.

استخدمنا:

| الجزء | الغرض |
| --- | --- |
| `flush_model()` | تجهيز القيم المطلوبة في Database قبل SQL |
| شرط Reference الفارغة | الحفاظ على القيم الموجودة |
| شروط Serial Number | نسخ القيم الموجودة فقط |
| `RETURNING id` | معرفة Records التي تغيرت |
| `invalidate_recordset()` | إزالة قيم Cache القديمة للحقل |
| `modified()` | إخطار ORM بالتغيير وتجهيز Dependencies عند وجودها |

SQL لا تنفذ `write()` Overrides أوPython Constraints أوChatter Tracking تلقائيًا.

اخترناها هنا لتحويل تاريخي محدود لا يحتاج إلى تلك الآثار. إذا كان التحويل يتطلب Business Logic أوآثارًا مرتبطة بها، فصمم التنفيذ باستخدام ORM أوالأدوات المناسبة.

وللجداول الكبيرة، راجع حجم النتائج والأقفال والذاكرة، ثم اختبر طريقة التجميع المناسبة وفق [[28 Performance]].

---

### 8. تطبيق Upgrade على النسخة المعزولة

يمكن تطبيق التحديث من الواجهة بعد إعادة تشغيل Server بالكود الجديد:

1. افتح **Apps**.
2. احذف Filter **Apps** عند الحاجة.
3. ابحث عن `Equipment Training`.
4. نفذ **Upgrade**.
5. راجع Logs وتحقق من تنفيذ Script.

وفي تجربة مضبوطة من CLI، أوقف Process التي تستخدم Database التجربة، ثم نفذ إحدى الطريقتين التاليتين.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/restore-test.conf \
  -d odoo20_restore_test \
  -u equipment_training \
  --workers 0 \
  --max-cron-threads 0 \
  --no-http \
  --stop-after-init
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\restore-test.conf `
  -d odoo20_restore_test `
  -u equipment_training `
  --workers 0 `
  --max-cron-threads 0 `
  --no-http `
  --stop-after-init
```

الأوامر تستهدف Database التجربة من الشابتر السابق.

إذا استخدمت اسمًا مختلفًا، عدله مع Config بما يطابق البيئة الفعلية.

---

### 9. فحص التنفيذ

راجع Logs بحثًا عن:

```text
post-10-fill-reference
```

ورسالة عدد الأجهزة التي عُدلت.

ثم شغّل نسخة التجربة بالطريقة المعتادة، وافتح جهازًا كان له رقم تسلسلي قبل التحديث.

الناتج المتوقع:

```text
Serial Number:
TRAIN-MIG-001

Reference:
TRAIN-MIG-001
```

وتحقق من بقاء القيم والـRecords الأصلية وفق قاعدة التحويل.

عدد صفوف معدلة يساوي صفرًا قد يكون صحيحًا إذا لم توجد بيانات مطابقة، أوقد يكشف مشكلة في تجهيز التجربة؛ قارنه بتوقعاتك المسجلة.

---

## Verification

### مصفوفة التحقق

| الحالة | المتوقع |
| --- | --- |
| جهاز موجود برقم تسلسلي | تهيئة Reference بالقيمة نفسها |
| جهاز بلا رقم تسلسلي | Reference فارغة |
| جهاز مؤرشف برقم تسلسلي | يدخل في التحويل |
| جهاز في Company أخرى | يدخل في التحويل الإداري |
| عدد الأجهزة قبل وبعد | لا يتغير بسبب هذا التحويل |
| الرقم التسلسلي الأصلي | يبقى كما هو |
| Business Actions | تستمر وفق القواعد السابقة |
| صلاحيات المستخدمين | تستمر كما صممناها |

تحقق باسم مستخدم مناسب عند فحص واجهة الشركات والصلاحيات.

---

### اختبار مسارين مختلفين

نحتاج إلى اختبار:

```text
Installation جديدة
```

و:

```text
Upgrade من Version سابقة تحتوي بيانات
```

الأول يتحقق من أن تعريف Module الجديد يعمل من البداية.

الثاني يتحقق من أن البيانات القديمة تحولت بصورة صحيحة.

نجاح Installation لا يثبت نجاح Migration؛ Scripts المعتادة لا تعمل تلقائيًا في هذا المسار.

---

### إعادة التجربة

بعد نجاح Upgrade وتسجيل Version الجديدة، لا تتوقع أن تعيد `-u` نفسها تشغيل Script الخاصة بالنسخة المكتملة.

لإعادة اختبار التحويل كاملًا:

1. استعد Backup جديدة للحالة السابقة باسم تجربة جديد.
2. استخدم الكود المستهدف نفسه.
3. نفذ Upgrade.
4. قارن النتائج بالتوقعات.

حافظ على Script المنشورة كسجل للتحويل الذي نفذته الإصدارات السابقة.

إذا احتجت إلى تصحيح لاحق على Database وصلت بالفعل إلى الإصدار الجديد، فجهز Version وScript لاحقتين.

---

### الاختبارات الآلية وBusiness Acceptance

استخدم اختبارات [[26 Testing]] للتحقق من السلوك المستهدف، واختبارات Upgrade المناسبة للتحقق من البيانات قبل التحويل وبعده.

ثم نفذ قبولًا وظيفيًا يشمل:

- فتح Views.
- العلاقات والبيانات المعروفة.
- Reports وPDF.
- Security وMulti-Company.
- Wizards.
- Cron والتكاملات ضمن البيئة المعزولة.
- OWL والـAssets المرتبطة.

لا تكتفِ بانتهاء الأمر دون Traceback.

---

## Major Version Upgrade

### مسار ترقية Database مخصصة

عند الانتقال مثلًا من Odoo 19 إلى Odoo 20:

1. ثبت هوية الكود والبيانات المصدر.
2. حدد مسار ترقية Database المدعوم لبيئتك.
3. جهز Custom Modules للإصدار الهدف.
4. اختبر Installation جديدة على الإصدار الهدف.
5. اختبر Modules على نسخة Database المرقاة.
6. نفذ تحويلات البيانات المخصصة المطلوبة.
7. أعد اختبارات السلوك والتكاملات.
8. نفذ Rehearsal لعملية Production كاملة.
9. اعتمد خطة النشر والاستعادة.

توفر Odoo مسارات وأدوات لترقية Database وفق بيئة الاستضافة والتغطية المتاحة.

وتظل مسؤولية تكييف Custom Code وبياناتها جزءًا واضحًا من خطة المشروع.

---

### مراجعة Custom Modules

راجع التغييرات في:

| الجزء | أمثلة لما قد يتأثر |
| --- | --- |
| Python وORM | Methods وتواقيعها وسلوكها |
| Models وFields | إزالة تعريفات أوإعادة تسميتها |
| Security | آليات وقواعد وتعريفات الوصول |
| Views | البنية وXPath وAttributes |
| Frontend | OWL وServices وPlugins وAssets |
| Reports | Templates والتنسيق والتبعيات |
| Integrations | Authentication وEndpoints وتنسيق البيانات |
| Tests | Fixtures وHelpers والتوقعات |

تغيير Version داخل Manifest إلى `20.0` لا يكفي لإثبات التوافق.

---

## Rollback & Recovery

### حدود الرجوع إلى الكود

إعادة Commit سابقة لا تعكس تلقائيًا:

- Schema Changes.
- تحويل القيم.
- حذف البيانات.
- تغيير العلاقات.
- الآثار التي حدثت في أنظمة خارجية.

حدد قبل النشر هل الرجوع يحتاج إلى:

```text
الكود السابق
+
Database وFilestore المتوافقتين
+
Dependencies وConfig
```

راجع خطة [[29 Deployment]] ونسخ [[30 Backup & Restore]].

---

### فشل Upgrade

إذا فشلت العملية:

1. احتفظ بالـLogs.
2. حدد المرحلة والخطأ.
3. افحص حالة Database بدل افتراض اكتمال التراجع.
4. تجنب تشغيل كود غير متوافق عليها.
5. أصلح السبب وأعد التجربة من نقطة بداية معروفة.

لا تستخدم `commit()` داخل Script لتثبيت جزء من التحويل وتجاوز الفشل.

ولا تحذف بيانات المصدر لمجرد أن نسخة منها ظهرت في الحقل الجديد؛ حدد خطة الإزالة والتحقق منها بصورة مستقلة.

---

## Important Notes

- فرق بين Restart وModule Upgrade وMajor Version Upgrade وData Migration.
- `-u` ليست خطة لترقية Database بين إصدارات Odoo الرئيسية.
- Upgrade Scripts تعمل داخل عملية التحديث تلقائيًا.
- اختر مرحلة `pre` أو`post` أو`end` وفق البنية المتاحة وقت التنفيذ.
- مجلد Version يجب أن يحقق شروط الاختيار مقارنةً بالمثبتة والهدف.
- لا تضف Scripts إلى Manifest Data أوImports المعتادة.
- Installation جديدة لا تختبر تحويل البيانات القديمة.
- لا تعدل Version داخل Database لإجبار Script على التكرار.
- Default ليست بديلًا عن قاعدة صحيحة للبيانات التاريخية.
- SQL تتجاوز سلوك ORM؛ راجع الحاجة إلى Flush وCache Invalidation وDependencies.
- لا تضف `commit()` يدويًا إلى Script المعتادة.
- احتفظ ببيانات المصدر إلى أن تنجح خطوات التحقق المعتمدة.
- اختبر من Starting Version وبيانات تمثل البيئة الفعلية.
- لإعادة Migration Test، استعد نقطة البداية السابقة.
- التصحيح بعد إصدار مكتمل يحتاج إلى مسار لاحق يصل إلى Databases التي رُقيت بالفعل.
- Code Rollback لا يعيد Database أوالآثار الخارجية تلقائيًا.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[32 Translations & Localization]]

---

## Sources

- [Odoo 20 Documentation — Upgrade](https://www.odoo.com/documentation/20.0/administration/upgrade.html)
- [Odoo 20 Documentation — Upgrade Scripts](https://www.odoo.com/documentation/20.0/developer/reference/upgrades/upgrade_scripts.html)
- [Odoo 20 Documentation — Upgrade Utils](https://www.odoo.com/documentation/20.0/developer/reference/upgrades/upgrade_utils.html)
- [Odoo 20 Documentation — Upgrade a Customized Database](https://www.odoo.com/documentation/20.0/developer/howtos/upgrade_custom_db.html)
- [Odoo 20 Documentation — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Official Source — Migration Manager](https://github.com/odoo/odoo/blob/20.0/odoo/modules/migration.py)
- [Odoo 20 Official Source — Module Loading](https://github.com/odoo/odoo/blob/20.0/odoo/modules/loading.py)
- [Odoo 20 Official Source — Module Version Fields](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_module.py)