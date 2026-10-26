---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 26 Testing

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Testing Fundamentals|Testing Fundamentals]]
- [[#Practical Example|Practical Example]]
- [[#Other Testing Layers|Other Testing Layers]]
- [[#Running Tests|Running Tests]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

عندما نضيف Business Rule إلى Model، نحتاج إلى التأكد من أنها تعمل في الحالة الصحيحة، وترفض الحالات المخالفة، وتستمر في ذلك بعد التعديلات القادمة.

هذا هو دور **Testing**.

في مشروع الأجهزة، نريد التحقق من متطلبات مثل:

- يستطيع Equipment Manager بدء استخدام جهاز متاح.
- لا يمكن إعادة جهاز متقاعد إلى الاستخدام.
- يستطيع Equipment User قراءة الأجهزة، ويُمنع من تعديلها.
- تخضع قراءة الأجهزة لنطاق الشركات.
- يجب أن تكون نهاية الإعارة بعد بدايتها.
- تتحدث المدة المحسوبة عند تعديل التواريخ.

هذه اختبارات لسلوك النظام الذي بنيناه في الشابترات السابقة.

سنستخدم Python Tests داخل Module، ونشغّلها على Database مستقلة باسم:

```text
odoo20_tests
```

---

## Testing Fundamentals

### تحويل Requirement إلى Test

ابدأ بوصف السلوك المطلوب، ثم حدد ثلاثة أجزاء:

```text
Arrange
تجهيز الحالة
    ↓
Act
تنفيذ العملية
    ↓
Assert
التحقق من النتيجة
```

مثال:

| الجزء | التطبيق |
| --- | --- |
| Requirement | يستطيع Manager بدء استخدام جهاز متاح |
| Arrange | جهاز حالته `available` وحساب Manager |
| Act | استدعاء `action_start_use()` |
| Assert | أصبحت الحالة `in_use` |

وللحالة المرفوضة:

| الجزء | التطبيق |
| --- | --- |
| Requirement | الجهاز المتقاعد لا يعود إلى الاستخدام |
| Arrange | جهاز حالته `retired` |
| Act | استدعاء `action_start_use()` |
| Assert | حدوث `UserError` وبقاء الحالة `retired` |

وجود اختبار للحالة المسموحة واختبار للحالة المرفوضة يجعل المقصود من القاعدة أوضح.

---

### مكان الاختبارات واكتشافها

نضع Python Tests داخل مجلد `tests`:

```text
equipment_training/
└── tests/
    ├── __init__.py
    └── test_equipment.py
```

داخل `tests/__init__.py`:

```python
from . import test_equipment
```

يكتشف Odoo حزمة الاختبارات من خلال Test Loader.

احرص على:

- تسمية الملف ببداية `test_`.
- استيراده داخل `tests/__init__.py`.
- تسمية Test Methods ببداية `test_`.

لا تضف استيراد `tests` إلى Root `__init__.py`؛ اترك تحميلها لآلية الاختبارات.

كما أن ملفات Python Tests لا تضاف إلى قائمة `data` في Manifest.

---

### TransactionCase

سنستخدم:

```python
from odoo.tests import TransactionCase
```

توفر `TransactionCase` بيئة ORM للاختبار.

في تنفيذ Odoo 20 الذي راجعناه، تعمل Test Methods ضمن Transaction مشتركة للـClass، مع عزل كل Test باستخدام Savepoint وإرجاع تغييراتها بعد انتهائها.

هذا يسمح بتجهيز بيانات مشتركة دون أن تعتمد الاختبارات على تغييرات بعضها.

لا تستدعِ `commit()` داخل هذه الاختبارات.

---

### `setUpClass()` و`setUp()`

| الأداة | متى تعمل؟ | الاستخدام المناسب |
| --- | --- | --- |
| `setUpClass()` | مرة لكل Test Class | تجهيز الشركات والمستخدمين والبيانات المشتركة |
| `setUp()` | قبل كل Test Method | تجهيز حالة خاصة متكررة قبل كل اختبار |

ابدأ دائمًا باستدعاء التنفيذ الأصلي:

```python
@classmethod
def setUpClass(cls):
    super().setUpClass()
```

وفي المثال سنجهز البيانات المشتركة باستخدام `setUpClass()`.

لا تعتمد على ترتيب تنفيذ الاختبارات، ولا تجعل Test تحتاج إلى نجاح Test أخرى قبلها.

---

### Assertions

الـAssertion تحدد النتيجة التي يجب أن تتحقق.

| Assertion | الاستخدام |
| --- | --- |
| `assertEqual()` | التحقق من تساوي قيمتين |
| `assertTrue()` | التحقق من شرط صحيح |
| `assertFalse()` | التحقق من شرط غير صحيح |
| `assertIn()` | التحقق من وجود عنصر |
| `assertAlmostEqual()` | مقارنة أرقام عشرية بدقة مناسبة |
| `assertRaises()` | التحقق من حدوث Exception محددة |

مثال:

```python
self.assertEqual(asset.state, "in_use")
```

نجاح استدعاء Method دون Exception لا يثبت وحده أن أثرها صحيح؛ افحص النتيجة المطلوبة.

---

### Expected Exceptions وSavepoints

عندما يجب أن ترفض العملية، اختبر نوع الخطأ المتوقع:

```python
with self.assertRaises(UserError):
    asset.action_start_use()
```

إذا لم يحدث الخطأ، يفشل الاختبار.

وفي Odoo 20، تدير `assertRaises()` الخاصة ببيئة الاختبارات Savepoint داخليًا عند توفر Environment.

لذلك لا نحتاج إلى إضافة:

```python
self.cr.savepoint()
```

إلى كل استخدام لها في هذا الشابتر.

يمكن استخدام Savepoints صراحةً عند وجود حاجة محددة، لكن افهم أولًا ما يوفره Test Framework.

تجنب:

```python
with self.assertRaises(Exception):
    ...
```

لأنه قد يعتبر خطأ برمجيًا غير مقصود نتيجة ناجحة للاختبار.

---

### المستخدم الذي ينفذ العملية

لا يكفي اختبار Security باستخدام حساب إداري.

سنجهز مستخدمين محدودين، ثم ننفذ العملية باستخدام:

```python
record.with_user(user)
```

أما سياق الشركات فنحدده صراحةً باستخدام:

```python
record.with_context(
    allowed_company_ids=[company.id],
)
```

تغيير المستخدم وسياق الشركات مسؤوليتان مختلفتان؛ جهزهما بما يناسب الحالة التي تريد اختبارها.

---

## Practical Example

### 1. تحديد القواعد التي سنختبرها

يعتمد المثال على التنفيذ الذي بنيناه سابقًا:

| الجزء | التعريف |
| --- | --- |
| Asset Model | `equipment.asset` |
| Loan Model | `equipment.loan` |
| حالات الجهاز | `available` و`in_use` و`retired` |
| بدء الاستخدام | `action_start_use()` |
| إرجاع الجهاز | `action_return()` |
| تقاعد الجهاز | `action_retire()` |
| مجموعة المستخدم | `equipment_training.group_equipment_user` |
| مجموعة المدير | `equipment_training.group_equipment_manager` |

ويفترض وجود قواعد الصلاحيات والشركات من [[11 Security & Multi-Company]]، ومنطق الانتقالات من [[15 Business Logic]]، وقيد التواريخ من [[17 Constraints]].

سنكتب ثمانية اختبارات:

1. الانتقال إلى الاستخدام ثم العودة إلى الإتاحة.
2. رفض بدء استخدام جهاز متقاعد.
3. السماح للمستخدم بقراءة الجهاز.
4. رفض تعديل الجهاز بواسطة المستخدم.
5. رفض بدء الاستخدام بواسطة المستخدم.
6. منع الوصول إلى جهاز خارج نطاق الشركة.
7. رفض نهاية إعارة تساوي بدايتها.
8. إعادة حساب مدة الإعارة بعد تعديل النهاية.

---

### 2. تجهيز الملفات

داخل Module، أضف:

```text
equipment_training/
└── tests/
    ├── __init__.py
    └── test_equipment.py
```

داخل `tests/__init__.py`:

```python
from . import test_equipment
```

---

### 3. كتابة Test Class

داخل:

```text
equipment_training/tests/test_equipment.py
```

اكتب:

```python
from datetime import datetime

from odoo import Command
from odoo.exceptions import AccessError, UserError, ValidationError
from odoo.tests import TransactionCase, tagged
from odoo.tests.common import new_test_user


@tagged("post_install", "-at_install")
class TestEquipment(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()

        cls.company_a = cls.env["res.company"].create({
            "name": "Equipment Test Company A",
        })

        cls.company_b = cls.env["res.company"].create({
            "name": "Equipment Test Company B",
        })

        cls.manager = new_test_user(
            cls.env,
            login="equipment_test_manager",
            groups=(
                "base.group_user,"
                "equipment_training.group_equipment_manager"
            ),
            company_id=cls.company_a.id,
            company_ids=[
                Command.set([
                    cls.company_a.id,
                    cls.company_b.id,
                ]),
            ],
        )

        cls.reader = new_test_user(
            cls.env,
            login="equipment_test_reader",
            groups=(
                "base.group_user,"
                "equipment_training.group_equipment_user"
            ),
            company_id=cls.company_a.id,
            company_ids=[
                Command.set([cls.company_a.id]),
            ],
        )

        cls.Asset = (
            cls.env["equipment.asset"]
            .with_user(cls.manager)
            .with_context(
                allowed_company_ids=[cls.company_a.id],
            )
        )

        cls.asset = cls.Asset.create({
            "name": "Test Equipment",
            "company_id": cls.company_a.id,
        })

        cls.other_asset = cls.Asset.with_context(
            allowed_company_ids=[cls.company_b.id],
        ).create({
            "name": "Other Company Equipment",
            "company_id": cls.company_b.id,
        })

        cls.reader_asset = cls.asset.with_user(
            cls.reader,
        ).with_context(
            allowed_company_ids=[cls.company_a.id],
        )

        cls.Loan = (
            cls.env["equipment.loan"]
            .with_user(cls.manager)
            .with_context(
                allowed_company_ids=[cls.company_a.id],
            )
        )

        cls.loan = cls.Loan.create({
            "name": "Test Equipment Loan",
            "company_id": cls.company_a.id,
            "requested_by_id": cls.manager.id,
            "asset_id": cls.asset.id,
            "start_at": datetime(2026, 1, 15, 9, 0, 0),
            "end_at": datetime(2026, 1, 15, 10, 0, 0),
        })

    def test_manager_can_start_and_return_equipment(self):
        self.assertEqual(self.asset.state, "available")

        self.asset.action_start_use()

        self.assertEqual(self.asset.state, "in_use")

        self.asset.action_return()

        self.assertEqual(self.asset.state, "available")

    def test_retired_equipment_cannot_start_use(self):
        self.asset.action_retire()

        self.assertEqual(self.asset.state, "retired")

        with self.assertRaises(UserError):
            self.asset.action_start_use()

        self.assertEqual(self.asset.state, "retired")

    def test_reader_can_read_equipment(self):
        values = self.reader_asset.read(["name"])

        self.assertEqual(
            values[0]["name"],
            "Test Equipment",
        )

    def test_reader_cannot_write_equipment(self):
        with self.assertRaises(AccessError):
            self.reader_asset.write({
                "name": "Changed Equipment",
            })

        self.assertEqual(
            self.asset.name,
            "Test Equipment",
        )

    def test_reader_cannot_start_equipment_use(self):
        with self.assertRaises(AccessError):
            self.reader_asset.action_start_use()

        self.assertEqual(self.asset.state, "available")

    def test_reader_cannot_access_other_company_equipment(self):
        ReaderAsset = (
            self.env["equipment.asset"]
            .with_user(self.reader)
            .with_context(
                allowed_company_ids=[self.company_a.id],
            )
        )

        found = ReaderAsset.search([
            ("id", "=", self.other_asset.id),
        ])

        self.assertFalse(found)

        with self.assertRaises(AccessError):
            ReaderAsset.browse(
                self.other_asset.id,
            ).read(["name"])

    def test_loan_end_must_be_after_start(self):
        original_end = self.loan.end_at

        with self.assertRaises(ValidationError):
            self.loan.write({
                "end_at": self.loan.start_at,
            })

        self.assertEqual(
            self.loan.end_at,
            original_end,
        )

    def test_duration_updates_when_end_changes(self):
        self.assertAlmostEqual(
            self.loan.duration_hours,
            1.0,
        )

        self.loan.write({
            "end_at": datetime(2026, 1, 15, 11, 0, 0),
        })

        self.assertAlmostEqual(
            self.loan.duration_hours,
            2.0,
        )
```

---

### 4. فهم بيانات الاختبار

جهزنا شركتين ومستخدمين:

| الحساب | الدور | الشركات المسموحة |
| --- | --- | --- |
| `manager` | Equipment Manager | A وB |
| `reader` | Equipment User | A فقط |

استخدمنا `new_test_user()` لإنشاء حسابات الاختبار مع Groups محددة، بدل الاعتماد على مستخدم موجود مسبقًا في Database.

ثم أنشأنا:

- جهازًا في Company A.
- جهازًا في Company B.
- إعارة صحيحة مدتها ساعة في Company A.

الأسماء والتواريخ بيانات اختبار اصطناعية.

لا يعتمد المثال على Demo Data أوعلى أرقام IDs ثابتة.

---

### 5. فهم اختبارات Security

اختبار القراءة المسموحة مهم بقدر اختبار الرفض.

إذا كتبنا اختبارات رفض فقط، فقد تنجح حتى لو كان النظام يمنع المستخدم من كل شيء.

لذلك نتحقق من:

```text
المستخدم يستطيع القراءة المطلوبة
                +
المستخدم يُمنع من التعديل غير المسموح
```

واختبار استدعاء `action_start_use()` يثبت أن الحماية موجودة في مسار Server، حتى لو حاول المستخدم تجاوز Button.

أما اختبار الشركات فيتحقق من سلوكين مختلفين:

| العملية | الناتج المتوقع |
| --- | --- |
| البحث عن جهاز غير مسموح | لا يظهر ضمن النتائج |
| قراءة الجهاز مباشرةً بواسطة ID | يحدث `AccessError` |

وجود Recordset ناتجة من `browse()` لا يثبت السماح بقراءة Record؛ لذلك ننفذ `read()` داخل الاختبار.

لا تستخدم `sudo()` على العملية التي تريد اختبار صلاحياتها.

---

### 6. فهم اختبار Constraint وComputed Field

اختبار التواريخ يبدأ بإعارة صحيحة، ثم يحاول تعديلها إلى حالة مخالفة.

```text
بداية: 09:00
نهاية: 10:00
      ↓
محاولة جعل النهاية 09:00
      ↓
ValidationError
```

وبعد الرفض نتحقق من بقاء النهاية الأصلية.

أما اختبار المدة فيتحقق من أثر تعديل Dependency:

```text
09:00 → 10:00
مدة ساعة
      ↓
تغيير النهاية إلى 11:00
      ↓
مدة ساعتين
```

هذا يختبر إعادة الحساب التي بنيناها في [[16 Computed Fields, Related Fields & Onchange]].

لا نستدعي Compute Method يدويًا؛ نعدل المصدر ثم نقرأ النتيجة، حتى نختبر مسار ORM المعتاد.

---

### 7. فهم Test Tags

استخدمنا:

```python
@tagged("post_install", "-at_install")
```

المقصود:

| Tag | الأثر |
| --- | --- |
| `post_install` | اختيار مرحلة ما بعد اكتمال تحميل Modules |
| `-at_install` | إزالة مرحلة الاختبار المباشرة بعد تثبيت Module |

الـDecorator تضيف Tags أوتزيلها من الموجودة، ولا تستبدل المجموعة كاملةً.

لهذا نحدد المرحلة المطلوبة صراحةً في المثال.

إذا أردت اختبار سلوك Module مباشرةً بعد تثبيتها، يمكن اختيار:

```python
@tagged("at_install", "-post_install")
```

اختر المرحلة وفق ما يحتاجه الاختبار؛ مرحلة التشغيل لا تحدد وحدها إن كان الاختبار Unit Test أوIntegration Test.

---

## Other Testing Layers

### اختبار Onchange باستخدام Form

استدعاء ORM مثل:

```python
loan.write({
    "start_at": new_start,
})
```

لا يحاكي تلقائيًا سلوك تحرير Form في المتصفح.

عندما يكون المطلوب اختبار اقتراح قيمة أوOnchange، يمكن استخدام:

```python
from odoo.tests import Form
```

توفر `Form` محاكاة من جهة Server لجزء من سلوك Forms، مثل Defaults وOnchanges.

لكنها لا تختبر عرض الصفحة الفعلي أوأحداث JavaScript في المتصفح.

اختر View وصلاحيات مناسبة عند استخدامها، لأن Fields المتاحة وخصائصها تؤثر في الاختبار.

---

### HTTP وPortal

استخدم `HttpCase` عندما تحتاج إلى اختبار مسار HTTP أوتفاعل يتطلب Server HTTP.

من أمثلة مشروعنا في [[23 HTTP Controllers & Portal]]:

- مستخدم مخول يصل إلى الصفحة.
- مستخدم غير مسجل يُطلب منه تسجيل الدخول.
- Portal User يستطيع الوصول إلى إعاراته.
- تغيير ID في الرابط لا يسمح بقراءة إعارة مستخدم آخر.

اختبار Domain داخل Model مفيد، لكنه لا يغطي وحده طريقة معالجة Controller للطلب والاستجابة.

---

### JavaScript وOWL

Python Tests لا تتحقق تلقائيًا من سلوك Dashboard التي بنيناها في [[24 OWL & Frontend]].

لاختبارات JavaScript، توفر Odoo إطار **HOOT** وأدوات Web Test Helpers.

يمكن اختبار حالات مثل:

- بدء التحميل عند فتح Component.
- عرض العدد صفرًا.
- تعطيل Refresh أثناء الطلب.
- عرض رسالة عند فشل الطلب.

توجد اختبارات JavaScript عادةً داخل:

```text
static/tests/
```

وتستخدم أسماء تنتهي بـ:

```text
.test.js
```

وتضاف إلى Bundle:

```text
web.assets_unit_tests
```

أما Tours فتساعد على اختبار مسار مستخدم يربط الواجهة وسلوك Server.

---

### Integrations وExternal Services

اختبارات منطق التكامل المعتادة يجب أن تكون قابلة للتكرار دون الاعتماد على خدمة خارجية حية.

استخدم Mock لمحاكاة نتائج مثل:

- نجاح الطلب.
- Timeout.
- استجابة غير صالحة.
- تكرار Event.
- رفض النظام الخارجي.

واحتفظ باختبارات منفصلة للتحقق من التوافق الفعلي مع الخدمة الخارجية عند الحاجة.

هذا يكمل مبادئ [[25 APIs & Integrations]] دون جعل كل تشغيل للاختبارات يعتمد على الشبكة أوينشئ أثرًا خارجيًا حقيقيًا.

---

## Running Tests

### 1. تجهيز بيئة التشغيل

نفذ الأوامر من مجلد:

```text
odoo20-dev
```

وتأكد أن `addons_path` في Config تتضمن Module التدريب.

سنستخدم:

```text
odoo20_tests
```

ولا نستبدل بها Database العمل اليومية أوProduction.

سنخصص Port:

```text
8071
```

لتشغيل الاختبارات، ونعطل Cron Workers في هذه العملية حتى لا تنفذ أعمالًا مجدولة بالتوازي معها.

---

### 2. أول تشغيل باستخدام Installation

عند استخدام Database اختبار جديدة، ثبّت Module مع تشغيل الاختبارات.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_tests \
  -i equipment_training \
  --test-tags /equipment_training \
  --http-port 8071 \
  --max-cron-threads 0 \
  --stop-after-init
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\odoo.conf `
  -d odoo20_tests `
  -i equipment_training `
  --test-tags /equipment_training `
  --http-port 8071 `
  --max-cron-threads 0 `
  --stop-after-init
```

مع إعداد PostgreSQL مناسب، يستطيع Odoo تهيئة Database غير موجودة عند التثبيت.

إذا كان حساب PostgreSQL لا يملك صلاحية إنشاء Databases، جهز Database اختبار بالطريقة المناسبة لبيئتك أولًا.

---

### 3. التشغيل التالي باستخدام Upgrade

بعد تثبيت Module داخل Database الاختبار، استخدم `-u` لتطبيق تغييراتها وإعادة الاختبارات.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_tests \
  -u equipment_training \
  --test-tags /equipment_training \
  --http-port 8071 \
  --max-cron-threads 0 \
  --stop-after-init
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\odoo.conf `
  -d odoo20_tests `
  -u equipment_training `
  --test-tags /equipment_training `
  --http-port 8071 `
  --max-cron-threads 0 `
  --stop-after-init
```

---

### 4. فهم خيارات التشغيل

| الخيار | الوظيفة |
| --- | --- |
| `-d` | تحديد Database الاختبار |
| `-i` | تثبيت Module |
| `-u` | تحديث Module المثبتة |
| `--test-tags` | تفعيل الاختبارات وتحديد الاختبارات المختارة |
| `--http-port` | تحديد Port لعملية الاختبار |
| `--max-cron-threads 0` | تعطيل Cron Workers في هذه العملية |
| `--stop-after-init` | إنهاء Process بعد مرحلة التهيئة والاختبارات |

استخدام `--test-tags` يفعّل Testing؛ لذلك لا يلزم تكرار `--test-enable` معه.

تحديد Tags لا يثبت Module أويحدثها وحده؛ لهذا نجمعه مع `-i` أو`-u` في الأوامر السابقة.

---

### 5. تشغيل Test محددة

لتحديد Test واحدة، استبدل قيمة `--test-tags` بهذه القيمة:

```text
/equipment_training:TestEquipment.test_reader_cannot_write_equipment
```

ولتشغيل Class كاملة:

```text
/equipment_training:TestEquipment
```

ولتشغيل الاختبارات القياسية في Module:

```text
/equipment_training
```

هذه الصيغة تفترض بقاء Tag `standard`، كما في مثالنا.

إذا أزلت `standard` من Class، فاخترها باستخدام Tag أوSelector مناسبين بدل افتراض أن اختيار Module سيشملها دائمًا.

---

### 6. قراءة النتيجة

تحقق من أن السجل يعرض تنفيذ Test Class وMethods المتوقعة، ثم راجع ملخص النتائج.

| النتيجة | معناها |
| --- | --- |
| Passed | تحقق السلوك المتوقع |
| Failure | لم تتحقق Assertion |
| Error | حدث خطأ منع إكمال الاختبار |
| Skipped | لم ينفذ الاختبار بسبب شرط تخطٍّ |

Class المثال تحتوي ثماني Test Methods.

إذا لم تظهر، تحقق من:

1. Import داخل `tests/__init__.py`.
2. اسم ملف الاختبار.
3. أسماء Test Methods.
4. اختيار Tags ومرحلة التشغيل.
5. Module المستخدمة وDatabase المستهدفة.
6. مسار Addons الذي قرأ منه Odoo الكود.

> [!important] صفر Tests تحتاج إلى تفسير
> انتهاء Process دون أخطاء لا يثبت أن الاختبارات المطلوبة نُفذت.
>
> تحقق من عدد الاختبارات وأسمائها، إلى جانب ملخص النجاح.

---

### 7. Installation وUpgrade وMigration

هذه حالات مختلفة:

| الحالة | ما الذي تتحقق منه؟ |
| --- | --- |
| Installation جديدة | قدرة Module على العمل من بداية نظيفة |
| Upgrade على Test Database | استمرار السلوك بعد تطبيق التغييرات |
| Migration Test | صحة تحويل بيانات قديمة إلى الشكل الجديد |

تشغيل Test الحالية بعد Upgrade لا يثبت وحده أن بيانات إصدار قديم تحولت بصورة صحيحة.

اختبار Migration يحتاج إلى بيانات تمثل الحالة القديمة، وتشغيل مسار التحويل، ثم التحقق من النتيجة.

سنعود إلى ذلك في [[31 Upgrades & Data Migration]].

---

## Important Notes

- ابدأ بالـRequirement، ثم صمم البيانات والعملية والنتيجة المتوقعة.
- اختبر الحالات المسموحة والمرفوضة معًا.
- استخدم بيانات اختبار مستقلة عن Demo Data وعن IDs ثابتة.
- استورد ملفات الاختبارات من `tests/__init__.py`، واترك تحميل الحزمة لـTest Loader.
- لا تضف Python Tests إلى `data` في Manifest.
- اختبر Security باسم المستخدم المستهدف، مع سياق شركات واضح.
- لا تستخدم `sudo()` على العملية التي تختبر صلاحياتها.
- `browse()` وحدها لا تثبت السماح بقراءة Record.
- استخدم Exception محددة داخل `assertRaises()`.
- `assertRaises()` في Odoo 20 تدير Savepoint داخليًا عند توفر Environment.
- لا تستدعِ `commit()` داخل اختبارات `TransactionCase` المعتادة.
- لا تجعل الاختبارات تعتمد على ترتيب التنفيذ.
- حدد مرحلة الاختبار بوضوح عند استخدام Tags.
- تأكد من اكتشاف الاختبارات وتشغيلها؛ صفر Tests ليست دليل نجاح.
- اختبارات Python لا تغطي تلقائيًا JavaScript أوPDF Layout أوكل سلوك المتصفح.
- عند فشل اختبار، تحقق هل اكتشف خطأ في التنفيذ أم أن Requirement تغيرت؛ لا تعدل Assertion لمجرد جعل النتيجة خضراء.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[27 Debugging]]

---

## Sources

- [Odoo 20 Documentation — Testing Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/testing.html)
- [Odoo 20 Documentation — Safeguard Your Code with Unit Tests](https://www.odoo.com/documentation/20.0/developer/tutorials/unit_tests.html)
- [Odoo 20 Documentation — Command-line Interface: Testing](https://www.odoo.com/documentation/20.0/developer/reference/cli.html#testing)
- [Odoo 20 Documentation — JavaScript Unit Testing](https://www.odoo.com/documentation/20.0/developer/reference/frontend/unit_testing.html)
- [Odoo 20 Official Source — TransactionCase, Assertions and Test Helpers](https://github.com/odoo/odoo/blob/20.0/odoo/tests/common.py)
- [Odoo 20 Official Source — Test Loader](https://github.com/odoo/odoo/blob/20.0/odoo/tests/loader.py)
- [Odoo 20 Official Source — Test Tag Selection](https://github.com/odoo/odoo/blob/20.0/odoo/tests/tag_selector.py)