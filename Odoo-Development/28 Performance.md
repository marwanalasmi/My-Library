---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 28 Performance

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Measurement Fundamentals|Measurement Fundamentals]]
- [[#Profiling Tools|Profiling Tools]]
- [[#ORM Performance Patterns|ORM Performance Patterns]]
- [[#Practical Example|Practical Example]]
- [[#Verification|Verification]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

عندما تصبح شاشة أوعملية بطيئة، نحتاج إلى معرفة أين يذهب الوقت قبل اختيار التعديل.

قد يكون السبب:

- استعلامًا بطيئًا.
- عددًا كبيرًا من الاستعلامات الصغيرة.
- تكرار الحساب داخل Python.
- تحميل بيانات أكثر من المطلوب.
- إعادة حساب Fields كثيرة.
- انتظار خدمة خارجية.
- Rendering مكلفة داخل المتصفح.

لذلك نبدأ بهذا المسار:

```text
قياس
  ↓
تحديد الجزء المكلف
  ↓
تعديل محدد
  ↓
التحقق من صحة النتيجة
  ↓
إعادة القياس
```

في هذا الشابتر سنطبق ذلك على حساب عدد الإعارات لكل جهاز، باستخدام Models الموجودة في مشروع التدريب.

---

## Measurement Fundamentals

### تحديد العملية البطيئة

وصف مثل «Odoo بطيئة» لا يكفي لبناء مقارنة مفيدة.

حدد عملية واضحة:

> حساب عدد الإعارات لمئة جهاز يستغرق وقتًا أطول من المطلوب.

ثم سجل:

| العنصر | ما نحتاج إلى معرفته |
| --- | --- |
| العملية | الشاشة أوMethod التي نقيسها |
| البيانات | عدد الأجهزة والإعارات وتوزيعها |
| الحساب | المستخدم وصلاحياته |
| الشركات | النطاق المستخدم أثناء القياس |
| الزمن | المدة التي تستغرقها العملية |
| SQL | عدد الاستعلامات والوقت المستغرق فيها |
| الموارد | استهلاك الذاكرة أوCPU عند ارتباطهما بالمشكلة |

ابدأ ببيانات تدريب تمثل حجم الاستخدام الذي تريد دعمه.

---

### تحديد الطبقة المكلفة

| الملاحظة | ما الذي نفحصه؟ |
| --- | --- |
| طلب Server بطيء | Python وSQL والانتظار الخارجي |
| استعلامات صغيرة متكررة | Batching واحتمال N+1 |
| استعلام واحد مكلف | Domain وGrouping وIndexes وخطة التنفيذ |
| Server تنتهي سريعًا والشاشة تتأخر | حجم Response وJavaScript وRendering |
| استهلاك ذاكرة مرتفع | حجم Recordsets والنتائج والبيانات المحملة |
| البطء يظهر عند الكتابة | Constraints وComputes وTracking وأعمال إضافية |

استخدم أدوات [[27 Debugging]] لتحديد المسار الصحيح قبل بدء التحسين.

---

### مقارنة متكافئة

قارن التنفيذين باستخدام:

- البيانات نفسها.
- المستخدم نفسه.
- الشركات نفسها.
- إعدادات تشغيل متقاربة.
- حالة Cache معروفة.
- عدد تكرارات مناسب.

قد يكون أول تشغيل أبطأ بسبب تحميل بيانات أوتعريفات إلى Cache.

فرق بين:

| الحالة | المعنى |
| --- | --- |
| Cold Run | تشغيل قبل تجهيز Cache المطلوبة |
| Warm Run | تشغيل بعد تجهيز جزء من البيانات والتعريفات |

لا تقارن تشغيلًا باردًا لتنفيذ قديم بتشغيل دافئ لتنفيذ جديد ثم تنسب الفرق كله إلى التعديل.

ولا تعتمد على محاولة واحدة متأثرة بعمل آخر على الجهاز.

---

## Profiling Tools

### Profiler من الواجهة

توفر Odoo أداة Profiling لتسجيل معلومات عن تنفيذ Requests.

في بيئة تدعم الأداة وبالصلاحيات المناسبة:

1. فعّل Developer Mode.
2. افتح Developer Tools.
3. اختر **Enable profiling**.
4. فعّل الإتاحة العامة للـProfiler لمدة محددة إذا طلبت الواجهة ذلك.
5. فعّل Profiling للجلسة.
6. نفذ العملية التي تريد تحليلها.
7. أوقف التسجيل بعد التقاط الحالة.

ابدأ بتسجيل عملية محددة، حتى يسهل ربط النتيجة بما نفذته.

يمكن فتح نتائج الجلسة وتحليلها باستخدام عرض Speedscope المتاح في Odoo.

---

### Collectors

تحدد Collectors نوع البيانات المسجلة.

| Collector | ما الذي تساعد على فحصه؟ |
| --- | --- |
| SQL | الاستعلامات ومواضع استدعائها |
| Periodic | عينات دورية من مسار التنفيذ |
| QWeb | تكلفة Directives في Server QWeb |

اختر الأدوات وفق المشكلة.

تكرار SQL متشابهة قد يكشف مشكلة Batching، بينما الوقت الطويل داخل Python دون SQL كثيرة قد يشير إلى حساب أوتكرار مكلف.

QWeb Collector لا تعني قياس كل Rendering تحدث في OWL داخل المتصفح.

---

### أثر Profiler على القياس

التسجيل نفسه يضيف تكلفة، خصوصًا عند التقاط تفاصيل عدد كبير من الاستعلامات.

لذلك نفصل بين هدفين:

```text
Profiler
→ معرفة أين توجد التكلفة

قياس دون Profiler
→ معرفة أثر التعديل على الزمن
```

لا تحول كل فرق يظهر أثناء التسجيل إلى ادعاء عن سرعة الاستخدام الفعلية.

---

## ORM Performance Patterns

### N+1 Queries

تظهر مشكلة N+1 عندما ننفذ بحثًا أوليًا، ثم استعلامًا إضافيًا لكل Record.

مثال:

```text
تحميل الأجهزة
      ↓
الجهاز الأول → عد الإعارات
الجهاز الثاني → عد الإعارات
الجهاز الثالث → عد الإعارات
...
```

كلما زاد عدد الأجهزة، زاد عدد عمليات العد.

قد لا تظهر المشكلة بوضوح مع جهازين، لكنها تصبح مكلفة مع مئات الأجهزة.

الحل المناسب في مثالنا هو تجميع الإعارات لكل الأجهزة المطلوبة في عملية واحدة.

---

### Batch Operations

حاول تنفيذ العمل على Recordset عندما تسمح قواعد العملية بذلك.

من أمثلة Batching:

- تجميع أعداد Records المرتبطة.
- إنشاء عدة Records باستخدام قائمة قيم.
- كتابة قيمة مشتركة على Recordset.
- تجهيز Mapping بدل إعادة البحث عن النتيجة داخل كل Loop.

لكن راجع Business Methods وOverrides قبل تغيير طريقة الاستدعاء.

التعديل يجب أن يحافظ على Validation والصلاحيات والآثار المطلوبة، مثل Tracking.

---

### `_read_group()`

تسمح `_read_group()` بتجميع البيانات داخل Database.

مثلًا:

```python
rows = Loan._read_group(
    [("asset_id", "in", assets.ids)],
    groupby=["asset_id"],
    aggregates=["__count"],
)
```

في هذا المثال:

| الجزء | الوظيفة |
| --- | --- |
| Domain | اختيار إعارات الأجهزة المطلوبة |
| `groupby` | تكوين مجموعة لكل جهاز |
| `__count` | حساب عدد الإعارات في كل مجموعة |

تعيد Method قائمة Tuples.

عند التجميع بواسطة Many2one مثل `asset_id`، تكون قيمة المجموعة Recordset للجهاز، يليها العدد.

```text
(asset_record, count)
```

هذه واجهة Python داخل Server؛ لا تستخدم اسم Method التي تبدأ بشرطة سفلية كاستدعاء خارجي من JSON-2.

---

### Prefetching

تستخدم ORM الـPrefetching لتجهيز بيانات مجموعة Records عند الوصول إلى Fields، بدل التعامل مع كل Record كعملية مستقلة دائمًا.

حافظ على Recordset المجمعة:

```python
assets = Asset.browse(asset_ids)

names = [
    asset.name
    for asset in assets
]
```

تجنب إعادة بناء Singleton مستقلة داخل كل دورة دون حاجة.

ولا تفترض أن كل وصول إلى Field ينفذ SQL جديدة؛ قد تكون القيمة موجودة في Cache.

وفي المقابل، Prefetching لا تجمع تلقائيًا كل استدعاءات `search_count()` المستقلة التي تضعها داخل Loop.

---

### Domains وحجم النتائج

إذا كانت Database تستطيع تنفيذ شرط البحث، مرره إليها بدل تحميل مجموعة كبيرة ثم تصفيتها في Python.

```python
assets = Asset.search(
    [("state", "=", "available")],
    order="id",
    limit=100,
)
```

القيمة الصحيحة لحالة الجهاز في مشروعنا هي `available`.

اختر العملية وفق المطلوب:

| المطلوب | نقطة البداية |
| --- | --- |
| عدد Records | `search_count()` |
| Records للعمل عليها | `search()` |
| بيانات محددة للإرجاع | `search_read()` مع Fields محددة |
| نتائج مجمعة | `_read_group()` |

استخدم `limit` عندما تسمح Requirement بنتيجة جزئية أوPagination.

لا تضفه إلى حساب إجمالي أوتجميع كامل إذا كان سيحذف جزءًا مطلوبًا من النتيجة.

---

### Indexes

قد تساعد Index على بعض عمليات البحث والربط والترتيب، حسب Query وحجم البيانات وتوزيعها.

يمكن طلب Index ضمن تعريف Field:

```python
index=True
```

لكن وجود Index لا يضمن أن Database ستستخدمها أوأنها ستجعل Query أسرع.

لها تكلفة في التخزين والكتابة والصيانة.

قبل إضافة Index:

1. حدد الاستعلام المكلف.
2. افحص الفلاتر والربط والترتيب.
3. تحقق من Indexes الموجودة.
4. قارن التنفيذ على بيانات ممثلة.
5. قس أثر التغيير على القراءة والكتابة.

راجع Index الموجودة على `serial_number` في مشروعنا قبل اقتراح إضافة أخرى لها.

---

### Stored Computed Fields

تخزين Computed Field قد يجعل البحث والتجميع ممكنين بصورة مباشرة، لكنه ينقل جزءًا من التكلفة إلى عمليات الكتابة وإعادة الحساب.

اسأل:

- كم مرة نقرأ القيمة؟
- كم مرة تتغير Dependencies؟
- كم Record ستحتاج إلى إعادة الحساب؟
- هل القيمة تعتمد على المستخدم أوContext؟

لا تحول إحصائية تختلف حسب صلاحيات المستخدم إلى قيمة مخزنة مشتركة دون تصميم يضمن صحة معناها وحماية البيانات.

راجع [[16 Computed Fields, Related Fields & Onchange]].

---

## Practical Example

### 1. تحديد المطلوب

نريد Dictionary تحتوي عدد الإعارات لكل جهاز ضمن Recordset محددة.

يعتمد المثال على:

```text
equipment.asset
       ↑
       │ asset_id
       │
equipment.loan
```

سنحسب الإعارات التي يستطيع المستخدم الحالي قراءتها.

هذا يعني أن العدد قد يختلف بين Equipment Manager ومستخدم يرى إعاراته فقط.

لن نضيف Field أوMenu جديدة؛ سنضيف Helper Method داخل Model الأجهزة.

---

### 2. فهم التنفيذ المتكرر

هذا شكل التنفيذ الذي نريد مقارنته:

```python
def count_loans_one_by_one(assets):
    if not assets:
        return {}

    assets.check_access("read")

    Loan = assets.env["equipment.loan"]

    return {
        asset.id: Loan.search_count([
            ("asset_id", "=", asset.id),
        ])
        for asset in assets
    }
```

سنستخدم هذه الدالة داخل Shell للمقارنة.

المشكلة أنها تستدعي `search_count()` مرة لكل جهاز.

وجود Python Loop ليس المشكلة وحده؛ التكلفة المهمة هنا هي العمل المتكرر على Database داخلها.

---

### 3. إضافة التنفيذ المجمع

داخل Class الحالية الخاصة بـ`equipment.asset`، أضف:

```python
def _get_loan_counts(self):
    if not self:
        return {}

    self.check_access("read")

    rows = self.env["equipment.loan"]._read_group(
        [("asset_id", "in", self.ids)],
        groupby=["asset_id"],
        aggregates=["__count"],
    )

    counts_by_asset = {
        asset.id: count
        for asset, count in rows
    }

    return {
        asset.id: counts_by_asset.get(asset.id, 0)
        for asset in self
    }
```

احتفظ ببقية تعريف Model وMethods الموجودة.

المنطق:

```text
التحقق من قراءة الأجهزة
          ↓
تجميع الإعارات حسب الجهاز
          ↓
تحويل النتائج إلى Mapping
          ↓
إرجاع عدد لكل جهاز، بما في ذلك الصفر
```

استخدام `get(..., 0)` مهم؛ لأن الجهاز الذي لا توجد له إعارات مطابقة لا يظهر كمجموعة في نتائج البحث.

لم نستخدم `sudo()`، لذلك تبقى صلاحيات الإعارات وسياق الشركات جزءًا من النتيجة.

---

### 4. تجهيز جلسة المقارنة

احفظ التعديل، ثم افتح Odoo Shell جديدة على Database التدريب بالطريقة التي تعلمناها في [[27 Debugging]].

استخدم حساب تدريب محددًا:

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

assets = Asset.search(
    [("state", "=", "available")],
    order="id",
    limit=100,
)

if not assets:
    raise UserError(
        "Create available training equipment before measuring."
    )
```

تأكد من أن الحساب يستطيع قراءة الإعارات، وأن بيانات التدريب تتضمن حالات مناسبة للمقارنة.

المثال يستخدم الشركة الرئيسية للحساب. عدل النطاق بصورة صريحة عند اختبار عدة شركات.

---

### 5. التحقق من تطابق النتائج

عرّف دالة المقارنة السابقة داخل Shell، ثم نفذ:

```python
baseline_counts = count_loans_one_by_one(assets)

grouped_counts = assets._get_loan_counts()

assert baseline_counts == grouped_counts
```

نفذ المقارنة على بيانات مستقرة، حتى لا تتغير الإعارات بين الاستدعاءين.

تطابق التنفيذين خطوة أولى، ثم نتحقق لاحقًا من أعداد معروفة داخل بيانات الاختبار.

لا نريد تسريع عملية تعيد نتيجة ناقصة أوتكشف Records إضافية.

---

### 6. قياس استعلامات SQL

داخل Shell:

```python
from odoo.tools.profiler import Profiler, SQLCollector


def capture_queries(callback):
    collector = SQLCollector()

    with Profiler(
        collectors=[collector],
        db=None,
    ):
        result = callback()

    return result, len(collector.entries)
```

ثم:

```python
baseline_result, baseline_queries = capture_queries(
    lambda: count_loans_one_by_one(assets)
)

grouped_result, grouped_queries = capture_queries(
    lambda: assets._get_loan_counts()
)

assert baseline_result == grouped_result

print({
    "assets": len(assets),
    "baseline_queries": baseline_queries,
    "grouped_queries": grouped_queries,
})
```

استخدمنا `db=None` للاحتفاظ بالتسجيل في الذاكرة بدل حفظ Profile في Database.

المقارنة السابقة جهزت جزءًا من Cache لكلا المسارين. حافظ على سياسة قياس متشابهة عند إعادة التجربة.

الناتج المتوقع هو تقليل العمل المتكرر المرتبط بالعد.

لا نفترض أن إجمالي التنفيذ أصبح استعلامًا واحدًا دائمًا؛ قد توجد استعلامات للصلاحيات أوتجهيز البيانات أوأعمال أخرى.

---

### 7. قياس الزمن دون Profiler

داخل Shell:

```python
from statistics import median
from time import perf_counter


def measure_seconds(callback, repeats=5):
    callback()

    samples = []

    for _ in range(repeats):
        started = perf_counter()
        callback()
        samples.append(perf_counter() - started)

    return median(samples)
```

ثم:

```python
baseline_seconds = measure_seconds(
    lambda: count_loans_one_by_one(assets)
)

grouped_seconds = measure_seconds(
    lambda: assets._get_loan_counts()
)

print({
    "assets": len(assets),
    "baseline_median_seconds": baseline_seconds,
    "grouped_median_seconds": grouped_seconds,
})
```

تتضمن الدالة تشغيلًا تمهيديًا، ثم تعرض وسيط عدة قياسات.

هذا يقيس Method داخل Shell، ولا يمثل زمن فتح صفحة كاملة من المتصفح.

سجل نتائجك الفعلية، ثم كرر التجربة بأحجام بيانات مختلفة.

| المقياس | التنفيذ المتكرر | التنفيذ المجمع |
| --- | --- | --- |
| عدد الأجهزة | العدد نفسه | العدد نفسه |
| نتيجة العد | نتيجة مرجعية | يجب أن تطابقها |
| SQL Queries | القياس الفعلي | القياس الفعلي |
| الزمن الوسيط | القياس الفعلي | القياس الفعلي |

---

## Verification

### صحة النتائج والصلاحيات

جهز حالات ذات نتائج معروفة:

| الحالة | المتوقع |
| --- | --- |
| جهاز بلا إعارات | صفر |
| جهاز له إعارتان مسموحتان | اثنتان |
| مجموعة أجهزة | عدد صحيح لكل جهاز |
| Recordset فارغة | Dictionary فارغة |
| مستخدم يرى بعض الإعارات فقط | احتساب الإعارات المسموحة |
| جهاز خارج نطاق الوصول | رفض الوصول عند تمريره مباشرةً |

قارن هذه الحالات بالـRequirement، وليس بالتنفيذ القديم فقط.

---

### النمو مع حجم البيانات

كرر القياس مع مجموعات أكبر، مثل:

```text
10 أجهزة
50 جهازًا
100 جهاز
```

راقب كيف يتغير:

- عدد الاستعلامات.
- الزمن.
- حجم النتائج والذاكرة.
- أثر عدد الإعارات لكل جهاز.

الهدف أن تفهم سلوك النمو، وليس الحصول على رقم صغير في تجربة واحدة.

إذا أصبحت Recordset نفسها كبيرة جدًا، فقد تحتاج إلى Batches مناسبة، كما تعلمنا في [[22 Automation & Scheduled Actions]].

---

### Regression Tests

احتفظ باختبارات صحة النتائج والصلاحيات وفق [[26 Testing]].

ويمكن استخدام أدوات مثل:

```python
self.assertQueryCount(...)
```

لمراقبة عدد الاستعلامات في سيناريو مضبوط.

حدد Budget بعد القياس، وثبت بيانات الاختبار وسياق المستخدم وWarm-up المطلوبة.

لا تستخدم زمنًا صغيرًا جدًا كشرط نجاح هش يتأثر بسهولة بسرعة الجهاز أوالعمليات الأخرى.

---

### قبول التحسين

اعتمد التعديل عندما تثبت المقارنة أنه:

- يحافظ على النتيجة المطلوبة.
- يحافظ على قواعد الوصول.
- يحسن المقياس الذي حددته.
- لا ينقل تكلفة كبيرة غير مقبولة إلى الكتابة أوالذاكرة.
- يعمل على حجم بيانات يمثل الاستخدام المستهدف.

إذا قل عدد Queries دون تحسن الزمن، افحص تكلفة Query المجمعة قبل افتراض نجاح التحسين.

---

## Important Notes

- ابدأ بقياس عملية محددة وبيانات ممثلة.
- فرق بين زمن Server وزمن الصفحة الكامل.
- حافظ على المستخدم والشركات وDataset عند المقارنة.
- ميز بين Cold Run وWarm Run.
- استخدم Profiler لتحديد السبب، ثم قس الزمن بدونه.
- Loop ليست مشكلة تلقائيًا؛ افحص العمل الذي تكرره.
- Prefetching لا تلغي كل استعلام مستقل داخل Loop.
- `_read_group()` مفيدة للتجميع، وتعيد Tuples وفق ترتيب Grouping وAggregates.
- في مثالنا، التجميع بواسطة `asset_id` يعيد Recordset للجهاز مع العدد.
- أعد صفرًا للأجهزة التي لا تظهر ضمن المجموعات.
- لا تضف `limit` إذا كانت Requirement تحتاج إلى نتيجة كاملة.
- Index وStored Compute لهما تكلفة يجب قياسها.
- لا تستخدم `sudo()` لتغيير نطاق البيانات بهدف تحسين الأداء.
- الإحصائية المقيدة بصلاحيات المستخدم يجب أن تحافظ على هذا المعنى بعد التحسين.
- انخفاض عدد Queries لا يضمن وحده تحسن الزمن.
- لا تعتمد على أرقام سرعة افتراضية؛ سجل قياسات بيئتك.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[29 Deployment]]

---

## Sources

- [Odoo 20 Documentation — Performance](https://www.odoo.com/documentation/20.0/developer/reference/backend/performance.html)
- [Odoo 20 Documentation — ORM API](https://www.odoo.com/documentation/20.0/developer/reference/backend/orm.html)
- [Odoo 20 Documentation — Computed Fields and Onchanges](https://www.odoo.com/documentation/20.0/developer/tutorials/server_framework_101/08_compute_onchange.html)
- [Odoo 20 Official Source — ORM Models and Grouping](https://github.com/odoo/odoo/blob/20.0/odoo/orm/models.py)
- [Odoo 20 Official Source — Profiler and SQL Collector](https://github.com/odoo/odoo/blob/20.0/odoo/tools/profiler.py)
- [Odoo 20 Official Source — Query Count Test Helpers](https://github.com/odoo/odoo/blob/20.0/odoo/tests/common.py)