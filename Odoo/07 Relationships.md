---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 07 Relationships

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Relationship Fundamentals|Relationship Fundamentals]]
- [[#Defining Relationships|Defining Relationships]]
- [[#Reading Related Records|Reading Related Records]]
- [[#Updating Relationships|Updating Relationships]]
- [[#Example|Example]]
- [[#اختيار العلاقة المناسبة|اختيار العلاقة المناسبة]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في الفصول السابقة تعلمنا تعريف Models وFields، واستخدام ORM، والتعامل مع Recordsets.

الخطوة التالية هي ربط البيانات ببعضها: الموظف ينتمي إلى قسم، والطلب يرتبط بعميل، والمنتج قد يرتبط بعدة Tags.

Relationships تمثل هذه الروابط داخل Odoo. في هذا الفصل سنتعلم اختيار نوع العلاقة المناسب، وتعريفها، وقراءة Records المرتبطة وتعديلها، مع فهم الفرق بين تغيير الرابط وحذف البيانات.

---

## Relationship Fundamentals

### لماذا نستخدم Relationships؟

لنفترض أننا نخزن اسم القسم داخل كل Employee باستخدام `Char`.

قد نجد القيم التالية:

```text
IT
Information Technology
I.T.
```

رغم أن المقصود قسم واحد.

المشكلة هنا أننا نخزن نصوصًا مستقلة، ولا ننشئ ارتباطًا فعليًا بقسم محدد. تغيير اسم القسم مثلًا قد يحتاج تعديل بيانات عدة Employees.

باستخدام Relationship، ننشئ Record مستقلة للقسم، ثم نربط الموظفين بها:

```text
Employee: Nora ──┐
                ├──→ Department: IT
Employee: Ahmed ──┘
```

الآن تشير Employee إلى Department Record، وتبقى بيانات القسم موجودة في مكان واحد.

### نفهم العلاقة من الجانبين

اختيار العلاقة يبدأ من معنى البيانات.

اسأل أولًا:

1. Record الحالية يمكن أن ترتبط بكم Record؟
2. Record في الطرف الآخر يمكن أن ترتبط بكم Record؟

سنستخدم ثلاثة Models في أمثلة الفصل:

| Model | ما تمثله |
|---|---|
| `company.department` | الأقسام |
| `company.employee` | الموظفون |
| `company.skill` | المهارات |

وقواعد المثال هي:

- Employee تنتمي إلى Department واحدة كحد أقصى.
- Department يمكن أن تضم عدة Employees.
- Employee يمكن أن تمتلك عدة Skills.
- Skill يمكن أن ترتبط بعدة Employees.

هذه قواعد اخترناها للمثال. لو اختلفت متطلبات العمل، فقد يختلف تصميم العلاقة.

### أنواع Relational Fields

توفر Odoo ثلاثة أنواع أساسية:

| النوع | معناه من جهة Record الحالية | مثال |
|---|---|---|
| `Many2one` | تشير إلى Record واحدة كحد أقصى | Employee → Department |
| `One2many` | تصل إلى Records التي تشير إليها عبر Many2one | Department → Employees |
| `Many2many` | تسمح بالتعدد في الجانبين | Employees ↔ Skills |

سنبدأ بتعريف العلاقات، ثم ننتقل إلى قراءتها وتعديلها.

---

## Defining Relationships

### مكان تعريف Fields

Relational Fields تُكتب داخل Python Classes الخاصة بالـModels، مثل Fields التي تعلمناها في [[04 Models & Fields]].

في هذا الفصل سنضع Models المثال داخل ملف واحد:

```text
relationship_training/
├── __init__.py
├── __manifest__.py
└── models/
    ├── __init__.py
    └── company.py
```

يبدأ ملف `company.py` بالاستيراد:

```python
from odoo import fields, models
```

وسنعرّف داخله Models الأقسام والموظفين والمهارات.

تعريفات Fields التالية تُكتب داخل Class المناسبة. وستجد التعريف الكامل للـModels، مع Imports المطلوبة، في قسم Example.

### Many2one

نريد أن تشير Employee إلى Department واحدة كحد أقصى.

داخل Model الموظف نكتب:

```python
department_id = fields.Many2one(
    "company.department",
    string="Department",
)
```

| الجزء | معناه |
|---|---|
| `department_id` | اسم Field داخل Employee |
| `fields.Many2one` | نوع العلاقة |
| `"company.department"` | Technical Name للـModel المرتبطة |
| `string` | الاسم المعروض للمستخدم |

أول Argument يمثل:

```text
comodel_name
```

أي اسم Model التي تشير إليها العلاقة.

المعنى من جهة الموظف:

```text
Employee واحدة
    │
    └──→ Department واحدة كحد أقصى
```

لكن عدة Employees يمكن أن تشير إلى Department نفسها؛ لذلك تسمى العلاقة `Many2one`.

إذا كانت Department إلزامية، نضيف:

```python
department_id = fields.Many2one(
    "company.department",
    string="Department",
    required=True,
)
```

في مثالنا سنترك العلاقة اختيارية، حتى يمكن إنشاء Employee لم تُعيّن لها Department بعد.

### One2many

من جهة Department، نريد الوصول إلى Employees الذين ينتمون إليها.

داخل Model القسم نكتب:

```python
employee_ids = fields.One2many(
    "company.employee",
    "department_id",
    string="Employees",
)
```

| Argument | معناه |
|---|---|
| `"company.employee"` | Model التي تحتوي Records المرتبطة |
| `"department_id"` | اسم Many2one الموجودة داخل Employee |

الـArgument الثاني يسمى `inverse_name`.

وهو يحدد Field التي تستخدمها Odoo لمعرفة Employees المرتبطين بالقسم:

```text
Employee.department_id
→ يحدد القسم الذي ينتمي إليه الموظف

Department.employee_ids
→ يعيد الموظفين الذين يشيرون إلى هذا القسم
```

**الجانبان يمثلان العلاقة نفسها.**

One2many لا تخزن قائمة IDs مستقلة داخل Department. الارتباط في هذا التصميم تحدده Many2one الموجودة داخل Employee.

إذا تغيرت `department_id` لدى موظف، تغيرت عضويته في `employee_ids` تبعًا لذلك.

ولا يشترط تعريف One2many لكل Many2one؛ نضيفها عندما نحتاج الوصول إلى العلاقة من الجانب الآخر.

### Many2many

نريد ربط Employees بالـSkills:

- الموظف يمكن أن يمتلك عدة مهارات.
- المهارة الواحدة يمكن أن ترتبط بعدة موظفين.

داخل Model الموظف نكتب:

```python
skill_ids = fields.Many2many(
    "company.skill",
    string="Skills",
)
```

العلاقة هنا متعددة من الجانبين:

```text
Employees ←──→ Skills
```

Many2many لا تعتمد على Many2one عكسية. تُحفظ الروابط في Relation Table وسيطة.

تصور مبسط لمحتويات الجدول:

| employee_id | skill_id |
|---|---|
| 1 | 10 |
| 1 | 20 |
| 2 | 10 |

Employee رقم `1` مرتبطة بمهارتين، وSkill رقم `10` مرتبطة بموظفين.

#### القراءة من الجانبين

تعريف `skill_ids` يكفي لاستخدام العلاقة من جهة Employee.

إذا احتجنا الوصول من Skill إلى Employees أيضًا، نضيف Field مقابلة تستخدم Relation Table نفسها.

في Employee:

```python
skill_ids = fields.Many2many(
    "company.skill",
    relation="company_employee_skill_rel",
    column1="employee_id",
    column2="skill_id",
    string="Skills",
)
```

وفي Skill:

```python
employee_ids = fields.Many2many(
    "company.employee",
    relation="company_employee_skill_rel",
    column1="skill_id",
    column2="employee_id",
    string="Employees",
)
```

| Parameter | وظيفتها |
|---|---|
| `relation` | اسم Relation Table |
| `column1` | العمود الذي يشير إلى Records الـModel الحالية |
| `column2` | العمود الذي يشير إلى Records الـModel المقابلة |

استخدمنا اسم الجدول نفسه، وبدّلنا العمودين لأن جهة العلاقة انعكست.

هذه Parameters اختيارية في العلاقات البسيطة بين Models مختلفة؛ يمكن لـOdoo توليدها تلقائيًا. كتبناها هنا صراحةً لتوضيح أن Fieldين تستخدمان الروابط نفسها.

### Naming Conventions

المعتاد استخدام الأسماء التالية:

| نوع Field | النهاية المعتادة | مثال |
|---|---|---|
| `Many2one` | `_id` | `department_id` |
| `One2many` | `_ids` | `employee_ids` |
| `Many2many` | `_ids` | `skill_ids` |

الاسم يساعد على فهم الكود، لكن نوع العلاقة يتحدد بتعريف Field، وليس بنهايتها.

---

## Reading Related Records

> [!info] مكان تنفيذ الأمثلة
> أمثلة القراءة والتعديل التالية تُنفذ في Odoo Shell بعد تحميل Models.
>
> نفترض أن `employee` و`department` و`skill` تمثل كل منها Singleton موجودة، وأن `employees` تمثل مجموعة موظفين. سننشئ البيانات عمليًا في Example.

### ماذا تعيد Relational Fields؟

عند الوصول مباشرةً إلى Relational Field، نحصل على Recordset.

إذا بدأنا من Singleton:

| التعبير | Model الناتجة | الحجم الممكن |
|---|---|---|
| `employee.department_id` | `company.department` | صفر أو واحدة |
| `department.employee_ids` | `company.employee` | صفر أو واحدة أو عدة |
| `employee.skill_ids` | `company.skill` | صفر أو واحدة أو عدة |

لهذا يعتمد التعامل مع العلاقات على فهم [[06 Records & Recordsets]].

### قراءة Many2one

للحصول على Department المرتبطة بالموظف:

```python
department = employee.department_id
```

النتيجة:

- Singleton إذا كانت Employee مرتبطة بقسم.
- Empty Recordset إذا لم تُعيّن لها Department.

يمكن التحقق قبل متابعة القراءة:

```python
if employee.department_id:
    print(employee.department_id.name)
```

التعبير:

```python
employee.department_id.name
```

يعني:

```text
Employee
    │
    └──→ Department المرتبطة
              │
              └──→ name
```

وللحصول على ID الخاصة بالقسم:

```python
employee.department_id.id
```

قيمة `department_id` عند الوصول إليها بهذه الطريقة Recordset، وليست مجرد Integer ID.

### قراءة One2many

للحصول على Employees داخل Department:

```python
employees = department.employee_ids
```

يمكن المرور عليهم:

```python
for employee in employees:
    print(employee.name)
```

أو جمع الأسماء:

```python
names = employees.mapped("name")
```

### قراءة Many2many

للحصول على Skills الخاصة بموظف:

```python
skills = employee.skill_ids
```

ثم قراءة أسمائها:

```python
names = skills.mapped("name")
```

ومن الجهة الأخرى، إذا عرّفنا Field المقابلة داخل Skill:

```python
employees = skill.employee_ids
```

### العلاقات مع عدة Records

إذا احتوت `employees` عدة موظفين، يمكن جمع Departments المرتبطة بهم:

```python
departments = employees.mapped("department_id")
```

ويمكن أيضًا كتابة:

```python
departments = employees.department_id
```

النتيجة قد تحتوي عدة Departments، لأن نقطة البداية عدة Employees.

هنا يظهر الفرق الذي أشرنا إليه في Chapter 06:

```python
employees.mapped("name")
```

تعيد List من الأسماء، بينما:

```python
employees.mapped("department_id")
```

تعيد Recordset من الأقسام المرتبطة، مع إزالة التكرارات.

لا تعتمد على ترتيب هذه الأقسام باعتباره مطابقًا لترتيب الموظفين.

إذا أردت أسماء الأقسام:

```python
department_names = departments.mapped("name")
```

أما `departments.name` فلا تصلح عندما تحتوي `departments` عدة Records؛ لأن `name` Field غير Relational.

---

## Updating Relationships

### تعيين Many2one

داخل `create()` و`write()` نمرر ID الخاصة بالـRecord المطلوبة:

```python
employee.write({
    "department_id": department.id,
})
```

ويمكن استخدام Field Assignment:

```python
employee.department_id = department.id
```

لتفريغ العلاقة الاختيارية:

```python
employee.write({
    "department_id": False,
})
```

هذا يفصل Employee عن Department، دون حذف أي منهما.

إذا كانت Field تستخدم `required=True`، فلا يمكن حفظها فارغة.

### أثر التغيير على One2many

إذا نقلنا Employee إلى Department أخرى:

```python
employee.write({
    "department_id": sales_department.id,
})
```

ستصبح ضمن:

```python
sales_department.employee_ids
```

ولن تعود ضمن Employees الخاصة بالقسم السابق.

لا نحتاج تحديث قائمة مستقلة في كل Department؛ فالجانبان يعكسان العلاقة نفسها.

### حذف Record المشار إليها: `ondelete`

`ondelete` في Many2one تحدد ما يحدث عندما تُحذف Record التي تشير إليها Field.

مثلًا:

```python
department_id = fields.Many2one(
    "company.department",
    string="Department",
    ondelete="restrict",
)
```

السؤال الذي تجيب عنه هنا:

**إذا حُذفت Department، ماذا يحدث للـEmployees الذين يشيرون إليها؟**

| القيمة | أثر حذف Department |
|---|---|
| `set null` | تفريغ `department_id` لدى Employees المرتبطين |
| `restrict` | منع حذف Department ما دام هناك Employees يشيرون إليها |
| `cascade` | حذف Employees الذين يشيرون إلى Department أيضًا |

`set null` تحتاج علاقة تسمح بالقيمة الفارغة؛ فلا تجمعها مع `required=True`.

سنستخدم `restrict` في مثالنا، لأن حذف Department لا ينبغي أن يحذف موظفيها تلقائيًا.

انتبه إلى اتجاه السلوك:

```text
حذف Department
→ تطبق سياسة ondelete على Employees الذين يشيرون إليها

حذف Employee
→ لا يعني حذف Department الخاصة بها
```

كذلك `restrict` لا تمنع نقل Employee إلى قسم آخر أو تفريغ علاقتها الاختيارية؛ فهي تتعلق بحذف Department المشار إليها.

### Commands الخاصة بالعلاقات المتعددة

داخل `create()` و`write()`، نستخدم قائمة Commands لتعديل `One2many` و`Many2many`.

نستورد:

```python
from odoo.fields import Command
```

داخل Odoo Shell نكتب الاستيراد قبل الاستخدام. وداخل Model File نضعه مع Imports في بداية الملف.

الشكل العام:

```python
employee.write({
    "skill_ids": [
        Command.link(skill.id),
    ],
})
```

الأجزاء هي:

| الجزء | معناه |
|---|---|
| `"skill_ids"` | Field التي نريد تعديلها |
| `[...]` | قائمة Commands المطلوبة |
| `Command.link(skill.id)` | وصف عملية ربط Skill موجودة |

استدعاء Command وحدها لا يطبق العملية على Employee؛ التغيير يحدث عند تمريرها إلى ORM.

#### إضافة رابط: `Command.link()`

لربط Skill موجودة مع الاحتفاظ بالروابط الحالية:

```python
employee.write({
    "skill_ids": [
        Command.link(skill.id),
    ],
})
```

يمكن تنفيذ عدة Commands في العملية نفسها:

```python
employee.write({
    "skill_ids": [
        Command.link(python_skill.id),
        Command.link(odoo_skill.id),
    ],
})
```

#### استبدال الروابط: `Command.set()`

لجعل Skills الخاصة بالموظف مطابقة لقائمة محددة:

```python
employee.write({
    "skill_ids": [
        Command.set([
            python_skill.id,
            odoo_skill.id,
        ]),
    ],
})
```

ستُزال أي روابط حالية غير موجودة في القائمة الجديدة.

> [!warning] الإضافة تختلف عن الاستبدال
> `link()` تضيف رابطًا مع الاحتفاظ بالروابط الحالية.
>
> `set()` تجعل الروابط مطابقة للمجموعة الجديدة؛ استخدامها للإضافة فقط قد يزيل روابط أردت الاحتفاظ بها.

#### إنشاء Record مرتبطة: `Command.create()`

لإنشاء Skill وربطها بالموظف:

```python
employee.write({
    "skill_ids": [
        Command.create({
            "name": "PostgreSQL",
        }),
    ],
})
```

القيم داخل Dictionary تخص Model المرتبطة، وهي هنا:

```text
company.skill
```

ويمكن إنشاء Employee من خلال One2many الخاصة بالقسم:

```python
department.write({
    "employee_ids": [
        Command.create({
            "name": "Lina",
        }),
    ],
})
```

تنشئ ORM الموظفة وتضبط `department_id` لتشير إلى Department الحالية.

الأمثلة السابقة تبدأ من Singleton. أما عند تطبيق `Command.create()` على عدة Records:

| العلاقة | النتيجة |
|---|---|
| `Many2many` | Record جديدة واحدة مرتبطة بجميع Records الحالية |
| `One2many` | Record جديدة لكل Record في المجموعة الحالية |

لذلك حجم Recordset التي تستدعي عليها `write()` جزء من فهم العملية.

#### تعديل Record مرتبطة: `Command.update()`

لتعديل Skill مرتبطة:

```python
employee.write({
    "skill_ids": [
        Command.update(
            skill.id,
            {
                "name": "Advanced Python",
            },
        ),
    ],
})
```

هذا يعدل Record الخاصة بالـSkill نفسها.

إذا كانت مرتبطة بموظفين آخرين، سيظهر الاسم الجديد لديهم أيضًا؛ فهي بيانات مشتركة، وليست نسخة خاصة بالموظف الحالي.

#### إزالة رابط: `Command.unlink()`

في Many2many:

```python
employee.write({
    "skill_ids": [
        Command.unlink(skill.id),
    ],
})
```

تُزال العلاقة بين Employee وSkill، وتبقى Skill موجودة.

أما في One2many، فيعتمد السلوك على Many2one العكسية:

- إذا كانت تستخدم `ondelete="cascade"`، تُحذف Record التابعة.
- خلاف ذلك تُفرّغ Many2one وتبقى Record، بشرط أن يسمح تعريف Field بالقيمة الفارغة.

#### إزالة جميع الروابط: `Command.clear()`

لإزالة جميع Skills الخاصة بالموظف:

```python
employee.write({
    "skill_ids": [
        Command.clear(),
    ],
})
```

في Many2many تبقى Skill Records موجودة.

`clear()` تتصرف مثل تنفيذ `unlink()` على كل رابط؛ لذلك تنطبق عليها محاذير One2many نفسها.

#### حذف Record المرتبطة: `Command.delete()`

لحذف Skill نفسها من Database:

```python
employee.write({
    "skill_ids": [
        Command.delete(skill.id),
    ],
})
```

هذه عملية حذف بيانات، وليست مجرد إزالة رابط.

قد يُرفض الحذف إذا كانت Skill لا تزال مرتبطة بـRecords أخرى أو كانت هناك قيود تمنعه.

استخدمها عندما يكون المطلوب حذف Record نفسها.

### مراعاة سلوك One2many

لربط Employee بقسم من جهة One2many:

```python
department.write({
    "employee_ids": [
        Command.link(employee.id),
    ],
})
```

هذا يعيّن Department الحالية للموظف.

إذا كان تابعًا لقسم آخر، سينتقل منه؛ لأن `department_id` لا يمكن أن تشير إلى قسمين في الوقت نفسه.

> [!warning] إزالة One2many قد تحذف البيانات
> `unlink()` و`clear()`، وكذلك الروابط التي تزيلها `set()`، قد تؤدي إلى حذف Records التابعة عندما تستخدم Many2one العكسية `ondelete="cascade"`.
>
> راجع تعريف العلاقة قبل التعامل معها كأنها مجرد قائمة روابط.

---

## Example

سنطبق مثالًا يربط الأقسام والموظفين والمهارات:

```text
Department
    │
    └──→ Employees ←──→ Skills
```

سنستخدم Module تطوير باسم `relationship_training`، مُعدّة كما تعلمنا في [[03 Modules & Addons]]، وتحتوي Manifest صالحة تعتمد على `base`.

نفّذ المثال في Database تجريبية.

### 1. إعداد Import Chain

داخل:

```text
relationship_training/__init__.py
```

تأكد من وجود:

```python
from . import models
```

وداخل:

```text
relationship_training/models/__init__.py
```

أضف:

```python
from . import company
```

احتفظ بأي Imports أخرى موجودة في الملفات.

### 2. تعريف Models

داخل:

```text
relationship_training/models/company.py
```

نكتب:

```python
from odoo import fields, models


class CompanyDepartment(models.Model):
    _name = "company.department"
    _description = "Department"

    name = fields.Char(required=True)

    employee_ids = fields.One2many(
        "company.employee",
        "department_id",
        string="Employees",
    )


class CompanyEmployee(models.Model):
    _name = "company.employee"
    _description = "Employee"

    name = fields.Char(required=True)

    department_id = fields.Many2one(
        "company.department",
        string="Department",
        ondelete="restrict",
    )

    skill_ids = fields.Many2many(
        "company.skill",
        relation="company_employee_skill_rel",
        column1="employee_id",
        column2="skill_id",
        string="Skills",
    )


class CompanySkill(models.Model):
    _name = "company.skill"
    _description = "Skill"

    name = fields.Char(required=True)

    employee_ids = fields.Many2many(
        "company.employee",
        relation="company_employee_skill_rel",
        column1="skill_id",
        column2="employee_id",
        string="Employees",
    )
```

بعد حفظ التعريفات، أعد تشغيل Odoo وثبّت Module أو نفّذ Upgrade لها بحسب حالتها.

إذا احتجت مراجعة الفرق بين Restart وUpgrade، ارجع إلى [[03 Modules & Addons]].

### 3. فتح Odoo Shell

افتح Odoo Shell على Database التجريبية، باستخدام إعدادات بيئة التطوير التي أعددتها في
[[02 Development Environment]]

سنطبق المثال بصلاحيات إدارية في بيئة التطوير. إعداد صلاحيات المستخدمين العاديين سيأتي في [[11 Security & Multi-Company]].

داخل Shell:

```python
from odoo.fields import Command

department_model = env["company.department"]
employee_model = env["company.employee"]
skill_model = env["company.skill"]
```

### 4. إنشاء Department وEmployee

أنشئ القسم:

```python
it_department = department_model.create({
    "name": "IT",
})
```

ثم الموظفة:

```python
nora = employee_model.create({
    "name": "Nora",
    "department_id": it_department.id,
})
```

اقرأ اسم القسم من الموظفة:

```python
nora.department_id.name
```

النتيجة:

```text
IT
```

ثم اقرأ أسماء الموظفين من القسم:

```python
it_department.employee_ids.mapped("name")
```

النتيجة:

```python
["Nora"]
```

هنا قرأنا العلاقة نفسها من الجانبين.

### 5. إنشاء Employee عبر One2many

```python
it_department.write({
    "employee_ids": [
        Command.create({
            "name": "Adam",
        }),
    ],
})
```

تحقق:

```python
it_department.employee_ids.mapped("name")
```

ستجد الاسمين:

```python
["Nora", "Adam"]
```

قد يختلف الترتيب.

لم نمرر `department_id` داخل قيم الموظف؛ لأن الإنشاء تم من خلال One2many الخاصة بالقسم.

### 6. إنشاء Skills وربطها

أنشئ المهارتين:

```python
python_skill = skill_model.create({
    "name": "Python",
})

odoo_skill = skill_model.create({
    "name": "Odoo",
})
```

اربطهما بالموظفة:

```python
nora.write({
    "skill_ids": [
        Command.link(python_skill.id),
        Command.link(odoo_skill.id),
    ],
})
```

اقرأ المهارات:

```python
nora.skill_ids.mapped("name")
```

ستجد:

```python
["Python", "Odoo"]
```

دون الاعتماد على ترتيب محدد.

ومن جهة Skill:

```python
python_skill.employee_ids.mapped("name")
```

النتيجة:

```python
["Nora"]
```

لم نحتج كتابة الرابط مرة ثانية في Skill؛ لأن Fieldين تستخدمان Relation Table نفسها.

### 7. إزالة رابط دون حذف Skill

```python
nora.write({
    "skill_ids": [
        Command.unlink(odoo_skill.id),
    ],
})
```

تحقق من Skills المتبقية:

```python
nora.skill_ids.mapped("name")
```

النتيجة:

```python
["Python"]
```

ثم تحقق أن Skill ما زالت موجودة:

```python
bool(odoo_skill.exists())
```

النتيجة:

```python
True
```

أزلنا الرابط، وبقيت البيانات.

### 8. نقل Employee إلى Department أخرى

أنشئ القسم الجديد:

```python
sales_department = department_model.create({
    "name": "Sales",
})
```

انقل الموظفة:

```python
nora.write({
    "department_id": sales_department.id,
})
```

تحقق من قسمها:

```python
nora.department_id.name
```

النتيجة:

```text
Sales
```

ثم افحص القسم السابق:

```python
it_department.employee_ids.mapped("name")
```

النتيجة:

```python
["Ahmed"]
```

والقسم الجديد:

```python
sales_department.employee_ids.mapped("name")
```

النتيجة:

```python
["Nora"]
```

هذا يوضح أن One2many تتبع Many2one العكسية، وليست قائمة مستقلة عنها.

---

## اختيار العلاقة المناسبة

### ابدأ من قواعد العمل

| السؤال | إذا كانت الإجابة نعم |
|---|---|
| هل Record الحالية تشير إلى Record واحدة كحد أقصى؟ | استخدم `Many2one` |
| هل أحتاج الوصول إلى Records التي تشير إليّ عبر Many2one؟ | أضف `One2many` |
| هل التعدد مسموح في الجانبين؟ | استخدم `Many2many` |

أمثلة:

| الحالة | التصميم المناسب |
|---|---|
| عدة Employees تنتمي إلى Department واحدة | `Many2one` في Employee |
| الوصول إلى Employees من Department | `One2many` في Department |
| عدة Employees تمتلك عدة Skills | `Many2many` |

### حدد نوع التغيير المطلوب

قبل كتابة عملية التعديل، اسأل:

**هل أريد تغيير الرابط، أم تعديل Record المرتبطة، أم حذفها؟**

| المطلوب | الأسلوب |
|---|---|
| تعيين Record في Many2one | كتابة ID |
| تفريغ Many2one اختيارية | كتابة `False` |
| إضافة رابط موجود | `Command.link()` |
| إنشاء Record مرتبطة | `Command.create()` |
| تعديل بيانات Record مرتبطة | `Command.update()` |
| استبدال مجموعة الروابط | `Command.set()` |
| إزالة رابط | `Command.unlink()` |
| إزالة جميع الروابط | `Command.clear()` |
| حذف Record المرتبطة نفسها | `Command.delete()` |

في One2many، راجع Many2one العكسية قبل إزالة الروابط.

---

## Important Notes

- `One2many` تعتمد على Many2one عكسية؛ نقل Record إلى جهة جديدة قد يزيل ارتباطها بالجهة السابقة.

- `Command.set()` تستبدل الروابط الحالية. استخدامها للإضافة فقط قد يزيل روابط موجودة.

- إزالة روابط One2many عبر `unlink()` أو `clear()` أو `set()` قد تحذف Records التابعة عند استخدام `ondelete="cascade"`.

- `Command.update()` تعدل Record المرتبطة نفسها؛ إذا كانت مشتركة، تظهر التغييرات لدى Records الأخرى المرتبطة بها.

- `Command.delete()` تحذف البيانات نفسها، بينما `Command.unlink()` تزيل الرابط في Many2many.

- Many2one تعيد صفرًا أو Record واحدة عند قراءتها من Singleton. أما قراءتها من عدة Records فقد تجمع عدة Records مرتبطة.

- في Multi-Company، اختيار نوع العلاقة وحده لا يضمن توافق الشركات. إعداد `check_company` وآليات التحقق المرتبطة به سيُشرح في [[11 Security & Multi-Company]].

---

> [!tip] الشابتر التالي
> انتقل إلى [[08 Domains]] لتتعلم كتابة شروط البحث والتصفية، بما فيها الشروط التي تستخدم Relationships.

---

## Sources

- [Odoo 20 Documentation — Relational Fields](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#relational-fields)
- [Odoo 20 Documentation — Field Access](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#field-access)
- [Odoo 20 Documentation — ORM Commands](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#odoo.fields.Command)
- [Odoo 20 Documentation — mapped()](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#odoo.models.Model.mapped)
- [Odoo 20 Source Code — Relational Fields](https://github.com/odoo/odoo/blob/20.0/odoo/orm/fields_relational.py)
- [Odoo 20 Source Code — Commands](https://github.com/odoo/odoo/blob/20.0/odoo/orm/commands.py)