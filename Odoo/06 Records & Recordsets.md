---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 06 Records & Recordsets

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Recordset Fundamentals|Recordset Fundamentals]]
- [[#Working with Recordsets|Working with Recordsets]]
- [[#Designing Methods for Recordsets|Designing Methods for Recordsets]]
- [[#Example|Example]]
- [[#اختيار الأسلوب المناسب|اختيار الأسلوب المناسب]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في [[05 ORM]] تعلمنا استخدام ORM لإنشاء Records والبحث عنها وقراءتها وتعديلها. وتعرفنا بشكل تمهيدي على Recordsets التي تعيدها كثير من هذه العمليات.

نحتاج الآن إلى فهم سلوكها: ماذا يحدث عندما تكون المجموعة فارغة؟ ومتى نستطيع قراءة Field مباشرة؟ ولماذا يظهر خطأ `Expected singleton`؟

في هذا الفصل سنتعلم التعامل مع Record واحدة أو عدة Records، واختيار أدوات التصفية وجمع القيم والترتيب، ثم تصميم Methods تدعم العدد المناسب من Records.

---

## Recordset Fundamentals

### Record وRecordset

Recordset هي مجموعة مرتبة من Records تنتمي إلى Model واحدة.

قد تحتوي صفر Records، أو Record واحدة، أو عدة Records. وحتى Record الواحدة نتعامل معها تقنيًا من خلال Recordset تحتوي عنصرًا واحدًا.

لنفترض أن لدينا Model باسم `equipment.asset`.

قد تظهر Recordset في Odoo Shell بهذا الشكل:

```text
equipment.asset(5)
```

هذا يعني أنها تحتوي Record واحدة، والرقم `5` هو ID الخاصة بها.

أما:

```text
equipment.asset(1, 5, 8)
```

فتمثل مجموعة تحتوي ثلاث Records من Model نفسها.

هذه الصيغة هي طريقة عرض Recordset في Shell. للوصول إلى Records أو إنشائها نستخدم ORM Methods التي تعلمناها سابقًا.

### أحجام Recordset

| الحالة | عدد Records | مثال على شكل العرض |
|---|---|---|
| Empty Recordset | صفر | `equipment.asset()` |
| Singleton | واحدة | `equipment.asset(5)` |
| Multi-record Recordset | أكثر من واحدة | `equipment.asset(1, 5, 8)` |

معرفة حجم المجموعة مهمة؛ بعض العمليات تدعم عدة Records، بينما تحتاج عمليات أخرى إلى Record واحدة بالضبط.

---

### أمثلة الوصول إلى Records

سنطبق الأمثلة التالية داخل Odoo Shell، ونفترض أن Model باسم `equipment.asset` محمّلة في Database وتحتوي Fields مثل `name` و`state`.

للوصول إلى Model:

```python
equipment_model = env["equipment.asset"]
```

أما أمثلة Model Methods التي ستظهر لاحقًا، فتُكتب داخل Class الخاصة بالـModel في ملف Python، كما تعلمنا في [[04 Models & Fields]].

#### Empty Recordset

إذا بحثنا عن اسم غير موجود:

```python
assets = equipment_model.search([
    ("name", "=", "Does Not Exist"),
])
```

تكون النتيجة:

```text
equipment.asset()
```

أي Recordset فارغة.

يمكن التحقق منها:

```python
if not assets:
    print("No records found")
```

#### Singleton

إذا احتوت النتيجة Record واحدة، نسميها Singleton.

مثلًا:

```python
asset = equipment_model.search(
    [
        ("name", "=", "Laptop"),
    ],
    limit=1,
)
```

إذا وُجدت نتيجة، فقد تظهر بهذا الشكل:

```text
equipment.asset(5)
```

النتيجة ما زالت Recordset، لكن حجمها واحدة.

> [!warning] `limit=1` لا تضمن وجود Record
> هي تحدد الحد الأقصى للنتائج. إذا لم توجد Record مطابقة، ستعيد Search مجموعة فارغة.

#### Multi-record Recordset

إذا أعادت Search أكثر من Record:

```python
assets = equipment_model.search([])
```

فقد تظهر النتيجة بهذا الشكل:

```text
equipment.asset(1, 5, 8)
```

هذه Multi-record Recordset.

في الحالات الثلاث، نوع النتيجة يبقى Recordset؛ الذي يتغير هو عدد Records الموجودة داخلها.

### `self` داخل Model Method

داخل Model Method، `self` هي Recordset التي استُدعيت عليها Method.

مثلًا:

```python
def print_asset_names(self):
    ...
```

قد تُستدعى Method على مجموعة فارغة، أو Record واحدة، أو عدة Records:

```text
equipment.asset()
equipment.asset(5)
equipment.asset(1, 5, 8)
```

لذلك لا تفترض أن `self` تعني Record واحدة دائمًا.

عدد Records المقبول يعتمد على تصميم Method وطريقة استدعائها.

### Iteration

يمكن المرور على Records داخل Recordset:

```python
for asset in assets:
    print(asset.name)
```

كل `asset` داخل Loop هي Singleton Recordset.

إذا كانت المجموعة:

```text
equipment.asset(1, 5, 8)
```

فإن Loop تتعامل بالتتابع مع:

```text
equipment.asset(1)
equipment.asset(5)
equipment.asset(8)
```

إذا كانت المجموعة فارغة، فلن يُنفذ جسم Loop.

---

## Working with Recordsets

### الوصول إلى Fields

عندما تكون لديك Singleton، تستطيع قراءة Fields مباشرة:

```python
asset.name
asset.state
```

إذا كان المتغير ناتجًا من Search قد تعيد صفرًا أو نتيجة واحدة، يمكن التحقق منه قبل القراءة:

```python
if asset:
    print(asset.name)
```

أما عند وجود عدة Records، فلا نفترض أن قراءة Field عادية تجمع قيمها تلقائيًا.

لقراءة اسم كل Record:

```python
for asset in assets:
    print(asset.name)
```

ولجمع الأسماء في List سنستخدم `mapped()` لاحقًا في هذا القسم.

### Expected Singleton

يظهر الخطأ `Expected singleton` عندما تحتاج عملية إلى Record واحدة، لكنها لا تتلقى Singleton.

من الأمثلة الشائعة قراءة Field عادية من عدة Records داخل Model Method:

```python
def print_asset_name(self):
    print(self.name)
```

قراءة `self.name` بهذه الطريقة تحتاج Singleton. لذلك تفشل إذا احتوت `self` عدة Records، حتى لو كانت أسماؤها متطابقة.

المشكلة تتعلق بعدد Records التي تتعامل معها العملية:

```text
Code تتوقع Record واحدة
        ↓
لكن self تحتوي عدة Records
        ↓
Expected singleton
```

اختيار الحل يعتمد على معنى العملية:

- إذا كانت مطلوبة لكل Record، نصمم Method لدعم المجموعة.
- إذا كانت تخص Record واحدة فعلًا، نتحقق من هذا الشرط.

### `ensure_one()`

`ensure_one()` تتحقق أن Recordset تحتوي Record واحدة بالضبط.

يمكن تجربتها داخل Shell:

```python
asset.ensure_one()
```

أو استخدامها داخل Model Method:

```python
self.ensure_one()
```

| حجم Recordset | النتيجة |
|---|---|
| صفر | ترفع `ValueError` |
| واحدة | تنجح ويتابع الكود التنفيذ |
| أكثر من واحدة | ترفع `ValueError` |

هي لا تختار أول Record، ولا تحوّل المجموعة إلى Singleton؛ تتحقق فقط من شرط العدد.

كذلك لا تثبت أن Record موجودة في Database. التحقق من الوجود له أداة أخرى هي `exists()`.

### `id` و`ids`

كل Record عادية محفوظة في Database لديها ID.

للوصول إلى ID الخاصة بـSingleton:

```python
asset.id
```

قد تكون النتيجة:

```python
5
```

وللحصول على IDs الخاصة بالمجموعة:

```python
assets.ids
```

قد تكون النتيجة:

```python
[1, 5, 8]
```

| التعبير | النتيجة |
|---|---|
| `asset.id` | ID الخاصة بـRecord واحدة |
| `assets.ids` | List من IDs الخاصة بالمجموعة |

يمكن استخدام `ids` مع Singleton أيضًا؛ فتكون النتيجة List تحتوي ID واحدة.

اختيار `id` أو `ids` يعتمد على ما تحتاجه العملية، وليس على اسم المتغير فقط.

### `exists()`

تعلمنا في [[05 ORM]] أن `browse()` تتيح الوصول إلى Records باستخدام IDs معروفة، لكنها لا تثبت وجودها فعليًا.

مثلًا:

```python
records = equipment_model.browse([1, 2, 3])
```

للاحتفاظ بالـRecords الموجودة:

```python
existing_records = records.exists()
```

النتيجة Recordset من Model نفسها، وقد تكون فارغة أو تحتوي واحدة أو عدة Records.

الفرق الذي نحتاج تذكره هنا:

| الأداة | ما الذي تتحقق منه؟ |
|---|---|
| `ensure_one()` | أن المجموعة تحتوي Record واحدة بالضبط |
| `exists()` | الاحتفاظ بالـRecords الموجودة فعليًا |

قد تنجح `ensure_one()` على Recordset مرتبطة بـID واحدة دون أن تثبت وجود هذه Record في Database.

### `filtered()`

`filtered()` تختار Records من Recordset موجودة بالفعل، بناءً على شرط.

مثلًا:

```python
available_assets = assets.filtered(
    lambda asset: asset.state == "available"
)
```

هنا نحتفظ بالـRecords التي تساوي قيمة `state` لديها `available`.

النتيجة Recordset من Model نفسها:

```text
assets
   │
   └──→ filtered()
              │
              └──→ available_assets
```

التصفية تعمل ضمن المجموعة التي لديك، ولا توسعها لتشمل Records أخرى من Model.

إذا لم تجلب Records بعد، وكان بإمكانك تضييق Search من البداية، فاستخدم Domain مناسبة:

```python
available_assets = equipment_model.search([
    ("state", "=", "available"),
])
```

| نقطة البداية | الأداة المناسبة |
|---|---|
| أبحث عن Records داخل Model | `search()` |
| لدي Recordset وأريد اختيار جزء منها | `filtered()` |

لا تجعل Search واسعة ثم Python Filtering الخيار الافتراضي مع البيانات الكبيرة.

تفاصيل كتابة شروط البحث تأتي في [[08 Domains]].

### `mapped()`

يمكن استخدام `mapped()` لجمع قيم Field من Records الموجودة داخل المجموعة.

مثلًا، لجمع الأسماء:

```python
names = assets.mapped("name")
```

قد تكون النتيجة:

```python
["Laptop", "Monitor", "Printer"]
```

هنا حصلنا على List من قيم `name`.

بعض أنواع Fields تجعل `mapped()` تعيد Recordset بدل List. سنستكمل هذا السلوك بعد تعلم [[07 Relationships]].

### `sorted()`

`sorted()` ترتب Recordset موجودة، وتعيد Recordset مرتبة:

```python
ordered_assets = assets.sorted("name")
```

ولعرض الأسماء بعد الترتيب:

```python
ordered_assets.mapped("name")
```

هذا يختلف عن تحديد الترتيب أثناء Search:

```python
assets = equipment_model.search(
    [],
    order="name",
)
```

| الأسلوب | متى نستخدمه؟ |
|---|---|
| `search(..., order="name")` | عندما يكون الترتيب جزءًا من البحث |
| `assets.sorted("name")` | عندما نريد ترتيب Recordset موجودة |

`sorted()` لا تعيد ترتيب المتغير الأصلي في مكانه؛ استخدم القيمة التي تعيدها عندما تحتاج المجموعة المرتبة.

### Set Operations

Recordsets من Model نفسها تدعم عمليات الجمع والتقاطع والفرق:

| العملية | التعبير | النتيجة |
|---|---|---|
| Union | `records_a \| records_b` | Records الموجودة في إحدى المجموعتين أو كلتيهما |
| Intersection | `records_a & records_b` | Records المشتركة بين المجموعتين |
| Difference | `records_a - records_b` | Records الموجودة في الأولى وغير الموجودة في الثانية |

أمثلة:

```python
combined = records_a | records_b

common = records_a & records_b

difference = records_a - records_b
```

النتائج تبقى Recordsets، ويمكن متابعة استخدام ORM عليها.

هذه العمليات تتعامل مع هوية Records، ولا تقارن قيم Fields مثل `name`. وجود اسم متطابق لا يجعل Recordين مختلفتين Record واحدة.

### Recordset ليست Python List

نستطيع المرور على Recordset، وترتيبها، واختيار جزء منها، لكنها Object خاصة بـOdoo.

هي مرتبطة بـModel وEnvironment، وتتيح تنفيذ ORM Operations مثل:

```python
assets.write({
    "state": "in_use",
})
```

أما:

```python
names = assets.mapped("name")
```

فتعطينا List من Strings، يمكن التعامل معها باستخدام أدوات Python العادية.

احتفظ بالـRecordset عندما تريد متابعة العمل على Records، واستخرج القيم عندما تكون هذه القيم هي ما تحتاجه.

تفاصيل Environment تأتي في [[09 Environment & Context]].

---

## Designing Methods for Recordsets

عند كتابة Model Method، ابدأ بالسؤال:

**كم Record يجب أن تدعمها العملية؟**

ثم حدد هل تحتاج معالجة كل Record بشكل مستقل، أم تطبيق عملية واحدة على المجموعة.

### معالجة كل Record على حدة

إذا كانت العملية تحتاج قراءة بيانات كل Record واستخدامها بشكل مستقل، يمكن المرور على المجموعة:

```python
def print_asset_names(self):
    for asset in self:
        print(asset.name)
```

هذه Method تطبع اسم كل Record، سواء استُدعيت على Singleton أو عدة Records.

وإذا كانت `self` فارغة، فلن يُنفذ جسم Loop.

### تطبيق تغيير واحد على المجموعة

دعم عدة Records لا يعني دائمًا الحاجة إلى Loop.

إذا كانت القيم نفسها مطلوبة للجميع، فإن `write()` تدعم Recordset مباشرة:

```python
def mark_as_in_use(self):
    self.write({
        "state": "in_use",
    })
```

هنا نغير حالة جميع Records داخل `self` إلى `in_use`.

| المطلوب | التصميم |
|---|---|
| استخدام بيانات كل Record بشكل مستقل | Iteration |
| تطبيق القيم نفسها على المجموعة | `write()` على Recordset |

### اشتراط Record واحدة

إذا صُممت العملية لتخص Record واحدة في كل استدعاء، يمكن استخدام `ensure_one()` في بدايتها.

مثلًا، Method مصممة لطباعة بيانات Label واحدة:

```python
def print_asset_label(self):
    self.ensure_one()

    print(self.id, self.name)
```

بذلك نتحقق من شرط العدد قبل قراءة بيانات المعدة.

المهم أن يكون اشتراط Singleton قرارًا مقصودًا في تصميم العملية.

> [!warning] لا تعالج Expected singleton بإضافة `ensure_one()` تلقائيًا
> إضافتها إلى Method تستقبل عدة Records لن تجعلها تعمل على الجميع؛ ستجعلها ترفض المجموعة.
>
> إذا كانت العملية يجب أن تدعم عدة Records، فالحل هو تعديل تصميمها لتتعامل معها بشكل صحيح.

---

## Example

سنطبق ما تعلمناه داخل Odoo Shell باستخدام Model باسم `equipment.asset`، مثل التي عرّفناها في [[04 Models & Fields]].

نفترض أنها محمّلة في Database التجريبية وتحتوي:

| Field | المطلوب في المثال |
|---|---|
| `name` | اسم المعدة |
| `state` | تقبل القيمتين `available` و`in_use` |

سننشئ بيانات المثال، ثم نعمل على هذه Records فقط.

### 1. الوصول إلى Model

```python
equipment_model = env["equipment.asset"]
```

### 2. إنشاء Records

```python
printer = equipment_model.create({
    "name": "Printer",
    "state": "available",
})

laptop = equipment_model.create({
    "name": "Laptop",
    "state": "available",
})

monitor = equipment_model.create({
    "name": "Monitor",
    "state": "in_use",
})
```

كل Variable تحتوي Singleton Recordset تمثل Record التي أنشأناها.

### 3. تكوين مجموعة المثال

نجمع Records باستخدام Union:

```python
assets = printer | laptop | monitor
```

بهذا نعمل على Records الثلاث التي أنشأناها، حتى لو كانت Database تحتوي Equipment Records أخرى.

اعرض المجموعة:

```python
assets
```

سترى شكلًا مشابهًا لـ:

```text
equipment.asset(1, 2, 3)
```

الأرقام الفعلية تعتمد على IDs التي حصلت عليها Records في Database.

### 4. قراءة IDs والمرور على المجموعة

ID الخاصة بالـLaptop:

```python
laptop.id
```

وIDs الخاصة بالمجموعة:

```python
assets.ids
```

ثم:

```python
for asset in assets:
    print(asset.id, asset.name)
```

في كل دورة تكون `asset` Singleton.

### 5. التحقق من Singleton

المتغير `laptop` يحتوي Record واحدة أعادتها `create()`.

تحقق:

```python
laptop.ensure_one()
```

تنجح العملية، ثم يمكن قراءة بياناتها:

```python
print(laptop.name)
```

النتيجة:

```text
Laptop
```

أما تطبيق `ensure_one()` على `assets` فسيؤدي إلى `Expected singleton`؛ لأنها تحتوي ثلاث Records.

### 6. التعامل مع Empty Recordset

يمكن تكوين مجموعة فارغة باستخدام Difference بين المجموعة ونفسها:

```python
empty_assets = assets - assets
```

اعرضها:

```python
empty_assets
```

النتيجة:

```text
equipment.asset()
```

ثم تحقق:

```python
if not empty_assets:
    print("No records found")
```

النتيجة:

```text
No records found
```

هذا يوضح أن Empty Recordset قد تنتج من عملية على مجموعة موجودة، وليس من Search فقط.

### 7. تصفية Records

```python
available_assets = assets.filtered(
    lambda asset: asset.state == "available"
)

in_use_assets = assets.filtered(
    lambda asset: asset.state == "in_use"
)
```

افحص أسماء المتاح:

```python
available_assets.mapped("name")
```

النتيجة:

```python
["Printer", "Laptop"]
```

وأسماء المستخدم:

```python
in_use_assets.mapped("name")
```

النتيجة:

```python
["Monitor"]
```

### 8. جمع القيم

لجمع أسماء المجموعة كلها:

```python
names = assets.mapped("name")
```

النتيجة:

```python
["Printer", "Laptop", "Monitor"]
```

`names` هنا List من Strings، بينما `assets` ما زالت Recordset.

### 9. ترتيب المجموعة

```python
ordered_assets = assets.sorted("name")
```

اعرض الأسماء:

```python
ordered_assets.mapped("name")
```

النتيجة:

```python
["Laptop", "Monitor", "Printer"]
```

أما المجموعة الأصلية:

```python
assets.mapped("name")
```

فتبقى:

```python
["Printer", "Laptop", "Monitor"]
```

استخدمنا Recordset المرتبة التي أعادتها `sorted()`، دون تغيير ترتيب `assets` الأصلية.

### 10. تطبيق Set Operations

#### Union

```python
combined = available_assets | in_use_assets
```

النتيجة تجمع Records المتاحة والمستخدمة، وهي Records المثال الثلاث.

#### Intersection

كوّن مجموعة للمراجعة:

```python
review_assets = laptop | monitor
```

ثم احصل على المشترك بينها وبين المتاح:

```python
common_assets = available_assets & review_assets
```

تحقق:

```python
common_assets.mapped("name")
```

النتيجة:

```python
["Laptop"]
```

#### Difference

استبعد Records المستخدمة من مجموعة المثال:

```python
remaining_assets = assets - in_use_assets
```

تحقق:

```python
remaining_assets.mapped("name")
```

النتيجة:

```python
["Printer", "Laptop"]
```

جميع نتائج هذه العمليات تبقى Recordsets من `equipment.asset`.

---

## اختيار الأسلوب المناسب

### بحسب حجم المجموعة وطبيعة العملية

| المطلوب | الأسلوب |
|---|---|
| التحقق من أن المجموعة فارغة | `if not records:` |
| اشتراط Record واحدة بالضبط | `records.ensure_one()` |
| معالجة كل Record بشكل مستقل | Iteration |
| تطبيق القيم نفسها على المجموعة | `records.write(...)` |

### بحسب النتيجة المطلوبة

| المطلوب | الأداة |
|---|---|
| البحث عن Records داخل Model | `search()` |
| الاحتفاظ بالموجود فعليًا من مجموعة IDs | `exists()` |
| اختيار جزء من Recordset موجودة | `filtered()` |
| جمع قيم Field من المجموعة | `mapped()` |
| ترتيب Recordset موجودة | `sorted()` |
| ترتيب النتائج أثناء البحث | `search(..., order=...)` |
| جمع مجموعتين | `\|` |
| الحصول على Records المشتركة | `&` |
| استبعاد Records مجموعة أخرى | `-` |

قبل اختيار الأداة، حدد ما لديك الآن وما تحتاج الحصول عليه.

إذا كنت تحتاج متابعة العمل على Records باستخدام ORM، احتفظ بها كـRecordset. وإذا كنت تحتاج قيم Fields فقط، استخرج هذه القيم بالأداة المناسبة.

---

## Important Notes

- `self` داخل Model Method قد تحتوي صفرًا أو Record واحدة أو عدة Records؛ لا تفترض Singleton دون مراعاة تصميم العملية وطريقة استدعائها.

- `limit=1` تحدد الحد الأقصى للنتائج، ولا تضمن وجود Record مطابقة.

- `ensure_one()` تتحقق من العدد فقط؛ لا تختار أول Record ولا تثبت وجودها في Database. و`exists()` بدورها لا تضمن أن النتيجة Singleton.

- لا تعالج `Expected singleton` بإضافة `ensure_one()` تلقائيًا؛ فقد تكون العملية بحاجة إلى دعم عدة Records أصلًا.

- `filtered()` تعمل ضمن Recordset الموجودة؛ لا تجعل Search واسعة ثم تصفية في Python الخيار الافتراضي إذا أمكن تضييق البحث من البداية.

- Set Operations تحتاج Recordsets من Model نفسها، وتعتمد على هوية Records لا على تشابه قيم Fields فيها.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[07 Relationships]]

---

## Sources

- [Odoo 20 Documentation — Recordsets](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#recordsets)
- [Odoo 20 Documentation — Field Access](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#field-access)
- [Odoo 20 Documentation — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Source Code — Models and Recordsets](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)