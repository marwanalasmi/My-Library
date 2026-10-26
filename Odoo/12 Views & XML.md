---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 12 Views & XML

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Views Fundamentals|Views Fundamentals]]
- [[#View Structure|View Structure]]
- [[#Basic View Types|Basic View Types]]
- [[#Writing XML|Writing XML]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في الشابترات السابقة، عرّفنا Models وFields، وتعاملنا مع Records من خلال ORM، ثم حددنا صلاحيات الوصول إليها.

الآن نحتاج إلى تنظيم طريقة عرض هذه البيانات للمستخدم.

مثلًا، Model الأجهزة تحتوي على الاسم والرقم التسلسلي والحالة والشركة. لكن وجود هذه Fields لا يحدد وحده:

- الأعمدة التي تظهر في قائمة الأجهزة.
- ترتيب Fields داخل صفحة الجهاز.
- خيارات البحث والفلاتر المتاحة.

هذه مسؤولية **Views**.

في هذا الشابتر، سنبني ثلاث Views للـModel `equipment.asset`:

| View | دورها |
| --- | --- |
| List | عرض الأجهزة في صفوف وأعمدة |
| Form | عرض تفاصيل جهاز واحد |
| Search | توفير خيارات البحث والفلاتر |

سنعرّفها باستخدام XML، ونحمّلها داخل Module التدريب.

> [!info] حدود التطبيق في هذا الشابتر
> سنجهز Views ونتحقق من تحميلها.
>
> إنشاء Action وMenu لفتحها من الواجهة يأتي في [[13 Actions, Menus & Buttons]].

---

## Views Fundamentals

### العلاقة بين Model وView

Model تحدد البيانات وسلوكها، بينما View تحدد طريقة عرضها والتفاعل معها من الواجهة.

مثلًا:

```text
equipment.asset
├── List View
├── Form View
└── Search View
```

الثلاث مرتبطة بالـModel نفسها، لكن لكل واحدة وظيفة مختلفة.

فتح الجهاز من List إلى Form لا ينشئ نسخة جديدة منه؛ نحن نعرض Record نفسها بطريقة أخرى.

---

### View هي Record أيضًا

تعريف View يُخزن كـRecord داخل Model تقنية اسمها:

```text
ir.ui.view
```

لذلك، ينطبق عليها ما تعلمناه عن Data Files في [[10 Data Files & External IDs]]:

- يمكن تعريفها داخل XML File.
- يمكن إعطاؤها External ID.
- يجب تحميل الملف من Manifest.
- يمكن تحديث تعريفها من خلال Module Upgrade.

ملف XML هو جزء من Source Code الخاصة بالـModule، أما View المحمّلة فهي Record داخل Database.

---

### XML لا تعني View دائمًا

Odoo تستخدم XML لتعريف أنواع مختلفة من Records، مثل Views وActions وMenus وإعدادات أخرى.

الذي يحدد أننا ننشئ View هو استهداف Model:

```xml
<record id="view_equipment_form" model="ir.ui.view">
    ...
</record>
```

أما XML نفسها، فهي صيغة لتنظيم البيانات والعناصر.

---

## View Structure

### التعريف الأساسي

هذا تعريف Form View بسيطة للـModel `equipment.asset`:

```xml
<odoo>
    <record id="view_equipment_form" model="ir.ui.view">
        <field name="name">equipment.asset.form</field>
        <field name="model">equipment.asset</field>
        <field name="arch" type="xml">
            <form>
                <sheet>
                    <group>
                        <field name="name"/>
                        <field name="state"/>
                    </group>
                </sheet>
            </form>
        </field>
    </record>
</odoo>
```

يتكون التعريف من مستويين:

1. بيانات Record الخاصة بالـView، مثل اسمها والـModel المرتبطة بها.
2. **View Architecture**، وهي العناصر الموجودة داخل `arch`.

---

### معنى الأسماء داخل التعريف

| الجزء | معناه |
| --- | --- |
| `view_equipment_form` | Local External ID الخاصة بالـView |
| `equipment.asset.form` | اسم تقني يساعدنا على تمييز View |
| `equipment.asset` | Model التي تعرض View بياناتها |
| `arch` | تعريف بنية View |
| `<form>` | العنصر الرئيسي الذي يحدد نوعها |

الـExternal ID الكاملة لهذا المثال هي:

```text
equipment_training.view_equipment_form
```

اسم View التقني ليس هو External ID، وليس بالضرورة عنوان الشاشة الذي يراه المستخدم.

---

### لماذا تظهر `model` مرتين؟

في هذا السطر:

```xml
<record id="view_equipment_form" model="ir.ui.view">
```

نطلب إنشاء Record داخل `ir.ui.view`.

أما هذا السطر:

```xml
<field name="model">equipment.asset</field>
```

فيحدد Model التي ستعرض View بياناتها.

إذًا:

```text
Model التي نخزن فيها تعريف View
→ ir.ui.view

Model التي تعرضها View
→ equipment.asset
```

---

### لماذا تظهر `field` داخل `field`؟

في المستوى الخارجي:

```xml
<field name="name">equipment.asset.form</field>
```

نضع قيمة في Field اسمها `name` داخل Record الخاصة بالـView.

أما داخل `arch`:

```xml
<field name="name"/>
```

فنطلب عرض Field اسمها `name` من الـModel `equipment.asset`.

الاسم متشابه، لكن السياق مختلف:

| الموقع | المقصود |
| --- | --- |
| خارج `arch` | بيانات Record الخاصة بالـView |
| داخل `arch` | عناصر العرض الخاصة بالـModel المستهدفة |

كتابة Field داخل View لا تنشئ Field جديدة في Model.

إذا كتبت:

```xml
<field name="serial_number"/>
```

فيجب أن تكون `serial_number` معرّفة مسبقًا في `equipment.asset`.

---

## Basic View Types

### List View

تعرض عدة Records في صفوف، وتعرض Fields المختارة كأعمدة.

في Odoo 20، العنصر الرئيسي هو:

```xml
<list>
    <field name="name"/>
    <field name="serial_number"/>
    <field name="state"/>
</list>
```

ترتيب عناصر `field` يحدد ترتيب الأعمدة في تعريف View.

هذا النوع مناسب عندما يريد المستخدم استعراض الأجهزة ومقارنة معلوماتها الأساسية بسرعة.

> [!note] صيغة Odoo 20
> نستخدم `<list>` لتعريف List View.
>
> قد تجد `<tree>` في أمثلة لإصدارات أقدم؛ لا تنقلها مباشرة إلى مشروع Odoo 20.

---

### Form View

تعرض تفاصيل Record واحدة، وتسمح بتحريرها عندما تسمح الصلاحيات وإعدادات الواجهة بذلك.

مثال على البنية الداخلية:

```xml
<form>
    <sheet>
        <group>
            <group string="Equipment">
                <field name="name"/>
                <field name="serial_number"/>
            </group>

            <group string="Status">
                <field name="state"/>
                <field name="company_id"/>
            </group>
        </group>
    </sheet>
</form>
```

العناصر المستخدمة هنا:

| العنصر | دوره |
| --- | --- |
| `form` | العنصر الرئيسي لـForm View |
| `sheet` | تنظيم المحتوى ضمن مساحة مناسبة لعرض النموذج |
| `group` | تنظيم Fields ومجموعاتها |
| `string` | النص الظاهر كعنوان للمجموعة |
| `field` | عرض Field من Model |

استخدام مجموعتين داخل `group` خارجية يساعد على توزيع المعلومات بدل وضع جميع Fields في قائمة واحدة طويلة.

أما `sheet`، فهي عنصر لتنسيق الواجهة؛ لا تشير إلى جدول في Database.

---

### Search View

توفر خيارات البحث والفلاتر التي تؤثر على Records المعروضة في View الحالية، مثل List.

هي لا تعرض تفاصيل Records بنفسها.

مثال على بنيتها الداخلية:

```xml
<search>
    <field name="name"/>
    <field name="serial_number"/>

    <filter
        name="available"
        string="Available"
        domain="[('state', '=', 'available')]"
    />
</search>
```

هنا أضفنا:

- البحث باستخدام `name`.
- البحث باستخدام `serial_number`.
- Filter جاهزة للأجهزة المتاحة.

في تعريف Filter:

| Attribute | معناها |
| --- | --- |
| `name` | اسم تقني للـFilter |
| `string` | النص الظاهر للمستخدم |
| `domain` | شرط تصفية Records |

استخدمنا القيمة `available` لأنها القيمة المخزنة في Selection Field، بينما `Available` هي Label الظاهرة للمستخدم.

راجع [[08 Domains]] إذا احتجت إلى مراجعة تركيب Domain.

> [!important] البحث والصلاحيات
> Search Filter تحدد ما يريد المستخدم عرضه ضمن البيانات المسموح له بالوصول إليها.
>
> لا تمنح صلاحيات إضافية، ولا تستبدل قواعد الوصول التي تعلمناها في [[11 Security & Multi-Company]].

---

### أنواع أخرى من Views

اختيار View يعتمد على طبيعة البيانات والمهمة التي ينفذها المستخدم.

| النوع | الاستخدام المعتاد |
| --- | --- |
| Kanban | عرض Records كبطاقات |
| Calendar | عرض Records مرتبطة بتواريخ |
| Graph | عرض بيانات مجمعة في رسوم بيانية |
| Pivot | تحليل البيانات باستخدام جدول تجميعي |

لا نضيف جميع الأنواع لمجرد أنها متاحة.

مثلًا، Calendar تحتاج Field مناسبة لتمثيل بداية الحدث. لا يكفي نسخ تعريف يشير إلى `start_at` إذا كانت هذه Field غير موجودة في Model.

في تطبيق هذا الشابتر، سنكتفي بـList وForm وSearch.

---

## Writing XML

### إغلاق العناصر وتداخلها

XML تتطلب إغلاق العناصر بطريقة صحيحة.

يمكن كتابة عنصر لا يحتوي محتوى داخليًا بهذه الصيغة:

```xml
<field name="name"/>
```

أما العنصر الذي يحتوي عناصر أخرى، فيحتاج إلى بداية ونهاية:

```xml
<group>
    <field name="name"/>
    <field name="state"/>
</group>
```

ويجب أن تُغلق العناصر بعكس ترتيب فتحها؛ أي نغلق العنصر الداخلي قبل الخارجي.

المسافات البادئة تساعد على قراءة هذا التداخل، لكنها لا تعوض الإغلاق الصحيح.

---

### التعامل مع الرموز الخاصة

بعض الرموز لها معنى خاص في XML، ولذلك نستخدم الصيغة المناسبة عند كتابتها داخل النصوص أو Attribute Values.

| الرمز | كتابته داخل XML |
| --- | --- |
| `<` | `&lt;` |
| `&` | `&amp;` |

مثلًا، لكتابة عنوان يحتوي على علامة **&**:

```xml
<group string="Identity &amp; Status">
    <field name="name"/>
    <field name="state"/>
</group>
```

المستخدم يرى:

```text
Identity & Status
```

انتبه لهذه النقطة عند نقل Domain تحتوي Operators أو مقارنات إلى XML؛ التعبير يجب أن يكون صحيحًا كـDomain، وأن تكون كتابته صحيحة كـXML أيضًا.

---

### استخدام الصيغة المناسبة للإصدار

قد تجد في أمثلة قديمة Attributes مثل `attrs` و`states` للتحكم في سلوك عناصر الواجهة.

هذه الصيغة غير مستخدمة في Views الحديثة في Odoo 20.

التحكم في خصائص مثل `invisible` و`readonly` و`required` يُشرح باستخدام الصيغة المناسبة في [[14 Widgets & View Attributes]].

---

### تعديل View موجودة

عندما نحتاج إلى توسيع View تابعة إلى Module أخرى، نستخدم عادةً **View Inheritance** بدل نسخ تعريفها كاملًا.

الفكرة هي الإشارة إلى View الأصلية، ثم تحديد الجزء الذي نريد تعديله.

سنشرح `inherit_id` وXPath وتطبيق التعديلات في [[18 Inheritance & Module Interaction]].

في هذا الشابتر، ننشئ Views أصلية للـModel التي بنيناها بأنفسنا.

---

## Example

سنجهز واجهات الأجهزة داخل Module `equipment_training`.

يعتمد التطبيق على:

- Model `equipment.asset` من [[04 Models & Fields]].
- Field `company_id` وصلاحيات الوصول من [[11 Security & Multi-Company]].
- طريقة تحميل Data Files من [[10 Data Files & External IDs]].

لن نضيف Python Fields جديدة في هذا التطبيق.

---

### 1. تجهيز ملف Views

داخل Module، أنشئ الملف:

```text
custom-addons/equipment_training/views/equipment_views.xml
```

مكانه ضمن الملفات الحالية:

```text
equipment_training/
├── __init__.py
├── __manifest__.py
├── models/
│   └── ...
├── security/
│   ├── groups.xml
│   └── ir.access.csv
├── data/
│   └── equipment_category_data.xml
└── views/
    └── equipment_views.xml
```

النقاط داخل `models/` تعني أن ملفات Models الموجودة تبقى كما هي.

إذا سبق أن أنشأت Views بنفس External IDs المستخدمة في الخطوة التالية، عدّل تعريفاتها الموجودة بدل تكرارها.

---

### 2. تعريف List وForm وSearch

ضع المحتوى التالي داخل `views/equipment_views.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<odoo>
    <!-- List View -->
    <record id="view_equipment_list" model="ir.ui.view">
        <field name="name">equipment.asset.list</field>
        <field name="model">equipment.asset</field>
        <field name="arch" type="xml">
            <list>
                <field name="name"/>
                <field name="serial_number"/>
                <field name="state"/>
                <field name="company_id"/>
            </list>
        </field>
    </record>

    <!-- Form View -->
    <record id="view_equipment_form" model="ir.ui.view">
        <field name="name">equipment.asset.form</field>
        <field name="model">equipment.asset</field>
        <field name="arch" type="xml">
            <form>
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

    <!-- Search View -->
    <record id="view_equipment_search" model="ir.ui.view">
        <field name="name">equipment.asset.search</field>
        <field name="model">equipment.asset</field>
        <field name="arch" type="xml">
            <search>
                <field name="name"/>
                <field name="serial_number"/>
                <field name="company_id"/>

                <filter
                    name="available"
                    string="Available"
                    domain="[('state', '=', 'available')]"
                />
            </search>
        </field>
    </record>
</odoo>
```

هذا الملف ينشئ ثلاث Records داخل `ir.ui.view`.

الـExternal IDs الكاملة هي:

```text
equipment_training.view_equipment_list
equipment_training.view_equipment_form
equipment_training.view_equipment_search
```

استخدمنا Fields الموجودة فعلًا في `equipment.asset`، ولم نضف Field لمجرد ظهور اسمها داخل XML.

---

### 3. تسجيل الملف في Manifest

داخل `equipment_training/__manifest__.py`، أضف ملف Views إلى قائمة `data` الموجودة.

مع ملفات الشابترات السابقة، تصبح القائمة:

```python
"data": [
    "security/groups.xml",
    "security/ir.access.csv",
    "data/equipment_category_data.xml",
    "views/equipment_views.xml",
],
```

إذا كانت القائمة تحتوي ملفات أخرى تحتاجها Module، احتفظ بها أيضًا.

المسارات هنا تبدأ من مجلد Module، ولذلك نكتب:

```text
views/equipment_views.xml
```

ولا نكتب المسار الكامل للمشروع.

> [!note] لا نستخدم Python Imports لتحميل XML
> ملفات Python تُربط من خلال `__init__.py`.
>
> أما ملف Views هذا، فيُحمّل من قائمة `data` داخل Manifest.

---

### 4. تطبيق Upgrade

احفظ الملفين:

```text
views/equipment_views.xml
__manifest__.py
```

ثم طبّق Upgrade على Module التدريب:

1. افتح Database `odoo20_training`.
2. افتح **Apps**.
3. احذف Filter **Apps** إذا كانت تمنع ظهور Module.
4. ابحث عن `Equipment Training`.
5. من قائمة **⋮** الخاصة بها، اختر **Upgrade**.
6. انتظر اكتمال العملية.

نفترض هنا أن Module مثبتة مسبقًا، وأن تغييرات Models من الشابتر السابق طُبقت بنجاح.

في هذه الخطوة عدّلنا XML وManifest فقط؛ لا نحتاج إلى إعادة تشغيل Server لتحميل Python جديدة.

إذا عدّلت Python أيضًا، أعد تشغيل Server قبل Upgrade من الواجهة، كما تعلمنا في [[03 Modules & Addons]].

مجرد Refresh للمتصفح لا يحمّل XML المعدّلة من ملفات Module في طريقة التشغيل العادية المستخدمة هنا.

---

### 5. التحقق من تحميل Views

بعد اكتمال Upgrade، افتح Odoo Shell على Database التدريب، كما تعلمنا في [[02 Development Environment]].

نفّذ:

```python
view_external_ids = [
    "equipment_training.view_equipment_list",
    "equipment_training.view_equipment_form",
    "equipment_training.view_equipment_search",
]

for external_id in view_external_ids:
    view = env.ref(external_id)
    print(view.name, view.model, view.type)
```

الناتج المتوقع:

```text
equipment.asset.list equipment.asset list
equipment.asset.form equipment.asset form
equipment.asset.search equipment.asset search
```

بهذا نتحقق من:

- وجود Records الخاصة بالـViews.
- ارتباطها بالـModel الصحيحة.
- تطابق نوع كل View مع تعريفها.

هذا فحص للتحميل والبيانات التقنية. تجربة العرض والبحث من واجهة الأجهزة تأتي بعد ربطها بـAction وMenu في الشابتر التالي.

---

### 6. فهم النتيجة الحالية

أصبح لدينا:

| الجزء | النتيجة |
| --- | --- |
| List View | تعريف أعمدة قائمة الأجهزة |
| Form View | تعريف صفحة تفاصيل الجهاز |
| Search View | تعريف خيارات البحث وFilter المتاح |
| Manifest | تحميل ملف Views مع Module |

عدم ظهور Equipment Menu حتى الآن متوقع؛ تعريف View لا ينشئ Menu تلقائيًا.

كذلك، صلاحيات المستخدم تظل مؤثرة:

- مستخدم Equipment العادي لديه صلاحية قراءة الأجهزة وفق إعدادات الشابتر السابق.
- Equipment Manager لديه صلاحيات أوسع ضمن نطاق الشركات المسموح بها.

إضافة Fields إلى Form لا تمنح المستخدم صلاحية تعديلها، وعرض `company_id` لا يسمح بتجاوز قيود الشركات.

---

### 7. تشخيص الأخطاء الشائعة

| المشكلة | ما تراجعه |
| --- | --- |
| رسالة تفيد بأن Field غير موجودة | الاسم التقني للـField والـModel، وهل حُمّل تعريف Python الخاص بها |
| XML Parsing Error | إغلاق العناصر، علامات الاقتباس، والرموز الخاصة |
| تعذر العثور على External ID | صحة الاسم وترتيب التحميل ووجود Module التي تعرفه |
| التعديل لا يظهر بعد حفظ XML | تسجيل الملف في Manifest وتطبيق Upgrade على Database الصحيحة |
| Views محمّلة ولا توجد Equipment Menu | لم ننشئ Action وMenu بعد |
| المستخدم يستطيع العرض ولا يستطيع التعديل | صلاحياته وإعدادات الواجهة |

عند فشل Upgrade، اقرأ رسالة الخطأ في Server Log؛ غالبًا تحدد الملف والموضع المرتبط بالمشكلة.

---

## Important Notes

- `<field>` داخل View تعرض Field موجودة؛ لا تنشئ Field في Model أو Database.
- External ID الخاصة بالـView تختلف عن اسمها التقني وعن Model التي تعرضها.
- وجود View داخل Database لا يعني وجود Menu تفتحها.
- Search Filters وإعدادات العرض لا تستبدل صلاحيات الوصول أو قيود الشركات.
- لا تضع Views التي تريد تحديثها مع Module داخل `noupdate="1"` دون سبب واضح؛ فقد يمنع ذلك تطبيق تعديلات XML على Records الموجودة أثناء Upgrade.
- عند إضافة عدة Filters، لا تفترض أن اختيارها معًا يعني AND دائمًا؛ الفلاتر المتجاورة ضمن المجموعة نفسها يمكن أن تُجمع باستخدام OR.
- في تطوير Module، احفظ تعديلات الواجهة في ملفاتها؛ التعديل اليدوي داخل Database وحده لا ينقل التغيير إلى نسخة Module لديك.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[13 Actions, Menus & Buttons]]

---

## Sources

- [Odoo 20 Documentation — View Records](https://www.odoo.com/documentation/20.0/developer/reference/user_interface/view_records.html)
- [Odoo 20 Documentation — View Architectures](https://www.odoo.com/documentation/20.0/developer/reference/user_interface/view_architectures.html)
- [Odoo 20 Tutorial — Basic Views](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/06_basicviews.html)
- [Odoo 20 Documentation — Data Files](https://www.odoo.com/documentation/20.0/developer/reference/backend/data.html)
- [Odoo 20 Documentation — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)