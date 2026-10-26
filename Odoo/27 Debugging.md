---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 27 Debugging

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Debugging Workflow|Debugging Workflow]]
- [[#Logs & Tracebacks|Logs & Tracebacks]]
- [[#Browser Tools|Browser Tools]]
- [[#Common Errors|Common Errors]]
- [[#Practical Examples|Practical Examples]]
- [[#Server Debugging Tools|Server Debugging Tools]]
- [[#Verification|Verification]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

قد يظهر الخطأ أثناء تشغيل Odoo، أوتحميل Module، أوفتح View، أوتنفيذ Button.

وقد تعمل العملية دون Exception، لكنها تعيد نتيجة غير صحيحة.

**Debugging** هي عملية تحديد السبب الذي أنتج هذا السلوك، ثم إصلاحه والتحقق من النتيجة.

مثلًا:

- ظهور `RPC_ERROR` لا يحدد وحده سبب المشكلة.
- ظهور `AccessError` قد يعني أن الحماية تعمل كما صممناها.
- ظهور عدد أجهزة يساوي صفرًا قد ينتج عن Domain غير صحيحة.
- عدم ظهور تعديل قد يعني أن Process تعمل بكود مختلف عن الملف الذي نعدله.

في هذا الشابتر سنستخدم مشروع التدريب لتعلم طريقة ثابتة للتشخيص، بدل تجربة تعديلات متفرقة دون دليل.

---

## Debugging Workflow

### 1. وصف المشكلة وإعادة إنتاجها

ابدأ بوصف قابل للتنفيذ:

```text
الحساب:
Equipment User

الخطوات:
فتح Equipment
ثم فتح جهاز
ثم محاولة تنفيذ عملية محددة

المتوقع:
نتيجة معروفة

الفعلي:
رسالة خطأ أو نتيجة مختلفة
```

سجل أيضًا:

- Database المستخدمة.
- الشركة أوالشركات المفعلة.
- Record التي ظهرت عليها المشكلة.
- آخر تعديل له علاقة بالسلوك.
- هل المشكلة تظهر دائمًا أم في حالات محددة؟

كلما كانت خطوات إعادة الإنتاج واضحة، أصبح فحص الإصلاح أسهل.

---

### 2. تحديد الطبقة التي فشلت

| العَرَض | نقطة البداية |
| --- | --- |
| فشل تشغيل Server | Terminal وStartup Logs |
| فشل Upgrade | Server Logs وسياق تحميل الملف |
| رفض عملية على Record | Business Method والصلاحيات |
| فشل فتح View | تعريف View وModel وFields |
| خطأ OWL أوTemplate | Browser Console والـAssets |
| فشل طلب من المتصفح | Network ثم Server Logs |
| نتيجة صحيحة تقنيًا لكنها غير متوقعة | Domain وContext والبيانات |
| فشل تكامل خارجي | Request وResponse وحالة العملية |

هذه نقاط بداية، وقد تحتاج المشكلة إلى تتبع أكثر من طبقة.

---

### 3. بناء فرضية واختبارها

بدل تعديل عدة أجزاء معًا، اكتب فرضية محددة:

> View تشير إلى Field موجودة على Model أخرى.

ثم اختبرها بفحص Model الخاصة بالـView وتعريف Field.

أو:

> Dashboard تستخدم قيمة حالة غير موجودة في مشروعنا.

ثم افحص Domain المرسلة.

المسار:

```text
ملاحظة
   ↓
فرضية
   ↓
فحص يثبتها أو ينفيها
   ↓
تعديل محدود
   ↓
إعادة التجربة
```

إذا لم تؤيد الأدلة الفرضية، غيّر الفرضية قبل تغيير الكود.

---

## Logs & Tracebacks

### أين نبحث عن Logs؟

في بيئة التطوير، تظهر Logs عادةً في Terminal التي شغلت منها Odoo.

إذا كانت Config تحدد `logfile`، فراجع الملف المحدد.

عند حدوث المشكلة:

1. حدد وقت المحاولة.
2. ابحث عن الرسائل المرتبطة بها.
3. اقرأ Exception وسياقها.
4. حدد الملف أوMethod التي قادت إلى الفشل.

وجود عدة Requests أوWorkers قد يجعل رسائل عمليات مختلفة متجاورة؛ اربط السجل بالمحاولة التي تحقق فيها.

---

### قراءة Traceback

الـTraceback تعرض سلسلة الاستدعاءات التي أدت إلى Exception.

ابحث عن:

| المعلومة | فائدتها |
| --- | --- |
| Exception Type | تحديد نوع الفشل |
| Exception Message | وصف السبب المباشر |
| File وLine | تحديد موضع التنفيذ |
| Model وRecord | فهم البيانات المستهدفة |
| الاستدعاءات السابقة | فهم كيف وصل التنفيذ إلى هذا الموضع |

ابدأ برسالة الخطأ المحددة، ثم ارجع إلى الاستدعاء المرتبط بكود Module الخاصة بك.

إذا كانت هناك Exceptions متسلسلة، فاقرأ العلاقة بينها؛ قد تكون الرسالة الأخيرة تغلف خطأ أكثر تحديدًا.

السطر الأخير ليس دائمًا وحده كافيًا، وأول رسالة في Log ليست دائمًا أصل المشكلة.

---

### `RPC_ERROR`

قد يعرض المتصفح:

```text
RPC_ERROR
```

بينما يوضح Server سببًا مثل:

```text
AccessError
ValidationError
ValueError
```

`RPC_ERROR` تصف فشل استدعاء من جهة Client، لكنها لا تحدد وحدها القاعدة أوالسطر الذي فشل.

افتح تفاصيل الخطأ، ثم راجع Server Logs المرتبطة بالطلب.

---

### `ParseError`

قد تظهر `ParseError` أثناء تحميل ملف XML حتى لو كانت Tags مغلقة بصورة صحيحة.

من الأسباب المحتملة:

- Field غير موجودة على Model الخاصة بالـView.
- External ID غير موجودة.
- Reference حُمّلت قبل تعريفها.
- XPath لا يطابق العنصر المستهدف.
- قيمة أوتعريف لا يقبله Odoo.

افحص اسم الملف، وسياق View، والرسالة الموجودة داخل الخطأ.

رقم السطر قد يشير إلى بداية Record أوالعنصر الذي كان Odoo يعالجه، وليس بالضرورة إلى أصغر جزء يحتاج إلى التعديل.

---

## Browser Tools

### Developer Mode

يمكن تفعيل Developer Mode من:

```text
Settings
   ↓
Developer Tools
   ↓
Activate the developer mode
```

تساعد الأدوات التقنية في معرفة معلومات مثل:

- Model المستخدمة.
- View الحالية.
- External IDs.
- Actions المرتبطة.

الخيارات المتاحة تختلف حسب الشاشة وصلاحيات المستخدم.

Developer Mode لا تمنح الحساب صلاحيات إضافية على Records، ولا تشغّل Python Debugger.

---

### Browser Console

استخدم Console عند تشخيص:

- أخطاء JavaScript.
- Imports غير المحملة.
- Registry Keys غير الموجودة.
- OWL Lifecycle Errors.
- Templates المفقودة.

إذا ظهر خطأ أثناء Rendering، اقرأ سببه والتفاصيل المرتبطة به؛ قد يكون الخطأ في تعبير داخل Template، أوفي قيمة وصلت إليها، أوفي فشل طلب سابق.

احتفظ بالخطأ الأصلي عند الفحص بدل الاكتفاء بالرسالة العامة التي تعرضها الشاشة.

---

### Network

افتح Network، ثم أعد تنفيذ العملية مرة واحدة.

افحص:

| الجزء | ما الذي تبحث عنه؟ |
| --- | --- |
| Request URL | هل وصل الطلب إلى المسار المتوقع؟ |
| Payload | هل Model وMethod وDomain صحيحة؟ |
| Status | هل فشل النقل أوالطلب؟ |
| Response | هل توجد نتيجة أم تفاصيل خطأ؟ |
| Timing | أين استغرق الطلب وقته؟ |

فرق بين:

```text
ملف JavaScript لم يُحمّل
```

وبين:

```text
JavaScript حمّلت، ثم رفض Server قراءة البيانات
```

لكل حالة سبب ومسار تشخيص مختلف.

> [!note] HTTP Status لا تكفي وحدها
> بعض استدعاءات JSON-RPC في Web Client قد تعيد HTTP ناجحة مع وجود خطأ داخل Response.
>
> أما JSON-2 التي استخدمناها في [[25 APIs & Integrations]] فلها تنسيق استجابة ومعالجة HTTP مختلفان.
>
> افحص البروتوكول وBody، إلى جانب Status.

---

### `debug=assets`

لتشخيص ملفات الواجهة، يمكن تفعيل Developer Mode with Assets.

مثال محلي:

```text
http://127.0.0.1:8069/odoo?debug=assets
```

يساعد هذا الوضع على تتبع ملفات Assets أثناء التطوير، ويعطل Minification المعتادة للملفات المعنية.

إذا كان الرابط يحتوي Query Parameters بالفعل، أضف الخيار مع الحفاظ عليها.

هذا الوضع لا يصلح Import أوTemplate مفقودة؛ دوره جعل التشخيص أوضح.

---

## Common Errors

### External ID غير موجودة

مثال:

```text
External ID not found:
equipment_training.action_equipment
```

راجع بالترتيب:

1. الاسم التقني كاملًا، بما فيه اسم Module.
2. وجود Record التي تعرف External ID.
3. وجود ملف التعريف داخل `data` في Manifest.
4. ترتيب التحميل قبل الملف الذي يستخدم Reference.
5. وجود Dependency عند الإشارة إلى Module أخرى.
6. تثبيت أوتحديث Module داخل Database الصحيحة.

لا تستبدل External ID برقم Database ثابت لتجاوز الخطأ.

راجع [[10 Data Files & External IDs]].

---

### Field غير موجودة

عندما تشير الرسالة إلى Field غير معروفة، اسأل أولًا:

> غير موجودة على أي Model؟

ثم افحص:

- الاسم التقني للـField.
- Model الخاصة بالـView أوالعملية.
- تعريف Python.
- سلسلة Imports.
- النسخة الفعلية من Module التي حمّلها Server.
- Restart بعد تعديل Python.
- Upgrade عند الحاجة إلى تحديث Schema أوتعريفات Module.

وجود Field على `equipment.asset` لا يجعلها Field مباشرةً على `equipment.loan`.

كما أن وجودها في Registry يختلف عن وجود Column المطلوبة في Database.

---

### `Expected singleton`

تعني الرسالة أن العملية احتاجت Record واحدة، لكنها استقبلت عددًا مختلفًا.

داخل Debugger أوShell، افحص:

```python
self.ids
len(self)
```

ثم حدد عقد Method:

| المطلوب | المعالجة المناسبة |
| --- | --- |
| تنفيذ العملية على كل Record | معالجة Recordset بما يناسبها |
| فتح Dialog لسجل واحد | التحقق من Singleton عند هذا المدخل |
| استخراج قيم من مجموعة | استخدام عملية تدعم المجموعة |

لا تضف `ensure_one()` أو`limit=1` قبل فهم المطلوب؛ فقد تخفي Records كان يجب التعامل معها.

راجع [[06 Records & Recordsets]].

---

### `AccessError`

حدد العملية المرفوضة أولًا:

```text
Read
Create
Write
Unlink
```

ثم افحص:

- المستخدم الفعلي.
- Groups والصلاحيات الممنوحة.
- القيود العامة وDomains.
- صلاحيات Fields عند ارتباط الخطأ بها.
- الشركات المسموحة والمفعلة.
- أي Business Method تفرض شروطًا إضافية.

قد تكون النتيجة صحيحة، مثل رفض تعديل جهاز بواسطة Equipment User.

إذا كانت العملية مطلوبة ومسموحة حسب Requirement، أصلح قاعدة الوصول أوالمسار الذي أرسل الطلب.

نجاح العملية بعد `sudo()` لا يثبت صحة الإصلاح.

---

### فشل XPath

إذا لم يجد Odoo عنصرًا تريد تعديله، راجع:

- `inherit_id`.
- View التي ورثتها فعلًا.
- بنية Parent View.
- موضع العنصر داخل البنية.
- XPath المستخدمة.
- اختلاف البنية بعد تثبيت Modules أخرى.

إذا نجحت XPath في Version سابقة، فهذا لا يثبت أنها تطابق View الحالية.

استخدم أدوات Developer Mode لفحص Views، ثم قارنها بملفات Module.

راجع [[12 Views & XML]] و[[18 Inheritance & Module Interaction]].

---

### فشل SQL أوUpgrade

إذا ظهر خطأ متعلق بـColumn أوConstraint، حدد هل السبب:

- كود جديد مع Schema قديمة.
- Upgrade لم تكتمل.
- بيانات موجودة تخالف Constraint جديدة.
- Migration لم تعالج الحالة القديمة.

لا تحذف Database أوConstraint لتجاوز التشخيص.

حدد البيانات المخالفة والتغيير المطلوب، ثم عالجها وفق خطة مناسبة كما سنناقش في [[31 Upgrades & Data Migration]].

---

## Practical Examples

### 1. Field من Model أخرى داخل View

نفترض أننا أضفنا إلى Form الإعارة:

```xml
<field name="serial_number"/>
```

ثم ظهرت رسالة تشير إلى أن Field غير موجودة على:

```text
equipment.loan
```

**التشخيص:**

1. افحص `model` الخاصة بالـView.
2. تحقق أن `serial_number` معرفة على `equipment.asset`.
3. لاحظ أن الإعارة ترتبط بالجهاز بواسطة `asset_id`.
4. حدد ما الذي تحتاج View إلى عرضه فعلًا.

إذا كان المطلوب اختيار الجهاز، فاستخدم Field الموجودة:

```xml
<field name="asset_id"/>
```

إذا كانت موجودة أصلًا، احذف الإضافة الخاطئة فقط.

أما إذا كان المطلوب عرض الرقم التسلسلي مباشرةً داخل الإعارة، فهذا يحتاج إلى تصميم Field مناسبة، مثل Related Field، وفق [[16 Computed Fields, Related Fields & Onchange]].

العلاقة لا تنقل جميع Fields الخاصة بالجهاز إلى Model الإعارة تلقائيًا.

**تطبيق الإصلاح:**

1. احفظ تعديل XML.
2. افتح **Apps** واحذف Filter **Apps** عند الحاجة.
3. ابحث عن `Equipment Training`.
4. نفذ **Upgrade**.
5. افتح Form الإعارة مجددًا.

إذا شمل الإصلاح تعديل Python، أعد تشغيل Server قبل Upgrade من الواجهة.

---

### 2. Dashboard تعرض صفرًا دون Exception

نفترض أن جهازًا متاحًا يظهر في قائمة الأجهزة، بينما Dashboard تعرض صفرًا.

من Network، وجدنا أن الطلب يستخدم:

```javascript
[["state", "=", "active"]]
```

لكن حالات الجهاز في مشروعنا هي:

```text
available
in_use
retired
```

إذًا المشكلة في Domain.

الطلب الصحيح:

```javascript
const count = await this.orm.searchCount(
    "equipment.asset",
    [["state", "=", "available"]]
);
```

قبل اعتماد النتيجة، تحقق أيضًا من:

- الحساب نفسه.
- الشركات نفسها.
- حالة الأرشفة.
- وصول المتصفح إلى نسخة JavaScript المعدلة.

هذه مشكلة في النتيجة دون ضرورة ظهور Traceback.

الدرس: اختبر المعنى الفعلي للطلب، وليس نجاح تنفيذه فقط.

---

### 3. اختيار الإجراء المناسب بعد التعديل

| التغيير | الإجراء المعتاد |
| --- | --- |
| تعديل منطق Python فقط | Restart للـProcess التي تنفذ الكود، ثم إعادة التجربة |
| إضافة Field مخزنة أوتغيير يتطلب Schema | Restart ثم Upgrade مناسبة |
| تعديل XML داخل `data` | Upgrade، مع مراعاة `noupdate` |
| تعديل قائمة Assets في Manifest | إعادة قراءة Manifest وتطبيق Upgrade ثم تحميل الصفحة |
| تعديل JavaScript محملة بالفعل | التأكد من تحديث Assets وإعادة تحميل المتصفح |
| تعديل بيانات من الواجهة | التحقق من حفظ Record والـContext |

هذه إجراءات معتادة، ويتحدد المطلوب النهائي حسب نوع التغيير.

Restart تعيد تحميل الكود في Process جديدة.

Upgrade تطبق تحديث Module داخل Database.

إعادة تحميل المتصفح تجلب الصفحة ومواردها مجددًا.

كل واحدة تعالج جزءًا مختلفًا.

---

## Server Debugging Tools

### Logging موجهة إلى Module

داخل ملف Python، يمكن استخدام:

```python
import logging


_logger = logging.getLogger(__name__)
```

ثم داخل Method التي تحقق فيها:

```python
_logger.debug(
    "Equipment operation: ids=%s user_id=%s company_ids=%s",
    self.ids,
    self.env.uid,
    self.env.companies.ids,
)
```

أضف السطر إلى التنفيذ الحالي دون استبدال Business Logic.

سجل المعلومات التي تساعد على فهم المسار، مثل IDs والفرع الذي وصل إليه التنفيذ.

تجنب تسجيل Passwords وAPI Keys وHeaders السرية أوPayloads كاملةً دون حاجة.

بعد التشخيص، احذف Logging المؤقتة أوخففها وفق قيمتها التشغيلية.

---

### تشغيل Logs الخاصة بالـModule

من مجلد `odoo20-dev`، شغّل Server التدريب بهذه الإضافة:

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_training \
  --log-handler odoo.addons.equipment_training:DEBUG
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\odoo.conf `
  -d odoo20_training `
  --log-handler odoo.addons.equipment_training:DEBUG
```

أوقف Process التدريب السابقة إذا كانت تستخدم Port نفسها.

هذا الاختيار يرفع مستوى Logging للـModule بدل رفعه لكل أجزاء النظام.

وللتفاصيل الإضافية أثناء تشخيص تحميل XML، يمكن إضافة:

```text
--log-handler odoo.tools.convert:DEBUG
```

استخدم مستوى التفاصيل المطلوب للمشكلة، ثم أعد الإعداد المعتاد بعد الانتهاء.

---

### Odoo Shell

تساعد Shell على فحص Registry والبيانات باستخدام ORM.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin shell \
  -c config/odoo.conf \
  -d odoo20_training
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin shell `
  -c config\odoo.conf `
  -d odoo20_training
```

داخل Shell، افحص البيئة:

```python
env.cr.dbname
env.uid
env.su
env.company.id
env.companies.ids
```

وتحقق من وجود Field في Registry:

```python
"serial_number" in env["equipment.asset"]._fields
```

ولمعرفة مسار Module التي حمّلتها هذه Process:

```python
import importlib


training_module = importlib.import_module(
    "odoo.addons.equipment_training"
)

training_module.__file__
```

هذا مفيد إذا كانت هناك نسختان من Module داخل مسارات Addons مختلفة.

> [!note] Shell Process مستقلة
> نتائج Shell تصف البيئة التي فتحتها الآن.
>
> لا تثبت وحدها أن Web Server القديمة أعادت تحميل Python أوتستخدم الإعدادات نفسها.

---

### تشخيص الصلاحيات من Shell

نجاح القراءة في Shell قد ينتج عن اختلاف المستخدم أووضع Superuser.

لإعادة الفحص باسم حساب التدريب:

```python
from odoo.exceptions import UserError


login = input("Training user login: ").strip()

debug_user = env["res.users"].search(
    [("login", "=", login)],
    limit=1,
)

if not debug_user:
    raise UserError("Training user not found.")

Asset = (
    env["equipment.asset"]
    .with_user(debug_user)
    .with_context(
        allowed_company_ids=[
            debug_user.company_id.id,
        ],
    )
)

Asset.search_count([
    ("state", "=", "available"),
])
```

اختر الشركات التي تطابق حالة المشكلة؛ المثال يستخدم شركة المستخدم الرئيسية.

نفحص وجود المستخدم قبل تمريره إلى `with_user()` حتى لا نعتمد على نتيجة بحث فارغة.

استخدم Shell للفحص بحذر: استدعاء Method قد يعدل البيانات أويرسل طلبًا خارجيًا. تراجع Transaction لا يلغي أثرًا حدث في نظام خارجي.

---

### Breakpoints

عندما لا تكفي Logs، شغّل Odoo من Python Debugger داخل بيئة التطوير.

اضبط:

| الإعداد | القيمة في مشروعنا |
| --- | --- |
| Python Interpreter | البيئة `.venv` |
| Script | `odoo/odoo-bin` |
| Working Directory | `odoo20-dev` |
| Arguments | Config وDatabase التدريب |

ضع Breakpoint في Method تعلم أن العملية تمر بها، مثل `action_start_use()`.

ثم افحص:

```text
self.ids
self.env.uid
self.env.su
self.env.companies.ids
القيم الداخلة
الشرط الذي اختاره التنفيذ
العملية التالية
```

إذا لم يصل التنفيذ إلى Breakpoint، تحقق من Process المستخدمة ومسار الاستدعاء والنسخة المحملة من Module.

إيقاف التنفيذ قد يترك Request منتظرة وTransaction مفتوحة؛ استخدم Debugger في بيئة التطوير، ثم أكمل التنفيذ أوأنه الجلسة عند الانتهاء.

---

## Verification

### إعادة الحالة الأصلية

بعد الإصلاح، أعد التجربة باستخدام:

- الحساب نفسه.
- الشركات نفسها.
- البيانات التي أظهرت المشكلة.
- خطوات التنفيذ نفسها.

ثم تحقق من أن النتيجة المطلوبة تحققت.

لا تكتفِ باختفاء الرسالة إذا أصبحت العملية تتجاوز Rule مطلوبة أوتعرض بيانات إضافية.

---

### فحص الحالات القريبة

اختبر حالة مسموحة وحالة مرفوضة ذات علاقة.

مثلًا، بعد إصلاح Access Rule:

```text
المستخدم يصل إلى الأجهزة المسموحة
                +
يبقى ممنوعًا من الأجهزة خارج نطاقه
```

وبعد إصلاح Domain:

```text
وجود Records مطابقة
        +
عدم وجود Records مطابقة
```

---

### Regression Test

إذا كشف الخطأ عن قاعدة يمكن اختبارها آليًا، أضف اختبارًا يعيد إنتاجها ويثبت النتيجة الصحيحة، كما تعلمنا في [[26 Testing]].

اختر الطبقة المناسبة:

| نوع المشكلة | التحقق المناسب |
| --- | --- |
| Business Rule | Python Test |
| Access أوCompany Scope | اختبار بصلاحيات وسياق محددين |
| XML غير صالح عند التحميل | Installation أوUpgrade للتحقق |
| OWL Rendering أوEvent | JavaScript Test أوBrowser Test |
| سلوك تكامل | اختبار الطلب والاستجابة والفشل المناسب |

أما مشاكل الأداء، فتحتاج إلى قياس يثبت أثر الإصلاح، وهو موضوع [[28 Performance]].

---

## Important Notes

- اجعل المشكلة قابلة لإعادة الإنتاج قبل تعديل الكود.
- اربط Logs بالطلب والوقت وDatabase التي تحقق فيها.
- اقرأ Exception وسياقها، ولا تكتفِ بعنوان `RPC_ERROR`.
- `ParseError` قد تغلف خطأ في تعريفات Odoo، وليس في Syntax XML فقط.
- نجاح Request لا يثبت صحة النتيجة.
- تحقق من Model قبل افتراض أن Field لم تُحمّل.
- حدد هل Method تدعم Recordset قبل إضافة `ensure_one()`.
- قد تكون `AccessError` نتيجة متوقعة لحماية صحيحة.
- Developer Mode لا تمنح صلاحيات إضافية ولا تحل محل Python Debugger.
- Restart وUpgrade وإعادة تحميل المتصفح إجراءات مختلفة.
- تحقق من نسخة Module وConfig وDatabase التي تستخدمها Process الفعلية.
- نجاح العملية في Shell لا يثبت نجاحها باسم مستخدم الواجهة.
- لا تستخدم `sudo()` أوحذف القيود كإصلاح افتراضي.
- لا تسجل أسرار الاتصال أوالبيانات الحساسة دون حاجة.
- أعد اختبار الحالة الأصلية والحالات القريبة بعد الإصلاح.
- احتفظ باختبار آلي للمشكلة عندما يكون ذلك مناسبًا.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[28 Performance]]

---

## Sources

- [Odoo 20 Documentation — Command-line Interface](https://www.odoo.com/documentation/20.0/developer/reference/cli.html)
- [Odoo 20 Documentation — Developer Mode](https://www.odoo.com/documentation/20.0/applications/general/developer_mode.html)
- [Odoo 20 Documentation — Assets](https://www.odoo.com/documentation/20.0/developer/reference/frontend/assets.html)
- [Odoo 20 Documentation — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Documentation — Security in Odoo](https://www.odoo.com/documentation/20.0/developer/reference/backend/security.html)
- [Odoo 20 Official Source — XML Loading and ParseError](https://github.com/odoo/odoo/blob/20.0/odoo/tools/convert.py)
- [Odoo 20 Official Source — Web Client RPC](https://github.com/odoo/odoo/blob/20.0/addons/web/static/src/core/network/rpc.js)
- [Odoo 20 Official Source — Odoo Shell](https://github.com/odoo/odoo/blob/20.0/odoo/cli/shell.py)