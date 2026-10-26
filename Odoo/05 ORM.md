---
type: learning
odoo_version: "20.0"
status: complete
created: 2026-10-13
reviewed: 2026-10-13
---
# 05 ORM

## Table of Contents

- [[#Introduction|Introduction]]
- [[#ORM Fundamentals|ORM Fundamentals]]
- [[#Working with Records|Working with Records]]
- [[#ORM وOdoo Framework|ORM وOdoo Framework]]
- [[#Transactions وDirect SQL|Transactions وDirect SQL]]
- [[#Example|Example]]
- [[#اختيار ORM Method المناسبة|اختيار ORM Method المناسبة]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في [[04 Models & Fields]] تعلمنا تمثيل Business Objects باستخدام Models وFields. الخطوة التالية هي التعامل مع بيانات هذه Models: إنشاء Records، والبحث عنها، وقراءتها، وتعديلها، وحذفها.

الواجهة الأساسية لتنفيذ هذه العمليات من Python هي Odoo ORM.

في هذا الفصل سنتعلم الوصول إلى Models واستخدام أهم ORM Methods، ثم نفهم علاقتها بسلوك Odoo Framework وTransactions، ونطبق العمليات داخل Odoo Shell.

---

## ORM Fundamentals

### ما هي ORM؟

ORM اختصار لـ**Object-Relational Mapping**.

تسمح لنا بالتعامل مع البيانات باستخدام Models وPython APIs، بدل كتابة SQL مباشرة لكل عملية.

المسار العام:

```text
Python Code
    ↓
Odoo ORM
    ↓
PostgreSQL
```

نحدد العملية المطلوبة باستخدام مفاهيم Odoo، مثل Models وRecords وFields، وتتولى ORM تنفيذ التعامل مع Database ضمن Odoo Framework.

---

### CRUD

ستقابل كثيرًا مصطلح **CRUD**، وهو اختصار لأربع عمليات أساسية:

| الحرف | العملية | معناها |
|---|---|---|
| C | Create | إنشاء بيانات جديدة |
| R | Read | قراءة البيانات |
| U | Update | تعديل البيانات |
| D | Delete | حذف البيانات |

توفر ORM Methods لتنفيذ هذه العمليات، بالإضافة إلى البحث والعد والوصول إلى Records باستخدام IDs معروفة.

الهدف هو اختيار Method التي تناسب العملية المطلوبة، وليس حفظ أسماء Methods فقط.

---

### لماذا نستخدم ORM؟

التعامل مع بيانات Odoo يشمل سلوك Framework، مثل الصلاحيات والقيم الافتراضية والمنطق المرتبط بالـModels.

استخدام ORM يسمح للعملية بالتفاعل مع هذا السلوك، بدل التعامل مع Database بمعزل عنه.

لذلك ستكون القاعدة العامة:

**ORM أولًا، وDirect SQL عند وجود سبب واضح.**

سنوضح علاقتها ببقية Framework لاحقًا في هذا الفصل.

---

### أين نكتب ORM Code؟

سنستخدم سياقين رئيسيين:

| السياق | الوصول إلى Environment |
|---|---|
| Odoo Shell | `env` |
| داخل Model Method | `self.env` |

#### داخل Odoo Shell

Odoo Shell توفر جلسة Python مرتبطة بـOdoo وبـDatabase محددة.

نستخدمها لتجربة العمليات وفحص البيانات مباشرة.

إذا كنت تستخدم بيئة التطوير التي أعددناها في [[02 Development Environment]]، شغّل الأمر من مجلد المشروع الرئيسي.

##### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin shell \
  -c config/odoo.conf \
  -d odoo20_training
```

##### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin shell `
  -c config/odoo.conf `
  -d odoo20_training
```

استخدم اسم Database وإعدادات المشروع المناسبة لبيئتك.

بعد الدخول إلى Shell، توفر Odoo متغيرًا باسم `env` يمثل Environment الحالية، ويمكننا من خلاله الوصول إلى Models.

سننفذ أمثلة هذا الفصل داخل Database تجريبية.

#### داخل Model Methods

عندما نكتب Method داخل Class الخاصة بـModel، نصل إلى Environment باستخدام:

```python
self.env
```

هذه Methods تُكتب داخل ملفات Models، مثل `models/equipment.py`، داخل Class التي عرّفناها في [[04 Models & Fields]].

يكفينا الآن معرفة الفرق:

```text
Odoo Shell
→ env

داخل Model Method
→ self.env
```

Environment تربط العمليات بسياق العمل الحالي، بما فيه Database والمستخدم. تفاصيلها تأتي في [[09 Environment & Context]].

---

### الوصول إلى Model

لنفترض أن Model باسم `equipment.asset` محمّلة في Database.

داخل Shell:

```python
equipment_model = env["equipment.asset"]
```

وداخل Model Method:

```python
equipment_model = self.env["equipment.asset"]
```

الأجزاء هي:

| الجزء | معناه |
|---|---|
| `env` أو`self.env` | Odoo Environment الحالية |
| `"equipment.asset"` | Technical Model Name |
| `equipment_model` | Python Variable اخترنا اسمها للإشارة إلى Model |

اسم `equipment_model` ليس Keyword خاصة بـOdoo. يمكن اختيار اسم آخر، لكننا سنستخدمه باستمرار لتوضيح المقصود.

هذا السطر يتيح الوصول إلى Model، لكنه لا ينفذ Search ولا يقرأ جميع Records.

---

### ماذا تعيد ORM؟

كثير من ORM Methods تتعامل مع **Recordsets**.

Recordset قد تمثل:

| العدد | المعنى |
|---|---|
| صفر | لا توجد Records في النتيجة |
| واحدة | النتيجة تحتوي Record واحدة |
| عدة | النتيجة تحتوي عدة Records |

لكن ليست كل ORM Methods تعيد Recordsets؛ بعضها يعيد عددًا أوData Structure، وسنوضح ذلك عند شرح كل Method.

سنستخدم غالبًا:

- `asset` عندما نتوقع Record واحدة.
- `assets` عندما نتوقع عدة Records.

هذه Naming Convention تساعد على القراءة، ولا تحدد نوع النتيجة أوتضمن عدد Records.

الفهم التفصيلي لسلوك Recordsets يأتي في [[06 Records & Recordsets]].

---

## Working with Records

سنستخدم Model باسم `equipment.asset`، مثل التي عرّفناها في Chapter 04، وتحتوي:

| Field | معناها |
|---|---|
| `name` | اسم المعدة |
| `serial_number` | الرقم التسلسلي |
| `state` | الحالة، مثل `available` أو`in_use` |
| `active` | تحديد ما إذا كانت Record نشطة أم مؤرشفة |

نفترض أن القيمة الافتراضية لـ`state` هي `available`، ولـ`active` هي `True`، كما في تعريف Model السابق.

الأمثلة التالية توضح كل عملية، ثم سنجمعها في خطوات مترابطة داخل Example.

---

### إنشاء Records

#### `create()`

تستخدم `create()` لإنشاء Record جديدة، ونمرر لها Dictionary تحتوي أسماء Fields وقيمها:

```python
asset = equipment_model.create({
    "name": "Office Laptop",
    "serial_number": "LT-100",
})
```

كل عنصر داخل Dictionary يمثل:

```text
Field Name → Value
```

في المثال:

| Field | Value |
|---|---|
| `name` | `Office Laptop` |
| `serial_number` | `LT-100` |

لم نمرر `state` و`active` لأن Model توفر لهما قيمًا افتراضية.

`create()` تعيد Recordset تمثل Record التي أنشأناها، وقد احتفظنا بها داخل `asset`.

---

### البحث عن Records

#### `search()`

تستخدم `search()` للبحث عن Records داخل Model.

تستقبل **Domain**، وهي قائمة تحدد شروط البحث.

عندما نمرر قائمة فارغة، لا نضيف شروط تصفية إلى Search:

```python
assets = equipment_model.search([])
```

النتيجة Recordset.

وجود Domain فارغة لا يعني تجاوز الصلاحيات أوإلغاء السلوك الآخر المطبق على البحث، مثل التعامل المعتاد مع Archived Records.

> [!warning] ضيّق البحث حسب الحاجة
> لا تجعل `search([])` خيارك التلقائي مع Models كبيرة. إذا كنت تحتاج Records محددة، ضع شروط البحث المناسبة من البداية.

#### البحث باستخدام Domain

للبحث عن Equipment التي حالتها `available`، نستخدم شرطًا يتكون من:

| الجزء | قيمته في المثال |
|---|---|
| Field | `state` |
| Operator | `=` |
| Value | `available` |

نكتب الشرط داخل Domain:

```python
available_assets = equipment_model.search([
    ("state", "=", "available"),
])
```

المعنى: ابحث عن Records التي تساوي قيمة `state` لديها `available`.

هذا القدر يكفينا الآن لاستخدام ORM. تفاصيل Domains وOperators تأتي في [[08 Domains]].

#### `limit`

تحدد الحد الأقصى لعدد Records التي تعيدها Search:

```python
assets = equipment_model.search(
    [],
    limit=10,
)
```

النتيجة تحتوي حتى عشر Records.

وللبحث عن Record واحدة كحد أقصى:

```python
asset = equipment_model.search(
    [
        ("serial_number", "=", "LT-100"),
    ],
    limit=1,
)
```

`limit=1` لا تضمن وجود نتيجة، ولا تثبت أن قيمة `serial_number` فريدة. إذا طابقت عدة Records الشرط، ستُعاد واحدة منها فقط.

#### `order`

تحدد ترتيب نتائج Search:

```python
assets = equipment_model.search(
    [],
    order="name, id",
)
```

تُرتب النتائج أولًا حسب `name`، وإذا تساوى الاسم بين أكثر من Record يكون الترتيب حسب `id`.

#### `offset`

تحدد عدد النتائج التي نتخطاها قبل إرجاع Records:

```python
assets = equipment_model.search(
    [],
    order="name, id",
    offset=20,
    limit=10,
)
```

المعنى:

```text
ترتيب النتائج حسب name ثم id
            ↓
تخطي أول 20 نتيجة
            ↓
إرجاع حتى 10 نتائج بعدها
```

يمكن استخدامها في بعض سيناريوهات Pagination. التعامل مع مجموعات البيانات الكبيرة سيأتي في [[28 Performance]].

#### `search_count()`

تستخدم لمعرفة عدد Records المطابقة لـDomain:

```python
count = equipment_model.search_count([
    ("state", "=", "available"),
])
```

قيمة `count` هي عدد Equipment Records التي حالتها `available`.

استخدمها عندما تحتاج العدد فقط، دون الحاجة إلى الحصول على Records نفسها.

---

### الوصول باستخدام ID معروف

#### `browse()`

إذا كانت لديك ID معروفة، يمكنك الحصول على Recordset تمثلها باستخدام `browse()`:

```python
asset = equipment_model.browse(42)
```

الرقم `42` هنا مثال على Record ID.

لكن وجود ID داخل Recordset لا يثبت أن Record موجودة فعليًا في Database.

#### `exists()`

إذا كان التحقق من الوجود مهمًا:

```python
asset = equipment_model.browse(42).exists()
```

ثم تحقق من النتيجة:

```python
if asset:
    print("Record exists")
```

إذا لم تكن Record موجودة، ستكون النتيجة فارغة.

الفرق الأساسي:

| Method | متى نستخدمها؟ |
|---|---|
| `search()` | عندما نبحث باستخدام شروط |
| `browse()` | عندما نملك IDs بالفعل |
| `exists()` | للاحتفاظ بالـRecords الموجودة فعليًا |

---

### قراءة البيانات

#### الوصول المباشر إلى Fields

إذا كانت لدينا Record واحدة، يمكن قراءة Fields مباشرة:

```python
asset.name
asset.serial_number
asset.state
```

مثلًا، بعد Search قد تعيد صفرًا أو نتيجة واحدة:

```python
asset = equipment_model.search(
    [
        ("serial_number", "=", "LT-100"),
    ],
    limit=1,
)

if asset:
    print(asset.name)
```

هذا الأسلوب شائع داخل Business Logic.

سلوك قراءة Fields عند وجود عدة Records سيُشرح في [[06 Records & Recordsets]].

#### `read()`

تستخدم `read()` لقراءة Fields من Recordset والحصول على البيانات كـList من Dictionaries:

```python
rows = assets.read([
    "name",
    "serial_number",
])
```

قد تكون النتيجة:

```python
[
    {
        "id": 1,
        "name": "Office Laptop",
        "serial_number": "LT-100",
    },
    {
        "id": 2,
        "name": "Office Monitor",
        "serial_number": "MN-200",
    },
]
```

تتضمن النتيجة `id` الخاصة بكل Record، إلى جانب Fields المطلوبة.

`rows` هنا List من Dictionaries، وليست Recordset.

إذا كنت تحتاج متابعة العمل على Records باستخدام ORM، احتفظ بالـRecordset. واستخدم `read()` عندما تكون هذه Data Structure هي ما تحتاجه.

#### `search_read()`

تجمع `search_read()` البحث وقراءة Fields في استدعاء واحد:

```python
rows = equipment_model.search_read(
    [
        ("state", "=", "available"),
    ],
    fields=[
        "name",
        "serial_number",
    ],
    limit=10,
)
```

النتيجة List من Dictionaries.

| العملية | نوع النتيجة |
|---|---|
| `search()` | Recordset |
| `read()` | List of Dictionaries |
| `search_read()` | List of Dictionaries |

---

### تعديل Records

#### `write()`

تستخدم `write()` لتعديل قيم Fields في Records موجودة:

```python
asset.write({
    "state": "in_use",
})
```

كما في `create()`، نمرر Dictionary تحتوي أسماء Fields وقيمها الجديدة.

`write()` تطبق التغيير على **جميع Records داخل Recordset**.

مثلًا:

```python
assets.write({
    "active": False,
})
```

إذا احتوت `assets` خمس Records، فسيُطبق التغيير على الخمس جميعًا.

لهذا يجب معرفة ما تحتويه Recordset قبل تعديلها. سنتوسع في هذا السلوك في
[[06 Records & Recordsets]].

---

### حذف Records وأرشفتها

#### `unlink()`

تستخدم `unlink()` لحذف Records فعليًا:

```python
asset.unlink()
```

تطبق العملية على جميع Records داخل Recordset، وفق الصلاحيات والقيود التي تنطبق على الحذف.

قبل استخدامها، تأكد أن حذف البيانات هو السلوك المطلوب فعلًا.

#### Delete أم Archive؟

إذا خرجت Equipment من الاستخدام، فقد نحتاج الاحتفاظ ببياناتها بدل حذفها.

في Model تحتوي Field باسم `active`، مثل Model المثال، يمكن أرشفة Record باستخدام:

```python
asset.write({
    "active": False,
})
```

| العملية | النتيجة |
|---|---|
| Delete | حذف Record |
| Archive | الاحتفاظ بالـRecord وبياناتها مع جعلها غير نشطة |

Archived Records لا تظهر عادةً في نتائج البحث المعتادة، لكن تفاصيل هذا السلوك وتأثير Context ستأتي في [[09 Environment & Context]].

الاختيار يعتمد على متطلبات العمل، وليس على أي Method أسهل في الاستخدام.

---

## ORM وOdoo Framework

ORM جزء من Odoo Framework، وعملياتها تتفاعل مع سلوك Models والأنظمة المحيطة بالبيانات.

نحتاج الآن إلى فهم هذا الارتباط بالقدر الذي يساعدنا على اختيار طريقة التعامل مع البيانات.

### ORM وSecurity

عند تنفيذ:

```python
assets = equipment_model.search([])
```

لا نطلب من ORM تجاهل صلاحيات المستخدم وإعادة جميع Rows الموجودة في PostgreSQL.

الوصول إلى البيانات يعتمد على Environment والصلاحيات المطبقة على العملية.

كذلك Domain تحدد البيانات المطلوبة، لكنها لا تمنح Permissions إضافية.

تفاصيل Security تأتي في [[11 Security & Multi-Company]].

---

### `sudo()`

`sudo()` تعيد نسخة من Recordset مع تفعيل **Superuser Mode**، بما يسمح بتجاوز فحوص صلاحيات الوصول.

هي لا تغيّر المستخدم الحالي.

قد ترى استخدامها بهذا الشكل:

```python
elevated_model = equipment_model.sudo()

assets = elevated_model.search(
    [],
    limit=10,
)
```

> [!warning] `sudo()` ليست حلًا تلقائيًا لـAccessError
> قبل استخدامها، افهم سبب رفض الوصول، وهل يجب السماح بالعملية أصلًا، وما البيانات التي قد تصبح متاحة نتيجة تجاوز الصلاحيات.

Odoo Shell الافتراضية تبدأ بصلاحيات Superuser. لذلك نجاح عملية داخل Shell لا يثبت أن المستخدم العادي يستطيع تنفيذها.

سنشرح استخدام `sudo()` وضوابطها بالتفصيل في
[[11 Security & Multi-Company]].

---

### ORM وRelationships

ORM تتعامل أيضًا مع العلاقات بين Models.

فبعض Fields قد تربط Record بـRecords موجودة في Models أخرى.

لا نحتاج معرفة أنواع العلاقات أوطريقة تعديلها الآن؛ سنشرحها في
[[07 Relationships]].

---

### ORM وComputed Fields

بعض Fields تحمل قيمًا تُحسب تلقائيًا داخل Odoo.

عمليات ORM تتفاعل مع هذا السلوك، بينما التعامل المباشر مع Database قد يتجاوز جزءًا منه.

تفاصيل Computed Fields تأتي في
[[16 Computed Fields, Related Fields & Onchange]].

---

## Transactions وDirect SQL

### ما هي Transaction؟

Transaction تجمع مجموعة من عمليات Database ضمن وحدة واحدة.

داخل طلب Odoo المعتاد، يمكن أن تُنفذ عدة ORM Operations ضمن Transaction واحدة. كل استدعاء لـ`create()` أو`write()` لا يعني Commit مستقلة.

المصطلحات الأساسية:

| المصطلح | معناه |
|---|---|
| Transaction | وحدة تضم عمليات Database المرتبطة |
| Commit | تثبيت التغييرات في Database |
| Rollback | التراجع عن التغييرات غير المثبتة |

في المسارات المعتادة، يدير Odoo Framework هذه العملية:

```text
بدء الطلب
    ↓
تنفيذ العمليات
    ↓
اكتمال التنفيذ بنجاح
    ↓
Commit
```

وإذا فشلت العملية ووصلت Exception إلى Framework دون معالجتها داخل الكود:

```text
بدء الطلب
    ↓
تنفيذ العمليات
    ↓
Exception غير معالجة
    ↓
Rollback
```

هذا يدعم **Atomicity**: تُثبت تغييرات Database المشمولة بالـTransaction كوحدة واحدة، أو يُتراجع عنها.

مثلًا:

```text
Create Record A
        ↓
Update Record B
        ↓
Create Record C
```

إذا فشلت العملية الأخيرة، يمكن التراجع عن التغييرات السابقة ضمن Transaction نفسها بدل الاحتفاظ بعملية مكتملة جزئيًا.

---

### استخدام `commit()` يدويًا

داخل Environment يوجد **Database Cursor** مرتبط بالـTransaction الحالية.

داخل Model Method نصل إليه باستخدام:

```python
self.env.cr
```

ومن خلاله يمكن استدعاء `commit()` لتثبيت التغييرات الحالية:

```python
self.env.cr.commit()
```

لكن في Business Methods العادية، Framework مسؤولة عن إدارة Transaction.

تنفيذ Commit يدويًا في منتصف العملية قد يثبت جزءًا منها قبل اكتمال بقية الأجزاء:

```text
Create Record A
        ↓
commit()
        ↓
Update Record B
        ↓
Error
```

هنا أصبحت Record A مثبتة، بينما فشل الجزء اللاحق. Rollback بعد الخطأ لن تلغي التغييرات التي ثُبتت بالـCommit السابقة.

> [!warning] لا تستخدم Manual Commit لمجرد ضمان الحفظ
> داخل Business Methods العادية، دع Odoo Framework تدير Transaction.
>
> إدارة Cursor مستقلة وTransactions يدويًا لها حالات متقدمة، ولا نحتاجها لتطبيق عمليات ORM المعتادة.

---

### Transactions داخل Odoo Shell

في Shell نصل إلى Cursor باستخدام:

```python
env.cr
```

للتراجع عن التغييرات غير المثبتة في Transaction الحالية:

```python
env.cr.rollback()
```

وإذا كنت تقصد الاحتفاظ بالتغييرات داخل Training Database:

```python
env.cr.commit()
```

Odoo Shell تتراجع افتراضيًا عن التغييرات غير المثبتة عند الخروج منها.

انتبه إلى نقطتين:

- `rollback()` تتراجع عن التغييرات منذ آخر Commit ضمن Transaction الحالية، وليس عن آخر ORM Method فقط.
- استخدام `commit()` أثناء تجربة Shell لا يعني نقله إلى Business Methods داخل Modules.

---

### Direct SQL

يمكن استخدام SQL مباشرة داخل Odoo، لكنها ليست الاختيار الأول لمعظم Business Logic.

التعامل المباشر مع Database قد يتجاوز فحوص الصلاحيات وسلوك Methods المرتبطة بالـModels، وقد يحتاج إلى معالجة اتساق Cache والقيم المحسوبة بشكل صريح.

لذلك نبدأ بـORM، ونستخدم Direct SQL عندما يوجد سبب واضح وفهم لتأثير العملية على Framework.

لن ندخل هنا في Syntax تنفيذ SQL أوتفاصيل تحسين الأداء. سنناقش هذه الجوانب في
[[28 Performance]].

---

## Example

سنطبق العمليات باستخدام Model باسم `equipment.asset`، مثل التي عرّفناها في
[[04 Models & Fields]].

نفترض أنها محمّلة في Database التجريبية وتحتوي:

| Field | الاستخدام في المثال |
|---|---|
| `name` | اسم المعدة |
| `serial_number` | رقم تسلسلي للتجربة |
| `state` | حالتها، وقيمتها الافتراضية `available` |
| `active` | حالتها النشطة، وقيمتها الافتراضية `True` |

---

### 1. بدء Odoo Shell

شغّل Shell باستخدام الأوامر الموضحة في قسم ORM Fundamentals، ثم نفّذ الخطوات التالية بالتتابع.

بعد الدخول، سيكون `env` متاحًا.

---

### 2. الوصول إلى Model

```python
equipment_model = env["equipment.asset"]
```

وصلنا إلى Model، ولم ننفذ Search بعد.

---

### 3. إنشاء Records وقراءة بياناتها

```python
laptop = equipment_model.create({
    "name": "Office Laptop",
    "serial_number": "LT-100",
})

monitor = equipment_model.create({
    "name": "Office Monitor",
    "serial_number": "MN-200",
})
```

كل Variable تحتوي Recordset تمثل Record التي أُنشئت.

افحص بيانات الـLaptop:

```python
laptop.id
laptop.name
laptop.serial_number
laptop.state
```

ستكون قيمة `state`:

```text
available
```

لأن Model توفر هذه القيمة افتراضيًا.

---

### 4. البحث عن Records

```python
assets = equipment_model.search(
    [],
    order="name, id",
    limit=10,
)
```

قد تتضمن النتيجة Records أنشأناها الآن وRecords موجودة مسبقًا في Database.

للبحث عن المعدات المتاحة:

```python
available_assets = equipment_model.search([
    ("state", "=", "available"),
])
```

---

### 5. البحث باستخدام Serial Number

```python
asset = equipment_model.search(
    [
        ("serial_number", "=", "LT-100"),
    ],
    limit=1,
)
```

تحقق من وجود نتيجة:

```python
if asset:
    print(asset.name)
```

إذا كانت عدة Records تحمل الرقم نفسه، ستُعاد واحدة منها فقط. تعريف `serial_number` في مثالنا لا يفرض أن تكون القيمة فريدة.

سنستخدم نتيجة هذا البحث للقراءة، ونعدل Record التي أنشأناها من خلال متغيرها المباشر في الخطوة التالية.

---

### 6. تعديل Record محددة

لتعديل الـLaptop التي أنشأناها:

```python
laptop.write({
    "state": "in_use",
})
```

تحقق:

```python
laptop.state
```

النتيجة:

```text
in_use
```

استخدام `laptop` هنا يحدد Record التي أنشأناها، حتى لو وجدت Records أخرى تحمل Serial Number نفسها.

---

### 7. حساب Records المطابقة

```python
count = equipment_model.search_count([
    ("state", "=", "available"),
])
```

اعرض العدد:

```python
count
```

القيمة تعتمد على جميع Records المطابقة في Database، وليس بيانات المثال فقط.

---

### 8. قراءة Fields كـDictionaries

نستخدم Recordset التي حصلنا عليها في خطوة البحث:

```python
rows = assets.read([
    "name",
    "serial_number",
    "state",
])
```

اعرض النتيجة:

```python
rows
```

ستكون List من Dictionaries.

---

### 9. البحث والقراءة معًا

```python
rows = equipment_model.search_read(
    [
        ("state", "=", "available"),
    ],
    fields=[
        "name",
        "serial_number",
        "state",
    ],
    limit=5,
)
```

اعرض:

```python
rows
```

النتيجة أيضًا List من Dictionaries، لكننا نفذنا البحث والقراءة في استدعاء واحد.

---

### 10. الوصول باستخدام ID معروفة

لدينا ID الخاصة بالـLaptop:

```python
laptop.id
```

يمكن استخدامها مع `browse()`:

```python
same_laptop = equipment_model.browse(laptop.id)
```

ثم:

```python
same_laptop.name
```

وللتحقق من الوجود:

```python
same_laptop = same_laptop.exists()

if same_laptop:
    print(same_laptop.name)
```

---

### 11. Archive

نؤرشف الـMonitor بدل حذفها:

```python
monitor.write({
    "active": False,
})
```

تحقق:

```python
monitor.active
```

النتيجة:

```python
False
```

Record ما زالت موجودة، لكنها أصبحت غير نشطة.

سلوك `active` سبق تقديمه في [[04 Models & Fields]]، وتأثير Context على ظهور Archived Records سيأتي في [[09 Environment & Context]].

---

### 12. تجربة الحذف

ننشئ Record عادية مخصصة لتجربة الحذف:

```python
test_asset = equipment_model.create({
    "name": "Deletion Test Asset",
})
```

ثم نحذفها:

```python
test_asset.unlink()
```

تحقق:

```python
if not test_asset.exists():
    print("Record deleted")
```

النتيجة:

```text
Record deleted
```

---

### 13. إنهاء التجربة

إذا كنت لا تريد الاحتفاظ بتغييرات التجربة:

```python
env.cr.rollback()
```

هذا يتراجع عن التغييرات غير المثبتة في Transaction الحالية.

أما إذا كنت تقصد الاحتفاظ بها داخل Training Database:

```python
env.cr.commit()
```

اختر الإجراء المناسب لهدف التجربة. ولا تنقل استخدام Manual Commit إلى Business Methods العادية.

---

## اختيار ORM Method المناسبة

ابدأ بالسؤال: **ما الذي أحتاج فعله بالبيانات؟**

| المطلوب | الأداة المناسبة | النتيجة أوالأثر |
|---|---|---|
| إنشاء Record | `create()` | Recordset تمثل البيانات المنشأة |
| البحث باستخدام شروط | `search()` | Recordset |
| الوصول باستخدام IDs معروفة | `browse()` | Recordset تمثل IDs |
| الاحتفاظ بالـRecords الموجودة | `exists()` | Recordset |
| معرفة العدد فقط | `search_count()` | عدد |
| قراءة Fields من Recordset كبيانات | `read()` | List of Dictionaries |
| البحث وقراءة Fields معًا | `search_read()` | List of Dictionaries |
| تعديل Records موجودة | `write()` | تطبيق القيم على المجموعة |
| حذف Records فعليًا | `unlink()` | حذف المجموعة |
| أرشفة Records في Model تحتوي `active` | `write({"active": False})` | الاحتفاظ بالبيانات وجعلها غير نشطة |

عند البحث، ضيّق Domain بما يناسب المطلوب، وحدد `limit` إذا كنت تحتاج عددًا محدودًا.

وعند التعديل أوالحذف، تأكد من Records التي تحتويها المجموعة قبل تنفيذ العملية.

---

## Important Notes

- `env["model.name"]` و`self.env["model.name"]` تتيحان الوصول إلى Model؛ لا تنفذان Search لجميع Records.

- `browse()` لا تثبت أن Records موجودة في Database. استخدم `exists()` عندما يكون التحقق من الوجود مهمًا.

- `limit=1` لا تضمن وجود نتيجة، ولا تجعل شرط البحث فريدًا.

- `write()` و`unlink()` تطبقان على جميع Records داخل Recordset؛ تأكد من المجموعة قبل التعديل أوالحذف.

- Domain لا تمنح Permissions إضافية. ونجاح العملية في Shell الافتراضية لا يثبت صلاحيات المستخدم العادي؛ لأنها تبدأ بصلاحيات Superuser.

- لا تستخدم `sudo()` تلقائيًا لمعالجة `AccessError`؛ تجاوز الصلاحيات يجب أن يكون مقصودًا ومفهومًا.

- Archive تحافظ على Record وبياناتها، بينما `unlink()` تحذفها؛ اختر وفق متطلبات العمل.

- Manual Commit قد يثبت جزءًا من العملية ويكسر Atomicity. وRollback لا تلغي التغييرات التي ثُبتت بـCommit سابقة.

- Direct SQL قد تتجاوز سلوك ORM والصلاحيات وتتطلب معالجة اتساق البيانات وCache؛ لا تجعلها الاختيار الافتراضي.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[06 Records & Recordsets]]

---

## Sources

- [Odoo 20 Documentation — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Documentation — Command-line Interface](https://www.odoo.com/documentation/20.0/developer/reference/cli.html)
- [Odoo 20 Documentation — Coding Guidelines: Never Commit the Transaction](https://www.odoo.com/documentation/20.0/contributing/development/coding_guidelines.html#never-commit-the-transaction)
- [Odoo 20 Source Code — Models and ORM](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)
- [Odoo 20 Source Code — Odoo Shell](https://github.com/odoo/odoo/blob/20.0/odoo/cli/shell.py)