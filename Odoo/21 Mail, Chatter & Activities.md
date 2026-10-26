---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 21 Mail, Chatter & Activities

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Mail Framework Fundamentals|Mail Framework Fundamentals]]
- [[#Messages & Tracking|Messages & Tracking]]
- [[#Activities|Activities]]
- [[#Practical Example|Practical Example]]
- [[#Notifications & Security|Notifications & Security]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في مشروع التدريب، نستطيع إنشاء الأجهزة وتغيير حالاتها وطباعة بياناتها.

لكن العمل على Record قد يحتاج أيضًا إلى تواصل ومتابعة:

- تسجيل ملاحظة مرتبطة بالجهاز.
- معرفة من غيّر حالته.
- متابعة تحديثاته.
- تكليف مستخدم بمراجعته قبل موعد محدد.

توفر Module باسم `mail` الأدوات الأساسية لهذه الاحتياجات.

في هذا الشابتر، سنضيف إلى `equipment.asset`:

1. Chatter لعرض الرسائل والملاحظات والتغييرات.
2. Tracking لاسم الجهاز وحالته.
3. Activities لجدولة مراجعة الجهاز.

وسنحتفظ بالحالات التي استخدمناها سابقًا:

```text
available
in_use
retired
```

---

## Mail Framework Fundamentals

### ما الذي تضيفه Mail Framework؟

توفر مجموعة أدوات مترابطة:

| الأداة | الغرض |
| --- | --- |
| Chatter | واجهة التواصل والمتابعة داخل Record |
| Messages | رسائل وملاحظات مرتبطة بالسجل |
| Followers | جهات مشتركة في متابعة تحديثات السجل |
| Field Tracking | تسجيل تغييرات حقول محددة |
| Activities | أعمال مطلوبة للمتابعة |
| Notifications | تنبيهات مرتبطة بالأحداث والرسائل |

وجود هذه الأدوات على Record يجعل التواصل مرتبطًا ببيانات العمل نفسها.

مثلًا، تظهر ملاحظة مراجعة جهاز داخل Form ذلك الجهاز، بدل الاحتفاظ بها في مكان منفصل يصعب ربطه به.

---

### Dependency على `mail`

للاستفادة من هذه الإمكانيات، نضيف `mail` إلى Dependencies:

```python
"depends": [
    "base",
    "web",
    "mail",
],
```

نحتفظ بـ`base` و`web` من الشابترات السابقة، ونضيف `mail`.

اسم Module التقني هو:

```text
mail
```

وقد يظهر التطبيق المرتبط بها للمستخدم باسم **Discuss**.

---

### دور `mail.thread`

تضيف Mixin باسم `mail.thread` إمكانيات الرسائل والمتابعين وربط تغييرات الحقول بالمحادثة.

داخل Class الأصلية للـModel، يمكن إضافة:

```python
_inherit = [
    "mail.thread",
]
```

الـMixin تضيف سلوكًا وحقولًا إلى Business Model، كما تعلمنا في [[18 Inheritance & Module Interaction]].

لكن إضافة Mixin إلى Python لا تعرض Chatter داخل Form وحدها؛ نحتاج إلى إضافتها في XML أيضًا.

---

### دور `mail.activity.mixin`

لإضافة إدارة Activities، نستخدم:

```python
_inherit = [
    "mail.thread",
    "mail.activity.mixin",
]
```

تضيف `mail.activity.mixin` حقولًا وMethods للتعامل مع الأنشطة المرتبطة بالسجل.

مثل:

- الأنشطة الحالية.
- الموعد الأقرب.
- المستخدم المسؤول.
- جدولة الأنشطة ومتابعتها.

في المثال العملي، سنستخدم Mixins الاثنتين.

---

### عرض Chatter داخل Form

نضيف العنصر:

```xml
<chatter/>
```

داخل `<form>`، عادةً بعد `<sheet>`:

```xml
<form>
    <sheet>
        <!-- Business fields -->
    </sheet>

    <chatter/>
</form>
```

قد تظهر Chatter أسفل Form أوإلى جانبها بحسب حجم الشاشة وتخطيط الواجهة.

لا نضعها داخل `<group>` الخاصة بالحقول العادية.

---

## Messages & Tracking

### الرسائل والملاحظات

تتيح Chatter عادةً خيارات مثل:

- **Send message** للتواصل.
- **Log note** لتسجيل ملاحظة.
- عرض التغييرات المسجلة.
- إدارة المتابعين والمرفقات.

قبل كتابة محتوى، حدد من يحتاج إلى رؤيته ومن قد يتلقى تنبيهًا عنه.

الملاحظة الداخلية ليست رسالة خاصة بين الكاتب ونفسه.

كما أن إضافة مستلمين أوMentions تؤثر في التنبيهات؛ لا تعتمد على اسم الزر وحده لتحديد جمهور المحتوى.

---

### كتابة Message من Python

يمكن استخدام `message_post()` على Record محفوظة:

```python
asset.message_post(
    body="A review has been requested.",
    message_type="comment",
    subtype_xmlid="mail.mt_note",
)
```

في هذا المثال:

| المعامل | الغرض |
| --- | --- |
| `body` | محتوى الرسالة |
| `message_type` | نوع الرسالة |
| `subtype_xmlid` | تصنيف الرسالة المستخدم في سلوك العرض والتنبيهات |

استخدمنا `mail.mt_note` لتسجيل ملاحظة داخلية.

في Odoo 20، تعيد `message_post()` Record من `mail.message`، ويمكن الوصول إلى معرّفها باستخدام `.id` عند الحاجة.

> [!note] Message على Record واحدة
> `message_post()` تتوقع Record واحدة محفوظة.
>
> عند التعامل مع Recordset متعددة، مرّ على Records وانشر الرسالة لكل واحدة حسب الحاجة.

---

### التعامل مع النصوص وHTML

عند تمرير `body` كنص عادي، يعالجه Odoo باستخدام Escaping.

مثلًا:

```python
asset.message_post(
    body=self.env._(
        "Review requested for %(equipment)s.",
        equipment=asset.display_name,
    ),
    message_type="comment",
    subtype_xmlid="mail.mt_note",
)
```

يمكن أن يحتوي اسم الجهاز على رموز خاصة، ويظل تمريره هنا جزءًا من نص عادي.

إذا احتجت إلى HTML منسقة، استخدم الأدوات المناسبة مع فصل HTML الموثوقة عن القيم القادمة من المستخدم.

لا تحول مدخلات المستخدم إلى HTML موثوقة بمجرد تغليفها في `Markup`.

---

### المتابعون: Followers

المتابعون جهات مشتركة في تحديثات Record.

يتأثر ما يصل إليهم بإعدادات الاشتراك وتصنيف الرسالة وسلوك التنبيهات.

هناك فرق مهم بين Models المستخدمة:

| الاستخدام | Model |
| --- | --- |
| المستخدم المسؤول عن Activity | `res.users` |
| الشريك المرتبط بالمستخدم | `res.partner` |
| الجهات المشار إليها عبر `partner_ids` في الرسائل | `res.partner` |

لذلك، `partner_ids` لا تستقبل User IDs لمجرد أن المستلمين مستخدمون داخل Odoo.

---

### تتبع تغييرات الحقول

لتسجيل تغييرات Field، نضيف:

```python
tracking=True
```

مثلًا:

```python
name = fields.Char(
    required=True,
    tracking=True,
)
```

عندما تتغير القيمة عبر مسار ORM المعتاد، يستطيع Mail Framework تسجيل التغيير في Chatter.

لا تحتاج إلى كتابة `message_post()` يدويًا لكل تغيير في هذه Field.

---

### اختيار الحقول التي نتابعها

Tracking مفيدة للحقول التي يساعد تاريخها المستخدم على فهم ما حدث.

في مشروعنا، سنختار:

- اسم الجهاز.
- حالة الجهاز.

تتبع كل Field قد ينتج سجلًا مزدحمًا يصعب قراءته.

كذلك، Tracking تبدأ بعد تفعيلها؛ لا تعيد بناء تاريخ تغييرات حدثت قبل إضافتها.

> [!important] Tracking وقواعد العمل
> Tracking تسجل التغيير المقبول.
>
> أما السماح بالتغيير أو رفضه، فيبقى مسؤولية الصلاحيات وقواعد العمل التي أنشأناها في [[15 Business Logic]] و[[17 Constraints]].

---

## Activities

### ما هي Activity؟

Activity تمثل عملًا مطلوبًا، ويمكن ربطها بـBusiness Record.

مثلًا:

```text
الجهاز: Training Laptop
المطلوب: مراجعة حالة الجهاز
المسؤول: مستخدم محدد
الموعد: بعد يومين
```

تختلف عن الملاحظة:

| Message أوNote | Activity |
| --- | --- |
| تسجل معلومة أوتواصلًا | تمثل عملًا مطلوبًا |
| قد تصف حدثًا وقع | تحتاج إلى متابعة وإنجاز |
| لا تحتاج إلى مسؤول وموعد دائمًا | يمكن تحديد مسؤول وموعد لها |

---

### نوع النشاط وRecord النشاط

نميز بين:

| Model | ما تمثله |
| --- | --- |
| `mail.activity.type` | نوع النشاط، مثل To-Do أوEquipment Review |
| `mail.activity` | نشاط فعلي مرتبط بسجل ومسؤول وموعد |

تعريف نوع باسم **Equipment Review** لا ينشئ مهمة مراجعة لكل جهاز تلقائيًا.

ننشئ Activity الفعلية عند الجدولة.

---

### جدولة Activity من Python

مثال باستخدام نوع To-Do القياسي:

```python
asset.activity_schedule(
    "mail.mail_activity_data_todo",
    date_deadline=fields.Date.context_today(asset),
    user_id=self.env.user.id,
    summary=self.env._("Review this equipment."),
)
```

هنا نحدد:

- نوع النشاط باستخدام External ID.
- الموعد النهائي.
- المستخدم المسؤول.
- ملخص المطلوب.

`date_deadline` حقل Date، ولذلك نمرر تاريخًا مناسبًا.

واستخدام `fields.Date.context_today()` يراعي سياق المنطقة الزمنية عند تحديد اليوم الحالي.

---

### دورة حياة Activity

```text
جدولة النشاط
    ↓
ظهوره ضمن الأعمال المطلوبة
    ↓
تنفيذ العمل
    ↓
Mark Done مع Feedback عند الحاجة
    ↓
تسجيل نتيجة الإنجاز
```

في مصدر Odoo 20، يؤرشف مسار الإنجاز المعتاد Activity، ويسجل رسالة مرتبطة بإنجازها على السجل.

لذلك اختفاء النشاط من قائمة الأعمال المفتوحة لا يعني بالضرورة حذف Record الخاصة به.

ولا يعني اكتمال Activity أن حالة الجهاز تغيرت؛ هذا يحتاج إلى Business Logic صريحة إذا كان مطلوبًا.

---

### تجنب الأنشطة المتكررة

استدعاء `activity_schedule()` ينشئ أنشطة؛ لا تفترض أنه يبحث عن نشاط مماثل ويعيد استخدامه.

قبل إضافة زر أوAutomation، حدد السياسة المطلوبة:

- هل نسمح بعدة مراجعات مفتوحة؟
- هل التكرار يحدد حسب الجهاز والمستخدم؟
- هل نعيد الجدولة أم ننشئ نشاطًا جديدًا؟

في التطبيق، سنبحث عن مراجعة مفتوحة لنفس الجهاز والمستخدم قبل إنشاء واحدة جديدة.

وسنستخدم نوعًا مخصصًا، حتى نميز مراجعة الأجهزة عن أنشطة To-Do الأخرى.

---

## Practical Example

سنضيف Chatter وTracking إلى `equipment.asset`.

ثم نضيف زر **Schedule Review** يقوم بالآتي:

1. يتحقق من صلاحيات المستخدم.
2. يبحث عن مراجعة مفتوحة لنفس الجهاز والمستخدم.
3. ينشئ مراجعة عند عدم وجودها.
4. يسجل ملاحظة بأن المراجعة طُلبت.

تُسند المراجعة إلى المدير الذي ضغط الزر، بموعد بعد يومين.

---

### 1. تحديد الملفات المطلوبة

سنستخدم الملفات التالية داخل `equipment_training`:

```text
equipment_training/
├── __manifest__.py
├── models/
│   └── equipment_asset.py
├── data/
│   └── equipment_activity_data.xml
└── views/
    └── equipment_mail_views.xml
```

نعدّل Python الحالية للجهاز، ونضيف ملفي XML.

لا ننشئ Model جديدة للأجهزة.

---

### 2. إضافة `mail` إلى Dependencies

داخل `__manifest__.py`:

```python
"depends": [
    "base",
    "web",
    "mail",
],
```

احتفظ بأي Dependencies إضافية موجودة في مشروعك.

---

### 3. إضافة Mixins إلى Model الحالية

افتح:

```text
models/equipment_asset.py
```

داخل Class الأصلية `EquipmentAsset`، أضف `_inherit`:

```python
class EquipmentAsset(models.Model):
    _name = "equipment.asset"
    _description = "Equipment Asset"

    _inherit = [
        "mail.thread",
        "mail.activity.mixin",
    ]

    _order = "name, id"
    _check_company_auto = True
```

هذا مقتطف لبداية Class فقط.

احتفظ بجميع Fields والـConstraints والـMethods التي أضفناها سابقًا، بما فيها قواعد انتقال الحالة وConstraint الرقم التسلسلي.

---

### 4. تفعيل Tracking على الحقول الحالية

عدّل تعريف `name` الموجود:

```python
name = fields.Char(
    required=True,
    tracking=True,
)
```

وعدّل تعريف `state` الموجود:

```python
state = fields.Selection(
    [
        ("available", "Available"),
        ("in_use", "In Use"),
        ("retired", "Retired"),
    ],
    default="available",
    required=True,
    copy=False,
    tracking=True,
)
```

أضفنا Tracking مع الاحتفاظ بالقيم والخصائص السابقة.

لا تضف تعريفًا ثانيًا للحقل نفسه داخل Class.

---

### 5. تعريف نوع مراجعة الأجهزة

أنشئ:

```text
data/equipment_activity_data.xml
```

واكتب:

```xml
<odoo>
    <record
        id="mail_activity_type_equipment_review"
        model="mail.activity.type"
    >
        <field name="name">Equipment Review</field>

        <field name="summary">Review equipment</field>

        <field name="res_model">equipment.asset</field>

        <field name="icon">check</field>

        <field name="delay_count">2</field>

        <field name="delay_unit">days</field>
    </record>
</odoo>
```

External ID الكاملة:

```text
equipment_training.mail_activity_type_equipment_review
```

الحقل `res_model` يجعل النوع مخصصًا للأجهزة.

وإعداد التأخير يوفر قيمة مناسبة عند استخدام هذا النوع من الواجهة. سنحدد الموعد صراحةً أيضًا في Method الخاصة بالزر.

---

### 6. إضافة Method الجدولة

في `models/equipment_asset.py`، أضف Import:

```python
from datetime import timedelta
```

وتأكد أن Imports الحالية تتضمن:

```python
from odoo import _, api, fields, models
from odoo.exceptions import AccessError, UserError
```

احتفظ بأي Imports أخرى يحتاجها الملف.

داخل Class الحالية، أضف:

```python
    def action_schedule_review(self):
        if not self.env.user.has_group(
            "equipment_training.group_equipment_manager"
        ):
            raise AccessError(
                _("Only equipment managers can schedule this review.")
            )

        self.check_access("read")
        self.check_access("write")

        activity_type_xmlid = (
            "equipment_training."
            "mail_activity_type_equipment_review"
        )

        for asset in self:
            existing_review = asset.activity_search(
                [activity_type_xmlid],
                user_id=self.env.user.id,
                additional_domain=[
                    ("active", "=", True),
                ],
                only_automated=False,
            )

            if existing_review:
                continue

            deadline = (
                fields.Date.context_today(asset)
                + timedelta(days=2)
            )

            asset.activity_schedule(
                activity_type_xmlid,
                date_deadline=deadline,
                user_id=self.env.user.id,
                summary=_("Review equipment."),
            )

            asset.message_post(
                body=_(
                    "A review has been requested for %(equipment)s.",
                    equipment=asset.display_name,
                ),
                message_type="comment",
                subtype_xmlid="mail.mt_note",
            )

        return True
```

#### فحص الصلاحيات

الزر مخصص للمدير، لكننا نتحقق من المجموعة داخل Python أيضًا.

ونفحص القراءة والتعديل على الأجهزة المستهدفة، مع الاحتفاظ بقيود الشركات.

لم نستخدم `sudo()`.

#### البحث عن مراجعة مفتوحة

`activity_search()` تبحث عن النوع المحدد على الجهاز الحالي.

أضفنا:

- المستخدم الحالي.
- شرط `active=True` لاختيار الأنشطة المفتوحة.
- `only_automated=False` لاحتساب النشاط سواء أُنشئ بالـHelper أويدويًا من الواجهة.

إذا وجدت مراجعة مطابقة، نتجاوز إنشاء مراجعة إضافية لهذا الجهاز.

#### حدود فحص التكرار

الفحص يتجنب التكرار المعتاد عند الضغط مرة أخرى بعد اكتمال الطلب السابق.

لكنه ليس Constraint تضمن التفرد بين طلبين متزامنين؛ إن أصبح ذلك مطلبًا، نحتاج إلى تصميم يناسب التزامن.

#### تسجيل الملاحظة

نسجل أن المراجعة طُلبت بعد إنشاء النشاط.

هذه الرسالة لا تقول إن الجهاز تمت مراجعته؛ الإنجاز يحدث لاحقًا عند تنفيذ العمل.

---

### 7. إضافة Chatter والزر إلى Form

أنشئ:

```text
views/equipment_mail_views.xml
```

واكتب:

```xml
<odoo>
    <record
        id="view_equipment_form_mail"
        model="ir.ui.view"
    >
        <field name="name">
            equipment.asset.form.mail
        </field>

        <field name="model">
            equipment.asset
        </field>

        <field
            name="inherit_id"
            ref="equipment_training.view_equipment_form"
        />

        <field name="arch" type="xml">
            <xpath expr="//form/header" position="inside">
                <button
                    name="action_schedule_review"
                    type="object"
                    string="Schedule Review"
                    groups="equipment_training.group_equipment_manager"
                />
            </xpath>

            <xpath expr="//form/sheet" position="after">
                <chatter/>
            </xpath>
        </field>
    </record>
</odoo>
```

نستخدم Header الموجودة في Form الجهاز لإضافة الزر.

ونضيف Chatter بعد `sheet`.

تبقى الأزرار والحقول السابقة موجودة؛ View الحالية توسعها باستخدام Inheritance.

إذا أضفت Chatter مسبقًا بنفسك، احتفظ بعنصر واحد بدل إضافتها مرة أخرى.

---

### 8. تسجيل XML في Manifest

تصبح قائمة `data`، مع ملفات الشابترات السابقة:

```python
"data": [
    "security/groups.xml",
    "security/ir.access.csv",
    "data/equipment_category_data.xml",
    "data/equipment_activity_data.xml",
    "views/equipment_views.xml",
    "views/equipment_mail_views.xml",
    "views/equipment_actions.xml",
    "views/equipment_loan_views.xml",
    "wizard/equipment_loan_reschedule_wizard_views.xml",
    "views/equipment_loan_reschedule_views.xml",
    "views/equipment_menus.xml",
    "report/equipment_templates.xml",
    "report/equipment_reports.xml",
],
```

نحمّل Form الجهاز الأصلية قبل View التي ترث منها.

ونحمّل نوع النشاط ضمن Data الخاصة بالـModule.

لا نضيف صلاحيات عامة جديدة على Mail Models؛ Module `mail` توفر آلياتها، ونحتفظ بصلاحيات الأجهزة الحالية.

---

### 9. تطبيق Upgrade

بعد حفظ الملفات:

1. أوقف Server التدريب.
2. أعد تشغيله لتحميل Python الجديدة.
3. سجل الدخول بحساب يستطيع إدارة Modules.
4. افتح **Apps**.
5. احذف Filter **Apps**.
6. ابحث عن `Equipment Training`.
7. نفّذ **Upgrade**.

تحتاج الإضافات إلى Upgrade لتحميل Dependency وحقول Mixins وXML الجديدة.

راجع [[03 Modules & Addons]] إذا أردت تنفيذ Upgrade باستخدام CLI.

---

### 10. تجربة Chatter وTracking

سجل الدخول بحساب تدريب يحمل مجموعة Equipment Manager.

ثم:

1. افتح **Equipment → Assets**.
2. افتح جهازًا محفوظًا.
3. تحقق من ظهور Chatter.
4. عدّل اسم الجهاز واحفظ.
5. تحقق من ظهور تغيير الاسم في Chatter.
6. إذا كان الجهاز `available`، استخدم **Start Use**.
7. تحقق من تسجيل انتقاله إلى `in_use`.
8. استخدم **Return** إذا أردت إعادته إلى `available`.

نستخدم أزرار انتقال الحالة الموجودة؛ فهي تمر بقواعد العمل السابقة.

يمكنك أيضًا إضافة ملاحظة من **Log note**:

```text
The equipment is ready for inspection.
```

النتيجة المتوقعة هي ظهور الملاحظة مرتبطة بالجهاز الحالي.

---

### 11. جدولة المراجعة وإنجازها

على الجهاز نفسه:

1. اضغط **Schedule Review**.
2. تحقق من ظهور Activity من نوع **Equipment Review**.
3. تحقق من إسنادها إلى المستخدم الحالي.
4. تحقق من الموعد النهائي بعد يومين.
5. تحقق من ظهور ملاحظة طلب المراجعة.
6. اضغط الزر مرة أخرى بعد اكتمال الطلب الأول.

المتوقع ألا تُنشأ مراجعة مفتوحة إضافية لنفس الجهاز والمستخدم.

بعد تنفيذ المراجعة:

1. افتح Activity.
2. اختر **Mark Done** أوالإجراء المقابل في الواجهة.
3. أدخل Feedback مناسبة، مثل:

```text
Condition and serial number checked.
```

4. أكد الإنجاز.

المتوقع أن تخرج من قائمة الأنشطة المفتوحة، وأن تظهر نتيجة الإنجاز في Chatter.

بعد اكتمالها، يمكن إنشاء مراجعة جديدة لاحقًا.

---

### 12. التحقق من الحالات الأخرى

| الحالة | النتيجة المتوقعة |
| --- | --- |
| تعديل Field عليها Tracking | ظهور التغيير بعد نجاح الحفظ |
| تعديل Field غير متتبعة | لا نتوقع سجل تغيير تلقائيًا لها |
| رفض عملية بسبب Business Rule | لا يثبت التغيير المرفوض |
| تكرار طلب المراجعة لنفس المستخدم | تجاوز الإنشاء عند وجود مراجعة مفتوحة |
| إنجاز المراجعة ثم طلب أخرى | إنشاء مراجعة جديدة |
| دخول Equipment User العادي | عدم ظهور الزر المخصص للمدير |
| استدعاء Method بحساب غير مدير | رفضها داخل Python |
| محاولة التعامل مع جهاز خارج الصلاحيات | رفض فحص الوصول |
| إنشاء Activity | لا نعتبر ذلك إثباتًا لتسليم Email |

استخدم حسابات تدريب عادية عند فحص الصلاحيات، دون Superuser Mode.

---

## Notifications & Security

### Message وNotification وEmail

هذه نتائج مختلفة:

```text
حفظ Message
    ↓
تحديد من يحتاج إلى Notification
    ↓
اختيار قناة التنبيه
    ↓
محاولة التسليم
```

قد تكون القناة داخل Odoo أوعبر Email، بحسب الإعدادات والسياق.

لذلك:

```text
Message محفوظة
≠
Email وصلت

Activity مجدولة
≠
العمل أُنجز
```

يمكن أن ينشئ Chatter تنبيهات دون Mail Template مخصصة.

أما Mail Templates فتفيد عند الحاجة إلى صياغة بريد قابلة لإعادة الاستخدام.

---

### ما الذي يؤثر في تسليم Email؟

عندما يتضمن المطلب إرسال Email، راجع:

- المستلمين الفعليين.
- عناوينهم.
- إعدادات التنبيهات والاشتراك.
- إعدادات الإرسال الخاصة بالبيئة.
- حالة الرسالة والأخطاء.
- آلية إعادة المحاولة عند الفشل.

إضافة `mail` إلى Dependencies لا تكفي وحدها لضمان تسليم البريد.

وسنتناول العمليات التلقائية في [[22 Automation & Scheduled Actions]].

---

### صلاحيات Chatter في مشروعنا

Equipment User يملك قراءة الأجهزة، بينما يملك Equipment Manager صلاحيات أوسع عليها.

في `mail.thread`، الوصول المطلوب للنشر على المستند يحدد عادةً عبر:

```python
_mail_post_access = "write"
```

لذلك قد يستطيع المستخدم قراءة الجهاز وسجل المحادثة، دون السماح له بإضافة ملاحظة بالطريقة نفسها التي يستطيعها المدير.

لا توسع صلاحيات الجهاز أوتخفض متطلبات النشر لمجرد إظهار الأزرار؛ حدد أولًا ما يحتاجه الدور وظيفيًا.

---

### المتابعة والتكليف لا يمنحان صلاحية تلقائية

إضافة شخص كمتابع أوإسناد Activity إليه ليست بديلًا عن منحه صلاحيات مناسبة على Business Record.

قبل التكليف، تحقق من أن الشخص يستطيع الوصول إلى البيانات التي يحتاجها لتنفيذ العمل.

كما أن التنبيه قد ينقل محتوى خارج شاشة Record، لذلك راجع المستلمين ومحتوى الرسائل عند التعامل مع بيانات حساسة.

---

### Tracking وسجل التدقيق

Tracking تساعد على فهم التغييرات المعتادة.

لكنها ليست بمفردها سجل تدقيق لا يمكن تغييره:

- تختار Fields التي تدخل في التتبع.
- يمكن لبعض سياقات التنفيذ تعطيل Tracking.
- التغييرات خارج مسار ORM المعتاد لا تكتسب Tracking تلقائيًا.
- الرسائل لها صلاحيات وسلوك إدارة خاص بها.

إذا كان المشروع يحتاج Audit History بمتطلبات محددة، فصمم تلك المتطلبات صراحةً.

---

## Important Notes

- أضف Dependency على `mail` قبل استخدام Mixins الخاصة بها.
- `mail.thread` تضيف إمكانيات المحادثة، و`mail.activity.mixin` تضيف إدارة الأنشطة.
- `<chatter/>` تعرض الواجهة، وتحتاج إلى Model مجهزة بالإمكانيات المناسبة.
- احتفظ بحالات Business Model وقواعدها عند إضافة Mail؛ لا تستبدلها بحالات مختلفة من مثال آخر.
- `tracking=True` تختار الحقول المتتبعة، ولا تعيد إنشاء تاريخها السابق.
- `message_post()` تعمل على Record واحدة محفوظة، والنص العادي يُعالج باستخدام Escaping.
- `partner_ids` في الرسائل تشير إلى Partners، بينما `user_id` في النشاط يشير إلى User.
- `activity_schedule()` لا تمنع التكرار تلقائيًا؛ حدد سياسة واضحة للأنشطة المتشابهة.
- في Odoo 20، مسار إنجاز Activity المعتاد يؤرشفها ويسجل رسالة الإنجاز.
- جدولة Activity لا تعني إنجاز العمل أوتغيير حالة الجهاز.
- الملاحظة الداخلية ليست محتوى خاصًا بالكاتب وحده، وقد تؤدي إلى تنبيهات بحسب المستلمين والسياق.
- صلاحيات السجل الأصلي ومحتوى الإشعارات تظل جزءًا أساسيًا من التصميم.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[22 Automation & Scheduled Actions]]

---

## Sources

- [Odoo 20 Documentation — Mixins and Useful Classes](https://www.odoo.com/documentation/20.0/developer/reference/backend/mixins.html)
- [Odoo 20 Documentation — Activities](https://www.odoo.com/documentation/20.0/applications/essentials/activities.html)
- [Odoo 20 Documentation — Chatter](https://www.odoo.com/documentation/20.0/applications/productivity/discuss/chatter.html)
- [Odoo 20 — Mail Thread: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/mail/models/mail_thread.py)
- [Odoo 20 — Field Tracking: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/mail/models/mail_track_mixin.py)
- [Odoo 20 — Activity Mixin: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/mail/models/mail_activity_mixin.py)
- [Odoo 20 — Activities: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/mail/models/mail_activity.py)
- [Odoo 20 — Activity Types: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/mail/models/mail_activity_type.py)
- [Odoo 20 — Mail Access Definitions: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/mail/security/ir.access.csv)