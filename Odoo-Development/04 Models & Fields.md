---
type: learning
odoo_version: "20.0"
status: complete
created: 2026-10-13
reviewed: 2026-10-13
---
# 04 Models & Fields

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Model Fundamentals|Model Fundamentals]]
- [[#Building a Model|Building a Model]]
- [[#Working with Fields|Working with Fields]]
- [[#Model Behavior & Metadata|Model Behavior & Metadata]]
- [[#Other Model Types & Next Concepts|Other Model Types & Next Concepts]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

تحتاج Business Applications إلى تمثيل الأشياء التي تتعامل معها المؤسسة، مثل المنتجات والموظفين والمعدات والعقود.

في Odoo نمثل هذه الأنواع باستخدام **Models**، ونحدد المعلومات التي تصفها باستخدام **Fields**. ثم نحفظ البيانات الفعلية على شكل **Records**.

في هذا الفصل سنتعلم بناء Model داخل Module، وتعريف Fields واختيار أنواعها وخصائصها، ثم نجمع ما تعلمناه في مثال عملي يمثل معدات المؤسسة.

---

## Model Fundamentals

### ما هي Model؟

Model تمثل **نوعًا من Business Objects** داخل Odoo.

إذا أردنا إدارة معدات المؤسسة، نحتاج إلى Model تمثل Equipment. ويمكن أن توجد منها Records متعددة:

```text
Equipment
    │
    ├── Office Laptop
    ├── Meeting Room Monitor
    └── Reception Printer
```

Model تعرّف النوع والبيانات والسلوك المرتبط به، بينما تمثل كل Record عنصرًا فعليًا من هذا النوع.

| المفهوم | المثال |
|---|---|
| Model | Equipment |
| Record | جهاز Office Laptop |

قد تستخدم Models أخرى لتمثيل أنواع مختلفة، مثل Customer أوAppointment أوContract، بحسب متطلبات العمل.

---

### ما هي Field؟

Field تمثل معلومة تصف Business Object.

إذا كانت Model تمثل Equipment، فقد نحتاج إلى:

```text
Equipment
    │
    ├── Name
    ├── Serial Number
    ├── Purchase Date
    └── State
```

كل Record تحمل قيمًا لهذه Fields.

مثلًا:

| العنصر | المثال |
|---|---|
| Model | Equipment |
| Record | Office Laptop |
| Field | Serial Number |
| Value | `EQ-001` |

إذن:

- **Model** تعرّف نوع البيانات.
- **Field** تعرّف معلومة داخل هذا النوع.
- **Record** تمثل عنصرًا فعليًا يحمل قيمًا لهذه Fields.

---

### Model وDatabase

Model العادية التي تخزن Business Records ترتبط عادةً بـTable داخل Database.

مثلًا، Technical Model Name:

```text
equipment.asset
```

يرتبط افتراضيًا بجدول اسمه:

```text
equipment_asset
```

في الاسم الافتراضي للجدول، تُستبدل النقاط بـ`_`.

لكن Model تشمل أكثر من مجرد تخزين البيانات؛ فهي تجمع تعريف Fields وMethods وMetadata والسلوك الذي تتعامل معه Odoo.

في Development المعتاد، سنستخدم ORM للتعامل مع Models وRecords. وهذا موضوع [[05 ORM]].

---

## Building a Model

سنبدأ بتحديد مكان الكود، ثم ربطه بالـModule، وبعدها نبني تعريف Model تدريجيًا.

### أين نكتب Models داخل Module؟

توضع ملفات Python الخاصة بالـModels عادةً داخل مجلد باسم:

```text
models/
```

لنفترض أن لدينا Module باسم `equipment_training`، مثل التي أنشأناها في [[03 Modules & Addons]].

يمكن تنظيم الملفات بهذا الشكل:

```text
equipment_training/
├── __init__.py
├── __manifest__.py
└── models/
    ├── __init__.py
    └── equipment.py
```

| الملف أوالمجلد | دوره |
|---|---|
| `models/` | تجميع ملفات Models |
| `models/equipment.py` | تعريف Model المعدات |

اسم `equipment.py` اختيار تنظيمي يوضح محتوى الملف. هو ليس الاسم التقني للـModel داخل Odoo.

---

### ربط الملفات بالـModule

وجود ملف Python داخل `models/` وحده لا يكفي لتحميله؛ يجب أن تصل إليه Import Chain.

داخل:

```text
equipment_training/__init__.py
```

نكتب:

```python
from . import models
```

وداخل:

```text
equipment_training/models/__init__.py
```

نكتب:

```python
from . import equipment
```

فتصبح السلسلة:

```text
equipment_training/__init__.py
                ↓
equipment_training/models/__init__.py
                ↓
equipment_training/models/equipment.py
```

عند تحميل Module، تصل Python إلى ملف `equipment.py` وتنفذ تعريفاته.

فكرة Import Chain شُرحت في [[03 Modules & Addons]]؛ هنا نطبقها على ملفات Models.

---

### تعريف Model داخل Odoo

سنكتب التعريف داخل:

```text
equipment_training/models/equipment.py
```

وسنبنيه تدريجيًا. عند تحديث المثال، نحدّث تعريف Class نفسه، ثم نطبق التغيير بعد اكتماله.

#### استيراد `models`

في بداية الملف:

```python
from odoo import models
```

`odoo` هي Python Package الخاصة بالـFramework، و`models` توفر Base Classes المستخدمة لبناء Odoo Models.

#### `models.Model`

لتعريف Model عادية تحفظ Business Records، ننشئ Python Class ترث من `models.Model`:

```python
class EquipmentAsset(models.Model):
    ...
```

`EquipmentAsset` هي Class الخاصة بنا، و`models.Model` هي Base Class التي تربطها بنظام Models في Odoo.

هذا استخدام لوراثة Python الأساسية. تفاصيل الوراثة والتفاعل بين Odoo Models تأتي في [[18 Inheritance & Module Interaction]].

#### Python Class Name و`_name`

في المثال السابق، `EquipmentAsset` هي **Python Class Name**.

أما هوية Model التقنية داخل Odoo فنحددها باستخدام `_name`:

```python
class EquipmentAsset(models.Model):
    _name = "equipment.asset"
```

لدينا الآن اسمان مختلفان:

| الاسم | دوره |
|---|---|
| `EquipmentAsset` | Python Class Name |
| `equipment.asset` | Odoo Technical Model Name |

Technical Model Name هو الاسم الذي تستخدمه Odoo للإشارة إلى Model في ORM وأجزاء النظام الأخرى.

من أمثلته:

```text
equipment.asset
sale.order
res.partner
project.task
```

سنستخدمه عمليًا للوصول إلى Model في [[05 ORM]].

#### `_description`

تضيف `_description` وصفًا مقروءًا للـModel:

```python
class EquipmentAsset(models.Model):
    _name = "equipment.asset"
    _description = "Equipment Asset"
```

| Attribute | القيمة | وظيفتها |
|---|---|---|
| `_name` | `equipment.asset` | الهوية التقنية |
| `_description` | `Equipment Asset` | الوصف المقروء للـModel |

تغيير `_description` لا يغيّر Technical Model Name.

#### التعريف حتى الآن

أصبح الملف يحتوي:

```python
from odoo import models


class EquipmentAsset(models.Model):
    _name = "equipment.asset"
    _description = "Equipment Asset"
```

أكملنا التعريف الأساسي في الكود. سيُحمّل عند تحميل Module، وتُطبّق بنيته داخل Database من خلال Install أوUpgrade بحسب حالة Module.

الخطوة التالية هي إضافة Fields الخاصة ببيانات المعدات.

---

### تعريف Fields داخل Model

نحتاج إلى استيراد `fields` بجانب `models`:

```python
from odoo import fields, models
```

| الأداة | استخدامها |
|---|---|
| `models` | بناء Odoo Models |
| `fields` | تعريف Fields داخلها |

لإضافة اسم نصي قصير، نستخدم `fields.Char`:

```python
from odoo import fields, models


class EquipmentAsset(models.Model):
    _name = "equipment.asset"
    _description = "Equipment Asset"

    name = fields.Char()
```

في هذا السطر:

```python
name = fields.Char()
```

- `name` هي Technical Field Name.
- `fields.Char` هي Field Type، وتستخدم للنصوص القصيرة.

Field تُعرّف كـClass Attribute داخل Model.

وقد نضيف إلى تعريفها Parameters تتحكم في خصائصها، مثل الاسم المعروض والقيمة الافتراضية وما إذا كانت مطلوبة.

---

## Working with Fields

عند تعريف Field نحتاج إلى قرارين:

| السؤال | ما يحدده |
|---|---|
| ما معنى البيانات التي سنحفظها؟ | Field Type |
| ما الخصائص المطلوبة لهذه Field؟ | Field Parameters |

أمثلة Fields في هذا القسم توضح الخيارات المتاحة. في Example سنختار منها ما تحتاجه Model المعدات.

### Field Types

#### `Char`

تستخدم للنصوص القصيرة:

```python
name = fields.Char()
```

مناسبة للأسماء والأكواد وReferences وSerial Numbers وأرقام الهواتف.

اختيارها يعتمد على معنى القيمة، وليس على احتوائها أرقامًا فقط.

مثلًا، Code بقيمة:

```text
00125
```

هو Identifier قد تكون أصفاره الأولى مهمة. وأرقام الهواتف قد تحتوي `+` أوامتدادًا.

لذلك تُستخدم `Char` عادةً لهذه البيانات، بينما تُستخدم الأنواع الرقمية للكميات والقيم التي تحتاج عمليات حسابية.

#### `Text`

تستخدم للنصوص الطويلة أومتعددة الأسطر:

```python
description = fields.Text()
```

مثل الوصف والملاحظات والأسباب والتعليقات.

| طبيعة النص | النوع المعتاد |
|---|---|
| نص قصير | `Char` |
| نص طويل أومتعدد الأسطر | `Text` |

#### `Boolean`

تمثل قيمة منطقية: `True` أو`False`.

```python
enabled = fields.Boolean()
```

مناسبة لخاصية لها حالتان، مثل Enabled أوDisabled.

#### `Integer`

تستخدم للأعداد الصحيحة:

```python
quantity = fields.Integer()
```

مثل عدد القطع أوالعناصر:

```text
0
1
15
100
```

#### `Float`

تستخدم للقيم التي قد تحتوي جزءًا عشريًا:

```python
duration = fields.Float()
```

مثل مدة بالساعات أوقياس:

```text
1.5
10.25
```

#### `Monetary`

تستخدم لتمثيل مبلغ مالي مرتبط بعملة.

القيمة `100` وحدها لا توضح هل المبلغ بالريال أوالدولار؛ لذلك تحتاج Monetary Field إلى Currency Field تحدد العملة المستخدمة.

تعريف Currency Field يعتمد على Relational Field، وسنتعرف على العلاقات في 
[[07 Relationships]].

يكفينا الآن معرفة استخدام هذا النوع، دون إضافة تعريف غير مكتمل إلى Model المثال.

#### `Date`

تستخدم للتاريخ دون وقت:

```python
purchase_date = fields.Date()
```

مثل:

```text
2026-10-01
```

مناسبة لتاريخ الشراء أوالانتهاء أوالميلاد.

#### `Datetime`

تستخدم للتاريخ والوقت معًا:

```python
start_at = fields.Datetime()
```

مثل بداية موعد أووقت عملية مجدولة.

تتعامل Odoo مع قيم Datetime على أساس UTC، وتعرضها الواجهة بحسب Time Zone الخاصة بالمستخدم. لذلك لا تضف أوتطرح فرقًا زمنيًا يدويًا لمجرد تغيير طريقة العرض.

#### `Selection`

تستخدم عندما تكون القيمة واحدة من مجموعة خيارات محددة:

```python
state = fields.Selection(
    [
        ("available", "Available"),
        ("in_use", "In Use"),
        ("retired", "Retired"),
    ],
)
```

كل Option تحتوي قيمتين:

| الجزء | المثال | استخدامه |
|---|---|---|
| Technical Value | `available` | داخل الكود والبيانات |
| Display Label | `Available` | النص المعروض للمستخدم |

داخل Business Logic نتعامل مع Technical Value، لأن Display Label قد تتغير أوتُترجم.

#### `Binary`

تستخدم للبيانات الثنائية، مثل محتوى ملف مرفوع:

```python
document = fields.Binary()
```

#### `Image`

تستخدم للصور:

```python
image = fields.Image()
```

مثل صورة منتج أومعدة.

#### `Html`

تستخدم للمحتوى المنسق بصيغة HTML:

```python
description_html = fields.Html()
```

اخترها عندما يكون Rich Text جزءًا من المتطلبات. وللنص العادي، تكون `Char` أو`Text` غالبًا أنسب.

---

### Field Parameters

Field Type تحدد نوع البيانات، بينما تضيف Parameters خصائص إلى تعريف Field.

سنغطي أكثر الخصائص شيوعًا في هذه المرحلة.

#### `string`

تحدد Display Label الخاصة بالـField:

```python
serial_number = fields.Char(
    string="Serial Number",
)
```

| الجزء | معناه |
|---|---|
| `serial_number` | Technical Field Name |
| `Serial Number` | Display Label |

إذا لم نحدد `string`، تشتق Odoo Label من اسم Field.

#### `required`

تجعل Field مطلوبة:

```python
name = fields.Char(
    required=True,
)
```

في هذا المثال، يجب توفير قيمة لـ`name` لحفظ Record.

#### `default`

تحدد قيمة افتراضية عند إنشاء Record جديدة:

```python
enabled = fields.Boolean(
    default=True,
)
```

تبدأ Field بالقيمة `True` ما لم تُعطَ قيمة أخرى.

يمكن أيضًا تمرير Function لتوفير Default ديناميكية:

```python
created_for_date = fields.Date(
    default=fields.Date.today,
)
```

هنا نمرر Function نفسها دون استدعائها مسبقًا، حتى تستدعيها Odoo عند طلب القيمة الافتراضية.

| النوع | المثال | السلوك |
|---|---|---|
| Static Default | `default=True` | استخدام قيمة ثابتة |
| Dynamic Default | `default=fields.Date.today` | الحصول على القيمة عند الحاجة |

Default توفر قيمة ابتدائية؛ لا تعني أن Field ستُحدَّث باستمرار بعد إنشاء Record.

#### `readonly`

تجعل Field للقراءة بالطريقة المعتادة في User Interface:

```python
reference = fields.Char(
    readonly=True,
)
```

لكنها لا تمنع الكتابة من كود الخادم بحد ذاتها؛ تظل العملية خاضعة للصلاحيات وقواعد العمل الأخرى.

> [!warning] `readonly` ليست Server-side Security
> إذا كان منع التعديل قاعدة عمل مهمة، فيجب فرضها على الخادم بالطريقة المناسبة، وليس الاعتماد على حالة Field في الواجهة فقط.

تفاصيل Security تأتي في [[11 Security & Multi-Company]].

#### `copy`

تتحكم في نسخ قيمة Field عند Duplicate لـRecord:

| القيمة | السلوك |
|---|---|
| `copy=True` | تسمح بنسخ قيمة Field |
| `copy=False` | تستبعد القيمة الأصلية من النسخ التلقائي |

مثلًا:

```python
serial_number = fields.Char(
    copy=False,
)
```

هذا مناسب عندما يجب إعطاء النسخة الجديدة Serial Number خاصة بها.

`copy=False` لا تعني أن القيمة الجديدة ستكون فارغة دائمًا؛ فقد تحصل Record الجديدة على Default مناسبة.

في مثال Serial Number، إذا لم توجد Default أوتخصيص آخر، ستكون القيمة فارغة بدل نسخ الرقم الأصلي.

> [!important] النسخ يختلف عن Uniqueness
> `copy=False` لا تمنع حفظ الرقم نفسه في Record أخرى.
>
> فرض عدم التكرار موضوع مختلف، وسيُشرح في [[17 Constraints]].

#### `index`

تطلب إنشاء Database Index للـField:

```python
serial_number = fields.Char(
    index=True,
)
```

Index قد تحسن بعض عمليات البحث، لكنها تضيف تكلفة في التخزين وعمليات الكتابة والصيانة.

استخدمها بناءً على نمط استخدام البيانات، دون إضافتها تلقائيًا لكل Field.

تفاصيل الأداء تأتي في [[28 Performance]].

#### `help`

تضيف نصًا يساعد المستخدم على فهم Field:

```python
serial_number = fields.Char(
    help="Serial number printed on the device.",
)
```

يمكن استخدامه لتوضيح معنى البيانات المطلوبة أوكيفية الحصول عليها.

---

## Model Behavior & Metadata

بعض خصائص Model تؤثر على تعامل Odoo مع Records وعرضها وترتيبها، بالإضافة إلى أنواع Fields وقيمها.

### Field `active`

اسم `active` له سلوك معروف داخل Odoo.

نعرفه عادةً هكذا:

```python
active = fields.Boolean(
    default=True,
)
```

عندما تكون قيمته `False`، تُعامل Record عادةً كـArchived وتُستبعد من كثير من عمليات البحث المعتادة.

| الحالة | النتيجة |
|---|---|
| Archive | Record تبقى موجودة مع جعلها غير نشطة |
| Delete | Record تُحذف |

أضفنا `active` صراحةً إلى Model؛ ليست كل Boolean Field مسؤولة عن Archive.

تفاصيل التعامل مع البيانات من خلال ORM تأتي في [[05 ORM]]، وتأثير Context على ظهور Archived Records يأتي في [[09 Environment & Context]].

---

### Automatic Fields

توفر Odoo بعض Technical Fields تلقائيًا في Models العادية.

#### `id`

تمثل ID الخاصة بالـRecord، وتُستخدم لتمييزها داخل Model نفسها.

قد توجد Equipment وEmployee تحملان ID نفسها؛ الرقم لا يحدد Record عبر جميع Models وحده.

#### Access Log Fields

عندما يكون Access Log مفعّلًا، وهو السلوك الافتراضي في Models العادية، تدير Odoo:

| Field | ما تسجله |
|---|---|
| `create_uid` | المستخدم الذي أنشأ Record |
| `create_date` | وقت الإنشاء |
| `write_uid` | المستخدم الذي نفّذ آخر تعديل |
| `write_date` | وقت آخر تعديل |

لا نحتاج تعريف هذه Fields يدويًا في Model العادية.

`write_uid` و`write_date` تصفان آخر تعديل؛ لا تمثلان سجلًا كاملًا لكل التغييرات السابقة.

---

### Record Display Name

ID مفيدة لتمييز Record تقنيًا، لكن المستخدم يحتاج عادةً إلى قيمة مقروءة، مثل اسم المعدة.

تستخدم Odoo تمثيلًا مقروءًا للـRecord في مواضع مثل عناوين Records والروابط بين البيانات.

#### Field `name`

في كثير من Models نعرّف Field باسم `name`:

```python
name = fields.Char()
```

قد تحمل قيمة مثل:

```text
Office Laptop
```

غالبًا تكون هذه هي القيمة الطبيعية التي يتعرف بها المستخدم على Record.

`name` هنا Field نعرّفها نحن؛ ليست من Access Log Fields التي تُضاف تلقائيًا.

#### `_rec_name`

إذا كانت Field أخرى أنسب لتمثيل Record، يمكن تحديدها باستخدام `_rec_name`.

مثلًا، إذا كانت المؤسسة تتعرف على المعدة باستخدام Code، نكتب داخل Class:

```python
_rec_name = "code"

code = fields.Char()
name = fields.Char()
```

بذلك تصبح `code` هي Field الأساسية المستخدمة لتمثيل Record بشكل مقروء.

اخترها بناءً على طريقة تعرف المستخدم على البيانات. إذا كانت `name` هي الأنسب، يمكن الاعتماد على السلوك الافتراضي.

#### `display_name`

توفر Odoo Field باسم `display_name` تمثل النص المقروء المستخدم لعرض Record.

بشكل مبسط:

```text
Field المحددة لتمثيل Record
            ↓
القيمة المقروءة
            ↓
display_name
```

مثلًا:

| Field | القيمة |
|---|---|
| `id` | `42` |
| `name` | `Office Laptop` |
| `display_name` | `Office Laptop` |

وإذا اعتمدت Model على `code` من خلال `_rec_name`، فقد يكون Display Name مثل `EQ-001`.

| العنصر | دوره |
|---|---|
| `name` | Field تحمل اسم Record |
| `_rec_name` | تحدد Field المستخدمة أساسًا لتمثيل Record |
| `display_name` | التمثيل المقروء الذي توفره Odoo |

لا نحتاج الآن إلى تخصيص طريقة حساب `display_name`؛ يكفينا فهم دورها.

---

### Default Ordering

يمكن تحديد الترتيب الافتراضي للـRecords باستخدام `_order` داخل Class:

```python
_order = "name, id"
```

هذا يرتب النتائج أولًا حسب `name`، وعند تساوي الأسماء يستخدم `id` كترتيب إضافي.

اختر ترتيبًا طبيعيًا مناسبًا للـModel. ويمكن تحديد ترتيب مختلف لعملية بحث معينة، كما سنتعلم في [[05 ORM]].

---

## Other Model Types & Next Concepts

حتى الآن استخدمنا `models.Model` والأنواع الأساسية من Fields.

سنأخذ هنا تمهيدًا للأنواع والمفاهيم التي ستأتي لاحقًا، بالقدر الذي يوضح مكانها في الصورة العامة.

### أنواع Models الأخرى

#### `models.TransientModel`

تستخدم لبيانات مؤقتة، مثل مدخلات Wizards أوبيانات عملية قصيرة العمر.

Records الخاصة بها تُخزن مؤقتًا داخل Database، وتقوم Odoo بتنظيفها دوريًا.

> [!warning] TransientModel لا تعني التخزين في RAM فقط
> بياناتها موجودة في Database، لكنها مخصصة للاستخدام المؤقت. لا تستخدمها لحفظ Business Records يجب الاحتفاظ بها بشكل دائم.

تفاصيلها تأتي في [[19 Wizards]].

#### `models.AbstractModel`

تستخدم لتعريف سلوك قابل للمشاركة مع Models أخرى، مثل Fields وMethods مشتركة.

الغرض منها إعادة استخدام السلوك، وليست تمثيل مجموعة مستقلة من Business Records بالطريقة المعتادة.

سنشرحها في [[18 Inheritance & Module Interaction]].

---

### Relational Fields

Business Objects غالبًا ترتبط ببعضها:

```text
Employee → Department
Sales Order → Customer
Equipment → Assigned Employee
```

توفر Odoo ثلاثة أنواع أساسية لتمثيل العلاقات:

- `Many2one`
- `One2many`
- `Many2many`

سنشرح معناها وتعريفها وسلوكها في [[07 Relationships]].

---

### Computed Fields

بعض القيم يمكن حسابها اعتمادًا على بيانات أخرى، مثل Subtotal أوDuration أوRemaining Amount.

لهذه الحالات توفر Odoo Computed Fields.

سنشرحها مع Related Fields وOnchange في
[[16 Computed Fields, Related Fields & Onchange]].

---

## Example

سنضيف Model للمعدات داخل Module باسم `equipment_training`، التي أعددناها وثبتناها في [[03 Modules & Addons]].

سنبدأ من متطلبات البيانات، ثم نعرّف الملفات وModel، وبعدها نطبق التغيير على Training Database.

---

### 1. تحديد المتطلبات

نحتاج إلى حفظ معدات المؤسسة، مع المعلومات التالية:

- الاسم.
- الرقم التسلسلي.
- الوصف.
- تاريخ الشراء.
- الحالة.
- حالة النشاط أوالأرشفة.

كل معدة تمثل Record مستقلة داخل Model المعدات.

---

### 2. تصميم Model وFields

سنستخدم:

| العنصر | الاسم |
|---|---|
| Python Class Name | `EquipmentAsset` |
| Technical Model Name | `equipment.asset` |
| Description | `Equipment Asset` |

ونختار Fields التالية:

| المعلومة | Technical Field Name | النوع |
|---|---|---|
| الاسم | `name` | `Char` |
| الرقم التسلسلي | `serial_number` | `Char` |
| الوصف | `description` | `Text` |
| تاريخ الشراء | `purchase_date` | `Date` |
| الحالة | `state` | `Selection` |
| النشاط | `active` | `Boolean` |

استخدمنا `Char` للـSerial Number لأنها Identifier، وليست كمية نستخدمها في العمليات الحسابية.

---

### 3. إعداد الملفات

تصبح Structure الخاصة بالـModule:

```text
equipment_training/
├── __init__.py
├── __manifest__.py
└── models/
    ├── __init__.py
    └── equipment.py
```

سنربط الملف الجديد بالـModule من خلال Import Chain.

---

### 4. Root `__init__.py`

داخل:

```text
equipment_training/__init__.py
```

نضيف:

```python
from . import models
```

إذا كانت هناك Imports أخرى، نحتفظ بها.

---

### 5. `models/__init__.py`

داخل:

```text
equipment_training/models/__init__.py
```

نضيف:

```python
from . import equipment
```

الآن تصل Import Chain إلى `models/equipment.py`.

---

### 6. تعريف Model

داخل:

```text
equipment_training/models/equipment.py
```

نكتب:

```python
from odoo import fields, models


class EquipmentAsset(models.Model):
    _name = "equipment.asset"
    _description = "Equipment Asset"
    _order = "name, id"

    name = fields.Char(
        required=True,
    )

    serial_number = fields.Char(
        string="Serial Number",
        copy=False,
        index=True,
        help="Serial number printed on the device.",
    )

    description = fields.Text()

    purchase_date = fields.Date()

    state = fields.Selection(
        [
            ("available", "Available"),
            ("in_use", "In Use"),
            ("retired", "Retired"),
        ],
        default="available",
        required=True,
    )

    active = fields.Boolean(
        default=True,
    )
```

---

### 7. فهم اختيارات التعريف

| الجزء | سبب استخدامه |
|---|---|
| `name` مع `required=True` | كل معدة تحتاج اسمًا |
| `serial_number` كـ`Char` | الرقم التسلسلي Identifier |
| `copy=False` | منع نقل الرقم الأصلي تلقائيًا عند Duplicate |
| `index=True` | نفترض في المثال وجود بحث متكرر بالرقم التسلسلي |
| `help` | توضيح مكان الحصول على الرقم |
| `state` كـ`Selection` | حصر الحالة في خيارات محددة |
| `default="available"` | بدء Records الجديدة بالحالة المتاحة |
| `active` مع `default=True` | بدء Record كنشطة ودعم الأرشفة |
| `_order = "name, id"` | ترتيب افتراضي واضح |

وجود Index أو`copy=False` على `serial_number` لا يجعلها Unique. فرض عدم التكرار سيأتي في [[17 Constraints]].

---

### 8. تطبيق التغيير باستخدام Upgrade

أضفنا Model وFields جديدة، لذلك نحتاج إلى Upgrade للـModule المثبتة في Training Database.

نفّذ العملية وفق Workflow التطوير الذي تعلمناه في [[03 Modules & Addons]].

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_training \
  -u equipment_training \
  --stop-after-init
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config/odoo.conf `
  -d odoo20_training `
  -u equipment_training `
  --stop-after-init
```

بعد اكتمال Upgrade بنجاح، شغّل Odoo بشكل طبيعي.

الفرق بين Install وUpgrade وRestart شُرح في [[03 Modules & Addons]].

---

### 9. ما الذي أصبح جاهزًا؟

أصبح لدينا Backend Model باسم `equipment.asset`، مع Fields وMetadata وسلوك الأرشفة والترتيب الافتراضي.

سنستخدمها في [[05 ORM]] لإنشاء Records وقراءتها وتعديلها.

> [!info] تعريف Model لا ينشئ واجهة تلقائيًا
> فتح Equipment من User Interface يحتاج إلى إعداد الصلاحيات وViews وActions وMenus.
>
> سنضيف هذه الأجزاء تدريجيًا في الفصول المخصصة لها.

---

## Important Notes

- يجب أن تكون ملفات Models جزءًا من Import Chain حتى تُحمّل تعريفاتها.

- `from . import models` تستورد Package داخل Module الخاصة بنا، بينما `from odoo import models` تستورد أدوات Models من Odoo Framework.

- Python Class Name وTechnical Model Name مختلفان؛ أجزاء Odoo تستخدم الاسم التقني للإشارة إلى Model.

- أرقام الهواتف وReferences وCodes تكون غالبًا `Char`، حتى لو احتوت أرقامًا؛ فقد تكون الأصفار الأولى أوالرموز جزءًا من هويتها.

- `readonly=True` تؤثر على التعامل المعتاد مع Field في الواجهة، ولا تفرض منع التعديل من الخادم بحد ذاتها.

- `copy=False` لا تجعل القيمة Unique، ولا تضمن أن القيمة الجديدة ستكون فارغة؛ قد تُطبق Default مناسبة.

- عند استخدام Default ديناميكية، مرر Function المناسبة دون استدعائها مسبقًا أثناء تحميل ملف Python.

- `active=False` تعني عادةً Archive، وليس Delete.

- في `Selection` استخدم Technical Value داخل الكود، وليس Display Label التي قد تتغير أوتُترجم.

- لا تضف Index لكل Field تلقائيًا؛ فائدتها تعتمد على الاستخدام، ولها تكلفة في التخزين والكتابة.

- `id` تميز Record داخل Model نفسها؛ قد تحمل Records من Models مختلفة الرقم نفسه.

- `TransientModel` تخزن Records مؤقتًا في Database، وتُنظف دوريًا؛ ليست تخزينًا في RAM فقط.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[05 ORM]]

---

## Sources

- [Odoo 20 Documentation — Models and Basic Fields](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/03_basicmodel.html)
- [Odoo 20 Documentation — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Documentation — Command-line Interface](https://www.odoo.com/documentation/20.0/developer/reference/cli.html)
- [Odoo 20 Source Code — Models](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)
- [Odoo 20 Source Code — Fields](https://github.com/odoo/odoo/blob/20.0/odoo/orm/fields.py)
- [Odoo 20 Source Code — Numeric and Monetary Fields](https://github.com/odoo/odoo/blob/20.0/odoo/orm/fields_numeric.py)