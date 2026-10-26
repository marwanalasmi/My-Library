---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 20 QWeb & Reports

## Table of Contents

- [[#Introduction|Introduction]]
- [[#QWeb Fundamentals|QWeb Fundamentals]]
- [[#Report Fundamentals|Report Fundamentals]]
- [[#Practical Example|Practical Example]]
- [[#Extending Reports|Extending Reports]]
- [[#Common Problems|Common Problems]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

تعلمنا سابقًا كيف نعرض Records داخل List وForm، وكيف ننفذ عمليات باستخدام Buttons وWizards.

لكننا نحتاج أحيانًا إلى إخراج البيانات في مستند مناسب للطباعة أوالمشاركة، مثل:

- بطاقة بيانات جهاز.
- مستند إعارة.
- فاتورة.
- ملخص عملية.

في Odoo، يمكن إنشاء هذه المستندات باستخدام **QWeb Reports**.

نكتب Template تحدد محتوى المستند، ثم نعرّف Report Action لربطها بالـModel وإتاحتها للمستخدم.

```text
Records
    ↓
QWeb Template
    ↓
HTML
    ↓
عرض التقرير أوإنشاء PDF
```

في هذا الشابتر، سنضيف تقريرًا باسم **Equipment Sheet** إلى مشروع التدريب.

سيعرض بيانات الأجهزة الموجودة بالفعل داخل `equipment.asset`، ويمكن تشغيله من قائمة **Print**.

---

## QWeb Fundamentals

### ما هي QWeb؟

QWeb هي Template Engine تعتمد على XML.

تجمع بين عناصر العرض العادية، مثل:

```xml
<h2>Equipment</h2>

<p>Equipment details</p>
```

وتعليمات ديناميكية تسمى **Directives**، تبدأ عادةً بـ`t-`:

```xml
<span t-field="o.name"/>
```

هنا لا نكتب اسم الجهاز يدويًا؛ نطلب من QWeb قراءة الحقل `name` من Record المشار إليها بالمتغير `o`.

تُستخدم QWeb في تقارير Server وصفحات Website، وتوجد صياغة Templates مرتبطة بها في Frontend وOWL.

> [!note] نطاق هذا الشابتر
> نركز هنا على Server QWeb المستخدمة في التقارير.
>
> Expressions في هذه Templates تُقيّم في سياق Python. لا تنقل إليها Expressions خاصة بـJavaScript أوOWL دون مراجعة.
>
> سنتناول Frontend في [[24 OWL & Frontend]].

---

### العنصر `<t>`

العنصر `<t>` يحمل تعليمات QWeb دون إضافة عنصر HTML مستقل إلى الناتج.

مثلًا:

```xml
<t t-if="o.serial_number">
    <p>Serial number is available.</p>
</t>
```

إذا تحقق الشرط، يظهر `<p>` في الناتج.

أما `<t>` نفسها فلا تظهر كعنصر HTML.

ويمكن وضع Directive على العنصر المطلوب مباشرةً:

```xml
<p t-if="o.serial_number">
    Serial number is available.
</p>
```

هنا يظهر `<p>` أو يُحذف وفق الشرط.

---

### عرض Expression باستخدام `t-out`

تقيّم `t-out` Expression وتعرض نتيجتها:

```xml
<span t-out="o.name"/>
```

ويمكن استخدامها لعرض نتيجة عامة، مثل عدد Records:

```xml
<span t-out="len(docs)"/>
```

تعالج `t-out` النصوص العادية باستخدام HTML Escaping، حتى لا تتحول الرموز الموجودة فيها إلى HTML تنفيذي.

مثلًا، النص:

```text
Laptop <Training>
```

يُعرض كنص، بدل تفسير `<Training>` كعنصر HTML.

توجد قيم تحمل علامة أنها HTML آمنة مسبقًا، ولها معالجة مختلفة؛ لذلك لا تحول مدخلات المستخدم إلى محتوى موثوق دون سبب وفحص مناسبين.

---

### عرض Field باستخدام `t-field`

عندما نعرض Field من Odoo Record، يمكن استخدام:

```xml
<span t-field="o.purchase_date"/>
```

تستفيد `t-field` من نوع الحقل لتنسيق القيمة.

مثلًا:

| نوع الحقل | مثال على التنسيق |
| --- | --- |
| Date | تنسيق التاريخ |
| Datetime | تنسيق التاريخ والوقت |
| Selection | عرض تسمية الاختيار |
| Many2one | عرض اسم Record المرتبطة |

تحتاج `t-field` إلى وصول لحقل على Odoo Record، مثل:

```xml
<span t-field="o.name"/>
```

ولا نستخدمها لتعبير حسابي عام:

```xml
<span t-out="len(docs)"/>
```

> [!tip] اختيار Directive المناسبة
> استخدم `t-field` عند عرض Field والاستفادة من تنسيقها.
>
> واستخدم `t-out` لعرض Expressions أوالقيم العامة.

---

### تخصيص التنسيق باستخدام `t-options`

يمكن تمرير خيارات إضافية لتنسيق الحقل:

```xml
<span
    t-field="o.purchase_date"
    t-options="{'format': 'yyyy-MM-dd'}"
/>
```

هنا حددنا شكل عرض التاريخ.

الخيارات المتاحة تعتمد على Field Converter أوWidget المستخدمة في QWeb.

Widgets التقارير ليست بالضرورة مطابقة لكل Widgets واجهة Form التي تناولناها في [[14 Widgets & View Attributes]].

---

### الشروط باستخدام `t-if`

يمكن إظهار جزء من التقرير عندما توجد قيمة:

```xml
<p t-if="o.serial_number">
    <strong>Serial Number:</strong>
    <span t-field="o.serial_number"/>
</p>
```

إذا كان `serial_number` فارغًا، لا تظهر الفقرة.

ويمكن استخدام فروع إضافية:

```xml
<p t-if="o.state == 'available'">
    Available for use.
</p>

<p t-elif="o.state == 'in_use'">
    Currently in use.
</p>

<p t-else="">
    Retired from use.
</p>
```

هذه الشروط تتحكم في محتوى المستند، ولا تمنح صلاحيات وصول إلى البيانات.

---

### التكرار باستخدام `t-foreach`

لإنشاء محتوى لكل Record:

```xml
<t t-foreach="docs" t-as="o">
    <p>
        <span t-field="o.name"/>
    </p>
</t>
```

هنا:

- `docs` هي مجموعة Records الخاصة بالتقرير.
- `t-foreach` تمر عليها.
- `t-as="o"` تسمي Record الحالية داخل التكرار.

الاسم `o` مجرد اختيار شائع، ويمكن استخدام اسم أوضح:

```xml
<t t-foreach="docs" t-as="asset">
    <p>
        <span t-field="asset.name"/>
    </p>
</t>
```

لكن بعض Layouts القياسية تتوقع متغيرًا باسم `o`؛ لذلك سنستخدمه في التقرير العملي.

---

### تعريف متغير باستخدام `t-set`

يمكن تعريف قيمة لاستخدامها لاحقًا:

```xml
<t
    t-set="report_title"
    t-value="'Equipment Sheet'"
/>

<h2 t-out="report_title"/>
```

ويمكن تعريف القيمة من محتوى العنصر:

```xml
<t t-set="report_title">
    Equipment Sheet
</t>

<h2 t-out="report_title"/>
```

` t-set` نفسها لا تطبع القيمة؛ العرض يحدث عند استخدامها لاحقًا.

استخدم هذه المتغيرات لتوضيح العرض، مع الاحتفاظ بالحسابات وقواعد العمل المعقدة داخل Python.

---

### إعادة استخدام Template باستخدام `t-call`

تستدعي `t-call` Template أخرى:

```xml
<t t-call="web.external_layout">
    <div class="page">
        <p>Equipment details</p>
    </div>
</t>
```

بهذا نستخدم Layout موجودة بدل بناء الترويسة والتذييل من البداية.

يمكن أيضًا تمرير متغيرات إلى Template المستدعاة داخل جسم `t-call`:

```xml
<t t-call="web.external_layout">
    <t t-set="company" t-value="o.company_id"/>

    <div class="page">
        <span t-field="o.name"/>
    </div>
</t>
```

سنستخدم ذلك لتحديد شركة الجهاز عند طباعة التقرير.

---

## Report Fundamentals

### الفرق بين Template وReport Action

يتكون التقرير الأساسي من جزأين:

| الجزء | المسؤولية |
| --- | --- |
| QWeb Template | تحديد محتوى المستند وطريقة عرضه |
| Report Action | تحديد Model والتنسيق والـTemplate وربط التقرير بالواجهة |

إنشاء Template وحدها لا يضيف خيارًا إلى **Print**.

وإنشاء Report Action تشير إلى Template غير موجودة لا ينتج تقريرًا صالحًا.

---

### Records المتاحة داخل التقرير

في التقرير المعتاد، يوفر Odoo:

| المتغير | المحتوى |
| --- | --- |
| `docs` | Records التي سيعرضها التقرير |
| `doc_ids` | Database IDs الخاصة بها |
| `doc_model` | الاسم التقني للـModel |

مثلًا، عند طباعة جهازين:

```text
doc_model → equipment.asset

doc_ids   → IDs الجهازين

docs      → Recordset تحتوي الجهازين
```

هذه IDs تحدد Records المطلوبة في هذه العملية.

أما ربط Report Action بالـTemplate فيستخدم External ID، كما تعلمنا في [[10 Data Files & External IDs]].

---

### الهيكل المعتاد للتقرير

سنستخدم:

```text
web.html_container
    ↓
التكرار على docs
    ↓
web.external_layout
    ↓
div.page
    ↓
محتوى الجهاز
```

`web.html_container` توفر الغلاف العام للتقرير.

`web.external_layout` تضيف Layout الشركة، بما يشمل عناصر مثل الترويسة والتذييل.

أما `<div class="page">` فتحتوي جسم المستند.

هذا لا يعني ضمان أن كل محتوى داخل `page` سيتسع في ورقة واحدة؛ طول البيانات والتنسيق قد يؤديان إلى امتداده على عدة صفحات.

---

### تعريف Report Action

نعرّف التقرير باستخدام Model:

```text
ir.actions.report
```

أهم الحقول:

| الحقل | الغرض |
| --- | --- |
| `name` | اسم التقرير الظاهر للمستخدم |
| `model` | Model التي يعالج التقرير Records منها |
| `report_type` | نوع الإخراج، مثل `qweb-pdf` |
| `report_name` | External ID الخاصة بالـQWeb Template |
| `binding_model_id` | ربط التقرير بقائمة Print الخاصة بـModel |
| `binding_view_types` | تحديد أنواع Views التي يظهر فيها الربط |
| `group_ids` | مجموعات التقرير في Odoo 20 |
| `paperformat_id` | تنسيق الورق عند تحديد تنسيق خاص |

> [!important] اسم حقل المجموعات
> في مصدر Odoo 20، اسم الحقل هو `group_ids`.
>
> قد تجد أمثلة من إصدارات أخرى تستخدم `groups_id`؛ راجع اسم الحقل قبل نقلها.

---

### HTML وPDF

يمكن أن يكون نوع التقرير:

```text
qweb-html
qweb-pdf
```

في الحالتين، تبدأ العملية من QWeb Template.

لكن إنشاء PDF يتطلب أيضًا توفر محرك PDF مناسب وعمله في بيئة Odoo المستخدمة.

لذلك قد ينجح عرض HTML بينما يفشل إخراج PDF بسبب إعدادات المحرك أوالموارد التي يحتاجها.

لا تعتبر نجاح HTML وحده تحققًا نهائيًا من المستند المطبوع.

---

### صلاحيات التقرير والبيانات

ربط التقرير بقائمة **Print** لا يمنح المستخدم صلاحية قراءة الأجهزة.

وكذلك إخفاء فقرة باستخدام `t-if` لا يجعل البيانات سرية.

في المثال، نعرض Records وFields باستخدام صلاحيات المستخدم الحالية، ونحتفظ بقيود الشركات الموجودة على `equipment.asset`.

لا نضيف `sudo()` إلى قراءة الأجهزة لحل مشكلة صلاحيات.

راجع [[11 Security & Multi-Company]] عند تصميم تقارير تحتوي معلومات مقيدة.

---

## Practical Example

سننشئ تقرير **Equipment Sheet** داخل Module الحالية `equipment_training`.

سيعرض:

- اسم الجهاز.
- الرقم التسلسلي إن وجد.
- الحالة.
- تاريخ الشراء إن وجد.
- الشركة.
- الوصف إن وجد.

كل هذه الحقول موجودة في `equipment.asset`؛ لا نحتاج إلى إضافة Business Fields جديدة.

---

### 1. تجهيز الملفات

أنشئ مجلد `report` داخل Module.

الأجزاء المطلوبة:

```text
equipment_training/
├── __manifest__.py
└── report/
    ├── equipment_templates.xml
    └── equipment_reports.xml
```

الملف الأول يحتوي QWeb Template.

والملف الثاني يحتوي Report Action.

هذه ملفات XML؛ لا تحتاج إلى Python Imports أو`__init__.py` داخل مجلد `report` ما دمنا لا نضيف Python إليه.

---

### 2. إضافة Dependency على `web`

افتح `__manifest__.py`.

تأكد أن `depends` تتضمن:

```python
"depends": [
    "base",
    "web",
],
```

نستخدم Templates من Module `web`، مثل:

```text
web.html_container
web.external_layout
```

لذلك نصرح بالاعتماد عليها حتى لو كانت مثبتة بالفعل في Database التدريب.

إذا كانت لديك Dependencies إضافية من تطبيقات سابقة، احتفظ بها.

---

### 3. إنشاء QWeb Template

داخل:

```text
report/equipment_templates.xml
```

اكتب:

```xml
<odoo>
    <template id="report_equipment_document">
        <t t-call="web.html_container">
            <t t-foreach="docs" t-as="o">
                <t t-call="web.external_layout">
                    <t
                        t-set="company"
                        t-value="o.company_id"
                    />

                    <t t-set="layout_document_title">
                        Equipment Sheet
                    </t>

                    <div class="page">
                        <h3>
                            <span t-field="o.name"/>
                        </h3>

                        <table class="table table-bordered">
                            <tbody>
                                <tr t-if="o.serial_number">
                                    <th scope="row">
                                        Serial Number
                                    </th>

                                    <td>
                                        <span
                                            t-field="o.serial_number"
                                        />
                                    </td>
                                </tr>

                                <tr>
                                    <th scope="row">
                                        Status
                                    </th>

                                    <td>
                                        <span t-field="o.state"/>
                                    </td>
                                </tr>

                                <tr t-if="o.purchase_date">
                                    <th scope="row">
                                        Purchase Date
                                    </th>

                                    <td>
                                        <span
                                            t-field="o.purchase_date"
                                        />
                                    </td>
                                </tr>

                                <tr>
                                    <th scope="row">
                                        Company
                                    </th>

                                    <td>
                                        <span
                                            t-field="o.company_id.name"
                                        />
                                    </td>
                                </tr>
                            </tbody>
                        </table>

                        <div t-if="o.description">
                            <h4>Description</h4>

                            <div t-field="o.description"/>
                        </div>
                    </div>
                </t>
            </t>
        </t>
    </template>
</odoo>
```

External ID الكاملة لهذه Template:

```text
equipment_training.report_equipment_document
```

#### لماذا حددنا `company` داخل التكرار؟

نمرر شركة الجهاز الحالي إلى Layout:

```xml
<t t-set="company" t-value="o.company_id"/>
```

بهذا تكون Layout مرتبطة بشركة الجهاز عند طباعة Records من شركات مختلفة مسموح للمستخدم الوصول إليها.

هذا الاختيار يتعلق بعرض التقرير، ولا يغير صلاحيات المستخدم أوالشركات المسموح بها.

#### ما وظيفة `layout_document_title`؟

هذا المتغير تستخدمه Layouts القياسية في Odoo 20 لعرض عنوان المستند.

حددنا له النص:

```text
Equipment Sheet
```

واحتفظنا باسم الجهاز داخل جسم التقرير.

#### لماذا استخدمنا `t-field` للحالة؟

الحقل `state` من نوع Selection.

استخدام `t-field` يسمح بعرض تسمية الحالة، مثل:

```text
Available
```

بدل عرض القيمة التقنية مباشرةً:

```text
available
```

ويظل العرض مرتبطًا باللغة والتنسيق المتاحين في سياق التقرير.

---

### 4. إنشاء Report Action

داخل:

```text
report/equipment_reports.xml
```

اكتب:

```xml
<odoo>
    <record
        id="action_report_equipment"
        model="ir.actions.report"
    >
        <field name="name">Equipment Sheet</field>

        <field name="model">equipment.asset</field>

        <field name="report_type">qweb-pdf</field>

        <field name="report_name">
            equipment_training.report_equipment_document
        </field>

        <field
            name="binding_model_id"
            ref="equipment_training.model_equipment_asset"
        />

        <field name="binding_view_types">
            list,form
        </field>

        <field
            name="group_ids"
            eval="[
                (4, ref('equipment_training.group_equipment_user'))
            ]"
        />
    </record>
</odoo>
```

هنا يوجد External ID للتقرير نفسه:

```text
equipment_training.action_report_equipment
```

وExternal ID مختلفة للـTemplate:

```text
equipment_training.report_equipment_document
```

> [!important] لا تخلط بين المرجعين
> `action_report_equipment` تشير إلى Report Action.
>
> `report_equipment_document` تشير إلى QWeb Template.
>
> قيمة `report_name` يجب أن تشير إلى Template.

#### ربط التقرير بالـModel

المرجع:

```text
equipment_training.model_equipment_asset
```

يشير إلى Record الخاصة بتعريف Model داخل `ir.model`.

ينشئ Odoo هذا التعريف للـModel المسجلة.

استخدمناه في `binding_model_id` لربط التقرير بقائمة **Print** الخاصة بالأجهزة.

ولا نحتاج إلى تحديد `binding_type`؛ Report Action تستخدم القيمة `report` افتراضيًا.

#### المجموعة المستخدمة

حددنا مجموعة Equipment User.

ومجموعة Equipment Manager التي أنشأناها سابقًا تتضمنها، لذلك يمكن للمدير الوصول إلى التقرير من الواجهة أيضًا.

هذا لا يوسع صلاحيات قراءة الأجهزة نفسها.

---

### 5. تسجيل الملفات في Manifest

أضف ملفي التقرير إلى قائمة `data` الحالية:

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
    "report/equipment_templates.xml",
    "report/equipment_reports.xml",
],
```

احتفظ بملفات الشابترات السابقة.

نحمّل Template ثم Report Action التي تشير إليها.

لا نحتاج إلى إضافة صلاحيات Business Model جديدة؛ التقرير يستخدم `equipment.asset` الموجودة بالفعل.

---

### 6. تطبيق Upgrade

بعد حفظ الملفات:

1. أعد تشغيل Server التدريب.
2. سجل الدخول بحساب يستطيع إدارة Modules.
3. افتح **Apps**.
4. احذف Filter **Apps**.
5. ابحث عن `Equipment Training`.
6. نفّذ **Upgrade**.

أضفنا Dependency وXML Data جديدة، لذلك يجب تطبيق Upgrade على Module.

يمكن مراجعة [[03 Modules & Addons]] لتنفيذ العملية باستخدام CLI عند الحاجة.

---

### 7. طباعة جهاز واحد

سجل الدخول بحساب تدريب يحمل مجموعة Equipment User أوEquipment Manager.

ثم:

1. افتح **Equipment → Assets**.
2. افتح جهازًا محفوظًا.
3. افتح قائمة **Print**، وقد تكون ضمن قائمة الإجراءات بحسب عرض الواجهة.
4. اختر **Equipment Sheet**.

الناتج المتوقع عند جاهزية محرك PDF:

- إنشاء مستند PDF للجهاز.
- ظهور بياناته الحالية.
- ظهور Layout الشركة.
- غياب صف الرقم التسلسلي إذا كانت قيمته فارغة.
- غياب قسم الوصف إذا لم يوجد وصف.

إذا ظهرت نافذة إعداد Layout الشركة، أكمل إعدادها ثم أعد الطباعة.

> [!note] إذا ظهر HTML أوخطأ متعلق بمحرك PDF
> افحص إعدادات محرك PDF في بيئتك ورسالة الخطأ وسجل Server.
>
> اختيار `qweb-pdf` داخل Report Action يحدد نوع التقرير، لكنه لا يثبت أو يصلح محرك PDF تلقائيًا.

---

### 8. طباعة عدة أجهزة

من List الأجهزة:

1. حدد جهازين أوأكثر.
2. افتح **Print**.
3. اختر **Equipment Sheet**.

تمر Template على `docs`، وتنشئ محتوى مستقلًا لكل جهاز داخل المستند الناتج.

تحقق من ظهور جميع الأجهزة المحددة، ومن فواصل الصفحات والشركة المعروضة لكل منها.

اختبار Record واحدة لا يكفي للتحقق من سلوك التقرير مع مجموعة Records.

---

### 9. معاينة HTML عند التشخيص

يمكن فتح HTML الخاصة بالتقرير من المتصفح أثناء تسجيل الدخول إلى Odoo:

```text
http://127.0.0.1:8069/report/html/equipment_training.report_equipment_document/RECORD_ID
```

استبدل `RECORD_ID` بالـDatabase ID لجهاز موجود تستطيع قراءته.

يمكن معرفة ID من أدوات Developer Mode أوOdoo Shell.

هذا الرقم يحدد جهاز التجربة في الرابط؛ لم نضع Database ID ثابتة داخل Module.

تساعد المعاينة على التمييز بين:

- خطأ داخل Template أوالبيانات.
- مشكلة تحدث أثناء إنشاء PDF.

إذا نجح HTML وفشل PDF، افحص محرك PDF والوصول إلى CSS والصور والخطوط، مع متابعة سجل Server.

---

### 10. التحقق من النتيجة

جرّب الحالات التالية:

| الحالة | ما تتحقق منه |
| --- | --- |
| جهاز ببيانات مكتملة | ظهور القيم الصحيحة |
| وصف فارغ | عدم ظهور قسم وصف فارغ |
| رقم تسلسلي فارغ | عدم ظهور صف الرقم التسلسلي |
| عدة أجهزة | ظهور جميع Records المحددة |
| اسم أووصف طويل | التفاف النص وعدم قصه |
| نص عربي | اتصال الحروف واتجاه العرض في PDF |
| أجهزة من شركات مسموحة مختلفة | Layout الشركة الصحيحة لكل جهاز |
| جهاز غير مسموح للمستخدم بقراءته | عدم تجاوز صلاحيات البيانات |

واختبر إخراج PDF نفسه، حتى لو كانت معاينة HTML سليمة.

---

## Extending Reports

### تشغيل التقرير من Python

استخدمنا قائمة **Print** في التطبيق الأساسي.

إذا احتجت لاحقًا إلى تشغيل التقرير من Button أوWizard، يمكنك الحصول على Report Action باستخدام `env.ref()` ثم استدعاء `report_action()`.

داخل Class الحالية لـ`equipment.asset`، يمكن إضافة:

```python
def action_print_equipment(self):
    self.check_access("read")

    report = self.env.ref(
        "equipment_training.action_report_equipment"
    )

    return report.report_action(self)
```

هذه Method تعيد Action يفسرها Web Client؛ لا تعيد PDF Bytes مباشرةً.

ولربطها بزر، أضف داخل Header الموجودة في Form الجهاز:

```xml
<button
    name="action_print_equipment"
    type="object"
    string="Print Equipment"
    groups="equipment_training.group_equipment_user"
/>
```

هذه توسعة اختيارية؛ لا تحتاج إليها لتشغيل التقرير من قائمة **Print**.

لم نستخدم `ensure_one()` هنا لأن Template تدعم عدة Records.

---

### متى نحتاج إلى Custom Report Model؟

التقرير الحالي يعتمد على Fields موجودة داخل `docs`، ولذلك لا يحتاج إلى Python Report Model إضافية.

نحتاج إليها عندما نريد تجهيز بيانات خاصة بالعرض، مثل:

- تجميع بيانات من Models مختلفة.
- بناء ملخصات لا تمثلها Fields الحالية.
- معالجة خيارات قادمة من Wizard.
- تجهيز صفوف منظمة لتقرير تحليلي.

يبحث Odoo عن Model اسمها يتبع النمط:

```text
report.<report_name>
```

وبالنسبة إلى تقريرنا، سيكون الاسم:

```text
report.equipment_training.report_equipment_document
```

تُعرّف عادةً باستخدام `models.AbstractModel`، وتوفر Method:

```python
def _get_report_values(self, docids, data=None):
    ...
```

تعيد هذه Method Dictionary بالقيم التي ستستخدمها Template.

> [!warning] القيم الافتراضية عند تخصيص التقرير
> عند استخدام Custom Report Model، لا تفترض إضافة `docs` و`doc_ids` و`doc_model` تلقائيًا.
>
> أعد هذه القيم صراحةً إذا كانت Template تحتاج إليها.

احتفظ بعمليات البحث والحسابات المعقدة داخل Python بدل توزيعها بين عناصر Template.

ولا تستخدم إعداد التقرير لتنفيذ تغييرات على Business Records؛ مهمته تجهيز المستند.

---

### تخصيص Paper Format

يمكن تعريف Record من:

```text
report.paperformat
```

لتحديد إعدادات مثل:

- حجم الورق.
- الاتجاه: Portrait أوLandscape.
- الهوامش.
- المسافة الخاصة بالترويسة.

ثم ربطها بالتقرير عبر `paperformat_id`.

إذا لم نحدد تنسيقًا خاصًا، يستخدم التقرير تنسيق الشركة وفق آلية التقارير.

في المثال الحالي، نكتفي بالتنسيق الموجود.

عند تخصيص الهوامش، اختبر الترويسة والتذييل مع البيانات الطويلة؛ تقليل الهامش قد يؤدي إلى تداخل المحتوى.

---

### العربية وRTL والخطوط

لعرض تقرير عربي بصورة صحيحة، نحتاج إلى فحص عدة أجزاء معًا:

- النصوص المترجمة داخل Template.
- لغة التقرير.
- اتجاه العرض.
- الخطوط المتاحة للتقرير.
- نتيجة محرك PDF.

تغيير الاتجاه إلى RTL لا يترجم النصوص الإنجليزية تلقائيًا.

كما أن نجاح الخط داخل Form لا يضمن توفره داخل Report؛ التقارير تستخدم Assets مخصصة، مثل:

```text
web.report_assets_common
```

سنتناول الترجمة وإدارة اللغات في [[32 Translations & Localization]].

---

### التقارير الحالية والمستندات التاريخية

يعرض تقريرنا القيم الموجودة وقت الطباعة.

إذا تغير اسم الجهاز أووصفه ثم أعدنا الطباعة، فالتقرير الجديد يعكس البيانات الحالية.

هذا مختلف عن حفظ نسخة تاريخية معتمدة من المستند.

توجد إعدادات للتعامل مع Attachments وإعادة استخدام مستند محفوظ، لكنها تحتاج إلى تصميم واضح للصلاحيات وسلوك إعادة الطباعة.

لم نفعّل حفظ Attachments أوإعادة استخدامها في هذا المثال.

---

## Common Problems

### التقرير لا يظهر في Print

تحقق من:

1. تنفيذ Upgrade على Module.
2. وجود ملف Report Action داخل Manifest.
3. صحة `binding_model_id`.
4. توافق `binding_view_types` مع View الحالية.
5. عضوية المستخدم في المجموعة المحددة.
6. تحديد Records عند العمل من List.

إنشاء Template وحدها لا يضيف خيار الطباعة.

---

### Template غير موجودة

راجع قيمة:

```text
report_name
```

يجب أن تطابق External ID الكاملة للـTemplate:

```text
equipment_training.report_equipment_document
```

ثم تحقق من تحميل ملف XML الذي يعرّفها.

لا تستخدم External ID الخاصة بالـReport Action مكانها.

---

### خطأ عند قراءة Field

إذا استخدمت داخل Template:

```xml
<span t-field="o.employee_id"/>
```

بينما `equipment.asset` لا تحتوي `employee_id`، فسيفشل التقرير.

راجع Model الفعلية واسم Field ومحتوى `docs`.

ولا تنقل أمثلة من تقرير Model أخرى دون تعديلها.

---

### XML غير صالح

لأن Template ملف XML، يجب إغلاق العناصر وكتابة الرموز المحجوزة بصورة صحيحة.

مثلًا، داخل XML نكتب:

```xml
<p>Equipment &amp; Accessories</p>
```

ولا نكتب علامة `&` منفردة داخل النص.

راجع أول خطأ واضح في سجل تحميل Module قبل معالجة الأخطاء التابعة له.

---

### PDF يختلف عن HTML

قد يظهر الاختلاف في:

- الخطوط.
- التفاف النص.
- الصور.
- فواصل الصفحات.
- الهوامش.
- الترويسة والتذييل.

افحص PDF الفعلي، وتأكد أن الموارد المطلوبة متاحة أثناء إنشائه.

لا تفترض أن كل CSS تعمل بالطريقة نفسها في كل محرك PDF.

---

## Important Notes

- Template تحدد المحتوى، وReport Action تربطها بالـModel وطريقة التشغيل.
- `report_name` تشير إلى External ID للـTemplate، وليست External ID للـAction.
- `docs` قد تحتوي عدة Records؛ صمم التقرير واختبره على هذا الأساس.
- `t-field` تعرض Field من Odoo Record مع تنسيقها، بينما `t-out` تعرض Expression عامة.
- `t-if` تتحكم في العرض ولا تعوض صلاحيات البيانات.
- في Odoo 20، راجع استخدام `group_ids` عند تحديد مجموعات Report Action.
- أضف `web` إلى Dependencies عند استخدام Layouts الخاصة بها.
- لا تستخدم `sudo()` لتجاوز مشكلة صلاحيات داخل التقرير.
- Custom `_get_report_values()` تحتاج إلى إعادة القيم التي تعتمد عليها Template صراحةً.
- نجاح HTML لا يضمن نجاح PDF؛ اختبر الخطوط والعربية والصفحات داخل المستند النهائي.
- التقرير المبني على البيانات الحالية لا يمثل نسخة تاريخية محفوظة تلقائيًا.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[21 Mail, Chatter & Activities]]

---

## Sources

- [Odoo 20 Documentation — QWeb Templates](https://www.odoo.com/documentation/20.0/developer/reference/frontend/qweb.html)
- [Odoo 20 Documentation — QWeb Reports](https://www.odoo.com/documentation/20.0/developer/reference/backend/reports.html)
- [Odoo 20 Documentation — Report Actions](https://www.odoo.com/documentation/20.0/developer/reference/backend/actions.html#report-actions-ir-actions-report)
- [Odoo 20 — Report Actions: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_actions_report.py)
- [Odoo 20 — Report Layouts: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/web/views/report_templates.xml)
- [Odoo 20 — QWeb Field Formatting: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_qweb_fields.py)
- [Odoo 20 — Report Routes: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/web/controllers/report.py)