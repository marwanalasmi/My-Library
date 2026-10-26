---
type: learning
odoo_version: "20.0"
status: complete
created: 2026-10-13
reviewed: 2026-10-13
---
# 01 Odoo Architecture

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Core Architecture|Core Architecture]]
- [[#Client & Server Flow|Client & Server Flow]]
- [[#Modules & Architectural Thinking|Modules & Architectural Thinking]]
- [[#Example|Example]]
- [[#كيف تستخدم هذا الفهم؟|كيف تستخدم هذا الفهم؟]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

قبل أن نبدأ في إعداد Environment أوكتابة Modules، نحتاج إلى فهم الصورة العامة لـOdoo.

عندما يفتح المستخدم صفحة أوينفذ عملية، تتعاون عدة أجزاء حتى تظهر النتيجة:

- واجهة تعرض المعلومات وتستقبل التفاعل.
- Server تنفذ العمليات وتطبق القواعد.
- نظام تخزين يحتفظ بالبيانات.

فهم العلاقة بينها يساعدنا على الإجابة عن سؤال أساسي أثناء Development:

> ما مسؤولية هذا الجزء، وأين يجب تنفيذها؟

في هذا الشابتر سنبني صورة واضحة لـOdoo Architecture، ثم نتتبع عملية بسيطة من تفاعل المستخدم حتى ظهور النتيجة.

لن نحتاج إلى كتابة Code الآن؛ سنستخدم هذا الفهم كأساس للشابترات التالية.

---

## Core Architecture

Odoo تعتمد **Three-tier Architecture**، أي تقسيم النظام إلى ثلاث طبقات رئيسية:

| الطبقة | المسؤولية الأساسية | التقنية الرئيسية |
| --- | --- | --- |
| Presentation | عرض المعلومات واستقبال تفاعل المستخدم | HTML وCSS وJavaScript، ومنها OWL |
| Business Logic | تنفيذ العمليات وتطبيق قواعد الأعمال والصلاحيات | Odoo Server باستخدام Python |
| Data Storage | حفظ بيانات الأعمال | PostgreSQL |

يمكن تصور العلاقة الأساسية:

```text
Presentation
     ↕
Business Logic
     ↕
Data Storage
```

هذا التقسيم يوضح المسؤوليات، وسنستخدمه بشكل مبسط دون الدخول في جميع التفاصيل الداخلية.

> [!info] ثلاث طبقات لا تعني ثلاثة أجهزة
> يمكن تشغيل Odoo Server وPostgreSQL على نفس جهاز التطوير.
>
> ويمكن في بيئات أخرى تشغيلهما على أجهزة منفصلة.
>
> توزيع المسؤوليات شيء، وتوزيع المكونات على الأجهزة شيء آخر.

---

### Presentation Layer

Presentation Layer هي الجزء الذي يعرض المعلومات للمستخدم ويستقبل تفاعله.

من أمثلتها:

- Forms وLists.
- Kanban وDashboards.
- Buttons وMenus.
- صفحات Website وPortal.

مثلًا، عندما يفتح المستخدم Customer Form:

```text
Customer
──────────────────

Name
Email
Phone

[ Save ]
```

فطريقة عرض Fields واستقبال المدخلات وتفاعل المستخدم مع زر Save كلها مرتبطة بالـPresentation.

داخل Browser، تستخدم الواجهة تقنيات مثل HTML وCSS وJavaScript.

وتستخدم Odoo أيضًا **OWL** لبناء مكونات تفاعلية في Web Client.

لا نحتاج إلى تفاصيل هذه التقنيات الآن؛ سنشرح Views في
[[12 Views & XML]]، وOWL وتطوير الواجهة في [[24 OWL & Frontend]].

#### تعريف الواجهة وتنفيذها

قد نكتب تعريف View داخل ملفات Module على Server، ثم تستخدمه الواجهة لعرض البيانات.

لذلك، وجود ملف داخل Source Code الخاصة بالـServer لا يعني أن مسؤوليته Business Logic.

نحدد المسؤولية حسب **ما يفعله الجزء**، وليس حسب مكان الملف وحده.

---

### Business Logic Layer

Business Logic Layer تحدد ما الذي يجب أن يحدث عند تنفيذ عملية.

تعمل داخل Odoo Server، التي تعتمد بشكل أساسي على Python.

مثلًا، عند محاولة تأكيد Order، قد تحتاج العملية إلى الإجابة عن أسئلة مثل:

- هل البيانات المطلوبة مكتملة؟
- هل State الحالية تسمح بالتأكيد؟
- هل المستخدم مسموح له بتنفيذ العملية؟
- ما البيانات التي يجب تغييرها؟
- هل توجد عمليات أخرى مرتبطة بالتأكيد؟

هذه القرارات تنتمي إلى Server-side Behavior.

سنقابل لاحقًا مفاهيم مثل Models وMethods وBusiness Logic، لكن يكفي الآن فهم دورها العام:

> Server تستقبل العملية، وتنفذ السلوك المطلوب، وتطبق القواعد المناسبة.

التفاصيل في [[04 Models & Fields]] و[[15 Business Logic]].

#### قواعد الأعمال والصلاحيات

قد تساعد الواجهة المستخدم بإخفاء Button أوعرض تنبيه، لكن القواعد المهمة يجب فرضها على Server.

مثلًا:

```text
إخفاء Confirm Button
```

يعالج ما يراه المستخدم في الواجهة.

أما:

```text
منع تأكيد Order غير مكتملة
```

فيحتاج تحققًا على Server حتى يسري على طرق تنفيذ العملية المختلفة.

> [!important] الواجهة ليست وحدها جهة الحماية
> إخفاء Button لا يثبت أن العملية ممنوعة Server-side.
>
> يجب تطبيق الصلاحيات وقواعد الأعمال المهمة داخل Server، حتى عندما تأتي العملية من خارج الواجهة المعتادة.

سنشرح الصلاحيات في [[11 Security & Multi-Company]].

---

### Data Storage Layer

Odoo تستخدم **PostgreSQL** كـDatabase Management System لحفظ بيانات الأعمال.

من أمثلة البيانات:

- Customers.
- Products.
- Orders.
- Employees.
- Tasks.
- Invoices.

مثلًا، Order تحتاج إلى حفظ معلومات مثل حالتها والجهة المرتبطة بها وبقية بياناتها.

Database توفر التخزين الدائم الذي يسمح بالرجوع إلى هذه المعلومات بعد انتهاء العملية أوإعادة تشغيل Odoo.

#### ماذا عن الملفات والمرفقات؟

ليست كل محتويات النظام محفوظة كنصوص وقيم داخل PostgreSQL فقط.

بعض الملفات والمرفقات قد تحفظ في **Filestore**، وهو مكان لحفظ الملفات تديره Odoo، مع بقاء معلوماتها وارتباطاتها مسجلة في Database.

يكفي الآن معرفة أن التخزين قد يشمل:

```text
PostgreSQL
→ بيانات الأعمال ومعلومات ارتباط الملفات

Filestore
→ محتوى بعض الملفات والمرفقات
```

سنرى مكان Runtime Data أثناء إعداد Environment في
[[02 Development Environment]]، وعلاقة Database والFilestore بالنسخ الاحتياطي في [[30 Backup & Restore]].

---

### أين تقع ORM؟

عندما تحتاج Server إلى قراءة بيانات الأعمال أوتعديلها، تستخدم عادةً **ORM**.

ORM اختصار لـ**Object-Relational Mapping**، وهي آلية تتيح التعامل مع بيانات Odoo من خلال Models بدل كتابة SQL مباشرة لكل عملية.

الصورة المبسطة:

```text
Business Logic
      ↓
     ORM
      ↓
 PostgreSQL
```

ORM تعمل ضمن Odoo Server.

قد تقابل تعبير **ORM Layer** في التوثيق. هذا وصف لطبقة برمجية مسؤولة عن الوصول إلى البيانات، وليس Tier رابعة مستقلة بجانب الطبقات الثلاث السابقة.

في هذه المرحلة يكفي أن تعرف:

> Business Logic تستخدم Models والـORM للتعامل مع البيانات، والـORM تتولى تحويل عملياتها إلى التعامل المناسب مع PostgreSQL.

سنشرح Models في [[04 Models & Fields]]، والـORM في [[05 ORM]].

---

## Client & Server Flow

بعد فهم المسؤوليات، نحتاج إلى معرفة المكونات التي تشارك في تنفيذ عملية من الواجهة.

الصورة الأساسية:

```text
Browser
└── Odoo Web Client
          ↕
      Odoo Server
          ↕
      PostgreSQL
```

الـWeb Client يعمل داخل Browser، بينما Odoo Server تنفذ Server-side Behavior وتتصل بـPostgreSQL.

---

### Browser وOdoo Web Client

Browser هي البرنامج الذي نستخدمه لفتح Odoo.

أما **Odoo Web Client** فهي Application تعمل داخل Browser وتوفر واجهة العمل المعتادة، مثل Forms وLists وMenus.

مسؤولياتها تشمل:

- عرض البيانات.
- استقبال المدخلات.
- إدارة التفاعل والتنقل داخل الواجهة.
- إرسال Requests إلى Server عند الحاجة.
- تحديث الشاشة بناءً على النتائج.

ليست كل حركة في الواجهة تحتاج تحميل صفحة كاملة من جديد؛ يمكن للـWeb Client طلب المعلومات المطلوبة وتحديث الجزء المناسب من الشاشة.

ولا يعني وجود Website أوPortal أن جميع واجهات Odoo تعمل بنفس تفاصيل Web Client الداخلية؛ سنتناول كل جزء في شابتره.

---

### Request وResponse

عندما تحتاج الواجهة إلى عملية من Server، ترسل **Request**.

مثلًا، قد تطلب:

- قراءة بيانات Order.
- حفظ تعديل.
- تنفيذ عملية Confirm.

تتعامل Odoo Server مع الطلب، وقد تحتاج أثناءه إلى قراءة بيانات أوتعديلها.

بعد ذلك تعيد **Response** إلى Client، وقد تحتوي بيانات أونتيجة عملية أورسالة توضح سبب تعذر تنفيذها.

المبدأ الأساسي:

```text
Client
   ↓ Request
Server
   ↓ Response
Client
```

---

### مسار عملية تحتاج إلى Database

يمكن تقسيم المسار إلى اتجاهين:

#### اتجاه الطلب

```text
Odoo Web Client
       ↓
   Odoo Server
       ↓
      ORM
       ↓
  PostgreSQL
```

#### اتجاه رجوع النتيجة

```text
  PostgreSQL
       ↓
      ORM
       ↓
   Odoo Server
       ↓
Odoo Web Client
```

Server هي التي تتواصل مع Database وتعالج نتيجة العملية، ثم تتواصل مع Client.

المتصفح لا يرسل عمليات Database مباشرة إلى PostgreSQL في هذا المسار.

هذه صورة عامة لعملية تحتاج إلى البيانات. التفاصيل الداخلية تختلف حسب العملية، وقد تنفذ الواجهة أكثر من Request لإكمال تفاعل واحد.

---

### ليست كل عملية تبدأ من Browser

المثال المعتاد يبدأ بتفاعل المستخدم، لكن Odoo تنفذ عمليات أخرى أيضًا.

مثلًا:

| بداية العملية | مثال |
| --- | --- |
| تفاعل المستخدم | الضغط على Confirm |
| تشغيل مجدول داخل Server | تنفيذ مهمة دورية |
| تكامل خارجي | إرسال طلب من نظام آخر |

لذلك، قواعد الأعمال المهمة يجب أن تعمل على Server دون الاعتماد على أن العملية بدأت من Button معينة.

سنشرح Scheduled Actions في [[22 Automation & Scheduled Actions]]، والتكاملات في [[25 APIs & Integrations]].

---

## Modules & Architectural Thinking

### أين تقع Module في الصورة؟

الـ**Module** هي طريقة Odoo لتنظيم Functionality داخل مجموعة ملفات مترابطة.

يمكنها إضافة وظيفة جديدة أوتوسيع وظيفة موجودة.

Module ليست واحدة من الطبقات الثلاث؛ يمكن أن توفر أجزاء تخدم أكثر من طبقة.

مثلًا، Equipment Module قد توفر:

| الجزء | ما توفره Module |
| --- | --- |
| Presentation | تعريف Forms وLists لعرض Equipment |
| Business Logic | قواعد استخدام Equipment وتغيير حالتها |
| Data Definitions | تعريف البيانات المطلوبة، وربما بيانات أولية |

الصورة:

```text
Equipment Module
│
├── Presentation Definitions
├── Business Logic
└── Data Definitions
```

Architecture تساعدنا على فصل المسؤوليات، بينما Module تجمع أجزاء Feature داخل وحدة برمجية يمكن إضافتها إلى Odoo.

سنشرح تنظيم ملفاتها وتثبيتها في [[03 Modules & Addons]].

---

### تعريف البيانات ليس هو البيانات التي ينشئها المستخدم

قد تحتوي Module على تعريف Model أوبيانات أولية تحتاجها للعمل.

لكن عندما يستخدم شخص النظام وينشئ Equipment جديدة، تحفظ بياناتها في Database.

الفرق:

```text
Module Files
→ تعريف الوظيفة وسلوكها والبيانات الأولية عند الحاجة

Database
→ Records التي يديرها النظام أثناء الاستخدام
```

الـ**Record** هنا تعني سجل بيانات واحدًا، مثل جهاز أومهمة أوOrder.

سنشرح Records بالتفصيل في [[06 Records & Recordsets]].

هذا التمييز يمنع تصور أن كل البيانات التي ينشئها المستخدم تضاف إلى ملفات Module.

---

### مصطلحات Odoo والمعمارية العامة

قد تكون تعرف تقسيم **MVC**:

```text
Model
View
Controller
```

وهو تقسيم معماري آخر يستخدم لتنظيم مسؤوليات التطبيقات.

يمكن أن يساعدك في فهم بعض المصطلحات، لكن لا نفترض أن أسماء Odoo تطابق استخدام كل Framework أخرى حرفيًا.

مثلًا:

- **Odoo Model** تمثل Business Object وتوفر تعريفات وسلوكًا؛ ليست مجرد اسم لـDatabase Table.
- **Odoo View** تستخدم نظام تعريف وعرض خاصًا بـOdoo؛ ليست مجرد HTML File.
- **HTTP Controller** تعالج أنواعًا من Web Requests، وسنشرحها لاحقًا.

نستخدم في هذا الشابتر تقسيم الطبقات الثلاث لفهم الصورة العامة. ثم نتعلم معنى كل Concept داخل Odoo في وقتها المناسب.

---

## Example

سنفترض Requirement بسيطة:

> يستطيع المستخدم تأكيد Order بعد التحقق من اكتمال بياناتها وصلاحية حالتها الحالية.

في هذا المثال التوضيحي، سنسمي الحالة الأولى Draft والحالة بعد التأكيد Confirmed.

هذه أسماء لتتبع الفكرة؛ الحالات الفعلية تحددها Model والوظيفة التي نبنيها.

---

### 1. عرض Order وبدء العملية

تظهر للمستخدم Order وحالتها الحالية، مع Button لبدء التأكيد:

```text
Order
──────────────────

State: Draft

[ Confirm ]
```

هذه مسؤولية Presentation:

- عرض البيانات.
- إظهار التفاعل المناسب.
- استقبال ضغط المستخدم.

---

### 2. إرسال Request

عندما يضغط المستخدم Confirm، ترسل الواجهة Request إلى Odoo Server لتنفيذ العملية.

```text
User
  ↓
Odoo Web Client
  ↓ Request
Odoo Server
```

الواجهة تبدأ العملية، بينما Server تنفذ السلوك المرتبط بها.

---

### 3. تطبيق القواعد على Server

تتعامل Server مع العملية، وتتحقق من الشروط المطلوبة.

مثلًا:

- هل المستخدم مسموح له بتنفيذها؟
- هل Order في حالة تسمح بالتأكيد؟
- هل البيانات المطلوبة مكتملة؟

هذه مسؤولية Business Logic والتحقق على Server.

إذا لم تسمح القواعد بالتأكيد، تعيد Server نتيجة توضح سبب تعذر العملية، وتعرضها الواجهة للمستخدم.

---

### 4. تحديث البيانات

إذا سمحت القواعد، تنفذ Business Logic التغيير المطلوب من خلال ORM.

في مثالنا:

```text
State: Draft
      ↓
State: Confirmed
```

ومسار التعامل مع البيانات:

```text
Odoo Server
     ↓
    ORM
     ↓
PostgreSQL
```

لا نحتاج الآن إلى معرفة اسم Method أوطريقة كتابة التعديل. المهم معرفة أين تقع مسؤولية تنفيذ العملية وحفظ نتيجتها.

---

### 5. عرض النتيجة

ترجع نتيجة التعامل مع البيانات إلى Server، ثم تعيد Server نتيجة العملية إلى Web Client.

تحدث الواجهة المعلومات المعروضة، وقد تطلب قراءة البيانات المحدثة عند الحاجة.

في النهاية يرى المستخدم:

```text
Order
──────────────────

State: Confirmed
```

ظهور النتيجة مسؤولية Presentation، بينما تنفيذ قواعد التأكيد وتحديث البيانات تم على Server.

---

### الصورة الكاملة

```text
User
  ↓
Odoo Web Client
  ↓ Request
Odoo Server
  ↓
التحقق من الصلاحيات وقواعد الأعمال
  ↓
ORM
  ↓
PostgreSQL
  ↓
ORM
  ↓
Odoo Server
  ↓ Response
Odoo Web Client
  ↓
User
```

استطعنا تحديد مسؤولية كل جزء دون كتابة Python أوXML.

في الشابترات التالية سنتعلم الأدوات التي تنفذ هذه المسؤوليات.

---

## كيف تستخدم هذا الفهم؟

عندما تستلم Requirement، ابدأ بتقسيمها إلى مسؤوليات قبل اختيار الملفات والـMethods.

استخدم الأسئلة التالية:

| السؤال | المسؤولية الأقرب |
| --- | --- |
| ما المعلومات التي يجب أن تظهر؟ | Presentation |
| كيف يبدأ المستخدم العملية؟ | Presentation |
| ما الشروط التي تسمح بالعملية؟ | Business Logic والتحقق على Server |
| من يحق له تنفيذها؟ | Security على Server |
| ما البيانات التي نحتاج قراءتها أوتعديلها؟ | Models وORM |
| أين تحفظ البيانات؟ | PostgreSQL، والملفات عند الحاجة |

بعد ذلك، اربط كل مسؤولية بمفهوم Odoo المناسب:

| المسؤولية | المفاهيم التي سنتعلمها |
| --- | --- |
| تنظيم Feature داخل وحدة برمجية | Modules |
| تعريف بيانات الأعمال وسلوكها | Models وFields |
| قراءة البيانات وتعديلها | ORM |
| فرض الصلاحيات | Security |
| عرض بيانات الأعمال | Views |
| توفير التنقل وتشغيل العمليات من الواجهة | Actions وMenus وButtons |
| بناء تفاعل Client مخصص | OWL وFrontend |
| معالجة Web Requests مخصصة | HTTP Controllers |

لا تحتاج إلى إتقان هذه المفاهيم الآن. يكفي أن تعرف أن لكل واحدة دورًا ضمن الصورة العامة.

المسار الذي سنبني عليه:

```text
Business Requirement
        ↓
تحديد المسؤوليات
        ↓
اختيار Odoo Concepts المناسبة
        ↓
تنفيذ الحل
        ↓
التحقق من السلوك والنتيجة
```

---

## Important Notes

- **إخفاء عنصر في الواجهة لا يضمن منع العملية.** يجب فرض الصلاحيات وقواعد الأعمال المهمة على Server.

- **Three-tier Architecture لا تشترط ثلاثة أجهزة.** يمكن تشغيل أكثر من مكون على نفس جهاز التطوير.

- **ORM تعمل داخل Odoo Server.** وصفها كـORM Layer لا يعني أنها Tier رابعة مستقلة.

- **ملفات Module ليست مكان حفظ كل بيانات المستخدم.** Module توفر التعريفات والسلوك، بينما Records تحفظ في Database.

- **لا تختزل التخزين كله في PostgreSQL.** بعض الملفات والمرفقات قد تحفظ في Filestore، ولهذا علاقة مهمة بالنسخ الاحتياطي.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[02 Development Environment]]

---

## Sources

- [Odoo 20 — Architecture Overview](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/01_architecture.html)
- [Odoo 20 — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 — JavaScript Framework Overview](https://www.odoo.com/documentation/20.0/developer/reference/frontend/framework_overview.html)
- [Odoo 20 Source — Attachment Storage and Filestore](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_attachment.py)