---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 15 Business Logic

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Business Rules & Security|Business Rules & Security]]
- [[#Model Methods|Model Methods]]
- [[#State Transitions|State Transitions]]
- [[#Extending Create & Write|Extending Create & Write]]
- [[#Errors & Transactions|Errors & Transactions]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

Business Logic هي القواعد التي تحدد متى تكون العملية صحيحة، وما الذي يجب أن يحدث عند تنفيذها.

مثلًا، في نظام إدارة الأجهزة:

- لا نُخرج جهازًا من الخدمة وهو مستخدم قبل إعادته.
- لا نعيد جهازًا متقاعدًا إلى الاستخدام.
- لا نسمح بتغيير الرقم التسلسلي بعد التقاعد.

في [[14 Widgets & View Attributes]]، استخدمنا شروطًا لإخفاء Button أو جعل Field للقراءة.

هذه الشروط تساعد المستخدم داخل الواجهة، لكن قواعد العمل المهمة يجب أن تُطبق على Server.

قد يصل التعديل من Button أوImport أوIntegration أوPython Code. إذا كانت هذه المسارات تستخدم ORM، فيجب أن تمر بالقواعد المناسبة أيضًا.

سنكمل في هذا الشابتر Model الأجهزة `equipment.asset`، مع الاحتفاظ بالحالات الحالية:

```text
available
in_use
retired
```

---

## Business Rules & Security

### سؤالان مختلفان

قبل تنفيذ عملية، نحتاج إلى الإجابة عن سؤالين:

| السؤال | المسؤولية |
| --- | --- |
| هل المستخدم مخوّل بالتعامل مع هذه Record؟ | Security |
| هل التغيير صحيح وفق قواعد العمل؟ | Business Logic |

مثلًا، Equipment Manager يملك صلاحية كتابة الأجهزة ضمن نطاق الشركات المسموح به.

لكن امتلاكه هذه الصلاحية لا يعني أن كل انتقال بين الحالات مسموح.

قد تسمح له Security بكتابة الجهاز، ثم ترفض Business Rule تحويله من `in_use` إلى `retired`.

---

### Button ليست القاعدة نفسها

يمكن إخفاء زر Retire عندما يكون الجهاز مستخدمًا.

لكن هذا لا يمنع تنفيذ:

```python
asset.write({"state": "retired"})
```

من مسار آخر.

إذا كان منع تقاعد الجهاز المستخدم قاعدة عمل، فيجب أن تُفحص عند تغيير الحالة على Server.

الزر يصبح وسيلة لطلب العملية، بينما مسار الكتابة يفرض القاعدة.

---

### اختيار مكان التحقق

| المطلوب | المكان المناسب |
| --- | --- |
| تنظيم ما يظهر للمستخدم | View Attributes |
| تحديد صلاحيات المستخدم على البيانات | تعريفات الوصول وفحوص الصلاحيات |
| تنفيذ عملية عمل مسماة | Model Method |
| فرض شرط على الإنشاء | `create()` عند الحاجة |
| مقارنة القيمة القديمة بالتغيير المطلوب | `write()` عند الحاجة |
| التحقق من صحة قيم Fields | Constraints المناسبة |

لا نضع جميع القواعد في `write()` تلقائيًا.

نختار المكان بحسب طبيعة الشرط، والمسارات التي يجب أن تغطيها القاعدة.

سنشرح Computed Fields وOnchange في الشابتر التالي، ثم Constraints في [[17 Constraints]].

---

## Model Methods

### Method تمثل عملية واضحة

بدل توزيع تنفيذ العملية داخل الواجهة، نضعها في Model Method ذات اسم يوضح غرضها.

مثلًا:

```python
def action_retire(self):
    return self.write({"state": "retired"})
```

هذه Method تطلب تحويل الأجهزة الموجودة في `self` إلى `retired`.

يمكن استدعاؤها من Button:

```xml
<button
    name="action_retire"
    type="object"
    string="Retire"
/>
```

ويمكن استدعاؤها من Python أيضًا:

```python
asset.action_retire()
```

وجود Method مسماة يجعل العملية واضحة وقابلة لإعادة الاستخدام.

لكن إذا اكتفينا بالتحقق داخلها، فستظل الكتابة المباشرة إلى `state` مسارًا منفصلًا؛ لذلك سنضع قواعد الانتقال المشتركة في مسار `write()`.

---

### Public وPrivate Methods

الاسم الذي لا يبدأ بشرطة سفلية، مثل:

```python
action_retire
```

يمكن أن يكون نقطة استدعاء عامة من الواجهة أوRPC.

أما اسم مثل:

```python
_check_state_transition
```

فيستخدم لتنظيم منطق داخلي، ولا يُستدعى مباشرةً من Button أوRPC.

هذا التقسيم يساعد على تنظيم الكود، لكنه لا يعوض فحوص الصلاحيات داخل مسار التنفيذ.

---

### التعامل مع Recordsets

`self` قد تحتوي Record واحدة أوعدة Records.

إذا كانت العملية تنطبق على المجموعة، يمكن دعمها مباشرةً:

```python
self.write({"state": "in_use"})
```

أما عندما تعتمد القاعدة على الحالة الحالية لكل جهاز، نفحص Records أولًا:

```python
for asset in self:
    ...
```

ثم ننفذ الكتابة بعد نجاح الفحص للجميع.

لا نستخدم `ensure_one()` لمجرد أن الزر موجود في Form؛ نستخدمها عندما تحتاج العملية فعلًا إلى Record واحدة.

---

## State Transitions

### Field الحالة لا تحدد الانتقالات وحدها

تعريف Selection Field يحدد القيم التي تقبلها Field.

لكنه لا يحدد تلقائيًا الانتقالات المسموحة بينها.

مثلًا، وجود:

```text
available
in_use
retired
```

لا يمنع وحده كتابة `available` فوق جهاز حالته `retired`.

نحتاج إلى تعريف قاعدة الانتقال.

---

### قواعد مثال التدريب

سنطبق الانتقالات التالية:

```text
available ──→ in_use
    ↑            │
    └────────────┘

available ──→ retired
```

أي:

| الحالة الحالية | الحالة المطلوبة | القرار |
| --- | --- | --- |
| `available` | `in_use` | مسموح |
| `available` | `retired` | مسموح |
| `in_use` | `available` | مسموح |
| `in_use` | `retired` | مرفوض |
| `retired` | `available` | مرفوض |
| `retired` | `in_use` | مرفوض |

الجهاز المستخدم يجب أن يعود إلى `available` قبل تقاعده.

وسنسمح بإرسال الحالة الحالية نفسها؛ إعادة كتابة القيمة نفسها لا تُعد انتقالًا جديدًا في هذا المثال.

هذه قواعد اخترناها لمشروع التدريب، وليست قواعد تفرضها Odoo على كل Model تحتوي Field اسمها `state`.

---

### تمثيل الانتقالات في Python

يمكن تمثيل الحالات التالية المسموحة باستخدام Dictionary:

```python
allowed_transitions = {
    "available": {"in_use", "retired"},
    "in_use": {"available"},
    "retired": set(),
}
```

المفتاح هو الحالة الحالية، والقيمة هي مجموعة الحالات التي يمكن الانتقال إليها.

مثلًا:

```python
allowed_transitions["in_use"]
```

تعطي:

```python
{"available"}
```

سنستخدم هذا التعريف داخل Method داخلية لفحص طلب تغيير الحالة.

---

## Extending Create & Write

### معنى Override

عندما نعرّف `create()` أو`write()` داخل Model، فإننا نوسع سلوك العملية الموجودة في ORM.

نضيف التحقق المطلوب، ثم نمرر التنفيذ إلى الجزء التالي باستخدام `super()`.

لا نحتاج إلى إعادة كتابة عملية الحفظ أو التعامل مع Database يدويًا.

تفاصيل الوراثة والتفاعل بين Modules تأتي في [[18 Inheritance & Module Interaction]].

---

### توسيع `create()`

تستطيع Odoo إنشاء عدة Records في استدعاء واحد.

لذلك نستخدم:

```python
@api.model_create_multi
def create(self, vals_list):
    ...
    return super().create(vals_list)
```

داخل هذه Method، تكون `vals_list` قائمة Dictionaries:

```python
[
    {"name": "Training Laptop"},
    {"name": "Training Monitor"},
]
```

كل Dictionary تصف قيم Record واحدة.

`super().create()` تعيد Recordset الخاصة بالـRecords المنشأة، ولذلك نحافظ على هذه النتيجة عند إرجاعها.

---

### القيم الافتراضية جزء من الإنشاء

قد لا تحتوي القيم الواردة على `state`:

```python
{"name": "Training Laptop"}
```

لكن الحالة قد تصل من:

- Default المعرفة على Field.
- Context مثل `default_state`.
- إعدادات Defaults الأخرى التي يعالجها ORM.

لذلك، إذا كانت قاعدة الإنشاء تعتمد على الحالة، فلا يكفي فحص Dictionaries التي تحتوي المفتاح `state` فقط.

سنستخدم:

```python
self.default_get(["state"])
```

لمعرفة القيمة الافتراضية عندما لا تُرسل الحالة صراحةً.

راجع [[09 Environment & Context]] لمراجعة طريقة عمل Context Defaults.

---

### توسيع `write()`

تطبق `write()` القيم نفسها على Records الموجودة في `self`.

لكن الحالة الحالية قد تختلف من Record إلى أخرى، ولذلك نفحص كل واحدة قبل تنفيذ التغيير.

التسلسل في مثالنا:

```text
فحص صلاحية الكتابة
        ↓
فحص صلاحية التعامل مع الحالة
        ↓
فحص الانتقالات المطلوبة
        ↓
فحص حماية الرقم التسلسلي
        ↓
super().write()
```

بهذا لا نبدأ تعديل الأجهزة واحدًا بعد الآخر أثناء فحص القاعدة نفسها.

---

### أهمية `super()`

هذا الاستدعاء:

```python
super().write(values)
```

يكمل سلسلة التنفيذ، بما فيها سلوك ORM والتوسعات الأخرى الموجودة على Model.

أما هذا:

```python
self.write(values)
```

داخل `write()` نفسها، فيستدعي Method مرة أخرى.

كذلك، تعيين قيمة Field مخزنة مثل:

```python
self.state = "retired"
```

داخل Override الخاصة بـ`write()` قد يعيد الدخول إلى `write()` ويسبب Recursion.

لذلك نمرر القيم المطلوبة إلى `super().write()` بدل إعادة استدعاء المسار نفسه.

---

## Errors & Transactions

### اختيار نوع الخطأ

| النوع | الاستخدام |
| --- | --- |
| `AccessError` | المستخدم غير مخول بالعملية |
| `UserError` | العملية مرفوضة وفق قاعدة عمل |
| `ValidationError` | فشل تحقق من صحة البيانات، وسنستخدمها مع Constraints |

مثلًا:

- مستخدم غير مخول يطلب تغيير الحالة: `AccessError`.
- انتقال من `in_use` إلى `retired`: `UserError`.

اجعل الرسالة توضح سبب الرفض، واكتب النصوص القابلة للترجمة باستخدام `_()`.

---

### فشل العملية والمعاملة

في طلب RPC العادي، إذا وصل Exception إلى Framework دون معالجته، تتراجع تغييرات المعاملة المرتبطة بالطلب.

لكن التقاط الخطأ داخل الكود ثم متابعة التنفيذ يغير هذا المسار؛ لا تفترض أن كل خطأ تم التقاطه تسبب تلقائيًا في Rollback.

في Business Methods المعتادة:

- دع الأخطاء غير المعالجة تصل إلى Framework.
- لا تضف `commit()` يدويًا لتثبيت جزء من العملية.
- عند الحاجة إلى التقاط خطأ مع متابعة العمل، استخدم تصميمًا واضحًا للمعاملة وSavepoints.

راجع [[05 ORM]] لمراجعة أساسيات Transactions.

---

## Example

سنوسع `equipment.asset` بالقواعد التالية:

1. يبدأ الجهاز الجديد في حالة `available`.
2. تغيير الحالة يتطلب Equipment Manager.
3. كل انتقال يجب أن يطابق جدول الانتقالات.
4. لا يمكن تغيير الرقم التسلسلي لجهاز متقاعد.
5. تدعم العمليات عدة أجهزة في الاستدعاء نفسه.

هذه القواعد ستعمل في المسارات التي تستخدم `create()` و`write()` عبر ORM، بما فيها Methods الأزرار.

---

### 1. تحديث Model الأجهزة

داخل الملف:

```text
custom-addons/equipment_training/models/equipment_asset.py
```

استخدم التعريف التالي كأساس يجمع Fields السابقة مع المنطق الجديد.

إذا أضفت Fields أوMethods أخرى إلى Model، احتفظ بها، وادمج هذه التغييرات داخل Class الموجودة.

```python
from odoo import _, api, fields, models
from odoo.exceptions import AccessError, UserError


class EquipmentAsset(models.Model):
    _name = "equipment.asset"
    _description = "Equipment Asset"
    _order = "name, id"
    _check_company_auto = True

    name = fields.Char(required=True)

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
        copy=False,
    )

    active = fields.Boolean(default=True)

    company_id = fields.Many2one(
        "res.company",
        required=True,
        default=lambda self: self.env.company,
    )

    @api.model_create_multi
    def create(self, vals_list):
        self.browse().check_access("create")

        default_state = self.default_get(["state"]).get("state")

        for values in vals_list:
            initial_state = values.get("state", default_state)

            if initial_state != "available":
                raise UserError(
                    _("New equipment must start in the available state.")
                )

        return super().create(vals_list)

    def write(self, values):
        if not self:
            return super().write(values)

        self.check_access("write")

        if "state" in values:
            if not self.env.user.has_group(
                "equipment_training.group_equipment_manager"
            ):
                raise AccessError(
                    _("Only equipment managers can set equipment states.")
                )

            self._check_state_transition(values["state"])

        if "serial_number" in values:
            new_serial_number = values["serial_number"] or False

            for asset in self:
                if (
                    asset.state == "retired"
                    and new_serial_number != asset.serial_number
                ):
                    raise UserError(
                        _(
                            "The serial number of retired equipment "
                            "cannot be changed."
                        )
                    )

        return super().write(values)

    def _check_state_transition(self, new_state):
        allowed_transitions = {
            "available": {"in_use", "retired"},
            "in_use": {"available"},
            "retired": set(),
        }

        for asset in self:
            if new_state == asset.state:
                continue

            if new_state not in allowed_transitions[asset.state]:
                raise UserError(
                    _(
                        "Transition from %(current)s to %(target)s "
                        "is not allowed for %(asset)s.",
                        current=asset.state,
                        target=new_state,
                        asset=asset.display_name,
                    )
                )

    def action_start_use(self):
        return self.write({"state": "in_use"})

    def action_return(self):
        return self.write({"state": "available"})

    def action_retire(self):
        return self.write({"state": "retired"})
```

استبدل Method `action_retire` السابقة بالتعريف الجديد، بدل إبقاء تعريفين بالاسم نفسه.

---

### 2. فهم توزيع المسؤوليات

| الجزء | مسؤوليته |
| --- | --- |
| `create()` | التحقق من حالة البداية |
| `write()` | تجميع فحوص التعديل المشتركة |
| `_check_state_transition()` | التحقق من الانتقال لكل جهاز |
| `action_start_use()` | طلب التحويل إلى `in_use` |
| `action_return()` | طلب التحويل إلى `available` |
| `action_retire()` | طلب التحويل إلى `retired` |

نقلنا فحص Group من `action_retire` إلى `write()`، حتى يمر به طلب تغيير الحالة المباشر أيضًا.

أما `check_access()`، فترفع خطأ إذا لم تكن العملية مسموحة على Model أوRecords المستهدفة.

استخدمنا فحص الكتابة قبل قراءة الحالات الحالية لإعداد رسائل قواعد العمل، مع استمرار فحوص ORM عند تنفيذ `super()`.

---

### 3. لماذا أضفنا `copy=False` إلى الحالة؟

نسخ جهاز مستخدم أومتقاعد يجب أن ينتج جهازًا جديدًا يبدأ وفق قاعدة الإنشاء.

لذلك أضفنا إلى `state`:

```python
copy=False
```

حتى لا تُنسخ حالة الجهاز الأصلي تلقائيًا، وتُستخدم Defaults المناسبة للنسخة الجديدة.

هذا لا يلغي تحقق `create()`؛ إذا طُلب إنشاء النسخة بحالة غير مسموحة صراحةً أوعبر Context، يظل الطلب مرفوضًا.

كذلك، شرط طلب الرقم التسلسلي عند `available` من الشابتر 14 ما زال شرط واجهة فقط. القاعدة الجديدة على Server هنا هي منع تغييره بعد التقاعد.

---

### 4. تحديث Form Buttons

داخل:

```text
views/equipment_views.xml
```

عدّل Header داخل Form الحالية ليصبح:

```xml
<header>
    <button
        name="action_start_use"
        type="object"
        string="Start Use"
        class="btn-primary"
        groups="equipment_training.group_equipment_manager"
        invisible="state != 'available'"
    />

    <button
        name="action_return"
        type="object"
        string="Return"
        groups="equipment_training.group_equipment_manager"
        invisible="state != 'in_use'"
    />

    <button
        name="action_retire"
        type="object"
        string="Retire"
        groups="equipment_training.group_equipment_manager"
        invisible="state != 'available'"
    />

    <field
        name="state"
        widget="statusbar"
        statusbar_visible="available,in_use,retired"
        readonly="True"
    />
</header>
```

احتفظ ببقية Form وList وSearch كما هي.

لاحظ أن Retire أصبحت متاحة في الواجهة عندما يكون الجهاز `available` فقط، بما يتوافق مع القاعدة الجديدة.

شروط XML توجه المستخدم، وفحص `write()` يحمي الانتقال حتى إذا لم يصل الطلب من هذه الأزرار.

---

### 5. تطبيق التغيير

لأننا عدّلنا Python وXML:

1. احفظ الملفات.
2. أوقف Server التدريب.
3. شغّله مجددًا.
4. افتح Database `odoo20_training`.
5. افتح **Apps** وابحث عن `Equipment Training`.
6. نفّذ **Upgrade**.
7. أعد فتح واجهة الأجهزة.

لم نضف ملف Python جديدًا، لذلك لا نحتاج إلى Import جديدة في `models/__init__.py`.

---

### 6. تجربة الواجهة

باستخدام Equipment Manager:

1. أنشئ جهاز تدريب.
2. أدخل الاسم والرقم التسلسلي، ثم احفظ.
3. اضغط **Start Use**.
4. تحقق من تغير الحالة إلى **In Use**.
5. تحقق من ظهور **Return** واختفاء **Retire**.
6. اضغط **Return**.
7. تحقق من عودة الحالة إلى **Available**.
8. اضغط **Retire**.
9. تحقق من ظهور **Retired** واختفاء أزرار الانتقال.

الرقم التسلسلي يصبح للقراءة وفق View، وأصبح تغييره محميًا داخل `write()` أيضًا.

---

### 7. تجهيز فحوص Shell

لفحص الكتابة المباشرة وعمليات المجموعات، افتح Odoo Shell على Database التدريب.

سنستخدم حساب المدير التدريبي من الشابتر 11، الذي يحمل Login:

```text
equipment.training.manager
```

إذا جهزت حسابًا تدريبيًا آخر، استخدم Login الخاصة به بدل القيمة التجريبية.

نفّذ:

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

نستخدم `with_user()` حتى نفحص القواعد بحساب المدير، مع اختيار شركة مسموحة له.

فحص وجود المستخدم مهم؛ لا نمرر نتيجة بحث فارغة ونتوقع أنها تمثل حساب الاختبار.

---

### 8. إنشاء عدة أجهزة

نفّذ:

```python
assets = equipment_model.create(
    [
        {
            "name": "Logic Training Laptop",
            "serial_number": "LOGIC-015-A",
        },
        {
            "name": "Logic Training Monitor",
            "serial_number": "LOGIC-015-B",
        },
    ]
)

first_asset = assets[0]
second_asset = assets[1]

assets.mapped("state")
```

المتوقع:

```python
["available", "available"]
```

هذا الاستدعاء يوضح أن Override الخاصة بـ`create()` تدعم عدة Dictionaries وتعيد عدة Records.

---

### 9. فحص رفض التقاعد أثناء الاستخدام

حوّل الجهاز الأول إلى الاستخدام:

```python
first_asset.action_start_use()

assets.mapped("state")
```

المتوقع:

```python
["in_use", "available"]
```

الآن حاول تقاعد الجهاز الأول باستخدام كتابة مباشرة:

```python
with env.cr.savepoint():
    first_asset.write({"state": "retired"})
```

المتوقع ظهور `UserError`؛ الانتقال مرفوض حتى دون استخدام Button.

نفّذ فحوص الأخطاء التالية كلًا على حدة.

نستخدم Savepoint لعزل العملية التي نتوقع فشلها، حتى نستطيع متابعة الفحص داخل معاملة Shell نفسها.

---

### 10. فحص المجموعة قبل الكتابة

حاول تقاعد الجهازين معًا:

```python
with env.cr.savepoint():
    assets.action_retire()
```

المتوقع رفض الطلب؛ لأن الجهاز الأول ما زال `in_use`.

ثم افحص:

```python
assets.mapped("state")
```

المتوقع:

```python
["in_use", "available"]
```

الجهاز الثاني لم يتقاعد، رغم أن انتقاله منفردًا كان مسموحًا.

فحوصنا راجعت المجموعة قبل استدعاء `super().write()`.

---

### 11. تنفيذ انتقال مسموح للمجموعة

أعد الجهاز الأول، ثم تقاعد الجهازين معًا:

```python
first_asset.action_return()

assets.action_retire()

assets.mapped("state")
```

المتوقع:

```python
["retired", "retired"]
```

هذا يوضح دعم Method للمجموعة عندما تكون انتقالات جميع Records مسموحة.

---

### 12. فحص حماية الجهاز المتقاعد

حاول إعادة الجهاز إلى `available`:

```python
with env.cr.savepoint():
    first_asset.write({"state": "available"})
```

المتوقع رفض الانتقال.

ثم حاول تغيير الرقم التسلسلي:

```python
with env.cr.savepoint():
    first_asset.write({"serial_number": "LOGIC-CHANGED"})
```

المتوقع رفض التعديل.

هذه الحماية موجودة على Server، بالإضافة إلى `readonly` داخل Form.

---

### 13. فحص حالة البداية

حاول إنشاء جهاز متقاعد مباشرةً:

```python
with env.cr.savepoint():
    equipment_model.create(
        {
            "name": "Invalid Initial State",
            "state": "retired",
        }
    )
```

المتوقع رفض الإنشاء.

ثم جرّب إرسال الحالة عبر Context بدل Dictionary:

```python
with env.cr.savepoint():
    equipment_model.with_context(
        default_state="retired"
    ).create(
        {
            "name": "Invalid Context Default",
        }
    )
```

المتوقع رفض الإنشاء أيضًا.

أصبحت حالة البداية هنا قاعدة عمل، ولذلك لا تستطيع Context Default تحويلها إلى حالة أخرى مسموحة تلقائيًا.

أمثلة Context السابقة كانت توضح توفير القيم الافتراضية؛ القاعدة الجديدة تحدد الآن القيم التي يقبلها الإنشاء.

---

### 14. مراجعة نطاق الحماية

| الحالة | النتيجة المتوقعة |
| --- | --- |
| إنشاء جهاز جديد بـ`available` | مسموح لمن يملك صلاحية الإنشاء |
| إنشاء جهاز جديد بـ`retired` | مرفوض |
| انتقال `available` إلى `in_use` | مسموح للمدير المخول بالكتابة |
| انتقال `in_use` إلى `retired` | مرفوض |
| انتقال `retired` إلى `available` | مرفوض |
| تغيير الرقم التسلسلي بعد التقاعد | مرفوض |
| تعديل الوصف بعد التقاعد | يخضع للصلاحيات؛ هذه القاعدة لا تمنعه |
| الكتابة على جهاز خارج نطاق الشركات | تخضع لقيود الوصول من الشابتر 11 |

فحوص Shell السابقة يدوية للتعلم، وتبقى بياناتها ضمن معاملة Shell. سننقل هذا النوع من السيناريوهات إلى اختبارات منظمة في [[26 Testing]].

---

## Important Notes

- فحص الحالة داخل Button Method وحدها لا يغطي الكتابة المباشرة إلى Field.
- إذا كانت القاعدة تقيد الحالة الابتدائية، فافحص الإنشاء أيضًا، مع القيم الافتراضية.
- `write()` تستقبل التعديلات المطلوبة فقط؛ غياب المفتاح لا يعني طلب مسح Field.
- فحص عدة Records قبل الكتابة يمنع تنفيذ جزء من عمليتنا أثناء اكتشاف انتقال غير مسموح.
- صلاحية المدير لا تلغي قواعد العمل، و`sudo()` لا يلغي تلقائيًا فحوص Python المخصصة.
- لا تستخدم Context Flag يمكن للعميل إرسالها لتجاوز التحقق الحساس.
- عند إضافة آثار أخرى للانتقال، مثل إنشاء حركة أوإرسال إشعار، راعِ تكرار الطلب؛ السماح بإعادة كتابة الحالة نفسها لا يبرر تكرار الأثر.
- القواعد هنا تنظم حالات الجهاز ورقمه التسلسلي؛ ليست قفلًا شاملًا لكل Fields بعد التقاعد.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[16 Computed Fields, Related Fields & Onchange]]

---

## Sources

- [Odoo 20 Documentation — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Tutorial — Ready For Some Action?](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/09_actions.html)
- [Odoo 20 Documentation — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)
- [Odoo 20 Coding Guidelines](https://www.odoo.com/documentation/20.0/contributing/development/coding_guidelines.html)
- [Odoo 20 Source — ORM Models](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)
- [Odoo 20 Source — API Decorators](https://github.com/odoo/odoo/blob/20.0/odoo/orm/decorators.py)