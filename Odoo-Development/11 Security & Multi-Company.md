---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 11 Security & Multi-Company

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Security Fundamentals|Security Fundamentals]]
- [[#Groups & Access Definitions|Groups & Access Definitions]]
- [[#Multi-Company Design|Multi-Company Design]]
- [[#Verifying Access|Verifying Access]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

بعد تعريف Models وتحميل البيانات، نحتاج إلى تحديد من يستطيع الوصول إليها وما العمليات المسموحة له.

Security تجيب عن أسئلة مثل:

- من المستخدم الذي ينفذ العملية؟
- ما Models التي يستطيع التعامل معها؟
- هل يستطيع القراءة أوالإنشاء أوالتعديل أوالحذف؟
- هل الوصول يشمل كل Records أم بعضها؟
- كيف تؤثر الشركة الحالية والشركات المفعلة في نطاقه؟

في هذا الشابتر سنبني سياسة وصول بسيطة، ثم نطبقها باستخدام آلية Odoo 20، ونختبرها بمستخدمين محدودي الصلاحيات.

سنميز أيضًا بين أمرين:

- **Access Control:** من يستطيع الوصول إلى Record؟
- **Company Consistency:** هل العلاقات بين Records متوافقة من ناحية الشركة؟

نجاح أحدهما لا يغني عن الآخر.

---

## Security Fundamentals

### نظام الوصول في Odoo 20

تستخدم Odoo 20 الـModel التقنية:

```text
ir.access
```

لتعريف صلاحيات الوصول وقيودها، مع إمكانية تحديد Domain لكل تعريف.

قد تقابل في شروحات إصدارات سابقة:

```text
ir.model.access
ir.rule
```

لا ننقل ملفاتها وحقولها كما هي إلى هذا الشابتر. سنستخدم Syntax الخاصة بـOdoo 20.

التمييز الأساسي داخل `ir.access`:

| نوع التعريف | كيف نحدده؟ | دوره |
| --- | --- | --- |
| Permission | مرتبط بـGroup | يمنح وصولًا |
| Restriction | دون Group | يقيد الوصول على مستوى Model |

سنشرح طريقة جمع هذه التعريفات بعد التعرف على Groups.

---

### ابدأ بسياسة مكتوبة

قبل كتابة XML أوCSV، حدد Roles والعمليات ونطاق Records.

سنستخدم السياسة التالية:

| Model | Equipment User | Equipment Manager |
| --- | --- | --- |
| Equipment Categories | قراءة | CRUD |
| Equipment Assets | قراءة | CRUD |
| Equipment Loans | إنشاء وقراءة طلباته فقط | CRUD لكل الطلبات |

ومع Multi-Company:

- الأجهزة والطلبات مرتبطة بشركة.
- الوصول إليها مقيد بالشركات المفعلة.
- التصنيفات مشتركة بين الشركات في هذا التصميم.

كلمة «طلباته» تعني الطلبات المرتبطة بحساب المستخدم، وليس بالضرورة كل ما أنشأه تقنيًا مستخدم آخر نيابةً عنه.

سنمثل صاحب الطلب باستخدام Field باسم `requested_by_id`.

---

### العمليات الأربع

يستخدم Field باسم `operation` رموزًا من التسلسل `crud`:

| الرمز | العملية | ORM Method المعتادة |
| --- | --- | --- |
| `c` | Create | `create()` |
| `r` | Read | `search()` و`read()` |
| `u` | Update | `write()` |
| `d` | Delete | `unlink()` |

أمثلة على القيم:

| القيمة | العمليات |
| --- | --- |
| `r` | قراءة |
| `cr` | إنشاء وقراءة |
| `ru` | قراءة وتعديل |
| `cru` | إنشاء وقراءة وتعديل |
| `crud` | العمليات الأربع |

وجود صلاحية للعملية لا يضمن نجاحها في كل حالة؛ قد تمنعها Business Rules أوConstraints أوعلاقات البيانات.

---

### المنع عند غياب Permission

في التنفيذ العادي، إذا لم توجد Permission مناسبة تمنح المستخدم العملية على Model والـRecords المطلوبة، فلا يحصل على الوصول.

وجود Model أوMenu أومعرفة Database ID لا يمنح صلاحية.

كذلك، Restriction لا تمنح وصولًا من تلقاء نفسها. يجب أن توجد Permission أولًا، ثم تتحقق القيود المطبقة عليها.

---

## Groups & Access Definitions

### Groups

الـ**Group** تجمع مستخدمين يشتركون في دور أوصلاحيات معينة.

تُمثل داخل Odoo باستخدام:

```text
res.groups
```

مثلًا:

```text
Equipment User
Equipment Manager
```

إنشاء Group وحدها لا يمنح CRUD على Models الخاصة بنا. نربط بها Access Definitions المناسبة.

---

### Group Inheritance

يمكن أن تتضمن Group مجموعات أخرى من خلال `implied_ids`.

في مثالنا:

```text
Equipment Manager
        ↓
Equipment User
        ↓
Internal User
```

هذا يعني أن Manager تحصل أيضًا على عضوية Groups المضمنة وصلاحياتها.

داخل XML، نستخدم Relationship Command مثل:

```xml
<field
    name="implied_ids"
    eval="[Command.link(ref('base.group_user'))]"
/>
```

هذا مقتطف من تعريف Group، وليس ملفًا مستقلًا.

`Command.link()` تضيف العلاقة، و`ref()` تحل External ID للمجموعة المطلوبة، كما تعلمنا في [[07 Relationships]] و[[10 Data Files & External IDs]].

استخدم Group Inheritance عندما تكون العلاقة بين Roles صحيحة فعلًا. صلاحيات Group المضمنة تصبح جزءًا من صلاحيات المجموعة الأعلى.

---

### تنظيم Roles في واجهة المستخدم

في Odoo 20، يمكن ربط Groups بـScope باستخدام:

```text
res.groups.privilege
```

والـField:

```text
privilege_id
```

يساعد هذا في تنظيم مستويات الوصول إلى التطبيق داخل إعدادات المستخدم.

في المثال سنجمع User وManager تحت Scope واحدة باسم Equipment Training.

هذا التنظيم لا يستبدل Access Definitions؛ الصلاحيات الفعلية ستبقى معرفة في `ir.access`.

---

### تعريف Permission باستخدام CSV

نكتب التعريفات في:

```text
security/ir.access.csv
```

مثال:

```csv
id,name,model_id,group_id/id,operation,domain
access_equipment_user,Equipment User Read,equipment.asset,equipment_training.group_equipment_user,r,
```

معنى الأعمدة:

| العمود | المعنى |
| --- | --- |
| `id` | External ID لتعريف الوصول |
| `name` | اسم توضيحي للتعريف |
| `model_id` | Model التي يطبق عليها التعريف |
| `group_id/id` | External ID للـGroup |
| `operation` | العمليات المسموحة |
| `domain` | نطاق Records |

في الصيغة المستخدمة هنا، يحتوي `model_id` على Technical Name مثل `equipment.asset`، كما في ملفات Odoo 20 الرسمية.

أما `group_id/id` فتستخدم External ID للمجموعة.

في المثال، Domain فارغة، لذلك تمنح Permission القراءة على كل Records من ناحية هذا التعريف. قد تضيق Restrictions النطاق لاحقًا.

---

### Permission بنطاق محدد

لمنح المستخدم إنشاء وقراءة طلباته:

```csv
id,name,model_id,group_id/id,operation,domain
access_loan_user,Own Equipment Loans,equipment.loan,equipment_training.group_equipment_user,cr,"[('requested_by_id', '=', user.id)]"
```

Domain هنا تعتمد على:

```text
requested_by_id = المستخدم الحالي
```

الـVariable باسم `user` توفرها Odoo عند تقييم Security Domain، وتمثل المستخدم الحالي.

هذه ليست مجرد Domain للعرض في الواجهة؛ هي جزء من تعريف الوصول على Server.

---

### Global Restriction

عندما نترك Group فارغة، يصبح التعريف Restriction عامة على Model والعمليات المحددة.

مثلًا:

```csv
id,name,model_id,group_id/id,operation,domain
restrict_equipment_company,Equipment Company Scope,equipment.asset,,crud,"[('company_id', 'in', company_ids)]"
```

هذا التعريف يقيد الوصول بالشركات المفعلة، لكنه لا يمنح المستخدم CRUD بنفسه.

> [!important] Group الفارغة تغير معنى التعريف
> في `ir.access` الخاصة بـOdoo 20، ترك Group فارغة يعني Restriction عامة.
>
> لا تستخدمها باعتبارها Permission تمنح الوصول لكل المستخدمين.

---

### كيف تتجمع Permissions وRestrictions؟

التقييم يكون لكل Model وعملية.

الصورة العامة في التنفيذ العادي:

```text
تحقق Permission واحدة على الأقل
              AND
تحقق جميع Restrictions المطبقة
```

| التعريفات | طريقة الجمع |
| --- | --- |
| Permissions من Groups المستخدم | OR |
| Global Restrictions | AND |
| النتيجتان معًا | AND |

مثال:

- Group تمنح قراءة الطلبات الخاصة بالمستخدم.
- Group أخرى تمنح قراءة كل الطلبات.
- Restriction عامة تقيد الطلبات بالشركات المفعلة.

إذا كان المستخدم عضوًا في المجموعتين، يمكنه قراءة كل الطلبات داخل نطاق الشركات المفعلة.

إضافة Permission ضيقة لا تلغي Permission أوسع.

وفي مثالنا، Manager ترث User، لكنها تحصل أيضًا على Permission أوسع للطلبات؛ لذلك لا تبقى مقيدة بطلباتها الشخصية.

---

### Security Domain Variables

من القيم التي توفرها Odoo عند تقييم Access Domain:

| Variable | نوعها ومعناها |
| --- | --- |
| `user` | Record للمستخدم الحالي |
| `company_id` | ID للشركة الحالية |
| `company_ids` | List من IDs الشركات المفعلة في Environment |

مثلًا:

```python
[
    ("company_id", "in", company_ids),
]
```

`company_ids` هنا List من IDs، وليست Recordset؛ لذلك لا نكتب `company_ids.ids`.

هذه Variables يوفرها سياق تقييم Security Domain. لا نفترض أنها موجودة بنفس الشكل في كل مكان نكتب فيه Domain.

---

### UI Groups وField Security

نميز بين استخدامين:

| المكان | التأثير |
| --- | --- |
| `groups` على عنصر واجهة | التحكم في ظهور العنصر |
| `groups` على تعريف Field في Python | تقييد الوصول إلى Field على Server |

مثلًا، داخل Model يمكن تعريف Field مقيدة:

```python
internal_note = fields.Text(
    groups="equipment_training.group_equipment_manager",
)
```

هذا التعريف يفترض استيراد `fields` في ملف Model.

المستخدم الذي لا يملك Group المناسبة لا يحصل على الوصول إلى Field بالطريقة المعتادة، ومحاولة قراءتها أوكتابتها صراحةً قد ترفع `AccessError`.

لكن إخفاء Button في XML وحده لا يمنع تنفيذ العملية بطرق أخرى.

سنشرح Syntax عناصر الواجهة في [[12 Views & XML]] و[[13 Actions, Menus & Buttons]].

---

### CRUD ليست كل Business Security

Permission تسمح بتعديل Model لا تعني بالضرورة أن كل Business Operation عليها مسموحة لكل من يملك `write()`.

قد تحتاج عملية حساسة إلى تحقق إضافي من الدور وحالة Record على Server.

كما أن Parameters القادمة من Client لا تصبح موثوقة لمجرد أن Button معينة ترسلها.

سنطبق منطق العمليات والتحقق داخل Methods في [[15 Business Logic]].

---

## Multi-Company Design

### حدد ملكية Records أولًا

قبل كتابة Company Domain، حدد طبيعة Model:

| التصميم | مثال |
| --- | --- |
| Record تخص شركة واحدة | جهاز أوطلب تابع لشركة |
| Record مشتركة | تصنيف عام تستخدمه عدة شركات |
| Record تحتوي قيمًا تختلف حسب الشركة | تصميم يحتاج Company-dependent Values |

لا ننسخ نفس Domain إلى كل Models.

في هذا الشابتر:

- `equipment.asset` تخص شركة.
- `equipment.loan` تخص شركة.
- `equipment.category` مشتركة.

---

### `company_id`

نمثل شركة Record عادةً باستخدام Many2one:

```python
company_id = fields.Many2one(
    "res.company",
    required=True,
    default=lambda self: self.env.company,
)
```

يكتب هذا التعريف داخل Class الخاصة بالـModel.

القيمة الافتراضية تأتي من الشركة الحالية في Environment، كما تعلمنا في [[09 Environment & Context]].

وجود Field وحدها لا يعزل Records ولا يخفيها. نحتاج Access Restrictions المناسبة.

---

### Allowed Companies وEnabled Companies

نميز بين:

- الشركات المسموح بها للمستخدم.
- الشركات المفعلة حاليًا في Environment.
- الشركة الحالية.

المستخدم قد يسمح له باستخدام شركتين، لكنه يفعل واحدة فقط أثناء عملية معينة.

في Restriction التالية:

```python
[
    ("company_id", "in", company_ids),
]
```

نقيد Records بالشركات المفعلة في سياق التنفيذ.

ولا نعتبر تمرير Company غير مسموحة في Context طريقة لمنح الوصول إليها.

---

### Company Visibility

لـModel مرتبطة دائمًا بشركة، يمكن استخدام Restriction مثل:

```csv
id,name,model_id,group_id/id,operation,domain
restrict_equipment_company,Equipment Company Scope,equipment.asset,,crud,"[('company_id', 'in', company_ids)]"
```

تطبق Restriction على User وManager في التنفيذ العادي.

Manager الخاصة بالتطبيق ليست استثناءً تلقائيًا من العزل بين الشركات.

---

### Shared Records

إذا كان تصميم Model يسمح بوجود Records مشتركة، فقد تكون `company_id` اختيارية، وتكون القيمة `False` للدلالة على المشاركة في هذا التصميم.

يمكن أن يكون نطاق الوصول:

```python
[
    "|",
    ("company_id", "=", False),
    ("company_id", "in", company_ids),
]
```

هذا يسمح بالمشتركة وبالمرتبطة بالشركات المفعلة.

لكن Model مثل `equipment.category` في مثالنا لا تحتوي `company_id` أصلًا؛ لذلك لا نضيف إليها Domain تشير إلى Field غير موجودة.

كذلك، Record مشتركة لا تعني تلقائيًا أن قيم Fields تتغير حسب الشركة. هذا تصميم آخر.

---

### Company Consistency

قد يستطيع المستخدم الوصول إلى شركتين، لكن هذا لا يعني السماح بربط Records بينهما دون توافق.

مثلًا:

```text
Loan في Company A
        ↓
Asset في Company B
```

إذا كان تصميمنا يشترط أن يكون الجهاز والطلب تابعين لنفس الشركة، فيجب منع هذا الربط حتى لو كان المستخدم يستطيع رؤية الشركتين.

نستخدم:

```python
_check_company_auto = True
```

داخل Class، ثم نضع `check_company=True` على العلاقات التي تحتاج التحقق.

مثلًا:

```python
asset_id = fields.Many2one(
    "equipment.asset",
    required=True,
    check_company=True,
)
```

تجري Odoo التحقق التلقائي أثناء عمليات مثل `create()` و`write()`.

---

### حدود `check_company`

ننتبه إلى الآتي:

- Model المرتبطة يجب أن تدعم بيانات الشركة المناسبة للتحقق.
- لا نضع `check_company=True` على `company_id` نفسها.
- التحقق من توافق الشركات لا يغني عن Access Restrictions.
- السماح بقراءة Record لا يثبت أن ربطها بالـRecord الحالية صحيح.

وعند التعامل مع Records مشتركة، تكون القواعد أكثر دقة: Record بلا شركة لا يمكنها تلقائيًا الارتباط بأي Record خاصة بشركة.

أما العلاقات إلى `res.users`، فيراعي التحقق الشركات المسموح بها للمستخدم المرتبط، وليس شركته الافتراضية فقط.

---

### `with_company()` و`sudo()`

سبق شرح الفرق في [[09 Environment & Context]].

في سياق Security:

| API | ما لا نفترضه عنها |
| --- | --- |
| `with_company()` | لا تمنح صلاحية الوصول إلى الشركة |
| `sudo()` | لا تعني أن التصميم أصبح صحيحًا أوأن العلاقات متوافقة |

`sudo()` تتجاوز فحوص الوصول المعتادة دون تغيير المستخدم الحالي، وقد تكشف بيانات خارج الحدود المقصودة.

لكن Company Consistency وBusiness Rules لا تختفي لمجرد استخدام Superuser Mode.

---

## Verifying Access

### اختبر بمستخدم محدود

لا نكتفي بتجربة Admin أوShell الافتراضية.

قد تعمل Shell في Superuser Mode، فتنجح عمليات لا يسمح بها لمستخدم عادي.

عند الاختبار باستخدام `with_user()`، تحقق من:

```python
records.env.su
```

ونتوقع `False` عندما نختبر حسابًا عاديًا.

كذلك، اضبط `allowed_company_ids` المناسبة للمستخدم حتى لا تختلط نتيجة الصلاحيات بسياق شركة غير مقصود.

---

### `has_access()`

يمكن فحص الوصول إلى Records دون تنفيذ تعديل عليها:

```python
records.has_access("write")
```

تعيد Boolean توضح هل المستخدم في Environment الحالية يملك الوصول للعملية على جميع Records الموجودة في Recordset.

الأسماء المستخدمة هنا:

| العملية | القيمة |
| --- | --- |
| قراءة | `"read"` |
| إنشاء | `"create"` |
| تعديل | `"write"` |
| حذف | `"unlink"` |

هذه تختلف عن اختصارات `operation` داخل CSV.

الفحص على Recordset فارغة يتعلق بإمكانية الوصول العامة إلى Model، وليس إثبات أن بيانات إنشاء معينة ستقبل.

كما أن `has_access()` لا تنفذ العملية ولا تختبر كل Business Rules أوConstraints المرتبطة بها.

---

### تجربة رفض عملية داخل Shell

عندما نجرب عملية كتابة نتوقع فشلها، نحتاج إلى ضمان التراجع عن أي تغييرات حدثت قبل الخطأ.

يمكن استخدام:

```python
env.cr.savepoint()
```

وهي نقطة استرجاع داخل Transaction الحالية، وليست Transaction مستقلة أوCommit.

نضع العملية داخل كتلة `with`. إذا خرج منها Exception، تتراجع تغييرات الكتلة.

سنستخدمها في المثال لاختبار Company Consistency، ونلتقط الخطأ خارج الكتلة.

التوسع في كتابة اختبارات منظمة يأتي في [[26 Testing]].

---

## Example

سنطبق السياسة السابقة على Module التدريب:

- User تقرأ الأجهزة والتصنيفات.
- User تنشئ وتقرأ طلباتها فقط.
- Manager تملك CRUD داخل نطاق الشركات المفعلة.
- لا يسمح بربط طلب من شركة بجهاز خاص بشركة أخرى.

يفترض المثال وجود `equipment.asset` من الشابترات السابقة، و`equipment.category` من [[10 Data Files & External IDs]].

---

### 1. تنظيم الملفات

سنضيف:

```text
equipment_training/
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── equipment_loan.py
└── security/
    ├── groups.xml
    └── ir.access.csv
```

وسنعدل Class الحالية الخاصة بـ`equipment.asset`.

احتفظ ببقية Files وImports الموجودة.

---

### 2. إضافة شركة إلى Equipment Asset

داخل Class الموجودة لـ`equipment.asset`، أضف:

```python
_check_company_auto = True

company_id = fields.Many2one(
    "res.company",
    required=True,
    default=lambda self: self.env.company,
)
```

لا تنشئ Class ثانية بنفس `_name`، ولا تحذف Fields السابقة.

> [!important] عند وجود بيانات سابقة
> إضافة شركة إلزامية إلى Model موجودة تحتاج مراجعة ملكية Records السابقة.
>
> نفذ المثال على Database التدريب. في مشروع فعلي، لا تعتبر تعبئة قيمة افتراضية بديلًا عن تحديد الشركة الصحيحة لكل Record وخطة ترحيل مناسبة.

---

### 3. تعريف Equipment Loan

داخل:

```text
equipment_training/models/equipment_loan.py
```

اكتب:

```python
from odoo import fields, models


class EquipmentLoan(models.Model):
    _name = "equipment.loan"
    _description = "Equipment Loan"
    _order = "name, id"

    _check_company_auto = True

    name = fields.Char(required=True)

    company_id = fields.Many2one(
        "res.company",
        required=True,
        default=lambda self: self.env.company,
    )

    requested_by_id = fields.Many2one(
        "res.users",
        string="Requested By",
        required=True,
        default=lambda self: self.env.user,
        check_company=True,
    )

    asset_id = fields.Many2one(
        "equipment.asset",
        required=True,
        check_company=True,
    )
```

الـField باسم `requested_by_id` تشير إلى حساب Odoo User، وليس Employee Record من تطبيق HR.

أضف إلى `models/__init__.py` دون حذف Imports السابقة:

```python
from . import equipment_loan
```

---

### 4. كتابة Groups

داخل:

```text
equipment_training/security/groups.xml
```

اكتب:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record
        id="module_category_equipment_training"
        model="ir.module.category"
    >
        <field name="name">Equipment Training</field>
    </record>

    <record
        id="privilege_equipment_training"
        model="res.groups.privilege"
    >
        <field name="name">Equipment Training</field>
        <field
            name="category_id"
            ref="equipment_training.module_category_equipment_training"
        />
    </record>

    <record id="group_equipment_user" model="res.groups">
        <field name="name">User</field>
        <field name="sequence" eval="10"/>
        <field
            name="privilege_id"
            ref="equipment_training.privilege_equipment_training"
        />
        <field
            name="implied_ids"
            eval="[Command.link(ref('base.group_user'))]"
        />
    </record>

    <record id="group_equipment_manager" model="res.groups">
        <field name="name">Manager</field>
        <field name="sequence" eval="20"/>
        <field
            name="privilege_id"
            ref="equipment_training.privilege_equipment_training"
        />
        <field
            name="implied_ids"
            eval="[Command.link(ref('equipment_training.group_equipment_user'))]"
        />
    </record>
</odoo>
```

التعريفان الأولان ينظمان Groups تحت Scope واضحة.

أما User وManager فتمثلان الدورين، مع تضمين Manager لصلاحيات User.

---

### 5. كتابة Access Definitions

داخل:

```text
equipment_training/security/ir.access.csv
```

اكتب:

```csv
id,name,model_id,group_id/id,operation,domain
access_category_user,Category User Read,equipment.category,equipment_training.group_equipment_user,r,
access_category_manager,Category Manager CRUD,equipment.category,equipment_training.group_equipment_manager,crud,
access_equipment_user,Equipment User Read,equipment.asset,equipment_training.group_equipment_user,r,
access_equipment_manager,Equipment Manager CRUD,equipment.asset,equipment_training.group_equipment_manager,crud,
access_loan_user,Own Equipment Loans,equipment.loan,equipment_training.group_equipment_user,cr,"[('requested_by_id', '=', user.id)]"
access_loan_manager,Equipment Loan Manager CRUD,equipment.loan,equipment_training.group_equipment_manager,crud,
restrict_equipment_company,Equipment Company Scope,equipment.asset,,crud,"[('company_id', 'in', company_ids)]"
restrict_loan_company,Loan Company Scope,equipment.loan,,crud,"[('company_id', 'in', company_ids)]"
```

لاحظ:

- Permission الخاصة بطلبات User تحتوي Domain ملكية.
- Permission الخاصة بطلبات Manager أوسع.
- Company Restrictions عامة، فتطبق على الدورين.
- Categories مشتركة، لذلك لم نضف لها Company Domain.
- User لا تملك تعديل الطلب أوحذفه في هذه السياسة.

---

### 6. ترتيب Manifest

أضف ملفي Security قبل Data File الخاصة بالتصنيفات:

```python
"data": [
    "security/groups.xml",
    "security/ir.access.csv",
    "data/equipment_category_data.xml",
],
```

هذا الجزء داخل Manifest الحالية. احتفظ ببقية إعداداتها وFiles اللازمة للمشروع.

Groups يجب أن تكون متاحة قبل Access CSV التي تشير إليها.

---

### 7. تحميل التغييرات

لأننا أضفنا Python Code:

1. احفظ Files.
2. أوقف Server التدريب.
3. شغّلها من جديد.
4. افتح **Apps** وأزل Filter **Apps**.
5. ابحث عن `Equipment Training`.
6. نفذ **Upgrade**.

يمكن استخدام CLI بدل الواجهة بالطريقة التي تعلمناها في [[03 Modules & Addons]].

لم ننشئ واجهات إدارة الأجهزة والطلبات بعد؛ سنختبر الوصول من Shell، ثم نبني الواجهات في الشابترات القادمة.

---

### 8. تجهيز بيانات اختبار داخل Shell

افتح جلسة Odoo Shell جديدة على `odoo20_training`.

هذه التهيئة تنشئ شركات ومستخدمين تجريبيين، لذلك نفذها في Database التدريب وبسياق Shell الإدارية فقط.

نفذها مرة واحدة داخل الجلسة. اختبارات المستخدمين لاحقًا ستستخدم `with_user()` دون Superuser Mode.

نبدأ بالـGroups:

```python
from odoo import fields

user_group = env.ref(
    "equipment_training.group_equipment_user"
)

manager_group = env.ref(
    "equipment_training.group_equipment_manager"
)
```

ثم شركتين:

```python
company_a = env["res.company"].create({
    "name": "Equipment Training A",
})

company_b = env["res.company"].create({
    "name": "Equipment Training B",
})
```

ونحدد قائمة الشركات:

```python
test_company_ids = (company_a | company_b).ids
```

سننشئ المستخدمين دون إرسال دعوات ضبط Password عند وجود إضافة `auth_signup`. الـContext التالية مخصصة لهذا الغرض:

```python
user_model = env["res.users"].with_context(
    no_reset_password=True,
)
```

أنشئ User:

```python
employee_user = user_model.create({
    "name": "Equipment Test User",
    "login": "equipment.training.user",
    "company_id": company_a.id,
    "company_ids": [
        fields.Command.set(test_company_ids),
    ],
    "group_ids": [
        fields.Command.set([user_group.id]),
    ],
})
```

ثم Manager:

```python
manager_user = user_model.create({
    "name": "Equipment Test Manager",
    "login": "equipment.training.manager",
    "company_id": company_a.id,
    "company_ids": [
        fields.Command.set(test_company_ids),
    ],
    "group_ids": [
        fields.Command.set([manager_group.id]),
    ],
})
```

في Odoo 20 استخدمنا `group_ids` لعضوية Groups الصريحة.

الحسابان مسموح لهما بالشركتين، لكننا سنختار الشركات المفعلة لكل اختبار صراحةً.

---

### 9. تجهيز جهاز لكل شركة

من سياق التهيئة الإدارية:

```python
asset_a = env["equipment.asset"].create({
    "name": "Training Laptop A",
    "company_id": company_a.id,
})

asset_b = env["equipment.asset"].create({
    "name": "Training Laptop B",
    "company_id": company_b.id,
})
```

نحصر الاختبارات على هذين الجهازين:

```python
test_asset_ids = (asset_a | asset_b).ids
```

---

### 10. اختبار القراءة والتعديل

نجهز Model تعمل باسم User، مع تفعيل Company A فقط:

```python
employee_assets_a = env["equipment.asset"].with_user(
    employee_user,
).with_context(
    allowed_company_ids=[company_a.id],
)
```

نتحقق من وضع التنفيذ:

```python
employee_assets_a.env.su
```

المتوقع:

```python
False
```

نبحث:

```python
employee_assets_a.search(
    [
        ("id", "in", test_asset_ids),
    ],
    order="name, id",
).mapped("name")
```

المتوقع:

```python
[
    "Training Laptop A",
]
```

ثم نفحص صلاحية تعديل الجهاز:

```python
employee_assets_a.browse(
    asset_a.id,
).has_access("write")
```

المتوقع:

```python
False
```

نجهز نفس السياق للـManager:

```python
manager_assets_a = env["equipment.asset"].with_user(
    manager_user,
).with_context(
    allowed_company_ids=[company_a.id],
)
```

ثم:

```python
manager_assets_a.browse(
    asset_a.id,
).has_access("write")
```

المتوقع:

```python
True
```

أما قراءة جهاز Company B:

```python
manager_assets_a.browse(
    asset_b.id,
).has_access("read")
```

فالمتوقع:

```python
False
```

Manager ما زالت خاضعة للـCompany Restriction.

---

### 11. تفعيل الشركتين

نشتق سياقًا يفعل الشركتين للمستخدم:

```python
employee_assets_ab = employee_assets_a.with_context(
    allowed_company_ids=test_company_ids,
)
```

ثم نكرر نفس البحث:

```python
employee_assets_ab.search(
    [
        ("id", "in", test_asset_ids),
    ],
    order="name, id",
).mapped("name")
```

المتوقع:

```python
[
    "Training Laptop A",
    "Training Laptop B",
]
```

لم نغير Permissions. غيرنا نطاق الشركات المفعلة ضمن الشركات المسموح بها للمستخدم.

---

### 12. اختبار ملكية الطلبات

نجهز Model باسم User في Company A:

```python
employee_loans_a = env["equipment.loan"].with_user(
    employee_user,
).with_context(
    allowed_company_ids=[company_a.id],
)
```

ثم تنشئ User طلبها:

```python
employee_loan = employee_loans_a.create({
    "name": "Employee Request",
    "asset_id": asset_a.id,
})
```

القيم الافتراضية تعين:

- Company A.
- المستخدم الحالي كصاحب الطلب.

نجهز Model باسم Manager:

```python
manager_loans_a = env["equipment.loan"].with_user(
    manager_user,
).with_context(
    allowed_company_ids=[company_a.id],
)
```

وتنشئ Manager طلبًا لها:

```python
manager_loan = manager_loans_a.create({
    "name": "Manager Request",
    "asset_id": asset_a.id,
})
```

نحصر الفحص على الطلبين:

```python
test_loan_ids = (employee_loan | manager_loan).ids
```

بحث User:

```python
employee_loans_a.search(
    [
        ("id", "in", test_loan_ids),
    ],
    order="name, id",
).mapped("name")
```

المتوقع:

```python
[
    "Employee Request",
]
```

بحث Manager:

```python
manager_loans_a.search(
    [
        ("id", "in", test_loan_ids),
    ],
    order="name, id",
).mapped("name")
```

المتوقع:

```python
[
    "Employee Request",
    "Manager Request",
]
```

هذا يوضح أن Permission الأوسع للـManager توسع الوصول، رغم أنها ترث Group الخاصة بـUser.

---

### 13. اختبار Company Consistency

سنفعل الشركتين للـManager، حتى يكون الجهازان داخل نطاق وصولها:

```python
manager_loans_ab = manager_loans_a.with_context(
    allowed_company_ids=test_company_ids,
)
```

ثم نجرب إنشاء طلب لشركة A مرتبط بجهاز شركة B.

نستخدم Savepoint حتى تتراجع تغييرات المحاولة عند رفضها:

```python
from odoo.exceptions import AccessError, UserError

try:
    with env.cr.savepoint():
        manager_loans_ab.create({
            "name": "Invalid Company Request",
            "company_id": company_a.id,
            "asset_id": asset_b.id,
        })

        raise AssertionError(
            "Expected company consistency rejection."
        )

except AccessError:
    raise

except UserError as error:
    print(error)
```

المتوقع رسالة عدم توافق الشركات.

وضعنا `AssertionError` إذا قُبلت العملية بشكل غير متوقع، حتى تظهر المشكلة وتتراجع المحاولة أيضًا. أما `AccessError` فلا نعتبرها دليلًا على نجاح اختبار Consistency؛ نعيد رفعها لمراجعة إعداد الوصول.

التمييز الذي اختبرناه:

```text
إمكانية الوصول إلى Company A وCompany B
                    ≠
السماح بربط طلب A بجهاز B
```

---

### 14. مراجعة النتائج

| الحالة | المتوقع |
| --- | --- |
| User تقرأ جهازًا في شركة مفعلة | مسموح |
| User تعدل جهازًا | غير مسموح |
| Manager تعدل جهازًا في شركة مفعلة | مسموح من ناحية Access |
| Manager تقرأ جهازًا خارج الشركات المفعلة | غير مسموح |
| User تنشئ طلبًا لها | مسموح ضمن بقية القواعد |
| User تبحث عن طلبات غيرها | لا تدخل في نتيجتها |
| Manager تقرأ الطلبين في الشركة المفعلة | مسموح |
| طلب A يرتبط بجهاز B | مرفوض بسبب Company Consistency |

البيانات التجريبية هنا داخل جلسة Shell. قواعد حفظ التغييرات والخروج من الجلسة كما شرحنا في [[05 ORM]]؛ لا نضيف Manual Commit لمجرد تشغيل الاختبارات.

---

## Important Notes

- **ترك Group فارغة في `ir.access` يعني Restriction عامة.** لا يعني Permission للجميع.

- **Permissions تتجمع وتوسع الوصول.** Permission ضيقة لا تلغي Permission أوسع من Group أخرى.

- **Global Restrictions تضيق الوصول حتى على Manager الخاصة بالتطبيق** في التنفيذ العادي.

- **`company_ids` داخل Security Domain تمثل الشركات المفعلة**، وليست Recordset ولا قائمة نثق بها دون تحقق من المستخدم.

- **وجود `company_id` لا يعزل Records تلقائيًا.** نحتاج سياسة Access مناسبة.

- **`check_company` تتحقق من توافق العلاقات.** لا تستبدل الصلاحيات، ولا توضع على `company_id` نفسها.

- **إخفاء Button أوتصفية Action لا يحمي العملية وحده.** والـField المقيدة بـ`groups` في Python تختلف عن مجرد عنصر مخفي في الواجهة.

- **اختبار Superuser لا يثبت صلاحيات المستخدم العادي.** تحقق من `env.su` وسياق المستخدم والشركات.

- **`sudo()` ليست علاجًا تلقائيًا لـ`AccessError`.** راجع السياسة والتعريفات ومكان تنفيذ العملية أولًا.

- **Manager داخل التطبيق ليست PostgreSQL Superuser ولا Odoo Superuser.** هذه مفاهيم مختلفة.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[12 Views & XML]]

---

## Sources

- [Odoo 20 — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)
- [Odoo 20 — Multi-company Guidelines](https://www.odoo.com/documentation/20.0/developer/howtos/company.html)
- [Odoo 20 Source — Access Definitions](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_access.py)
- [Odoo 20 Source — Access CSV](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/security/ir.access.csv)
- [Odoo 20 Source — Groups](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/res_groups.py)
- [Odoo 20 Source — Group Scopes](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/res_groups_privilege.py)
- [Odoo 20 Source — Users and Company Membership](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/res_users.py)
- [Odoo 20 Source — Access Checks and Company Consistency](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)
- [Odoo 20 Source — User Invitation Context](https://github.com/odoo/odoo/blob/20.0/addons/auth_signup/models/res_users.py)