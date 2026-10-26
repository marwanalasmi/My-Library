---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 22 Automation & Scheduled Actions

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Automation Tools|Automation Tools]]
- [[#Scheduled Action Fundamentals|Scheduled Action Fundamentals]]
- [[#Batching & Transactions|Batching & Transactions]]
- [[#Practical Example|Practical Example]]
- [[#Testing & Troubleshooting|Testing & Troubleshooting]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

في الشابتر السابق، أضفنا زرًا ينشئ Activity لمراجعة الجهاز.

لكن بعض العمليات يجب أن تحدث عند وصول موعد أوتحقق شرط، دون الحاجة إلى ضغط المستخدم زرًا في كل مرة.

مثلًا:

- طلب مراجعة الأجهزة التي وصل موعد مراجعتها.
- معالجة سجلات جاهزة على دفعات.
- مزامنة بيانات في أوقات محددة.
- تنفيذ إجراء عند إنشاء Record أوتعديلها.

توفر Odoo أدوات مختلفة لهذه الاحتياجات.

في هذا الشابتر، سنركز على **Scheduled Actions**، وننشئ مهمة يومية تطلب مراجعة الأجهزة التي وصل موعد طلب مراجعتها.

سنستخدم منطق الجدولة الذي أنشأناه في [[21 Mail, Chatter & Activities]]، مع تحديد:

- السجلات المؤهلة.
- مستخدم التنفيذ.
- نطاق الشركة.
- حجم الدفعة.
- سلوك إعادة التشغيل والفشل.

---

## Automation Tools

### الفرق بين الأدوات الأساسية

| الأداة | Model | متى تُستخدم؟ |
| --- | --- | --- |
| Scheduled Action | `ir.cron` | تنفيذ عمل حسب جدول زمني |
| Server Action | `ir.actions.server` | تعريف إجراء يمكن تشغيله من مسارات مختلفة |
| Automation Rule | `base.automation` | ربط إجراء بحدث أوشرط |
| Sequence | `ir.sequence` | توليد أرقام أوReferences عند طلبها |

هذه الأدوات قد تتعاون، لكنها تؤدي أدوارًا مختلفة.

مثلًا، Scheduled Action تستخدم إجراءً ينفذ Python Code، بينما Automation Rule قد تشغّل إجراءً عند تعديل Record.

أما Sequence فتولد رقمًا عندما يستدعيها Code؛ وجودها وحده لا ينشئ مهمة زمنية.

---

### Server Actions

Server Action تصف إجراءً يمكن أن يعمل على Model أوRecords محددة.

قد تنفذ Code مختصرة تستدعي Method موجودة:

```python
if records:
    records.action_schedule_review()
```

هذا مثال على إجراء يعمل على Records مختارة، وليس تعريفًا لمهمة يومية.

يفضل إبقاء Business Logic داخل Python Methods، واستخدام Code الإجراء كنقطة دخول واضحة.

كذلك، بيئة تنفيذ Code داخل Server Actions لها سياق وقيود خاصة؛ لا تتعامل معها كبديل كامل لملفات Python.

---

### Automation Rules

تسمح Automation Rules بربط إجراء بأحداث، مثل:

- إنشاء Record.
- تعديل حقول محددة.
- الأرشفة.
- الوصول إلى توقيت مرتبط بحقل تاريخ.

توفر Module مثل `base_automation` هذه الإمكانيات، وقد تتأثر واجهة إعدادها بالـModules المثبتة.

عند تصميم Rule تعمل على التعديل، انتبه إلى أن الإجراء نفسه قد يعدّل Record أيضًا، فيؤدي إلى تشغيل قواعد أخرى أوتكرار غير مقصود.

تحتاج إلى تحديد Trigger والحقول والشروط بدقة.

التطبيق العملي هنا يستخدم `ir.cron`، ولا يحتاج إلى إضافة `base_automation`.

---

## Scheduled Action Fundamentals

### ما هي Scheduled Action؟

Scheduled Action هي Record داخل:

```text
ir.cron
```

تحدد للمجدول العمل المطلوب وتكراره ومستخدم تنفيذه.

التسلسل المعتاد:

```text
وصول موعد التنفيذ
    ↓
التقاط المهمة بواسطة Scheduler
    ↓
تشغيل Code بصلاحيات مستخدم المهمة
    ↓
البحث عن Records المؤهلة
    ↓
تنفيذ العمل وتسجيل التقدم
    ↓
تحديد الاستكمال أوالموعد التالي
```

يجب أن تكون خدمة Odoo وعمال Cron قادرين على العمل في البيئة المستخدمة.

وجود Record للمهمة لا يعني أنها ستعمل إذا كان Server متوقفًا أوكانت معالجة Cron معطلة.

---

### أهم حقول `ir.cron`

| الحقل | الغرض |
| --- | --- |
| `name` | اسم المهمة |
| `model_id` | Model التي ستعمل عليها Code |
| `state` | نوع الإجراء، وسنستخدم `code` |
| `code` | نقطة دخول التنفيذ |
| `user_id` | مستخدم التنفيذ |
| `interval_number` | عدد وحدات التكرار |
| `interval_type` | نوع الوحدة، مثل `days` |
| `nextcall` | موعد التنفيذ التالي |
| `active` | تفعيل المهمة أوتعطيلها |
| `priority` | أولوية المعالجة بين المهام |

مثلًا، تشغيل يومي:

```xml
<field name="interval_number">1</field>
<field name="interval_type">days</field>
```

هذا يحدد التكرار، وليس ضمانًا للتنفيذ في ثانية بعينها؛ قد يتأثر البدء بتوفر العمال والمهام الأخرى.

---

### Cron Method

نضع العمل داخل Method واضحة:

```python
@api.model
def _cron_schedule_reviews(self):
    ...
```

ونستدعيها من Code المهمة:

```python
model._cron_schedule_reviews()
```

المتغير `model` يمثل Model المحددة في `model_id`.

لا نفترض أن `self` تحتوي Records مختارة من واجهة المستخدم؛ الدالة تبحث عن السجلات المؤهلة بنفسها.

واستخدام بادئة `_cron_` يساعد على توضيح الغرض من Method.

---

### مستخدم التنفيذ

تنفذ المهمة باستخدام `user_id` الخاصة بها.

لذلك يجب أن يملك هذا المستخدم:

- صلاحيات Models المطلوبة.
- الوصول إلى Records المستهدفة.
- المجموعات التي تتطلبها Business Methods.
- الشركات المناسبة.

قد تنجح العملية عند تنفيذها بحساب المدير من Form، ثم تفشل داخل Cron لأن مستخدم المهمة مختلف.

في التطبيق، سنعرّف حسابًا تقنيًا للتدريب ونربطه بالمهمة صراحةً.

---

### نطاق الشركة

لا تفترض أن Cron ستستخدم الشركات المحددة حاليًا في متصفح الشخص الذي يراجع المهمة.

حدد نطاق المعالجة داخل التصميم.

في مثالنا:

```text
شركة مستخدم التنفيذ الرئيسية
    ↓
أجهزة هذه الشركة فقط
```

ونضبط Context بحيث تحتوي شركة واحدة محددة.

للتوسع إلى عدة شركات، نحتاج إلى سياسة واضحة لاختيارها وإسناد الأنشطة والصلاحيات داخل كل شركة، كما تعلمنا في [[11 Security & Multi-Company]].

---

### إعادة التشغيل وIdempotency

قد تعمل المهمة عدة مرات:

- حسب جدولها.
- بواسطة Run Manually.
- بعد فشل أوتوقف.
- لاستكمال دفعات متبقية.

نريد أن تتعامل مع العمل الذي سبق تنفيذه دون إنشاء نتائج ضارة مكررة.

لكن Boolean عامة مثل:

```text
review_activity_created
```

قد تكون غير كافية.

إذا بقيت قيمتها صحيحة بعد إنجاز المراجعة، فقد تمنع طلب مراجعة جديدة لاحقًا.

الأفضل تعريف حالة العمل المطلوبة ومعنى اكتمال معالجته بصورة واضحة.

---

## Batching & Transactions

### لماذا نستخدم Batches؟

البحث عن جميع السجلات ومعالجتها في استدعاء طويل قد يؤدي إلى:

- استهلاك وقت وذاكرة كبيرين.
- احتجاز Worker فترة طويلة.
- الوصول إلى Timeout.
- خسارة قدر كبير من العمل غير المثبت عند الفشل.

لذلك نعالج عددًا محدودًا في كل دفعة:

```text
البحث عن دفعة صغيرة
    ↓
تنفيذ العمل
    ↓
تحديث حالة السجلات المعالجة
    ↓
تسجيل التقدم
    ↓
العودة إلى Scheduler
```

سنستخدم حدًا قدره `100` جهاز في كل استدعاء.

---

### وظيفة `_commit_progress()`

يمكن لدالة Cron تسجيل تقدمها:

```python
self.env["ir.cron"]._commit_progress(
    processed,
    remaining=remaining,
)
```

هنا:

- `processed` عدد العناصر التي عالجناها في هذه الخطوة.
- `remaining` عدد العناصر التي ما زالت مؤهلة للمعالجة.

هذه Method تثبت Transaction أيضًا.

وفي سياق Cron، تعيد مقدار الوقت المتبقي للتنفيذ بالثواني.

إذا كنت تدير Loop داخل الدالة، ووصل الوقت المتبقي إلى صفر، فيجب العودة سريعًا.

أما مثالنا فيعالج دفعة واحدة ثم يعود، ويترك للمجدول إدارة الاستكمال.

> [!warning] استدعاؤها خارج Cron
> في Odoo 20، استدعاء `_commit_progress()` خارج سياق Cron ينفذ Commit أيضًا.
>
> لذلك لا تضعها داخل Business Method مشتركة تستخدمها Buttons أوWizards.
>
> سنبقيها داخل Method Cron فقط.

---

### يجب أن تتقدم الدفعة

وجود `limit=100` لا يكفي وحده.

إذا بقيت السجلات المعالجة مطابقة لشرط البحث، فقد تعيد كل دفعة اختيار السجلات نفسها:

```text
نفس أول 100 Record
    ↓
نفس المعالجة
    ↓
نفس أول 100 Record
```

يجب أن يخرج العمل المكتمل من مجموعة السجلات المؤهلة، أوتستخدم آلية تقدم مناسبة.

في التطبيق، سنستهلك تاريخ طلب المراجعة بعد تغطية الطلب.

---

### ماذا يحدث عند الفشل؟

سننفذ داخل الدفعة:

```text
إنشاء أوإعادة استخدام مراجعة مفتوحة
    ↓
استهلاك موعد الطلب
    ↓
Commit للدفعة
```

إذا فشلت الدفعة قبل تثبيتها، تتراجع تغييرات Database غير المثبتة.

أما الدفعات السابقة التي تم تثبيتها، فتبقى محفوظة.

لذلك لا نفترض أن تشغيل المهمة كلها يمثل Transaction واحدة.

كذلك، آثار الأنظمة الخارجية، مثل طلب API نُفذ بالفعل، لا تتراجع تلقائيًا مع Rollback لقاعدة البيانات. سنعود لهذا عند [[25 APIs & Integrations]].

---

## Practical Example

سنضيف حقلًا باسم:

```text
review_request_date
```

يمثل **موعد طلب المراجعة**، وليس تاريخ إنجازها.

سيعمل المثال وفق السياسة التالية:

| الحالة | السلوك |
| --- | --- |
| التاريخ فارغ | لا يوجد طلب مجدول |
| التاريخ مستقبلي | ينتظر |
| التاريخ وصل والجهاز `available` | يطلب المراجعة |
| توجد مراجعة مفتوحة لنفس مستخدم التنفيذ | يعتبر الطلب مغطى بها |
| عولج الطلب بنجاح | يُفرغ تاريخ الطلب |
| نحتاج مراجعة أخرى لاحقًا | نحدد تاريخ طلب جديدًا |

سنستخدم `action_schedule_review()` من الشابتر السابق.

وبالتالي يكون موعد Activity بعد يومين من يوم تشغيل الطلب، كما حددنا هناك.

> [!note] نطاق المثال
> هذه مهمة تفحص طلبات المراجعة يوميًا.
>
> لا تنشئ دورة مراجعة دورية تلقائيًا بعد إنجاز كل Activity؛ يمكن للمدير تحديد موعد طلب جديد عندما يحتاج ذلك.

---

### 1. تحديد الملفات

سنستخدم الأجزاء التالية داخل `equipment_training`:

```text
equipment_training/
├── __manifest__.py
├── models/
│   └── equipment_asset.py
├── security/
│   └── equipment_scheduler_user.xml
├── data/
│   └── equipment_cron_data.xml
└── views/
    └── equipment_automation_views.xml
```

نفترض تطبيق الشابتر السابق، بما يشمل:

- Mixins الخاصة بالرسائل والأنشطة.
- نوع `Equipment Review`.
- Method باسم `action_schedule_review()`.
- مجموعة Equipment Manager.

---

### 2. إضافة موعد طلب المراجعة

داخل Class الحالية لـ`equipment.asset` في:

```text
models/equipment_asset.py
```

أضف:

```python
review_request_date = fields.Date(
    string="Review Request Date",
    copy=False,
    index=True,
    help=(
        "Date when the scheduled action should request a review. "
        "Cleared after the request is covered."
    ),
)
```

لا نضع قيمة افتراضية؛ يبقى الجهاز خارج المعالجة حتى يحدد المدير موعدًا له.

واستخدمنا `copy=False` حتى لا ينتقل طلب الجدولة إلى جهاز آخر عند Duplicate.

---

### 3. إنشاء حساب التنفيذ

أنشئ:

```text
security/equipment_scheduler_user.xml
```

واكتب:

```xml
<odoo>
    <data noupdate="1">
        <record
            id="user_equipment_scheduler"
            model="res.users"
        >
            <field name="name">Equipment Scheduler</field>

            <field name="login">
                equipment.scheduler
            </field>

            <field name="tz">UTC</field>

            <field
                name="company_id"
                ref="base.main_company"
            />

            <field
                name="company_ids"
                eval="[
                    (6, 0, [ref('base.main_company')])
                ]"
            />

            <field
                name="group_ids"
                eval="[
                    (6, 0, [
                        ref('equipment_training.group_equipment_manager')
                    ])
                ]"
            />
        </record>
    </data>
</odoo>
```

هذا الملف ينشئ حسابًا تقنيًا للتدريب، بمرجع:

```text
equipment_training.user_equipment_scheduler
```

حددنا شركة واحدة باستخدام External ID الموجودة في Odoo:

```text
base.main_company
```

ومنحناه مجموعة Equipment Manager لأن Method الجدولة السابقة تتطلبها.

لا نمنحه صلاحيات إدارة Settings أوModules.

ولم نضع Password داخل XML؛ تنفيذ Cron لا يحتاج إلى تسجيل دخول تفاعلي بهذا الحساب.

> [!note] التاريخ المستخدم في المثال
> المنطقة الزمنية للحساب التقني هي `UTC`.
>
> تحديد اليوم الحالي والموعد المشتق منه يتم وفق سياق حساب التنفيذ، وقد يختلف عن يوم مستخدم آخر قرب منتصف الليل.

---

### 4. إضافة Method Cron

في `models/equipment_asset.py`، تأكد أن `api` و`fields` موجودتان ضمن Imports الحالية.

أضف داخل Class:

```python
    @api.model
    def _cron_schedule_reviews(self, *, limit=100):
        assert self.env.context.get("cron_id"), (
            "Run this method through Scheduled Actions."
        )

        company = self.env.user.company_id

        asset_model = self.with_company(company).with_context(
            allowed_company_ids=[company.id],
        )

        today = fields.Date.context_today(asset_model)

        domain = [
            ("active", "=", True),
            ("company_id", "=", company.id),
            ("state", "=", "available"),
            ("review_request_date", "!=", False),
            ("review_request_date", "<=", today),
        ]

        assets = asset_model.search(
            domain,
            limit=limit,
            order="id",
        )

        assets.action_schedule_review()

        assets.write({
            "review_request_date": False,
        })

        remaining = asset_model.search_count(domain)

        self.env["ir.cron"]._commit_progress(
            len(assets),
            remaining=remaining,
        )
```

#### نطاق البحث

نختار الأجهزة التي:

- ليست مؤرشفة.
- تخص شركة حساب التنفيذ الرئيسية.
- حالتها `available`.
- لديها تاريخ طلب.
- وصل تاريخ طلبها أوتجاوزه اليوم الحالي.

الأجهزة التي لا تحقق هذه الشروط لا تدخل في الدفعة.

#### إعادة استخدام Business Method

استدعاء:

```python
assets.action_schedule_review()
```

يعيد استخدام منطق الشابتر السابق، بما فيه:

- فحص مجموعة المدير.
- صلاحيات قراءة الأجهزة وتعديلها.
- البحث عن مراجعة مفتوحة لنفس المستخدم.
- إنشاء Activity والملاحظة عند الحاجة.

المستخدم الحالي هنا هو **Equipment Scheduler**.

لذلك تُسند المراجعات المنشأة إليه.

#### استهلاك الطلب

بعد تغطية طلبات الدفعة، نفرغ:

```python
review_request_date
```

هذا يعني أن طلب الجدولة تمت معالجته.

ولا يعني أن المراجعة نفسها أُنجزت.

إذا وُجدت Activity مفتوحة مطابقة، نعتبر الطلب مغطى بها، دون تغيير موعد تلك Activity.

#### تسجيل التقدم

بعد إخراج الأجهزة المعالجة من Domain، نحسب المتبقي ثم نثبت تقدم الدفعة.

لا نحتاج إلى Loop مفتوحة أوتعديل `nextcall` يدويًا داخل Method.

المجدول يدير الاستكمال وفق التقدم المسجل وحدود التنفيذ.

---

### 5. عرض الحقل في Form

أنشئ:

```text
views/equipment_automation_views.xml
```

واكتب:

```xml
<odoo>
    <record
        id="view_equipment_form_automation"
        model="ir.ui.view"
    >
        <field name="name">
            equipment.asset.form.automation
        </field>

        <field name="model">
            equipment.asset
        </field>

        <field
            name="inherit_id"
            ref="equipment_training.view_equipment_form"
        />

        <field name="arch" type="xml">
            <xpath expr="//form/sheet" position="inside">
                <group
                    name="equipment_review_automation"
                    string="Review Scheduling"
                    groups="equipment_training.group_equipment_manager"
                >
                    <field name="review_request_date"/>
                </group>
            </xpath>
        </field>
    </record>
</odoo>
```

هذا القسم يسمح للمدير بتحديد موعد الطلب.

استخدام `groups` هنا يتحكم في عرض القسم، بينما تبقى صلاحيات تعديل الجهاز مطبقة على Server.

---

### 6. تعريف Scheduled Action

أنشئ:

```text
data/equipment_cron_data.xml
```

واكتب:

```xml
<odoo>
    <data noupdate="1">
        <record
            id="ir_cron_equipment_schedule_reviews"
            model="ir.cron"
        >
            <field name="name">
                Equipment: Schedule Due Reviews
            </field>

            <field
                name="model_id"
                ref="equipment_training.model_equipment_asset"
            />

            <field name="state">code</field>

            <field name="code">
                model._cron_schedule_reviews()
            </field>

            <field
                name="user_id"
                ref="equipment_training.user_equipment_scheduler"
            />

            <field name="interval_number">1</field>

            <field name="interval_type">days</field>

            <field name="active" eval="False"/>
        </record>
    </data>
</odoo>
```

المهمة تبدأ غير مفعلة، حتى نختبر نطاقها ونتيجتها أولًا.

وربطنا مستخدم التنفيذ صراحةً، بدل الاعتماد على المستخدم الافتراضي وقت تحميل XML.

---

### 7. فهم أثر `noupdate`

استخدمنا:

```xml
<data noupdate="1">
```

لإنشاء إعدادات الحساب والمهمة دون إعادة تطبيق تعريفها تلقائيًا أثناء كل Upgrade معتادة.

هذا يسمح ببقاء تعديلات الإدارة، مثل:

- تفعيل المهمة.
- تغيير التكرار.
- تعديل مستخدم التنفيذ.
- تعديل إعدادات الحساب.

لكن تعديل XML لاحقًا لا يضمن وصول التعديل إلى Record الموجودة.

مثلًا، إعادة كتابة:

```xml
<field name="active" eval="False"/>
```

لا تعني أن Upgrade ستعطل مهمة فُعلت سابقًا.

راجع [[10 Data Files & External IDs]] و[[31 Upgrades & Data Migration]] عند الحاجة إلى تغيير بيانات محمية بـ`noupdate`.

---

### 8. تحديث Manifest

أضف الملفات الجديدة مع الحفاظ على ترتيب الملفات السابقة:

```python
"data": [
    "security/groups.xml",
    "security/ir.access.csv",
    "security/equipment_scheduler_user.xml",
    "data/equipment_category_data.xml",
    "data/equipment_activity_data.xml",
    "views/equipment_views.xml",
    "views/equipment_mail_views.xml",
    "views/equipment_automation_views.xml",
    "views/equipment_actions.xml",
    "views/equipment_loan_views.xml",
    "wizard/equipment_loan_reschedule_wizard_views.xml",
    "views/equipment_loan_reschedule_views.xml",
    "views/equipment_menus.xml",
    "report/equipment_templates.xml",
    "report/equipment_reports.xml",
    "data/equipment_cron_data.xml",
],
```

المجموعة تُحمّل قبل الحساب الذي يشير إليها، والحساب يُحمّل قبل Cron.

لا نحتاج إلى Dependency إضافية لهذا المثال فوق Dependencies السابقة.

---

### 9. تطبيق Upgrade

بعد حفظ الملفات:

1. أوقف Server التدريب.
2. أعد تشغيله لتحميل Python الحالية.
3. سجل الدخول بحساب يستطيع إدارة Modules.
4. افتح **Apps** واحذف Filter **Apps**.
5. ابحث عن `Equipment Training`.
6. نفّذ **Upgrade**.

بعد اكتماله، تصبح Fields والـViews والحساب والمهمة موجودة داخل Database.

تبقى المهمة غير مفعلة عند إنشائها أول مرة.

---

### 10. تجهيز بيانات التجربة

استخدم حساب Equipment Manager يستطيع الوصول إلى شركة حساب **Equipment Scheduler**.

راجع اسم تلك الشركة من إعدادات المستخدم التقني.

جهز جهازين محفوظين في هذه الشركة:

| الجهاز | الحالة | Review Request Date |
| --- | --- | --- |
| جهاز التجربة الأول | `available` | اليوم حسب حساب التنفيذ |
| جهاز التجربة الثاني | `available` | تاريخ مستقبلي |

اختر Records لا تملك مراجعة مفتوحة مسندة إلى Equipment Scheduler حتى يكون أثر الإنشاء واضحًا.

احفظ البيانات قبل تشغيل المهمة.

---

### 11. تشغيل المهمة يدويًا

باستخدام حساب يملك إدارة الإعدادات التقنية:

1. فعّل Developer Mode.
2. افتح **Settings → Technical → Automation → Scheduled Actions**.
3. ابحث عن **Equipment: Schedule Due Reviews**.
4. استخدم Filter **All** أو**Archived** إذا لم تظهر، لأنها غير مفعلة.
5. افتح المهمة.
6. تحقق من Scheduler User.
7. اضغط **Run Manually**.

هذا الزر يستخدم مسار تنفيذ Cron.

لا تختبر الدالة باستدعائها مباشرةً من Odoo Shell؛ تحتوي على Progress API تثبت Transactions، كما أن الاستدعاء المباشر لا يعيد سياق المجدول كاملًا.

المهمة غير المفعلة يمكن اختبارها من **Run Manually** دون تفعيل التشغيل الدوري أولًا.

---

### 12. التحقق من النتيجة

ارجع إلى الجهاز الأول وحدّث Form.

المتوقع:

- وجود Activity من نوع **Equipment Review**.
- إسنادها إلى **Equipment Scheduler**.
- موعدها بعد يومين من يوم إنشاء الطلب.
- ظهور ملاحظة طلب المراجعة.
- إفراغ `Review Request Date`.

أما الجهاز ذو التاريخ المستقبلي:

- يبقى تاريخ طلبه موجودًا.
- لا تُنشأ له Activity بسبب هذا التشغيل.

شغّل المهمة يدويًا مرة ثانية.

المتوقع ألا تعيد معالجة الجهاز الأول، لأن تاريخ طلبه أصبح فارغًا.

---

### 13. تفعيل التشغيل الدوري

بعد نجاح التجربة:

1. افتح Scheduled Action.
2. فعّل **Active**.
3. راجع **Next Execution Date**.
4. تحقق من التكرار اليومي.
5. احفظ.

إذا أردت فحص التشغيل التلقائي أثناء التدريب، حدد موعدًا قريبًا مناسبًا في **Next Execution Date**، ثم راقب النتيجة وسجل Server.

تشغيل Cron يوميًا لا ينشئ مراجعة يومية لكل جهاز؛ إنشاء الطلب يعتمد على Domain وتاريخ الطلب الموجود على الجهاز.

---

## Testing & Troubleshooting

### حالات ينبغي اختبارها

| الحالة | النتيجة المتوقعة |
| --- | --- |
| جهاز متاح وموعد طلبه اليوم أوفي الماضي | معالجة طلبه |
| تاريخ مستقبلي | يبقى دون معالجة |
| تاريخ فارغ | لا يدخل في المعالجة |
| جهاز `in_use` | ينتظر حتى يطابق شرط الحالة |
| جهاز مؤرشف | لا يدخل في المعالجة |
| جهاز من شركة أخرى | لا يدخل في نطاق المثال |
| مراجعة مفتوحة لنفس مستخدم التنفيذ | تغطي الطلب دون إنشاء مراجعة أخرى |
| إعادة التشغيل بعد استهلاك الطلب | لا تتكرر معالجة الطلب نفسه |
| عدد مؤهل أكبر من حجم الدفعة | استكمال عبر عدة دفعات |
| مستخدم تنفيذ بلا الصلاحيات المطلوبة | فشل واضح بدل تجاوز الصلاحيات |

لا تعتبر نجاح التشغيل بحساب Superuser اختبارًا كافيًا للصلاحيات.

---

### المهمة تعمل دون نتائج

قد تكون المهمة سليمة، لكن لا توجد Records مطابقة.

راجع Domain:

- هل الشركة صحيحة؟
- هل الجهاز `available`؟
- هل `active=True`؟
- هل التاريخ موجود؟
- هل وصل التاريخ حسب سياق حساب التنفيذ؟
- هل المستخدم يستطيع قراءة الجهاز؟

`remaining=0` تعني عدم وجود عمل مطابق في النطاق الحالي، ولا تعني فحص جميع Records في Database بصلاحيات غير محدودة.

---

### بعض السجلات لا تصل إليها الدفعات

راجع ما يحدث للسجلات بعد المعالجة.

إذا بقيت أول دفعة مؤهلة دائمًا، فقد تستمر في حجز مقدمة البحث.

في المثال، إفراغ تاريخ الطلب يجعل الدفعة التالية تنتقل إلى سجلات أخرى.

ولا نتجاوز الأخطاء باستخدام `continue` دون تحديد كيف ستخرج Record المتعذرة من دورة المعالجة.

---

### المهمة تفشل في منتصف العمل

اقرأ أول Exception واضحة في سجل Server.

راجع خصوصًا:

- مستخدم التنفيذ ومجموعاته.
- صلاحيات الأجهزة والشركات.
- وجود نوع Activity وExternal ID الصحيحة.
- القيم المخالفة لقواعد العمل.
- زمن الدفعة وحجمها.

في المثال، نترك الخطأ ينتقل إلى Scheduler حتى يتعامل معه كفشل.

لا نستخدم `except Exception` لإخفائه وإظهار نجاح غير حقيقي.

وقد توقف Odoo أوتعطل المهام التي تتكرر إخفاقاتها وفق آليات المجدول؛ لذلك راجع حالة المهمة أيضًا.

---

### تغيير XML لا يغير الجدول الحالي

إذا كانت Record محمية بـ`noupdate`، فقد يبقى إعدادها الحالي بعد Upgrade.

افحص Record داخل Database بدل افتراض تطبيق الملف مجددًا.

عند الحاجة، عدّل إعداد المهمة من الواجهة أوطبّق Migration مناسبة.

---

### تكرار Activities

راجع معيار التكرار المستخدم في الشابتر السابق:

```text
نفس الجهاز
+
نفس نوع المراجعة
+
نفس المستخدم
+
Activity مفتوحة
```

وجود مراجعة لمستخدم آخر لا يمنع تلقائيًا إنشاء مراجعة لمستخدم التنفيذ.

وكذلك، فحص Search قبل Create ليس ضمانًا كاملًا ضد التزامن بين عدة مسارات.

إذا أصبحت المهمة مرتبطة بتكامل خارجي أوعدة Workers ومسارات مختلفة، صمم هوية واضحة للعمل وآلية مناسبة لمنع تكراره.

---

## Important Notes

- Scheduled Action تحدد متى يبدأ العمل، وBusiness Method تحدد ما الذي يحدث.
- Server Action ليست بالضرورة مهمة زمنية، وSequence لا تعمل كمجدول.
- احتفظ بحالات المشروع الفعلية؛ `equipment.asset` لا تحتوي حالة باسم `active`.
- حدد مستخدم التنفيذ والشركة صراحةً، واختبر باستخدام صلاحياته.
- `limit` يحد حجم الدفعة، لكنه لا يضمن التقدم إذا بقيت السجلات مطابقة لنفس Domain.
- `_commit_progress()` تثبت Transaction؛ لا تستخدمها داخل Button أوWizard عادية.
- اختبر Cron عبر مسار المجدول، مثل **Run Manually**، بدل استدعاء دالتها مباشرةً.
- الدفعات المثبتة تبقى محفوظة إذا فشلت دفعة لاحقة.
- استهلاك تاريخ طلب المراجعة يعني تغطية طلب الجدولة، وليس إنجاز Activity.
- `noupdate` تحفظ إعدادات موجودة، وقد تمنع تطبيق تعديلات XML أثناء Upgrade.
- Restart لا يضمن تشغيلًا واحدًا أوفوريًا للمهمة.
- نجاح إنشاء Activity لا يثبت إرسال Email أوإنجاز العمل المطلوب.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[23 HTTP Controllers & Portal]]

---

## Sources

- [Odoo 20 Documentation — Actions](https://www.odoo.com/documentation/20.0/developer/reference/backend/actions.html)
- [Odoo 20 Documentation — Scheduled Actions](https://www.odoo.com/documentation/20.0/developer/reference/backend/actions.html#scheduled-actions-ir-cron)
- [Odoo 20 — Cron and Progress API: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_cron.py)
- [Odoo 20 — Scheduled Action Views: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/views/ir_cron_views.xml)
- [Odoo 20 — Server Actions: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_actions.py)
- [Odoo 20 — Automation Rules: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/base_automation/models/base_automation.py)
- [Odoo 20 — Sequences: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_sequence.py)
- [Odoo 20 — Users: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/res_users.py)