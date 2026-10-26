---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 08 Domains

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Domain Fundamentals|Domain Fundamentals]]
- [[#Comparison Operators|Comparison Operators]]
- [[#Combining Conditions|Combining Conditions]]
- [[#Domains with Relationships|Domains with Relationships]]
- [[#Building Domains with Domain Class|Building Domains with Domain Class]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في [[05 ORM]] استخدمنا Domains بسيطة لتحديد Records التي نريد البحث عنها.

في هذا الشابتر سنتعلم كيف نبني هذه الشروط، وكيف نجمعها للتعبير عن طلب أكثر تفصيلًا.

مثلًا:

- الأجهزة المتاحة.
- الأجهزة التي يحتوي اسمها على كلمة معينة.
- الأجهزة النشطة التي تحقق واحدًا من شرطين.
- جهات الاتصال المرتبطة بدولة محددة.
- Records التي لديها Records مرتبطة تحقق شرطًا معينًا.

الهدف ليس حفظ Operators فقط، بل تحويل الطلب إلى منطق واضح، ثم كتابة Domain تمثل هذا المنطق.

---

## Domain Fundamentals

### ما هي Domain؟

الـ**Domain** هي تعبير يصف شروطًا يجب أن تحققها Records حتى تدخل ضمن النتيجة المطلوبة.

مثلًا:

```python
[
    ("state", "=", "available"),
]
```

تعني:

```text
state يساوي available
```

Domain نفسها لا تنفذ بحثًا. هي تصف الشروط، ثم نمررها إلى عملية مثل `search()` لتنفيذ البحث.

في أمثلة الأجهزة، نفترض أن `equipment.asset` معرفة كما في [[04 Models & Fields]]، وتحتوي على:

| Field | النوع أوالقيم المستخدمة |
| --- | --- |
| `name` | Char |
| `serial_number` | Char |
| `purchase_date` | Date |
| `state` | `available` أو`in_use` أو`retired` |
| `active` | Boolean |

سنكتب أمثلة البحث داخل **Odoo Shell**، ونبدأ بالحصول على Model:

```python
equipment_model = env["equipment.asset"]
```

بعدها نحفظ شروط البحث في Variable ونستخدمها:

```python
domain = [
    ("state", "=", "available"),
]

available_assets = equipment_model.search(domain)
```

هنا:

- `domain` تحتوي شروط البحث.
- استدعاء `equipment_model.search(domain)` ينفذ البحث.
- `available_assets` تحتوي Recordset بالنتائج.

لن نعيد شرح ORM Methods أوRecordsets من البداية؛ يمكن الرجوع إلى [[05 ORM]] و[[06 Records & Recordsets]] عند الحاجة.

---

### Condition واحدة

في List Syntax، نكتب كل Condition عادةً داخل Tuple من ثلاثة أجزاء:

```python
("field_name", "operator", value)
```

مثلًا:

```python
("state", "=", "available")
```

| الجزء | وظيفته في المثال |
| --- | --- |
| `"state"` | Technical Name للـField |
| **=** | Operator المساواة |
| `"available"` | القيمة التي نقارن بها |

ثم نضع Condition داخل List لتصبح Domain:

```python
domain = [
    ("state", "=", "available"),
]
```

استخدم Technical Names، وليس Labels الظاهرة في الواجهة.

مثلًا، نقارن Selection Field بالقيمة المخزنة `available`، وليس بالـLabel `Available`.

---

### نوع Value

القيمة الموجودة في الشرط يجب أن تناسب Field والـOperator.

مثلًا:

```python
domain = [
    ("active", "=", True),
]
```

هنا `True` هي Boolean، وليست النص `"True"`.

أمثلة أخرى:

| نوع المقارنة | مثال Condition |
| --- | --- |
| نص | `("name", "=", "Office Printer")` |
| Boolean | `("active", "=", True)` |
| رقم | `("id", ">", 10)` |
| مجموعة قيم | `("state", "in", ["available", "in_use"])` |

الأرقام المستخدمة مع `id` في الأمثلة توضيحية؛ لا نفترض وجود Records بهذه IDs في كل Database.

---

### Empty Domain

الـDomain الفارغة:

```python
[]
```

لا تضيف شروط تصفية من جانبنا.

مثلًا:

```python
assets = equipment_model.search([])
```

لكن هذا لا يعني بالضرورة إرجاع كل Records الموجودة في Database.

ما زالت النتيجة تتأثر بالصلاحيات وسلوك البحث المعتاد. مثلًا، في Model تستخدم `active`، تستبعد عمليات البحث المعتادة Records المؤرشفة افتراضيًا.

> [!important] Empty Domain لا تلغي القيود الأخرى
> `[]` تعني أننا لم نضف شروطًا إلى البحث.
>
> لا تعني تجاوز الصلاحيات أوإلغاء استبعاد Records المؤرشفة.

سنشرح تأثير Context في [[09 Environment & Context]]، والصلاحيات في [[11 Security & Multi-Company]].

---

### أين تستخدم Domains؟

نركز هنا على كتابة الشروط وفهمها، لكن نفس الفكرة تستخدم في أكثر من مكان:

| المكان | دور Domain |
| --- | --- |
| ORM Search | تحديد Records المطلوبة من البحث |
| Relational Field | تضييق الخيارات المقترحة في الواجهة |
| Window Action | تحديد Records التي يعرضها Action |
| Search View | تعريف شروط البحث والـFilters |
| تعريفات Security | التعبير عن شروط الوصول إلى Records |

سنطبق Domain على Relational Field في هذا الشابتر، لأن Relationships شُرحت في [[07 Relationships]].

أما تفاصيل Actions وSearch Views وSecurity فتبقى في شابتراتها.

**وجود Domain في Action أوField لا يجعلها تلقائيًا قاعدة حماية.** مكان استخدامها هو الذي يحدد دورها.

---

## Comparison Operators

### المقارنات الأساسية

هذه Operators تقارن قيمة Field بالقيمة المحددة:

| Operator | المعنى | مثال Condition |
| --- | --- | --- |
| **=** | يساوي | `("state", "=", "available")` |
| **!=** | لا يساوي | `("state", "!=", "retired")` |
| **>** | أكبر من | `("id", ">", 10)` |
| **>=** | أكبر من أويساوي | `("id", ">=", 10)` |
| **<** | أصغر من | `("id", "<", 50)` |
| **<=** | أصغر من أويساوي | `("id", "<=", 50)` |

مثلًا، للبحث عن الأجهزة التي ليست بحالة Retired:

```python
assets = equipment_model.search([
    ("state", "!=", "retired"),
])
```

استخدم المقارنة التي تناسب نوع Field والمعنى المطلوب.

---

### `in` و`not in`

نستخدم `in` عندما نريد أن تكون قيمة Field واحدة من عدة قيم:

```python
assets = equipment_model.search([
    ("state", "in", ["available", "in_use"]),
])
```

تعني أن `state` تساوي `available` أو`in_use`.

أما `not in` فتستبعد القيم المحددة:

```python
assets = equipment_model.search([
    ("state", "not in", ["retired"]),
])
```

في هذا المثال، يمكن أيضًا استخدام **!=** لأننا نستبعد قيمة واحدة فقط.

ونستخدم نفس الفكرة مع IDs:

```python
assets = equipment_model.search([
    ("id", "in", [1, 5, 8]),
])
```

النتيجة تحتوي Records المطابقة الموجودة والمسموح بالوصول إليها؛ لا ينشئ البحث Records للـIDs غير الموجودة.

> [!tip] عدة قيم لنفس Field
> إذا كان المطلوب مطابقة واحدة من عدة قيم لنفس Field، تكون `in` غالبًا أوضح من كتابة عدة شروط OR.

---

### البحث النصي

#### `like` و`ilike`

نستخدمهما للبحث عن نص داخل قيمة Field.

مثلًا:

```python
assets = equipment_model.search([
    ("name", "ilike", "laptop"),
])
```

يمكن أن يطابق أسماء مثل:

- `Laptop`
- `Office Laptop`
- `LAPTOP Stand`

الفرق الأساسي:

| Operator | حساسية حالة الأحرف | نمط المطابقة |
| --- | --- | --- |
| **like** | حساس لحالة الأحرف | يضيف `%` قبل القيمة وبعدها |
| **ilike** | غير حساس لحالة الأحرف | يضيف `%` قبل القيمة وبعدها |

عند تمرير نص عادي مثل `"laptop"` إلى `ilike`، نبحث عن وجوده داخل الاسم، وليس عن تطابق الاسم كاملًا.

#### Wildcards

في Patterns الخاصة بالبحث النصي:

| الرمز | معناه |
| --- | --- |
| **%** | أي عدد من الأحرف، بما فيها صفر |
| **_** | حرف واحد |

هذه الرموز جزء من Pattern، وليست دائمًا أحرفًا حرفية نبحث عنها.

#### التحكم في Pattern باستخدام =like و=ilike

نستخدم **=like** و**=ilike** عندما نريد تحديد Pattern دون أن تضيف Odoo `%` حولها تلقائيًا.

مثلًا، للبحث عن اسم **يبدأ** بكلمة Laptop دون حساسية لحالة الأحرف:

```python
assets = equipment_model.search([
    ("name", "=ilike", "Laptop%"),
])
```

أما:

```python
assets = equipment_model.search([
    ("name", "=ilike", "Laptop"),
])
```

فتبحث عن مطابقة الاسم كاملًا دون حساسية لحالة الأحرف؛ لأننا لم نضع Wildcards في Pattern.

| Operator | حساسية حالة الأحرف | هل تضيف `%` حول القيمة؟ |
| --- | --- | --- |
| **=like** | حساس | لا |
| **=ilike** | غير حساس | لا |

#### نفي البحث النصي

يمكن استخدام `not like` و`not ilike` لنفي المطابقة.

مثلًا:

```python
assets = equipment_model.search([
    ("name", "not ilike", "laptop"),
])
```

تعني أن الاسم لا يطابق البحث النصي المحدد.

---

### `False` والقيم غير المحددة

مع Boolean Field، تمثل `False` القيمة المنطقية المعتادة:

```python
archived_assets = equipment_model.search([
    ("active", "=", False),
])
```

هنا طلبنا Records المؤرشفة صراحةً باستخدام Field `active`.

أما في Many2one Field، فنستخدم `False` للبحث عن علاقة غير محددة.

مثلًا، إذا كانت Model تحتوي `department_id`:

```python
[
    ("department_id", "=", False),
]
```

تعني أن Record غير مرتبطة بأي Department.

وللبحث عن Records المرتبطة بـDepartment:

```python
[
    ("department_id", "!=", False),
]
```

لا تستخدم النص `"False"` بدل القيمة `False`.

---

### المقارنة الاختيارية

قد تقابل Operator باسم **=?**.

تتصرف مثل المساواة عندما تكون القيمة المقدمة محددة، لكنها لا تفرض قيدًا عندما تكون القيمة `False` أو`None`.

مثلًا:

```python
[
    ("department_id", "=?", False),
]
```

لا تعني البحث عن Records التي ليس لديها Department؛ هذا الشرط لا يضيف قيدًا في هذه الحالة.

الفرق:

| Condition | المعنى |
| --- | --- |
| `("department_id", "=", False)` | العلاقة غير محددة |
| `("department_id", "=?", False)` | لا تضف قيدًا من هذا الشرط |

نحتاج **=?** عادةً عندما نبني Domain اعتمادًا على قيمة اختيارية.

استخدم **المساواة =** إذا كان المطلوب فعلًا التحقق من عدم وجود العلاقة.

---

## Combining Conditions

### AND الافتراضية

عندما نضع Conditions متتالية داخل List، تجمعها Odoo باستخدام AND:

```python
domain = [
    ("active", "=", True),
    ("state", "=", "available"),
]
```

تعني:

```text
active = True
AND
state = available
```

يجب أن تحقق Record الشرطين معًا.

يمكن تنفيذ البحث:

```python
available_assets = equipment_model.search(domain)
```

---

### Logical Operators وPrefix Notation

للتعبير عن تركيب أوضح أوأكثر تعقيدًا، نستخدم:

| الرمز | العملية | عدد التعبيرات التي تعمل عليها |
| --- | --- | --- |
| **&** | AND | اثنان |
| **\|** | OR | اثنان |
| **!** | NOT | واحد |

في List Syntax، نكتب هذه الرموز كنصوص **قبل التعبيرات** التي تعمل عليها.

هذا هو المقصود بـ**Prefix Notation**.

مثلًا، رمز OR يأتي قبل الشرطين اللذين نريد جمعهما، كما سنطبق في المثال التالي.

---

### OR

للبحث عن الأجهزة المتاحة أوالتي يحتوي اسمها على Laptop:

```python
domain = [
    "|",
    ("state", "=", "available"),
    ("name", "ilike", "Laptop"),
]
```

تعني:

```text
state = available
OR
name يحتوي Laptop
```

يكفي تحقق واحد من الشرطين، ويمكن أن يتحققا معًا.

الرمز `"|"` يجمع التعبيرين التاليين له، وليس فاصلًا نضعه بين Conditions.

---

### NOT

نستخدم `"!"` لنفي تعبير واحد:

```python
domain = [
    "!",
    ("state", "=", "retired"),
]
```

تعني:

```text
NOT (state = retired)
```

في هذه الحالة، الصياغة التالية أبسط وتؤدي نفس الغرض:

```python
domain = [
    ("state", "!=", "retired"),
]
```

تظهر فائدة NOT أكثر عندما نريد نفي مجموعة شروط.

---

### AND الصريحة

يمكن كتابة AND بشكل صريح:

```python
domain = [
    "&",
    ("active", "=", True),
    ("state", "=", "available"),
]
```

وهي تعادل:

```python
domain = [
    ("active", "=", True),
    ("state", "=", "available"),
]
```

استخدم الشكل الأبسط عندما يكون المعنى واضحًا.

تصبح AND الصريحة مفيدة عند شرح المجموعات المركبة أوكتابتها.

---

### دمج AND وOR

نريد:

> الأجهزة النشطة التي تكون متاحة أوالتي يحتوي اسمها على Laptop.

نبدأ بالمنطق:

```text
A = active يساوي True
B = state يساوي available
C = name يحتوي Laptop

A AND (B OR C)
```

نبني المجموعة الداخلية أولًا:

```python
[
    "|",
    ("state", "=", "available"),
    ("name", "ilike", "Laptop"),
]
```

ثم نربطها بشرط `active`:

```python
domain = [
    "&",
    ("active", "=", True),
    "|",
    ("state", "=", "available"),
    ("name", "ilike", "Laptop"),
]
```

اقرأها بهذا الترتيب:

1. الرمز `"&"` يجمع تعبيرين.
2. التعبير الأول هو شرط `active`.
3. التعبير الثاني يبدأ بالرمز `"|"`، ويجمع شرط الحالة وشرط الاسم.

لا نضع Parentheses داخل List لتجميع Conditions؛ تركيب Operators هو الذي يحدد المجموعة.

ويمكن الاعتماد على AND الافتراضية وكتابتها أيضًا:

```python
domain = [
    ("active", "=", True),
    "|",
    ("state", "=", "available"),
    ("name", "ilike", "Laptop"),
]
```

الصياغتان تعبران عن نفس الطلب.

---

### OR بين ثلاثة شروط

لأن OR تجمع تعبيرين، نحتاج إلى تكرار الرمز عند جمع ثلاثة شروط باستخدام OR.

مثلًا، للبحث عن اسم يحتوي Laptop أوPrinter أوMonitor:

```python
domain = [
    "|",
    "|",
    ("name", "ilike", "Laptop"),
    ("name", "ilike", "Printer"),
    ("name", "ilike", "Monitor"),
]
```

تعني:

```text
(name يحتوي Laptop OR name يحتوي Printer)
OR
name يحتوي Monitor
```

وجود رمز OR واحد قبل ثلاثة شروط لا يعني أن الشروط الثلاثة كلها أصبحت OR.

---

### نفي مجموعة شروط

نريد الأجهزة التي لا يتحقق فيها أي من الشرطين:

- الحالة Retired.
- الاسم يحتوي Broken.

نكتب المنطق أولًا:

```text
NOT (
    state = retired
    OR
    name يحتوي Broken
)
```

ثم Domain:

```python
domain = [
    "!",
    "|",
    ("state", "=", "retired"),
    ("name", "ilike", "Broken"),
]
```

هنا NOT تنفي مجموعة OR كاملة.

الطريقة العملية لقراءة Domain مركبة هي تحديد كل Operator، ثم تحديد التعبيرات التي تعمل عليها، بدل قراءة List كسلسلة Conditions منفصلة.

---

## Domains with Relationships

### البحث عبر Many2one باستخدام Dot Notation

يمكن للـDomain أن تعبر علاقة Many2one للوصول إلى Field في Model المرتبطة.

سنستخدم مثالًا من Models الأساسية في Odoo:

| Model | Field | المعنى |
| --- | --- | --- |
| `res.partner` | `country_id` | Many2one تشير إلى `res.country` |
| `res.country` | `code` | رمز الدولة |

داخل Odoo Shell:

```python
partner_model = env["res.partner"]

partners = partner_model.search([
    ("country_id.code", "=", "CA"),
])
```

نحن نبحث في `res.partner`، لكن الشرط يقرأ Field من الدولة المرتبطة:

```text
res.partner
    ↓ country_id
res.country
    ↓ code
"CA"
```

كل جزء من المسار هو Technical Name.

نفس الفكرة تنطبق على:

```python
[
    ("department_id.name", "=", "Operations"),
]
```

بشرط أن تحتوي Model التي نبحث فيها على `department_id`، وأن تكون العلاقة مرتبطة بـModel تحتوي `name`.

لا يكفي كتابة مسار يبدو مناسبًا؛ يجب أن تكون Fields والعلاقات معرفة فعلًا.

---

### البحث باستخدام ID المرتبطة

يمكن مقارنة Many2one مباشرةً بـID الخاصة بالـRecord المرتبطة.

مثلًا:

```python
[
    ("department_id", "=", 7),
]
```

تعني أن Department المرتبطة لها ID تساوي `7`.

هذا الرقم توضيحي، وليس قيمة ثابتة يمكن الاعتماد عليها في كل Database.

البحث بالاسم قد يطابق أكثر من Record إذا تكررت الأسماء، بينما ID تحدد Record داخل Model المعنية.

---

### `any`: وجود Record مرتبطة تحقق الشرط

نستخدم `any` عندما نريد Records لديها **Record مرتبطة واحدة على الأقل** تحقق Domain أخرى.

في `res.partner`، تحتوي `child_ids` على Contacts المرتبطة عبر علاقة `parent_id`.

للبحث عن Partners لديها Contact مرتبطة تحتوي Email:

```python
partners = partner_model.search([
    (
        "child_ids",
        "any",
        [
            ("email", "!=", False),
        ],
    ),
])
```

لدينا مستويان:

| المستوى | Model التي يطبق عليها الشرط |
| --- | --- |
| البحث الخارجي | `res.partner` التي نريد إرجاعها |
| Domain داخل `any` | Contacts الموجودة عبر `child_ids` |

النتيجة هي Partners المطابقة، وليست قائمة Contacts الداخلية.

يكفي وجود Contact واحدة تحقق الشرط؛ لا يشترط أن تحققه كل Contacts.

---

### `not any`: عدم وجود Record مرتبطة تحقق الشرط

لنفس العلاقة:

```python
partners = partner_model.search([
    (
        "child_ids",
        "not any",
        [
            ("email", "!=", False),
        ],
    ),
])
```

تعني:

> لا توجد Contact مرتبطة تحقق شرط وجود Email.

قد تطابق Partner:

- ليس لديها Contacts مرتبطة أصلًا.
- لديها Contacts، لكن لا تحتوي أي واحدة منها على Email.

انتبه إلى الفرق:

**عدم وجود Contact تحقق الشرط ليس هو وجود Contact لا تحققه.**

---

### Domain على Relational Field

تعلمنا تعريف Many2one في [[07 Relationships]].

يمكن إضافة `domain` لتضييق الخيارات التي تقترحها الواجهة.

مثلًا، داخل Class الخاصة بـModel تحتوي Field للدولة:

```python
country_id = fields.Many2one(
    "res.country",
    domain=[
        ("code", "in", ["CA", "DE"]),
    ],
)
```

هذا التعريف يفترض استيراد `fields` في ملف Python، كما تعلمنا في [[04 Models & Fields]].

هنا تطبق Domain على **Model المرتبطة `res.country`**.

لذلك نستخدم `code` مباشرةً، ولا نكتب `country_id.code`.

الفرق بين الاستخدامين:

| الاستخدام | Model التي تنتمي إليها Fields داخل Domain |
| --- | --- |
| `partner_model.search(domain)` | `res.partner` |
| Parameter باسم `domain` على Many2one إلى `res.country` | `res.country` |

> [!important] تضييق الاختيارات ليس ضمانًا لصحة كل كتابة
> Domain على Relational Field تساعد الواجهة على اقتراح Records مناسبة.
>
> لا تعتبرها وحدها Server Constraint تمنع إسناد قيمة أخرى من Python أوAPI، ولا بديلًا عن الصلاحيات.

سنشرح Constraints في [[17 Constraints]]، والصلاحيات في [[11 Security & Multi-Company]].

---

## Building Domains with Domain Class

### إنشاء Domain Object

في Odoo 20، يمكن بناء Domains باستخدام `Domain` Class بدل كتابة List يدويًا في كل مرة.

داخل Odoo Shell، نستوردها أولًا:

```python
from odoo.fields import Domain
```

ثم ننشئ شرطًا:

```python
available_domain = Domain(
    "state",
    "=",
    "available",
)
```

`available_domain` هي Variable تحتوي Domain Object؛ ليست نتيجة بحث.

نمررها إلى عملية البحث:

```python
available_assets = equipment_model.search(available_domain)
```

المعنى هو نفسه الذي عبرنا عنه سابقًا باستخدام List Syntax.

---

### دمج Domain Objects

تدعم Domain Objects تركيب الشروط باستخدام Python Operators:

| العملية | الرمز |
| --- | --- |
| AND | **&** |
| OR | **\|** |
| NOT | **~** |

مثلًا:

```python
active_domain = Domain("active", "=", True)
available_domain = Domain("state", "=", "available")
laptop_domain = Domain("name", "ilike", "Laptop")
```

لبناء:

```text
active = True
AND
(state = available OR name يحتوي Laptop)
```

نكتب:

```python
domain = active_domain & (
    available_domain | laptop_domain
)

assets = equipment_model.search(domain)
```

هنا نستطيع استخدام Parentheses المعتادة لتنظيم Python Expression.

للنفي:

```python
not_retired_domain = ~Domain(
    "state",
    "=",
    "retired",
)
```

انتبه إلى الفرق بين الصيغتين:

| الجانب | List Syntax | Domain Objects |
| --- | --- | --- |
| Operators | نصوص داخل List | Python Operators بين Objects |
| طريقة التركيب | Prefix Notation | Python Expression |
| النفي | الرمز **!** كنص | الرمز **~** |

استخدم Operators الموضحة لتركيب Domain Objects، وليس Python Keywords مثل `and` و`or` و`not`.

هذه العمليات تختلف أيضًا عن Set Operations على Recordsets في [[06 Records & Recordsets]]:

- هنا نبني **شروطًا**.
- هناك كنا نجمع **Records موجودة**.

---

### دمج مجموعة Domains

يمكن استخدام `Domain.AND()` و`Domain.OR()` عندما تكون لدينا مجموعة شروط جاهزة.

مثلًا:

```python
domain = Domain.AND([
    active_domain,
    available_domain,
])
```

تعني تحقق الشرطين.

أما:

```python
domain = Domain.OR([
    available_domain,
    laptop_domain,
])
```

فتعني تحقق واحد منهما على الأقل.

هذه الطريقة مفيدة عندما نجمع Conditions في قائمة أثناء بناء البحث.

---

### التحويل بين الصيغتين

يمكن إنشاء Domain Object من List:

```python
domain = Domain([
    ("active", "=", True),
    ("state", "=", "available"),
])
```

ويمكن تحويل Domain Object إلى List باستخدام `list()`:

```python
domain_list = list(domain)
```

هنا `list()` تستخدم للحصول على List Representation للـDomain؛ لا تنفذ بحثًا ولا تعيد Records.

قد تتضمن النتيجة Logical Operators صريحة حتى لو لم نكتبها في القائمة الأصلية.

المهم هو المنطق الذي تمثله.

---

### أي صيغة أستخدم؟

| الحالة | الاختيار المناسب |
| --- | --- |
| شروط قصيرة وثابتة | List Syntax واضحة ومباشرة |
| تركيب عدة مجموعات شروط في Python | Domain Class قد تكون أسهل للقراءة |
| قراءة Code موجودة أوتعريفات XML | فهم List Syntax أساسي |

لا تحتاج إلى تحويل كل Domain بسيطة إلى Domain Class.

اختر الصيغة التي تجعل الشروط أوضح في مكان استخدامها.

---

## Example

سنطبق ما تعلمناه على بيانات صغيرة حتى نستطيع توقع نتيجة البحث.

الأمثلة التالية تكتب داخل **Odoo Shell**، وتستخدم `equipment.asset` المعرفة في
[[04 Models & Fields]].

---

### 1. تجهيز Records للتجربة

سننشئ أربع Records باستخدام `create()` التي تعلمناها في [[05 ORM]].

نفّذ هذا الجزء مرة واحدة أثناء التجربة؛ تكراره ينشئ Records إضافية.

```python
equipment_model = env["equipment.asset"]

demo_assets = equipment_model.create([
    {
        "name": "Training Laptop",
        "state": "in_use",
        "active": True,
    },
    {
        "name": "Office Printer",
        "state": "available",
        "active": True,
    },
    {
        "name": "Desk Monitor",
        "state": "retired",
        "active": True,
    },
    {
        "name": "Spare Laptop",
        "state": "available",
        "active": False,
    },
])
```

البيانات التي سنختبر عليها:

| الاسم | State | Active |
| --- | --- | --- |
| Training Laptop | `in_use` | `True` |
| Office Printer | `available` | `True` |
| Desk Monitor | `retired` | `True` |
| Spare Laptop | `available` | `False` |

احتفظ بـ`demo_assets`؛ سنستخدم IDs الخاصة بها لتقييد البحث على Records التي أنشأناها في هذه التجربة.

---

### 2. تحديد المطلوب

نريد:

> Records التجربة النشطة، بشرط أن تكون متاحة أوأن يحتوي اسمها على Laptop.

نفصل الطلب إلى ثلاثة أجزاء:

```text
Record ضمن بيانات التجربة
AND
active = True
AND
(
    state = available
    OR
    name يحتوي Laptop
)
```

النتيجة المتوقعة:

| Record | هل تدخل النتيجة؟ | السبب |
| --- | --- | --- |
| Training Laptop | نعم | نشطة واسمها يحتوي Laptop |
| Office Printer | نعم | نشطة ومتاحة |
| Desk Monitor | لا | ليست متاحة واسمها لا يحتوي Laptop |
| Spare Laptop | لا | غير نشطة |

---

### 3. بناء Domain

نبدأ بشرط حصر البحث على بيانات التجربة:

```python
("id", "in", demo_assets.ids)
```

`demo_assets.ids` توفر IDs الخاصة بالـRecords التي أنشأناها، كما تعلمنا في [[06 Records & Recordsets]].

ثم نضيف شرط النشاط ومجموعة OR:

```python
domain = [
    ("id", "in", demo_assets.ids),
    ("active", "=", True),
    "|",
    ("state", "=", "available"),
    ("name", "ilike", "Laptop"),
]
```

Conditions الموجودة خارج مجموعة OR ترتبط بها باستخدام AND الافتراضية.

شرط IDs هنا خاص بعزل بيانات المثال؛ ليس جزءًا مطلوبًا في كل بحث فعلي عن الأجهزة.

---

### 4. تنفيذ البحث والتحقق

نفذ البحث:

```python
matching_assets = equipment_model.search(
    domain,
    order="name, id",
)
```

ثم اعرض الأسماء باستخدام `mapped()`:

```python
matching_assets.mapped("name")
```

النتيجة المتوقعة:

```python
[
    "Office Printer",
    "Training Laptop",
]
```

استخدمنا `order` لتكون نتيجة العرض مرتبة.

الترتيب ليس جزءًا من Domain:

- Domain تحدد Records المطابقة.
- `order` يحدد ترتيب النتائج.

---

### 5. كتابة نفس الشروط باستخدام Domain Class

نستورد `Domain` ثم نبني نفس التعبير:

```python
from odoo.fields import Domain

domain_object = (
    Domain("id", "in", demo_assets.ids)
    & Domain("active", "=", True)
    & (
        Domain("state", "=", "available")
        | Domain("name", "ilike", "Laptop")
    )
)
```

ننفذ البحث:

```python
matching_assets = equipment_model.search(
    domain_object,
    order="name, id",
)

matching_assets.mapped("name")
```

النتيجة المتوقعة هي نفس الاسمين السابقين.

غيرنا طريقة بناء الشروط، ولم نغير منطق البحث.

---

### 6. مراجعة Domain عندما تختلف النتيجة

إذا اختلفت النتيجة عما تتوقع، راجع بالترتيب:

1. هل تبحث في Model الصحيحة؟
2. هل Technical Names للـFields صحيحة؟
3. هل Values مناسبة لأنواع Fields وقيم Selection؟
4. هل تريد AND أم OR؟
5. ما التعبيرات التي تعمل عليها كل Logical Operator؟
6. هل توجد Records مؤرشفة أوقيود صلاحيات تؤثر في النتيجة؟

ابدأ باختبار مجموعة صغيرة من الشروط، ثم أضف بقية المجموعات.

Domain الطويلة تصبح أسهل عندما تعرف نتيجة كل جزء منها.

---

## Important Notes

- **Domain تصف الشروط ولا تنفذ عملية بنفسها.** تمريرها إلى عملية البحث هو الذي ينفذ البحث.

- **Empty Domain لا تلغي الصلاحيات ولا تعيد المؤرشف تلقائيًا.** النتيجة ما زالت تخضع لسلوك البحث والقيود المطبقة.

- **استخدم Technical Names والقيم المخزنة.** مثل `available` بدل Label مثل `Available`.

- **AND وOR الصريحتان تجمع كل منهما تعبيرين.** لا تفترض أن Operator واحدة تشمل كل Conditions التي تأتي بعدها.

- **المقارنة الاختيارية =? مع `False` لا تبحث عن علاقة فارغة.** استخدم **المساواة =** إذا كان المطلوب التحقق من عدم وجود العلاقة.

- **Domain على Relational Field تطبق على Model المرتبطة.** وهي لا تكفي وحدها كـServer Constraint أوكقاعدة حماية.

- **`not any` لا تعني وجود Record مرتبطة لا تحقق الشرط.** تعني عدم وجود Record مرتبطة تحقق الشرط، وقد تطابق علاقة فارغة.

- **استخدم Domain لتضييق البحث من البداية.** لا تحمل كل Records ثم تستخدم `filtered()` في Python عندما يمكن التعبير عن المطلوب مباشرةً في البحث.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[09 Environment & Context]]

---

## Sources

- [Odoo 20 — Search Domains](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#search-domains)
- [Odoo 20 — ORM Search and Read](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#search-read)
- [Odoo 20 — Relational Fields](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#relational-fields)
- [Odoo 20 Source — Domain Expressions](https://github.com/odoo/odoo/blob/20.0/odoo/orm/domains.py)
- [Odoo 20 Source — ORM Search Implementation](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)
- [Odoo 20 Source — Relational Fields](https://github.com/odoo/odoo/blob/20.0/odoo/orm/fields_relational.py)
- [Odoo 20 Source — Domain Import](https://github.com/odoo/odoo/blob/20.0/odoo/fields/__init__.py)