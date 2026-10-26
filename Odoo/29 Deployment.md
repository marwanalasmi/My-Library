---
type: learning
odoo_version: "20.0"
status: planned
created: 2026-10-13
---
# 29 Deployment

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Environments & Hosting|Environments & Hosting]]
- [[#Production Architecture|Production Architecture]]
- [[#Release Preparation|Release Preparation]]
- [[#Practical Deployment Flow|Practical Deployment Flow]]
- [[#Verification & Rollback|Verification & Rollback]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

تعلمنا كيف نشغّل Odoo محليًا، ونبني Modules، ونختبر السلوك، ونقيس الأداء.

**Deployment** هي نقل إصدار محدد إلى بيئة تشغيل، مع تجهيز طريقة تشغيله وتحديثه ومراقبته واستعادته عند الحاجة.

الإصدار المنشور يتكون من أجزاء مترابطة:

```text
Odoo Source
+
Custom Modules
+
Dependencies
+
Configuration
+
Database
+
Filestore
```

نجاح تشغيل Process لا يثبت وحده نجاح النشر.

نحتاج أيضًا إلى التأكد من أن المستخدمين يستطيعون تنفيذ العمليات المطلوبة، وأن الملفات والتقارير والتكاملات تعمل ضمن الصلاحيات الصحيحة.

سنشرح في هذا الشابتر مسار النشر، ونستخدم مثال Self-hosted على Linux لتوضيح تنظيم الملفات والإعدادات.

---

## Environments & Hosting

### بيئات العمل

| البيئة | الغرض |
| --- | --- |
| Development | تطوير الكود والتجارب والتشخيص |
| Test | اختبارات آلية ببيانات وإعداد قابلين للتكرار |
| Staging | تجربة الإصدار والتحديثات في بيئة قريبة من Production |
| Production | تشغيل المستخدمين والبيانات الفعلية |

وجود Test ناجحة لا يغني عن تجربة النشر على Staging.

قد ينجح Python Test، بينما يفشل تشغيل تقرير بسبب تبعية ناقصة، أوتفشل Assets بسبب إعداد مختلف.

---

### Staging وNeutralization

يمكن تجهيز Staging من نسخة من بيانات Production للحصول على تجربة أكثر تمثيلًا للاستخدام الفعلي.

قبل تشغيل النسخة، عالج الاتصالات والعمليات التي قد تؤثر في أنظمة حقيقية، مثل:

- Outgoing Email.
- Payment Providers.
- Shipping Integrations.
- Webhooks.
- Scheduled Actions.
- API Keys الخاصة بالإنتاج.

توفر Odoo آلية **Neutralization** لتعطيل إعدادات تشغيلية محددة في النسخ المخصصة للاختبار.

لكن التكامل المخصص قد يحتاج إلى معالجة إضافية.

> [!important] النسخة المستعادة تحتاج إلى فحص
> نسخ Database أوتغيير اسمها لا يثبت أن كل اتصال خارجي أصبح معزولًا.
>
> تحقق من Neutralization، وراجع Custom Modules والـCredentials قبل السماح للنسخة بتنفيذ أعمال مجدولة أوطلبات خارجية.

قيد الوصول إلى Staging أيضًا، خاصةً إذا كانت تحتوي نسخة من بيانات فعلية.

---

### خيارات الاستضافة

| الخيار | ما الذي تديره المنصة؟ | ما الذي يهم مشروعنا؟ |
| --- | --- | --- |
| Odoo Online | تشغيل Odoo والبنية المستضافة | لا تدعم تثبيت Custom Python Modules الخاصة بمشروعنا |
| Odoo.sh | بيئة تستند إلى Git مع أدوات بناء واختبار وتشغيل | تدعم مسار تطوير ونشر Custom Modules |
| Self-hosted | أنت أوفريق التشغيل تديرون البنية | تحكم أوسع ومسؤولية مباشرة عن التشغيل والتحديث والاستعادة |

مشروع `equipment_training` يحتوي Python Modules مخصصة؛ لذلك نختار بيئة تدعمها.

في Odoo.sh، يرتبط النشر بمسار الفروع والبناء الخاص بالمنصة.

أما المثال التالي فيشرح تنظيم Self-hosted على Linux.

---

## Production Architecture

### مسار الطلب والتخزين

في مثال Server واحدة:

```text
Browser / API Client
         ↓ HTTPS
    Reverse Proxy
         ↓
        Odoo
         ├── PostgreSQL → Database
         └── Filesystem → Filestore
```

Database وFilestore مخزنان مرتبطان.

PostgreSQL تحفظ بيانات النظام، بينما تحفظ Filestore ملفات المرفقات التي تستخدم التخزين على القرص.

لذلك يجب الحفاظ على اتساقهما عند النسخ والاستعادة.

ولا تنشر مجلد Filestore كمسار Static عام؛ الوصول إلى المرفقات يجب أن يحافظ على قواعد السماح الخاصة بها.

---

### Reverse Proxy وHTTPS

يمكن استخدام Reverse Proxy مثل Nginx لاستقبال الاتصال العام ثم توجيهه إلى Odoo.

من مسؤولياتها:

- تقديم HTTPS بشهادة صالحة.
- توجيه Domain إلى الخدمة الصحيحة.
- تمرير Headers اللازمة.
- التعامل مع WebSocket عند استخدامها.
- ضبط حدود الطلبات والمهل بما يناسب الاستخدام.

في مثالنا، تعمل Proxy وOdoo على Server نفسها، وتصل Proxy إلى Odoo عبر Loopback.

```text
https://odoo.example.com
           ↓
      Reverse Proxy
           ↓
    127.0.0.1:8069
```

إذا كانت Proxy داخل Container أوServer أخرى، فاستخدم عنوانًا داخليًا يمكنها الوصول إليه، مع تقييد الشبكة وفق التصميم.

---

### `proxy_mode`

نفعّل:

```ini
proxy_mode = True
```

عندما تعمل Odoo خلف Proxy موثوقة ومهيأة لتمرير معلومات Host وProtocol وClient IP بصورة صحيحة.

هذا الإعداد يجعل Odoo تتعامل مع Headers الممررة من Proxy.

يجب أن يتوافق معه مسار الشبكة؛ لا تسمح باتصالات عامة تتجاوز Proxy وتصل إلى الخدمة الداخلية مباشرةً.

---

### Workers وWebSocket

في وضع Multi-process، توجد HTTP Workers لمعالجة الطلبات، وعملية مخصصة للاتصالات المعتمدة على Gevent.

في المثال المعتاد:

| نوع الطلب | الوجهة الداخلية |
| --- | --- |
| HTTP Requests المعتادة | `127.0.0.1:8069` |
| مسار `/websocket` | Gevent Port، مثل `127.0.0.1:8072` |

تحتاج Proxy إلى دعم WebSocket Upgrade Headers وتوجيه المسار إلى الوجهة المناسبة.

نجاح صفحة تسجيل الدخول لا يثبت صحة هذا الجزء؛ اختبر الوظائف التي تعتمد على التحديثات الفورية.

> [!note] نظام التشغيل
> وضع Multi-process الخاص بـOdoo غير متاح على Windows.
>
> يمكن متابعة التطوير المحلي على Windows، بينما يوضح هذا الشابتر نموذج تشغيل Linux.

---

### اختيار عدد Workers

اختر العدد بعد مراعاة:

- CPU المتاحة.
- RAM.
- استهلاك PostgreSQL والخدمات الأخرى.
- طبيعة Requests.
- Cron Workers.
- الأعمال الطويلة.
- عدد اتصالات Database.

ثم اختبر الحمل على Staging باستخدام طريقة القياس في [[28 Performance]].

زيادة Workers قد تزيد استهلاك الذاكرة والاتصالات دون تحسين النتيجة إذا كان الجزء المقيد هو Database أوخدمة خارجية.

حدد كذلك حدود الذاكرة ووقت التنفيذ وفق العمل المقاس، بدل رفعها لتغطية عملية غير مناسبة داخل Request طويلة.

---

### Database واختيارها

في بيئة تحتوي Database واحدة، يمكن تثبيت اختيارها باستخدام اسم وFilter واضحين.

راجع:

```text
db_name
dbfilter
list_db
```

وبعد التأكد من اختيار Database بصورة صحيحة، قيد الوصول إلى مسارات إدارة Databases.

استخدم حساب PostgreSQL مناسبًا لتشغيل Odoo، دون منحه Superuser.

هذه إعدادات تشغيل واختيار Database؛ صلاحيات Records داخل التطبيق تظل مسؤولية Security التي بنيناها في [[11 Security & Multi-Company]].

---

## Release Preparation

### تثبيت هوية الإصدار

احتفظ بمعلومات تسمح بإعادة بناء الإصدار نفسه.

| العنصر | ما الذي نسجله؟ |
| --- | --- |
| Odoo Community | Commit محددة |
| Enterprise عند استخدامها | Commit متوافقة ومحددة |
| Custom Modules | Commit أوTag للإصدار |
| Python | الإصدار المستخدم |
| Dependencies | الإصدارات المعتمدة |
| System Dependencies | التبعيات اللازمة للتشغيل والتقارير |
| Module Updates | Modules المطلوب تحديثها |
| Configuration | نسخة الإعدادات المرتبطة بالإصدار |

اسم الفرع `20.0` وحده لا يحدد Commit ثابتة.

جهز الإصدار واختبره قبل نافذة النشر، حتى لا تصبح مرحلة تشغيل Production هي أول تجربة لتثبيت Dependencies.

---

### تشغيل Odoo كخدمة

استخدم Service Manager مناسبة، مثل systemd على Linux، لتحديد:

- حساب تشغيل مخصص.
- مسار Python Environment.
- مسار Odoo.
- Config المستخدمة.
- طريقة بدء الخدمة وإيقافها.
- جمع Logs.
- سياسة إعادة التشغيل عند الفشل.

اجعل حساب التشغيل قادرًا على قراءة الكود وكتابة البيانات التي يحتاجها، مثل `data_dir`.

احتفظ بمسار البيانات الدائمة خارج مجلد الإصدار، حتى لا يتغير عند تبديل الكود.

---

### Secrets

احفظ الأسرار خارج Git، ومنها:

```text
Database Password
Database Manager Master Password
API Keys
SMTP Credentials
Private Tokens
```

استخدم ملفات إعداد محمية أوآلية Secrets تناسب بيئة التشغيل.

فرق بين:

| السر | الغرض |
| --- | --- |
| `db_password` | اتصال Odoo بـPostgreSQL |
| `admin_passwd` | عمليات إدارة Databases |
| User Password | دخول مستخدم إلى تطبيق Odoo |
| API Key | مصادقة تكامل باسم حساب محدد |

لا تستخدم قيمة واحدة لهذه الأغراض.

---

### النسخ الاحتياطي وخطة الرجوع

قبل تغيير Production، جهز نسخة قابلة للاستعادة تشمل:

- Database.
- Filestore المتوافقة معها.
- الكود وDependencies المرتبطة.
- Config والمعلومات التشغيلية المطلوبة، مع حماية الأسرار.

حدد أيضًا:

- من يتخذ قرار الرجوع؟
- ما آخر وقت مقبول لاتخاذه؟
- هل تغير الإصدار Schema أوالبيانات؟
- هل يمكن تشغيل الكود السابق على الحالة الجديدة؟
- ماذا يحدث للبيانات المكتوبة بعد إعادة فتح الخدمة؟

سنفصل النسخ والاستعادة في [[30 Backup & Restore]]، وتحويل البيانات في [[31 Upgrades & Data Migration]].

---

## Practical Deployment Flow

### 1. اعتماد الإصدار على Staging

طبق على Staging الإصدار وخطوات التحديث التي ستستخدمها في Production.

تحقق من:

- تثبيت Dependencies.
- نجاح تحديث Modules المطلوبة.
- الاختبارات المناسبة من [[26 Testing]].
- تشغيل التقارير والمرفقات والواجهات.
- الصلاحيات والشركات.
- التكاملات باستخدام إعدادات اختبار.

سجل مدة العملية ونتيجتها وأي خطوات إضافية احتاجت إليها.

---

### 2. تنظيم ملفات Self-hosted

مثال لتنظيم الملفات:

```text
/srv/odoo/
├── releases/
│   └── release-001/
│       ├── .venv/
│       ├── odoo/
│       └── custom-addons/
└── current -> releases/release-001

/etc/odoo/production.conf

/var/lib/odoo/
└── filestore/
    └── equipment_production/
```

في هذا المثال:

| المسار | الوظيفة |
| --- | --- |
| `releases` | إصدارات الكود وبيئاتها |
| `current` | الإشارة إلى الإصدار المختار |
| `production.conf` | إعداد الخدمة |
| `/var/lib/odoo` | البيانات الدائمة |

تبديل `current` يغير الكود المختار، لكنه لا يعيد Database إلى حالة سابقة.

---

### 3. إعداد الاتصال والتخزين

هذا مقطع إعداد للشبكة والتخزين واختيار Database في نموذج Server واحدة:

```ini
[options]

addons_path = /srv/odoo/current/odoo/addons,/srv/odoo/current/custom-addons

data_dir = /var/lib/odoo

db_host = 127.0.0.1
db_port = 5432
db_user = odoo_runtime

db_name = equipment_production
dbfilter = ^equipment_production$
list_db = False

http_interface = 127.0.0.1
http_port = 8069
gevent_port = 8072

proxy_mode = True
```

أكمل Config بإعدادات المصادقة المحمية، وWorkers وحدود الموارد المعتمدة، وطريقة Logging.

لم نضع رقم Workers داخل هذا المقطع؛ يحدد بعد اختبار السعة.

ويفترض المثال أن PostgreSQL وProxy تعملان محليًا. عدل الاتصال عند توزيع الخدمات على أكثر من Server.

---

### 4. بدء نافذة الصيانة

عندما يتضمن النشر تحديث Modules أوSchema:

1. فعّل وضع صيانة مناسبًا على مدخل الخدمة.
2. أوقف استقبال عمليات كتابة جديدة.
3. أوقف Odoo Workers والأعمال الخلفية ذات العلاقة.
4. تعامل مع Requests الجارية وفق خطة الإيقاف.
5. تأكد من عدم وجود Process أخرى تعدل Database أثناء التحديث.

ضع خطة للتكاملات الواردة؛ قد تحتاج إلى الانتظار أوإعادة المحاولة بدل الاستمرار في إرسال عمليات أثناء الصيانة.

---

### 5. أخذ نسخة متسقة

بعد إيقاف التغييرات، خذ النسخة المعتمدة للرجوع.

اربطها بمعلومات الإصدار السابق، وتحقق من اكتمال Database وFilestore.

وجود ملف Backup لا يثبت وحده أنه صالح؛ يجب أن تكون طريقة الاستعادة قد جُربت مسبقًا.

---

### 6. اختيار الإصدار الجديد

ضع الكود وPython Environment المعتمدين في مسار الإصدار الجديد، ثم اجعل الخدمة تشير إليهما.

تحقق من:

- مسارات Addons.
- Config الصحيحة.
- Database المستهدفة.
- صلاحية الوصول إلى `data_dir`.
- توافق Odoo وCustom Modules.

لا تشغّل Workers قديمة بالتوازي مع إصدار يغير Schema أوتعريفات Models.

---

### 7. تطبيق Module Upgrade عند الحاجة

حدد Modules المطلوب تحديثها بناءً على التغيير.

في Odoo.sh، تستخدم آلية النشر والبناء الخاصة بالمنصة؛ وتوضح الوثائق كيف يؤدي رفع Version الخاصة بالـModule إلى تطبيق تحديثها ضمن النشر.

أما في Self-hosted، فيمكن تنفيذ تحديث مضبوط خلال الصيانة.

مثال Linux، بهوية حساب تشغيل الخدمة:

```bash
/srv/odoo/current/.venv/bin/python \
  /srv/odoo/current/odoo/odoo-bin \
  -c /etc/odoo/production.conf \
  -d equipment_production \
  -u equipment_training \
  --workers 0 \
  --max-cron-threads 0 \
  --no-http \
  --stop-after-init
```

هذا أمر تحديث مؤقت:

- يستخدم الإصدار الجديد.
- يستهدف Database محددة.
- يحدث Module المحددة.
- يعطل استقبال HTTP وCron Workers في هذه العملية.
- يتوقف بعد انتهاء التهيئة.

راجع Logs ونتيجة الأمر قبل تشغيل الخدمة المعتادة.

إذا فشل التحديث، أبقِ الخدمة في حالة الصيانة وافحص السبب وفق [[27 Debugging]].

---

### 8. تشغيل الخدمة والتحقق

شغّل الخدمة بإعدادات التشغيل المعتادة، ثم نفذ Smoke Tests.

بعد اجتياز الفحص:

1. أعد فتح مدخل الخدمة.
2. أعد تفعيل الأعمال والتكاملات وفق الخطة.
3. راقب Logs والموارد والطلبات.
4. سجل الإصدار المنشور ووقت النشر والنتائج.

لا تكتفِ برسالة أن Process بدأت بنجاح.

---

## Verification & Rollback

### Smoke Tests لمشروع الأجهزة

Smoke Tests هي فحوص قصيرة تثبت أن المسارات الأساسية تعمل بعد النشر.

| المسار | ما الذي نتحقق منه؟ |
| --- | --- |
| تسجيل الدخول | الدخول بالحساب المناسب عبر Domain وHTTPS |
| الأجهزة | فتح List وForm وقراءة البيانات |
| Business Actions | نجاح عملية مسموحة وبقاء العملية الممنوعة مرفوضة |
| الإعارات | فتح البيانات وتطبيق Validation |
| Wizard | فتحها وتنفيذ العملية المطلوبة |
| التقارير | توليد PDF والتحقق من المحتوى |
| المرفقات | فتح ملف معروف من Filestore |
| Dashboard | تحميل العدد وتحديثه |
| Portal | الوصول إلى Records المسموحة فقط |
| API | نجاح طلب مناسب بحساب التكامل |
| الاتصالات الفورية | عمل WebSocket عند اعتماد الوظائف عليها |

اختر بيانات فحص معدة مسبقًا، واحسب أي أثر ستنشئه عمليات الكتابة.

---

### المراقبة بعد النشر

راقب:

- Errors الجديدة.
- زمن Requests.
- CPU وRAM.
- اتصالات PostgreSQL.
- مساحة القرص.
- فشل Cron أوتراكم العمل.
- Outgoing Mail والتكاملات.
- WebSocket.
- صلاحية شهادة HTTPS.
- نجاح النسخ الاحتياطية التالية.

اربط أي تغير بالإصدار ووقت النشر، ثم قارنه بخط أساس سابق.

سنوسع هذا الجانب في [[33 Maintenance]].

---

### الرجوع قبل وبعد تغيير Database

| الحالة | ما الذي يحتاج إلى التحقق؟ |
| --- | --- |
| الكود تغير دون تغيير Schema أوبيانات | هل الإصدار السابق متوافق مع الحالة الحالية؟ |
| Upgrade غيرت Schema أوالبيانات | هل نحتاج إلى استعادة Database وFilestore مع الإصدار السابق؟ |
| أعيد فتح الخدمة واستقبلت كتابات جديدة | كيف نحافظ على البيانات الجديدة أونعالج فقدانها عند الاستعادة؟ |
| حدث أثر في نظام خارجي | هل يحتاج إلى تعويض أوتسوية مستقلة؟ |

الرجوع في Git يعيد الكود، ولا يعكس تلقائيًا Migration أوكتابات Database.

واستعادة Backup قد تعيد الحالة إلى وقت سابق؛ لذلك حدد أثرها على العمليات التي حدثت بعدها.

---

### حدود النشر التدريجي

وجود عدة Odoo Servers لا يجعل تحديث Module آمنًا دون تنسيق.

إذا كانت جميعها تستخدم Database واحدة، فيجب أن يتوافق الكود الجاري مع Schema وتعريفات Models الموجودة.

كما تحتاج الملفات والجلسات والأعمال الخلفية إلى تصميم يناسب توزيع الخدمة.

اختر استراتيجية النشر وفق توافق الإصدارات وحدود التغيير، بدل افتراض أن تبديل Workers واحدةً تلو الأخرى يكفي لكل تحديث.

---

## Important Notes

- انشر إصدارًا محددًا يمكن إعادة بنائه.
- استخدم Hosting تدعم Custom Modules الخاصة بالمشروع.
- اختبر الإصدار وخطوات تحديثه على Staging.
- تحقق من Neutralization والتكاملات المخصصة في النسخ الاختبارية.
- Database وFilestore مخزنان مرتبطان يحتاجان إلى اتساق.
- احتفظ بالبيانات الدائمة خارج مجلد الإصدار.
- شغّل Odoo كخدمة بحساب وإعدادات واضحين.
- استخدم HTTPS وProxy موثوقة، واضبط `proxy_mode` وفق مسار الاتصال.
- اختبر WebSocket عند استخدام Multi-process والوظائف المرتبطة بها.
- وضع Multi-process الخاص بـOdoo غير متاح على Windows.
- اختر Workers وحدود الموارد بالقياس.
- اضبط اختيار Database وقيد الوصول إلى أدوات إدارتها.
- احفظ الأسرار خارج Git.
- أوقف التغييرات المتعارضة أثناء تحديث Schema.
- تحقق من نجاح Upgrade قبل تشغيل الخدمة وفتحها للمستخدمين.
- الرجوع إلى الكود السابق لا يعيد Database تلقائيًا.
- استمرار المراقبة والنسخ الاحتياطي جزء من تشغيل الإصدار المنشور.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[30 Backup & Restore]]

---

## Sources

- [Odoo 20 Documentation — System Configuration](https://www.odoo.com/documentation/20.0/administration/on_premise/deploy.html)
- [Odoo 20 Documentation — Source Install](https://www.odoo.com/documentation/20.0/administration/on_premise/source.html)
- [Odoo 20 Documentation — Hosting](https://www.odoo.com/documentation/20.0/administration/hosting.html)
- [Odoo 20 Documentation — Odoo Online](https://www.odoo.com/documentation/20.0/administration/odoo_online.html)
- [Odoo 20 Documentation — Odoo.sh](https://www.odoo.com/documentation/20.0/administration/odoo_sh.html)
- [Odoo 20 Documentation — Odoo.sh Branches](https://www.odoo.com/documentation/20.0/administration/odoo_sh/getting_started/branches.html)
- [Odoo 20 Documentation — Neutralized Database](https://www.odoo.com/documentation/20.0/administration/neutralized_database.html)
- [Odoo 20 Official Source — Configuration Options](https://github.com/odoo/odoo/blob/20.0/odoo/tools/config.py)
- [Odoo 20 Official Source — Server Processes](https://github.com/odoo/odoo/blob/20.0/odoo/service/server.py)