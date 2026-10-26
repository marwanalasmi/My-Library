---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 32 Translations & Localization

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Translation Fundamentals|Translation Fundamentals]]
- [[#Writing Translatable Text|Writing Translatable Text]]
- [[#Practical Translation Workflow|Practical Translation Workflow]]
- [[#Language, Formatting & RTL|Language, Formatting & RTL]]
- [[#Verification & Troubleshooting|Verification & Troubleshooting]]
- [[#Optional CLI Workflow|Optional CLI Workflow]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

قد تحتاج Module إلى عرض النصوص بالعربية والإنجليزية، مع الحفاظ على السلوك نفسه في اللغتين.

يشمل ذلك:

- Field Labels.
- أسماء القوائم والـActions.
- Selection Labels.
- رسائل الأخطاء.
- نصوص OWL.
- QWeb Reports.

هذه مسؤولية **Translation**.

أما **Localization** فهي أوسع، وقد تشمل إعدادات وتخصيصات مرتبطة ببلد أوبيئة عمل، مثل العملات والتنسيقات والتوطين المالي.

ترجمة الواجهة إلى العربية لا تثبت وحدها اكتمال التوطين أوالامتثال لأي متطلبات خاصة.

في هذا الشابتر سنجهز ترجمة عربية لجزء من مشروع الأجهزة، ثم نفحص النصوص والتنسيق وRTL.

---

## Translation Fundamentals

### ترجمة الواجهة وترجمة المحتوى

هناك فرق بين اسم Field والقيمة المحفوظة فيها.

| المثال | نوع الترجمة |
| --- | --- |
| عرض Label باسم «الوصف» بدل `Description` | ترجمة واجهة |
| حفظ وصف عربي ووصف إنجليزي للجهاز نفسه | ترجمة محتوى Field |
| عرض «متاح» بدل `Available` | ترجمة Selection Label |
| تغيير القيمة التقنية `available` | تغيير في تعريف البيانات، وليس ترجمة Label |

لا نحتاج إلى `translate=True` لمجرد ترجمة Label.

نستخدمها عندما يجب أن تحمل قيمة Field نفسها ترجمات بحسب اللغة.

---

### النصوص التي يستخرجها Odoo

يستطيع Odoo استخراج نصوص من تعريفات مثل:

- `string` و`help` الخاصة بالـFields.
- Labels الخاصة بـSelection المعرفة بصورة مناسبة.
- نصوص وAttributes قابلة للترجمة داخل Views.
- النصوص الثابتة داخل QWeb وOWL Templates.

أما رسائل Python وJavaScript فنحددها باستخدام أدوات الترجمة المناسبة.

الخطوة الأولى هي جعل النص قابلًا للاستخراج، ثم توفير ترجمته.

---

### ملفات `i18n`

نحتفظ بملفات الترجمة داخل Module:

```text
equipment_training/
└── i18n/
    ├── equipment_training.pot
    └── ar_001.po
```

| الملف | الوظيفة |
| --- | --- |
| `.pot` | Template للمصطلحات المستخرجة |
| `.po` | المصطلحات وترجمتها للغة محددة |

سنستخدم في المثال:

```text
ar_001
```

وهو كود العربية العامة في بيانات اللغات الرسمية للفرع الذي راجعناه.

تحقق من كود اللغة التي اخترتها فعلًا؛ العربية العامة والعربية الخاصة بمنطقة ليستا بالضرورة الكود نفسه.

---

### تركيب PO Entry

تحتوي Entry على النص الأصلي والترجمة:

```po
msgid "Equipment Overview"
msgstr "نظرة عامة على الأجهزة"
```

كما يحتوي الملف المصدر من Odoo على معلومات مهمة، مثل:

- Module التي ينتمي إليها النص.
- مصدره داخل Python أوXML أوJavaScript.
- نوع المصطلح.
- Header الخاصة بالملف.

لا تختزل الملف إلى `msgid` و`msgstr` فقط؛ احتفظ بالمعلومات التي ولدها Export.

---

## Writing Translatable Text

### Python

داخل Model Method، استخدم:

```python
raise UserError(
    self.env._(
        "The equipment is not available."
    )
)
```

ضع الرسالة عند الشرط الذي يرفض العملية.

تستخدم `self.env._()` لغة Environment وسياق Module المناسب للبحث عن الترجمة.

---

### المتغيرات داخل الرسالة

اجعل Format String نفسها ثابتة:

```python
message = self.env._(
    "Equipment %(equipment)s is not available.",
    equipment=self.display_name,
)
```

هذا مثال داخل Method تتعامل مع جهاز واحد.

تجنب صياغة النص أولًا ثم طلب ترجمته:

```python
message = self.env._(
    f"Equipment {self.display_name} is not available."
)
```

لأن النص الناتج يتغير بحسب اسم الجهاز، ولا يمثل Literal ثابتة يستطيع Extractor التعامل معها بالطريقة المطلوبة.

كما أن ترجمة الجملة كاملةً تسمح للمترجم بتغيير ترتيب الكلمات والـPlaceholder.

---

### JavaScript وOWL

استخدم:

```javascript
import { _t } from "@web/core/l10n/translation";
```

مثال الرسالة الموجودة في Dashboard:

```javascript
this.error.set(
    _t("Could not load equipment. Please try again.")
);
```

ولتمرير قيمة:

```javascript
const message = _t(
    "Available equipment: %(count)s",
    { count: 5 }
);
```

حافظ على أسماء الـPlaceholders وأنواعها داخل الترجمة.

ولا تكوّن الجملة من أجزاء مترجمة منفصلة إذا كان المطلوب جملة واحدة يمكن أن يختلف ترتيبها بين اللغات.

---

### XML وQWeb وOWL Templates

النص الثابت داخل Template يمكن استخراجه:

```xml
<h1>Equipment Overview</h1>

<button type="button">
    Refresh
</button>
```

نحتفظ بالنص المصدر داخل Template، ونضع العربية داخل PO.

أما القيمة الديناميكية مثل اسم Record فلا تصبح مترجمة تلقائيًا لمجرد عرضها باستخدام:

```xml
<span t-out="o.name"/>
```

ترجمة هذه القيمة تعتمد على تعريف Field والبيانات المتاحة لها.

وإذا احتوى جزء على:

```xml
t-translation="off"
```

فراجع سبب استبعاده قبل توقع ظهور نصوصه في Export.

---

### `translate=True`

عندما تتطلب الوظيفة وصفًا مختلفًا بحسب اللغة، يمكن تعريف Field مثل:

```python
description = fields.Text(
    string="Description",
    translate=True,
)
```

هنا:

- `string` تخص Label.
- `translate=True` تخص المحتوى المحفوظ.

الخاصية لا تترجم النص آليًا؛ تتيح حفظ وقراءة قيم مترجمة.

لا تجعل الأكواد التقنية أوالأرقام التسلسلية أوExternal IDs قابلة للترجمة.

وإذا غيرت خاصية الترجمة لـField موجودة، فاختبر Upgrade والبيانات القديمة وفق [[31 Upgrades & Data Migration]].

---

### Technical Values

تعريف الحالة يبقى مثلًا:

```python
[
    ("available", "Available"),
    ("in_use", "In Use"),
    ("retired", "Retired"),
]
```

تترجم Labels، بينما تستمر Business Logic في استخدام القيم التقنية:

```python
if asset.state == "available":
    ...
```

وكذلك Domain:

```python
[("state", "=", "available")]
```

لا تبنِ المنطق على النص المعروض للمستخدم.

---

## Practical Translation Workflow

### 1. تحديد نطاق التجربة

سنترجم نصوص Dashboard من [[24 OWL & Frontend]]، ومنها:

```text
Equipment Overview
Refresh
Loading equipment...
Available Equipment
Available equipment you can access.
Could not load equipment. Please try again.
```

وسنترجم Label الحقل `reference` الذي أضفناه في الشابتر السابق.

احتفظ ببقية المصطلحات في الملف، ويمكن استكمال ترجمتها تدريجيًا.

---

### 2. إضافة العربية واختيارها

من Settings، افتح قسم Languages واستخدم **Add Languages** لإضافة العربية العامة.

ثم من ملف المستخدم:

1. افتح **My profile**.
2. اختر العربية من **Language**.
3. احفظ الإعداد.
4. أعد تحميل الصفحة إذا احتاجت الواجهة إلى ذلك.

للمقارنة، يمكن الاحتفاظ بحساب تدريب بالإنجليزية وحساب آخر بالعربية، مع صلاحيات وشركات متطابقة.

---

### 3. مراجعة النصوص المصدر

قبل Export:

- تأكد أن رسائل Python تستخدم النصوص الثابتة مع `self.env._()`.
- تأكد أن رسائل JavaScript تستخدم `_t()`.
- تحقق من تحميل Views وTemplates المطلوبة.
- احفظ التغييرات وطبق Restart أوUpgrade المناسبة.

رسالة Dashboard السابقة مكتوبة بالفعل باستخدام `_t()`؛ احتفظ بها.

لا تحتاج إلى تغيير نصوص Template إلى العربية داخل Source Code.

---

### 4. تصدير المصطلحات

فعّل Developer Mode، ثم انتقل إلى مسار تصدير الترجمات:

```text
Settings
    ↓
Translations
    ↓
Import / Export
    ↓
Export Translations
```

قد تختلف طريقة الوصول الظاهرة بحسب الصلاحيات والواجهة.

اختر:

| الخيار | القيمة |
| --- | --- |
| Language | New Language / Empty Template |
| Format | PO |
| Module | `equipment_training` |

صدر الملف، ثم احفظ Template داخل:

```text
equipment_training/i18n/equipment_training.pot
```

بعد ذلك أنشئ ملف العربية من Template، أوصدر ملف اللغة المستهدفة من الواجهة، واحفظه باسم:

```text
equipment_training/i18n/ar_001.po
```

احتفظ بالـHeader، واضبط معلومات اللغة والترميز باستخدام محرر مناسب.

---

### 5. تعبئة الترجمات

داخل ملف العربية المصدر، احتفظ بمراجع كل Entry واملأ `msgstr`.

هذه أمثلة لجزء من الملف:

```po
#. module: equipment_training
#. odoo-javascript
#: code:addons/equipment_training/static/src/dashboard/dashboard.xml:0
msgid "Equipment Overview"
msgstr "نظرة عامة على الأجهزة"

#. module: equipment_training
#. odoo-javascript
#: code:addons/equipment_training/static/src/dashboard/dashboard.xml:0
msgid "Refresh"
msgstr "تحديث"

#. module: equipment_training
#. odoo-javascript
#: code:addons/equipment_training/static/src/dashboard/dashboard.xml:0
msgid "Loading equipment..."
msgstr "جارٍ تحميل الأجهزة..."

#. module: equipment_training
#. odoo-javascript
#: code:addons/equipment_training/static/src/dashboard/dashboard.xml:0
msgid "Available Equipment"
msgstr "الأجهزة المتاحة"

#. module: equipment_training
#. odoo-javascript
#: code:addons/equipment_training/static/src/dashboard/dashboard.xml:0
msgid "Available equipment you can access."
msgstr "الأجهزة المتاحة التي يمكنك الوصول إليها."

#. module: equipment_training
#. odoo-javascript
#: code:addons/equipment_training/static/src/dashboard/dashboard.js:0
msgid "Could not load equipment. Please try again."
msgstr "تعذر تحميل الأجهزة. حاول مرة أخرى."

#. module: equipment_training
#: model:ir.model.fields,field_description:equipment_training.field_equipment_asset__reference
msgid "Reference"
msgstr "المرجع"
```

استخدم Entries التي استخرجها Odoo من مشروعك؛ قد تختلف المسارات أوتجتمع عدة مراجع تحت النص نفسه.

لا تضف Entry مكررة للنص نفسه دون حاجة.

احفظ الملف بترميز UTF-8، وراجع الاقتباسات والأسطر والـPlaceholders قبل تطبيقه.

> [!important] احتفظ بمعلومات المصدر
> التعليقات مثل `odoo-javascript` و`odoo-python` والمراجع ليست مجرد زينة.
>
> يستخدم تحميل ترجمات الكود في الفرع الحالي معلومات النوع والمصدر داخل PO.

---

### 6. تحميل الترجمة من Module

ملفات `i18n` لا تضاف إلى:

```text
Manifest data
Asset Bundles
Python imports
```

ضعها في المجلد الصحيح، ثم:

1. احفظ الملفات.
2. أعد تشغيل Server لقراءة ملفات ترجمات الكود من جديد.
3. افتح **Apps**.
4. احذف Filter **Apps** عند الحاجة.
5. ابحث عن `Equipment Training`.
6. نفذ **Upgrade**.
7. أعد تحميل المتصفح تحميلًا كاملًا.

يساعد Upgrade على تطبيق ترجمات تعريفات Module للغات المحملة.

وفي الكود الحالي، تقرأ ترجمات Python وJavaScript من ملفات PO مع Cache؛ لذلك احتفظ بالملفات ضمن الإصدار المنشور، وأعد تشغيل Processes عند تحديثها.

استيراد PO إلى Database وحده لا يغني عن توفير ملفات ترجمات الكود على Server.

---

### 7. فحص Dashboard

باستخدام الحساب العربي:

1. افتح Dashboard.
2. تحقق من العنوان.
3. تحقق من زر التحديث.
4. تحقق من Label العدد.
5. جرب حالة العدد صفرًا.
6. جرب حالة تحميل بطيء داخل بيئة التدريب.
7. تحقق من رسالة الخطأ عند فشل الطلب.

ثم افتح Form الجهاز وتحقق من ظهور Label:

```text
المرجع
```

مع بقاء الاسم التقني:

```text
reference
```

---

### 8. تحديث الملف بعد تغييرات الكود

عند إضافة نص أوتعديل النص المصدر:

1. صدر Template جديدة.
2. ادمج المصطلحات الجديدة مع PO الموجودة.
3. احتفظ بالترجمات التي ما زالت صالحة.
4. راجع المصطلحات المعدلة أوغير المراجعة.
5. طبق التحديث واختبره.

تغيير Source String قد يجعل الترجمة القديمة غير مطابقة.

لا تعدل `msgid` داخل PO وحدها بينما يبقى النص المصدر مختلفًا.

---

## Language, Formatting & RTL

### إعدادات مختلفة

| الإعداد | مسؤوليته |
| --- | --- |
| Language | النصوص وإعدادات تنسيق مرتبطة باللغة |
| Time Zone | عرض الأوقات وتحويلها بحسب السياق |
| Currency | العملة ومعنى القيم المالية |
| Localization | إعدادات ووظائف مرتبطة بمتطلبات بيئة العمل |

تغيير اللغة لا يمثل قاعدة لتحويل العملات أوتغيير Time Zone.

افحص كل إعداد وفق الحاجة الفعلية.

---

### الأرقام والتواريخ والعملات

استخدم أدوات Formatting وWidgets المناسبة بدل تكوين القيم كنصوص يدويًا.

مثلًا، عرض قيمة مالية يحتاج إلى معرفة العملة، وليس مجرد إضافة رمز إلى رقم.

وفي QWeb Reports، تساعد `t-field` على عرض Fields بتنسيقها المناسب.

راجع [[14 Widgets & View Attributes]] و[[20 QWeb & Reports]].

---

### RTL والنصوص المختلطة

اختبر الواجهة العربية مع:

- Labels طويلة.
- نصوص عربية وإنجليزية في السطر نفسه.
- Serial Numbers.
- URLs.
- أرقام وتواريخ.
- Buttons ضيقة.
- List Columns.

استفد من اتجاه الواجهة، واستخدم تنسيقًا يحترم اتجاه النص.

وقد تحتاج قيمة تقنية إلى اتجاه مستقل داخل التقرير، مثل:

```xml
<span
    t-field="o.serial_number"
    dir="ltr"
/>
```

هذا يعزل عرض الرقم التسلسلي، دون تغيير اتجاه التقرير كاملًا.

اختبر النتيجة الفعلية، خاصةً عند وجود علامات ترقيم وأرقام مختلطة.

---

### لغة التقارير

قد تختلف لغة التقرير المطلوبة عن لغة المستخدم الذي ضغط Print.

إذا احتاج التقرير إلى لغة محددة، يمكن تصميم Document Template مستقلة واستدعاؤها باستخدام `t-lang` على `t-call`.

وعند عرض Fields تحتوي قيمًا مترجمة، قد تحتاج Records إلى Context اللغة المناسبة أيضًا.

افصل بين:

- ترجمة النص الثابت داخل Template.
- لغة قيم Fields المترجمة.
- تنسيق التواريخ والأرقام.
- اتجاه Layout.

واختبر ملف PDF الناتج نفسه، بما فيه الخطوط والالتفاف وترتيب الصفحات.

---

### حدود Localization

الترجمة جزء من تجهيز المنتج لبيئة متعددة اللغات.

أما التوطين فقد يتطلب Modules وإعدادات وقواعد عمل متخصصة.

لا تعتبر ظهور الواجهة بالعربية دليلًا على اكتمال متطلبات البلد أوالتقارير المحلية.

حدد متطلبات التوطين مع المسؤولين عنها، ثم تحقق من تغطية Modules المستخدمة.

---

## Verification & Troubleshooting

### مصفوفة التحقق

| الجزء | ما الذي نختبره؟ |
| --- | --- |
| Field Labels | ظهور الترجمة المناسبة |
| Selection | ترجمة Label مع ثبات Technical Value |
| Python Errors | ظهور رسالة مفهومة باللغة المستهدفة |
| OWL | النصوص والتحميل والخطأ والعدد صفرًا |
| Forms وLists | عدم قص النصوص أوتداخلها |
| Reports | لغة النصوص والقيم |
| PDF | الخطوط وRTL والالتفاف |
| Business Logic | النتيجة نفسها في اللغتين |

اختبر العربية والإنجليزية بالحالة والصلاحيات والشركات نفسها.

---

### عندما لا تظهر الترجمة

راجع:

1. هل اللغة مثبتة ومختارة للحساب أوالسياق؟
2. هل النص موجود فعلًا في Export؟
3. هل `msgid` تطابق Source String؟
4. هل `msgstr` معبأة؟
5. هل الملف في `i18n` وباسم مناسب؟
6. هل مراجع Entry وتعليقات النوع سليمة؟
7. هل Server تستخدم نسخة Module التي عدلتها؟
8. هل أعدت تشغيل Processes وتحميل المتصفح؟
9. هل توجد ترجمة محفوظة سبق تعديلها وتحتاج إلى قرار بشأن استبدالها؟

لا تفعّل خيار Overwrite دون معرفة الترجمات التي تريد اعتمادها كمصدر.

---

### Backend وWebsite/Portal

Dashboard الخاصة بمشروعنا تعمل داخل Web Client الداخلي.

في الكود الحالي، يحمل Web Client ترجمات Modules المحملة ضمن آليته.

أما JavaScript الخاصة بـWebsite أوPortal فقد تستخدم نطاق تحميل مختلفًا للترجمات.

إذا ظهرت المشكلة هناك، افحص سياق التحميل وآلية تضمين Module قبل إضافة Overrides.

لا تطبق إرشادًا خاصًا بصفحة Website على Dashboard داخلية لمجرد أن كليهما يستخدم JavaScript.

---

## Optional CLI Workflow

### تصدير POT

يمكن استخدام أوامر `i18n` في Odoo 20 بدل التصدير من الواجهة.

من مجلد `odoo20-dev`:

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin i18n export equipment_training \
  -c config/odoo.conf \
  -d odoo20_training \
  --languages pot
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin i18n export equipment_training `
  -c config\odoo.conf `
  -d odoo20_training `
  --languages pot
```

يصدر الأمر Template داخل مجلد `i18n` الخاص بالـModule.

راجع التغيير قبل اعتماد الملف، خاصةً إذا كانت Template موجودة سابقًا.

وللتصدير بلغة مثبتة، استخدم كودها بدل `pot`، مثل:

```text
--languages ar_001
```

هذه أوامر تصدير؛ استكمال الترجمة وتطبيقها والتحقق منها يظل جزءًا من Workflow الشابتر.

---

## Important Notes

- Translation وLocalization مفهومان مختلفان.
- ترجمة Field Label تختلف عن ترجمة محتوى Field.
- `translate=True` تتيح قيمًا مترجمة، ولا تنشئ الترجمة آليًا.
- استخدم Literal Strings قابلة للاستخراج.
- مرر المتغيرات كـPlaceholders، واحتفظ بها داخل الترجمة.
- لا تبنِ Business Logic على Labels مترجمة.
- احتفظ بالقيم التقنية والأكواد والأرقام التسلسلية مستقرة.
- احفظ PO بترميز UTF-8 مع مراجعها وتعليقات النوع.
- لا تضف PO إلى Manifest Data أوAsset Bundle.
- انشر ملفات ترجمات الكود مع Module، وراعِ Cache وRestart.
- اختلاف كود اللغة أوالنص المصدر قد يمنع مطابقة الترجمة.
- راجع أثر Overwrite على الترجمات الموجودة.
- اختبر RTL والنصوص المختلطة، وليس وجود النص العربي فقط.
- لغة التقرير ولغة قيمه واتجاهه أجزاء تحتاج إلى ضبط وفحص.
- اختبر PDF الفعلية إلى جانب واجهة المتصفح.
- تحقق من أن السلوك والصلاحيات ثابتان عند تغيير اللغة.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[33 Maintenance]]

---

## Sources

- [Odoo 20 Documentation — Translating Modules](https://www.odoo.com/documentation/20.0/developer/howtos/translations.html)
- [Odoo 20 Documentation — Change Languages](https://www.odoo.com/documentation/20.0/applications/general/users/language.html)
- [Odoo 20 Documentation — QWeb Reports](https://www.odoo.com/documentation/20.0/developer/reference/backend/reports.html)
- [Odoo 20 Documentation — Internationalization CLI](https://www.odoo.com/documentation/20.0/developer/reference/cli.html#i18n-internationalization)
- [Odoo 20 Official Source — Translation Extraction and Loading](https://github.com/odoo/odoo/blob/20.0/odoo/tools/translate.py)
- [Odoo 20 Official Source — JavaScript Translation](https://github.com/odoo/odoo/blob/20.0/addons/web/static/src/core/l10n/translation.js)
- [Odoo 20 Official Source — Language Definitions](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/data/res.lang.csv)
- [Odoo 20 Official Source — Web Translation Loading](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/models/ir_http.py)