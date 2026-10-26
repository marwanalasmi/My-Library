---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 24 OWL & Frontend

## Table of Contents

- [[#Introduction|Introduction]]
- [[#OWL Fundamentals|OWL Fundamentals]]
- [[#Connecting Frontend to Odoo|Connecting Frontend to Odoo]]
- [[#Practical Example|Practical Example]]
- [[#Verification & Debugging|Verification & Debugging]]
- [[#Choosing an Extension Approach|Choosing an Extension Approach]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

تعلمنا في [[12 Views & XML]] كيف نبني واجهات باستخدام Views القياسية، مثل List وForm، ثم ربطناها بالـActions والقوائم في [[13 Actions, Menus & Buttons]].

هذه الأدوات تكفي لبناء جزء كبير من واجهات Odoo.

لكن بعض المتطلبات تحتاج شاشة مخصصة تحتوي تفاعلًا خاصًا، مثل Dashboard تجمع مؤشرات وتحدثها دون إعادة تحميل الصفحة.

هنا نستخدم **OWL — Odoo Web Library**، وهي مكتبة لبناء الواجهات من Components داخل Odoo Web Client.

في هذا الشابتر سنضيف شاشة **Equipment Overview** إلى مشروع التدريب. ستعرض عدد الأجهزة المتاحة للمستخدم، مع زر لتحديث العدد وحالات واضحة للتحميل والخطأ.

> [!info] إصدار OWL في هذا المثال
> يعتمد المثال على OWL المضمّنة في الفرع الرسمي `20.0` وقت المراجعة، والتي توفر واجهات مثل `signal` و`usePlugin`.
>
> قد تجد أمثلة أقدم تستخدم `useState` أو طرقًا مختلفة للوصول إلى الخدمات. راجع إصدار المثال قبل دمج أجزائه مع هذا الشابتر.

---

## OWL Fundamentals

### Component وTemplate

الـ**Component** تجمع سلوك جزء من الواجهة، بينما تحدد الـ**Template** العناصر التي تظهر للمستخدم.

في مثالنا:

| الجزء | مسؤوليته |
| --- | --- |
| Component | طلب عدد الأجهزة وإدارة حالة الشاشة |
| Template | عرض العنوان والزر والعدد والرسائل |

تُكتب Component باستخدام JavaScript:

```javascript
import { Component } from "@odoo/owl";

export class EquipmentDashboard extends Component {
    static template = "equipment_training.Dashboard";
}
```

القيمة الموجودة في `static template` هي اسم Template التي ستستخدمها Component.

وفي ملف XML:

```xml
<templates xml:space="preserve">
    <t t-name="equipment_training.Dashboard">
        <h1>Equipment Overview</h1>
    </t>
</templates>
```

يجب أن يتطابق الاسمان تمامًا.

استخدام اسم Module في بداية اسم Template يساعد على تجنب تعارض الأسماء مع Modules أخرى.

---

### State وSignals

الـ**State** هي البيانات المتغيرة التي تعتمد عليها الواجهة.

في شاشة الأجهزة سنحتاج إلى:

| القيمة | معناها |
| --- | --- |
| `loading` | هل يوجد طلب قيد التنفيذ؟ |
| `count` | عدد الأجهزة، أو `null` قبل الحصول على نتيجة |
| `error` | رسالة الخطأ، أو نص فارغ عند عدم وجود خطأ |

سنمثل هذه القيم باستخدام `signal`.

```javascript
import { signal } from "@odoo/owl";

const count = signal(null);
```

لقراءة القيمة:

```javascript
count();
```

ولتغييرها:

```javascript
count.set(5);
```

عندما تقرأ Template قيمة Signal، يستطيع OWL تحديث الجزء المعتمد عليها عند تغيرها.

```xml
<strong t-out="this.count()"/>
```

لا تكتب:

```javascript
this.count = 5;
```

إذا كانت `this.count` تحتوي Signal؛ لأن هذا يستبدل Signal نفسها بدل تحديث قيمتها.

---

### Props

الـ**Props** هي مدخلات تمررها Component إلى Component أخرى.

مثلًا، يمكن أن تمرر Dashboard عنوانًا ورقمًا إلى Component مسؤولة عن عرض بطاقة إحصائية.

الفرق الأساسي:

| المفهوم | الاستخدام |
| --- | --- |
| Props | مدخلات تأتي من Component أخرى |
| State | حالة تديرها Component أثناء عملها |

ابدأ بتقسيم الشاشة إلى Components إضافية عندما توجد أجزاء قابلة لإعادة الاستخدام أو عندما تصبح مسؤوليات الشاشة كبيرة.

مثال هذا الشابتر صغير؛ لذلك سنبنيه باستخدام Component واحدة.

---

### Events

نستخدم Events لربط تفاعل المستخدم بمنطق JavaScript.

```xml
<button
    type="button"
    t-on-click="this.refresh"
>
    Refresh
</button>
```

عند الضغط على الزر، ينفذ OWL الدالة `refresh` الخاصة بالـComponent.

الفرق بين تمرير الدالة واستدعائها مهم:

```xml
t-on-click="this.refresh"
```

هذا يربط الدالة بالحدث.

أما تنفيذ تعبير عند وقوع الحدث فيمكن كتابته باستخدام Arrow Function:

```xml
t-on-click="() => this.refresh()"
```

---

### Lifecycle Hooks

للـComponent دورة حياة تشمل الإنشاء، والظهور في الصفحة، والإزالة منها.

توفر OWL Hooks للتعامل مع هذه المراحل.

في مثالنا سنستخدم:

```javascript
onMounted(() => {
    this.refresh();
});
```

لتنفيذ أول طلب بعد ظهور الشاشة.

وبذلك لا يحتاج المستخدم إلى الضغط على **Refresh** للحصول على العدد لأول مرة.

نسجل Hooks داخل `setup()` في هذا المثال، حيث نجهز أيضًا الحالة والاعتماديات التي تحتاجها Component.

---

## Connecting Frontend to Odoo

### الوصول إلى ORM

الـJavaScript تعمل في المتصفح، بينما Models وBusiness Logic تعمل على Server.

عندما نحتاج قراءة Records أو استدعاء Model Method، نستخدم واجهة ORM التي يوفرها Odoo.

في الكود الحالي للفرع `20.0` يمكن الحصول عليها بهذه الطريقة:

```javascript
import { usePlugin } from "@odoo/owl";
import { ORM } from "@web/core/orm_plugin";
```

ثم داخل `setup()`:

```javascript
this.orm = usePlugin(ORM);
```

مثال لعد الأجهزة المتاحة:

```javascript
const count = await this.orm.searchCount(
    "equipment.asset",
    [["state", "=", "available"]]
);
```

الطلب يصل إلى Server وينفذ `search_count` على Model.

القيمة `available` هي القيمة التقنية لحالة الجهاز في مشروع التدريب.

أما `active` فهي Field منفصلة نستخدمها للأرشفة، وليست حالة من حالات `state`.

> [!note] الصلاحيات ما زالت على Server
> طلب ORM ينفذ بصلاحيات المستخدم الحالي وسياق الطلب.
>
> العدد المعروض يعكس الأجهزة التي يستطيع المستخدم الوصول إليها، مع تطبيق القيود ذات الصلة، مثل قواعد الشركات.

ستجد أيضًا نمط `useService("orm")` في أمثلة وكود آخر. يوفر الفرع الذي راجعناه طبقة توافق لهذا النمط، بينما سنستخدم `usePlugin(ORM)` في المثال الجديد.

---

### Registry

الـ**Registry** مكان يسجل فيه Odoo أنواعًا مختلفة من الامتدادات، مثل Client Actions وField Widgets.

لتسجيل شاشة الأجهزة:

```javascript
registry.category("actions").add(
    "equipment_training.dashboard",
    EquipmentDashboard
);
```

هنا:

- `actions` هي فئة Registry.
- `equipment_training.dashboard` هو مفتاح الشاشة.
- `EquipmentDashboard` هي Component التي ستعرضها.

التسجيل يجعل الشاشة قابلة للوصول من نظام Actions، لكنه لا ينشئ Menu تلقائيًا.

---

### Client Action

الـ**Client Action** تستخدم عندما نريد أن يفتح Odoo واجهة ينفذها كود Frontend.

ننشئ Record من Model:

```text
ir.actions.client
```

وتشير `tag` إلى مفتاح Registry:

```xml
<record id="action_equipment_dashboard"
        model="ir.actions.client">
    <field name="name">Equipment Overview</field>
    <field name="tag">equipment_training.dashboard</field>
</record>
```

توجد هنا أسماء مختلفة تؤدي وظائف مختلفة:

| الاسم | وظيفته |
| --- | --- |
| `action_equipment_dashboard` | External ID للـAction داخل Module |
| `equipment_training.dashboard` | مفتاح Registry الذي تطابقه `tag` |
| `equipment_training.Dashboard` | اسم Template الخاصة بالـComponent |

ليس مطلوبًا أن تكون الأسماء الثلاثة متطابقة.

المطلوب أن تتطابق كل إشارة مع التعريف الذي تشير إليه.

---

### Assets

وجود ملف JavaScript داخل Module لا يعني أن المتصفح سيحمله تلقائيًا.

يجب إضافته إلى Asset Bundle مناسبة.

من أشهر Bundles:

| Bundle | الاستخدام |
| --- | --- |
| `web.assets_backend` | واجهة Odoo الداخلية |
| `web.assets_frontend` | Website وPortal |

شاشة هذا الشابتر جزء من الواجهة الداخلية؛ لذلك سنستخدم:

```text
web.assets_backend
```

سنضيف إليها ملف JavaScript وملف Template.

أما ملف XML الذي ينشئ Client Action وMenu، فسنضيفه إلى قائمة `data` في Manifest.

هذا يربط ما تعلمناه هنا بآلية تحميل البيانات في [[10 Data Files & External IDs]].

---

## Practical Example

### 1. تحديد المطلوب والمتطلبات

سنضيف إلى قائمة **Equipment** عنصرًا باسم **Overview**.

عند فتحه:

1. تظهر الشاشة.
2. يبدأ تحميل عدد الأجهزة المتاحة.
3. يظهر العدد، بما في ذلك الصفر.
4. يستطيع المستخدم تحديثه باستخدام **Refresh**.
5. تظهر رسالة مناسبة إذا فشل الطلب.

يعتمد المثال على وجود:

- Module باسم `equipment_training`.
- Model باسم `equipment.asset`.
- Field باسم `state` تحتوي القيمة `available`.
- القائمة الرئيسية `equipment_training.menu_equipment_root`.
- المجموعة `equipment_training.group_equipment_user`.
- صلاحية قراءة الأجهزة للمستخدم الذي سيجرب الشاشة.

سنستخدم الصلاحيات التي بنيناها سابقًا؛ الشاشة تقرأ Model موجودة ولا تنشئ Model جديدة.

---

### 2. تجهيز الملفات

أضف الملفات التالية داخل Module:

```text
equipment_training/
├── __manifest__.py
├── static/
│   └── src/
│       └── dashboard/
│           ├── dashboard.js
│           └── dashboard.xml
└── views/
    └── equipment_dashboard_views.xml
```

هذه شجرة للملفات المتعلقة بالإضافة فقط.

| الملف | المحتوى |
| --- | --- |
| `dashboard.js` | Component وطلب البيانات والتسجيل |
| `dashboard.xml` | Template الخاصة بالشاشة |
| `equipment_dashboard_views.xml` | Client Action وMenu |

ملفات JavaScript وTemplate لا تحتاج إلى Imports داخل `__init__.py`.

---

### 3. إنشاء Component

داخل:

```text
equipment_training/static/src/dashboard/dashboard.js
```

اكتب:

```javascript
import {
    Component,
    onMounted,
    signal,
    usePlugin,
    useScope,
} from "@odoo/owl";

import { _t } from "@web/core/l10n/translation";
import { ORM } from "@web/core/orm_plugin";
import { registry } from "@web/core/registry";


export class EquipmentDashboard extends Component {
    static template = "equipment_training.Dashboard";

    setup() {
        this.orm = usePlugin(ORM);
        this.scope = useScope();

        this.loading = signal(false);
        this.count = signal(null);
        this.error = signal("");

        onMounted(() => {
            this.refresh();
        });
    }

    async refresh() {
        if (this.loading() || this.scope.isDestroyed()) {
            return;
        }

        this.loading.set(true);
        this.count.set(null);
        this.error.set("");

        try {
            const count = await this.orm.searchCount(
                "equipment.asset",
                [["state", "=", "available"]]
            );

            if (!this.scope.isDestroyed()) {
                this.count.set(count);
            }
        } catch (error) {
            if (!this.scope.isDestroyed()) {
                console.error(
                    "Failed to load the equipment dashboard.",
                    error
                );

                this.error.set(
                    _t("Could not load equipment. Please try again.")
                );
            }
        } finally {
            if (!this.scope.isDestroyed()) {
                this.loading.set(false);
            }
        }
    }
}


registry.category("actions").add(
    "equipment_training.dashboard",
    EquipmentDashboard
);
```

تتولى `refresh()` إدارة دورة الطلب:

```text
منع طلب متكرر أثناء التحميل
            ↓
تهيئة حالة الشاشة
            ↓
طلب العدد من Server
            ↓
عرض النتيجة أو رسالة الخطأ
            ↓
إنهاء حالة التحميل
```

استخدمنا `null` للتمييز بين **عدم وجود نتيجة بعد** وبين **وجود نتيجة تساوي صفرًا**.

كما نمسح العدد السابق عند بدء طلب جديد، حتى لا يظهر وكأنه نتيجة الطلب الجاري.

أما فحص `isDestroyed()` فيمنع تحديث حالة الشاشة إذا غادرها المستخدم قبل اكتمال الطلب.

تظهر للمستخدم رسالة مختصرة، بينما تسجل تفاصيل الخطأ في Browser Console للمساعدة في التشخيص.

---

### 4. إنشاء Template

داخل:

```text
equipment_training/static/src/dashboard/dashboard.xml
```

اكتب:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<templates xml:space="preserve">

    <t t-name="equipment_training.Dashboard">
        <section class="o_equipment_dashboard h-100 overflow-auto p-4">

            <div class="d-flex flex-wrap align-items-center
                        justify-content-between gap-3 mb-4">

                <div>
                    <h1 class="mb-2">
                        Equipment Overview
                    </h1>

                    <p class="text-muted mb-0">
                        Available equipment you can access.
                    </p>
                </div>

                <button
                    type="button"
                    class="btn btn-primary"
                    t-on-click="this.refresh"
                    t-att-disabled="this.loading()"
                >
                    Refresh
                </button>

            </div>

            <p
                t-if="this.loading()"
                class="text-muted"
                role="status"
            >
                Loading equipment...
            </p>

            <div
                t-elif="this.error()"
                class="alert alert-danger"
                role="alert"
            >
                <t t-out="this.error()"/>
            </div>

            <div
                t-elif="this.count() !== null"
                class="card shadow-sm"
            >
                <div class="card-body">
                    <h2 class="h5 text-muted">
                        Available Equipment
                    </h2>

                    <p class="display-4 fw-bold mb-0">
                        <t t-out="this.count()"/>
                    </p>
                </div>
            </div>

        </section>
    </t>

</templates>
```

استخدمنا Classes متاحة في واجهة Odoo لتنسيق الشاشة دون إضافة ملف Styling خاص لهذا المثال.

أهم Directives هنا:

| Directive | الاستخدام |
| --- | --- |
| `t-on-click` | ربط الضغط على الزر بالدالة |
| `t-att-disabled` | تعطيل الزر أثناء التحميل |
| `t-if` و`t-elif` | اختيار حالة العرض |
| `t-out` | عرض قيمة التعبير مع Escaping |

شرط عرض العدد هو:

```javascript
this.count() !== null
```

وليس:

```javascript
this.count()
```

لأن الصفر نتيجة صحيحة، لكنه يعتبر قيمة False عند استخدامه مباشرةً كشرط في JavaScript.

> [!note] تعبيرات Template
> التعبيرات في OWL Template تستخدم JavaScript.
>
> تشابه Directives مع Server QWeb التي درسناها في [[20 QWeb & Reports]] لا يعني أن سياق التعبير واحد.

---

### 5. إنشاء Client Action والقائمة

داخل:

```text
equipment_training/views/equipment_dashboard_views.xml
```

اكتب:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<odoo>

    <record id="action_equipment_dashboard"
            model="ir.actions.client">
        <field name="name">Equipment Overview</field>
        <field name="tag">equipment_training.dashboard</field>
    </record>

    <menuitem
        id="menu_equipment_dashboard"
        name="Overview"
        parent="equipment_training.menu_equipment_root"
        action="action_equipment_dashboard"
        sequence="5"
        groups="equipment_training.group_equipment_user"
    />

</odoo>
```

عرّفنا Action قبل Menu التي تشير إليها.

ويجب تحميل القائمة الرئيسية من ملفها السابق قبل تحميل هذا الملف.

وجود `groups` يحدد ظهور القائمة، بينما تظل صلاحيات Model وقواعد الوصول على Server مسؤولة عن حماية البيانات.

---

### 6. تحديث Manifest

افتح:

```text
equipment_training/__manifest__.py
```

تأكد أن قائمة `depends` تتضمن `web`.

إذا كنت تتابع إضافات الشابترات السابقة، فاحتفظ بالاعتماديات التي أضفناها:

```python
"depends": [
    "base",
    "web",
    "mail",
    "portal",
],
```

أضف هذا المسار إلى قائمة `data` الحالية، بعد الملف الذي ينشئ القائمة الرئيسية:

```python
"views/equipment_dashboard_views.xml",
```

في ترتيب مشروعنا، يجب أن يأتي بعد:

```python
"views/equipment_menus.xml",
```

ثم أضف تعريف Assets داخل Dictionary الخاصة بالـManifest:

```python
"assets": {
    "web.assets_backend": [
        "equipment_training/static/src/dashboard/dashboard.js",
        "equipment_training/static/src/dashboard/dashboard.xml",
    ],
},
```

إذا كانت `assets` موجودة بالفعل، أضف الملفات إلى تعريفها الحالي.

> [!important] ملفا XML لهما طريقتا تحميل مختلفتان
> ملف `equipment_dashboard_views.xml` يحتوي Records ويضاف إلى `data`.
>
> ملف `dashboard.xml` يحتوي Static Template ويضاف إلى Asset Bundle.
>
> لا تضف ملف Template إلى قائمة `data` لمجرد أن امتداده XML.

---

### 7. تطبيق التعديل

بعد حفظ الملفات:

1. أعد تشغيل Server التدريب لقراءة Manifest الحالية.
2. افتح **Apps**.
3. احذف Filter **Apps** إذا كان يخفي Module التدريب.
4. ابحث عن `Equipment Training`.
5. نفذ **Upgrade**.
6. أعد تحميل صفحة المتصفح تحميلًا كاملًا، ويمكن استخدام `Ctrl + Shift + R`.
7. افتح قائمة **Equipment** ثم **Overview**.

الناتج المتوقع:

- ظهور شاشة **Equipment Overview**.
- بدء تحميل العدد عند فتحها.
- ظهور عدد الأجهزة المتاحة للمستخدم.
- تعطيل **Refresh** أثناء الطلب ثم تفعيلها بعد انتهائه.

للتحديث باستخدام CLI عند الحاجة، استخدم طريقة Upgrade التي تعلمناها في [[03 Modules & Addons]]، ثم شغّل Server وأعد تحميل المتصفح.

---

## Verification & Debugging

### التحقق من العدد

افتح قائمة الأجهزة بالحساب نفسه، وطبق Filter على الحالة **Available**.

قارن العدد بنتيجة Dashboard، مع استخدام الشركات نفسها.

العدد يتأثر بما يلي:

- Domain الخاصة بالطلب.
- صلاحيات المستخدم وقواعد الوصول.
- سياق الشركات.
- سلوك استبعاد Records المؤرشفة في البحث المعتاد.

لذلك لا تقارن Dashboard بحساب مختلف أو بسياق شركات مختلف وتتوقع نتيجة واحدة دائمًا.

---

### اختبار الصفر والتحميل والخطأ

اختبر الحالات التالية داخل Database التدريب:

| الحالة | الناتج المتوقع |
| --- | --- |
| توجد أجهزة متاحة | ظهور العدد |
| لا توجد أجهزة متاحة | ظهور `0` |
| الطلب بطيء | ظهور رسالة التحميل وتعطيل الزر |
| الضغط المتكرر أثناء التحميل | استمرار الطلب الجاري دون إنشاء طلبات إضافية من الزر |
| فشل الاتصال | ظهور رسالة الخطأ |
| عودة الاتصال والضغط على Refresh | محاولة تحميل جديدة |
| مغادرة الشاشة أثناء الطلب | عدم تحديث Component بعد إزالتها |

يمكن استخدام Browser DevTools لتجربة اتصال بطيء أو Offline داخل بيئة التدريب.

إذا فعلت Offline، أعد الاتصال إلى وضعه الطبيعي قبل تجربة Refresh مرة أخرى.

---

### استخدام Browser DevTools

ابدأ بأداتي **Console** و**Network**.

| العَرَض | ما الذي تتحقق منه؟ |
| --- | --- |
| Action غير موجودة في Registry | تحميل JavaScript وتطابق `tag` مع مفتاح Registry |
| Template غير موجودة | إضافتها إلى Bundle وتطابق `t-name` مع `static template` |
| الطلب يعيد Access Error | صلاحيات المستخدم على `equipment.asset` |
| العدد غير متوقع | Domain والحالة التقنية والشركات والأرشفة |
| التعديل لا يظهر | Upgrade وAsset Bundle وإعادة تحميل الصفحة |

لا تجعل حذف Asset Attachments من Database أول خطوة للتشخيص.

ابدأ بقراءة الخطأ والتأكد من المسارات والتسجيل.

يمكن استخدام وضع:

```text
debug=assets
```

لتسهيل تتبع ملفات الواجهة أثناء التطوير.

إذا كان رابط الصفحة يحتوي Query Parameters بالفعل، أضف الخيار بطريقة تحافظ على هذه المعاملات.

للتوسع في تتبع الأخطاء، راجع [[27 Debugging]].

---

## Choosing an Extension Approach

### متى نستخدم OWL؟

اختر الأداة وفق طبيعة المطلوب:

| المطلوب | نقطة البداية المناسبة |
| --- | --- |
| عرض Records وتحريرها بالطرق المعتادة | Standard Views |
| تخصيص عرض Field | Widget أو Field Component |
| شاشة تفاعلية مستقلة | OWL Client Action |
| صفحة للمستخدم الخارجي | Website أو Portal حسب المتطلب |
| تعديل سلوك قائم لا تغطيه Extension Points مناسبة | دراسة Patching |

تعلمنا في [[23 HTTP Controllers & Portal]] بناء Routes وصفحات Portal.

أما شاشة هذا الشابتر فتعمل داخل Odoo Web Client وتستخدم ORM الموجودة فيه للوصول إلى Model.

لا نحتاج إلى Controller مخصصة لمجرد تنفيذ عملية العد.

---

### Services وPlugins

يوفر Odoo أدوات مشتركة للوصول إلى وظائف النظام، مثل التعامل مع ORM أو التنقل أو الإشعارات.

ستجد هذه الأدوات بصيغ Services أو Plugins بحسب الجزء والإصدار.

الفكرة العملية هي استخدام الأداة التي يوفرها Framework للمهمة، مع مراجعة طريقة الوصول إليها في الإصدار المستهدف.

في مثالنا:

```javascript
this.orm = usePlugin(ORM);
```

تمنحنا ORM الخاصة بالواجهة، فنستخدمها بدل بناء طلبات HTTP يدويًا لاستدعاء Model.

---

### Patching

تسمح `patch` بتعديل سلوك Object أو Class موجودة.

مسار استيرادها:

```javascript
import { patch } from "@web/core/utils/patch";
```

لكن قبل استخدامها، تحقق من وجود طريقة توسعة أوضح، مثل:

- إضافة Component جديدة.
- التسجيل في Registry مناسبة.
- استخدام Props أو Extension Point يوفرها الجزء المستهدف.
- توريث Template عند ملاءمة ذلك.

عند الحاجة إلى تعديل Instance Methods في Class، يكون الاستهداف عادةً للـ`prototype`.

وتستخدم `super` للحفاظ على السلوك السابق عندما يتطلب التعديل ذلك.

Patching تؤثر في السلوك المشترك للجزء المستهدف؛ لذلك اختبر أثرها على الشاشات التي تستخدمه، وراجعها عند ترقية Odoo.

Dashboard التي بنيناها مسجلة كـClient Action مستقلة، ولذلك لا تحتاج Patch.

---

## Important Notes

- استخدم واجهات OWL المطابقة لنسخة Odoo التي تعمل عليها؛ لا تجمع أجزاء من أمثلة مختلفة دون مراجعة.
- Signal تُقرأ باستدعائها، وتُحدّث باستخدام `set()`.
- قيمة الصفر تختلف عن عدم وجود نتيجة؛ اختر شرط العرض وفق هذا الفرق.
- اسم Template يجب أن يطابق `static template`.
- مفتاح Action Registry يجب أن يطابق `tag` في Client Action.
- ملفات الشاشة داخل `static` تحتاج إلى Asset Bundle مناسبة.
- XML الخاصة بالـAction والقائمة تُحمّل من `data`، بينما Static Template تُحمّل من Assets.
- تغيير Manifest يتطلب تطبيق التحديث المناسب، وقد يحتاج المتصفح إلى إعادة تحميل كاملة.
- تعطيل الزر يحسن التفاعل، لكن حماية البيانات تظل مسؤولية Server.
- أخطاء الوصول تُحل بمراجعة الصلاحيات، وليس بتجاوزها.
- لا تحفظ API Keys أو أسرار التكامل داخل Frontend؛ الملفات المحملة إلى المتصفح قابلة للفحص.
- ضع Business Logic القابلة لإعادة الاستخدام داخل Models، واجعل الواجهة تستدعيها.
- عالج حالات التحميل والخطأ وإزالة Component، إلى جانب حالة نجاح الطلب.
- العدد المعروض في مثالنا يعكس Records التي تسمح بها صلاحيات المستخدم وسياق الطلب.
- استخدم Patching عندما يلائم المطلوب وبعد مراجعة نقاط التوسعة المتاحة.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[25 APIs & Integrations]]

---

## Sources

- [Odoo 20 Documentation — Create a Client Action](https://www.odoo.com/documentation/20.0/developer/howtos/javascript_client_action.html)
- [Odoo 20 Documentation — Assets](https://www.odoo.com/documentation/20.0/developer/reference/frontend/assets.html)
- [Odoo 20 Documentation — JavaScript Modules](https://www.odoo.com/documentation/20.0/developer/reference/frontend/javascript_modules.html)
- [Odoo 20 Documentation — Patching Code](https://www.odoo.com/documentation/20.0/developer/reference/frontend/patching_code.html)
- [Odoo 20 Official Source — Bundled OWL](https://github.com/odoo/odoo/blob/20.0/addons/web/static/lib/owl/owl.js)
- [Odoo 20 Official Source — ORM Plugin](https://github.com/odoo/odoo/blob/20.0/addons/web/static/src/core/orm_plugin.js)