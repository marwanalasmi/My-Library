---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 09 Environment & Context

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Environment Fundamentals|Environment Fundamentals]]
- [[#Working with Context|Working with Context]]
- [[#Changing User & Company Context|Changing User & Company Context]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

استخدمنا في الشابترات السابقة `env` للوصول إلى Models وتنفيذ عمليات ORM.

مثلًا:

```python
equipment_model = env["equipment.asset"]
```

لكن عملية ORM لا تعتمد على اسم Model والـRecords فقط. تحتاج أيضًا إلى معرفة المستخدم الذي ينفذها، والـContext التي تعمل بها، والاتصال المستخدم مع Database.

هذه المعلومات تجمعها **Environment**.

في هذا الشابتر سنتعلم:

- العلاقة بين Recordset وEnvironment.
- الفرق بين Environment وContext.
- قراءة معلومات المستخدم والشركة.
- تعديل Context وتأثيرها في البحث والقيم الافتراضية.
- الفرق بين تغيير المستخدم وتفعيل Superuser Mode.
- حدود تغيير Company Context.

سنستخدم أمثلة من المفاهيم التي تعلمناها، مع تأجيل تفاصيل Security وMulti-Company إلى [[11 Security & Multi-Company]].

---

## Environment Fundamentals

### ما هي Environment؟

الـ**Environment** تجمع المعلومات والخدمات التي تحتاجها ORM أثناء التنفيذ.

كل Recordset مرتبطة بـEnvironment، وتصل إليها من خلال الخاصية `env`.

الصورة العامة:

```text
Recordset
    ↓
Environment
    ├── Current User
    ├── Context
    ├── Superuser Mode
    ├── Database Cursor
    └── الوصول إلى Models والخدمات الداخلية
```

تستطيع Environment أيضًا توفير معلومات مثل الشركة الحالية والشركات المفعلة، اعتمادًا على المستخدم والـContext.

لذلك، قد يختلف سلوك نفس عملية البحث عندما ننفذها بمستخدم آخر أوContext مختلفة.

---

### `env` و`self.env`

داخل **Odoo Shell** توفر Odoo Variable باسم `env`:

```python
equipment_model = env["equipment.asset"]
```

أما داخل **Model Method**، فنصل إلى Environment المرتبطة بـ`self`:

```python
equipment_model = self.env["equipment.asset"]
```

الفرق في مكان الوصول إليها، وليس في وجود نوعين مختلفين من Environment.

| المكان | طريقة الوصول |
| --- | --- |
| Odoo Shell | `env` |
| داخل Model Method | `self.env` |
| من Recordset موجودة | `records.env` |

كما تعلمنا في [[06 Records & Recordsets]]، فإن `self` داخل Model Method هي Recordset.

في أمثلة هذا الشابتر سنستخدم **Odoo Shell**، إلا إذا ذكرنا غير ذلك.

---

### الوصول إلى Models

تدعم Environment الوصول إلى Models باستخدام Technical Name:

```python
equipment_model = env["equipment.asset"]
```

هذه الكتابة تشبه الوصول إلى قيمة داخل Dictionary، لكن Environment توفر أكثر من مجرد تخزين قيم.

الناتج هنا Recordset فارغة تمثل Model، ويمكن استخدامها لتنفيذ عمليات مثل البحث والإنشاء.

مثلًا:

```python
available_assets = equipment_model.search([
    ("state", "=", "available"),
])
```

Recordset الناتجة مرتبطة أيضًا بـEnvironment:

```python
available_assets.env
```

هذا الارتباط هو ما يسمح باستمرار معلومات التنفيذ عند التعامل مع Records.

---

### المستخدم الحالي وSuperuser Mode

من أهم المعلومات التي توفرها Environment:

| الخاصية | ما تعيده |
| --- | --- |
| `env.uid` | ID للمستخدم الحالي |
| `env.user` | Record تمثل المستخدم الحالي من `res.users` |
| `env.su` | هل التنفيذ في Superuser Mode؟ |

داخل Shell:

```python
current_user_id = env.uid
current_user = env.user
superuser_mode = env.su
```

يمكن فحص معلومات المستخدم:

```python
current_user.name
current_user.id
```

ويمكن التحقق من وضع التنفيذ:

```python
env.su
```

التمييز مهم؛ **هوية المستخدم وSuperuser Mode ليستا نفس الشيء**.

سنرى ذلك عند شرح `sudo()`.

> [!important] انتبه إلى سياق Shell
> Odoo Shell قد تعمل في Superuser Mode.
>
> لذلك، نجاح عملية داخل Shell لا يثبت أن مستخدمًا عاديًا يملك صلاحية تنفيذها. تحقق من المستخدم ووضع التنفيذ قبل تفسير نتيجة اختبار الصلاحيات.

---

### الشركة الحالية والشركات المفعلة

توفر Environment:

```python
current_company = env.company
enabled_companies = env.companies
```

| الخاصية | المعنى |
| --- | --- |
| `env.company` | الشركة الحالية للعملية |
| `env.companies` | Recordset بالشركات المفعلة في Environment |

مثلًا:

```python
current_company.name
enabled_companies.mapped("name")
```

لا نخلط بين:

- الشركات التي يسمح للمستخدم بالوصول إليها.
- الشركات المفعلة حاليًا في Environment.
- الشركة الحالية التي تعتمد عليها العملية.

تستخدم Odoo معلومات من Context، ومنها `allowed_company_ids`، لتحديد الشركات في سياق التنفيذ. وعند غيابها، تستخدم القيم الافتراضية المرتبطة بالمستخدم.

يكفي الآن فهم الفرق بين المفاهيم الثلاثة. تفاصيل قواعد الوصول والعزل بين الشركات في
[[11 Security & Multi-Company]].

---

### Cursor وRegistry وCache

قد تقابل هذه المصطلحات عند قراءة Code أوالتوثيق:

| المصطلح | دوره العام |
| --- | --- |
| Database Cursor | الوصول إلى Database ضمن Transaction الحالية |
| Model Registry | توفير Models المعروفة للـDatabase |
| Cache | تقليل تكرار بعض عمليات القراءة والعمل الداخلي |

نصل إلى Cursor من خلال:

```python
env.cr
```

داخل Model Method تصبح:

```python
self.env.cr
```

لا نحتاج إلى إدارة هذه الأجزاء يدويًا في العمليات المعتادة؛ نستخدم ORM.

سبق شرح Transactions وحدود التعامل المباشر مع Database في [[05 ORM]]، وسنتوسع في موضوعات الأداء عند الوصول إلى [[28 Performance]].

---

## Working with Context

### ما هي Context؟

الـ**Context** هي Mapping من Keys وValues تحمل معلومات إضافية قد تؤثر في تنفيذ العملية.

نصل إليها من خلال:

```python
env.context
```

إذن:

```text
Environment
    └── Context
```

Context جزء من Environment، وليستا اسمين لنفس الشيء.

من Keys الشائعة:

| Key | الاستخدام |
| --- | --- |
| `lang` | تحديد اللغة المستخدمة حيث يدعم السلوك ذلك |
| `tz` | معلومات المنطقة الزمنية المستخدمة في العمليات التي تعتمد عليها |
| `active_test` | التحكم في استبعاد Records المؤرشفة أثناء البحث المعتاد |
| `default_<field_name>` | تقديم Default Value لـField |

وقد تحتوي Context على Keys إضافية تقرؤها Modules أوMethods معينة.

وجود Key وحده لا ينشئ Behavior؛ يجب أن توجد Code تتعامل معها.

---

### قراءة قيم Context

نستخدم `get()` لقراءة قيمة:

```python
language = env.context.get("lang")
```

إذا لم توجد Key، تعيد `get()` القيمة `None`.

ويمكن تقديم قيمة بديلة:

```python
active_test = env.context.get(
    "active_test",
    True,
)
```

هنا نقرأ `active_test`، ونستخدم `True` إذا لم تكن موجودة.

يمكن أيضًا استخدام Key خاصة بالـModule:

```python
show_details = env.context.get(
    "equipment_training_show_details",
    False,
)
```

اسم `equipment_training_show_details` اخترناه كمثال، وليس Key خاصة مدمجة في Odoo.

إذا لم توجد Code تقرأ هذه القيمة وتستخدمها، فلن تغير طريقة عرض التفاصيل تلقائيًا.

---

### Context ليست بديلًا عن Fields

نستخدم Context لتمرير معلومات إضافية للتنفيذ.

أما بيانات الأعمال التي نريد حفظها والرجوع إليها، فمكانها المعتاد Fields على Models.

مثلًا:

| المعلومة | المكان المناسب عادةً |
| --- | --- |
| حالة الجهاز | Field مثل `state` |
| الرقم التسلسلي | Field مثل `serial_number` |
| طلب تضمين المؤرشف في بحث معين | Context |
| Default State عند إنشاء Record | Context أوتعريف Default حسب الحاجة |

Context ليست مخزنًا دائمًا لبيانات Record.

قد تؤثر قيمها في البيانات التي تنشئها العملية، لكن تمريرها وحده لا يعني حفظ كل محتوياتها داخل Database.

---

### تعديل Context باستخدام `with_context()`

Context المرتبطة بـEnvironment لا نعدلها مباشرةً.

نستخدم `with_context()` للحصول على Recordset مرتبطة بالـContext المطلوبة:

```python
equipment_model = env["equipment.asset"]

inclusive_equipment_model = equipment_model.with_context(
    active_test=False,
)
```

الآن يمكن استخدام الناتج في البحث:

```python
assets = inclusive_equipment_model.search(
    [],
    limit=5,
)
```

ما الذي حدث؟

1. بدأنا من `equipment_model`.
2. طلبنا Context تختلف في قيمة `active_test`.
3. احتفظنا بالناتج داخل `inclusive_equipment_model`.
4. نفذنا البحث من خلال الناتج.

لم نغير Field اسمها `active`، ولم ننشئ Equipment جديدة.

> [!important] استخدم القيمة المعادة
> `with_context()` لا تعدل Recordset الأصلية مباشرةً.
>
> التغيير يرافق Recordset الناتجة والعمليات التي تنفذ من خلالها.

يمكن كتابة الاستدعاء متصلًا أيضًا:

```python
assets = equipment_model.with_context(
    active_test=False,
).search(
    [],
    limit=5,
)
```

الصياغتان تطبقان نفس الفكرة.

---

### إضافة قيم إلى Context الحالية

عند استخدام Keyword Arguments، تدمج Odoo القيم مع Context الحالية:

```python
details_model = equipment_model.with_context(
    equipment_training_show_details=True,
)
```

ثم يمكن إضافة قيمة أخرى:

```python
inspection_model = details_model.with_context(
    equipment_training_stage="inspection",
)
```

الـContext الناتجة تحتفظ بالقيمة الأولى وتضيف الثانية:

```python
inspection_model.env.context.get(
    "equipment_training_show_details",
)
```

النتيجة:

```python
True
```

ولقراءة القيمة الثانية:

```python
inspection_model.env.context.get(
    "equipment_training_stage",
)
```

النتيجة:

```python
"inspection"
```

هذه Keys توضيحية خاصة بالمثال؛ لا تضيف سلوكًا إلى Model بمجرد تمريرها.

---

### تمرير Dictionary صريحة

يمكن أيضًا تمرير Dictionary إلى `with_context()`:

```python
replacement_model = details_model.with_context({
    "equipment_training_stage": "inspection",
})
```

هنا تستخدم Dictionary المقدمة كأساس للـContext، بدل دمج جميع القيم الحالية تلقائيًا كما في Keyword Arguments.

مثلًا:

```python
replacement_model.env.context.get(
    "equipment_training_show_details",
)
```

النتيجة:

```python
None
```

لأننا لم نضع هذه Key في Dictionary الجديدة.

لذلك، تمرير Dictionary صريحة قد يفقد قيمًا كنت تحتاج الاحتفاظ بها.

> [!note] تفاصيل خاصة بالشركات
> في Odoo 20، تحافظ `with_context()` على `allowed_company_ids` الموجودة في Context السابقة إذا لم تتضمنها Dictionary المقدمة.
>
> لذلك لا نفترض أن تمرير Dictionary فارغة يمسح كل حالة التنفيذ، ولا نستخدمه كطريقة عامة لإعادة ضبط المستخدم أوالشركات أوالصلاحيات.

---

### `active_test` والبحث عن المؤرشف

في Models التي تستخدم `active`، تستبعد عمليات البحث المعتادة Records المؤرشفة افتراضيًا.

مثلًا:

```python
assets = equipment_model.search(
    [],
    limit=5,
)
```

لتضمين المؤرشف:

```python
assets = equipment_model.with_context(
    active_test=False,
).search(
    [],
    limit=5,
)
```

هذا التغيير:

- يسمح للبحث بتضمين Records المؤرشفة.
- لا يعيد تفعيلها.
- لا يغير قيمة `active`.
- لا يتجاوز الصلاحيات.

أما إذا كان المطلوب البحث عن المؤرشف فقط، فيمكن استخدام شرط صريح:

```python
archived_assets = equipment_model.search(
    [
        ("active", "=", False),
    ],
    limit=5,
)
```

في البحث القياسي، وجود شرط مباشر على `active` يجعل هذا المثال لا يحتاج إلى إضافة `active_test=False`.

---

### Domain وContext

تعلمنا في [[08 Domains]] أن Domain تصف شروط مطابقة Records.

Context تحمل معلومات إضافية قد تؤثر في سلوك العملية، ومنها البحث.

مثلًا:

```python
domain = [
    ("state", "=", "available"),
]
```

ثم:

```python
available_assets = equipment_model.with_context(
    active_test=False,
).search(domain)
```

الأدوار هنا مختلفة:

| الجزء | دوره |
| --- | --- |
| Domain | طلب الأجهزة التي حالتها `available` |
| Context | السماح بتضمين المؤرشف ضمن البحث |

النتيجة تبقى خاضعة للصلاحيات.

---

### Default Values عبر Context

يمكن استخدام النمط:

```text
default_<field_name>
```

لتقديم قيمة افتراضية عند إنشاء Record.

سنستخدم Model الأجهزة، التي تحتوي Selection Field باسم `state`.

داخل Shell:

```python
in_use_equipment_model = equipment_model.with_context(
    default_state="in_use",
)
```

ثم ننشئ Record دون تمرير `state`:

```python
asset = in_use_equipment_model.create({
    "name": "Training Laptop",
})
```

نتحقق:

```python
asset.state
```

النتيجة المتوقعة:

```python
"in_use"
```

في سلوك ORM القياسي، تستخدم Context Default عند الحاجة إلى قيمة لم نمررها صراحةً.

أما إذا مررنا `state` داخل `create()`:

```python
asset = in_use_equipment_model.create({
    "name": "Training Printer",
    "state": "available",
})
```

فتكون النتيجة:

```python
asset.state
```

```python
"available"
```

القيمة الصريحة في بيانات الإنشاء لها الأولوية.

> [!info] Default ليست قاعدة إلزامية
> `default_state` تقدم قيمة افتراضية.
>
> لا تمنع اختيار قيمة أخرى، ولا تغير State للـRecords الموجودة.

تستخدم نفس الفكرة أيضًا عند فتح Forms وWizards، لكن تفاصيل Actions وWizards ستأتي في [[13 Actions, Menus & Buttons]] و[[19 Wizards]].

---

### حدود تأثير Context

التعديل يرافق Recordset الناتجة، وليس كل Environment في النظام.

مثلًا:

```python
inclusive_equipment_model = equipment_model.with_context(
    active_test=False,
)
```

هذا لا يغير Context الخاصة بـ`equipment_model` الأصلية.

لكن طالما نستخدم `inclusive_equipment_model` أوRecordsets مشتقة منها، فقد تنتقل Context إلى العمليات التالية.

لذلك:

- استخدم Variable واضحة للسياق المعدل.
- مرر Keys التي تحتاجها فقط.
- لا توسع تأثير Default أوFlag دون حاجة.

كذلك، `with_context()` لا تنشئ Database Transaction مستقلة. تغيير Context لا يعزل عمليات الكتابة في Transaction جديدة.

---

## Changing User & Company Context

### الفرق بين العمليات

هذه APIs تعيد Recordset مرتبطة بسياق التنفيذ المطلوب:

| API | ما الذي تغيره؟ |
| --- | --- |
| `with_context()` | قيم Context |
| `with_user()` | المستخدم المرتبط بالتنفيذ |
| `sudo()` | Superuser Mode |
| `with_company()` | الشركة الحالية ضمن Company Context |

لا نعتبرها بدائل تؤدي نفس الوظيفة.

---

### تغيير المستخدم باستخدام `with_user()`

تستخدم `with_user()` لتنفيذ العمليات من خلال Recordset مرتبطة بمستخدم محدد.

لتطبيق المثال، استخدم Login لحساب موجود فعلًا في Database التدريب.

الاسم التالي توضيحي:

```python
target_user = env["res.users"].search(
    [
        ("login", "=", "training.operator@example.com"),
    ],
    limit=1,
)
```

نتحقق من وجود المستخدم قبل المتابعة:

```python
if not target_user:
    raise ValueError(
        "Select an existing user before continuing."
    )
```

ثم نشتق Recordset مرتبطة به:

```python
user_equipment_model = equipment_model.with_user(
    target_user,
)
```

يمكن فحص هوية التنفيذ:

```python
user_equipment_model.env.uid == target_user.id
```

النتيجة المتوقعة:

```python
True
```

وعند تنفيذ البحث:

```python
user_assets = user_equipment_model.search(
    [],
    limit=5,
)
```

تتأثر العملية بصلاحيات المستخدم المختار.

قد تعيد Records مسموحة، أوترفع `AccessError` إذا لم يكن الوصول المطلوب مسموحًا.

> [!important] تحقق من المستخدم قبل التبديل
> في Odoo 20، تمرير Recordset فارغة إلى `with_user()` يبقي Recordset الأصلية.
>
> لذلك، البحث الذي لم يجد حسابًا قد يجعل التجربة تستمر بالمستخدم الأصلي، فتكون نتيجة اختبار الصلاحيات مضللة.

تعمل `with_user()` عادةً دون Superuser Mode، باستثناء حساب Superuser التقني الذي يعمل بهذا الوضع بطبيعته.

سنشرح اختبارات الصلاحيات بشكل أوسع في [[11 Security & Multi-Company]] و[[26 Testing]].

---

### Superuser Mode باستخدام `sudo()`

تستخدم `sudo()` لتفعيل Superuser Mode:

```python
sudo_equipment_model = equipment_model.sudo()
```

هذا الوضع يتجاوز فحوص الوصول المعتادة، لكنه **لا يغير المستخدم الحالي**.

يمكن ملاحظة الفرق:

```python
sudo_equipment_model.env.uid == equipment_model.env.uid
```

النتيجة:

```python
True
```

وفي الوقت نفسه:

```python
sudo_equipment_model.env.su
```

النتيجة:

```python
True
```

إذن:

```text
with_user()
→ تغيير المستخدم

sudo()
→ تغيير وضع فحوص الوصول
```

ولا تعني `sudo()` أن Business Logic نفسها توقفت عن العمل.

> [!warning] لا تستخدم `sudo()` كحل تلقائي
> إذا ظهرت `AccessError`، حدد أولًا هل المستخدم يجب أن يملك الوصول المطلوب.
>
> استخدام `sudo()` دون فهم قد يسمح بالوصول إلى بيانات يفترض عزلها، ومنها بيانات شركات أخرى.

تفاصيل استخدامها وحدودها في [[11 Security & Multi-Company]].

---

### تغيير الشركة باستخدام `with_company()`

تستخدم `with_company()` للحصول على Recordset تعمل بشركة حالية محددة.

نمرر إليها Record تمثل شركة أوID مناسبة.

مثلًا، يمكن داخل Shell قراءة أسماء الشركات المفعلة مع جعل كل واحدة منها الشركة الحالية أثناء دورتها:

```python
for company in env.companies:
    company_equipment_model = equipment_model.with_company(
        company,
    )

    print(company_equipment_model.env.company.name)
```

إذا كانت هناك شركة واحدة فقط، ينفذ المثال دورة واحدة.

الغرض هنا توضيح سياق التنفيذ، وليس إنشاء شركات أوتعديل بياناتها.

ننتبه إلى ثلاث نقاط:

- تصبح الشركة المحددة هي `env.company` في الناتج.
- تضم `env.companies` الشركة المحددة إلى الشركات المفعلة السابقة؛ لا تتحول بالضرورة إلى شركة واحدة فقط.
- تغيير الشركة لا يمنح المستخدم صلاحية الوصول إليها تلقائيًا.

كذلك، لا تنقل `with_company()` Records من شركة إلى أخرى، ولا تعدل Field باسم `company_id` بمجرد استدعائها.

سنشرح قواعد Multi-Company في [[11 Security & Multi-Company]].

---

### Context والصلاحيات

قد تصل بعض قيم Context من Client، أوتمر عبر عمليات أخرى.

لذلك، وجود Flag في Context لا يثبت أن المستخدم مسموح له بتنفيذ عملية حساسة.

مثلًا، Key تحمل معنى «اسمح بالموافقة» ليست وحدها أساسًا كافيًا لمنح الموافقة.

نستخدم Context لتوجيه السلوك المناسب، بينما تطبق Server الصلاحيات وقواعد الأعمال المطلوبة.

---

## Example

سنستخدم مثالًا صغيرًا يجمع فكرتين:

1. تقديم Default Value عبر Context.
2. تغيير البحث ليشمل Records المؤرشفة.

الأمثلة داخل **Odoo Shell**، وتستخدم `equipment.asset` المعرفة في [[04 Models & Fields]].

نفذها داخل نفس الجلسة حتى تبقى Variables متاحة. سلوك حفظ التغييرات وTransaction في Shell كما شرحنا في [[05 ORM]].

---

### 1. تجهيز Model بقيمة افتراضية

نبدأ من Model:

```python
equipment_model = env["equipment.asset"]
```

ثم نجهز Recordset تقدم Default State:

```python
training_equipment_model = equipment_model.with_context(
    default_state="in_use",
)
```

هذه الخطوة لا تنشئ أي Equipment بعد. هي تجهز السياق الذي سنستخدمه في الإنشاء.

---

### 2. إنشاء بيانات التجربة

نفذ هذا الجزء مرة واحدة؛ تكراره ينشئ Records إضافية.

```python
demo_assets = training_equipment_model.create([
    {
        "name": "Context Laptop",
        "active": True,
    },
    {
        "name": "Context Printer",
        "state": "available",
        "active": True,
    },
    {
        "name": "Context Monitor",
        "active": False,
    },
])
```

النتيجة المتوقعة:

| Record | State | Active | سبب اختيار State |
| --- | --- | --- | --- |
| Context Laptop | `in_use` | `True` | Context Default |
| Context Printer | `available` | `True` | قيمة صريحة في `create()` |
| Context Monitor | `in_use` | `False` | Context Default |

يمكن فحص البيانات:

```python
demo_assets.read([
    "name",
    "state",
    "active",
])
```

ستتضمن نتيجة `read()` أيضًا IDs الفعلية للـRecords.

---

### 3. حصر البحث على بيانات التجربة

نستخدم Domain حتى لا تدخل Records أخرى في النتيجة:

```python
domain = [
    ("id", "in", demo_assets.ids),
]
```

هذه Domain تطلب Records التي أنشأناها فقط، دون تحديد قيمة `active`.

---

### 4. البحث مع استبعاد المؤرشف

نجعل `active_test` مفعلة صراحةً لتكون التجربة واضحة:

```python
active_equipment_model = equipment_model.with_context(
    active_test=True,
)
```

ثم نبحث:

```python
active_assets = active_equipment_model.search(
    domain,
    order="name, id",
)
```

نعرض الأسماء:

```python
active_assets.mapped("name")
```

النتيجة المتوقعة:

```python
[
    "Context Laptop",
    "Context Printer",
]
```

لم تدخل `Context Monitor` لأنها مؤرشفة.

---

### 5. تضمين المؤرشف

نشتق سياقًا آخر:

```python
inclusive_equipment_model = equipment_model.with_context(
    active_test=False,
)
```

ثم ننفذ نفس Domain:

```python
all_assets = inclusive_equipment_model.search(
    domain,
    order="name, id",
)
```

نعرض الأسماء:

```python
all_assets.mapped("name")
```

النتيجة المتوقعة:

```python
[
    "Context Laptop",
    "Context Monitor",
    "Context Printer",
]
```

لم نغير Domain. غيرنا Context التي تعمل بها عملية البحث.

---

### 6. التحقق من أن Record لم تتغير

نبحث عن المؤرشفة ضمن بيانات التجربة:

```python
archived_assets = equipment_model.search([
    ("id", "in", demo_assets.ids),
    ("active", "=", False),
])
```

ثم نفحص قيمة `active`:

```python
archived_assets.mapped("active")
```

النتيجة المتوقعة:

```python
[
    False,
]
```

ظهور Record في البحث السابق لم يجعلها نشطة.

---

### 7. مقارنة السياقين

نقرأ القيمة من كل Recordset:

```python
active_equipment_model.env.context.get("active_test")
```

النتيجة:

```python
True
```

ثم:

```python
inclusive_equipment_model.env.context.get("active_test")
```

النتيجة:

```python
False
```

احتفظنا بسياقين مختلفين واستخدمنا كل واحد في العملية المناسبة.

هذا هو جوهر التعامل مع Environment وContext: تحديد سياق التنفيذ بوضوح، واستخدام Recordset المرتبطة به.

---

## Important Notes

- **Context جزء من Environment.** تغيير Context لا يعني تغيير المستخدم أوإنشاء اتصال وTransaction مستقلتين.

- **استخدم ناتج `with_context()`.** الاستدعاء لا يعدل Context الخاصة بالـRecordset الأصلية مباشرةً.

- **تمرير Dictionary صريحة يختلف عن إضافة Keyword Arguments.** قد تفقد قيمًا من Context الحالية إذا لم تنتبه إلى طريقة الدمج.

- **`active_test=False` لا تعيد تفعيل Records.** تغير سلوك البحث، ولا تتجاوز الصلاحيات.

- **Context Default ليست قيمة إلزامية.** القيمة المرسلة صراحةً في `create()` لها الأولوية في السلوك القياسي.

- **تحقق من وجود المستخدم قبل `with_user()`.** Recordset فارغة قد تبقي سياق المستخدم الأصلي.

- **`sudo()` لا تغير المستخدم الحالي.** هي تغير Superuser Mode، وقد تتجاوز حدود الوصول والعزل بين الشركات.

- **`with_company()` لا تمنح صلاحية ولا تنقل Records.** تغير Company Context، وقد تبقي شركات أخرى مفعلة معها.

- **لا تعتمد على Context القادمة من Client لمنح صلاحية حساسة.** تحقق من الوصول وقواعد الأعمال على Server.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[10 Data Files & External IDs]]

---

## Sources

- [Odoo 20 — Environment](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#environment)
- [Odoo 20 — Altering the Environment](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#altering-the-environment)
- [Odoo 20 — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Source — Environment Implementation](https://github.com/odoo/odoo/blob/20.0/odoo/orm/environments.py)
- [Odoo 20 Source — Recordset Environment APIs and Search](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)