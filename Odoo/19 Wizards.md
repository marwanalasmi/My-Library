---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 19 Wizards

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Wizard Fundamentals|Wizard Fundamentals]]
- [[#Wizard Lifecycle|Wizard Lifecycle]]
- [[#Practical Example|Practical Example]]
- [[#How to Use|How to Use]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في الشابترات السابقة، أنشأنا Models لحفظ الأجهزة والإعارات، ثم أضفنا Views وButtons لتنفيذ العمليات عليها.

لكن بعض العمليات تحتاج إلى مدخلات إضافية قبل تنفيذها.

مثلًا، عند تعديل فترة إعارة، نريد أن يحدد المستخدم وقت البداية والنهاية الجديدين، ثم يؤكد التعديل.

يمكننا تقديم هذه الخطوات داخل نافذة قصيرة تسمى **Wizard**:

```text
فتح الإعارة
    ↓
الضغط على Reschedule
    ↓
إدخال الفترة الجديدة
    ↓
تأكيد العملية
    ↓
تحديث الإعارة
```

الـWizard تجمع المدخلات المؤقتة، بينما تُحفظ النتيجة المطلوبة على Business Model المناسبة.

في هذا الشابتر، سننشئ Wizard لتعديل فترة إعارة موجودة داخل `equipment.loan`.

سنستخدم الحقول التي أنشأناها سابقًا:

- `start_at`
- `end_at`
- `duration_hours`

وسيظل التحقق من ترتيب التواريخ مسؤولية Constraint الموجودة في [[17 Constraints]].

---

## Wizard Fundamentals

### ما هي Wizard؟

الـWizard واجهة ترشد المستخدم خلال عملية محددة، مثل:

- اختيار فترة جديدة لإعارة.
- إدخال سبب لعملية.
- تحديد خيارات قبل إنشاء تقرير.
- تطبيق عملية على مجموعة Records.

قد تحتوي Wizard على شاشة واحدة أو عدة خطوات.

في المثال الحالي، نحتاج شاشة واحدة تجمع وقت البداية والنهاية، ثم تنفذ التعديل.

---

### العلاقة بين Wizard وTransientModel

نستخدم عادةً `models.TransientModel` لإنشاء Model تحمل مدخلات الـWizard:

```python
from odoo import fields, models


class EquipmentLoanRescheduleWizard(models.TransientModel):
    _name = "equipment.loan.reschedule.wizard"
    _description = "Reschedule Equipment Loan"

    new_start_at = fields.Datetime(
        string="New Start",
        required=True,
    )
```

هذه Class تعرّف Model حقيقية داخل Odoo.

لكنها لا تنشئ النافذة وحدها؛ ما زلنا نحتاج إلى Form View وطريقة لفتحها.

تتكون Wizard المعتادة من:

| الجزء | مسؤوليته |
| --- | --- |
| TransientModel | حمل المدخلات المؤقتة |
| Form View | عرض المدخلات والأزرار |
| Window Action | فتح النموذج داخل Dialog |
| Object Method | معالجة التأكيد وتنفيذ العملية |

يمكن أيضًا فتح Form خاصة بـModel عادية داخل Dialog؛ لذلك فتح نافذة منبثقة وحده لا يجعل Model مؤقتة.

---

### أين تُحفظ Records المؤقتة؟

Records الخاصة بـ`TransientModel` تُحفظ في Database.

فهي ليست مجرد متغيرات داخل Memory.

الاختلاف أن Odoo يملك آلية لتنظيف Records المؤقتة وفق إعدادات العمر والعدد.

> [!important] مؤقتة لا تعني أنها تُحذف عند إغلاق النافذة
> قد تبقى Record الخاصة بالـWizard بعد إغلاق Dialog إلى أن تُحذف بواسطة آلية التنظيف.
>
> لا تعتمد على موعد بقائها أو حذفها لحفظ بيانات تشغيل دائمة.

---

### أين نحفظ نتيجة العملية؟

نحفظ النتيجة على Model التي تمثل بيانات العمل.

في مثالنا:

```text
equipment.loan.reschedule.wizard
    المدخلات المؤقتة للفترة الجديدة
                ↓
equipment.loan
    الفترة المعتمدة للإعارة
```

إذا احتجنا لاحقًا إلى سجل دائم لكل تعديل، مع الفترة السابقة والجديدة وسبب التغيير، فننشئ له تخزينًا دائمًا مناسبًا.

بقاء Record مؤقتة للـWizard لا يوفر Audit History موثوقًا.

---

### الصلاحيات في Odoo 20

في Odoo 20، تستخدم `TransientModel` آليات الصلاحيات نفسها المستخدمة مع Models العادية.

لذلك نحدد صراحةً:

1. المجموعة التي تستطيع استخدام Wizard.
2. العمليات المسموحة على Records الخاصة بها.
3. القيود التي تحدد أي Records يستطيع المستخدم الوصول إليها.

في التطبيق، سنسمح لمجموعة Equipment Manager باستخدام Wizard، ونقصر الوصول إلى Records المؤقتة التي أنشأها المستخدم نفسه.

أما الإعارة المرتبطة، فتبقى خاضعة لصلاحيات `equipment.loan`.

راجع [[11 Security & Multi-Company]] لفهم تعريف الصلاحيات باستخدام `ir.access`.

---

## Wizard Lifecycle

### من فتح النافذة إلى حفظ النتيجة

تمر العملية المعتادة بهذا التسلسل:

```text
Button على Business Record
            ↓
Python Method تعيد Window Action
            ↓
فتح Form للـWizard
            ↓
تعبئة القيم الافتراضية
            ↓
المستخدم يعدّل المدخلات
            ↓
Button تستدعي Object Method
            ↓
التحقق من الصلاحيات والمدخلات
            ↓
تحديث Business Record
            ↓
إغلاق Dialog
```

فتح Wizard لا يجب أن ينفذ التعديل الذي ما زال المستخدم سيؤكده.

في مثالنا، تتغير فترة الإعارة بعد الضغط على **Apply**.

---

### تمرير القيم الافتراضية باستخدام Context

يمكن تمرير قيمة افتراضية لحقل باستخدام النمط:

```text
default_<field_name>
```

مثلًا، لتعبئة `loan_id`:

```python
{
    "default_loan_id": self.id,
}
```

ولتعبئة الفترة الحالية داخل الحقول الجديدة:

```python
{
    "default_loan_id": self.id,
    "default_new_start_at": self.start_at,
    "default_new_end_at": self.end_at,
}
```

هذه القيم تهيئ Form عند فتحها، كما تعلمنا في [[09 Environment & Context]].

> [!warning] Default ليست صلاحية
> تمرير `default_loan_id` لا يثبت أن المستخدم مسموح له بتعديل الإعارة.
>
> يجب التحقق من صلاحيات الإعارة عند تنفيذ العملية.

---

### فتح Wizard داخل Dialog

تعيد Method من Python وصف Window Action:

```python
return {
    "type": "ir.actions.act_window",
    "name": self.env._("Reschedule Loan"),
    "res_model": "equipment.loan.reschedule.wizard",
    "views": [(False, "form")],
    "target": "new",
}
```

هنا:

| المفتاح | المعنى |
| --- | --- |
| `res_model` | Model التي سنعرضها |
| `views` | أنواع Views المطلوبة، ومعرّفاتها إن حددناها |
| `target` | مكان فتح العرض |

القيمة `"new"` في `target` تفتح العرض داخل Dialog.

واستخدام `False` في موضع View ID يطلب من Odoo اختيار Form View المناسبة.

في التطبيق، سنحدد View المطلوبة باستخدام External ID.

---

### أزرار التأكيد والإلغاء

زر التأكيد يستدعي Method باستخدام `type="object"`:

```xml
<button
    name="action_apply"
    type="object"
    string="Apply"
    class="btn-primary"
/>
```

أما زر الإلغاء:

```xml
<button
    string="Cancel"
    special="cancel"
/>
```

فيغلق Dialog ويتجاهل التعديلات غير المحفوظة في النموذج، دون استدعاء `action_apply()`.

هذا لا يعني التراجع عن عمليات سابقة اكتملت في طلبات أخرى.

راجع [[13 Actions, Menus & Buttons]] لفهم أنواع الأزرار والـActions.

---

### فصل المدخلات عن Business Logic

الـWizard تعرف كيف تجمع المدخلات وتتعامل مع تأكيد المستخدم.

أما العملية الأساسية، فمن الأفضل أن تكون داخل Business Model، بحيث يمكن إعادة استخدامها من مسارات أخرى.

سنستخدم هذا التصميم:

```text
Wizard.action_apply()
        ↓
Loan.action_reschedule()
        ↓
Loan.write()
        ↓
Constraints + Recompute
```

بهذا تستفيد العملية من:

- صلاحيات الإعارة.
- Constraint الخاصة بترتيب التواريخ.
- إعادة حساب `duration_hours`.

ولا نحتاج إلى تكرار حساب المدة أو شرط ترتيب التواريخ داخل Wizard.

---

## Practical Example

سنضيف Wizard باسم **Reschedule Loan** إلى Module التدريب الحالية `equipment_training`.

تسمح للمدير بتحديد بداية ونهاية جديدتين لإعارة محفوظة.

> [!note] متطلبات التطبيق
> يفترض المثال أنك طبقت الأجزاء السابقة التي أنشأنا فيها:
>
> - Model باسم `equipment.loan`.
> - الحقول `start_at` و`end_at` و`duration_hours`.
> - Constraint تمنع أن تكون النهاية مساوية للبداية أو قبلها.
> - مجموعة `equipment_training.group_equipment_manager`.
> - Form View بالـExternal ID التالية:
>   `equipment_training.view_equipment_loan_form`.
>
> سنضيف إلى الملفات الحالية مع الاحتفاظ بالحقول والـMethods السابقة.

---

### 1. تحديد الملفات المطلوبة

الأجزاء التي سنستخدمها من Module:

```text
equipment_training/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── equipment_loan.py
├── security/
│   ├── groups.xml
│   └── ir.access.csv
├── views/
│   ├── equipment_loan_views.xml
│   └── equipment_loan_reschedule_views.xml
└── wizard/
    ├── __init__.py
    ├── equipment_loan_reschedule_wizard.py
    └── equipment_loan_reschedule_wizard_views.xml
```

المجلد `wizard` يجمع ملفات Wizard في مكان واضح.

وجود المجلد وحده لا يكفي لتحميل Python أوXML؛ سنربط الملفات باستخدام Imports وManifest.

---

### 2. إضافة العملية إلى Business Model

افتح `models/equipment_loan.py`.

تأكد أن Imports الحالية تتضمن:

```python
from odoo import _, api, fields, models
from odoo.exceptions import (
    AccessError,
    UserError,
    ValidationError,
)
```

احتفظ أيضًا بأي Imports سابقة، مثل `timedelta` المستخدمة لحساب الفترة.

داخل Class الحالية `EquipmentLoan`، أضف الـMethods التالية:

```python
    def _check_reschedule_access(self):
        self.ensure_one()

        if not self.env.user.has_group(
            "equipment_training.group_equipment_manager"
        ):
            raise AccessError(
                _("Only equipment managers can reschedule loans.")
            )

        self.check_access("read")
        self.check_access("write")

    def action_reschedule(self, start_at, end_at):
        self._check_reschedule_access()

        start_at = fields.Datetime.to_datetime(start_at)
        end_at = fields.Datetime.to_datetime(end_at)

        if not start_at or not end_at:
            raise ValidationError(
                _("Enter both a start time and an end time.")
            )

        return self.write({
            "start_at": start_at,
            "end_at": end_at,
        })

    def action_open_reschedule_wizard(self):
        self._check_reschedule_access()

        wizard_view = self.env.ref(
            "equipment_training."
            "view_equipment_loan_reschedule_wizard_form"
        )

        return {
            "type": "ir.actions.act_window",
            "name": _("Reschedule Loan"),
            "res_model": "equipment.loan.reschedule.wizard",
            "views": [(wizard_view.id, "form")],
            "target": "new",
            "context": {
                **self.env.context,
                "default_loan_id": self.id,
                "default_new_start_at": self.start_at,
                "default_new_end_at": self.end_at,
            },
        }
```

هذه إضافات داخل Class الموجودة؛ لا تنشئ Class ثانية بالاسم التقني نفسه باستخدام `_name`.

#### ماذا تحقق هذه الإضافات؟

`_check_reschedule_access()` تجمع فحص الصلاحيات المشتركة بين فتح Wizard وتنفيذ العملية.

تتحقق من:

- معالجة إعارة واحدة.
- عضوية المستخدم في مجموعة المدير.
- صلاحية قراءة الإعارة وتعديلها.

`action_reschedule()` تنفذ التعديل على الإعارة نفسها.

واستخدمنا `fields.Datetime.to_datetime()` لتحويل قيم التواريخ المدعومة إلى الشكل الذي نتعامل معه داخل Python.

وجود البداية والنهاية مطلوب لهذه العملية، حتى لو كانت الحقول الأصلية تسمح بإعارة لم تُحدد فترتها بعد.

أما ترتيب التواريخ، فتتحقق منه Constraint السابقة عند `write()`.

بعد نجاح التعديل، يعيد Odoo حساب `duration_hours` اعتمادًا على Dependencies التي عرفناها في [[16 Computed Fields, Related Fields & Onchange]].

#### لماذا نتحقق عند الفتح وعند التنفيذ؟

قد تتغير الصلاحيات أو الشركات المفعلة بين فتح النافذة والضغط على **Apply**.

كما أن Public Methods يمكن استدعاؤها عبر RPC.

لذلك نتحقق عند التنفيذ أيضًا، ولا نعتمد على ظهور الزر.

> [!note] نطاق العملية
> هذه Method توفر مسارًا واضحًا لإعادة الجدولة.
>
> صلاحيات المدير الحالية على `equipment.loan` تسمح له أيضًا بالتعديل المباشر وفق القواعد السابقة. لم نضف قاعدة تفرض مرور كل تعديل للتواريخ عبر Wizard.

---

### 3. إنشاء Wizard Model

أنشئ الملف:

```text
wizard/equipment_loan_reschedule_wizard.py
```

واكتب:

```python
from odoo import _, fields, models
from odoo.exceptions import UserError


class EquipmentLoanRescheduleWizard(models.TransientModel):
    _name = "equipment.loan.reschedule.wizard"
    _description = "Reschedule Equipment Loan"

    loan_id = fields.Many2one(
        "equipment.loan",
        string="Loan",
        required=True,
        ondelete="cascade",
    )

    new_start_at = fields.Datetime(
        string="New Start",
        required=True,
    )

    new_end_at = fields.Datetime(
        string="New End",
        required=True,
    )

    def action_apply(self):
        self.ensure_one()

        self.check_access("read")
        self.check_access("write")

        loan = self.loan_id.exists()

        if not loan:
            raise UserError(
                _("The loan no longer exists. Close this window.")
            )

        loan.action_reschedule(
            self.new_start_at,
            self.new_end_at,
        )

        return {
            "type": "ir.actions.act_window_close",
        }
```

الحقل `loan_id` يربط المدخلات المؤقتة بالإعارة التي سنعدلها.

أما `new_start_at` و`new_end_at` فيحملان القيم المقترحة إلى أن يؤكد المستخدم العملية.

داخل `action_apply()`:

1. نتحقق من أن العملية تخص Wizard واحدة.
2. نتحقق من صلاحية الوصول إلى Record الخاصة بالـWizard.
3. نتحقق من بقاء الإعارة موجودة.
4. نستدعي العملية على الإعارة.
5. نعيد Action لإغلاق Dialog بعد النجاح.

لم نستخدم `sudo()`؛ التنفيذ يجري بصلاحيات المستخدم الحالي.

#### اتجاه `ondelete="cascade"`

تعني هذه القيمة أن حذف الإعارة المرتبطة يؤدي إلى حذف Records الخاصة بالـWizard التي تشير إليها على مستوى العلاقة.

حذف Wizard أو تنظيفها لا يحذف الإعارة.

---

### 4. ربط Python Imports

داخل `wizard/__init__.py`:

```python
from . import equipment_loan_reschedule_wizard
```

وفي ملف `__init__.py` الرئيسي للـModule، احتفظ باستيراد Models وأضف Wizard:

```python
from . import models
from . import wizard
```

بهذا تُحمّل Class الخاصة بالـWizard عند تحميل Python الخاصة بالـModule.

---

### 5. تعريف صلاحيات Wizard

افتح:

```text
security/ir.access.csv
```

أضف الصفين التاليين إلى الصفوف الموجودة:

```csv
access_loan_reschedule_manager,Loan Reschedule Manager,equipment.loan.reschedule.wizard,equipment_training.group_equipment_manager,cru,
restrict_loan_reschedule_owner,Own Loan Reschedule Wizards,equipment.loan.reschedule.wizard,,crud,"[('create_uid', '=', user.id)]"
```

يجب أن يبقى Header الموجود في أول الملف مرة واحدة:

```csv
id,name,model_id,group_id/id,operation,domain
```

لا تستبدل صلاحيات الأجهزة والإعارات السابقة بهذين الصفين.

#### الصف الأول: منح الصلاحية

يمنح مجموعة Equipment Manager العمليات التالية على Wizard:

- Create
- Read
- Update

لم نمنح Delete؛ إغلاق Dialog لا يحتاج إلى حذف Record المؤقتة يدويًا.

#### الصف الثاني: تقييد الوصول

لأن `group_id` فارغة، فالصف يمثل قيدًا عامًا.

يسمح بالوصول فقط إلى Records التي يساوي منشئها المستخدم الحالي:

```python
[("create_uid", "=", user.id)]
```

هذا القيد لا يمنح صلاحية بمفرده؛ يجب أن تتوافر أيضًا الصلاحية المناسبة من الصف الأول.

> [!important] صلاحيات Wizard وصلاحيات الإعارة منفصلتان
> السماح بإنشاء Wizard لا يمنح صلاحية تعديل كل الإعارات.
>
> عند تطبيق العملية، نفحص الإعارة المرتبطة وتطبق عليها صلاحياتها وقيود الشركات الحالية.

---

### 6. إنشاء Form View للـWizard

أنشئ الملف:

```text
wizard/equipment_loan_reschedule_wizard_views.xml
```

واكتب:

```xml
<odoo>
    <record
        id="view_equipment_loan_reschedule_wizard_form"
        model="ir.ui.view"
    >
        <field name="name">
            equipment.loan.reschedule.wizard.form
        </field>

        <field name="model">
            equipment.loan.reschedule.wizard
        </field>

        <field name="arch" type="xml">
            <form string="Reschedule Loan">
                <group>
                    <field
                        name="loan_id"
                        readonly="True"
                        options="{'no_open': True}"
                    />

                    <field name="new_start_at"/>

                    <field name="new_end_at"/>
                </group>

                <footer>
                    <button
                        name="action_apply"
                        type="object"
                        string="Apply"
                        class="btn-primary"
                    />

                    <button
                        string="Cancel"
                        special="cancel"
                    />
                </footer>
            </form>
        </field>
    </record>
</odoo>
```

عرضنا الإعارة للقراءة فقط حتى يعرف المستخدم أي إعارة يعدلها.

واستخدمنا `no_open` لمنع فتح الإعارة من هذا الحقل أثناء هذه الخطوة.

هذه إعدادات واجهة، وليست قيودًا أمنية على Server.

أما `<footer>` فتجمع أزرار تنفيذ العملية وإلغاء النافذة في موضع مناسب للـDialog.

---

### 7. إضافة زر إلى Form الإعارة

سنستخدم View Inheritance التي تعلمناها في [[18 Inheritance & Module Interaction]].

أنشئ الملف:

```text
views/equipment_loan_reschedule_views.xml
```

واكتب:

```xml
<odoo>
    <record
        id="view_equipment_loan_form_reschedule"
        model="ir.ui.view"
    >
        <field name="name">
            equipment.loan.form.reschedule
        </field>

        <field name="model">
            equipment.loan
        </field>

        <field
            name="inherit_id"
            ref="equipment_training.view_equipment_loan_form"
        />

        <field name="arch" type="xml">
            <xpath expr="//form/sheet" position="before">
                <header>
                    <button
                        name="action_open_reschedule_wizard"
                        type="object"
                        string="Reschedule"
                        groups="equipment_training.group_equipment_manager"
                    />
                </header>
            </xpath>
        </field>
    </record>
</odoo>
```

هذا المثال يعتمد على Form الإعارة التي أنشأناها سابقًا، والتي تبدأ بـ`sheet` ولا تحتوي `header`.

أضفنا Header قبلها، ثم وضعنا زر **Reschedule** داخله.

إذا كنت قد أضفت Header إلى Form بنفسك، فأضف الزر داخل Header الموجودة بدل إنشاء واحدة أخرى.

`groups` تجعل الزر ظاهرًا للمديرين، بينما يبقى التحقق داخل Python مسؤولًا عن منع الاستدعاء غير المسموح.

---

### 8. تحديث Manifest

أضف ملفي XML الجديدين إلى قائمة `data`، مع الحفاظ على ترتيب الملفات السابقة:

```python
"data": [
    "security/groups.xml",
    "security/ir.access.csv",
    "data/equipment_category_data.xml",
    "views/equipment_views.xml",
    "views/equipment_actions.xml",
    "views/equipment_loan_views.xml",
    "wizard/equipment_loan_reschedule_wizard_views.xml",
    "views/equipment_loan_reschedule_views.xml",
    "views/equipment_menus.xml",
],
```

يجب تحميل Form الإعارة الأصلية قبل View التي ترث منها.

لم ننشئ Window Action Record منفصلة؛ Method فتح Wizard تعيد Action Dictionary مباشرةً.

وكما تعلمنا في [[10 Data Files & External IDs]]، نستخدم External ID للوصول إلى Form الخاصة بالـWizard.

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

نحدّث `equipment_training` لأن الإضافات الحالية موجودة داخلها.

إذا كانت Module التوسعة التي أنشأناها في الشابتر السابق مثبتة، فلا تحتاج إلى نقل ملفات Wizard إليها.

يمكنك مراجعة [[03 Modules & Addons]] إذا أردت تنفيذ Upgrade باستخدام CLI بدل الواجهة.

---

### 10. تجربة العملية

سجل الدخول بحساب تدريب عادي يحمل مجموعة Equipment Manager، مع اختيار الشركة التي تخص الإعارة.

ثم:

1. افتح **Equipment → Loans**.
2. افتح إعارة محفوظة، أو أنشئ إعارة جديدة واحفظها.
3. اضغط **Reschedule**.
4. تحقق أن حقل **Loan** يعرض الإعارة الحالية.
5. أدخل بداية ونهاية جديدتين.
6. اضغط **Apply**.

مثلًا، اختر فترة في اليوم نفسه:

| الحقل | القيمة |
| --- | --- |
| New Start | 09:00 |
| New End | 12:00 |

الناتج المتوقع:

- تُغلق Wizard بعد نجاح العملية.
- تتغير `start_at` و`end_at` على الإعارة.
- تصبح `duration_hours` مساوية لـ`3.0`.

إذا احتاج العرض إلى تحديث، أعد فتح الإعارة أو حدّثها.

الأوقات المعروضة في الواجهة تتبع المنطقة الزمنية للمستخدم.

> [!note] ماذا تنفذ هذه العملية؟
> تعدل فترة الإعارة وتحسب مدتها باستخدام المنطق الموجود.
>
> لم نضف هنا فحصًا لتعارض مواعيد الإعارات أو حجز الأجهزة خلال الفترة.

---

### 11. التحقق من الحالات الأخرى

جرّب الحالات التالية بعد نجاح المثال الأساسي:

| الحالة | النتيجة المتوقعة |
| --- | --- |
| فتح Wizard ثم الضغط على Cancel | تبقى فترة الإعارة كما كانت |
| جعل النهاية قبل البداية | ترفض Constraint التعديل |
| جعل النهاية مساوية للبداية | ترفض Constraint التعديل |
| ترك أحد التاريخين فارغًا | يمنع النموذج التأكيد بسبب `required=True` |
| استدعاء `action_reschedule()` دون أحد التاريخين | ترفع Method رسالة تحقق |
| فتح الإعارة بحساب Equipment User العادي | لا يظهر زر Reschedule |
| محاولة استدعاء العملية بحساب غير مدير عبر RPC أوShell | ترفض Method العملية |
| محاولة تعديل إعارة غير مسموح بها وفق قيود الشركات | يرفض فحص صلاحيات الإعارة |
| محاولة الوصول إلى Wizard أنشأها مستخدم آخر | يمنع قيد `create_uid` الوصول في الوضع العادي |

عند اختبار Constraint، تظل فترة الإعارة السابقة محفوظة إذا رُفض التعديل في الطلب.

ولفحص الصلاحيات، استخدم حسابات تدريب عادية دون تفعيل Superuser Mode.

---

## How to Use

### متى تكون Wizard مناسبة؟

استخدم Wizard عندما تكون المدخلات مرتبطة بعملية محددة، ثم تنتهي الحاجة إليها بعد تنفيذها.

| الحاجة | التصميم المناسب |
| --- | --- |
| اختيار فترة جديدة ثم تطبيقها | Wizard مرتبطة بالإعارة |
| إدخال خيارات قبل إنشاء تقرير | Wizard تجمع خيارات التقرير |
| حفظ إعارة قابلة للمتابعة والبحث لاحقًا | Model عادية |
| حفظ سجل دائم لكل تغيير | Model دائمة أو آلية History مناسبة |
| تعديل حقل مباشرة دون خطوات إضافية | قد تكفي Form الحالية |

لا تحتاج كل عملية إلى Wizard.

إذا كانت العملية واضحة بزر واحد ولا تحتاج مدخلات إضافية، فقد تكون Object Method مباشرة كافية.

---

### تصميم Wizard لعدة Records

المثال الحالي يعالج إعارة واحدة، ولهذا استخدمنا `ensure_one()` وحقل `loan_id`.

إذا احتجت لاحقًا إلى معالجة عدة إعارات، فيجب تصميم العملية لذلك صراحةً:

- كيف نختار Records المستهدفة؟
- هل القيم الجديدة مناسبة لها جميعًا؟
- هل المستخدم يملك الصلاحية على كل Record؟
- هل يجب أن تفشل العملية كاملة إذا رُفضت Record واحدة؟

لا تحوّل المثال إلى عملية جماعية بمجرد حذف `ensure_one()`.

كذلك، قيم Context مثل `active_model` و`active_ids` تساعد في معرفة الاختيار الحالي، لكنها لا تعوض التحقق من Model والصلاحيات.

---

## Important Notes

- Records الخاصة بـ`TransientModel` تُحفظ مؤقتًا في Database؛ لا تستخدمها لحفظ History دائمة.
- في Odoo 20، عرّف صلاحيات Wizard وقيود الوصول إلى Records الخاصة بها صراحةً.
- `target="new"` تحدد فتح Dialog، ولا تحدد ما إذا كانت Model مؤقتة.
- `default_<field_name>` تعبئ قيمًا افتراضية، ولا تثبت صلاحية المستخدم على Record المشار إليها.
- افحص صلاحيات Business Record عند التأكيد؛ قد تختلف الظروف عن وقت فتح Wizard.
- `readonly` و`groups` في View لا تغنيان عن حماية Public Methods على Server.
- إغلاق Dialog لا يعني حذف Wizard فورًا، ولا حذف الإعارة المرتبطة.
- احتفظ بالنتيجة على Business Model، واستفد من Constraints وComputed Fields الموجودة فيها.
- لا تنشئ علاقة `Many2one` من Model دائمة إلى TransientModel لحفظ مرجع دائم للـWizard؛ الاتجاه المستخدم هنا من Wizard إلى الإعارة الدائمة.
- تعديل فترة الإعارة لا يحفظ الفترة القديمة تلقائيًا؛ إن احتجت سجل تغييرات دائمًا، صممه بصورة مستقلة.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[20 QWeb & Reports]]

---

## Sources

- [Odoo 20 — TransientModel: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models_transient.py)
- [Odoo 20 Documentation — Actions](https://www.odoo.com/documentation/20.0/developer/reference/backend/actions.html)
- [Odoo 20 Documentation — View Architectures](https://www.odoo.com/documentation/20.0/developer/reference/user_interface/view_architectures.html)
- [Odoo 20 Documentation — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)
- [Odoo 20 — Access Definitions: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/security/ir.access.csv)
- [Odoo 20 — ORM Access Checks: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)