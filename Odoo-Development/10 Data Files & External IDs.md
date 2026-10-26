---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 10 Data Files & External IDs

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Data Files Fundamentals|Data Files Fundamentals]]
- [[#External IDs|External IDs]]
- [[#Writing Data Files|Writing Data Files]]
- [[#Loading & Update Behavior|Loading & Update Behavior]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

Odoo Modules يمكن أن توفر بيانات وتعريفات يحتاجها النظام، إلى جانب Python Code.

مثلًا، قد تحتاج Module إلى تحميل:

- تصنيفات أولية.
- إعدادات تشغيل.
- تعريفات Views وActions.
- تعريفات Security.
- بيانات تجريبية.

نكتب كثيرًا من هذه التعريفات في **Data Files**، ثم نحددها داخل Manifest حتى تعالجها Odoo.

وللإشارة إلى Records دون الاعتماد على أرقامها داخل Database، نستخدم **External IDs**.

في هذا الشابتر سنتعلم كيف نكتب Data Files، وكيف تشير إلى Records، وماذا يحدث عند إعادة تحميلها أثناء Module Upgrade.

سنستخدم تصنيفات Equipment كمثال، ونؤجل تفاصيل Security وViews وActions إلى شابتراتها.

---

## Data Files Fundamentals

### ما هي Data File؟

Data File هي ملف يحتوي تعريفات أوعمليات تعالجها Odoo أثناء تحميل بيانات Module.

من الصيغ الشائعة:

| الصيغة | الاستخدام المناسب |
| --- | --- |
| XML | تعريف Records وقيمها وعلاقاتها، مع مرونة في تنظيم العمليات |
| CSV | تعريف عدد من Records البسيطة التابعة لنفس Model |

مثلًا، يمكن أن توفر Module تصنيفين:

```text
Laptop
Monitor
```

بدل مطالبة المستخدم بإنشائهما يدويًا في كل Database، يمكن تعريفهما ضمن ملفات Module.

هذا لا يعني أن كل بيانات المستخدم يجب كتابتها داخل Data Files؛ نستخدمها للبيانات والتعريفات التي نريد توفيرها وإدارتها مع Module.

---

### Data Record ليست Model جديدة

هناك فرق بين:

- تعريف Model داخل Python.
- إنشاء Record داخل Model موجودة.

مثلًا:

```text
equipment.category
```

هي Technical Name لـModel.

أما Laptop وMonitor، فيمكن أن تكونا Records داخلها.

عندما نكتب:

```xml
<record id="category_laptop" model="equipment.category">
    <field name="name">Laptop</field>
</record>
```

فنحن نطلب إنشاء Record أوتحديثها داخل `equipment.category`.

هذا التعريف لا ينشئ Python Class جديدة، ولا يغني عن وجود Model نفسها.

يجب أن تكون Model وFields المستخدمة معروفة لـOdoo قبل معالجة بياناتها.

---

### Model التي سنستخدمها

سنستخدم Model باسم `equipment.category` تحتوي:

| Field | النوع | الغرض |
| --- | --- | --- |
| `name` | Char | اسم التصنيف |
| `parent_id` | Many2one إلى نفس Model | تصنيف رئيسي اختياري |
| `active` | Boolean | حالة النشاط |

سنكتب تعريفها كاملًا في Example باستخدام ما تعلمناه في [[04 Models & Fields]] و[[07 Relationships]].

الأمثلة النظرية التالية توضح Syntax وسلوك التحميل. تنفيذها والتحقق منها يأتي بعد تجهيز Model وتحميل الملفات في التطبيق العملي.

---

## External IDs

### ما هي External ID؟

الـ**External ID** هي مرجع منطقي تستخدمه Odoo للوصول إلى Record.

شكلها الكامل:

```text
module_name.local_id
```

مثلًا:

```text
equipment_training.category_laptop
```

تتكون من:

| الجزء | القيمة في المثال |
| --- | --- |
| Module Technical Name | `equipment_training` |
| Local ID | `category_laptop` |

Local ID اسم نختاره للمرجع، وليست رقم Record داخل Database.

---

### External ID وDatabase ID

لكل Record محفوظة Database ID رقمية، كما تعلمنا في [[06 Records & Recordsets]].

لكن الرقم قد يختلف بين Databases:

| Database | Record | Database ID |
| --- | --- | --- |
| Database A | Laptop | `42` |
| Database B | Laptop | `87` |

يمكن أن يشير نفس External ID في كل Database إلى Record المقابلة:

```text
equipment_training.category_laptop
                 ↓
       Model + Database ID
```

Odoo تحفظ هذا الربط ضمن Model تقنية باسم `ir.model.data`.

الفكرة المهمة:

> External ID توفر اسمًا منطقيًا للمرجع، بينما Database ID تحدد Record داخل Model في Database معينة.

وجود نفس النص في Code لا ينشئ Record أوالربط تلقائيًا؛ يجب تحميل التعريف أوتوفير المرجع في Database.

---

### أسماء متشابهة وأدوار مختلفة

في المثال التالي:

```xml
<record id="category_laptop" model="equipment.category">
    <field name="name">Laptop</field>
</record>
```

لدينا أكثر من نوع اسم:

| القيمة | معناها |
| --- | --- |
| `equipment.category` | Technical Name للـModel |
| `category_laptop` | Local External ID |
| `equipment_training.category_laptop` | External ID الكاملة |
| `Laptop` | قيمة Field باسم `name` |
| قيمة `record.id` في Python | Database ID الرقمية |

لا نستخدم هذه القيم مكان بعضها.

مثلًا، `equipment.category` ليست External ID لمجرد أنها تحتوي نقطة.

---

### ثبات المرجع وحدوده

External ID مفيدة عندما نحتاج إلى الإشارة إلى نفس التعريف في Databases مختلفة، دون افتراض رقم ثابت.

لكن:

- يجب أن تكون Record والمرجع موجودين.
- ليست كل Record منشأة يدويًا لها External ID بالضرورة.
- تغيير اسم Record لا يتطلب تغيير External ID.
- تغيير External ID ليس مجرد تعديل Label.

كذلك، يجب أن يكون Local ID فريدًا داخل Namespace الخاصة بالـModule، حتى لو كانت Records تتبع Models مختلفة.

اختر أسماء تصف دور Record، مثل:

```text
category_computers
category_laptop
category_monitor
```

---

### الوصول من Python باستخدام `env.ref()`

بعد تحميل Record وExternal ID الخاصة بها، يمكن الوصول إليها داخل Odoo Shell:

```python
laptop_category = env.ref(
    "equipment_training.category_laptop"
)
```

الناتج Record من Model المرتبطة بالمرجع.

مثلًا:

```python
laptop_category.name
```

داخل Model Method نستخدم نفس الفكرة من خلال `self.env`:

```python
laptop_category = self.env.ref(
    "equipment_training.category_laptop"
)
```

تستدعى `ref()` على Environment، ونمرر إليها External ID الكاملة.

لا تحتاج إلى البحث عن Record بالاسم أوكتابة Database ID الخاصة بها.

---

### ماذا يحدث إذا لم توجد External ID؟

افتراضيًا، ترفع `env.ref()` خطأ من نوع `ValueError` إذا لم تجد Record المطلوبة.

إذا كان المرجع اختياريًا، يمكن استخدام:

```python
optional_category = env.ref(
    "equipment_training.optional_category",
    raise_if_not_found=False,
)
```

عند عدم العثور عليها، تكون النتيجة:

```python
None
```

لذلك نتحقق قبل استخدام Fields:

```python
if optional_category is not None:
    print(optional_category.name)
```

أما إذا كان المرجع مطلوبًا لعمل Module، فلا نستخدم هذا الخيار لإخفاء مشكلة في الملف أوDependencies أوLoad Order.

والوصول إلى Record بواسطة External ID لا يمنح المستخدم صلاحيات إضافية عليها.

---

## Writing Data Files

### الشكل الأساسي لملف XML

يحتوي XML File على عنصر جذر باسم `<odoo>`، وداخله التعريفات المطلوبة.

مثلًا، ملف تصنيفات بسيط:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="category_laptop" model="equipment.category">
        <field name="name">Laptop</field>
    </record>
</odoo>
```

الأجزاء الأساسية:

| الجزء | دوره |
| --- | --- |
| `<odoo>` | عنصر الجذر |
| `<record>` | تعريف Record لإنشائها أوتحديثها |
| `model` | Technical Name للـModel |
| `id` | External ID الخاصة بالتعريف |
| `<field>` | تحديد قيمة Field |
| `name` داخل `<field>` | Technical Name للـField |

إذا كان الملف تابعًا لـ`equipment_training`، تصبح External ID الكاملة:

```text
equipment_training.category_laptop
```

يفضل توفير External ID للـRecords التي نريد تحديثها أوالإشارة إليها لاحقًا.

---

### قيم Fields و`eval`

يمكن كتابة قيمة نصية داخل `<field>`:

```xml
<field name="name">Laptop</field>
```

وعندما نحتاج قيمة مثل Boolean، نستخدم `eval` لتقييم Expression بسيطة:

```xml
<field name="active" eval="False"/>
```

هذا يمرر القيمة المنطقية `False`.

نحن نكتب تعريف بيانات داخل XML، وليس ملف Python عاديًا. بعض Attributes مثل `eval` تسمح بتقييم Expressions ضمن السياق الذي يوفره محمّل البيانات.

---

### الإشارة إلى Record أخرى باستخدام `ref`

لنفترض أن Laptop تصنيف فرعي من Computers.

نعرف Computers أولًا، ثم نشير إليها من Laptop:

```xml
<odoo>
    <record id="category_computers" model="equipment.category">
        <field name="name">Computers</field>
    </record>

    <record id="category_laptop" model="equipment.category">
        <field name="name">Laptop</field>
        <field
            name="parent_id"
            ref="equipment_training.category_computers"
        />
    </record>
</odoo>
```

الـAttribute باسم `ref` تحتوي External ID للـRecord المرتبطة.

Odoo تحل هذا المرجع وتستخدمه لإسناد العلاقة.

بما أن `parent_id` تشير إلى `equipment.category`، يجب أن تكون Record المشار إليها من Model المناسبة.

---

### المرجع المختصر والكامل

داخل XML التابعة لنفس Module، يمكن كتابة Local ID:

```xml
<field name="parent_id" ref="category_computers"/>
```

في هذا السياق، تكمل Odoo المرجع باسم Module الحالية.

أما الصيغة الكاملة:

```xml
<field
    name="parent_id"
    ref="equipment_training.category_computers"
/>
```

فتوضح Namespace صراحةً.

عند الإشارة إلى Record من Module أخرى:

- استخدم External ID الكاملة.
- صرح بالـModule المطلوبة داخل `depends`.
- تأكد أن المرجع متاح قبل استخدامه.

ذكر External ID داخل `ref` لا يضيف Dependency تلقائيًا.

أما في `env.ref()` فنستخدم الصيغة الكاملة.

---

### CSV Data Files

CSV مناسبة عندما نريد تعريف Records بسيطة من نفس Model في جدول.

عند تحميلها كـModule Data File، نستخدم اسم Model في اسم الملف.

في مثالنا:

```text
data/equipment.category.csv
```

محتوى كامل لتصنيفات مترابطة:

```csv
id,name,parent_id:id
category_computers,Computers,
category_laptop,Laptop,equipment_training.category_computers
category_monitor,Monitor,equipment_training.category_computers
```

معنى الأعمدة:

| العمود | معناه |
| --- | --- |
| `id` | External ID للـRecord |
| `name` | قيمة Field |
| `parent_id:id` | إسناد العلاقة باستخدام External ID |

الخلية الأخيرة في صف Computers فارغة لأنها لا ترتبط بتصنيف رئيسي في هذا المثال.

ونحدد الملف داخل Manifest:

```python
"data": [
    "data/equipment.category.csv",
],
```

هذا بديل لتعريف نفس القيم والعلاقات باستخدام XML.

> [!important] اختر صيغة المثال قبل تحميلها
> التطبيق العملي في هذا الشابتر سيستخدم XML حتى نجرب `noupdate` على جزء من الملف.
>
> لا تضف XML وCSV معًا لنفس بيانات التجربة؛ فقد تعيد معالجة نفس External IDs حسب ترتيب التحميل.
>
> CSV السابقة توضح القيم والعلاقات، ولا تتضمن حماية `noupdate` الجزئية الموجودة في مثال XML العملي.

---

## Loading & Update Behavior

### إدراج الملفات في Manifest

وجود Data File داخل Directory لا يجعل Odoo تحملها تلقائيًا.

نضع Paths الخاصة بها داخل القائمة المناسبة في `__manifest__.py`.

مثلًا:

```python
"data": [
    "data/equipment_category_data.xml",
],
```

المسار يبدأ من جذر Module:

```text
equipment_training/
├── __manifest__.py
└── data/
    └── equipment_category_data.xml
```

هذه القائمة جزء من Manifest Dictionary، وليست ملف Python مستقلًا.

تعالج Odoo هذه الملفات أثناء Installation وUpgrade، وفق قواعد التحميل الخاصة بكل تعريف.

Restart وحدها لا تعيد تطبيق Data Files.

---

### Load Order

الترتيب مهم على أكثر من مستوى:

| المستوى | ما الذي يحدد الترتيب؟ |
| --- | --- |
| بين Modules | Dependencies داخل `depends` |
| بين Data Files | ترتيب Paths في Manifest |
| داخل XML File | ترتيب العمليات والتعريفات |

مثلًا، إذا قسمنا التعريفات إلى ملفين:

```python
"data": [
    "data/equipment_category_data.xml",
    "data/equipment_asset_data.xml",
],
```

يمكن للملف الثاني الإشارة إلى Categories معرفة في الأول.

وداخل نفس XML، يجب تعريف Record قبل أن تشير إليها عملية لاحقة.

لذلك نضع Computers قبل Laptop عندما تعتمد Laptop على مرجعها.

---

### كيف تعرف Odoo أنها ستحدث Record؟

عند تحميل تعريف يحمل External ID:

- إذا لم توجد Record المقابلة، يمكن إنشاء Record وربطها بالمرجع.
- إذا كانت موجودة، تستخدم Odoo المرجع لتحديد Record التي ستعالجها وفق قواعد التحديث.

مثلًا:

```xml
<record id="category_laptop" model="equipment.category">
    <field name="name">Portable Computers</field>
</record>
```

إذا كانت `category_laptop` تشير بالفعل إلى Record، فهذا تعريف لتحديثها، وليس طلبًا بإنشاء Category جديدة بسبب اختلاف الاسم.

المفتاح هنا هو External ID، وليس قيمة `name`.

> [!important] احتفظ بالمرجع عند تغيير الاسم
> إذا أردت تغيير اسم Laptop إلى Portable Computers، عدّل Field باسم `name`.
>
> لا تغير `category_laptop` لمجرد تغيير الاسم الظاهر؛ تغيير External ID قد يجعل المحمّل يتعامل مع تعريف مختلف.

---

### الحقول المحذوفة من XML

عدم ذكر Field في تحديث XML لا يعني طلب مسح قيمتها.

مثلًا، إذا كان التعريف الجديد يحتوي `name` فقط، فلا نفترض أن `parent_id` ستصبح فارغة لمجرد عدم كتابتها.

لمسح علاقة اختيارية صراحةً، يمكن كتابة:

```xml
<field name="parent_id" eval="False"/>
```

هذا يختلف عن حذف سطر Field من الملف.

وتظل صلاحية القيمة وقواعد Model مطبقة أثناء المعالجة.

---

### `noupdate`

نستخدم `noupdate` عندما نريد توفير قيمة أولية مع حماية Record الموجودة من إعادة تطبيق تعريفها أثناء Upgrade المعتادة.

يمكن تطبيقها على الملف كله:

```xml
<odoo noupdate="1">
    <record id="category_monitor" model="equipment.category">
        <field name="name">Monitor</field>
    </record>
</odoo>
```

أوعلى جزء من الملف:

```xml
<odoo>
    <data noupdate="1">
        <record id="category_monitor" model="equipment.category">
            <field name="name">Monitor</field>
        </record>
    </data>
</odoo>
```

الصيغتان صحيحتان، لكن الثانية تسمح بجمع تعريفات محمية وغير محمية داخل نفس الملف.

السلوك الأساسي:

| الحالة | السلوك المعتاد |
| --- | --- |
| Record جديدة يحتاج المحمّل إلى إنشائها | تنشأ بالقيم الأولية |
| Record موجودة وغير محمية | يعاد تطبيق القيم المحددة أثناء Upgrade |
| Record موجودة ومحمية بـ`noupdate` | لا يعاد تطبيق قيم XML عليها أثناء Upgrade المعتادة |

هذه الحماية تخص سلوك التحميل والتحديث؛ ليست منعًا عامًا من تعديل Record.

يمكن تعديلها من الواجهة أوPython عندما تسمح الصلاحيات وقواعد Model بذلك.

---

### حدود `noupdate`

من المهم عدم اختصار معناها إلى «لن يحدث أي شيء بعد التثبيت».

مثلًا، يمكن أن تنشأ Record جديدة معرفة داخل `noupdate` أثناء Upgrade إذا لم تكن موجودة، وفق إعدادات الإنشاء المستخدمة.

كذلك، حالة الحماية تحفظ مع External ID. حذف `noupdate` من XML لاحقًا لا يضمن وحده تغيير حالة Record المثبتة.

تغيير سلوك بيانات مثبتة يحتاج فهم حالتها وخطة التحديث المناسبة. سنتناول حالات الترحيل في [[31 Upgrades & Data Migration]].

---

### Data وDemo Data

Manifest تميز بين بيانات Module العادية وDemo Data:

| القائمة | الاستخدام |
| --- | --- |
| `data` | تعريفات وبيانات تحتاجها Module للعمل |
| `demo` | بيانات اختيارية للتجربة والعرض |

مثلًا، Categories أساسية تعتمد عليها الوظيفة توضع ضمن `data`.

أما Records إضافية لتوضيح طريقة الاستخدام، فيمكن وضعها ضمن `demo`.

تحميل Demo Data يعتمد على إعدادات التثبيت والـDatabase، لذلك لا تجعل وظيفة أساسية تعتمد على وجودها.

وقواعد External IDs والمراجع وترتيب التحميل تبقى مهمة في الحالتين.

---

## Example

سنضيف Model لتصنيفات Equipment، ثم نحمل ثلاث Categories:

- Computers.
- Laptop.
- Monitor.

سنسمح بتحديث Laptop من XML، ونحمي Monitor باستخدام `noupdate` حتى نلاحظ الفرق.

نفترض أن Module باسم `equipment_training` مثبتة داخل `odoo20_training`، كما تعلمنا في [[03 Modules & Addons]].

---

### 1. تجهيز الملفات

سنضيف الملفات التالية إلى Module الموجودة:

```text
equipment_training/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── equipment_category.py
└── data/
    └── equipment_category_data.xml
```

هذه الشجرة تعرض الملفات المتعلقة بالمثال. احتفظ ببقية Files وImports الموجودة في Module.

إذا كانت لديك Model معرفة بالفعل بنفس Technical Name، استخدم تعريفها المناسب بدل إنشاء Class أخرى بنفس الاسم.

---

### 2. تعريف Model

داخل:

```text
equipment_training/models/equipment_category.py
```

اكتب:

```python
from odoo import fields, models


class EquipmentCategory(models.Model):
    _name = "equipment.category"
    _description = "Equipment Category"
    _order = "name, id"

    name = fields.Char(required=True)

    parent_id = fields.Many2one(
        "equipment.category",
        string="Parent Category",
    )

    active = fields.Boolean(default=True)
```

استخدمنا مفاهيم سابقة:

- Model وFields من [[04 Models & Fields]].
- Many2one من [[07 Relationships]].

`parent_id` تشير إلى نفس Model، حتى نتمكن من جعل Laptop وMonitor تصنيفين فرعيين من Computers.

---

### 3. ربط Python Imports

تأكد أن ملف:

```text
equipment_training/__init__.py
```

يحتوي استيراد `models`:

```python
from . import models
```

ثم أضف إلى:

```text
equipment_training/models/__init__.py
```

السطر:

```python
from . import equipment_category
```

احتفظ بالـImports السابقة. لا تستبدل محتوى الملف كله بهذا السطر إذا كان يحتوي Models أخرى.

---

### 4. كتابة XML Data

داخل:

```text
equipment_training/data/equipment_category_data.xml
```

اكتب:

```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <record id="category_computers" model="equipment.category">
        <field name="name">Computers</field>
    </record>

    <record id="category_laptop" model="equipment.category">
        <field name="name">Laptop</field>
        <field
            name="parent_id"
            ref="equipment_training.category_computers"
        />
    </record>

    <data noupdate="1">
        <record id="category_monitor" model="equipment.category">
            <field name="name">Monitor</field>
            <field
                name="parent_id"
                ref="equipment_training.category_computers"
            />
        </record>
    </data>
</odoo>
```

لاحظ:

- Computers معرفة قبل Records التي تشير إليها.
- Laptop خارج `noupdate`.
- Monitor داخل `noupdate`.
- كل Record لها External ID واضحة.

---

### 5. إضافة الملف إلى Manifest

داخل `equipment_training/__manifest__.py`، أضف المسار إلى قائمة `data`:

```python
"data": [
    "data/equipment_category_data.xml",
],
```

إذا كانت القائمة تحتوي ملفات سابقة، احتفظ بها وضع الملف في الترتيب المناسب.

وتأكد أن Dependency على `base` موجودة كما في Module التدريب السابقة.

---

### 6. تحميل التغيير من الواجهة

أضفنا Python Model جديدة، لذلك نحتاج إلى تحميل Python Code الحالية ثم تطبيق Upgrade.

1. احفظ Files.
2. أوقف Server التدريب.
3. شغّلها من جديد باستخدام أمر التشغيل المعتاد في [[02 Development Environment]].
4. افتح **Apps**.
5. احذف Filter **Apps**.
6. ابحث عن `Equipment Training`.
7. اختر **Upgrade**.

رغم أن Module مثبتة مسبقًا، فإن Categories جديدة. لذلك يستطيع التحميل إنشاء Monitor أيضًا، ثم يحميها من إعادة تطبيق قيمها في Upgrades اللاحقة.

لن تظهر شاشة لإدارة Categories تلقائيًا؛ لم ننشئ Views أوActions أوMenus بعد.

سنفحص النتيجة من Odoo Shell التي تعلمنا استخدامها سابقًا.

---

### 7. التحقق باستخدام External IDs

افتح جلسة Odoo Shell مرتبطة بـ`odoo20_training`.

استخدم بيئة التدريب التي تعلمناها سابقًا، مع الانتباه إلى سياق المستخدم والصلاحيات كما شرحنا في [[09 Environment & Context]].

احصل على Records:

```python
computers_category = env.ref(
    "equipment_training.category_computers"
)

laptop_category = env.ref(
    "equipment_training.category_laptop"
)

monitor_category = env.ref(
    "equipment_training.category_monitor"
)
```

تحقق من الاسم:

```python
laptop_category.name
```

النتيجة المتوقعة:

```python
"Laptop"
```

وتحقق من العلاقة:

```python
laptop_category.parent_id == computers_category
```

النتيجة المتوقعة:

```python
True
```

ثم:

```python
monitor_category.parent_id == computers_category
```

النتيجة المتوقعة:

```python
True
```

يمكنك قراءة:

```python
laptop_category.id
```

ستظهر Database ID الفعلية، لكننا لم نحتج إلى كتابتها داخل Module للوصول إلى Record.

بعد الفحص، اخرج من Shell قبل متابعة تجربة التحديث.

---

### 8. تعديل قيم XML

داخل ملف XML، غير اسم Laptop:

```xml
<record id="category_laptop" model="equipment.category">
    <field name="name">Portable Computers</field>
    <field
        name="parent_id"
        ref="equipment_training.category_computers"
    />
</record>
```

وغير اسم Monitor داخل مجموعة `noupdate`:

```xml
<data noupdate="1">
    <record id="category_monitor" model="equipment.category">
        <field name="name">Displays</field>
        <field
            name="parent_id"
            ref="equipment_training.category_computers"
        />
    </record>
</data>
```

استبدل التعريفين الموجودين، ولا تضف نسختين جديدتين منهما.

احتفظ بنفس External IDs، ثم احفظ الملف.

---

### 9. تطبيق Upgrade والتحقق

نفذ **Upgrade** مرة أخرى من Apps.

في هذه الخطوة تغير XML فقط؛ لا نحتاج Restart لتحميل Python Code جديدة.

بعد اكتمال العملية، افتح **جلسة Odoo Shell جديدة** حتى نفحص حالة Database بعد التحديث، دون الاعتماد على بيانات مخزنة في جلسة الفحص السابقة.

نفذ:

```python
env.ref(
    "equipment_training.category_laptop"
).name
```

النتيجة المتوقعة:

```python
"Portable Computers"
```

ثم:

```python
env.ref(
    "equipment_training.category_monitor"
).name
```

النتيجة المتوقعة:

```python
"Monitor"
```

ملخص ما حدث:

| Record | الاسم الجديد داخل XML | النتيجة في Database |
| --- | --- | --- |
| Laptop | Portable Computers | تغير الاسم |
| Monitor | Displays | بقي Monitor بسبب `noupdate` |

External ID الخاصة بـLaptop بقيت:

```text
equipment_training.category_laptop
```

لم نغيرها لأننا غيرنا الاسم الظاهر فقط.

---

### 10. البديل: تطبيق Upgrade من CLI

يمكن استخدام CLI بدل زر Upgrade. لا تحتاج إلى تنفيذ الطريقتين لنفس التغيير.

أوقف Server التدريب أولًا، ثم نفذ الأمر من Project Root.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_training \
  -u equipment_training \
  --stop-after-init
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\odoo.conf `
  -d odoo20_training `
  -u equipment_training `
  --stop-after-init
```

تبدأ Process جديدة فتقرأ Python Code وManifest الحالية، ثم تنفذ Upgrade وتتوقف.

بعد اكتمالها، يمكنك تشغيل Server بشكل طبيعي، أوفتح Shell جديدة للتحقق من النتائج.

---

## Important Notes

- **External ID ليست Database ID ولا Model Name.** لكل واحدة دور مختلف.

- **لا تفترض وجود External ID لكل Record.** يجب أن يكون المرجع معرفًا ومتاحًا قبل استخدامه.

- **غيّر قيمة `name` لتغيير الاسم الظاهر.** لا تغير External ID لمجرد إعادة تسمية Record.

- **وجود Data File داخل Module لا يكفي لتحميلها.** يجب إدراجها في Manifest، ومراعاة Dependencies وترتيب العمليات.

- **Restart لا تعيد تطبيق XML وCSV Data.** نستخدم Installation أوUpgrade المناسبة.

- **عدم ذكر Field في XML لا يعني مسح قيمتها.** طلب قيمة فارغة يختلف عن حذف التعريف من الملف.

- **`noupdate` ليست حماية من كل تعديل.** هي تؤثر في إعادة تطبيق البيانات أثناء التحديث، ولا تمنع تعديل Record بالطرق الأخرى المسموحة.

- **إزالة `noupdate` من XML ليست ضمانًا لتغيير حالة بيانات مثبتة.** قد تحتاج معالجة ترحيل مدروسة.

- **لا تعتمد على Demo Data لتشغيل وظيفة أساسية.** قد لا تكون محملة في Database.

- **تحقق أن Git تتتبع Data Files الخاصة بالـModules.** عند استبعاد Runtime Data في جذر المشروع، استخدم Pattern محددة مثل `/data/` حتى لا تستبعد Directories باسم `data` داخل Custom Modules.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[11 Security & Multi-Company]]

---

## Sources

- [Odoo 20 — Data Files](https://www.odoo.com/documentation/20.0/developer/reference/backend/data.html)
- [Odoo 20 — CSV Data Files](https://www.odoo.com/documentation/20.0/developer/reference/backend/data.html#csv-data-files)
- [Odoo 20 — Module Manifests](https://www.odoo.com/documentation/20.0/developer/reference/backend/module.html)
- [Odoo 20 — Environment Methods](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html#useful-environment-methods)
- [Odoo 20 Source — Data File Loading](https://github.com/odoo/odoo/blob/20.0/odoo/tools/convert.py)
- [Odoo 20 Source — External ID Mapping](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_model.py)
- [Odoo 20 Source — Environment Reference Lookup](https://github.com/odoo/odoo/blob/20.0/odoo/orm/environments.py)
- [Odoo 20 Source — CSV Reference Example](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/data/res.country.state.csv)