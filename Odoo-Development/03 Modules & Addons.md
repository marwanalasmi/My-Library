---
type: learning
odoo_version: "20.0"
status: complete
created: 2026-10-13
reviewed: 2026-10-13
---
# 03 Modules & Addons

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Module Fundamentals|Module Fundamentals]]
- [[#Module Structure & Metadata|Module Structure & Metadata]]
- [[#Discovery & Loading|Discovery & Loading]]
- [[#Module Lifecycle|Module Lifecycle]]
- [[#Example|Example]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

عندما نريد إضافة Functionality جديدة إلى Odoo أو توسيع Functionality موجودة، نبني عادةً **Custom Module**.

قد تكون Module صغيرة تضيف تعديلًا واحدًا، أو أكبر وتجمع عدة Features مترابطة. في الحالتين، نحتاج إلى معرفة كيف ننظم ملفاتها، وكيف تكتشفها Odoo، وكيف نثبتها ونطبق التغييرات عليها.

هذا الشابتر يجمع بين فهم الفكرة والتطبيق العملي:

- ما الفرق بين Module وAddon وApp؟
- ما وظيفة `__manifest__.py` و`__init__.py`؟
- كيف تصل Odoo إلى ملفات Module؟
- ما الفرق بين Installation وRestart وUpgrade؟

في النهاية سننشئ Module بسيطة باسم `equipment_training` ونثبتها داخل Database التدريب، لتكون أساسًا للتطبيق في الشابترات التالية.

---

## Module Fundamentals

### ما هي Module؟

الـ**Module** هي وحدة برمجية نستخدمها لإضافة وظيفة إلى Odoo أو تعديل وظيفة موجودة.

من ناحية الملفات، هي Directory تحتوي ملفات تعمل معًا لتنفيذ هذه الوظيفة.

مثلًا، يمكن بناء Module من أجل:

- إدارة Equipment.
- إضافة Approval إلى Sales.
- إنشاء Report.
- ربط Odoo بخدمة خارجية.

قد تحتوي Module على Models وViews وSecurity وغيرها، لكن ليس مطلوبًا أن تحتوي على جميع هذه الأجزاء.

**محتوى Module يتحدد حسب الوظيفة التي تنفذها.**

---

### Module وAddon وApp

هذه المصطلحات مرتبطة ببعض، لكن استخدامها يختلف قليلًا:

| المصطلح | المقصود |
| --- | --- |
| Module | الوحدة البرمجية التي تضيف Functionality أو توسع Functionality موجودة |
| Addon | اسم شائع للـModule باعتبارها إضافة إلى Odoo |
| App | Module تمثل عادةً Application رئيسية للمستخدم |

مثلًا، Sales توفر Application للمستخدم، بينما Module تضيف Field إلى Sales قد تكون مجرد Extension صغيرة.

كلتاهما Module، لكن الثانية لا تحتاج أن تكون App مستقلة.

> [!info] العلاقة بين المصطلحات
> يستخدم مصطلحا **Module** و**Addon** غالبًا للإشارة إلى نفس الوحدة البرمجية.
>
> أما **App** فهي Module مصنفة كتطبيق رئيسي؛ ليست كل Module عبارة عن App.

---

### Technical Name وDisplay Name

لكل Module اسم تقني يسمى **Technical Name**، ويأتي من اسم Directory الخاصة بها.

مثلًا:

```text
equipment_training/
```

تعطي Technical Name:

```text
equipment_training
```

أما الاسم المقروء الذي يظهر للمستخدم، فنحدده داخل Manifest:

```text
Equipment Training
```

| النوع | المثال | مصدره |
| --- | --- | --- |
| Technical Name | `equipment_training` | اسم Directory |
| Display Name | `Equipment Training` | قيمة `name` داخل Manifest |

نستخدم Technical Name في أوامر Installation وUpgrade، وفي تعريف Dependencies بين Modules.

يفضل اختيار اسم واضح باستخدام حروف صغيرة وUnderscores، مثل:

- `equipment_training`
- `sale_approval`
- `library_management`

وتجنب المسافات والشرطات في Technical Name.

---

## Module Structure & Metadata

### Module Structure

أبسط Structure سنبدأ بها:

```text
equipment_training/
├── __init__.py
└── __manifest__.py
```

لكل ملف مسؤولية مختلفة:

| الملف | المسؤولية |
| --- | --- |
| `__manifest__.py` | وصف Module وتحديد Dependencies والملفات التي تحتاج Odoo إلى معالجتها |
| `__init__.py` | نقطة البداية لاستيراد Python Code الخاصة بالـModule |

مع نمو Module، قد نضيف Directories أخرى:

| Directory | الاستخدام المعتاد |
| --- | --- |
| `models/` | تعريف Models وPython Logic المرتبطة بها |
| `views/` | تعريف واجهات عرض البيانات |
| `security/` | تعريف صلاحيات الوصول وقواعد الحماية |
| `data/` | ملفات بيانات وإعدادات تحتاجها Module |
| `demo/` | بيانات تجريبية |
| `wizard/` | مكونات العمليات المؤقتة والخطوات المساعدة |
| `reports/` | تعريف Reports |
| `controllers/` | معالجة HTTP Requests |
| `static/` | ملفات Frontend مثل JavaScript وStyles وTemplates |
| `tests/` | الاختبارات |
| `i18n/` | ملفات الترجمة |

هذه أسماء تنظيمية شائعة، وليست قائمة يجب إنشاؤها كاملة لكل Module.

أنشئ فقط ما تحتاجه الوظيفة الحالية. سنشرح كل جزء عندما نصل إلى الشابتر الخاص به.

---

### `__manifest__.py`

الـ**Manifest** هي ملف يصف Module ويعطي Odoo المعلومات اللازمة للتعامل معها.

تكتب داخل ملف `__manifest__.py` الموجود في جذر Module، وتحتوي **Python Dictionary**.

مثلًا، داخل `equipment_training/__manifest__.py`:

```python
{
    "name": "Equipment Training",
    "version": "20.0.1.0.0",
    "summary": "Training module for learning Odoo development",
    "license": "LGPL-3",
    "depends": [
        "base",
    ],
    "data": [],
    "application": False,
    "installable": True,
}
```

هذا الملف ليس JSON؛ لذلك نستخدم Python Syntax، ومنها `True` و`False`.

#### `name`

تحدد الاسم المقروء للـModule:

```python
"name": "Equipment Training",
```

هذه القيمة لا تغير Technical Name؛ يبقى الاسم التقني مأخوذًا من اسم Directory.

#### `summary`

تقدم وصفًا مختصرًا لهدف Module:

```python
"summary": "Training module for learning Odoo development",
```

اكتب فيها ما تفعله Module بجملة واضحة، دون تحويلها إلى شرح تفصيلي لكل Feature.

#### `version`

تحدد Version الخاصة بالـModule:

```python
"version": "20.0.1.0.0",
```

في هذا المرجع سنستخدم هذا النمط:

| الجزء | المقصود في النمط المستخدم هنا |
| --- | --- |
| `20.0` | Odoo Version المستهدفة |
| `1.0.0` | Version الخاصة بالـModule |

هذا أسلوب تسمية سنلتزم به في الأمثلة، وليس شرطًا بأن تستخدم جميع Modules نفس عدد الأجزاء.

تغيير رقم Version وحده لا يطبق التعديلات على Database، ولا يجعل Module متوافقة تلقائيًا مع إصدار آخر من Odoo.

التوافق مع Major Version أخرى يحتاج مراجعة Code وAPIs والتغييرات المطلوبة، كما سنشرح في [[31 Upgrades & Data Migration]].

#### `depends`

تحدد Modules التي تعتمد عليها Module الحالية.

في مثالنا:

```python
"depends": [
    "base",
],
```

تخبر Odoo بأن Module تعتمد على `base`.

وإذا كانت Functionality تحتاج Sales وMail، نصرح بهما أيضًا:

```python
"depends": [
    "base",
    "sale",
    "mail",
],
```

نكتب هنا **Technical Names** للـModules.

تستخدم Odoo هذه المعلومات لتحديد Dependencies وترتيب تحميلها قبل Module التي تحتاجها.

لا تعتمد على أن Module أخرى مثبتة بالصدفة في Database التدريب. إذا كانت Code تحتاجها، يجب التصريح بها داخل `depends`.

سنشرح توسيع Modules والتفاعل بينها في [[18 Inheritance & Module Interaction]].

#### `data`

تحدد الملفات التي تعالجها Odoo أثناء Installation وUpgrade، مثل ملفات تعريف Views أو البيانات الأساسية.

داخل Manifest، يمكن أن تكون بالشكل التالي:

```python
"data": [
    "data/categories.xml",
    "views/equipment_views.xml",
    "views/menus.xml",
],
```

هذا مثال توضيحي لأسماء الملفات؛ لا نضيفه إلى Module التدريب الآن لأن هذه الملفات لم تُنشأ بعد.

انتبه إلى نقطتين:

1. Paths تكون بالنسبة إلى **جذر Module**.
2. ترتيب الملفات داخل القائمة مهم.

مثلًا، المسار `data/categories.xml` يشير إلى:

```text
equipment_training/
└── data/
    └── categories.xml
```

وإذا كان ملف يعتمد على تعريف موجود في ملف آخر، نضع الملف الذي يوفر التعريف أولًا.

وجود ملف XML داخل Directory وحده لا يكفي لتحميله. يجب إدراجه في القائمة المناسبة داخل Manifest.

التفاصيل في [[10 Data Files & External IDs]].

#### `demo`

تحدد ملفات البيانات التجريبية:

```python
"demo": [
    "demo/equipment_demo.xml",
],
```

قد تحتوي مثلًا على Equipment جاهزة للتجربة أو Records توضح طريقة استخدام Module.

| القائمة | الاستخدام |
| --- | --- |
| `data` | بيانات وتعريفات تحتاجها Module للعمل |
| `demo` | بيانات اختيارية للتجربة والعرض |

تحميل Demo Data يعتمد على إعدادات Database والتثبيت؛ لذلك لا تجعل وظيفة أساسية تعتمد عليها.

#### `application`

تحدد ما إذا كانت Module تمثل Application رئيسية:

```python
"application": True,
```

أما Extension صغيرة أو Module تدريب بسيطة، فيمكن أن تستخدم:

```python
"application": False,
```

هذا التصنيف يؤثر في عرض Module ضمن Apps، لكنه لا ينشئ Menus أوViews تلقائيًا.

بناء الواجهة يحتاج تعريف هذه الأجزاء، كما سنشرح في [[12 Views & XML]] و[[13 Actions, Menus & Buttons]].

#### `installable`

تحدد ما إذا كانت Module قابلة للتثبيت:

```python
"installable": True,
```

أما القيمة `False` فتجعل Module غير قابلة للتثبيت.

> [!important] القابلية للتثبيت ليست حالة التثبيت
> `installable=True` تعني أن Module قابلة للتثبيت.
>
> لا تعني أنها أصبحت **Installed** داخل Database؛ هذه حالة منفصلة تتغير عند تنفيذ Installation.

#### `license`

تحدد License الخاصة بالـModule:

```python
"license": "LGPL-3",
```

نستخدم هذه القيمة في مثال التدريب. في مشروع فعلي، اختر License المناسبة لطريقة استخدام Code وتوزيعها.

#### `assets`

يمكن أن تحدد Manifest أيضًا ملفات Frontend، مثل JavaScript وStyles وXML Templates.

لن نحتاجها في هذا الشابتر. سنشرح طريقة تعريفها واستخدامها في [[24 OWL & Frontend]].

#### Manifest Keys أخرى

توجد Keys إضافية حسب الحاجة، مثل:

- `description` لشرح أطول عن Module.
- `category` لتصنيفها.
- `author` و`website` لمعلومات الجهة المطورة.
- `external_dependencies` للتبعيات الخارجية.
- `auto_install` للتحكم في التثبيت التلقائي بشروط معينة.

وتوجد Hooks لحالات متقدمة أثناء دورة التثبيت والإزالة.

لا نحتاج إلى إضافة هذه الخيارات كلها من البداية؛ استخدم ما يخدم Module فعلًا.

---

### `__init__.py` وImport Chain

الـManifest تصف Module، لكن Python Code تحتاج إلى **Import** حتى تُحمّل.

هنا يأتي دور `__init__.py`.

لنفترض أننا أضفنا لاحقًا هذا التنظيم:

```text
equipment_training/
├── __init__.py
├── __manifest__.py
└── models/
    ├── __init__.py
    └── equipment.py
```

وجود `equipment.py` داخل `models/` لا يجعل Python تستوردها تلقائيًا.

نبدأ من ملف `equipment_training/__init__.py`:

```python
from . import models
```

تعني النقطة `.` الاستيراد من Package الحالية، وهنا نطلب استيراد `models`.

ثم داخل `equipment_training/models/__init__.py`:

```python
from . import equipment
```

هنا نطلب استيراد ملف `equipment.py` الموجود داخل نفس Package.

فتصبح السلسلة:

```text
equipment_training/__init__.py
            ↓
equipment_training/models/__init__.py
            ↓
equipment_training/models/equipment.py
```

إذا انقطعت هذه السلسلة، فلن يصل التحميل إلى الملف المطلوب.

| نوع الملف | كيف نربطه بالـModule؟ |
| --- | --- |
| Python Code | من خلال Import Chain |
| ملفات XML وCSV التي نحتاج تحميلها إلى Database | من خلال القوائم المناسبة في Manifest |

لن نكتب Model الآن؛ الهدف هنا فهم طريقة وصول Odoo إلى Python Files. سنطبق ذلك في [[04 Models & Fields]].

---

## Discovery & Loading

### كيف تكتشف Odoo الـModule؟

تبحث Odoo عن Modules داخل Directories المحددة في إعداد `addons_path`.

في بيئة التدريب، لدينا Directory باسم `custom-addons/`:

```text
custom-addons/
└── equipment_training/
    ├── __init__.py
    └── __manifest__.py
```

المسار الذي نضيفه إلى `addons_path` هو **Directory التي تحتوي Modules**، وهنا هي `custom-addons/`، وليس Directory كل Module على حدة.

تم إعداد المسارات في  [[02 Development Environment]]، وسنستخدم نفس Configuration.

حتى تستطيع Odoo اكتشاف Module، يجب أن تتمكن من الوصول إلى Directory الخاصة بها وقراءة Manifest صحيحة.

لكن **اكتشاف ملفات Module لا يعني تثبيتها**.

---

### تحديث قائمة Modules

تحتفظ Database بقائمة Modules المعروفة لديها.

عند إضافة Module جديدة إلى الملفات، قد تحتاج إلى تحديث هذه القائمة حتى تظهر في الواجهة.

من واجهة Odoo:

1. فعّل Developer Mode من **Settings → Developer Tools → Activate the developer mode**.
2. افتح **Apps**.
3. اختر **Update Apps List**.
4. اضغط **Update**.

هذه العملية تحدث قائمة Modules، لكنها لا تثبت Module ولا تطبق Upgrade عليها.

كذلك، شاشة Apps تستخدم عادةً Filter باسم **Apps**. هذا Filter يعرض Modules المصنفة كتطبيقات رئيسية.

إذا كانت Module تستخدم:

```python
"application": False,
```

احذف Filter **Apps** حتى يشمل البحث Modules الأخرى.

> [!tip] إذا لم تظهر Module
> تحقق من `addons_path` وConfiguration المستخدمة وصحة Manifest، ثم حدّث قائمة Modules وأزل Filter **Apps**.
>
> اختفاء Module من نتائج البحث لا يكفي للحكم بأنها غير موجودة أو غير مثبتة.

عند تنفيذ Installation أوUpgrade من CLI بالطريقة المستخدمة في هذا الشابتر، تحدث Odoo قائمة Modules ضمن العملية؛ لا نحتاج إلى تحديثها يدويًا من الواجهة قبل الأمر.

---

### متى تُحمّل Python Code ومتى تُعالج Data Files؟

كلمة **Loading** قد تشير إلى عمليات مختلفة؛ لذلك نحتاج إلى التمييز بين التشغيل العادي وبين Installation أوUpgrade.

#### أثناء التشغيل العادي

تجهز Odoo الـModules المثبتة المطلوبة للـDatabase، وتحمّل Python Code الخاصة بها وDependencies التابعة لها.

إعادة تشغيل Server تسمح بتحميل Python Code الحالية في Process جديدة.

لكن التشغيل العادي لا يعني إعادة تطبيق جميع Data Files الموجودة في Manifest.

#### أثناء Installation أوUpgrade

تجري Odoo معالجة إضافية للـModule، ويمكن تصور ترتيبها بشكل مبسط:

```text
قراءة Manifest وتحديد Dependencies
                ↓
تحميل Dependencies وPython Code
                ↓
تهيئة أوتحديث Database Structure حسب تعريفات Models
                ↓
معالجة Data Files بالترتيب المحدد
                ↓
اكتمال Installation أوUpgrade
```

هذه صورة مبسطة تكفي الآن لفهم الفرق بين العمليات. سنتعرف على Models نفسها في
[[04 Models & Fields]]، وعلى قواعد تحميل البيانات في [[10 Data Files & External IDs]].

> [!important] Restart لا يعيد تطبيق Data Files
> إعادة تشغيل Odoo تحمّل Python Code، لكنها لا تقوم مقام Module Upgrade عند الحاجة إلى تحديث Database Structure أوتعريفات البيانات.

---

## Module Lifecycle

### Installation مرتبطة بالـDatabase

Source Code موجودة على الجهاز، لكن حالة Installation مرتبطة بكل Database.

مثلًا، يمكن لنفس Server أن يصل إلى ملفات `equipment_training` بينما تكون حالتها:

| Database | حالة Module |
| --- | --- |
| Database A | Installed |
| Database B | غير مثبتة |

لذلك نحتاج دائمًا إلى معرفة أمرين:

- أين توجد Source Code؟
- على أي Database نعمل؟

وجود الملفات وحده لا يضيف Functionality إلى كل Databases تلقائيًا.

---

### Installation

نستخدم **Installation** لإضافة Module إلى Database لأول مرة.

من CLI، يحدد الخيار `-i` اسم Module المراد تثبيتها:

```text
-i equipment_training
```

هذه ليست Command كاملة؛ هي جزء من أمر تشغيل Odoo سنطبقه في Example.

أثناء التثبيت، تتعامل Odoo مع Dependencies المطلوبة، ثم تجهز Module وتعالج تعريفاتها وبياناتها.

---

### Upgrade

نستخدم **Module Upgrade** لتطبيق التغييرات على Module مثبتة مسبقًا.

من CLI، نحددها بالخيار `-u`:

```text
-u equipment_training
```

نحتاج Upgrade عادةً عند تغيير أشياء مثل:

- تعريفات Models وFields.
- Views وData Files.
- تعريفات Security.
- Dependencies داخل Manifest.

Upgrade تعيد معالجة Module وفق ملفاتها الحالية، لكنها ليست ضمانًا بأن أي تغيير في البيانات سيطبق بالطريقة نفسها؛ توجد قواعد سنشرحها في الشابترات الخاصة بها.

> [!note] نوعان مختلفان من Upgrade
> Upgrade للـModule باستخدام `-u` تطبق تغييراتها داخل نفس Odoo Version.
>
> أما الانتقال من Odoo 20 إلى Major Version أخرى، فهو موضوع مختلف نشرحه في [[31 Upgrades & Data Migration]].

---

### Restart وUpgrade

الفرق الأساسي بينهما:

| العملية | الهدف |
| --- | --- |
| Restart | إيقاف Odoo وتشغيل Process جديدة لتحميل Python Code الحالية |
| Module Upgrade | إعادة معالجة Module وتطبيق التغييرات المطلوبة داخل Database |

قاعدة عملية أثناء Development:

| نوع التغيير | الإجراء المعتاد |
| --- | --- |
| تعديل Python Logic فقط | Restart غالبًا يكفي |
| إضافة Field أوتعديل تعريف يحتاج تحديث Database | تحميل Python Code الجديدة ثم Upgrade |
| تعديل XML View أوData File | Upgrade |
| تعديل Security Definition | Upgrade |
| تغيير Dependencies | Upgrade |

#### عند استخدام CLI

أمر Upgrade من CLI يبدأ Process جديدة؛ لذلك يقرأ Python Code الحالية أثناء العملية.

في بيئة التدريب سنوقف Server الجاري أولًا، ثم ننفذ Upgrade باستخدام `--stop-after-init`.

هذا الخيار يوقف Process بعد اكتمال العملية، وبعدها نشغّل Server بشكل طبيعي.

#### عند استخدام الواجهة

زر **Upgrade** لا نعتبره بديلًا عن Restart لتحميل Python Code المعدلة.

إذا عدلت Python Code، أعد تشغيل Server أولًا، ثم نفذ Upgrade من الواجهة عندما يحتاج التغيير ذلك.

لا تفترض أن الضغط على Upgrade سيعيد استيراد كل Python Files داخل Process الحالية.

---

## Example

سننشئ Module بسيطة باسم `equipment_training`، ثم نثبتها ونجرب Upgrade عليها.

يفترض المثال أنك جهزت بيئة العمل في
[[02 Development Environment]]
بما فيها:

- Virtual Environment.
- Odoo Source Code.
- ملف `config/odoo.conf`.
- Directory باسم `custom-addons/` ضمن `addons_path`.
- Database التدريب `odoo20_training`.

لن نضيف Models أوViews الآن؛ نريد أولًا Module صحيحة يمكن تثبيتها وتحديثها.

---

### 1. إنشاء File Structure

داخل `custom-addons/`، أنشئ Directory باسم `equipment_training`، وضع فيها الملفين:

```text
custom-addons/
└── equipment_training/
    ├── __init__.py
    └── __manifest__.py
```

اسم Directory هو Technical Name الذي سنستخدمه في الأوامر.

---

### 2. تجهيز `__init__.py`

اترك ملف `equipment_training/__init__.py` فارغًا.

لا توجد حاليًا Python Files إضافية نحتاج إلى استيرادها.

عندما نضيف `models/` في [[04 Models & Fields]]، سنربطها من هذا الملف.

---

### 3. كتابة Manifest

داخل `equipment_training/__manifest__.py`، اكتب:

```python
{
    "name": "Equipment Training",
    "version": "20.0.1.0.0",
    "summary": "Training module for learning Odoo development",
    "license": "LGPL-3",
    "depends": [
        "base",
    ],
    "data": [],
    "application": False,
    "installable": True,
}
```

اختيارات المثال:

| الإعداد | السبب |
| --- | --- |
| `depends: ["base"]` | التصريح بالاعتماد على Odoo Base |
| `data: []` | لا توجد Data Files بعد |
| `application: False` | Module تدريب بسيطة |
| `installable: True` | السماح بتثبيتها |

احفظ الملف قبل الانتقال إلى Installation.

---

### 4. التأكد من المسار المستخدم

استخدم نفس ملف Configuration الذي أعددناه في [[02 Development Environment]]، وتأكد أن `custom-addons/` موجودة ضمن `addons_path`.

سنشغّل الأوامر التالية من **Project Root** الذي يحتوي:

```text
.venv/
odoo/
config/
custom-addons/
```

إذا كان Server التدريب يعمل في Terminal، أوقفه باستخدام `Ctrl + C` قبل تنفيذ أمر Installation.

---

### 5. تثبيت Module

سنحدد Database باستخدام `-d`، والـModule باستخدام `-i`، ثم نطلب إيقاف Process بعد التثبيت.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_training \
  -i equipment_training \
  --stop-after-init
```

#### Windows

نفّذ الأمر داخل PowerShell:

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\odoo.conf `
  -d odoo20_training `
  -i equipment_training `
  --stop-after-init
```

معاني الخيارات:

| الخيار | وظيفته |
| --- | --- |
| `-c` | تحديد ملف Configuration |
| `-d` | تحديد Database المستهدفة |
| `-i` | تحديد Module المراد تثبيتها |
| `--stop-after-init` | إيقاف Process بعد اكتمال التهيئة والتثبيت |

في PowerShell، علامة Backtick الموجودة في نهاية السطر تسمح بمتابعة الأمر في السطر التالي. يجب أن تكون آخر حرف في السطر دون مسافات بعدها.

بعد اكتمال الأمر، ننتقل إلى تشغيل Server والتحقق من النتيجة داخل الواجهة.

---

### 6. تشغيل Odoo والتحقق من Installation

شغّل Odoo بشكل طبيعي، دون خيارات Installation أوالإيقاف بعد التهيئة.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_training
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\odoo.conf `
  -d odoo20_training
```

افتح المتصفح وانتقل إلى

```
http://127.0.0.1:8069
```

ثم سجل الدخول إلى Database التدريب.

للتحقق:

1. افتح **Apps**.
2. احذف Filter **Apps**.
3. ابحث عن `Equipment Training`.
4. تحقق أن حالة Module هي **Installed**.

هذه النتيجة تؤكد تثبيتها داخل Database التي فتحتها.

إذا لم تجدها، تحقق من Database المستخدمة ونتيجة أمر Installation، ثم راجع خطوات Discovery السابقة.

> [!info] عدم ظهور Equipment Menu متوقع
> ثبتنا Module فقط، ولم ننشئ Models أوViews أوMenus بعد.
>
> حالة **Installed** لا تعني أن واجهة Equipment أصبحت جاهزة.

---

### 7. إجراء تغيير لتجربة Upgrade

سنغير `summary` حتى تكون لدينا نتيجة واضحة يمكن فحصها بعد Upgrade.

داخل `equipment_training/__manifest__.py`، استبدل قيمة `summary` الحالية بهذه القيمة:

```python
"summary": "Practice module for equipment development",
```

لا تضف Key ثانية باسم `summary`؛ عدّل القيمة الموجودة، ثم احفظ الملف.

سنستخدم CLI لتطبيق Upgrade في الخطوة التالية.

---

### 8. تطبيق Upgrade من CLI

أوقف Server التدريب باستخدام `Ctrl + C`.

ثم نفّذ الأمر باستخدام `-u` بدل `-i`.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_training \
  -u equipment_training \
  --stop-after-init
```

  odoo\odoo-bin `
#### Windows

```powershell
.\.venv\Scripts\python.exe `
  -c config\odoo.conf `
  -d odoo20_training `
  -u equipment_training `
  --stop-after-init
```

هنا نطلب من Odoo تحديث Module المثبتة داخل `odoo20_training`.

تبدأ Process جديدة فتقرأ الملفات الحالية، ثم تتوقف بعد اكتمال العملية بسبب `--stop-after-init`.

---

### 9. التحقق من التغيير

بعد اكتمال Upgrade:

1. شغّل Odoo باستخدام أمر التشغيل العادي في الخطوة 6.
2. افتح **Apps** واحذف Filter **Apps**.
3. ابحث عن `Equipment Training`.
4. افتح قائمة Module من زر **⋮**، ثم اختر **Module Info**.
5. تحقق من ظهور Summary الجديدة:

```text
Practice module for equipment development
```

نستخدم **Module Info** للفحص؛ لأن بطاقة Module قد تعرض معلومات مختلفة عند تفعيل Developer Mode.

بهذا ربطنا العملية بنتيجة يمكن التحقق منها:

```text
تعديل Manifest
      ↓
Upgrade
      ↓
التحقق من القيمة الجديدة داخل Odoo
```

---

### 10. البديل: Upgrade من الواجهة

يمكن تنفيذ Upgrade من الواجهة بدل CLI؛ لا تحتاج إلى تنفيذ الطريقتين لنفس التغيير.

لتجربة هذه الطريقة مع تعديل محفوظ في الملفات:

1. أعد تشغيل Server لقراءة الملفات الحالية.
2. افتح **Apps**.
3. احذف Filter **Apps**.
4. ابحث عن `Equipment Training`.
5. افتح قائمة **⋮**، ثم اختر **Upgrade**.
6. بعد اكتمال العملية، تحقق من النتيجة داخل **Module Info**.

![[Upgrade Module.gif]]

> [!important] عند تعديل Python Code
> أعد تشغيل Server قبل Upgrade من الواجهة.
>
> زر Upgrade يعالج تحديث Module، لكنه لا يضمن إعادة تحميل Python Code المعدلة داخل Process الجارية.

---

### 11. ماذا أصبح لدينا؟

لدينا الآن Module:

- Technical Name الخاصة بها `equipment_training`.
- موجودة داخل `custom-addons/`.
- مثبتة في `odoo20_training`.
- جربنا تعديل Manifest وتطبيق Upgrade عليها.
- تحققنا من Installation ومن نتيجة التغيير داخل الواجهة.

لم نضف بعد Equipment List أوForm أوMenu، لأن هذه الأجزاء تحتاج تعريف Models وViews وActions.

سنبدأ في الشابتر التالي ببناء Model والـFields الخاصة بها.

---

## Important Notes

- **وجود الملفات لا يعني Installation.** قد تستخدم Databases نفس Source Code، بينما تختلف حالة Module في كل واحدة منها.

- **`Update Apps List` ليست Install أوUpgrade.** هي تحدث قائمة Modules المعروفة للـDatabase.

- **`application=False` قد تخفي Module خلف Filter Apps.** أزل Filter قبل افتراض أن Odoo لم تكتشفها.

- **Python Files تحتاج Import Chain صحيحة.** وجود الملف وحده داخل `models/` لا يكفي لتحميله.

- **Paths داخل `data` تبدأ من جذر Module.** يجب أن تكون الملفات موجودة، وأن يراعي ترتيبها ما تعتمد عليه من تعريفات سابقة.

- **Restart لا يقوم مقام Upgrade، وزر Upgrade لا يقوم مقام Restart لتحميل Python Code المعدلة.** اختر العملية حسب نوع التغيير.

- **لا تعتمد على Modules مثبتة بالصدفة.** صرّح داخل `depends` بكل Module تحتاج Functionality منها.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[04 Models & Fields]]

---

## Sources

- [Odoo 20 — Architecture Overview](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/01_architecture.html)
- [Odoo 20 — Module Manifests](https://www.odoo.com/documentation/20.0/developer/reference/backend/module.html)
- [Odoo 20 — Command-line Interface](https://www.odoo.com/documentation/20.0/developer/reference/cli.html)
- [Odoo 20 — Apps and Modules](https://www.odoo.com/documentation/20.0/applications/general/apps_modules.html)
- [Odoo 20 — Developer Mode](https://www.odoo.com/documentation/20.0/applications/general/developer_mode.html)
- [Odoo 20 Source — Module Loading](https://github.com/odoo/odoo/blob/20.0/odoo/modules/loading.py)
- [Odoo 20 Source — Python Module Loading](https://github.com/odoo/odoo/blob/20.0/odoo/modules/module.py)
- [Odoo 20 Source — Module Installation and Upgrade](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_module.py)