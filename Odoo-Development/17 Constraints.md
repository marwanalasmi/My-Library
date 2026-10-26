---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 17 Constraints

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Python Constraints|Python Constraints]]
- [[#SQL Constraints|SQL Constraints]]
- [[#Choosing the Right Validation|Choosing the Right Validation]]
- [[#Existing Data & Upgrades|Existing Data & Upgrades]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في [[16 Computed Fields, Related Fields & Onchange]]، حسبنا مدة الإعارة من وقت البداية والنهاية.

لكن الحساب وحده لا يمنع إدخال نهاية تسبق البداية؛ قد يعطينا مدة سالبة، وتبقى البيانات غير صحيحة.

هنا نحتاج إلى **Constraint**: شرط يجب أن تحققه البيانات حتى تُقبل العملية.

تدعم Odoo طريقتين أساسيتين:

| النوع | مكان التنفيذ |
| --- | --- |
| Python Constraint | داخل ORM باستخدام Python |
| SQL Constraint | داخل PostgreSQL |

سنطبق قاعدتين في مشروع التدريب:

1. إذا أُدخل وقتا البداية والنهاية، يجب أن تكون النهاية بعد البداية.
2. لا يتكرر الرقم التسلسلي للجهاز داخل الشركة نفسها.

---

## Python Constraints

### تعريف Constraint

نكتب Method داخل Model، ونربطها بالـFields المعنية باستخدام:

```python
@api.constrains("start_at", "end_at")
```

داخل Method، نفحص Records ونرفع `ValidationError` عند وجود مخالفة.

إذا نجح الفحص، تنتهي Method بشكل طبيعي؛ لا تحتاج إلى إرجاع `True`.

ولا نستخدم `return False` بدل Exception ونتوقع أن ترفض Odoo الحفظ.

---

### فحص Recordset

قد تستقبل Constraint عدة Records، لذلك نفحص كل Record:

```python
for loan in self:
    ...
```

في مثال ترتيب الوقت، نحتاج إلى فحص ثلاث نقاط:

```text
هل البداية موجودة؟
        ↓
هل النهاية موجودة؟
        ↓
هل النهاية بعد البداية؟
```

سنسمح حاليًا بنقص أحد الوقتين، كما في تصميم الشابتر السابق.

أما عند وجودهما معًا، فسنرفض النهاية المساوية للبداية أوالسابقة لها.

---

### متى تُستدعى؟

يربط `@api.constrains` الفحص بالـFields المحددة فيه.

مثلًا:

```python
@api.constrains("start_at", "end_at")
```

يرتبط بتعديل البداية أوالنهاية ضمن مسارات الحفظ التي يعالجها ORM.

لا تفترض أن جميع Constraints تعمل عند كل `write()`.

تعديل اسم الطلب وحده، مثلًا، ليس طلبًا لتغيير الوقتين.

كذلك، إذا كانت القاعدة تعتمد على غياب قيمة من المدخلات، فلا تفترض أن Decorator وحدها تغطي كل حالات الإنشاء؛ قد تحتاج إلى تحقق مناسب في `create()`.

راجع [[15 Business Logic]] لتنظيم القواعد التي يجب أن تغطي الإنشاء والكتابة صراحةً.

---

### أسماء Fields بسيطة

تأخذ `@api.constrains` أسماء Fields مباشرة من Model:

```python
@api.constrains("start_at", "end_at")
```

لا تستخدم Dot Notation كما نفعل مع `@api.depends`.

مثلًا، لا تعتمد على:

```python
@api.constrains("asset_id.name")
```

لتتبع تغيير اسم الجهاز.

إذا كانت القاعدة مرتبطة بأكثر من Model، صمم نقاط التحقق بحيث تغطي التغييرات المطلوبة على كل Model معنية.

---

### لا تصلح البيانات داخل Constraint

مسؤولية Constraint هي التحقق من صحة البيانات ورفض المخالفة.

لا تجعلها تعدل القيم تلقائيًا لتخفي المشكلة.

مثلًا، إذا كانت النهاية قبل البداية، لا تغيّر النهاية داخل Constraint حتى ينجح الحفظ.

افصل بين:

| المطلوب | المسؤولية |
| --- | --- |
| حساب قيمة مشتقة | Compute |
| اقتراح قيمة للمستخدم | Onchange |
| تنفيذ تغيير مقصود | Business Method |
| رفض بيانات مخالفة | Constraint |

---

## SQL Constraints

### التعريف في Odoo 20

نعرّف SQL Constraint باستخدام `models.Constraint` داخل Class الخاصة بالـModel.

مثلًا، داخل `equipment.asset`:

```python
_serial_company_unique = models.Constraint(
    "UNIQUE(company_id, serial_number)",
    "The serial number must be unique within the company.",
)
```

التعريف يحتوي:

1. اسم Class Attribute يمثل القيد.
2. تعريف SQL.
3. رسالة تُستخدم عند عرض المخالفة للمستخدم.

اسم Attribute يجب أن يبدأ بشرطة سفلية `_` في هذه الصيغة.

هذا اسم لتعريف تقني، وليس Field جديدة.

---

### `CHECK`

تستخدم `CHECK` لفرض شرط على قيم الصف.

مثلًا، يمكن تمثيل قاعدة ترتيب الوقت بهذه الصيغة:

```python
_time_order = models.Constraint(
    """
    CHECK(
        start_at IS NULL
        OR end_at IS NULL
        OR end_at > start_at
    )
    """,
    "End time must be after start time.",
)
```

التعريف يسمح بنقص أحد الوقتين، ويطلب ترتيبًا صحيحًا عند وجودهما.

هذه صيغة بديلة عن Python Constraint الخاصة بالوقت؛ لن نضيف النسختين معًا في التطبيق.

اخترنا Python لشرح Decorator، وSQL لتطبيق قاعدة تفرد مختلفة.

---

### `UNIQUE`

تحدد `UNIQUE` نطاق التفرد بحسب الأعمدة المدرجة فيها.

| التعريف | نطاق التفرد |
| --- | --- |
| `UNIQUE(serial_number)` | الرقم فريد على مستوى الجدول |
| `UNIQUE(company_id, serial_number)` | الرقم فريد داخل كل شركة |

في تطبيقنا:

| الشركة | الرقم التسلسلي | القرار |
| --- | --- | --- |
| Company A | `SER-100` | مسموح |
| Company A | `SER-100` مرة أخرى | مرفوض |
| Company B | `SER-100` | لا يخالف قيد التفرد داخل الشركة |

تبقى صلاحيات المستخدم وقيود الشركات مطلوبة لتنفيذ العمليات نفسها.

---

### التفرد لا يفرض وجود القيمة

`UNIQUE` لا تغني عن `required=True`.

في نمط التفرد المستخدم هنا، يمكن أن تتكرر القيم المخزنة كـSQL `NULL`.

أما النص الخالي، فليس هو SQL `NULL` نفسه.

لذلك حدد بشكل مستقل:

- هل الرقم التسلسلي مطلوب أصلًا؟
- كيف تتعامل مع القيم الفارغة؟
- هل تحتاج إلى تطبيع النصوص قبل مقارنتها؟

في مشروعنا، لا نغيّر تعريف Field إلى مطلوبة على Server في هذا الشابتر؛ نضيف قاعدة التفرد فقط.

---

### لماذا لا يكفي البحث قبل الإنشاء؟

قد تبحث عن الرقم، ولا تجده، ثم تنشئ الجهاز.

لكن طلبًا آخر قد ينفذ الفحص نفسه في الوقت ذاته.

لذلك، البحث المسبق وحده لا يضمن التفرد عند تزامن الطلبات.

SQL Unique Constraint هي التي تفرض هذا الضمان داخل Database.

يمكن إضافة فحص مبكر لتحسين تجربة المستخدم عند الحاجة، مع بقاء القيد هو الحماية الأساسية للتفرد.

---

### الصيغة مرتبطة بالإصدار

قد تجد `_sql_constraints` في أمثلة أقدم.

في هذا المرجع نستخدم صيغة Odoo 20:

```python
models.Constraint
```

كما يجب أن تشير SQL Constraint إلى أعمدة فعلية مناسبة في الجدول؛ لا تتعامل مع Field غير مخزنة وكأنها عمود موجود في Database.

---

## Choosing the Right Validation

### SQL أوPython؟

| طبيعة القاعدة | الاختيار المعتاد |
| --- | --- |
| تفرد قيمة أوتركيبة أعمدة | SQL |
| مقارنة بسيطة وثابتة بين أعمدة الصف | SQL مناسبة غالبًا |
| تحقق يحتاج ORM أوRelationships ومنطقًا أوسع | Python |
| فحص يعتمد على القيمة القديمة والانتقال المطلوب | منطق مناسب في `write()` |
| تحديد المستخدم المخول بالعملية | Security وفحوص العملية |

لا تحتاج إلى تكرار الشرط نفسه في SQL وPython لمجرد استخدام الطريقتين.

اختر التنفيذ الذي يغطي القاعدة بوضوح، وأضف طبقة أخرى عندما يكون لها غرض محدد.

---

### Constraint و`required`

| الأداة | السؤال الذي تجيب عنه |
| --- | --- |
| `required=True` على Model | هل يجب توفير قيمة؟ |
| Constraint | هل القيم تحقق العلاقة أوالشرط المطلوب؟ |

إلزام وجود البداية والنهاية لا يضمن أن النهاية بعد البداية.

وبالعكس، Constraint تتحقق من ترتيب الوقتين عند وجودهما لا تجعل إدخالهما مطلوبًا تلقائيًا.

---

### Constraint وOnchange

Onchange تساعد في إدخال البيانات داخل Form.

أما Constraint، فتتحقق أثناء مسار الحفظ المناسب على Server.

اقتراح نهاية بعد ساعة لا يمنع المستخدم أوIntegration من إرسال نهاية غير صحيحة.

لهذا نحتاج إلى Constraint حتى مع وجود Onchange.

---

### Constraint وSecurity

Constraint تتحقق من صحة البيانات، ولا تمنح المستخدم صلاحية إنشائها أوتعديلها.

كما أن امتلاك المستخدم صلاحية الكتابة لا يعني قبول بيانات تخالف Constraint.

نحتفظ بصلاحيات الشابتر 11، ونضيف فوقها شروط صحة البيانات.

---

## Existing Data & Upgrades

### القاعدة الجديدة لا تصلح البيانات القديمة

عند إضافة Constraint، قد توجد Records مخالفة أُنشئت قبل تعريفها.

مثلًا:

- طلب إعارة نهايته قبل بدايته.
- جهازان بالرقم نفسه داخل الشركة نفسها.

نحتاج إلى مراجعة هذه البيانات، لا مجرد إضافة الكود.

---

### الفرق عند التفعيل

| النوع | ما يجب الانتباه إليه |
| --- | --- |
| Python Constraint | لا تفترض أن إضافة Method تفحص جميع Records القديمة تلقائيًا |
| SQL Constraint | قد تمنع البيانات القديمة إنشاء القيد داخل Database |

في Odoo 20، قد يُسجل فشل إضافة SQL Constraint ويستمر Upgrade.

لذلك اكتمال Upgrade وحده ليس دليلًا كافيًا على وجود القيد.

افحص السجل، وتحقق من القيد نفسه، ثم جرّب رفض بيانات مخالفة.

---

### معالجة المخالفات

حدد التصحيح المناسب بحسب معنى البيانات.

قد تحتاج إلى تصحيح موعد أوتسوية تكرار أوخطة Migration.

لا تحذف Records لمجرد تسهيل تطبيق القيد دون فهم أثرها.

وإذا كانت Field محمية بقواعد العمل التي أضفناها سابقًا، فمعالجة البيانات القديمة قد تحتاج إلى Migration مدروسة، كما سنشرح في [[31 Upgrades & Data Migration]].

---

## Example

سنضيف:

| Model | القاعدة |
| --- | --- |
| `equipment.loan` | النهاية بعد البداية عند توفر الوقتين |
| `equipment.asset` | الرقم التسلسلي فريد داخل الشركة |

لن نضيف Quantity إلى طلب مرتبط بجهاز واحد، ولن نستبدل تعريفات Models السابقة.

---

### 1. مراجعة طلبات الإعارة القديمة

قبل تفعيل القواعد الجديدة، افتح Shell إداريًا على Database التدريب.

ابحث عن الطلبات التي تحتوي الوقتين:

```python
candidate_loans = env["equipment.loan"].search(
    [
        ("start_at", "!=", False),
        ("end_at", "!=", False),
    ]
)

invalid_loans = candidate_loans.filtered(
    lambda loan: loan.end_at <= loan.start_at
)

[(loan.id, loan.name) for loan in invalid_loans]
```

الناتج الفارغ يعني أن هذا الفحص لم يجد مواعيد مخالفة.

إذا ظهرت Records، راجعها وصحح الوقتين وفق البيانات الصحيحة قبل متابعة التحقق.

---

### 2. مراجعة الأرقام التسلسلية المكررة

لفحص التكرار على الجدول كاملًا، بما فيه الأجهزة المؤرشفة وجميع الشركات، نفّذ استعلام القراءة التالي في Shell الإداري:

```python
env.cr.execute(
    """
    SELECT company_id, serial_number, COUNT(*)
    FROM equipment_asset
    WHERE serial_number IS NOT NULL
    GROUP BY company_id, serial_number
    HAVING COUNT(*) > 1
    """
)

env.cr.fetchall()
```

نجمع الأجهزة حسب الشركة والرقم، ثم نعرض المجموعات التي تحتوي أكثر من جهاز.

كل نتيجة تحتوي:

```text
Company ID
Serial Number
Number of Records
```

إذا وجدت تكرارًا، حدد التصحيح المناسب قبل إضافة القيد.

هذا فحص إداري للبيانات الموجودة، وليس مسارًا لحفظ بيانات الأعمال.

---

### 3. إضافة Python Constraint للإعارة

افتح:

```text
models/equipment_loan.py
```

حدّث Import الاستثناءات الموجودة لتشمل:

```python
from odoo.exceptions import UserError, ValidationError
```

احتفظ ببقية Imports وFields وMethods من الشابتر 16.

داخل Class `EquipmentLoan` الموجودة، أضف:

```python
@api.constrains("start_at", "end_at")
def _check_time_order(self):
    for loan in self:
        if (
            loan.start_at
            and loan.end_at
            and loan.end_at <= loan.start_at
        ):
            raise ValidationError(
                _("End time must be after start time.")
            )
```

`api` و`_` مستوردتان مسبقًا في ملف الشابتر السابق.

أبقينا الوقتين اختياريين. هذا التحقق يفرض الترتيب الصحيح عند وجودهما معًا.

---

### 4. إضافة SQL Constraint للأجهزة

افتح:

```text
models/equipment_asset.py
```

داخل Class `EquipmentAsset` الموجودة، أضف التعريف التالي على مستوى Class، مثل تعريفات Fields، وخارج Methods:

```python
_serial_company_unique = models.Constraint(
    "UNIQUE(company_id, serial_number)",
    "The serial number must be unique within the company.",
)
```

`models` مستوردة مسبقًا.

احتفظ بمنطق الحالات وحماية الرقم التسلسلي من الشابتر 15.

هذا القيد لا ينشئ Field جديدة، ولا يحتاج ملف XML أوCSV.

---

### 5. إعادة التشغيل وتطبيق Upgrade

بعد حفظ الملفين:

1. أوقف Server التدريب.
2. شغّله مجددًا لتحميل Python المعدلة.
3. افتح Database `odoo20_training`.
4. افتح **Apps** وابحث عن `Equipment Training`.
5. نفّذ **Upgrade**.
6. راجع Server Log بحثًا عن رسائل تعذر إضافة القيد.

لا توجد تعديلات مطلوبة على Views لهذا التطبيق.

---

### 6. التحقق من وجود SQL Constraint

افتح Shell جديدة بعد Upgrade.

يمكن في تنفيذ Odoo 20 مقارنة تعريف Constraint بما هو موجود في Database:

```python
equipment_model = env["equipment.asset"]

serial_constraint = equipment_model._serial_company_unique

serial_constraint.matches_database(equipment_model)
```

المتوقع:

```python
True
```

إذا كانت النتيجة `False`، فلا تعتبر قاعدة التفرد مفعلة بعد.

راجع بيانات التكرار والسجل وDatabase المستخدمة، ثم عالج السبب وأعد Upgrade.

---

### 7. تجربة الواجهة

باستخدام Equipment Manager:

1. افتح **Equipment → Loans**.
2. أنشئ طلبًا بوقت بداية ونهاية صحيحة.
3. احفظه.
4. عدّل النهاية لتساوي البداية.
5. حاول الحفظ.
6. تحقق من ظهور رسالة رفض ترتيب الوقت.
7. اجعل النهاية بعد البداية، ثم احفظ.

لتجربة التفرد:

1. افتح **Equipment → Assets**.
2. أنشئ جهازًا برقم `CON-017-A`.
3. حاول إنشاء جهاز آخر بالرقم نفسه داخل الشركة نفسها.
4. تحقق من رفض التكرار.

اختر رقمًا تدريبيًا جديدًا إذا سبق استخدام القيمة في Database.

---

### 8. تجهيز فحوص ORM

لفحص المسارات المباشرة، استخدم حساب المدير التدريبي:

```python
manager_user = env["res.users"].search(
    [("login", "=", "equipment.training.manager")],
    limit=1,
)

if not manager_user:
    raise ValueError("Prepare the training manager account first.")

if not manager_user.has_group(
    "equipment_training.group_equipment_manager"
):
    raise ValueError("The training account needs the equipment manager role.")

test_company = manager_user.company_id

equipment_model = (
    env["equipment.asset"]
    .with_user(manager_user)
    .with_context(
        allowed_company_ids=[test_company.id],
        default_state="available",
    )
)

loan_model = (
    env["equipment.loan"]
    .with_user(manager_user)
    .with_context(
        allowed_company_ids=[test_company.id],
    )
)
```

أنشئ جهازين برقمين مختلفين:

```python
assets = equipment_model.create(
    [
        {
            "name": "Constraint Training Laptop",
            "serial_number": "CON-017-SHELL-A",
        },
        {
            "name": "Constraint Training Monitor",
            "serial_number": "CON-017-SHELL-B",
        },
    ]
)

first_asset = assets[0]
second_asset = assets[1]
```

---

### 9. فحص إنشاء طلب صحيح

نفّذ:

```python
loan = loan_model.create(
    {
        "name": "Constraint Training Request",
        "asset_id": first_asset.id,
        "start_at": "2026-02-10 08:00:00",
        "end_at": "2026-02-10 09:00:00",
    }
)

loan.duration_hours
```

المتوقع:

```python
1.0
```

نجح الإنشاء، ثم حُسبت المدة من الوقتين.

---

### 10. فحص رفض الإنشاء والكتابة المخالفة

نفّذ كل فحص خطأ على حدة.

إنشاء طلب بنهاية قبل البداية:

```python
with env.cr.savepoint():
    loan_model.create(
        {
            "name": "Invalid Time Request",
            "asset_id": first_asset.id,
            "start_at": "2026-02-10 08:00:00",
            "end_at": "2026-02-10 07:00:00",
        }
    )
```

المتوقع ظهور `ValidationError`.

ثم حاول تعديل الطلب الصحيح:

```python
with env.cr.savepoint():
    loan.write(
        {
            "end_at": "2026-02-10 08:00:00",
        }
    )
```

المتوقع رفض النهاية المساوية للبداية.

بعد الخطأ:

```python
print(loan.end_at)
```

المتوقع بقاء القيمة:

```text
2026-02-10 09:00:00
```

Savepoint تعزل التغيير المرفوض وتسمح بمتابعة الفحص داخل معاملة Shell.

---

### 11. فحص المسار القادم من Inverse

في الشابتر السابق، جعلنا تحرير المدة يغير النهاية.

جرّب:

```python
with env.cr.savepoint():
    loan.write(
        {
            "duration_hours": 0.0,
        }
    )
```

Inverse ستحاول جعل النهاية مساوية للبداية.

عند كتابة النهاية، تعمل Constraint الخاصة بالوقت وترفض النتيجة.

بهذا نحمي البيانات حتى عندما جاء تغيير النهاية من Method أخرى، وليس من تحرير `end_at` مباشرةً.

---

### 12. فحص التفرد عند الإنشاء

حاول إنشاء جهاز بالرقم نفسه:

```python
with env.cr.savepoint():
    equipment_model.create(
        {
            "name": "Duplicate Serial Device",
            "serial_number": first_asset.serial_number,
            "company_id": test_company.id,
        }
    )
```

المتوقع رفض SQL Constraint للعملية.

داخل Shell قد يظهر Exception من PostgreSQL مثل `UniqueViolation`. أما الواجهة، فتستخدم Odoo آلية تحويل أخطاء Database لعرض رسالة مناسبة.

لذلك لا تفترض أن كل مخالفة SQL ستظهر في Shell كـ`ValidationError` من Python.

---

### 13. فحص التفرد عند الكتابة

حاول إعطاء الجهاز الثاني رقم الجهاز الأول:

```python
with env.cr.savepoint():
    second_asset.write(
        {
            "serial_number": first_asset.serial_number,
        }
    )

    second_asset.flush_recordset(["serial_number"])
```

المتوقع رفض التكرار.

استخدمنا `flush_recordset()` لإرسال الكتابة المعلقة إلى Database داخل نطاق الفحص.

Flush ليست Commit؛ لا تثبت المعاملة.

بعد رفض العملية، يبقى الرقم السابق للجهاز الثاني.

---

### 14. مراجعة النتائج

| الحالة | النتيجة |
| --- | --- |
| البداية والنهاية موجودتان، والنهاية لاحقة | مسموح |
| النهاية مساوية للبداية | مرفوض |
| النهاية قبل البداية | مرفوض |
| أحد الوقتين غير موجود | مسموح في التصميم الحالي |
| تعديل المدة بما يجعل النهاية غير صحيحة | مرفوض عند كتابة النهاية |
| تكرار الرقم داخل الشركة نفسها | مرفوض |
| نقل جهاز إلى شركة تحتوي الرقم نفسه | يخضع لقيد التفرد أيضًا |
| تكرار الرقم مع جهاز مؤرشف في الشركة نفسها | مرفوض |
| الرقم نفسه في شركة أخرى | لا يخالف نطاق التفرد الذي عرّفناه |

الأجهزة المؤرشفة ما زالت صفوفًا في الجدول؛ القيد لا يستثنيها.

فحوص Shell هنا للتعلم، وتبقى بياناتها ضمن معاملة Shell.

---

## Important Notes

- Python Constraint ترفض المخالفة برفع Exception؛ `return False` ليست بديلًا.
- `@api.constrains` لا تتعامل مع Dot Notation مثل `@api.depends`.
- Constraint اختيارية التوقيت لا تجعل Fields مطلوبة تلقائيًا.
- لا تعتمد على Search مسبقة وحدها لضمان التفرد مع الطلبات المتزامنة.
- اكتمال Upgrade لا يكفي لإثبات إنشاء SQL Constraint؛ تحقق من القيد والسلوك.
- لا تفترض أن Python Constraint الجديدة راجعت جميع البيانات القديمة تلقائيًا.
- SQL Constraint تعمل على البيانات المخزنة، بما فيها Records التي تخفيها فلاتر الواجهة أوالأرشفة.
- عند التقاط أخطاء الفحص لمتابعة العمل، استخدم Savepoint مناسبة؛ التقاط Exception وحده لا يعيد المعاملة إلى حالتها السابقة.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[18 Inheritance & Module Interaction]]

---

## Sources

- [Odoo 20 Tutorial — Constraints](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/10_constraints.html)
- [Odoo 20 Documentation — ORM API: Constraints and Indexes](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#constraints-and-indexes)
- [Odoo 20 Source — Constraint Definitions](https://github.com/odoo/odoo/blob/20.0/odoo/orm/table_objects.py)
- [Odoo 20 Source — Constrains Decorator](https://github.com/odoo/odoo/blob/20.0/odoo/orm/decorators.py)
- [Odoo 20 Source — ORM Validation and SQL Error Messages](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)
- [Odoo 20 Source — Constraint Application During Upgrades](https://github.com/odoo/odoo/blob/20.0/odoo/orm/registry.py)
- [Odoo 20 Source — Savepoints](https://github.com/odoo/odoo/blob/20.0/odoo/sql_db.py)