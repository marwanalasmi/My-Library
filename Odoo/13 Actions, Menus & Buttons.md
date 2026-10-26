---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 13 Actions, Menus & Buttons

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Window Actions|Window Actions]]
- [[#Menus|Menus]]
- [[#Buttons|Buttons]]
- [[#Returning Actions from Python|Returning Actions from Python]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في [[12 Views & XML]]، جهزنا List وForm وSearch للـModel `equipment.asset`.

لكن تعريف Views وحده لا يوفر للمستخدم قائمة يفتح منها الأجهزة.

نحتاج إلى ربط ثلاثة أجزاء:

| الجزء | مسؤوليته |
| --- | --- |
| View | تحديد طريقة عرض البيانات |
| Action | تحديد ما سيُفتح أو يُنفذ |
| Menu | توفير نقطة وصول داخل التنقل |

مثلًا، عندما يختار المستخدم **Assets**:

```text
Equipment → Assets
        ↓
Window Action
        ↓
عرض equipment.asset باستخدام List View
        ↓
فتح جهاز باستخدام Form View
```

أما **Button** داخل View، فقد تستدعي Python Method أو تشغّل Action موجودة.

في هذا الشابتر، سنربط واجهات الأجهزة بقائمة، ثم نضيف زرًا يحوّل حالة الجهاز إلى `retired`.

---

## Window Actions

### ما هي Window Action؟

Window Action تحدد كيفية فتح بيانات Model داخل الواجهة.

نعرّفها كـRecord من Model:

```text
ir.actions.act_window
```

مثال أساسي:

```xml
<record id="action_equipment" model="ir.actions.act_window">
    <field name="name">Equipment</field>
    <field name="res_model">equipment.asset</field>
    <field name="view_mode">list,form</field>
</record>
```

هذا التعريف يطلب فتح بيانات `equipment.asset`، مع إتاحة List وForm.

الـExternal ID الكاملة هي:

```text
equipment_training.action_equipment
```

---

### أهم Fields

| Field | وظيفتها |
| --- | --- |
| `name` | اسم Action الظاهر في الواجهة |
| `res_model` | Model المطلوب عرض بياناتها |
| `view_mode` | أنواع Views المتاحة |
| `search_view_id` | Search View محددة للـAction |
| `domain` | شروط إضافية على Records المعروضة |
| `context` | Context إضافية تُمرر إلى الواجهة |
| `target` | مكان فتح الواجهة |

وجود `res_model` لا ينشئ Model جديدة؛ يجب أن تكون Model موجودة ومحمّلة مسبقًا.

---

### ترتيب View Modes

في التعريف الأساسي:

```xml
<field name="view_mode">list,form</field>
```

نطلب إتاحة List وForm، وتبدأ الواجهة المعتادة بـList.

اكتب الأنواع مفصولة بفاصلة دون مسافات:

```text
list,form
```

ولا تستخدم:

```text
list, form
```

عندما لا نحدد View بعينها، تختار Odoo View الافتراضية المناسبة للـModel والنوع.

أما إذا أردنا ربط Views معينة، فيمكن تعريف Records من:

```text
ir.actions.act_window.view
```

سنستخدم هذه الطريقة في التطبيق لربط List وForm اللتين أنشأناهما في الشابتر السابق.

---

### Domain داخل Action

يمكن إضافة Domain لتخصيص Action تعرض الأجهزة المتاحة فقط:

```xml
<field name="domain">[('state', '=', 'available')]</field>
```

هذا السطر يوضع داخل Record الخاصة بالـAction.

الفرق بينه وبين Search Filter:

| المكان | السلوك |
| --- | --- |
| Action Domain | قيد إضافي أثناء استخدام هذه Action |
| Search Filter | خيار بحث يستطيع المستخدم تفعيله أو إزالته |

إزالة Filter من شريط البحث لا تزيل Domain المعرفة على Action نفسها.

لكن Action Domain لا تصبح قاعدة أمنية؛ الوصول إلى Records من مسار آخر يظل محكومًا بصلاحيات Server، كما تعلمنا في [[11 Security & Multi-Company]].

---

### Context داخل Action

يمكن تمرير قيمة افتراضية عند إنشاء جهاز من هذه الواجهة:

```xml
<field name="context">{'default_state': 'available'}</field>
```

المفتاح `default_state` يطلب استخدام `available` كقيمة افتراضية للـField `state`.

هذا لا يعني:

- تصفية القائمة لعرض الأجهزة المتاحة فقط.
- تغيير حالة Records الموجودة.
- منع المستخدم من اختيار حالة أخرى.

Domain تختار Records، بينما هذا الاستخدام من Context يوفر قيمة افتراضية.

راجع [[09 Environment & Context]] لمراجعة `default_` وبقية استخدامات Context.

---

## Menus

### Menu هي نقطة وصول

نستخدم عنصر `menuitem` لتعريف Record من Model:

```text
ir.ui.menu
```

مثلًا، قائمة رئيسية باسم Equipment:

```xml
<menuitem
    id="menu_equipment_root"
    name="Equipment"
    sequence="10"
/>
```

ثم قائمة فرعية تفتح Action الأجهزة:

```xml
<menuitem
    id="menu_equipment_assets"
    name="Assets"
    parent="menu_equipment_root"
    action="action_equipment"
    sequence="10"
/>
```

النتيجة المطلوبة:

```text
Equipment
└── Assets
```

القائمة الرئيسية تجمع القوائم الفرعية، بينما **Assets** مرتبطة بـAction.

---

### أهم Attributes

| Attribute | معناها |
| --- | --- |
| `id` | Local External ID للقائمة |
| `name` | الاسم الظاهر للمستخدم |
| `parent` | External ID للقائمة الأب |
| `action` | External ID للـAction المطلوب تشغيلها |
| `sequence` | ترتيب القائمة بين القوائم في المستوى نفسه |
| `groups` | المجموعات التي يسمح لها برؤية القائمة |

في `parent` و`action`، يمكن استخدام الاسم المختصر عند الإشارة إلى Records داخل Module نفسها.

---

### ظهور Menu والصلاحيات

يمكن تقييد ظهور القائمة باستخدام Group:

```xml
<menuitem
    id="menu_equipment_assets"
    name="Assets"
    parent="menu_equipment_root"
    action="action_equipment"
    groups="equipment_training.group_equipment_user"
/>
```

هذا يربط ظهور القائمة بعضوية المستخدم في Group المحددة.

كما تراعي Odoo صلاحية قراءة Model المرتبطة بـWindow Action عند حساب القوائم المتاحة.

> [!important] ظهور القائمة لا يمنح الوصول إلى البيانات
> `groups` داخل Menu تتحكم في ظهور نقطة الوصول.
>
> صلاحيات Model والقيود على Records تبقى مطلوبة على Server.

القائمة التي تعمل كحاوية قد لا تظهر إذا لم يكن تحتها أي عنصر متاح للمستخدم.

---

## Buttons

### Button من نوع `object`

تستدعي Python Method على Model الخاصة بالـView.

مثال داخل Form الأجهزة:

```xml
<button
    name="action_retire"
    type="object"
    string="Retire"
/>
```

معناه:

```text
استدعاء action_retire
على equipment.asset
باستخدام Record الحالية
```

يجب أن تكون Method موجودة في Python Class الخاصة بالـModel.

اسم `action_retire` اختيار تنظيمي واضح؛ السابقة `action_` لا تسجل Action تلقائيًا داخل Database.

---

### وضع Button داخل Header

الأزرار التي تنفذ عمليات على Record توضع عادةً داخل `header` في Form:

```xml
<form>
    <header>
        <button
            name="action_retire"
            type="object"
            string="Retire"
            class="btn-primary"
        />
    </header>

    <sheet>
        <group>
            <field name="name"/>
            <field name="state"/>
        </group>
    </sheet>
</form>
```

`header` و`sheet` هنا عنصران داخل `form`.

أما `class="btn-primary"`، فتؤثر على المظهر، ولا تمنح الزر صلاحيات إضافية.

سنؤجل Widgets مثل `statusbar` وخصائص الإظهار المشروط إلى 
[[14 Widgets & View Attributes]].

---

### Button من نوع `action`

تشغّل Action معرّفة مسبقًا بدل استدعاء Method بالاسم.

مثال على الصيغة:

```xml
<button
    name="%(equipment_training.action_equipment)d"
    type="action"
    string="Open Equipment"
/>
```

هنا يشير `name` إلى External ID للـAction.

صيغة `%(... )d` المستخدمة في المثال تحوّل المرجع إلى ID الخاصة بالـAction أثناء تحميل XML.

يجب أن تكون Action معرّفة قبل تحميل View التي تستخدم هذا المرجع.

هذا مثال لتوضيح النوع؛ التطبيق العملي أدناه سيضيف زر `object`.

---

### الفرق بين النوعين

| المقارنة | `type="object"` | `type="action"` |
| --- | --- | --- |
| قيمة `name` | اسم Python Method | مرجع إلى Action |
| نقطة البداية | تنفيذ Method على Model | تشغيل Action موجودة |
| النتيجة المحتملة | تعديل بيانات أو إرجاع Action | تنفيذ سلوك Action المحددة |

نوع الزر لا يحدد وحده ما إذا كانت شاشة جديدة ستُفتح؛ Method الخاصة بزر `object` يمكن أن تعيد Action أيضًا.

---

### حماية التنفيذ على Server

إخفاء Button عن بعض المستخدمين لا يمنع استدعاء Method العامة من مسار آخر.

لذلك نفرّق بين:

1. ظهور Button داخل View.
2. السماح بتنفيذ العملية داخل Python.
3. صلاحيات الوصول إلى Records التي ستتغير.

في التطبيق، سنجعل زر Retire ظاهرًا لـEquipment Manager، ونفحص Group نفسها داخل Method، ثم نستخدم `write()` لتطبيق التغيير مع صلاحيات ORM.

---

## Returning Actions from Python

### Action Record وAction Dictionary

يمكن تعريف Action داخل XML وتخزينها في Database.

ويمكن أيضًا أن تعيد Python Method وصف Action مباشرةً على شكل Dictionary.

الطريقة الثانية مفيدة عندما يعتمد ما نريد فتحه على Record الحالية أو نتيجة العملية.

---

### فتح Record الحالية في Dialog

هذا مثال مستقل لـMethod داخل `equipment.asset`:

```python
def action_open_details(self):
    self.ensure_one()

    return {
        "type": "ir.actions.act_window",
        "name": "Equipment Details",
        "res_model": "equipment.asset",
        "views": [(False, "form")],
        "res_id": self.id,
        "target": "new",
    }
```

معنى الأجزاء المهمة:

| الجزء | وظيفته |
| --- | --- |
| `ensure_one()` | تتطلب Method Record واحدة |
| `views` | تحدد أنواع Views ومعرّفاتها |
| `(False, "form")` | استخدام Form الافتراضية للـModel |
| `res_id` | تحديد Record التي ستُفتح |
| `target="new"` | فتحها داخل Dialog |

استخدمنا `views` في Dictionary لأنها القائمة التي تستخدمها الواجهة لتحديد Views المطلوب فتحها.

أما `view_mode` في Record المخزنة، فتساعد Server على تجهيز هذه القائمة.

---

### مكان الفتح

من قيم `target` المهمة:

| القيمة | النتيجة |
| --- | --- |
| `current` | فتح الواجهة في مساحة المحتوى الحالية |
| `new` | فتحها في Dialog |

فتح Form داخل Dialog لا يجعل Model من نوع Wizard.

الـWizards لها استخدام وتصميم خاصان، وسنشرحها في [[19 Wizards]].

---

## Example

سنكمل Module التدريب بهدفين:

1. فتح الأجهزة من **Equipment → Assets**.
2. السماح لـEquipment Manager بتحويل حالة جهاز إلى `retired` باستخدام Button.

نفترض أنك طبقت الشابترين السابقين، وأن Module تحتوي:

- Model `equipment.asset` وField `state`.
- Groups وصلاحيات الوصول من الشابتر 11.
- List وForm وSearch من الشابتر 12.

---

### 1. تجهيز الملفات

سنضيف ملفين داخل `views/`:

```text
equipment_training/
├── __manifest__.py
├── models/
│   └── equipment_asset.py
├── security/
│   ├── groups.xml
│   └── ir.access.csv
└── views/
    ├── equipment_views.xml
    ├── equipment_actions.xml
    └── equipment_menus.xml
```

الهيكل يعرض الملفات المرتبطة بهذا التطبيق فقط؛ احتفظ ببقية ملفات Module.

---

### 2. إضافة Python Method

افتح ملف Model الأجهزة الذي أنشأته سابقًا.

في هذا المثال، اسمه:

```text
models/equipment_asset.py
```

أضف `_` إلى Import الموجودة، ثم أضف Import الخاصة بـ`AccessError`:

```python
from odoo import _, fields, models
from odoo.exceptions import AccessError
```

احتفظ بأي Imports أخرى يحتاجها ملفك.

داخل Class `EquipmentAsset` الموجودة، أضف Method التالية، بنفس مستوى المسافة البادئة لتعريفات Fields:

```python
def action_retire(self):
    if not self.env.user.has_group(
        "equipment_training.group_equipment_manager"
    ):
        raise AccessError(
            _("Only equipment managers can retire equipment.")
        )

    self.write({"state": "retired"})
    return True
```

لا تنشئ Class ثانية، ولا تستبدل Fields الموجودة بهذا المقطع.

#### ماذا تفعل Method؟

- `has_group()` تتحقق من عضوية المستخدم الحالي في Group المدير.
- `AccessError` توقف التنفيذ إذا لم يكن عضوًا فيها.
- `_()` تجعل رسالة الخطأ قابلة للترجمة.
- `write()` تغير الحالة إلى القيمة المخزنة `retired`.
- `return True` تعيد نتيجة بسيطة، دون طلب فتح شاشة أخرى.

استخدمنا `write()` على `self` مباشرةً؛ العملية تدعم Recordset تحتوي جهازًا أو أكثر.

وعند الكتابة، تظل صلاحيات الوصول وقيود الشركات مطبقة. لم نستخدم `sudo()`.

هذا مثال لتوصيل Button بعملية بسيطة. تنظيم قواعد انتقال الحالات بالتفصيل يأتي في [[15 Business Logic]].

---

### 3. إضافة Button إلى Form

داخل:

```text
views/equipment_views.xml
```

ابحث عن Record التي تحمل:

```text
view_equipment_form
```

استبدل تعريف هذه Record فقط بالتالي، واحتفظ بتعريفَي List وSearch وبعنصر `odoo` الخارجي:

```xml
<record id="view_equipment_form" model="ir.ui.view">
    <field name="name">equipment.asset.form</field>
    <field name="model">equipment.asset</field>
    <field name="arch" type="xml">
        <form>
            <header>
                <button
                    name="action_retire"
                    type="object"
                    string="Retire"
                    class="btn-primary"
                    groups="equipment_training.group_equipment_manager"
                />
            </header>

            <sheet>
                <group>
                    <group string="Equipment">
                        <field name="name"/>
                        <field name="serial_number"/>
                        <field name="purchase_date"/>
                    </group>

                    <group string="Status &amp; Company">
                        <field name="state"/>
                        <field name="company_id"/>
                        <field name="active"/>
                    </group>
                </group>

                <group string="Description">
                    <field name="description"/>
                </group>
            </sheet>
        </form>
    </field>
</record>
```

الـExternal ID بقيت نفسها، ولذلك سيحدث Upgrade تعريف Form الموجودة.

الزر ظاهر لأعضاء Group المدير. وفحص المجموعة داخل Python يحمي استدعاء Method نفسها.

---

### 4. تعريف Window Action

داخل الملف الجديد:

```text
views/equipment_actions.xml
```

ضع:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <record id="action_equipment" model="ir.actions.act_window">
        <field name="name">Equipment</field>
        <field name="res_model">equipment.asset</field>
        <field name="view_mode">list,form</field>
        <field
            name="search_view_id"
            ref="equipment_training.view_equipment_search"
        />
    </record>

    <record
        id="action_equipment_view_list"
        model="ir.actions.act_window.view"
    >
        <field name="sequence" eval="10"/>
        <field name="view_mode">list</field>
        <field
            name="view_id"
            ref="equipment_training.view_equipment_list"
        />
        <field
            name="act_window_id"
            ref="equipment_training.action_equipment"
        />
    </record>

    <record
        id="action_equipment_view_form"
        model="ir.actions.act_window.view"
    >
        <field name="sequence" eval="20"/>
        <field name="view_mode">form</field>
        <field
            name="view_id"
            ref="equipment_training.view_equipment_form"
        />
        <field
            name="act_window_id"
            ref="equipment_training.action_equipment"
        />
    </record>
</odoo>
```

Record الأولى تعرف Action.

أما Record الثانية والثالثة، فتربطان Views محددة بهذه Action:

| Sequence | النوع | View |
| --- | --- | --- |
| 10 | List | `view_equipment_list` |
| 20 | Form | `view_equipment_form` |

وربطنا Search View باستخدام `search_view_id`.

لم نضف Domain للحالة إلى Action الأساسية، حتى تعرض الأجهزة بمختلف حالاتها ضمن صلاحيات المستخدم.

يمكن للمستخدم تفعيل Filter **Available** التي أنشأناها في الشابتر السابق.

---

### 5. تعريف Menus

داخل الملف الجديد:

```text
views/equipment_menus.xml
```

ضع:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <menuitem
        id="menu_equipment_root"
        name="Equipment"
        sequence="10"
        groups="equipment_training.group_equipment_user"
    />

    <menuitem
        id="menu_equipment_assets"
        name="Assets"
        parent="menu_equipment_root"
        action="action_equipment"
        sequence="10"
        groups="equipment_training.group_equipment_user"
    />
</odoo>
```

المستخدم الذي ينتمي إلى Group الأجهزة يستطيع رؤية نقطة الوصول.

Equipment Manager يرث Group المستخدم وفق تعريفنا في الشابتر 11، ولذلك تشمل القوائم المدير أيضًا.

> [!note] العلاقة مع `application`
> وجود `"application": False` في Manifest لا يمنع إنشاء Menu رئيسية.
>
> هذه القيمة تتعلق بتصنيف Module كتطبيق، ولا تعني أن Module لا تستطيع إضافة واجهة.

---

### 6. ضبط ترتيب التحميل

داخل `__manifest__.py`، اجعل الملفات المتعلقة بالتطبيق بهذا الترتيب:

```python
"data": [
    "security/groups.xml",
    "security/ir.access.csv",
    "data/equipment_category_data.xml",
    "views/equipment_views.xml",
    "views/equipment_actions.xml",
    "views/equipment_menus.xml",
],
```

احتفظ بأي ملفات إضافية يحتاجها مشروعك.

سبب الترتيب:

```text
Groups
    ↓
Views التي تشير إليها
    ↓
Actions التي تشير إلى Views
    ↓
Menus التي تشير إلى Actions
```

وفي التطبيق الحالي، تحميل Python Method الجديدة يحدث عند تشغيل Server قبل معالجة هذه الملفات.

---

### 7. إعادة التشغيل وتطبيق Upgrade

عدّلنا Python وXML، ولذلك:

1. احفظ الملفات.
2. أوقف Server التدريب.
3. شغّله مجددًا بالطريقة المعتادة.
4. افتح Database `odoo20_training`.
5. افتح **Apps** واحذف Filter **Apps** عند الحاجة.
6. ابحث عن `Equipment Training`.
7. اختر **Upgrade** من قائمة **⋮**.
8. بعد اكتمال العملية، أعد تحميل صفحة المتصفح لتحديث التنقل.

يمكن استخدام طريقة Upgrade من CLI بدل الواجهة، كما شرحنا في [[03 Modules & Addons]].

---

### 8. التحقق من التنقل والبحث

استخدم حساب تدريب لديه صلاحية Equipment Manager، ومسموح له بالعمل على الشركة المختارة.

إذا لم تكن حسابات الاختبار من الشابتر 11 محفوظة، جهز مستخدمًا من إعدادات المستخدمين وامنحه الدور المناسب.

تحقق من التالي:

1. ظهور **Equipment** ضمن التنقل.
2. فتح **Assets** لقائمة الأجهزة.
3. ظهور الأعمدة التي عرّفناها في List.
4. فتح جهاز لعرض Form.
5. توفر البحث بالاسم والرقم التسلسلي.
6. عمل Filter **Available** على القائمة.

إذا كانت القائمة فارغة، تحقق من وجود أجهزة ضمن الشركات المفعلة وصلاحيات المستخدم.

---

### 9. تجربة Button

باستخدام Equipment Manager:

1. أنشئ جهاز تدريب باسم `Training Laptop`.
2. اجعل حالته **Available**.
3. احفظ Record.
4. افتحها واضغط **Retire**.
5. تحقق من تغير الحالة إلى **Retired**.

ثم قارن السلوك بحساب Equipment User:

| الاختبار | Equipment Manager | Equipment User |
| --- | --- | --- |
| فتح قائمة الأجهزة المسموحة | مسموح | مسموح |
| قراءة جهاز ضمن النطاق المسموح | مسموح | مسموح |
| ظهور زر Retire | ظاهر | مخفي |
| استدعاء `action_retire` | يتطلب أيضًا صلاحية كتابة Record | يرفضه فحص Group |

إذا كنت تعرض القائمة مع Filter **Available**، فقد يختفي الجهاز من نتائجها بعد تحويله إلى **Retired**.

Record لم تُحذف؛ تغيرت حالتها، فلم تعد تطابق Filter.

أما `active`، فتبقى كما هي؛ تغيير الحالة إلى `retired` لا يؤرشف الجهاز تلقائيًا.

---

### 10. تشخيص المشاكل الشائعة

| المشكلة | ما تراجعه |
| --- | --- |
| Equipment Menu لا تظهر | عضوية المستخدم، صلاحية قراءة Model، تحميل ملف Menus، ثم تحديث الصفحة |
| External ID غير موجودة | الاسم وترتيب الملفات في Manifest |
| Button Method غير موجودة | مكان Method داخل Class، واستيراد ملف Model، وإعادة تشغيل Server |
| Retire مخفية عن المدير | Group الفعلية للمستخدم وتعريف `groups` على Button |
| تظهر رسالة منع وصول | فحص Group وصلاحية الكتابة وقيود الشركات |
| تفتح View مختلفة | ربط Views بالـAction التي تفتحها Menu |
| القائمة فارغة | Domain وSearch Filters والشركات المفعلة والبيانات الموجودة |

---

## Important Notes

- `type="object"` تبحث عن Python Method، بينما `type="action"` تشير إلى Action موجودة.
- السابقة `action_` في اسم Method ليست تسجيلًا تلقائيًا لـAction.
- إخفاء Button أو Menu لا يحمي عملية Server وحده؛ ضع فحوص العملية داخل مسار التنفيذ المناسب.
- Group Check داخل Method لا يمنح صلاحية كتابة Records؛ صلاحيات ORM مطلوبة أيضًا.
- Action Domain وContext Defaults تؤديان وظيفتين مختلفتين، ولا تستبدلان Security.
- لا تفترض أن كل Method تُستدعى على Singleton؛ استخدم `ensure_one()` عندما تتطلب العملية Record واحدة فعلًا.
- `target="new"` تحدد مكان فتح Form؛ لا تحول Model تلقائيًا إلى Wizard.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[14 Widgets & View Attributes]]

---

## Sources

- [Odoo 20 Documentation — Actions](https://www.odoo.com/documentation/20.0/developer/reference/backend/actions.html)
- [Odoo 20 Documentation — View Architectures](https://www.odoo.com/documentation/20.0/developer/reference/user_interface/view_architectures.html)
- [Odoo 20 Documentation — Data Files](https://www.odoo.com/documentation/20.0/developer/reference/backend/data.html)
- [Odoo 20 Documentation — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)
- [Odoo 20 Tutorial — Finally, Some UI To Play With](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/05_firstui.html)
- [Odoo 20 Tutorial — Ready For Some Action?](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/09_actions.html)
- [Odoo 20 Source — Window Actions](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_actions.py)
- [Odoo 20 Source — User Group Checks](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/res_users.py)