---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 16 Computed Fields, Related Fields & Onchange

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Computed Fields|Computed Fields]]
- [[#Inverse Methods|Inverse Methods]]
- [[#Related Fields|Related Fields]]
- [[#Onchange|Onchange]]
- [[#Choosing the Right Tool|Choosing the Right Tool]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

بعض القيم يدخلها المستخدم مباشرةً، بينما يمكن للنظام اشتقاق قيم أخرى من البيانات الموجودة.

في طلب إعارة جهاز، مثلًا:

- المستخدم يحدد وقت البداية والنهاية.
- النظام يحسب المدة بينهما.
- اسم الجهاز يمكن عرضه من Record المرتبطة.
- عند إدخال البداية، يمكن للواجهة اقتراح نهاية بعد ساعة.

هذه الاستخدامات متقاربة في شكلها، لكنها تعتمد على أدوات مختلفة:

| الأداة | الاستخدام |
| --- | --- |
| Computed Field | حساب قيمة من بيانات أخرى |
| Related Field | عرض قيمة عبر Relationship |
| Onchange | اقتراح تغييرات أثناء تحرير Form |

سنكمل Model `equipment.loan` التي عرّفناها في [[11 Security & Multi-Company]].

وسنستخدم العلاقة الموجودة `asset_id` بدل إضافة Model موظفين أوFields غير معرّفة في المشروع.

---

## Computed Fields

### ما هي Computed Field؟

هي Field يحدد النظام قيمتها من خلال Compute Method.

مثلًا، مدة الإعارة مشتقة من وقت البداية والنهاية:

```text
duration_hours = (end_at - start_at) بالساعات
```

نربط Field بالـMethod باستخدام `compute`:

```python
duration_hours = fields.Float(
    string="Duration (Hours)",
    compute="_compute_duration_hours",
)
```

هذا التعريف يوضع داخل Class الخاصة بالـModel.

أما Method، فمسؤوليتها تعيين قيمة `duration_hours` لكل Record تتعامل معها.

---

### Dependencies باستخدام `@api.depends`

نحدد Fields التي يعتمد عليها الحساب:

```python
@api.depends("start_at", "end_at")
def _compute_duration_hours(self):
    for loan in self:
        if loan.start_at and loan.end_at:
            delta = loan.end_at - loan.start_at
            loan.duration_hours = delta.total_seconds() / 3600
        else:
            loan.duration_hours = 0.0
```

إذا تغيرت البداية أوالنهاية عبر ORM، تستخدم Odoo هذه Dependencies لتحديد الحساب الذي يحتاج إلى تحديث.

لا نحتاج إلى استدعاء Compute Method يدويًا بعد كل `write()`.

---

### تعيين قيمة لكل Record

قد تحتوي `self` عدة Records.

لذلك نمر عليها، ونعطي Field قيمة في جميع الحالات.

في المثال:

- عند توفر الوقتين، نحسب الفرق.
- عند نقص أحدهما، نعين `0.0`.

Compute Method لا تكتفي بإرجاع رقم باستخدام `return`؛ يجب أن تعين قيمة Field نفسها.

كما أن `0.0` عند نقص البيانات لا تعني أن الطلب مكتمل أوصحيح؛ هي القيمة التي اخترنا عرضها في هذه الحالة.

---

### التخزين باستخدام `store=True`

Computed Fields المخصصة تكون غير مخزنة افتراضيًا.

يمكن تخزين النتيجة:

```python
duration_hours = fields.Float(
    compute="_compute_duration_hours",
    store=True,
)
```

| الاختيار | السلوك |
| --- | --- |
| دون `store=True` | تُحسب القيمة عند الحاجة إليها، مع إدارة ORM للـCache |
| مع `store=True` | تُحفظ النتيجة، وتُجدَّد وفق Dependencies |

التخزين يسمح باستخدام Field المحسوبة في البحث المعتاد المعتمد على Database، ويساعد في الاستخدامات التحليلية المناسبة.

أما Computed Field المخصصة غير المخزنة، فتحتاج إلى آلية بحث مناسبة إذا أردنا البحث عليها.

لكن `store=True` ليست تحسين Performance تلقائيًا؛ قد يؤدي تغيير مصدر واحد إلى إعادة حساب عدد كبير من Records.

راجع [[28 Performance]] عند التعامل مع حسابات مكلفة أوحجم بيانات كبير.

---

### دقة Dependencies

إذا كان الحساب يعتمد على قيمة داخل Relationship، يمكن استخدام مسار مثل:

```python
@api.depends("asset_id.name")
```

هذا يختلف عن الاعتماد على `asset_id` فقط.

تغيير الجهاز المرتبط وتغيير اسم الجهاز حدثان مختلفان، ويجب أن تصف Dependencies ما يحتاجه الحساب فعلًا.

في Related Field، يعالج ORM Dependencies المرتبطة بالمسار عادةً دون Compute Method نكتبها بأنفسنا.

---

## Inverse Methods

### لماذا نحتاج Inverse؟

Computed Field تكون للقراءة افتراضيًا، لأن قيمتها تأتي من مصادر أخرى.

أحيانًا نريد السماح للمستخدم بتحرير النتيجة، ثم تحويل هذا التعديل إلى تغيير في المصادر.

في مثال المدة:

```text
Compute:
البداية + النهاية → المدة

Inverse:
البداية + المدة المعدلة → النهاية
```

نحدد Method باستخدام `inverse`:

```python
duration_hours = fields.Float(
    compute="_compute_duration_hours",
    inverse="_inverse_duration_hours",
    store=True,
)
```

---

### اختيار المصدر الذي سيتغير

المعادلة لا تحدد وحدها أي Field يجب تغييرها.

إذا عدّل المستخدم المدة، فقد نغير البداية أوالنهاية.

في تطبيقنا، سنحتفظ بالبداية ونغير النهاية:

```text
end_at = start_at + duration_hours
```

هذه سياسة اخترناها للمثال، ويجب أن تكون واضحة للمستخدم.

---

### Inverse ليست Compute أخرى

Compute Method تعين Field المحسوبة.

أما Inverse Method، فتعدل المصادر التي تسمح بإعادة حساب النتيجة المطلوبة.

سنستخدمها لتعديل `end_at`، وسنرفض تحرير المدة دون وجود `start_at`.

تعالج Inverse حفظ القيمة المعدلة؛ لا تعتمد عليها وحدها لتقديم اقتراح فوري داخل Form.

---

## Related Fields

### الوصول إلى قيمة عبر Relationship

لدينا في `equipment.loan` علاقة:

```python
asset_id = fields.Many2one("equipment.asset")
```

يمكن إضافة Field تعرض اسم الجهاز:

```python
asset_name = fields.Char(
    string="Equipment Name",
    related="asset_id.name",
    readonly=True,
)
```

المسار:

```text
طلب الإعارة
    ↓ asset_id
الجهاز
    ↓ name
اسم الجهاز
```

Related Field حالة خاصة من Computed Fields؛ يتولى ORM قراءة القيمة عبر المسار المحدد.

لا نكتب Compute Method لهذا الاستخدام البسيط.

---

### السلوك الافتراضي

Related Fields تكون افتراضيًا:

- غير مخزنة.
- للقراءة.
- غير منسوخة عند Duplicate.
- محسوبة بوضع Superuser.

يمكن تغيير الإعدادات المناسبة، لكن يجب فهم أثرها، خصوصًا عند الوصول إلى بيانات حساسة.

في التطبيق، سنستخدم:

```python
compute_sudo=False
```

حتى لا نطلب تجاوز صلاحيات الوصول أثناء حساب هذه Field.

هذا لا يغني عن تصميم صلاحيات Model والـField التي تعرض النتيجة.

---

### Related ليست Snapshot

إذا تغير اسم الجهاز، يفترض أن تعرض Related Field الاسم الحالي وفق Dependencies الخاصة بها.

حتى إذا أضفنا:

```python
store=True
```

فهي تظل قيمة مرتبطة قابلة لإعادة الحساب، وليست اسمًا تاريخيًا ثابتًا.

| المطلوب | التصميم |
| --- | --- |
| اسم الجهاز الحالي | Related Field |
| اسم الجهاز كما كان وقت اعتماد الطلب | Field مستقلة تُعبأ ضمن عملية العمل المناسبة |

التخزين يحدد مكان حفظ النتيجة، ولا يحدد وحده معناها التاريخي.

---

### Related ليست أداة تجميع

المسار عبر Many2one مناسب لقراءة قيمة من Record محددة.

لكن لا نستخدم Related لعبور مجموعة Records ثم نتوقع منها جمع القيم تلقائيًا.

إذا أردنا مثلًا مجموع قيم عدة Records مرتبطة، نكتب Compute Method مناسبة، مع Dependencies الصحيحة.

---

## Onchange

### اقتراح أثناء تحرير Form

Onchange تساعد المستخدم أثناء إدخال البيانات، قبل حفظ Record.

مثلًا، إذا أدخل البداية وكانت النهاية فارغة، نقترح نهاية بعد ساعة:

```python
@api.onchange("start_at")
def _onchange_start_at(self):
    if self.start_at and not self.end_at:
        self.end_at = self.start_at + timedelta(hours=1)
```

اخترنا شرط `not self.end_at` حتى لا نستبدل نهاية موجودة ربما عدّلها المستخدم.

---

### متى تعمل؟

تُستدعى Onchange ضمن تفاعل Form المناسب عند تعديل Field التي تعتمد عليها.

لكن استدعاء:

```python
loan_model.create(values)
```

لا يستدعي Onchange تلقائيًا لتعبئة Fields.

| المسار | هل يعتمد تلقائيًا على Onchange الخاصة بالـForm؟ |
| --- | --- |
| تحرير Field داخل Form | نعم، وفق تعريف View والتفاعل |
| `create()` مباشرةً | لا |
| `write()` مباشرةً | لا |
| Import المعتاد | لا |

أما Compute، فهي جزء من سلوك Field في ORM، ولذلك لا يقتصر حسابها على Form.

---

### Pseudo-record

داخل Onchange، تمثل `self` القيم الحالية في Form.

قد تكون Record جديدة لم تُحفظ بعد، ولذلك لا نفترض وجود Record عادية في Database.

نستخدم تعيين القيم:

```python
self.end_at = ...
```

ولا نستدعي `create()` أو`write()` أو`unlink()` على Pseudo-record لمحاولة حفظها.

القيم التي نعينها تُعاد إلى الواجهة، ويظل الحفظ خطوة مستقلة.

---

### Onchange ليست قاعدة إلزامية

إذا كان المطلوب أن يحصل كل طلب على نهاية بعد ساعة، حتى عند إنشائه من Integration، فهذه مسؤولية منطق Server المناسب.

أما في مثالنا، فهي مجرد اقتراح يستطيع المستخدم تغييره.

كذلك، Onchange لا تمنع نهاية تسبق البداية. سنعالج صحة ترتيب الوقتين في [[17 Constraints]].

---

## Choosing the Right Tool

| السؤال | الأداة المناسبة |
| --- | --- |
| هل القيمة مشتقة من Fields أخرى؟ | Compute |
| هل أحتاج قيمة حالية عبر Relationship؟ | Related |
| هل أريد اقتراحًا أثناء تحرير Form؟ | Onchange |
| هل أريد تحرير نتيجة محسوبة مع تعديل مصادرها؟ | Compute مع Inverse |
| هل أريد منع بيانات غير صحيحة؟ | Constraint أوقاعدة عمل مناسبة |
| هل أريد قيمة تاريخية ثابتة؟ | Snapshot تُحفظ ضمن العملية المناسبة |

يمكن لهذه الأدوات أن تعمل معًا، لكن لكل واحدة مسؤولية واضحة.

---

## Example

سنضيف إلى طلب الإعارة:

| Field | دورها |
| --- | --- |
| `start_at` | بداية الفترة |
| `end_at` | نهاية الفترة |
| `duration_hours` | مدة محسوبة ومخزنة، مع Inverse |
| `asset_name` | اسم الجهاز عبر Related |

وسنضيف Onchange تقترح نهاية بعد ساعة إذا كانت النهاية فارغة.

---

### 1. تحديث Model الإعارة

افتح الملف:

```text
custom-addons/equipment_training/models/equipment_loan.py
```

التعريف التالي يحافظ على Fields والصلاحيات المرتبطة بتصميم الشابتر 11، ويضيف Fields وMethods الجديدة.

ادمجه داخل Class الموجودة، مع الاحتفاظ بأي إضافات أخرى لديك.

```python
from datetime import timedelta

from odoo import _, api, fields, models
from odoo.exceptions import UserError


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
        string="Equipment",
        required=True,
        check_company=True,
    )

    asset_name = fields.Char(
        string="Equipment Name",
        related="asset_id.name",
        readonly=True,
        compute_sudo=False,
    )

    start_at = fields.Datetime(string="Start")

    end_at = fields.Datetime(string="End")

    duration_hours = fields.Float(
        string="Duration (Hours)",
        compute="_compute_duration_hours",
        inverse="_inverse_duration_hours",
        store=True,
        compute_sudo=False,
    )

    @api.depends("start_at", "end_at")
    def _compute_duration_hours(self):
        for loan in self:
            if loan.start_at and loan.end_at:
                delta = loan.end_at - loan.start_at
                loan.duration_hours = delta.total_seconds() / 3600
            else:
                loan.duration_hours = 0.0

    def _inverse_duration_hours(self):
        if any(not loan.start_at for loan in self):
            raise UserError(
                _("Set a start time before editing the duration.")
            )

        for loan in self:
            loan.end_at = loan.start_at + timedelta(
                hours=loan.duration_hours
            )

    @api.onchange("start_at")
    def _onchange_start_at(self):
        if self.start_at and not self.end_at:
            self.end_at = self.start_at + timedelta(hours=1)
```

لاحظ أننا:

- استخدمنا `asset_id` الموجودة.
- احتفظنا بـ`company_id` و`requested_by_id`.
- لم نستبدل Model بتعريف جديد يفقد Fields السابقة.
- حددنا سلوك الحساب صراحةً دون تجاوز صلاحيات الحساب عبر `compute_sudo`.

---

### 2. تجهيز Views للإعارة

حتى نجرب Onchange من Form، نحتاج إلى واجهة لطلبات الإعارة.

أنشئ:

```text
views/equipment_loan_views.xml
```

وضع فيه:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <record id="view_equipment_loan_list" model="ir.ui.view">
        <field name="name">equipment.loan.list</field>
        <field name="model">equipment.loan</field>
        <field name="arch" type="xml">
            <list>
                <field name="name"/>
                <field name="asset_id"/>
                <field name="requested_by_id"/>
                <field name="start_at"/>
                <field name="end_at"/>
                <field name="duration_hours"/>
                <field name="company_id"/>
            </list>
        </field>
    </record>

    <record id="view_equipment_loan_form" model="ir.ui.view">
        <field name="name">equipment.loan.form</field>
        <field name="model">equipment.loan</field>
        <field name="arch" type="xml">
            <form>
                <sheet>
                    <group>
                        <group string="Request">
                            <field name="name"/>
                            <field name="company_id"/>
                            <field name="requested_by_id"/>
                            <field
                                name="asset_id"
                                options="{'no_create': True}"
                            />
                            <field name="asset_name"/>
                        </group>

                        <group string="Period">
                            <field name="start_at"/>
                            <field name="end_at"/>
                            <field
                                name="duration_hours"
                                readonly="not start_at"
                            />
                        </group>
                    </group>
                </sheet>
            </form>
        </field>
    </record>

    <record
        id="action_equipment_loans"
        model="ir.actions.act_window"
    >
        <field name="name">Equipment Loans</field>
        <field name="res_model">equipment.loan</field>
        <field name="view_mode">list,form</field>
        <field
            name="view_id"
            ref="view_equipment_loan_list"
        />
    </record>
</odoo>
```

حددنا List View للـAction، وتستخدم Form الافتراضية المناسبة للـModel؛ في تطبيقنا الحالي هي Form التي عرّفناها هنا.

جعلنا تحرير المدة متاحًا بعد إدخال البداية، مع بقاء التحقق المقابل داخل Inverse على Server.

---

### 3. إضافة Menu وترتيب التحميل

داخل:

```text
views/equipment_menus.xml
```

أضف عنصر القائمة التالي داخل `odoo`، بعد تعريف القائمة الرئيسية الموجودة:

```xml
<menuitem
    id="menu_equipment_loans"
    name="Loans"
    parent="menu_equipment_root"
    action="action_equipment_loans"
    sequence="20"
    groups="equipment_training.group_equipment_user"
/>
```

ثم أضف ملف Views الجديد إلى Manifest قبل ملف Menus:

```python
"data": [
    "security/groups.xml",
    "security/ir.access.csv",
    "data/equipment_category_data.xml",
    "views/equipment_views.xml",
    "views/equipment_actions.xml",
    "views/equipment_loan_views.xml",
    "views/equipment_menus.xml",
],
```

احتفظ بأي ملفات إضافية يحتاجها مشروعك.

صلاحيات `equipment.loan` موجودة من الشابتر 11؛ لا نضيف تعريفات وصول مكررة.

---

### 4. تطبيق Upgrade

بعد حفظ الملفات:

1. أوقف Server التدريب.
2. شغّله مجددًا لتحميل Python المعدلة.
3. افتح Database `odoo20_training`.
4. افتح **Apps** وابحث عن `Equipment Training`.
5. نفّذ **Upgrade**.
6. أعد تحميل الواجهة.

ملف `equipment_loan.py` مستورد مسبقًا من الشابتر 11؛ تأكد من بقاء Import الخاصة به داخل `models/__init__.py`.

---

### 5. تجربة Form

استخدم Equipment Manager حتى تستطيع إنشاء الطلب وتعديله وفق صلاحيات الشابتر 11.

1. افتح **Equipment → Loans**.
2. أنشئ طلبًا جديدًا.
3. اختر جهازًا من شركة الطلب نفسها.
4. تحقق من ظهور اسمه في **Equipment Name**.
5. أدخل وقت البداية، واترك النهاية فارغة قبل إدخاله.
6. تحقق من اقتراح نهاية بعد ساعة.
7. تحقق من أن المدة أصبحت ساعة واحدة.
8. عدّل النهاية لتكون بعد البداية بساعة ونصف.
9. تحقق من أن المدة أصبحت `1.5`.

النتيجة تجمع ثلاثة سلوكيات:

| التغيير | المسؤول عنه |
| --- | --- |
| ظهور اسم الجهاز | Related |
| اقتراح النهاية | Onchange |
| حساب المدة | Compute |

إذا عدّلت البداية بعد وجود نهاية، فلن تستبدل Onchange النهاية الموجودة؛ هذا سلوك مقصود لحماية إدخال المستخدم.

---

### 6. تجربة Inverse

بعد وجود بداية ونهاية:

1. عدّل **Duration (Hours)** إلى `2.0`.
2. احفظ الطلب.
3. تحقق من أن النهاية أصبحت بعد البداية بساعتين.

في هذا المسار، عدّل المستخدم النتيجة المحسوبة، ثم استخدمت Inverse البداية والمدة لتحديث النهاية.

---

### 7. تجهيز فحص ORM

افتح Odoo Shell على Database التدريب.

سنستخدم حساب المدير التدريبي من الشابتر السابق:

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

asset = equipment_model.create(
    {
        "name": "Compute Training Laptop",
        "serial_number": "COMP-016",
        "company_id": test_company.id,
        "state": "available",
    }
)
```

نستخدم حساب المدير للفحص، مع شركة مسموحة له، وجهاز تدريب مستقل.

---

### 8. إثبات أن Onchange لا تعمل تلقائيًا مع `create()`

أنشئ طلبًا يحتوي بداية دون نهاية:

```python
loan = loan_model.create(
    {
        "name": "Compute Training Request",
        "asset_id": asset.id,
        "start_at": "2026-02-10 08:00:00",
    }
)

loan.end_at
```

المتوقع:

```python
False
```

ثم:

```python
loan.duration_hours
```

المتوقع:

```python
0.0
```

لم يقترح الإنشاء المباشر نهاية بعد ساعة؛ Onchange الخاصة بالـForm لم تُستدعَ تلقائيًا.

لكن Compute أعطت قيمة حسب البيانات المتاحة.

---

### 9. فحص إعادة الحساب والبحث

أضف النهاية باستخدام ORM:

```python
loan.write(
    {
        "end_at": "2026-02-10 09:30:00",
    }
)

loan.duration_hours
```

المتوقع:

```python
1.5
```

ثم ابحث على المدة المخزنة، مع قصر البحث على طلب التدريب:

```python
matching_loans = loan_model.search(
    [
        ("id", "=", loan.id),
        ("duration_hours", ">=", 1.0),
    ]
)

loan in matching_loans
```

المتوقع:

```python
True
```

التعديل جاء عبر ORM، ومع ذلك حدث حساب المدة دون الاعتماد على Onchange.

---

### 10. فحص Inverse عبر ORM

نفّذ:

```python
loan.write(
    {
        "duration_hours": 2.0,
    }
)

print(loan.end_at)
```

المتوقع:

```text
2026-02-10 10:00:00
```

الـDatetime في أمثلة Shell تستخدم توقيت UTC. قد تعرض Form الأوقات وفق المنطقة الزمنية للمستخدم، بينما تبقى المدة بين الوقتين نفسها.

---

### 11. فحص Related Field

اقرأ اسم الجهاز:

```python
loan.asset_name
```

المتوقع:

```python
"Compute Training Laptop"
```

ثم عدّل اسم جهاز التدريب:

```python
asset.write(
    {
        "name": "Compute Training Laptop Updated",
    }
)

loan.asset_name
```

المتوقع:

```python
"Compute Training Laptop Updated"
```

هذا يوضح أن Related تعرض الاسم الحالي، وليست نسخة تاريخية من الاسم وقت إنشاء الطلب.

فحوص Shell هنا للتعلم، وتبقى ضمن معاملة Shell كما شرحنا في [[05 ORM]].

---

## Important Notes

- Compute Method تعين Field لكل Record، ولا تعتمد على `return` لإعطاء قيمتها.
- `store=True` لا تعني أن القيمة أصبحت Snapshot، ولا تضمن تحسن Performance.
- Inverse يجب أن تملك سياسة واضحة لتعديل المصادر؛ معادلة الحساب وحدها قد تسمح بأكثر من تفسير.
- تجنب مشاركة Inverse واحدة بين عدة Fields دون فهم حماية القيم أثناء تنفيذها.
- Related Field لا تصبح آمنة لمجرد أنها للقراءة؛ راجع صلاحيات المصدر والنتيجة وسياق الحساب.
- Onchange لا تحفظ Record بنفسها، ولا تُستدعى تلقائيًا لمجرد استخدام `create()` أو`write()`.
- حساب مدة سالبة عند عكس الوقتين لا يمنع حفظ البيانات؛ سنضيف تحقق ترتيب البداية والنهاية في الشابتر التالي بدل إخفاء المشكلة داخل الحساب.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[17 Constraints]]

---

## Sources

- [Odoo 20 Tutorial — Computed Fields and Onchanges](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/08_compute_onchange.html)
- [Odoo 20 Documentation — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Source — Field Definitions, Related Fields and Computation](https://github.com/odoo/odoo/blob/20.0/odoo/orm/fields.py)
- [Odoo 20 Source — Depends and Onchange Decorators](https://github.com/odoo/odoo/blob/20.0/odoo/orm/decorators.py)