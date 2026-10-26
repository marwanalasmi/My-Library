---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 18 Inheritance & Module Interaction

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Model Inheritance|Model Inheritance]]
- [[#Method Overrides & Super|Method Overrides & Super]]
- [[#Incremental Field Definitions|Incremental Field Definitions]]
- [[#View Inheritance|View Inheritance]]
- [[#Module Dependencies & Interaction|Module Dependencies & Interaction]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

حتى الآن، أضفنا Fields وMethods وViews داخل Module `equipment_training`.

لكن عندما نحتاج إلى تخصيص Module موجودة، يمكننا إنشاء Module أخرى توسعها.

مثلًا:

```text
equipment_training
    ↓
تعريف الأجهزة وقواعدها وواجهاتها

equipment_training_note
    ↓
إضافة ملاحظة وتوسيع الواجهة والسلوك
```

تبقى ملفات التوسعة داخل Module مستقلة، وتجمع Odoo التعريفات عند تحميلها في Database.

نستخدم آليات مختلفة بحسب المطلوب:

| المطلوب | الآلية |
| --- | --- |
| توسيع Model موجودة | Extension Inheritance |
| إنشاء Model جديدة مبنية على أخرى | Classical Inheritance |
| تفويض Fields إلى Record مرتبطة | Delegation Inheritance |
| تعديل View موجودة | View Inheritance |

سنطبق عمليًا Extension Inheritance وView Inheritance، مع الحفاظ على قواعد الأجهزة من الشابترات السابقة.

---

## Model Inheritance

### Extension Inheritance

نستخدم `_inherit` لاستهداف Model موجودة، ونترك `_name` دون تعريف جديدة:

```python
from odoo import fields, models


class EquipmentAsset(models.Model):
    _inherit = "equipment.asset"

    training_note = fields.Text()
```

النتيجة:

```text
equipment.asset نفسها
    +
training_note
```

لم ننشئ Model باسم `equipment.training.note`.

تظل Records أجهزة من `equipment.asset`، وتصبح Field الجديدة جزءًا من تعريف Model بعد تثبيت Module التوسعة.

هذا النوع مناسب لإضافة:

- Fields.
- Methods.
- Constraints.
- تعديلات على Fields أوMethods موجودة.

---

### التوسعة تشمل Model داخل Database

التغيير لا يقتصر على Records التي تنشئها Module الجديدة.

بعد تثبيت التوسعة، تصبح Field متاحة على أجهزة Model نفسها، بما فيها الأجهزة الموجودة سابقًا.

كما تستخدم Odoo التعريف المركب عند الوصول إلى:

```python
env["equipment.asset"]
```

لا نحتاج إلى اختيار نسخة أخرى من Model للاستفادة من التوسعة.

---

### Classical Inheritance

إذا أعطينا `_name` جديدة مختلفة عن الأصل، مع `_inherit`، ننشئ Model مستقلة مبنية على تعريف Model أخرى.

مثال للمقارنة:

```python
from odoo import fields, models


class SpecialAsset(models.Model):
    _name = "equipment.special.asset"
    _inherit = "equipment.asset"
    _description = "Special Equipment Asset"

    special_code = fields.Char()
```

هنا يوجد اسمان مختلفان:

```text
equipment.asset
equipment.special.asset
```

Model الجديدة تستفيد من Fields وMethods وتعريفات موروثة من الأصل.

لكن Records الموجودة في `equipment.asset` لا تتحول تلقائيًا إلى Records في Model الجديدة.

كذلك، لا تفترض أن تعريفات الوصول والقوائم وViews أصبحت جاهزة للـModel الجديدة؛ تحتاج إلى تصميمها وفق اسمها واستخدامها.

---

### Delegation Inheritance

نستخدم `_inherits` عندما تكون Model مرتبطة بـRecord من Model أخرى، وتفوض إليها الوصول إلى Fields.

مثال توضيحي:

```python
from odoo import fields, models


class TrainingContactProfile(models.Model):
    _name = "training.contact.profile"
    _description = "Training Contact Profile"

    _inherits = {
        "res.partner": "partner_id",
    }

    partner_id = fields.Many2one(
        "res.partner",
        required=True,
        ondelete="cascade",
    )

    reference_code = fields.Char()
```

كل Profile ترتبط بـContact من خلال `partner_id`.

يمكن الوصول إلى Fields المفوضة، مثل اسم Contact، من خلال Profile، لكن تخزينها يبقى في Model الأصلية المرتبطة.

أما `reference_code`، فهي Field محلية في Profile.

Methods الخاصة بـ`res.partner` لا تُورث تلقائيًا بمجرد تعريف `_inherits`.

هذا التصميم يحتاج إلى فهم واضح للعلاقة والصلاحيات ودورة حياة Records. توثيق Odoo ينبه إلى قيود Delegation، خصوصًا عند بناء سلاسل منها.

---

### مقارنة الآليات

| الآلية | التعريف المعتاد | النتيجة |
| --- | --- | --- |
| Extension | `_inherit` دون اسم Model جديد | توسيع Model الموجودة |
| Classical | `_name` مختلفة مع `_inherit` | إنشاء Model جديدة |
| Delegation | `_inherits` مع Many2one | Model تفوض Fields إلى Records مرتبطة |

أمثلة Classical وDelegation هنا للمقارنة. التطبيق العملي سيستخدم Extension فقط.

---

## Method Overrides & Super

### توسيع Method موجودة

عندما نعرّف Method بالاسم نفسه داخل Extension، يمكننا إضافة سلوك إلى التنفيذ الموجود.

في الشابتر 15، وسعنا `create()` و`write()` داخل Model الأساسية.

الآن يمكن لـModule أخرى توسيعهما أيضًا.

التسلسل المبسط:

```text
تنفيذ Module التوسعة
        ↓ super()
تنفيذ Module الأساسية
        ↓ super()
تنفيذ ORM
```

وقد تشارك Modules أخرى في السلسلة إذا كانت توسع Method نفسها.

---

### ماذا تعني `super()`؟

`super()` تنتقل إلى التنفيذ التالي في تسلسل الوراثة الفعلي.

لا تعني دائمًا استدعاء ملف محدد أوModule ثابتة بالاسم.

تبني Odoo Model النهائية من التعريفات المحملة، ويستخدم Python تسلسل البحث عن Methods، المعروف بـMRO، لتحديد التنفيذ التالي.

لذلك، من المهم أن تحافظ التوسعات المتعاونة على السلسلة.

---

### الحفاظ على المدخلات والنتيجة

عند Override، حافظ على التوقيع المناسب والنتيجة المتوقعة:

| Method | ما نحافظ عليه |
| --- | --- |
| `create()` | دعم الدفعات وإرجاع Records المنشأة |
| `write()` | تمرير القيم وإرجاع نتيجة الكتابة |
| Method تعيد Action | الحفاظ على بنية النتيجة التي يتوقعها المستدعي |

في التطبيق، سننظف نص الملاحظة قبل `super()`، ثم نعيد نتيجة التنفيذ التالي كما هي.

---

### Methods الجديدة لا تحتاج تنفيذًا سابقًا

إذا أضفنا Helper جديدة لتنظيم الكود، فلا يوجد بالضرورة تنفيذ سابق نحتاج إلى استدعائه.

نستخدم `super()` عند توسيع سلوك قائم نريد المحافظة عليه، وليس لمجرد وجود Method داخل Class تستخدم `_inherit`.

---

## Incremental Field Definitions

### تعديل Attribute دون إعادة تعريف كل شيء

يمكن توسيع تعريف Field موجودة بإعادة ذكر اسمها ونوعها، مع Attributes التي نريد تعديلها.

مثلًا:

```python
serial_number = fields.Char(
    help="Serial number used to identify equipment within its company.",
)
```

داخل Extension، يعدل هذا التعريف `help`، مع الاحتفاظ بالـAttributes الأخرى الموروثة التي لم نستبدلها.

في مشروعنا، تبقى إعدادات مثل `copy=False` و`index=True` من التعريف الأساسي.

---

### حافظ على نوع Field ومعناها

تعديل Tooltip يختلف عن تغيير نوع البيانات أوإضافة `required=True`.

التغييرات التي تؤثر على البيانات الحالية أوالعلاقات قد تحتاج إلى مراجعة ومهاجرة للبيانات.

لذلك لا نعيد تعريف Field لمجرد نسخ تعريفها كاملًا؛ نحدد التغيير المطلوب ونراجع أثره.

---

## View Inheritance

### View جديدة تصف التعديل

لتوسيع Form موجودة، ننشئ Record جديدة في `ir.ui.view`، ونربطها بالأصل باستخدام `inherit_id`.

محتوى `arch` هنا يصف التعديلات المطلوبة، بدل إعادة كتابة Form كاملة.

مثلًا:

```xml
<field
    name="inherit_id"
    ref="equipment_training.view_equipment_form"
/>
```

هذا المرجع يشير إلى Form الأجهزة التي أنشأناها سابقًا.

---

### XPath تحدد المكان

نستخدم `xpath` للعثور على العنصر الذي نريد تعديله.

مثال:

```xml
<xpath expr="//form/sheet" position="inside">
    <group string="Training">
        <field name="training_note"/>
    </group>
</xpath>
```

المعنى:

1. ابحث عن `sheet` داخل `form`.
2. أضف المجموعة داخلها.
3. اعرض Field الجديدة ضمن المجموعة.

Field نفسها يجب أن تكون معرّفة في Model قبل التحقق من View.

---

### Positions الشائعة

| Position | الأثر |
| --- | --- |
| `inside` | إضافة المحتوى داخل العنصر المستهدف |
| `before` | إضافة المحتوى قبله |
| `after` | إضافة المحتوى بعده |
| `replace` | استبدال العنصر |
| `attributes` | تعديل Attributes الخاصة به |

لتغيير Label للرقم التسلسلي فقط:

```xml
<xpath
    expr="//field[@name='serial_number']"
    position="attributes"
>
    <attribute name="string">Device Serial Number</attribute>
</xpath>
```

هذا يعدل `string` مع الاحتفاظ ببقية Attributes، مثل شروط `readonly` و`required` الموجودة.

---

### استهداف العنصر الصحيح

تطبق آلية View Inheritance محدد XPath على أول عنصر مطابق تجده.

لذلك، إذا كانت View تحتوي عدة Fields بالاسم نفسه، استخدم مسارًا أكثر تحديدًا.

اعتمد على الأسماء التقنية والبنية المناسبة، وتجنب بناء المحدد على Label مترجمة.

إذا تعذر العثور على العنصر، راجع:

- View المشار إليها في `inherit_id`.
- بنية XML الحالية.
- التوسعات الأخرى التي تؤثر عليها.
- تغير البنية بعد Upgrade.

---

### تفاعل عدة View Extensions

يمكن أن توسع عدة Modules الـView نفسها.

تؤثر شجرة الوراثة و`priority` في ترتيب تطبيق التعديلات.

إذا كان تعديلك يعتمد على عنصر أضافته Module أخرى، فصرح بالاعتماد المناسب واستهدف View الملائمة، بدل الاعتماد على ترتيب عرضي.

---

## Module Dependencies & Interaction

### `depends` تعبر عن اعتماد فعلي

Module التوسعة ستستخدم:

- Model `equipment.asset`.
- View `equipment_training.view_equipment_form`.

لذلك يجب أن تعتمد على:

```python
"depends": [
    "base",
    "equipment_training",
],
```

تضمن Dependencies تحميل المتطلبات قبل Module التي تستخدمها.

وجود Module أخرى مثبتة في Database الحالية لا يغني عن إعلان الاعتماد.

---

### ثلاثة أنواع من الترتيب

| النوع | ما ينظمه |
| --- | --- |
| `depends` | ترتيب المتطلبات بين Modules |
| قائمة `data` | ترتيب تحميل ملفات البيانات داخل Module |
| MRO و`super()` | تسلسل تنفيذ Methods على Model المركبة |

ولا يحدد ترتيب Python Imports ترتيب تطبيق View Extensions.

كل آلية تعالج جانبًا مختلفًا.

---

### استخدام Model أخرى لا يتطلب وراثتها

يمكن لـModule التعامل مع Model أخرى عبر ORM عندما تكون الاعتمادات والصلاحيات مناسبة.

نستخدم الوراثة عندما نريد تعديل تعريف Model أوسلوكها.

أما قراءة Records أوإنشاؤها أوالربط بها، فقد يتم عبر ORM والعلاقات دون توسيع Model نفسها.

---

### Module تكامل مستقلة

إذا احتجنا إلى ربط ميزتين مستقلتين، يمكن إنشاء Module ثالثة تعتمد عليهما.

مثلًا:

```python
"depends": [
    "base",
    "equipment_training",
    "sale",
],
```

وتحتوي هذه Module منطق التكامل بين الأجهزة والمبيعات.

بهذا يكون الاعتماد بين الميزتين واضحًا في مكان مخصص للتكامل.

---

## Example

سننشئ Module باسم:

```text
equipment_training_note
```

ستضيف إلى الأجهزة:

1. Field باسم `training_note`.
2. Tooltip محدثة للرقم التسلسلي.
3. تنظيف المسافات من بداية الملاحظة ونهايتها عند الإنشاء والتعديل.
4. مجموعة جديدة داخل Form الحالية.

التطبيق يفترض اكتمال Module التدريب حتى الشابتر 17.

---

### 1. إنشاء Structure

داخل `custom-addons/`، أنشئ:

```text
custom-addons/
├── equipment_training/
│   └── ...
└── equipment_training_note/
    ├── __init__.py
    ├── __manifest__.py
    ├── models/
    │   ├── __init__.py
    │   └── equipment_asset.py
    └── views/
        └── equipment_asset_views.xml
```

هذه Module جديدة بجانب الأصلية.

---

### 2. كتابة Manifest

داخل `equipment_training_note/__manifest__.py`:

```python
{
    "name": "Equipment Training Note",
    "version": "20.0.1.0.0",
    "license": "LGPL-3",
    "depends": [
        "base",
        "equipment_training",
    ],
    "data": [
        "views/equipment_asset_views.xml",
    ],
    "application": False,
    "installable": True,
}
```

اعتمدنا على `equipment_training` لأننا نوسع موارد تعرفها.

---

### 3. ربط Python Imports

داخل `equipment_training_note/__init__.py`:

```python
from . import models
```

وداخل `equipment_training_note/models/__init__.py`:

```python
from . import equipment_asset
```

وجود ملف Python وحده لا يكفي؛ يجب استيراده حتى يُحمّل تعريف التوسعة.

---

### 4. توسيع Model

داخل `equipment_training_note/models/equipment_asset.py`:

```python
from odoo import api, fields, models


class EquipmentAsset(models.Model):
    _inherit = "equipment.asset"

    training_note = fields.Text(
        string="Training Note",
        copy=False,
    )

    serial_number = fields.Char(
        help=(
            "Serial number used to identify equipment "
            "within its company."
        ),
    )

    @api.model
    def _prepare_training_note_values(self, values):
        prepared_values = dict(values)

        if isinstance(prepared_values.get("training_note"), str):
            prepared_values["training_note"] = (
                prepared_values["training_note"].strip()
            )

        return prepared_values

    @api.model_create_multi
    def create(self, vals_list):
        prepared_vals_list = [
            self._prepare_training_note_values(values)
            for values in vals_list
        ]

        return super().create(prepared_vals_list)

    def write(self, values):
        prepared_values = self._prepare_training_note_values(values)
        return super().write(prepared_values)
```

#### ماذا تفعل التوسعة؟

- `_inherit` تستهدف Model الأجهزة نفسها.
- `training_note` تضيف Field اختيارية.
- تعريف `serial_number` يعدل Tooltip فقط.
- Helper تنسخ Dictionary قبل تعديلها.
- `strip()` تزيل المسافات والأسطر الفارغة من الطرفين.
- `create()` و`write()` تمرران التنفيذ إلى السلسلة باستخدام `super()`.

لا نغير المسافات الموجودة داخل نص الملاحظة.

كما أن Helper لا تحتاج إلى Records معينة في `self`، ولذلك استخدمنا `@api.model`.

---

### 5. توسيع Form

داخل `equipment_training_note/views/equipment_asset_views.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <record
        id="view_equipment_form_training_note"
        model="ir.ui.view"
    >
        <field name="name">
            equipment.asset.form.training.note
        </field>

        <field name="model">equipment.asset</field>

        <field
            name="inherit_id"
            ref="equipment_training.view_equipment_form"
        />

        <field name="arch" type="xml">
            <xpath expr="//form/sheet" position="inside">
                <group
                    name="training_note_group"
                    string="Training"
                >
                    <field name="training_note"/>
                </group>
            </xpath>

            <xpath
                expr="//field[@name='serial_number']"
                position="attributes"
            >
                <attribute name="string">
                    Device Serial Number
                </attribute>
            </xpath>
        </field>
    </record>
</odoo>
```

الـExternal ID الخاصة بالتوسعة:

```text
equipment_training_note.view_equipment_form_training_note
```

أما `inherit_id`، فتشير إلى View الأصلية داخل Module أخرى.

لم نكرر External ID الخاصة بالأصل، ولم ننشئ Form مستقلة نحتاج إلى ربطها بـAction جديدة.

---

### 6. فهم الصلاحيات الحالية

لم ننشئ Model جديدة، ولذلك نستخدم تعريفات الوصول الموجودة على:

```text
equipment.asset
```

لا نحتاج إلى ملف `ir.access.csv` جديد لمجرد إضافة هذه Field.

في التصميم الحالي:

- Equipment User يستطيع قراءتها ضمن الأجهزة المسموحة.
- Equipment Manager يستطيع تعديلها وفق صلاحياته.

Field ليست ملاحظة سرية للمدير. إذا احتجت إلى تقييد الوصول إليها، فصمم Field Security المناسبة كما شرحنا في [[11 Security & Multi-Company]].

---

### 7. اكتشاف Module وتثبيتها

بعد حفظ الملفات:

1. تأكد أن `custom-addons/` موجودة في `addons_path`.
2. أعد تشغيل Server التدريب.
3. افتح Database `odoo20_training`.
4. فعّل Developer Mode.
5. افتح **Apps**.
6. نفّذ **Update Apps List**.
7. احذف Filter **Apps**.
8. ابحث عن `Equipment Training Note`.
9. اضغط **Activate**.

هذه Module جديدة، لذلك نثبتها أول مرة.

أما عند تعديل ملفاتها لاحقًا، فنستخدم Upgrade، مع إعادة تشغيل Server عند تعديل Python.

راجع [[03 Modules & Addons]] لتفاصيل Discovery وInstallation وUpgrade.

---

### 8. تجربة الواجهة

باستخدام Equipment Manager:

1. افتح **Equipment → Assets**.
2. افتح جهاز تدريب موجودًا.
3. تحقق من ظهور مجموعة **Training**.
4. أدخل ملاحظة واحفظها.
5. تحقق من ظهور Label **Device Serial Number**.
6. تحقق من بقاء أزرار الحالات وشروط الواجهة السابقة.

الـAction نفسها ما زالت تفتح Form الأصلية، وتطبق Odoo عليها View Extension.

---

### 9. فحص التوسعة عبر ORM

افتح Shell جديدة بعد تثبيت Module.

جهز حساب المدير التدريبي:

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
```

ثم أنشئ جهازًا بملاحظة تحتوي مسافات في الطرفين:

```python
asset = equipment_model.create(
    {
        "name": "Inheritance Training Laptop",
        "serial_number": "INH-018-SHELL-A",
        "training_note": "  Checked during training.  ",
    }
)

asset.training_note
```

المتوقع:

```python
"Checked during training."
```

استخدم رقمًا جديدًا عند إعادة المثال حتى لا تصطدم بقيد التفرد من الشابتر 17.

---

### 10. فحص الكتابة وعدم تعديل المدخلات الأصلية

نفّذ:

```python
values = {
    "training_note": "  Checked again.  ",
}

asset.write(values)

asset.training_note
```

المتوقع:

```python
"Checked again."
```

ثم افحص Dictionary التي أرسلتها:

```python
values["training_note"]
```

المتوقع:

```python
"  Checked again.  "
```

التوسعة عدلت نسخة من القيم، وحافظت على Dictionary الأصلية.

هذا مهم عندما يستخدم المستدعي البيانات نفسها في مواضع أخرى.

---

### 11. التحقق من استمرار قواعد الأصل

حاول إنشاء جهاز بحالة بداية غير مسموحة:

```python
with env.cr.savepoint():
    equipment_model.create(
        {
            "name": "Invalid Inherited Creation",
            "serial_number": "INH-018-INVALID",
            "state": "in_use",
            "training_note": "  Example note.  ",
        }
    )
```

المتوقع رفض الإنشاء بقاعدة الشابتر 15.

ثم حوّل جهاز التدريب الصحيح إلى الاستخدام:

```python
asset.action_start_use()
```

وحاول تقاعده قبل إعادته:

```python
with env.cr.savepoint():
    asset.action_retire()
```

المتوقع رفض الانتقال.

في هذا المسار:

```text
action_retire() من Module الأساسية
        ↓
write() من Module الملاحظة
        ↓ super()
write() من Module الأساسية
        ↓
فحص الانتقال ورفضه
```

Method الأصلية تستخدم Model المركبة نفسها، ولذلك تمر الكتابة بتوسعة `write()` الجديدة أيضًا.

كما تبقى SQL Constraint الخاصة بالتفرد مطبقة في Database.

---

### 12. تشخيص المشاكل الشائعة

| المشكلة | ما تراجعه |
| --- | --- |
| Module لا تظهر | `addons_path` وManifest وتحديث قائمة Modules وإزالة Filter Apps |
| Field غير موجودة | Python Imports وتحميل Module التوسعة وتثبيتها |
| تعذر العثور على External ID | اسم Module واسم View و`depends` |
| XPath لا تجد العنصر | View الأب وبنية XML الحالية |
| Field موجودة لكن لا تظهر في Form | تحميل ملف View وصحة `inherit_id` والـView التي تفتحها Action |
| قواعد الأصل لم تعد تعمل | Overrides التي لم تمرر التنفيذ عبر `super()` |
| ظهرت Model جديدة غير مقصودة | قيمة `_name` وعلاقتها بـ`_inherit` |
| تعديل Attribute أثر على بيانات قديمة | معنى التغيير ومتطلبات Migration |

---

## Important Notes

- `_inherit` دون اسم Model جديد توسع Model الموجودة؛ `_name` المختلفة تنشئ Model أخرى.
- Python Class الجديدة في Extension لا تعني وجود مجموعة Records مستقلة.
- `_inherits` تفوض Fields عبر علاقات، ولا تورث Methods الخاصة بالـModel المفوضة تلقائيًا.
- `super()` تكمل تسلسل التنفيذ الفعلي، وقد تمر عبر أكثر من Module.
- لا تعتمد على ترتيب عرضي بين Modules مستقلة إذا كان كودك يحتاج إلى مورد من إحداها.
- XPath قد تطابق عنصرًا واحدًا غير المقصود إذا كان المحدد عامًا؛ راجع البنية الفعلية.
- Module منفصلة لا تجعل Field الجديدة خاصة بمستخدميها تلقائيًا؛ الصلاحيات تحتاج إلى تعريف واضح.
- View Inheritance توسع طريقة العرض، بينما Model Inheritance توسع البيانات والسلوك؛ كل واحدة تحتاج إلى تعريفها الخاص.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[19 Wizards]]

---

## Sources

- [Odoo 20 Documentation — Inheritance and Extension](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#inheritance-and-extension)
- [Odoo 20 Tutorial — Inheritance](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/12_inheritance.html)
- [Odoo 20 Documentation — View Records](https://www.odoo.com/documentation/20.0/developer/reference/user_interface/view_records.html)
- [Odoo 20 Documentation — Module Manifests](https://www.odoo.com/documentation/20.0/developer/reference/backend/module.html)
- [Odoo 20 Source — Model Composition and Inheritance](https://github.com/odoo/odoo/blob/20.0/odoo/orm/model_classes.py)
- [Odoo 20 Source — ORM Methods and Access Checks](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)
- [Odoo 20 Source — XML Inheritance](https://github.com/odoo/odoo/blob/20.0/odoo/tools/template_inheritance.py)