---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 23 HTTP Controllers & Portal

## Table of Contents

- [[#Introduction|Introduction]]
- [[#HTTP Controller Fundamentals|HTTP Controller Fundamentals]]
- [[#Authentication & Request Security|Authentication & Request Security]]
- [[#Portal Fundamentals|Portal Fundamentals]]
- [[#Practical Example|Practical Example]]
- [[#Controller Inheritance|Controller Inheritance]]
- [[#Testing & Troubleshooting|Testing & Troubleshooting]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

حتى الآن، تعاملنا مع بيانات المشروع من خلال واجهة Odoo الداخلية وOdoo Shell والتقارير.

لكننا قد نحتاج إلى:

- رابط يعيد بيانات بصيغة JSON.
- صفحة يعرضها المتصفح.
- بوابة تسمح للمستخدم الخارجي بمتابعة سجلات محددة.
- نقطة دخول لتكامل HTTP.

نستخدم **HTTP Controllers** لتعريف هذه الروابط ومعالجة الطلبات الواردة إليها.

```text
Browser أوClient
    ↓
HTTP Request
    ↓
Route
    ↓
Controller Method
    ↓
ORM وBusiness Methods
    ↓
HTTP Response
```

في هذا الشابتر، سنضيف:

1. مسار JSON يعرض عدد الأجهزة المتاحة التي يستطيع المستخدم قراءتها.
2. صفحة Portal تعرض إعارات المستخدم الحالي.
3. صفحة تفاصيل لإعارة واحدة، مع التحقق من حق الوصول إليها.

سنستخدم Models والحقول الموجودة في المشروع.

---

## HTTP Controller Fundamentals

### ما هي Controller؟

Controller هي Class ترث من:

```python
http.Controller
```

مثلًا:

```python
from odoo import http


class EquipmentController(http.Controller):
    ...
```

تحتوي على Methods يمكن نشرها كمسارات باستخدام:

```python
@http.route(...)
```

لا نستخدم `_name` لتعريف Controller؛ فهي ليست Business Model.

---

### ما هي Route؟

Route تربط عنوانًا معينًا بـMethod:

```python
@http.route(
    "/equipment/status",
    type="http",
    auth="user",
    methods=["GET"],
)
```

أهم الإعدادات هنا:

| الإعداد | الغرض |
| --- | --- |
| المسار | تحديد عنوان الطلب |
| `type` | طريقة معالجة الطلب والاستجابة |
| `auth` | طريقة المصادقة |
| `methods` | HTTP Methods المسموحة |

عند وصول طلب مطابق، يستدعي Odoo Method المرتبطة بالمسار.

يفضل تحديد `methods` صراحةً؛ غيابها قد يسمح بأكثر مما يحتاجه المسار.

---

### الفرق بين GET وPOST

نستخدم GET لجلب البيانات أوعرض صفحة.

مثلًا:

```text
GET /equipment/status
```

أما العمليات التي تغير بيانات العمل فتحتاج إلى تصميم يناسب التعديل، وغالبًا تستخدم POST في نماذج المتصفح.

لا تجعل زيارة رابط GET تنفذ عملية مثل:

- إلغاء إعارة.
- تغيير حالة جهاز.
- إنشاء طلب مراجعة.

المسارات العملية في هذا الشابتر للقراءة فقط.

---

### الوصول إلى ORM باستخدام `request.env`

نستورد:

```python
from odoo.http import request
```

ثم نستخدم:

```python
asset_model = request.env["equipment.asset"]
```

`request.env` هي Environment الخاصة بالطلب الحالي.

تحدد المستخدم والسياق اللذين ستعمل بهما عمليات ORM، كما تعلمنا في [[09 Environment & Context]].

مثلًا:

```python
count = asset_model.search_count([
    ("state", "=", "available"),
])
```

هذا البحث يخضع للصلاحيات والقيود المطبقة على المستخدم الحالي.

---

### إرجاع JSON

داخل Route من نوع `http`:

```python
return request.make_json_response({
    "service": "equipment",
    "available_count": count,
})
```

تتولى `make_json_response()` تحويل البيانات إلى JSON وضبط Content-Type المناسب.

لا نحتاج إلى تكوين JSON يدويًا بدمج Strings.

---

### إرجاع صفحة HTML

يمكن للـController عرض QWeb Template:

```python
return request.render(
    "equipment_training.portal_equipment_loans",
    {
        "loans": loans,
    },
)
```

المفتاح `loans` يصبح متاحًا داخل Template.

هذا امتداد لاستخدام QWeb الذي تعلمناه في [[20 QWeb & Reports]]، لكن الناتج هنا صفحة Browser بدل Report.

---

### JSON وJSON-RPC

إرجاع JSON لا يعني أن Route يجب أن تكون من نوع JSON-RPC.

في Odoo 20، نميز بين:

| النوع | الاستخدام |
| --- | --- |
| `type="http"` | معالجة HTTP وإرجاع Response مثل HTML أوJSON |
| `type="jsonrpc"` | التعامل مع بروتوكول JSON-RPC وبنية الطلب والاستجابة الخاصة به |

قد تجد أمثلة قديمة تستخدم:

```python
type="json"
```

هذه تسمية قديمة أصبحت Alias مهجورة لصالح `jsonrpc`.

في التطبيق الحالي، نستخدم `http` مع `make_json_response()`.

---

## Authentication & Request Security

### Authentication وAuthorization

المصادقة تحدد هوية المستخدم.

أما الصلاحيات فتحدد ما يستطيع هذا المستخدم فعله.

```text
المستخدم سجل الدخول
    ↓
ما السجلات المسموح له بقراءتها؟
    ↓
ما العمليات المسموح له بتنفيذها؟
```

لذلك، `auth="user"` لا تمنح المستخدم حق قراءة كل الأجهزة أوالإعارات.

---

### خيارات `auth`

| القيمة | السلوك الأساسي |
| --- | --- |
| `user` | تتطلب مستخدمًا مسجلًا وتعمل بصلاحياته |
| `public` | تسمح بطلب غير مسجل، ويستخدم Public User عند غياب المستخدم |
| `bearer` | تدعم المصادقة باستخدام API Token في Authorization Header |
| `none` | تستخدمها مسارات إطار العمل التي قد تعمل دون Database |

`public` لا تعني أن جميع الطلبات تعمل بهوية Public User؛ المستخدم المسجل يحتفظ بهويته وفق آلية المصادقة.

وفي التطبيق، نستخدم `auth="user"`.

---

### ملاحظة خاصة بـBearer في Odoo 20

وفق مصدر Odoo 20، تعريف Route باستخدام:

```python
auth="bearer"
```

يتطلب تحديد:

```python
bearer_scope
```

حتى يتحدد نطاق المفتاح المقبول.

وتوجد تفاصيل إضافية تتعلق باستخدام Session عند غياب Header، لذلك لا تفترض أن `bearer` تعني قبول Token وحدها في كل سياق.

سنعود إلى تصميم المصادقة للتكاملات في [[25 APIs & Integrations]].

---

### Record ID ليست Permission

رابط مثل:

```text
/my/equipment-loans/42
```

يحمل معرّف إعارة.

لكنه لا يثبت أن المستخدم يستطيع قراءتها.

يجب التحقق من الوصول حتى لو:

- جاء ID من رابط داخل الصفحة.
- كان الحقل مخفيًا.
- كان المستخدم يعرف الرقم.
- سبق للمستخدم فتح صفحة قائمة السجلات.

سنختبر تغيير ID في الرابط عمليًا.

---

### التحقق من المدخلات

يمكن أن تأتي المدخلات من:

- مسار URL.
- Query String.
- Form.
- JSON Body.

Converter مثل:

```text
<int:loan_id>
```

يساعد على تحويل المعامل إلى Integer.

لكنه لا يتحقق من ملكية الإعارة أوصلاحية الوصول إليها.

كذلك، لا تسمح للمستخدم بتمرير Model أوDomain أوأسماء Fields بحرية إلى Code تنفذها بصلاحيات واسعة.

حدد البيانات المقبولة ومعناها داخل المسار.

---

### CSRF

في مسارات `http` المعتادة، تُفعّل حماية CSRF افتراضيًا، ويطبق الفحص على الطلبات التي تحتاجه، مثل POST.

عند إنشاء Form باستخدام QWeb، يوضع Token داخلها:

```xml
<input
    type="hidden"
    name="csrf_token"
    t-att-value="request.csrf_token()"
/>
```

هذا الجزء يوضع داخل Form التي سترسل الطلب.

إذا فشل POST بسبب CSRF، افحص:

- وجود Token.
- صلاحية Session.
- إرسال الحقل بالاسم الصحيح.
- توافق Form مع Route.

لا تعطل الحماية لمجرد تجاوز الخطأ.

أما Webhooks والعملاء الخارجيون فيحتاجون إلى تصميم مصادقة وتحقق يناسبهم، وسنتناول ذلك في [[25 APIs & Integrations]].

---

## Portal Fundamentals

### ما هي Portal؟

Portal توفر واجهة للمستخدم الخارجي لعرض بيانات أوتنفيذ إجراءات محددة يسمح بها التطبيق.

Portal User ليس Internal User.

ولا يحصل تلقائيًا على صلاحيات Models المخصصة بمجرد تمكنه من تسجيل الدخول.

في مشروعنا، نحتاج إلى تحديد:

```text
حساب Portal
    ↓
الإعارات التابعة لهذا الحساب
    ↓
الحقول التي تعرضها الصفحات
```

---

### سياسة الملكية في المثال

لدينا داخل `equipment.loan` الحقل:

```text
requested_by_id
```

وهو مرتبط بـ`res.users`.

سنستخدمه لتحديد صاحب الإعارة:

```python
[("requested_by_id", "=", user.id)]
```

بذلك يقرأ حساب Portal الإعارات التي تشير إليه مباشرةً.

هذه هي سياسة المثال: ملكية على مستوى الحساب.

إذا احتاج مشروع آخر إلى مشاركة السجلات بين جهات الاتصال التابعة لشركة عميلة، فيجب تصميم العلاقة والقيود لذلك صراحةً.

---

### الحماية على مستوى Model

سنضيف صلاحية قراءة لمجموعة Portal مقيدة بصاحب الإعارة.

ثم نستخدم الشرط نفسه داخل Controller لتحديد محتوى الصفحات.

وجود قيد على Model مهم لأن البيانات قد تُطلب من مسارات أخرى غير الصفحة الحالية.

راجع [[11 Security & Multi-Company]] لفهم تركيب الصلاحيات والقيود العامة في Odoo 20.

---

### دور `portal.mixin`

توفر `portal.mixin` أدوات مثل روابط الوصول وAccess Tokens.

لكن إضافتها وحدها لا تنشئ صفحات Portal أوتحدد سياسة قراءة السجلات.

المثال الحالي يعتمد على تسجيل الدخول وصلاحيات ORM، لذلك لا يحتاج إلى Access Tokens.

إذا احتجنا لاحقًا إلى مشاركة مستند عبر Token، فذلك نمط تفويض إضافي يحتاج تحققًا مناسبًا قبل الوصول إلى البيانات.

---

## Practical Example

سنضيف التطبيق إلى `equipment_training`.

سيشمل المسارات التالية:

| المسار | النتيجة |
| --- | --- |
| `/equipment/status` | JSON بعدد الأجهزة المتاحة المقروءة |
| `/my/equipment-loans` | قائمة إعارات المستخدم |
| `/my/equipment-loans/<id>` | تفاصيل إعارة مسموحة للمستخدم |

صفحات Portal ستعرض اسم الإعارة وفترتها ومدتها.

لن نمنح Portal صلاحية عامة لقراءة الأجهزة حتى نعرض اسم الجهاز؛ صلاحيات Records المرتبطة تحتاج إلى تصميم مستقل.

---

### 1. تجهيز الملفات

الأجزاء المطلوبة:

```text
equipment_training/
├── __init__.py
├── __manifest__.py
├── controllers/
│   ├── __init__.py
│   └── main.py
├── security/
│   └── ir.access.csv
└── views/
    └── equipment_portal_templates.xml
```

سنضيف Controllers وTemplates، ونوسع صلاحيات الإعارات.

لا نحتاج إلى Fields جديدة.

---

### 2. إضافة Dependency على `portal`

داخل Manifest:

```python
"depends": [
    "base",
    "web",
    "mail",
    "portal",
],
```

نستخدم من Module `portal`:

- Class باسم `CustomerPortal`.
- Layout الخاصة بصفحات البوابة.
- أدوات Pagination.

احتفظ بأي Dependencies إضافية موجودة في المشروع.

---

### 3. إضافة صلاحية قراءة الإعارات للبوابة

داخل:

```text
security/ir.access.csv
```

أضف الصف التالي:

```csv
access_loan_portal,Portal Own Equipment Loans,equipment.loan,base.group_portal,r,"[('requested_by_id', '=', user.id)]"
```

يبقى Header الملف مرة واحدة في بدايته:

```csv
id,name,model_id,group_id/id,operation,domain
```

واحتفظ بالصفوف السابقة.

هذا الصف يمنح مجموعة Portal قراءة الإعارات التابعة للمستخدم الحالي.

لا يمنح Create أوUpdate أوDelete.

كذلك يبقى قيد الشركة العام الذي أنشأناه سابقًا مطبقًا:

```python
[("company_id", "in", company_ids)]
```

إذًا يجب أن تحقق الإعارة شرط الملكية وقيود الشركات المطبقة.

---

### 4. إنشاء Controllers

داخل:

```text
controllers/main.py
```

اكتب:

```python
from werkzeug.exceptions import Forbidden, NotFound

from odoo import http
from odoo.addons.portal.controllers.portal import (
    CustomerPortal,
    pager as portal_pager,
)
from odoo.http import request


class EquipmentController(http.Controller):

    @http.route(
        "/equipment/status",
        type="http",
        auth="user",
        methods=["GET"],
    )
    def equipment_status(self, **kwargs):
        asset_model = request.env["equipment.asset"]

        if not asset_model.has_access("read"):
            return request.make_json_response(
                {"error": "Access denied."},
                status=403,
                headers={"Cache-Control": "no-store"},
            )

        available_count = asset_model.search_count([
            ("state", "=", "available"),
        ])

        return request.make_json_response(
            {
                "service": "equipment",
                "available_count": available_count,
            },
            headers={"Cache-Control": "no-store"},
        )


class EquipmentPortal(CustomerPortal):

    def _equipment_loan_domain(self):
        return [
            ("requested_by_id", "=", request.env.user.id),
        ]

    def _equipment_loan_model(self):
        loan_model = request.env["equipment.loan"]

        if not loan_model.has_access("read"):
            raise Forbidden()

        return loan_model

    @http.route(
        [
            "/my/equipment-loans",
            "/my/equipment-loans/page/<int:page>",
        ],
        type="http",
        auth="user",
        methods=["GET"],
        website=True,
    )
    def equipment_loans(self, page=1, **kwargs):
        loan_model = self._equipment_loan_model()
        domain = self._equipment_loan_domain()

        pager_values = portal_pager(
            url="/my/equipment-loans",
            total=loan_model.search_count(domain),
            page=page,
            step=20,
        )

        loans = loan_model.search(
            domain,
            order="id desc",
            limit=20,
            offset=pager_values["offset"],
        )

        values = self._prepare_portal_layout_values()
        values.update({
            "page_name": "equipment_loans",
            "loans": loans,
            "pager": pager_values,
        })

        return request.render(
            "equipment_training.portal_equipment_loans",
            values,
            headers={"Cache-Control": "no-store"},
        )

    @http.route(
        "/my/equipment-loans/<int:loan_id>",
        type="http",
        auth="user",
        methods=["GET"],
        website=True,
    )
    def equipment_loan_detail(self, loan_id, **kwargs):
        loan_model = self._equipment_loan_model()

        domain = self._equipment_loan_domain() + [
            ("id", "=", loan_id),
        ]

        loan = loan_model.search(
            domain,
            limit=1,
        )

        if not loan:
            raise NotFound()

        values = self._prepare_portal_layout_values()
        values.update({
            "page_name": "equipment_loan",
            "loan": loan,
        })

        return request.render(
            "equipment_training.portal_equipment_loan_detail",
            values,
            headers={"Cache-Control": "no-store"},
        )
```

#### مسار JSON

نفحص أولًا وجود صلاحية قراءة على Model الأجهزة.

ثم نستخدم `search_count()`؛ فتطبق قيود السجلات والشركات على العدد الناتج.

إذا لم يملك المستخدم صلاحية القراءة، نعيد JSON بحالة `403`.

أما المستخدم غير المسجل، فتعالج طبقة المصادقة طلبه قبل دخول Method.

#### قائمة الإعارات

نستخرج هوية المستخدم من:

```python
request.env.user.id
```

ولا نقبل User ID من Query String لتحديد صاحب السجلات.

ونستخدم Pagination بدل تحميل جميع الإعارات دفعة واحدة.

#### صفحة التفاصيل

نبحث عن الإعارة باستخدام شرطين:

```text
صاحب الإعارة هو المستخدم الحالي
+
ID يطابق المطلوب
```

وتبقى صلاحيات ORM مطبقة على البحث.

إذا لم توجد Record مقروءة ومطابقة، نعيد `404`، سواء كان ID غير موجود أوغير متاح ضمن هذا المسار.

#### اختيار البيانات

لا نعيد `read()` لجميع Fields في Response.

مسار JSON يحدد مفاتيحه، وTemplates تحدد الحقول التي تعرضها.

ولم نستخدم `sudo()`.

---

### 5. ربط Imports

داخل:

```text
controllers/__init__.py
```

اكتب:

```python
from . import main
```

وفي `__init__.py` الرئيسي، احتفظ بالاستيرادات السابقة وأضف Controllers:

```python
from . import models
from . import wizard
from . import controllers
```

تحميل Controllers يتم عبر Python Imports.

لا تضف ملف Python الخاص بها إلى قائمة `data`.

---

### 6. إنشاء Templates

داخل:

```text
views/equipment_portal_templates.xml
```

اكتب:

```xml
<odoo>
    <template id="portal_equipment_loans">
        <t t-call="portal.portal_layout">
            <h1>My Equipment Loans</h1>

            <p t-if="not loans">
                No equipment loans are available.
            </p>

            <t t-if="loans">
                <div class="table-responsive">
                    <table class="table table-striped">
                        <thead>
                            <tr>
                                <th>Loan</th>
                                <th>Start</th>
                                <th>End</th>
                            </tr>
                        </thead>

                        <tbody>
                            <tr t-foreach="loans" t-as="loan">
                                <td>
                                    <a
                                        t-att-href="'/my/equipment-loans/%s' % loan.id"
                                    >
                                        <span t-field="loan.name"/>
                                    </a>
                                </td>

                                <td>
                                    <span t-field="loan.start_at"/>
                                </td>

                                <td>
                                    <span t-field="loan.end_at"/>
                                </td>
                            </tr>
                        </tbody>
                    </table>
                </div>

                <t t-call="portal.pager"/>
            </t>
        </t>
    </template>

    <template id="portal_equipment_loan_detail">
        <t t-call="portal.portal_layout">
            <p>
                <a href="/my/equipment-loans">
                    Back to my loans
                </a>
            </p>

            <h1>
                <span t-field="loan.name"/>
            </h1>

            <dl class="row">
                <dt class="col-sm-3">Start</dt>
                <dd class="col-sm-9">
                    <span t-field="loan.start_at"/>
                </dd>

                <dt class="col-sm-3">End</dt>
                <dd class="col-sm-9">
                    <span t-field="loan.end_at"/>
                </dd>

                <dt class="col-sm-3">Duration (Hours)</dt>
                <dd class="col-sm-9">
                    <span t-field="loan.duration_hours"/>
                </dd>
            </dl>
        </t>
    </template>

    <template
        id="portal_home_equipment_link"
        inherit_id="portal.portal_my_home"
    >
        <xpath
            expr="//div[hasclass('o_portal_docs')]"
            position="inside"
        >
            <div
                class="mt-3"
                t-if="request.env['equipment.loan'].has_access('read')"
            >
                <a
                    href="/my/equipment-loans"
                    class="btn btn-outline-primary"
                >
                    Equipment Loans
                </a>
            </div>
        </xpath>
    </template>
</odoo>
```

الملف يعرّف:

1. صفحة القائمة.
2. صفحة التفاصيل.
3. رابطًا بسيطًا من الصفحة الرئيسية للـPortal.

نستخدم `t-field` لتنسيق حقول الإعارة، و`t-att-href` لبناء رابط التفاصيل.

وجود الرابط ليس مصدر الحماية؛ التحقق موجود في Controller وصلاحيات Model.

---

### 7. تسجيل Template في Manifest

أضف الملف التالي إلى قائمة `data` الحالية:

```python
"views/equipment_portal_templates.xml",
```

مثلًا، ضعه بعد ملفات Views الحالية، مع الاحتفاظ بملفات Reports وCron السابقة.

ملف `security/ir.access.csv` موجود أصلًا في Manifest، لذلك لا تضفه مرة ثانية.

---

### 8. تطبيق Upgrade

بعد حفظ الملفات:

1. أوقف Server التدريب.
2. أعد تشغيله لتحميل Controllers.
3. سجل الدخول بحساب يستطيع إدارة Modules.
4. افتح **Apps** واحذف Filter **Apps**.
5. ابحث عن `Equipment Training`.
6. نفّذ **Upgrade**.

نحتاج إلى Upgrade لتحميل Dependency والصلاحية وTemplates.

أما تعديل Python فيحتاج إلى إعادة تحميل Server أيضًا، كما تعلمنا في [[03 Modules & Addons]].

---

### 9. تجربة مسار JSON

سجل الدخول بحساب Equipment User أوEquipment Manager.

افتح في المتصفح:

```text
http://127.0.0.1:8069/equipment/status
```

ناتج توضيحي:

```json
{
    "service": "equipment",
    "available_count": 3
}
```

العدد الفعلي يعتمد على بيانات المستخدم وصلاحياته والشركات المتاحة في سياق الطلب.

لا يمثل الرقم جميع الأجهزة في Database بالضرورة.

---

### 10. تجهيز حسابي Portal للتجربة

إذا كانت لديك حسابات Portal وإعارات مرتبطة بها، يمكنك استخدامها.

ولتجهيز بيانات مستقلة، افتح Odoo Shell لقاعدة التدريب بصلاحيات الإدارة، كما في [[06 Records & Recordsets]].

المثال التالي ينشئ حسابين تجريبيين وجهازًا وإعارة لكل حساب.

يطلب Password من الطرفية بدل وضعها داخل الملاحظة:

```python
from getpass import getpass

from odoo import Command


accounts = [
    ("Portal Learner A", "equipment.portal.a"),
    ("Portal Learner B", "equipment.portal.b"),
]

user_model = env["res.users"].with_context(
    no_reset_password=True,
)

existing_users = user_model.with_context(
    active_test=False,
).search([
    ("login", "in", [login for _, login in accounts]),
])

if existing_users:
    raise ValueError(
        "Training accounts already exist. Reuse them instead."
    )

company = env.ref("base.main_company")
portal_group = env.ref("base.group_portal")

portal_users = env["res.users"]

for name, login in accounts:
    training_password = getpass(
        f"Password for {login}: "
    )

    if not training_password:
        raise ValueError("Enter a training password.")

    portal_users |= user_model.create({
        "name": name,
        "login": login,
        "email": False,
        "password": training_password,
        "tz": "UTC",
        "group_ids": [
            Command.set([portal_group.id]),
        ],
        "company_id": company.id,
        "company_ids": [
            Command.set([company.id]),
        ],
    })

asset = env["equipment.asset"].create({
    "name": "Portal Training Equipment",
    "serial_number": "HTTP-023-PORTAL",
    "state": "available",
    "company_id": company.id,
})

loans = env["equipment.loan"].create([
    {
        "name": f"Portal Loan {index}",
        "company_id": company.id,
        "requested_by_id": portal_user.id,
        "asset_id": asset.id,
        "start_at": "2026-02-10 09:00:00",
        "end_at": "2026-02-10 11:00:00",
    }
    for index, portal_user in enumerate(
        portal_users,
        start=1,
    )
])

env.cr.commit()

print([
    (loan.requested_by_id.login, loan.id)
    for loan in loans
])
```

احتفظ بالـIDs المطبوعة لاختبار تبديل الروابط.

الحسابان يحملان مجموعة Portal فقط، ولا نمنحهما مجموعة Equipment User.

استخدمنا `no_reset_password=True` وتركنا Email فارغة لتجهيز الحسابات محليًا دون دعوات بريدية.

> [!note] Commit هنا
> نستخدم Commit صراحةً لحفظ بيانات التجربة المنشأة من Shell.
>
> لا تنقل هذا الاستدعاء إلى Controller؛ إطار الطلب يدير Transaction الخاصة بها.
>
> إذا حدث خطأ أثناء التجهيز قبل Commit، استخدم `env.cr.rollback()` قبل إعادة المحاولة.

---

### 11. تجربة Portal

افتح جلسة متصفح منفصلة وسجل الدخول باستخدام:

```text
equipment.portal.a
```

ثم افتح:

```text
http://127.0.0.1:8069/my
```

اضغط **Equipment Loans**، أوانتقل مباشرةً إلى:

```text
http://127.0.0.1:8069/my/equipment-loans
```

المتوقع:

- ظهور إعارة الحساب الأول.
- إمكانية فتح تفاصيلها.
- ظهور الفترة والمدة.
- عدم ظهور إعارة الحساب الثاني.

كرر التجربة بحساب:

```text
equipment.portal.b
```

المتوقع أن يرى إعارته الخاصة.

---

### 12. اختبار تغيير ID

أثناء تسجيل الدخول بالحساب الأول، غيّر ID في رابط التفاصيل إلى ID إعارة الحساب الثاني:

```text
http://127.0.0.1:8069/my/equipment-loans/OTHER_LOAN_ID
```

استبدل `OTHER_LOAN_ID` بالرقم الحقيقي الذي طبعته Shell.

المتوقع:

```text
404 Not Found
```

كذلك، إضافة Query String مثل:

```text
?requested_by_id=OTHER_USER_ID
```

لا تغير صاحب السجلات المعروضة؛ Controller تستخرج المستخدم من Environment، ولا تستخدم هذا المعامل.

هذا هو اختبار **ID Tampering** الأساسي في المثال.

---

## Controller Inheritance

### توسيع Controller من Module أخرى

يمكن توريث Controller باستخدام Python Inheritance.

في Module أخرى تعتمد على `equipment_training`، يكون النمط:

```python
from odoo import http
from odoo.addons.equipment_training.controllers.main import (
    EquipmentController,
)


class EquipmentControllerExtension(EquipmentController):

    @http.route()
    def equipment_status(self, **kwargs):
        return super().equipment_status(**kwargs)
```

`@http.route()` دون معاملات تحافظ على إعدادات Route الموروثة.

عند Override، أعد استخدام Route decorator وفق آلية Controllers، بدل تطبيق قواعد Model Inheritance عليها.

ويمكن تمرير إعدادات محددة لتعديلها، لكن يجب مراجعة أثر تغيير المصادقة أوطرق الطلب المسموحة.

---

## Testing & Troubleshooting

### حالات التحقق

| الحالة | النتيجة المتوقعة |
| --- | --- |
| Equipment User يفتح مسار الحالة | JSON بعدد الأجهزة المتاحة المقروءة |
| Portal User يفتح مسار الحالة | `403` لعدم امتلاك قراءة الأجهزة |
| Portal User يفتح قائمته | إعارات حسابه فقط |
| فتح إعارة الحساب نفسه | صفحة التفاصيل |
| تغيير ID إلى إعارة مستخدم آخر | `404` |
| ID غير موجود | `404` |
| إعارة خارج قيود الشركة | لا تظهر ضمن السجلات المسموحة |
| طلب غير مسجل على Route من نوع `user` | معالجة المصادقة وتوجيه تسجيل الدخول بحسب السياق |
| إرسال POST إلى مسارات GET في المثال | رفض طريقة الطلب |

افحص Status Code والمحتوى، وليس شكل الصفحة فقط.

---

### Route تعيد 404 دائمًا

تحقق من:

1. وجود Import في `controllers/__init__.py`.
2. استيراد `controllers` من جذر Module.
3. إعادة تشغيل Server.
4. تثبيت Module في Database الصحيحة.
5. صحة المسار المكتوب.

وفي صفحة تفاصيل الإعارة، قد تكون `404` نتيجة متوقعة لأن السجل لا يطابق صاحب الحساب أوقيود الوصول.

---

### قائمة Portal فارغة

راجع:

- قيمة `requested_by_id`.
- المستخدم الذي سجل الدخول فعليًا.
- الشركة المسموحة للحساب.
- صف صلاحية Portal داخل `ir.access.csv`.
- تنفيذ Upgrade بعد إضافة الصلاحية.

إضافة رابط إلى Portal لا تغير ملكية السجلات الموجودة.

---

### خطأ صلاحيات عند عرض Field مرتبطة

قد يملك المستخدم قراءة الإعارة، دون قراءة الجهاز المرتبط بها.

مثلًا، عرض:

```xml
<span t-field="loan.asset_id.name"/>
```

يحتاج إلى مراعاة الوصول إلى الجهاز أيضًا.

لا تعالج ذلك بإضافة `sudo()` إلى الصفحة كلها.

حدد أولًا البيانات التي يحتاجها المستخدم، ثم صمم الصلاحيات أوالبيانات المعروضة بما يناسبها.

---

### Client يحصل على HTML بدل JSON

قد يكون HTML صفحة تسجيل دخول أوصفحة خطأ.

`auth="user"` تعتمد على مصادقة الطلب؛ العميل الخارجي الذي لا يملك Session صالحة قد لا يصل إلى Method أصلًا.

افحص Redirects وStatus Code وContent-Type.

وسنتناول خيارات المصادقة المناسبة للعملاء الخارجيين في [[25 APIs & Integrations]].

---

### CSRF Error

في Form ترسل POST، تحقق من Token وSession قبل تغيير إعدادات Route.

وتذكر أن:

- CSRF ليست بديلًا عن صلاحيات السجلات.
- تسجيل الدخول لا يلغي الحاجة إلى فحص المدخلات.
- الحقول المخفية قابلة للتعديل من العميل.
- CORS لا تمنح صلاحيات قراءة Business Records.

---

## Important Notes

- Controllers نقاط دخول للطلبات؛ احتفظ بالـBusiness Logic القابلة لإعادة الاستخدام داخل Models.
- `auth="user"` تتحقق من تسجيل الدخول، وتبقى صلاحيات ORM مطلوبة.
- `auth="public"` قد تستخدم Public User عند غياب مستخدم مسجل.
- في Odoo 20، استخدم `jsonrpc` لتسمية Route الخاصة ببروتوكول JSON-RPC.
- إرجاع JSON عادية يمكن تنفيذه عبر `http` و`make_json_response()`.
- تعريف Bearer Route في Odoo 20 يحتاج إلى `bearer_scope`.
- تحويل ID إلى Integer لا يثبت صلاحية الوصول إلى Record.
- ضع قيود ملكية Portal على Model، ثم حدد محتوى الصفحة داخل Controller.
- حدد Fields التي يحتاجها الرد، وراجع صلاحيات Records المرتبطة.
- GET في المثال للقراءة؛ لا تربط زيارة الرابط بتغيير بيانات العمل.
- لا تستخدم `sudo()` أوتعطيل CSRF كحل تلقائي للأخطاء.
- اختبر بحسابي Portal مختلفين، مع تبديل IDs ومراجعة قيود الشركات.
- عند Override لـController Method، أعد Route decorator المناسبة.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[24 OWL & Frontend]]

---

## Sources

- [Odoo 20 Documentation — Web Controllers](https://www.odoo.com/documentation/20.0/developer/reference/backend/http.html)
- [Odoo 20 Documentation — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)
- [Odoo 20 — Routing and Authentication Options: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/http/routing_map.py)
- [Odoo 20 — HTTP Request and Response Helpers: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/http/requestlib.py)
- [Odoo 20 — HTTP Dispatching and CSRF: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/http/dispatcher.py)
- [Odoo 20 — Authentication Implementation: Official Source](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_http.py)
- [Odoo 20 — Portal Controllers: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/portal/controllers/portal.py)
- [Odoo 20 — Portal Templates: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/portal/views/portal_templates.xml)
- [Odoo 20 — Portal Mixin: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/portal/models/portal_mixin.py)
- [Odoo 20 — User Invitation Behavior: Official Source](https://github.com/odoo/odoo/blob/20.0/addons/auth_signup/models/res_users.py)