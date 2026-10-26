---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 14 Widgets & View Attributes

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Widgets|Widgets]]
- [[#Expressions & Field Availability|Expressions & Field Availability]]
- [[#Form View Attributes|Form View Attributes]]
- [[#Relational Field Attributes|Relational Field Attributes]]
- [[#List View Attributes|List View Attributes]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في الشابترين السابقين، أنشأنا Views للأجهزة وربطناها بـAction وMenus، ثم أضفنا Button لتنفيذ عملية من الواجهة.

الآن سنحسن طريقة عرض Fields والتفاعل معها.

مثلًا، نريد:

- عرض حالة الجهاز كشريط حالات.
- جعل الرقم التسلسلي غير قابل للتحرير بعد تقاعد الجهاز.
- إخفاء زر Retire عندما تصبح الحالة **Retired**.
- السماح للمستخدم بإظهار بعض أعمدة القائمة أو إخفائها.

نستخدم لهذا الغرض **Widgets** و**View Attributes**.

| الأداة | وظيفتها |
| --- | --- |
| Widget | تحديد طريقة عرض Field أو التفاعل معها |
| View Attribute | ضبط خصائص العنصر داخل View |

هذه الأدوات تنظم تجربة المستخدم، بينما تبقى صلاحيات الوصول وقواعد العمل على Server مسؤولة عن حماية البيانات وفرض شروط العمليات.

---

## Widgets

### ما هي Widget؟

كل Field لها طريقة عرض افتراضية تناسب نوعها.

مثلًا، Date Field تُعرض بأداة لاختيار التاريخ، وBoolean Field تُعرض عادةً كخيار يمكن تفعيله أو تعطيله.

باستخدام `widget`، نستطيع اختيار طريقة عرض أخرى مدعومة لنوع Field.

مثال:

```xml
<field name="state" widget="statusbar"/>
```

Field `state` تبقى Selection في Model، لكن الواجهة تعرضها كشريط حالات بدل أداة الاختيار المعتادة.

Widget لا تغير نوع Field في Python أو طريقة تخزين قيمتها.

---

### Widgets شائعة

| Widget | أنواع Fields مناسبة لها | الاستخدام |
| --- | --- | --- |
| `phone` | Char | عرض رقم الهاتف بطريقة مناسبة للاتصال |
| `email` | Char | عرض البريد كرابط بريد في وضع القراءة |
| `many2many_tags` | Many2many | عرض Records المرتبطة كوسوم |
| `statusbar` | Selection أو Many2one | عرض الحالة أو المرحلة |
| `monetary` | Monetary أو Float | عرض قيمة بتنسيق نقدي |
| `image` | Binary | عرض محتوى صورة |

اختيار Widget يعتمد على نوع Field وما تدعمه Widget نفسها.

مثلًا، كتابة `widget="monetary"` لا تنشئ Currency Field أو تحول Char إلى قيمة مالية.

نراجع متطلبات Widget قبل استخدامها، خصوصًا إذا كانت تعتمد على Fields إضافية.

---

### Statusbar

في مشروع التدريب، قيم `state` هي:

```text
available
in_use
retired
```

يمكن عرضها في Header:

```xml
<header>
    <field
        name="state"
        widget="statusbar"
        statusbar_visible="available,in_use,retired"
        readonly="True"
    />
</header>
```

هنا:

- `widget="statusbar"` تختار طريقة العرض.
- `statusbar_visible` تحدد الحالات التي نريد إظهارها.
- `readonly="True"` تجعل العنصر للعرض دون تحرير مباشر من خلاله.

استخدمنا القيم التقنية المخزنة داخل `statusbar_visible`، بينما تعرض الواجهة Labels المقابلة لها.

> [!note] شريط الحالات لا ينشئ Workflow
> Statusbar تعرض الحالة، لكنها لا تعرف وحدها الانتقالات المسموحة أو العمليات المطلوبة عند الانتقال.
>
> كما أن تغيير الحالة مباشرةً من Widget قابلة للتحرير لا يعني استدعاء Method مثل `action_retire`.

---

## Expressions & Field Availability

### Expressions مباشرة في Odoo 20

خصائص مثل `readonly` و`required` و`invisible` تقبل Python Expressions تُقيّم داخل الواجهة.

مثلًا:

```xml
<field
    name="serial_number"
    readonly="state == 'retired'"
/>
```

إذا كانت حالة الجهاز `retired`، تكون نتيجة Expression هي `True`، فتُفعّل خاصية `readonly`.

| النتيجة | أثرها |
| --- | --- |
| `True` | تفعيل الخاصية |
| `False` | عدم تفعيلها بهذا الشرط |

يمكن استخدام قيمة ثابتة أيضًا:

```xml
<field name="serial_number" readonly="True"/>
```

---

### Field المستخدمة في Expression

إذا كان شرط العنصر يعتمد على Field من Record الحالية، فيجب أن تكون تلك Field موجودة في View.

مثلًا، هذا الشرط يعتمد على `state`:

```xml
<field
    name="serial_number"
    readonly="state == 'retired'"
/>
```

لذلك يجب تضمين `state` في View نفسها.

يمكن عرضها بشكل طبيعي، أو تضمينها مخفية عند عدم الحاجة إلى إظهارها:

```xml
<field name="state" invisible="True"/>
```

في التطبيق، ستكون `state` موجودة في Header كـStatusbar، ولذلك لا نحتاج إلى إضافتها مرة أخرى كعنصر مخفي.

---

### سياق التقييم

من القيم المتاحة في Expressions، حسب موضع العنصر:

| القيمة | معناها |
| --- | --- |
| أسماء Fields الموجودة في View | قيم Record الحالية |
| `context` | Context المتاحة للواجهة |
| `uid` | ID المستخدم الحالي |
| `parent` | Record الأب داخل Subview لعلاقة، عند وجود هذا السياق |

لا تتعامل مع هذه Expressions على أنها Python Method داخل Server.

مثلًا، لا نكتب `self.env` داخل `readonly`.

كذلك، تختلف بيئة تقييم View Expressions عن بيئة Domain الخاصة بتعريفات الوصول؛ لا تفترض توفر المتغيرات نفسها في المكانين.

---

### الصيغ القديمة

قد تجد أمثلة قديمة تستخدم `attrs` أو`states` للتحكم في عناصر View.

في Odoo 20، نكتب Conditions مباشرةً على Attributes المناسبة:

```xml
<field
    name="serial_number"
    readonly="state == 'retired'"
/>
```

لا تنقل صيغة إصدار قديم إلى المشروع دون مراجعتها.

---

## Form View Attributes

### `readonly`

تمنع تحرير العنصر من خلال View عندما يتحقق الشرط.

مثال:

```xml
<field
    name="serial_number"
    readonly="state == 'retired'"
/>
```

في هذه Form، يستطيع المستخدم الذي يملك صلاحية التعديل تحرير الرقم التسلسلي قبل التقاعد، ويصبح العنصر للقراءة بعده.

لكن هذا الشرط لا يمنع Python Code أو مسارًا آخر لديه صلاحية الكتابة من تعديل القيمة.

إذا كان منع التعديل بعد التقاعد قاعدة عمل، فنحتاج إلى فرضها على Server أيضًا.

سنشرح تنظيم هذا النوع من القواعد في [[15 Business Logic]].

---

### `invisible`

تخفي العنصر عندما يتحقق الشرط.

مثلًا، نخفي زر Retire بعد وصول الجهاز إلى حالة التقاعد:

```xml
<button
    name="action_retire"
    type="object"
    string="Retire"
    invisible="state == 'retired'"
/>
```

يمكن تطبيق `invisible` على Field أوButton أوGroup وعناصر أخرى تدعمها.

إخفاء العنصر:

- لا يمسح قيمته.
- لا يغير حالة Record.
- لا يجعل البيانات سرية.

لمنع الوصول إلى بيانات حساسة، نستخدم صلاحيات Server المناسبة، كما تعلمنا في [[11 Security & Multi-Company]].

---

### `required`

تجعل إدخال قيمة مطلوبًا من خلال View عندما يتحقق الشرط.

مثال تدريبي:

```xml
<field
    name="serial_number"
    required="state == 'available'"
/>
```

هنا تطلب Form رقمًا تسلسليًا عندما تكون الحالة `available`.

هذا يختلف عن تعريف Field في Python:

```python
serial_number = fields.Char(required=True)
```

| المكان | المعنى |
| --- | --- |
| `required` داخل View | متطلب إدخال في هذه الواجهة |
| `required=True` داخل Model | تعريف Field كمطلوبة على مستوى Model |

إذا كانت القيمة مطلوبة دائمًا، نعبّر عن ذلك في تعريف Model.

أما إذا كانت مطلوبة وفق قاعدة عمل شرطية، فـView وحدها لا تكفي؛ نحتاج إلى تحقق مناسب على Server.

تفاصيل Constraints تأتي في [[17 Constraints]].

> [!important] تجنب الشروط المتعارضة
> لا تجعل Field فارغة مطلوبة وغير قابلة للتحرير في الوقت نفسه، أو مطلوبة ومخفية دون طريقة لتوفير قيمتها.
>
> راجع تفاعل الشروط معًا، وليس كل Attribute منفردة.

---

### `groups`

تحدد المجموعات التي يظهر لها العنصر.

مثال توضيحي باستخدام Field موجودة في مشروعنا:

```xml
<field
    name="description"
    groups="equipment_training.group_equipment_manager"
/>
```

هذه الصيغة تقيد ظهور العنصر في View بأعضاء Group المدير.

لكنها لا تضيف تقييدًا إلى Field داخل Model.

| المكان | الأثر |
| --- | --- |
| `groups` على عنصر داخل XML View | تقييد ظهور العنصر |
| `groups` على تعريف Field في Python | تقييد الوصول إلى Field على Server |

لن نطبق هذا التغيير على `description` في المثال العملي؛ سنبقي الوصف متاحًا وفق صلاحيات Model الحالية.

---

## Relational Field Attributes

### `domain`

على Relational Field، تضيق Domain الخيارات التي تعرضها الواجهة للاختيار.

مثلًا، داخل Form للـModel `equipment.loan`، يمكن تحديد الأجهزة المتاحة التابعة إلى شركة الطلب:

```xml
<field name="company_id"/>

<field
    name="asset_id"
    domain="[
        ('state', '=', 'available'),
        ('company_id', '=', company_id)
    ]"
/>
```

في هذا المثال:

- `state` داخل Domain تخص Model الأجهزة التي نبحث فيها.
- `company_id` في يمين الشرط هي قيمة شركة طلب الإعارة الحالي.
- Field `company_id` موجودة في View لتكون متاحة للتقييم.

هذا مقطع يخص Form الإعارة، وليس Form الأجهزة.

Domain تساعد المستخدم على الاختيار، لكنها لا تضمن بقاء الجهاز متاحًا، ولا تستبدل التحقق من صحة الربط على Server.

راجع [[08 Domains]] و[[11 Security & Multi-Company]] للتفريق بين تصفية الخيارات والتحقق من البيانات.

---

### `context`

يمكن تمرير Context إلى عمليات مرتبطة بالـRelational Field.

مثلًا، على `asset_id` داخل Form الإعارة:

```xml
<field
    name="asset_id"
    context="{'default_company_id': company_id}"
/>
```

إذا كان إنشاء جهاز من خلال هذا العنصر متاحًا، تُمرر شركة الطلب كقيمة افتراضية لشركة الجهاز الجديد.

هذه Context:

- لا تغير شركة جهاز موجود.
- لا تصفي الأجهزة تلقائيًا حسب الشركة.
- لا تمنح صلاحية إنشاء جهاز.

استخدم Domain للتصفية، وContext للقيم والسلوكيات التي تدعمها العملية المستقبلة.

---

### `options`

تضبط خيارات تدعمها Widget الخاصة بالعنصر.

مثلًا، لمنع إنشاء أجهزة من قائمة اختيار `asset_id`:

```xml
<field
    name="asset_id"
    options="{'no_create': True}"
/>
```

من خيارات Many2one:

| Option | أثرها في العنصر |
| --- | --- |
| `no_create` | تعطيل الإنشاء من قائمة الاختيار |
| `no_quick_create` | تعطيل الإنشاء السريع من النص المدخل |
| `no_create_edit` | تعطيل خيار الإنشاء باستخدام Form |
| `no_open` | تعطيل فتح Record المرتبطة من العنصر |

تعطيل الإنشاء السريع وحده لا يعني تعطيل جميع طرق الإنشاء من العنصر.

كذلك، `no_create` لا تسحب صلاحية Create من المستخدم على Model؛ يستطيع استخدام مسار آخر يسمح له بالإنشاء.

> [!note] Options تختلف بين Widgets
> لا تفترض أن Option تعمل على كل Field.
>
> اكتب القيم المنطقية داخل Dictionary بصيغة Python، مثل `True` و`False`، وراجع Widget والإصدار.

---

## List View Attributes

### `optional`

تسمح للمستخدم بإظهار عمود أو إخفائه من خيارات الأعمدة.

مثال:

```xml
<list>
    <field name="name"/>
    <field name="serial_number" optional="show"/>
    <field name="purchase_date" optional="hide"/>
    <field name="state"/>
</list>
```

| القيمة | الوضع الافتراضي |
| --- | --- |
| `show` | العمود ظاهر |
| `hide` | العمود مخفي |

المستخدم يستطيع تغيير هذا الاختيار من الواجهة.

`optional="hide"` لا تعني أن قيمة Field فارغة أو أن المستخدم ممنوع من قراءتها.

---

### `column_invisible`

تخفي العمود كاملًا وفق شرط ينطبق على العمود.

مثال يعتمد على Context:

```xml
<field
    name="serial_number"
    column_invisible="context.get('hide_serial_number', False)"
/>
```

عندما تحتوي Context على:

```python
{"hide_serial_number": True}
```

يُخفى العمود كاملًا.

يمكن تمرير هذا المفتاح من Context الخاصة بالـAction، كما تعلمنا في [[13 Actions, Menus & Buttons]].

لا تعتمد في `column_invisible` على قيمة `state` الخاصة بكل صف؛ الشرط يُقيّم للعمود كله، دون قيم سجلات الصفوف.

وفي List داخل Relational Field، يمكن أن يكون `parent` مناسبًا لشرط يعتمد على Record الأب.

---

### الفرق بين خصائص الإخفاء

| الخاصية | الاستخدام |
| --- | --- |
| `invisible` | إخفاء عنصر وفق شرط؛ داخل List لا تعني وحدها حذف العمود كله |
| `column_invisible` | إخفاء العمود كاملًا |
| `optional` | السماح للمستخدم بإظهار العمود أو إخفائه |

إذا أخفى شرط `column_invisible` العمود، فلا تعتمد على `optional` لإظهاره مع بقاء الشرط متحققًا.

---

## Example

سنحسن واجهات الأجهزة التي بنيناها في الشابترين السابقين.

سنطبق السلوك التالي:

| العنصر | السلوك |
| --- | --- |
| `state` | عرض كـStatusbar للقراءة |
| زر Retire | إخفاؤه عندما تكون الحالة `retired` |
| `serial_number` في Form | مطلوب عند `available`، وللقراءة عند `retired` |
| `company_id` في Form | تعطيل إنشاء شركة من عنصر الاختيار |
| الرقم التسلسلي في List | عمود اختياري ظاهر افتراضيًا |
| تاريخ الشراء في List | عمود اختياري مخفي افتراضيًا |

شرط طلب الرقم التسلسلي هنا تجربة لخاصية `required` في الواجهة؛ لا نضيف في هذا الشابتر قاعدة Server مقابلة له.

---

### 1. تعديل ملف Views

افتح الملف:

```text
custom-addons/equipment_training/views/equipment_views.xml
```

في مشروع التدريب الحالي، استبدل محتواه بالتالي.

إذا أضفت تعريفات أخرى إلى الملف، احتفظ بها أيضًا.

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
                <field
                    name="serial_number"
                    optional="show"
                />
                <field
                    name="purchase_date"
                    optional="hide"
                />
                <field name="state"/>
                <field
                    name="company_id"
                    optional="show"
                />
            </list>
        </field>
    </record>

    <!-- Form View -->
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
                        invisible="state == 'retired'"
                    />

                    <field
                        name="state"
                        widget="statusbar"
                        statusbar_visible="available,in_use,retired"
                        readonly="True"
                    />
                </header>

                <sheet>
                    <group>
                        <group string="Equipment">
                            <field name="name"/>

                            <field
                                name="serial_number"
                                required="state == 'available'"
                                readonly="state == 'retired'"
                            />

                            <field name="purchase_date"/>
                        </group>

                        <group string="Company &amp; Availability">
                            <field
                                name="company_id"
                                options="{'no_create': True}"
                            />

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

استخدمنا External IDs نفسها، ولذلك سيحدث Upgrade تعريفات Views الحالية.

كما أبقينا Search View حتى لا نفقد خيارات البحث التي أنشأناها سابقًا.

---

### 2. فهم شروط الرقم التسلسلي

يتغير سلوك العنصر حسب الحالة:

| الحالة | مطلوب في Form؟ | قابل للتحرير لمن يملك صلاحية التعديل؟ |
| --- | --- | --- |
| `available` | نعم | نعم |
| `in_use` | لا بهذا الشرط | نعم |
| `retired` | لا بهذا الشرط | لا |

Field `state` موجودة في Header، ولذلك تستطيع Expressions الاعتماد عليها.

وضعها كـStatusbar لا يمنع استخدام قيمتها في الشروط.

لم نغيّر تعريف `serial_number` داخل Python.

---

### 3. تطبيق Upgrade

احفظ ملف XML، ثم:

1. افتح Database التدريب.
2. افتح **Apps**.
3. احذف Filter **Apps** عند الحاجة.
4. ابحث عن `Equipment Training`.
5. اختر **Upgrade**.
6. بعد اكتمال العملية، أعد فتح واجهة الأجهزة.

الملف مسجل مسبقًا في Manifest من الشابتر 12؛ لا تضفه مرة ثانية.

في هذا التطبيق عدّلنا XML فقط، ولذلك لا نحتاج إلى إعادة تشغيل Server لتحميل Python جديدة.

---

### 4. تجربة Form

استخدم حساب Equipment Manager لديه صلاحية العمل على الشركة المختارة.

1. افتح **Equipment → Assets**.
2. أنشئ جهازًا باسم `Training Monitor`.
3. اترك الرقم التسلسلي فارغًا وحاول الحفظ.
4. تحقق من أن Form تطلب الرقم لأن الحالة الافتراضية `available`.
5. أدخل رقمًا تجريبيًا مثل `DEMO-014`.
6. احفظ الجهاز.
7. اضغط **Retire**.

بعد تنفيذ العملية، تحقق من:

- عرض **Retired** كالحالة الحالية في Statusbar.
- اختفاء زر Retire.
- تحول الرقم التسلسلي إلى عنصر للقراءة.
- بقاء الرقم التسلسلي محفوظًا.

Statusbar هنا للعرض، لذلك لا نستخدمها لتغيير الحالة بالنقر المباشر.

---

### 5. تجربة أعمدة List

ارجع إلى قائمة الأجهزة وافتح خيارات الأعمدة.

تحقق من إمكانية:

1. إظهار **Purchase Date**.
2. إخفاء **Serial Number**.
3. إعادة إظهار الرقم التسلسلي.

قد تتأثر حالة الأعمدة باختيارات محفوظة للمستخدم؛ `show` و`hide` تحددان الوضع الافتراضي، وليستا أمرًا يمنع المستخدم من تغييره.

إخفاء العمود لا يحذف قيمته من Record.

---

### 6. التحقق من حدود التغيير

قارن النتيجة بهذه الحدود:

| التغيير | ما يفعله | ما يحتاج تحققًا منفصلًا على Server |
| --- | --- | --- |
| `readonly` للرقم التسلسلي | يمنع تحريره من العنصر | منع تعديله من كل المسارات بعد التقاعد |
| `required` للرقم التسلسلي | تطلبه Form عند `available` | فرض المتطلب عند الإنشاء أو الكتابة من مسارات أخرى |
| إخفاء Retire | يزيل الزر بعد التقاعد | قواعد انتقال الحالات وصلاحية تنفيذ العملية |
| `no_create` للشركة | يعطل الإنشاء من عنصر الاختيار | صلاحية إنشاء شركات من مسارات أخرى |
| `optional` للأعمدة | يغير طريقة العرض | صلاحية قراءة Field نفسها |

Method `action_retire` ما زالت تحتوي فحص Group والكتابة عبر ORM من الشابتر السابق.

شروط الواجهة التي أضفناها لا تستبدل هذا التنفيذ.

---

## Important Notes

- Widget لا تغير Field Type، ويجب أن تكون متوافقة مع نوع Field ومتطلباتها.
- Conditions تستخدم القيم التقنية مثل `retired`، وليس Label مثل `Retired`.
- إخفاء Field لا يمسح قيمتها ولا يجعلها سرية.
- View Expressions لا تتحول تلقائيًا إلى قواعد تحقق على Server.
- `column_invisible` تُقيّم للعمود كله، لذلك تختلف عن شرط يعتمد على بيانات كل صف.
- Domain على علاقة تصفي خيارات الاختيار؛ لا تضمن صحة العلاقة أو استمرار تحقق الشرط بعد الحفظ.
- عند استخدام Groups مع Expressions، تأكد أن Fields التي تعتمد عليها الشروط متاحة للمستخدمين الذين تُطبق عليهم.
- لا تطلب قيمة من المستخدم في عنصر تمنعه الشروط من رؤيته أو تحريره.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[15 Business Logic]]

---

## Sources

- [Odoo 20 Documentation — View Architectures](https://www.odoo.com/documentation/20.0/developer/reference/user_interface/view_architectures.html)
- [Odoo 20 Documentation — JavaScript Reference: Fields](https://www.odoo.com/documentation/20.0/developer/reference/frontend/javascript_reference.html#fields)
- [Odoo 20 Tutorial — Add The Sprinkles](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/11_sprinkles.html)
- [Odoo 20 Documentation — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)
- [Odoo 20 Source — Statusbar Field](https://github.com/odoo/odoo/blob/20.0/addons/web/static/src/views/fields/statusbar/statusbar_field.js)
- [Odoo 20 Source — Many2one Field Options](https://github.com/odoo/odoo/blob/20.0/addons/web/static/src/views/fields/many2one/many2one_field.js)