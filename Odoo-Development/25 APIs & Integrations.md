---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 25 APIs & Integrations

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Integration Fundamentals|Integration Fundamentals]]
- [[#JSON-2 API|JSON-2 API]]
- [[#Practical Example|Practical Example]]
- [[#Transactions & Business Operations|Transactions & Business Operations]]
- [[#Errors, Retries & Idempotency|Errors, Retries & Idempotency]]
- [[#Webhooks & Outgoing Integrations|Webhooks & Outgoing Integrations]]
- [[#Verification|Verification]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

تعلمنا في [[24 OWL & Frontend]] كيف تتصل واجهة Odoo بالـModels الموجودة على Server.

لكن قد نحتاج إلى الوصول إلى Odoo من برنامج مستقل، مثل نظام لإدارة الأجهزة، أو خدمة شحن، أو أداة لتحليل البيانات.

هذا الربط يسمى **Integration**.

التكامل لا يقتصر على إرسال Request واستلام Response؛ يجب أن نحدد أيضًا صلاحيات الاتصال، وملكية البيانات، وطريقة التعامل مع الفشل وتكرار الطلبات.

في هذا الشابتر سنستخدم **JSON-2 API** لقراءة الأجهزة المتاحة من برنامج Python مستقل، ثم ندرس القرارات المطلوبة عند بناء تكامل ينشئ البيانات أو يعدلها.

---

## Integration Fundamentals

### Odoo بوصفه API Provider أوAPI Client

يمكن أن يكون اتجاه الاتصال إلى Odoo أو منه.

| الدور | المعنى | مثال |
| --- | --- | --- |
| API Provider | Odoo يستقبل الطلبات | نظام خارجي يقرأ الأجهزة المتاحة |
| API Client | Odoo يرسل الطلبات | Odoo يرسل طلب شحن إلى خدمة خارجية |

وقد يجمع التكامل بين الاتجاهين.

```text
External System
       ↓
    Odoo API
       ↓
Business Models
```

وفي الاتجاه الآخر:

```text
Business Operation
       ↓
      Odoo
       ↓
External API
```

---

### Source of Truth

قبل كتابة الكود، حدد النظام المسؤول عن كل معلومة.

مثلًا:

| المعلومة | النظام المسؤول |
| --- | --- |
| الرقم التسلسلي للجهاز | Odoo |
| حالة شحنة الجهاز | نظام الشحن |
| رقم الطلب في النظام الخارجي | النظام الخارجي |

إذا عدّل النظامان المعلومة نفسها، فستحتاج إلى قاعدة واضحة لحل التعارض.

من الأسئلة المهمة:

- أي تعديل يُقبل عند اختلاف القيم؟
- هل المزامنة في اتجاه واحد أم اتجاهين؟
- كيف نمثل الحذف والأرشفة؟
- كيف نربط Record في Odoo بنظيرتها في النظام الآخر؟

وثّق هذه القرارات قبل بناء المزامنة.

---

## JSON-2 API

### شكل الطلب

توفر Odoo 20 واجهة JSON-2 لاستدعاء Model Methods عبر HTTP.

صيغة المسار:

```text
POST /json/2/<model>/<method>
```

مثال لقراءة الأجهزة:

```text
POST /json/2/equipment.asset/search_read
```

نرسل Arguments داخل JSON Object بأسماء Parameters:

```json
{
    "domain": [
        ["state", "=", "available"]
    ],
    "fields": [
        "name",
        "serial_number",
        "state"
    ],
    "limit": 5
}
```

القيمة `available` هي الحالة التقنية الصحيحة في مشروع التدريب.

أما `active` فهي Field للأرشفة، وليست قيمة من قيم `state`.

في JSON-2 لا نضيف Envelope تحتوي `jsonrpc` و`method` و`params` كما في بروتوكول JSON-RPC القديم.

---

### Authentication

نرسل API Key داخل Header:

```text
Authorization: bearer <API_KEY>
```

ويحدد المفتاح المستخدم الذي ستنفذ العملية باسمه.

يمكن تحديد Database باستخدام:

```text
X-Odoo-Database: odoo20_training
```

الحاجة إلى هذا Header تعتمد على إعداد الاستضافة واختيار Database. سنرسله صراحةً في المثال.

نستخدم أيضًا:

```text
Content-Type: application/json
```

عند استخدام `requests.post(..., json=payload)` تضبط المكتبة Content Type المناسب.

---

### Authentication وAuthorization

وجود API Key صحيحة يثبت هوية المستخدم، لكنه لا يمنحه صلاحيات إضافية.

إذا كان الحساب يستطيع قراءة الأجهزة فقط، فلا يفترض أن يستطيع إنشاءها أو تعديلها عبر API.

راجع قواعد الوصول التي بنيناها في [[11 Security & Multi-Company]].

للتكامل المستمر، استخدم حسابًا مخصصًا، مثل:

```text
Equipment Integration
```

وامنحه الصلاحيات والشركات التي تحتاجها المهمة فقط.

> [!important] Public Methods
> أي Method عامة نسمح باستدعائها خارجيًا يجب أن تتحقق من مدخلاتها ومن متطلبات العملية.
>
> لا تفترض أن المستخدم وصل إليها من Button محددة، أو أن Record IDs المرسلة مسموح بها لمجرد وصولها داخل Request.

---

### شكل الاستجابة

عند النجاح، تعيد JSON-2 القيمة التي أرجعتها Method مباشرةً.

مثلًا، `search_read` تعيد List:

```json
[
    {
        "id": 17,
        "name": "Training Laptop",
        "serial_number": "TRAIN-API-001",
        "state": "available"
    }
]
```

هذه بيانات توضيحية؛ IDs الفعلية تحددها Database.

عندما لا توجد Records مطابقة:

```json
[]
```

هذه نتيجة ناجحة، وليست خطأ.

أما الأخطاء، فتستخدم HTTP Status مناسبة، ويمكن أن تتضمن JSON تحتوي اسم Exception ورسالتها.

---

### API وبيئة الاستضافة

تذكر وثائق Odoo قيودًا على الوصول الخارجي ضمن خطط الاستضافة التجارية، ومنها اشتراط خطة **Custom** بدل **Standard** أو**One App Free**.

راجع شروط البيئة التي ستنشر عليها التكامل.

في بيئة Self-hosted، تحقق من إعداد Server وتوفر Endpoint؛ لا تعامل أسماء الخطط التجارية كإعداد برمجي داخل مشروعك المحلي.

---

## Practical Example

### 1. تحديد المطلوب

سنكتب سكربت Python مستقلًا يقرأ حتى خمسة أجهزة متاحة من:

```text
equipment.asset
```

ويعرض:

- اسم الجهاز.
- الرقم التسلسلي.
- الحالة.

يعتمد المثال على:

- تشغيل Odoo.
- وجود Database التدريب `odoo20_training`.
- تثبيت Module `equipment_training`.
- استخدام حساب يملك صلاحية قراءة الأجهزة.
- إنشاء API Key لهذا الحساب.

سنستخدم حسابًا من مجموعة **Equipment User** التي جهزناها سابقًا.

---

### 2. إنشاء API Key

سجل الدخول بالحساب الذي سيستخدمه التكامل.

من إعدادات الحساب، انتقل إلى:

```text
Preferences
    ↓
Account Security
    ↓
New API Key
```

اكتب وصفًا يوضح الغرض، مثل:

```text
Equipment Training Reader
```

واختر مدة صلاحية مناسبة للتجربة، ثم أنشئ المفتاح واحفظه في مكان آمن.

قيمة المفتاح تعرض عند إنشائها؛ تعامل معها كسر يمكن إلغاؤه واستبداله عند الحاجة.

لا تشارك المفتاح مع السكربتات التي لا تحتاج إلى صلاحياته.

---

### 3. تجهيز ملف السكربت

داخل مجلد مشروع التطوير، أضف مجلدًا للبرامج الخارجية:

```text
odoo20-dev/
├── .venv/
├── odoo/
├── custom-addons/
├── config/
└── integrations/
    └── read_equipment.py
```

السكربت هنا برنامج مستقل يتصل بـOdoo عبر HTTP.

تأكد من توفر مكتبة `requests` داخل Python Environment التي ستشغله.

من مجلد `odoo20-dev`:

#### Fedora

```bash
.venv/bin/python -m pip install requests
```

#### Windows

```powershell
.\.venv\Scripts\python.exe -m pip install requests
```

---

### 4. كتابة Python Client

داخل:

```text
integrations/read_equipment.py
```

اكتب:

```python
import json
import os
from getpass import getpass

import requests


def main():
    try:
        base_url = os.environ["ODOO_URL"].rstrip("/")
        database = os.environ["ODOO_DB"]
    except KeyError as error:
        raise SystemExit(
            f"Missing environment variable: {error.args[0]}"
        ) from None

    api_key = (
        os.environ.get("ODOO_API_KEY")
        or getpass("Odoo API key: ")
    ).strip()

    if not api_key:
        raise SystemExit("An API key is required.")

    headers = {
        "Authorization": f"bearer {api_key}",
        "X-Odoo-Database": database,
        "User-Agent": "equipment-training-client/1.0",
    }

    payload = {
        "domain": [
            ["state", "=", "available"],
        ],
        "fields": [
            "name",
            "serial_number",
            "state",
        ],
        "order": "id",
        "limit": 5,
    }

    try:
        response = requests.post(
            f"{base_url}/json/2/equipment.asset/search_read",
            headers=headers,
            json=payload,
            timeout=(5, 20),
            allow_redirects=False,
        )
    except requests.Timeout:
        raise SystemExit(
            "The request timed out. Check connectivity before retrying."
        ) from None
    except requests.ConnectionError:
        raise SystemExit(
            "Could not connect to Odoo. Check the URL and server."
        ) from None
    except requests.RequestException:
        raise SystemExit(
            "The HTTP request could not be completed."
        ) from None

    if 300 <= response.status_code < 400:
        raise SystemExit(
            "Unexpected redirect. Check ODOO_URL and proxy configuration."
        )

    try:
        response.raise_for_status()
    except requests.HTTPError:
        error_name = "HTTPError"

        try:
            error_data = response.json()

            if isinstance(error_data, dict):
                error_name = error_data.get("name") or error_name
        except ValueError:
            pass

        raise SystemExit(
            f"Odoo rejected the request: "
            f"HTTP {response.status_code} — {error_name}"
        ) from None

    try:
        records = response.json()
    except ValueError:
        raise SystemExit(
            "Odoo returned a response that is not valid JSON."
        ) from None

    if not isinstance(records, list):
        raise SystemExit(
            "Unexpected response: search_read should return a list."
        )

    print(json.dumps(records, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    main()
```

السكربت ينفذ طلبًا واحدًا، ويتحقق من HTTP Status ومن شكل النتيجة.

إذا لم يجد `ODOO_API_KEY` ضمن Environment Variables، يطلب المفتاح باستخدام إدخال مخفي.

لا يطبع المفتاح أو Headers أو Traceback المرسلة من Server.

---

### 5. ضبط الاتصال وتشغيل السكربت

شغّل Server التدريب بالطريقة المعتادة أولًا.

ثم افتح Terminal أخرى داخل مجلد `odoo20-dev`.

#### Fedora

```bash
export ODOO_URL='http://127.0.0.1:8069'
export ODOO_DB='odoo20_training'

.venv/bin/python integrations/read_equipment.py
```

#### Windows

```powershell
$env:ODOO_URL = "http://127.0.0.1:8069"
$env:ODOO_DB = "odoo20_training"

.\.venv\Scripts\python.exe integrations\read_equipment.py
```

أدخل API Key عند طلبها.

نستخدم HTTP هنا للاتصال المحلي بعنوان Loopback. عند الاتصال ببيئة بعيدة، استخدم HTTPS مع التحقق الطبيعي من شهادة Server.

الناتج المتوقع هو JSON List تحتوي حتى خمسة أجهزة متاحة ومسموحًا للحساب بقراءتها.

---

### 6. فهم خيارات الطلب

| الخيار | سبب استخدامه |
| --- | --- |
| `json=payload` | إرسال Body بصيغة JSON |
| `timeout=(5, 20)` | تحديد مهلة الاتصال ومهلة انتظار القراءة |
| `allow_redirects=False` | كشف Redirect غير متوقعة بدل متابعتها تلقائيًا |
| `fields` | طلب البيانات التي نحتاجها فقط |
| `limit` | تحديد حجم النتيجة في التجربة |
| `order` | جعل ترتيب النتائج واضحًا |

قيم Timeout ليست ضمانًا بأن الزمن الإجمالي للعملية لن يتجاوز مجموع الرقمين.

كما أن `limit=5` تعني أن المثال يقرأ جزءًا من النتائج. المزامنة الكاملة تحتاج إلى Pagination وخطة واضحة لتتبع البيانات الجديدة والمعدلة.

---

## Transactions & Business Operations

### كل Request لها Transaction مستقلة

طلبات JSON-2 لا تشترك في Transaction واحدة.

```text
Request A
    ↓
Transaction A

Request B
    ↓
Transaction B
```

عند نجاح Request تثبت تغييراتها، وعند فشلها تتراجع تغييرات Transaction الخاصة بها.

إذا قرأت Record في Request، ثم عدلتها في Request أخرى، فقد تكون حالتها تغيرت بين الطلبين.

---

### جمع العملية داخل Model Method

افترض أن النظام الخارجي يريد بدء استخدام جهاز:

```text
التحقق من الصلاحية
        ↓
التحقق من الحالة الحالية
        ↓
تطبيق الانتقال
```

الأفضل أن يستدعي Business Method تنفذ العملية كوحدة واحدة على Server، مثل `action_start_use()` التي بنيناها في [[15 Business Logic]].

شكل مسارها:

```text
POST /json/2/equipment.asset/action_start_use
```

Methods التي تعمل على Records محددة تستقبل IDs ضمن Body.

هذه IDs تُستخرج من نتائج أو Mapping موثوقة، ثم يتحقق Server من السماح بالعملية.

لا تجعل النظام الخارجي يقرأ الحالة ثم يقرر وحده أن التعديل أصبح مسموحًا.

> [!note] Atomicity وConcurrency
> جمع العملية في Method واحدة يحدد حدود Transaction، لكنه لا يحل تلقائيًا كل تعارض بين عمليات متزامنة.
>
> بعض القواعد تحتاج أيضًا إلى Constraints أو آلية مناسبة للتعامل مع التزامن، كما ناقشنا في [[17 Constraints]].

لا تضف `commit()` يدويًا إلى Business Method لتثبيت جزء منها؛ دع Framework يدير حدود Transaction المعتادة.

---

## Errors, Retries & Idempotency

### تصنيف النتائج

لا تتعامل مع كل نتيجة غير متوقعة بالطريقة نفسها.

| النتيجة | معناها | التصرف المناسب |
| --- | --- | --- |
| List فارغة | الطلب نجح دون نتائج مطابقة | التعامل معها كنتيجة عادية |
| Authentication Failure | المفتاح غير مقبول أو منتهي | مراجعة المفتاح والحساب |
| Access Error | العملية غير مسموحة | مراجعة الصلاحيات |
| Validation Error | البيانات أو العملية تخالف Rule | تصحيح الطلب |
| Route أوModel غير موجودة | خطأ مسار أوتجهيز | مراجعة Endpoint وModule |
| Timeout | لم تصل النتيجة ضمن المهلة | تحديد حالة العملية قبل إعادة الكتابة |
| Network Error | تعذر الاتصال أوانقطع | تطبيق سياسة إعادة المحاولة المناسبة |
| Server Error | فشل على Server | فحص الخطأ قبل اتخاذ قرار الإعادة |

لا تعتمد على HTTP Status وحدها لتفسير كل Business Error؛ افحص نوع الخطأ وسياق الطلب أيضًا.

---

### Timeout لا تثبت فشل الكتابة

قد يحدث هذا التسلسل:

```text
Client ترسل طلب إنشاء
        ↓
Odoo تنشئ Record وتثبت Transaction
        ↓
ينقطع الاتصال قبل وصول Response
        ↓
Client تستقبل Timeout
```

إذا أعادت Client طلب الإنشاء مباشرةً، فقد تنشئ Record ثانية.

من جهة Client، النتيجة في هذه الحالة **غير معروفة**، وليست فشلًا مؤكدًا.

---

### Idempotency

تعني Idempotency أن تكرار الطلب المنطقي نفسه لا ينتج أثرًا إضافيًا غير مطلوب.

مثال تصميم:

```text
Source System: equipment_gateway
Request ID: request-00042
```

يمكن أن يحتفظ التكامل بسجل للطلب ونتيجته، مع قيد تفرد على هوية الطلب ضمن نطاقها المناسب.

عند تكرار الطلب:

1. يتحقق Server من هوية الطلب.
2. يتحقق أن المحتوى يطابق العملية السابقة.
3. يعيد النتيجة المعروفة عند اكتمالها سابقًا.
4. ينفذ العملية مرة واحدة عند قبولها لأول مرة.

قيد التفرد وحده يمنع بعض التكرار، لكن منطق التكامل يجب أن يحدد كيفية التعامل مع الطلب المكرر أوالمتزامن.

لا تفترض أن JSON-2 توفر هذا السلوك لمجرد إرسال Header باسم `Idempotency-Key`؛ يجب أن يدعم تصميم العملية هذا المفتاح ويستخدمه.

> [!important] هويات مختلفة
> External Request ID الخاصة بالتكامل ليست تلقائيًا External ID في Odoo.
>
> External IDs التي درسناها في [[10 Data Files & External IDs]] لها آلية مستقلة.

---

### Retry Policy

حدد سياسة إعادة المحاولة وفق طبيعة العملية.

| العملية | الاعتبار الأساسي |
| --- | --- |
| قراءة عادية | يمكن غالبًا إعادة المحاولة بعد خطأ مؤقت |
| إنشاء أوعملية ذات أثر | تحتاج إلى ضمان مناسب ضد التكرار |
| طلب مرفوض بسبب صلاحية | الإعادة بالمعلومات نفسها لن تصلح السبب |
| طلب ببيانات غير صحيحة | صحح المدخلات قبل الإعادة |

للتكامل المستمر، ضع حدًا لعدد المحاولات، واستخدم فواصل متزايدة، وسجل الفشل النهائي للمراجعة.

ولا تفترض أن استخدام HTTP `POST` يعني دائمًا أن العملية تعدل البيانات؛ `search_read` تستخدمه أيضًا في JSON-2.

---

## Webhooks & Outgoing Integrations

### Webhooks

Webhook إشعار يرسله نظام إلى نظام آخر عند حدوث Event.

قد يستقبل Odoo الإشعار:

```text
External Event
      ↓
Webhook Request
      ↓
Odoo
```

وقد يرسله إلى جهة أخرى.

عند بناء Webhook مخصصة، حدد:

- طريقة التحقق من المرسل وفق بروتوكول Provider.
- Event ID لمنع معالجة الحدث نفسه أكثر من مرة.
- البيانات التي يسمح الحدث بتعديلها.
- التعامل مع الأحداث المتأخرة أوغير المرتبة.
- سياسة المحاولات والردود.
- طريقة حفظ حالة المعالجة.

إذا كان Provider يستخدم Signature على Raw Body، فتحقق منها وفق مواصفاته قبل تحويل المحتوى أوتنفيذ العملية.

استخدم ما تعلمناه في [[23 HTTP Controllers & Portal]] عند بناء Route مخصصة.

وجود JSON داخل Request لا يجعلها طلب JSON-2؛ بروتوكول Webhook يحدده النظام المرسل.

---

### Odoo يستدعي External API

عندما يرسل Odoo طلبًا خارجيًا، حدد Timeout، وتحقق من HTTP Status وشكل Response، وحدد طريقة حفظ النتيجة أوالفشل.

انتبه خصوصًا إلى اختلاف حدود Transaction:

```text
نجاح الطلب الخارجي
        ↓
فشل Transaction المحلية لاحقًا
```

تراجع Transaction داخل Odoo لا يلغي الأثر الذي حدث في النظام الخارجي تلقائيًا.

كما أن انتظار شبكة بطيئة داخل عملية حساسة يطيل مدة Transaction وقد يؤخر أعمالًا أخرى.

---

### Outbox Pattern

يمكن استخدام **Outbox Pattern** عندما نريد تسجيل تغيير محلي ثم إرسال أثره إلى نظام خارجي.

هذا نمط تصميم نضيفه إلى التكامل:

```text
Business Transaction
        ↓
حفظ التغيير المحلي
+
إنشاء Outbox Record
        ↓
Commit
        ↓
Background Worker
        ↓
إرسال الطلب الخارجي
        ↓
حفظ النتيجة أوالمحاولة التالية
```

يجب إنشاء Outbox Record ضمن Transaction نفسها التي تحفظ التغيير المحلي.

ثم يمكن استخدام Scheduled Action لمعالجة الطلبات، وفق المبادئ التي درسناها في [[22 Automation & Scheduled Actions]].

إذا نجح الإرسال وتعطل Worker قبل تسجيل النجاح، فقد يعيد الإرسال لاحقًا؛ لذلك تظل Idempotency مهمة لدى الطرف المستقبل.

Outbox لا تحول قاعدتي البيانات إلى Transaction واحدة.

---

## Verification

### اختبار مثال القراءة

جرب الحالات التالية داخل Database التدريب:

| التجربة | الناتج المتوقع |
| --- | --- |
| مفتاح صحيح وحساب مخول | List بالأجهزة المسموح بها |
| لا توجد أجهزة متاحة | List فارغة |
| مفتاح غير صحيح | فشل Authentication |
| حساب بلا صلاحية قراءة الأجهزة | رفض الوصول |
| إيقاف Server | خطأ اتصال |
| مسار API غير صحيح | خطأ HTTP |
| أكثر من خمسة أجهزة مطابقة | إرجاع خمسة كحد أقصى |

إذا ظهر `404`، تحقق من:

- العنوان والمسار.
- اختيار Database.
- توفر Model داخلها.
- توفر Module `rpc` التي تقدم JSON-2 في الفرع الحالي.
- إعداد Reverse Proxy إذا كان الاتصال يمر عبره.

---

### اختبار التكامل الذي يكتب البيانات

عند إضافة عمليات إنشاء أوتعديل، اختبر أيضًا:

- إرسال الطلب نفسه مرتين.
- وصول طلبين متزامنين بالهوية نفسها.
- اختلاف المحتوى مع استخدام Request ID سابقة.
- انقطاع الاتصال بعد تنفيذ العملية.
- بيانات تخالف Constraint.
- محاولة الوصول إلى Record خارج نطاق الحساب.
- فشل النظام الخارجي بعد حفظ البيانات المحلية.

توسع في الاختبارات الآلية في [[26 Testing]]، واحتفظ بمعلومات تشخيص مناسبة دون تسجيل API Keys.

---

## Important Notes

- JSON-2 تستدعي Model Methods عبر `POST` وبـBody من نوع JSON Object.
- نتيجة النجاح هي القيمة المعادة من Method مباشرةً، وليست Envelope من بروتوكول JSON-RPC القديم.
- API Key تحدد الحساب؛ صلاحياته وقواعد الوصول تظل مطبقة.
- Public Method تحتاج إلى التحقق المناسب من المدخلات والصلاحيات.
- استخدم حسابًا مخصصًا للتكامل المستمر، بصلاحيات وشركات محددة.
- لا تحفظ API Keys داخل Git أوFrontend أوLogs.
- خطط لانتهاء صلاحية المفاتيح واستبدالها وإلغائها.
- استخدم HTTPS للاتصال البعيد، ولا تعطل التحقق من الشهادة لتجاوز خطأ إعداد.
- كل JSON-2 Request لها Transaction مستقلة.
- Timeout بعد الكتابة قد تعني أن النتيجة غير معروفة.
- إعادة المحاولة تحتاج إلى سياسة تناسب أثر العملية.
- قيد التفرد جزء من تصميم منع التكرار، وليس التصميم كله.
- Webhooks قد تتكرر أوتصل بترتيب مختلف.
- Outbox تساعد على حفظ العمل المطلوب إرساله، لكنها لا تضمن وحدها عدم تكرار الأثر الخارجي.
- مثال القراءة محدود بخمسة Records؛ التكامل الكامل يحتاج إلى Pagination ومراقبة للفشل.
- راجع شروط الاستضافة وتوفر API في البيئة المستهدفة.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[26 Testing]]

---

## Sources

- [Odoo 20 Documentation — External JSON-2 API](https://www.odoo.com/documentation/20.0/developer/reference/external_api.html)
- [Odoo 20 Documentation — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)
- [Odoo 20 Documentation — Web Controllers](https://www.odoo.com/documentation/20.0/developer/reference/backend/http.html)
- [Odoo 20 Documentation — Coding Guidelines: Never Commit the Transaction](https://www.odoo.com/documentation/20.0/contributing/development/coding_guidelines.html#never-commit-the-transaction)
- [Odoo 20 Documentation — Webhooks](https://www.odoo.com/documentation/20.0/applications/studio/automated_actions/webhooks.html)
- [Odoo 20 Official Source — JSON-2 Controller](https://github.com/odoo/odoo/blob/20.0/addons/rpc/controllers/json2.py)
- [Odoo 20 Official Source — RPC Module Manifest](https://github.com/odoo/odoo/blob/20.0/addons/rpc/__manifest__.py)