---
type: learning
odoo_version: "20.0"
status: complete
created: 2026-10-13
reviewed: 2026-10-13
---
# 02 Development Environment

## Table of Contents

- [[#Introduction|Introduction]]
- [[#Local Development Setup|Local Development Setup]]
- [[#Understanding the Environment|Understanding the Environment]]
- [[#Daily Development Workflow|Daily Development Workflow]]
- [[#Existing Company Project|Existing Company Project]]
- [[#Important Notes|Important Notes]]
- [[#Sources|Sources]]

---

## Introduction

**Development Environment** هي البيئة التي نستخدمها لتشغيل Odoo وتطوير Modules واختبار التغييرات على بيانات مخصصة للتطوير.

تتكون عادةً من:

```text
Odoo Source Code
Python
Python Dependencies
PostgreSQL
Development Database
Custom Addons
Configuration
Development Tools
```

في هذا الشابتر سنجهز بيئة محلية باستخدام **Odoo 20 Community** على أحد النظامين:

- **Fedora**.
- **Windows**.

سنبدأ بخطوات عملية حتى تعمل Odoo، ثم نشرح المكونات التي استخدمناها والعلاقة بينها.

بعد ذلك سنتعلم كيف نستخدم البيئة يوميًا دون إعادة خطوات Installation.

> [!note] نطاق الشابتر
> هذا الإعداد مخصص للتطوير المحلي.
>
> إعداد Production وطريقة تشغيلها سيأتي في [[29 Deployment]].

---

### متطلبات Odoo 20 الأساسية

المتطلبات الأساسية التي نعتمد عليها:

| المكوّن | المطلوب |
|---|---|
| Odoo | Branch باسم `20.0` |
| Edition | Community |
| Python | الحد الأدنى `3.12` |
| PostgreSQL Server | Version `16` أو أحدث |
| Git | لتحميل Source Code وإدارة التعديلات |
| Code Editor | مثل VS Code |
| Terminal أو PowerShell | لتنفيذ Commands |
| Browser | لفتح واجهة Odoo |

الأوامر التالية تستخدم **Python 3.12** صراحةً عند إنشاء Virtual Environment.

هذا اختيار واضح لمسار الشرح، ولا يعني أن Python 3.12 هي Version الوحيدة التي تدعمها Odoo.

---

## Local Development Setup

سنستخدم **Source Installation**.

أي أننا سنحمّل Source Code الخاصة بـOdoo، ثم نشغّلها باستخدام Python داخل Environment مستقلة للمشروع.

سيكون التنظيم النهائي قريبًا من:

```text
odoo20-dev/
├── .gitignore
├── .venv/
├── odoo/
├── custom-addons/
├── config/
│   ├── odoo.conf
│   └── odoo.conf.example
└── data/
```

وظيفة كل جزء:

| المسار | وظيفته |
|---|---|
| `.venv/` | Python Environment الخاصة بالمشروع |
| `odoo/` | Source Code المحمّلة من Odoo |
| `custom-addons/` | Modules التي سنطورها |
| `config/` | Configuration Files |
| `data/` | Runtime Data مثل Filestore وSessions |
| `.gitignore` | استبعاد الملفات المحلية والمولدة من Git |

> [!note] Community وEnterprise
> هذا المسار يستخدم Community.
>
> تشغيل Enterprise يحتاج أيضًا إلى Enterprise Addons المتوافقة مع النسخة، مع إضافة مسارها قبل بقية المسارات في `addons_path`.

اختر مسار نظامك:

- [[#Fedora|Fedora]]
- [[#Windows|Windows]]

---

### Fedora

سنجهز الأدوات وPostgreSQL، ثم ننشئ المشروع ونشغّل Odoo.

نفّذ كل Command على حدة، وتحقق من نتيجتها قبل متابعة الخطوات.

قد تختلف أسماء بعض Packages بين إصدارات Fedora؛ استخدم الحزم المناسبة لإصدار النظام الذي تعمل عليه.

---

#### 1. التحقق من الأدوات

قبل تثبيت الأدوات، تحقق مما هو موجود بالفعل.

افتح Terminal ونفّذ:

```bash
git --version
```

```bash
python3.12 --version
```

```bash
psql --version
```

ظهور Version يعني أن Command متاحة من Terminal الحالية.

أما ظهور:

```text
command not found
```

فيعني أن الأداة غير مثبتة، أو أن Directory الخاصة بها غير موجودة في `PATH`.

---

#### 2. تثبيت Git

إذا لم تكن Git مثبتة:

```bash
sudo dnf install git
```

تحقق:

```bash
git --version
```

سنستخدمها لتحميل Odoo Source من Repository الرسمية.

---

#### 3. تثبيت Python

ثبت Python الخاصة بهذا المسار وأدواتها:

```bash
sudo dnf install \
  python3.12 \
  python3.12-devel \
  python3-pip
```

تحقق:

```bash
python3.12 --version
```

المتوقع أن تبدأ النتيجة بـ:

```text
Python 3.12
```

إذا لم يوفر إصدار Fedora الذي تستخدمه Package مطلوبة، راجع الحزم المتاحة قبل متابعة الخطوات.

سنستخدم هذه Python لإنشاء `.venv`، ثم نثبت Dependencies داخلها.

---

#### 4. تثبيت PostgreSQL

ثبت PostgreSQL وأدواتها:

```bash
sudo dnf install \
  postgresql \
  postgresql-server \
  postgresql-contrib \
  libpq-devel
```

تحقق من أدوات Client:

```bash
psql --version
```

```bash
pg_config --version
```

تستخدم بعض Python Packages معلومات من `pg_config` أثناء البناء.

إذا لم تكن الأداة متاحة، ابحث عن Package التي توفرها:

```bash
dnf provides '*/pg_config'
```

ثم ثبت Package المناسبة التي يعرضها النظام.

---

#### 5. تثبيت Build Tools وSystem Libraries

بعض Python Dependencies قد تحتاج Compilation أثناء Installation.

ثبت Build Tools:

```bash
sudo dnf install \
  gcc \
  gcc-c++ \
  make
```

ثم Development Libraries:

```bash
sudo dnf install \
  libxml2-devel \
  libxslt-devel \
  openldap-devel \
  cyrus-sasl-devel \
  libjpeg-turbo-devel \
  zlib-devel \
  libffi-devel \
  openssl-devel
```

الحزم التي تنتهي بـ`-devel` توفر عادةً ملفات يحتاجها البناء، مثل Headers وDevelopment Libraries.

---

#### 6. تهيئة PostgreSQL وتشغيلها

إذا كانت PostgreSQL Installation جديدة ولم تُهيأ بعد:

```bash
sudo postgresql-setup --initdb --unit postgresql
```

ثم شغّل Service واجعلها تبدأ مع النظام:

```bash
sudo systemctl enable --now postgresql
```

تحقق من حالتها:

```bash
systemctl status postgresql
```

إذا ظهرت الحالة داخل شاشة عرض ولم تعد إلى Prompt، اضغط `q` للخروج.

ثم اختبر جاهزية Server:

```bash
pg_isready
```

من النتائج المتوقعة:

```text
accepting connections
```

هذه النتيجة تشير إلى أن Server تستقبل الاتصالات.

سنختبر تسجيل الدخول باستخدام Role المناسبة في الخطوات التالية.

> [!warning] Installation موجودة مسبقًا
> إذا كانت PostgreSQL مهيأة وتحتوي Databases، لا تعِد تهيئتها.
>
> افحص Service وإعدادات الاتصال الموجودة، ثم استخدم البيئة المناسبة للمشروع.

---

#### 7. إنشاء PostgreSQL Role

سنستخدم Role تحمل نفس اسم Linux User الحالي.

اعرف اسم المستخدم:

```bash
whoami
```

ثم أنشئ Role:

```bash
sudo -u postgres createuser -d -R -S "$USER"
```

اكتب `$USER` كما هي؛ Shell تستبدلها باسم المستخدم الحالي.

معنى الخيارات:

| الخيار | المعنى |
|---|---|
| `-d` | السماح بإنشاء Databases |
| `-R` | عدم السماح بإنشاء Roles أخرى |
| `-S` | عدم منح Superuser Privileges |

عندما تكون Local Authentication مهيأة لذلك، يسمح تطابق اسم Role مع Linux User بالاتصال المحلي دون Password.

إذا ظهرت رسالة أن Role موجودة بالفعل، افحصها بدل حذفها وإعادة إنشائها.

> [!note] حساب PostgreSQL
> هذه Role تستخدمها Odoo للاتصال بـPostgreSQL.
>
> ليست الحساب الذي سنستخدمه في صفحة Login الخاصة بـOdoo.

---

#### 8. اختبار PostgreSQL

افتح PostgreSQL Client باستخدام اتصال محلي إلى Database الإدارية:

```bash
psql -d postgres
```

داخل `psql`، تحقق من Role الحالية:

```sql
SELECT current_user;
```

ثم افحص إصدار Server:

```sql
SHOW server_version;
```

تحقق من أن Server تستخدم Version `16` أو أحدث.

لعرض Databases:

```text
\l
```

للخروج:

```text
\q
```

نستخدم Database الموجودة باسم `postgres` لاختبار الاتصال؛ لا نحتاج إلى إنشاء Database إضافية لهذا الفحص.

> [!warning] عند فشل Authentication
> راجع Role وطريقة الاتصال وإعدادات PostgreSQL المحلية.
>
> لا تغيّر Authentication إلى `trust` على نطاق واسع لمجرد تجاوز Error.

---

#### 9. إنشاء Project Directory

اختر مكانًا مناسبًا للمشروع.

مثلًا، انتقل إلى Documents:

```bash
cd ~/Documents
```

أنشئ Project:

```bash
mkdir odoo20-dev
```

```bash
cd odoo20-dev
```

ثم أنشئ Directories:

```bash
mkdir custom-addons config data
```

أصبح لدينا:

```text
odoo20-dev/
├── custom-addons/
├── config/
└── data/
```

من هذه النقطة، **Project Root** هي `odoo20-dev/`.

نفّذ بقية Commands هذا المسار من داخلها.

---

#### 10. تحميل Odoo Source

حمّل Community Branch الخاصة بـOdoo 20:

```bash
git clone \
  --branch 20.0 \
  --single-branch \
  https://github.com/odoo/odoo.git \
  odoo
```

بعد اكتمال Clone:

```text
odoo20-dev/
├── odoo/
├── custom-addons/
├── config/
└── data/
```

تحقق من Branch:

```bash
git -C odoo branch --show-current
```

المتوقع:

```text
20.0
```

Directory `odoo/` تحتوي Source Code التي سنشغّلها.

---

#### 11. إنشاء Virtual Environment

من Project Root:

```bash
python3.12 -m venv .venv
```

تحقق من Python الموجودة داخلها:

```bash
.venv/bin/python --version
```

أصبح تنظيم المشروع:

```text
odoo20-dev/
├── .venv/
├── odoo/
├── custom-addons/
├── config/
└── data/
```

يمكن تفعيل Environment:

```bash
source .venv/bin/activate
```

سنستخدم Python الموجودة داخل `.venv` مباشرةً عند تنفيذ الأوامر التالية، لذلك لا تحتاج إلى تفعيل Virtual Environment.

> [!note] إظهار `.venv`
> Directories التي تبدأ بنقطة تكون مخفية عادةً في Linux.
>
> لإظهارها في File Manager استخدم `Ctrl + H`، أو في Terminal:
>
> ```bash
> ls -a
> ```

---

#### 12. تثبيت Python Dependencies

جهز `pip` وأدوات بناء Packages داخل `.venv`:

```bash
.venv/bin/python \
  -m pip install --upgrade \
  pip setuptools wheel
```

ثم ثبت متطلبات Odoo:

```bash
.venv/bin/python \
  -m pip install \
  -r odoo/requirements.txt
```

بعد اكتمال Installation:

```bash
.venv/bin/python -m pip check
```

النتيجة المتوقعة:

```text
No broken requirements found.
```

إذا فشلت Package، اقرأ اسمها والسبب الفعلي، مثل:

- Compiler غير موجودة.
- Header ناقصة.
- System Library غير متوفرة.
- Python Version غير مناسبة.
- Download لم تكتمل.

لا تغير Versions داخل `requirements.txt` لتجاوز Error دون فهم السبب.

---

#### 13. التحقق من Odoo Version

نفّذ:

```bash
.venv/bin/python \
  odoo/odoo-bin \
  --version
```

المتوقع ظهور Version الخاصة بـOdoo 20.

هذا الفحص لا يختبر Database Connection؛ سنتحقق منها عند التهيئة والتشغيل.

---

#### 14. إنشاء Configuration File

افتح Project في Code Editor.

إذا كانت Command الخاصة بـVS Code متاحة:

```bash
code .
```

ويمكن فتح المجلد من الواجهة:

```text
File → Open Folder
```

داخل `config/`، أنشئ الملف:

```text
odoo.conf
```

وأضف:

```ini
[options]

admin_passwd = REPLACE_WITH_MASTER_PASSWORD

addons_path = /ABSOLUTE/PATH/odoo20-dev/odoo/addons,/ABSOLUTE/PATH/odoo20-dev/custom-addons

data_dir = /ABSOLUTE/PATH/odoo20-dev/data

http_interface = 127.0.0.1
http_port = 8069
```

هذا المحتوى يحتاج إلى تعديل قيمتين قبل استخدامه:

- **Master Password** لإدارة Databases.
- **Absolute Paths** الخاصة بالمشروع على جهازك.

استبدل `REPLACE_WITH_MASTER_PASSWORD` بقيمة تختارها.

هذه ليست Password تسجيل الدخول إلى واجهة Odoo.

للحصول على Absolute Paths، نفّذ من Project Root:

```bash
realpath odoo/addons
realpath custom-addons
realpath data
```

قد تظهر مسارات مثل:

```text
/home/developer/Documents/odoo20-dev/odoo/addons
/home/developer/Documents/odoo20-dev/custom-addons
/home/developer/Documents/odoo20-dev/data
```

استخدم النتائج الفعلية داخل Configuration:

- ضع مساري Addons في `addons_path`، وافصل بينهما بفاصلة.
- ضع مسار Runtime Directory في `data_dir`.

مثلًا، بعد تعديل المسارات:

```ini
[options]

admin_passwd = REPLACE_WITH_MASTER_PASSWORD

addons_path = /home/developer/Documents/odoo20-dev/odoo/addons,/home/developer/Documents/odoo20-dev/custom-addons

data_dir = /home/developer/Documents/odoo20-dev/data

http_interface = 127.0.0.1
http_port = 8069
```

المسارات السابقة توضيحية؛ استخدم مسارات جهازك، وتأكد من استبدال Master Password ثم احفظ الملف.

يعتمد هذا الإعداد على Local PostgreSQL Connection التي اختبرناها سابقًا.

---

#### 15. حماية Configuration File

الملف الحقيقي قد يحتوي Passwords وإعدادات خاصة بالبيئة.

قيد Permissions:

```bash
chmod 600 config/odoo.conf
```

تحقق:

```bash
ls -l config/odoo.conf
```

المتوقع Permissions مثل:

```text
-rw-------
```

يجب أن يستطيع حسابك قراءة Configuration والكتابة داخل `data/`.

---

#### 16. إنشاء Configuration Example

داخل `config/`، أنشئ:

```text
odoo.conf.example
```

واستخدم قيمًا وهمية:

```ini
[options]

admin_passwd = REPLACE_ME

addons_path = /PATH/TO/odoo20-dev/odoo/addons,/PATH/TO/odoo20-dev/custom-addons

data_dir = /PATH/TO/odoo20-dev/data

http_interface = 127.0.0.1
http_port = 8069
```

الفرق:

| الملف | استخدامه |
|---|---|
| `odoo.conf` | Configuration الفعلية |
| `odoo.conf.example` | Template قابلة للمشاركة بعد مراجعتها |

لا تضع Passwords الحقيقية في Example.

---

#### 17. إعداد `.gitignore`

داخل Project Root، أنشئ:

```text
.gitignore
```

وأضف:

```gitignore
/.venv/
/odoo/
/data/
/backups/

/config/*.conf

__pycache__/
*.py[cod]
*.log
```

بهذا نستبعد من Project Repository:

- Virtual Environment.
- Clone المستقلة الخاصة بـOdoo.
- Runtime Data.
- Backups.
- Configuration الحقيقية.
- Python Cache وLogs.

ملف `odoo.conf.example` لا ينتهي بـ`.conf`، لذلك لا تستبعده القاعدة السابقة.

> [!important] استبعاد Runtime Data فقط
> نستخدم `/data/` لتحديد Directory الموجودة في جذر المشروع.
>
> كتابة `data/` دون هذا التقييد قد تستبعد أيضًا مجلدات Data داخل Custom Modules، مع أننا نحتاج إلى تتبع ملفاتها.

تحميل Odoo أنشأ Repository داخل `odoo/` فقط.

إذا أردت تتبع Custom Project في Repository مستقلة، أنشئها من Project Root.

---

#### 18. إنشاء Development Database

أنشئ Database التدريب:

```bash
createdb odoo20_training
```

تحقق من وجودها:

```bash
psql -l
```

أصبحت لدينا PostgreSQL Database، لكنها لم تُهيأ بعد كـOdoo Database.

إذا كانت موجودة من محاولة سابقة، افحصها قبل اتخاذ أي إجراء عليها.

---

#### 19. تهيئة Database

أنشأنا PostgreSQL Database في الخطوة السابقة، لكنها لم تُهيأ بعد لتشغيل Odoo.

الآن سنستخدم Odoo لتثبيت Module الأساسية `base` وإنشاء Tables والبيانات المطلوبة.

نفّذ من Project Root:

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_training \
  -i base \
  --stop-after-init
```

معنى الخيارات:

| الخيار | وظيفته |
|---|---|
| `-c config/odoo.conf` | استخدام Configuration الخاصة بالمشروع |
| `-d odoo20_training` | تحديد Database التي سنهيئها |
| `-i base` | تثبيت Module الأساسية `base` |
| `--stop-after-init` | إيقاف Odoo بعد اكتمال التهيئة |

اقرأ Logs وتحقق من اكتمال العملية دون Errors تمنع التهيئة.

بعد انتهاء Process والعودة إلى Prompt، ننتقل إلى تشغيل Odoo بالأمر العادي في الخطوة التالية.

> [!note] تهيئة أولية
> نستخدم `-i base` عند تهيئة Database لأول مرة.
>
> لا نضيفها إلى أمر التشغيل اليومي.

---

#### 20. تشغيل Odoo لأول مرة

شغّل Server بالأمر العادي:

```bash
.venv/bin/python \
  odoo/odoo-bin \
  -c config/odoo.conf \
  -d odoo20_training
```

بعد اكتمال التحميل وبدء HTTP Server، افتح المتصفح وانتقل إلى:

```text
http://127.0.0.1:8069
```

اترك Terminal مفتوحة أثناء تشغيل Server.

للإيقاف:

```text
Ctrl + C
```

أكمل الآن من [[#After Starting Odoo|After Starting Odoo]].

---

### Windows

سنجهز الأدوات وPostgreSQL، ثم ننشئ Project ونشغّل Odoo من PowerShell.

قد تختلف شاشات Installers عن الصور؛ اعتمد على اسم الخيار ووظيفته.

---

#### 1. التحقق من الأدوات

افتح PowerShell ونفّذ:

```powershell
git --version
```

```powershell
py -3.12 --version
```

```powershell
psql --version
```

إذا لم تتعرف PowerShell على Command، فقد تكون الأداة:

- غير مثبتة.
- غير موجودة في `PATH`.
- ثبتت بعد فتح النافذة الحالية.

بعد تثبيت أداة أو تغيير `PATH`، افتح PowerShell جديدة قبل التحقق.

---

#### 2. تثبيت Git

إذا لم تكن Git مثبتة، افتح:

```text
https://git-scm.com/
```

اختر Download الخاصة بـWindows.

أثناء Installation، تأكد من إتاحة Git من Command Line.

بعد الانتهاء افتح PowerShell جديدة:

```powershell
git --version
```

المتوقع ظهور Version الخاصة بـGit.

---

#### 3. تثبيت Python

سنستخدم في أوامر هذا المسار:

```text
Python 3.12
Windows 64-bit
```

افتح:

```text
https://www.python.org/downloads/windows/
```

اختر Installer مناسبة لـPython 3.12 ضمن مسار الإعداد المستخدم.

تأكد أثناء Installation من وجود:

- `pip`.
- Python Launcher عند توفره.
- خيار إضافة Python إلى `PATH`.

![[Python install.png]]

بعد الانتهاء افتح PowerShell جديدة:

```powershell
py -3.12 --version
```

ثم تحقق من `pip` المرتبطة بها:

```powershell
py -3.12 -m pip --version
```

استخدام `py -3.12` يحدد Python المطلوبة إذا كان الجهاز يحتوي أكثر من Version.

لا يشترط الشرح رقم Patch محددًا؛ المهم أن تعرف Interpreter المستخدمة وتتحقق من توافقها مع المشروع.

إذا لم تكن Command `py` متاحة، تحقق من Launcher أو استخدم المسار الفعلي لـPython المطلوبة.

---

#### 4. تثبيت PostgreSQL

افتح:

```text
https://www.postgresql.org/download/windows/
```

ثبت PostgreSQL Server بإصدار `16` أو أحدث.

تأكد من وجود:

- PostgreSQL Server.
- Command Line Tools.
- pgAdmin، لاستخدام خطوات إنشاء Role من الواجهة.

![[PostgreSQL setup.png]]

سيطلب Installer Password للحساب الإداري:

```text
postgres
```

احتفظ بها لإدارة PostgreSQL.

يمكن استخدام Port الافتراضية:

```text
5432
```

إذا كانت Port مستخدمة، تحقق من الخدمة الموجودة قبل اختيار قيمة أخرى.

بعد Installation افتح PowerShell جديدة:

```powershell
psql --version
```

**إذا لم تتعرف PowerShell على `psql`:**

تحقق من Directory الخاصة بأدوات PostgreSQL.

تكون عادةً:

```text
C:\Program Files\PostgreSQL\<version>\bin
```

مثال لمسار Installation تستخدم PostgreSQL 18:

```text
C:\Program Files\PostgreSQL\18\bin
```

أضف المسار الفعلي إلى Windows `PATH`.

![[Edit environment variables.gif]]

ثم أغلق PowerShell وافتح نافذة جديدة:

```powershell
psql --version
```

اختبر جاهزية Server:

```powershell
pg_isready -h 127.0.0.1 -p 5432
```

إذا كانت Port مختلفة، استخدم قيمتها الفعلية.

---

#### 5. إنشاء PostgreSQL Role خاصة بـOdoo

سننشئ Role مستقلة باسم:

```text
odoo
```

افتح pgAdmin، واتصل بـServer باستخدام الحساب الإداري.

انتقل إلى:

```text
Login/Group Roles
→ Create
→ Login/Group Role
```

![[pgAdmin.gif]]

داخل **General**، ضع الاسم:

```text
odoo
```

داخل **Definition**، حدد Password خاصة بهذه Role واحتفظ بها.

سنستخدمها لاحقًا في:

```ini
db_password = ...
```

انتقل إلى **Privileges** واضبط الخيارات كما في الصورة:

![[Privileges PostgreSQL.png]]

| الخيار | الحالة |
|---|---|
| Can login? | مفعّل |
| Superuser? | غير مفعّل |
| Create roles? | غير مفعّل |
| Create databases? | مفعّل |
| Inherit rights from the parent roles? | مفعّل |
| Can initiate streaming replication and backups? | غير مفعّل |
| Bypass RLS? | غير مفعّل |

أهم خيارين لمسارنا:

- **Can login?** يسمح للـRole بالاتصال بـPostgreSQL.
- **Create databases?** يسمح بإنشاء Databases التدريبية.

أبقِ **Inherit rights from the parent roles?** مفعّلًا كما في الصورة، واترك بقية الخيارات غير مفعلة.

ثم احفظ Role.

> [!important] اسم Role وPassword
> `odoo` هو اسم Role، وليس Password مفروضة.
>
> ولا تستخدم الحساب الإداري `postgres` لتشغيل Odoo بشكل اعتيادي.

---

#### 6. اختبار PostgreSQL Role

افتح اتصالًا باستخدام Role الجديدة:

```powershell
psql `
  -h 127.0.0.1 `
  -p 5432 `
  -U odoo `
  -W `
  -d postgres
```

أدخل Password الخاصة بـRole `odoo`.

داخل `psql`:

```sql
SELECT current_user;
```

المتوقع:

```text
odoo
```

ثم افحص إصدار Server:

```sql
SHOW server_version;
```

تحقق من أن Server تستخدم Version `16` أو أحدث.

للخروج:

```text
\q
```

> [!note] ما الذي تحققنا منه؟
> اختبرنا تسجيل الدخول الفعلي وإصدار Server.
>
> الأمر `psql --version` وحده يعرض إصدار أداة Client فقط.

---

#### 7. تثبيت Visual Studio Build Tools

بعض Python Packages قد تحتاج Compilation على Windows.

افتح:

```text
https://visualstudio.microsoft.com/downloads/
```

وابحث عن:

```text
Build Tools for Visual Studio
```

داخل Installer اختر Workload:

```text
Desktop development with C++
```

![[Desktop development with C++.png]]

تأكد من وجود:

- MSVC Build Tools.
- Windows SDK.
- المكونات الأساسية التابعة لـWorkload.

ثم أكمل Installation.

إذا ظهرت مشكلة في اكتشاف Compiler، تحقق من بيئة البناء باستخدام Terminal المناسبة التي توفرها Build Tools.

لا تضف مسارات Compiler عشوائية إلى `PATH`.

---

#### 8. إنشاء Project Directory

انتقل إلى المكان الذي تريد وضع المشروع فيه.

مثلًا:

```powershell
cd "$HOME\Documents"
```

أنشئ Project:

```powershell
mkdir odoo20-dev
```

```powershell
cd odoo20-dev
```

ثم Directories:

```powershell
mkdir custom-addons
```

```powershell
mkdir config
```

```powershell
mkdir data
```

أصبح لدينا:

```text
odoo20-dev/
├── custom-addons/
├── config/
└── data/
```

من هذه النقطة، **Project Root** هي `odoo20-dev/`.

نفّذ بقية Commands هذا المسار من داخلها.

---

#### 9. تحميل Odoo Source

حمّل Community Branch الخاصة بـOdoo 20:

```powershell
git clone `
  --branch 20.0 `
  --single-branch `
  https://github.com/odoo/odoo.git `
  odoo
```

بعد اكتمال Clone:

```text
odoo20-dev/
├── odoo/
├── custom-addons/
├── config/
└── data/
```

تحقق من Branch:

```powershell
git -C odoo branch --show-current
```

المتوقع:

```text
20.0
```

---

#### 10. إنشاء Virtual Environment

من Project Root:

```powershell
py -3.12 -m venv .venv
```

تحقق من Python الموجودة داخلها:

```powershell
.\.venv\Scripts\python.exe --version
```

أصبح تنظيم المشروع:

```text
odoo20-dev/
├── .venv/
├── odoo/
├── custom-addons/
├── config/
└── data/
```

سنستخدم Python الموجودة داخل `.venv` مباشرةً عند تنفيذ الأوامر التالية، لذلك لا تحتاج إلى تفعيل Virtual Environment.

---

#### 11. تثبيت Python Dependencies

جهز `pip` وأدوات بناء Packages:

```powershell
.\.venv\Scripts\python.exe `
  -m pip install --upgrade `
  pip setuptools wheel
```

ثم ثبت متطلبات Odoo:

```powershell
.\.venv\Scripts\python.exe `
  -m pip install `
  -r odoo\requirements.txt
```

بعد اكتمال Installation:

```powershell
.\.venv\Scripts\python.exe -m pip check
```

النتيجة المتوقعة:

```text
No broken requirements found.
```

إذا فشلت Package، اقرأ اسمها والسبب الفعلي قبل تثبيت Packages إضافية أو تغيير Versions.

---

#### 12. التحقق من Odoo Version

نفّذ:

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  --version
```

المتوقع ظهور Version الخاصة بـOdoo 20.

هذا الفحص لا يختبر Database Connection؛ سنتحقق منها عند التهيئة والتشغيل.

---

#### 13. إنشاء Configuration File

افتح Project في Code Editor.

إذا كانت Command الخاصة بـVS Code متاحة:

```powershell
code .
```

ويمكن فتح المجلد من:

```text
File → Open Folder
```

داخل `config/`، أنشئ:

```text
odoo.conf
```

وأضف:

```ini
[options]

admin_passwd = REPLACE_WITH_MASTER_PASSWORD

db_host = 127.0.0.1
db_port = 5432
db_user = odoo
db_password = REPLACE_WITH_POSTGRESQL_ROLE_PASSWORD

addons_path = C:/ABSOLUTE/PATH/odoo20-dev/odoo/addons,C:/ABSOLUTE/PATH/odoo20-dev/custom-addons

data_dir = C:/ABSOLUTE/PATH/odoo20-dev/data

http_interface = 127.0.0.1
http_port = 8069
```

للحصول على Absolute Paths، نفّذ من Project Root:

```powershell
Resolve-Path .\odoo\addons
```

```powershell
Resolve-Path .\custom-addons
```

```powershell
Resolve-Path .\data
```

استبدل:

- `REPLACE_WITH_MASTER_PASSWORD` بقيمة تختارها لإدارة Databases.
- `REPLACE_WITH_POSTGRESQL_ROLE_PASSWORD` بالـPassword التي حددتها للـRole `odoo`.
- المسارات بالنتائج الفعلية من Commands السابقة.
- Port إذا كانت PostgreSQL تستخدم قيمة مختلفة عن `5432`.

بعد استبدال القيم، احفظ الملف.

---

#### 14. إنشاء Configuration Example

داخل `config/`، أنشئ:

```text
odoo.conf.example
```

واستخدم قيمًا وهمية:

```ini
[options]

admin_passwd = REPLACE_ME

db_host = 127.0.0.1
db_port = 5432
db_user = odoo
db_password = REPLACE_ME

addons_path = C:/PATH/TO/odoo20-dev/odoo/addons,C:/PATH/TO/odoo20-dev/custom-addons

data_dir = C:/PATH/TO/odoo20-dev/data

http_interface = 127.0.0.1
http_port = 8069
```

الفرق:

| الملف | استخدامه |
|---|---|
| `odoo.conf` | Configuration الفعلية |
| `odoo.conf.example` | Template قابلة للمشاركة بعد مراجعتها |

لا تضع Passwords الحقيقية داخل Example.

---

#### 15. إعداد `.gitignore`

داخل Project Root، أنشئ:

```text
.gitignore
```

وأضف:

```gitignore
/.venv/
/odoo/
/data/
/backups/

/config/*.conf

__pycache__/
*.py[cod]
*.log
```

بهذا نستبعد من Project Repository:

- Virtual Environment.
- Clone المستقلة الخاصة بـOdoo.
- Runtime Data.
- Backups.
- Configuration الحقيقية.
- Python Cache وLogs.

ملف `odoo.conf.example` لا ينتهي بـ`.conf`، لذلك لا تستبعده القاعدة السابقة.

> [!important] استبعاد Runtime Data فقط
> نستخدم `/data/` لتحديد Directory الموجودة في جذر المشروع.
>
> كتابة `data/` دون هذا التقييد قد تستبعد أيضًا مجلدات Data داخل Custom Modules، مع أننا نحتاج إلى تتبع ملفاتها.

تحميل Odoo أنشأ Repository داخل `odoo/` فقط.

إذا أردت تتبع Custom Project في Repository مستقلة، أنشئها من Project Root.

---

#### 16. إنشاء Development Database

سننشئ Database التدريب باستخدام PostgreSQL Role التي جهزناها.

نفّذ من PowerShell:

```powershell
createdb `
  -h localhost `
  -U odoo `
  -W `
  odoo20_training
```

معنى أجزاء الأمر:

| الجزء | وظيفته |
|---|---|
| `createdb` | أداة PostgreSQL لإنشاء Database |
| `-h localhost` | الاتصال بـPostgreSQL على الجهاز نفسه |
| `-U odoo` | استخدام Role باسم `odoo` |
| `-W` | طلب Password الخاصة بالـRole |
| `odoo20_training` | اسم Database الجديدة |

يستخدم هذا الأمر Port الافتراضية `5432`.

بعد تنفيذه، سيظهر طلب Password مثل:

```text
Password:
```

اكتب Password التي حددتها للـRole `odoo` داخل pgAdmin، ثم اضغط `Enter`.

لن تظهر الأحرف أو النجوم أثناء الكتابة؛ هذا طبيعي.

> [!note] اسم Role وPassword
> `odoo` هو اسم PostgreSQL Role.
>
> استخدم Password التي اخترتها عند إنشائها.

عند نجاح الأمر، تُنشأ PostgreSQL Database باسم:

```text
odoo20_training
```

لكنها لم تُهيأ بعد لتشغيل Odoo.

سننفذ التهيئة في الخطوة التالية.

---

#### 17. تهيئة Database

أنشأنا PostgreSQL Database في الخطوة السابقة، لكنها لم تُهيأ بعد لتشغيل Odoo.

الآن سنستخدم Odoo لتثبيت Module الأساسية `base` وإنشاء Tables والبيانات المطلوبة.

نفّذ من Project Root:

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\odoo.conf `
  -d odoo20_training `
  -i base `
  --stop-after-init
```

معنى أجزاء الأمر:

| الجزء | وظيفته |
|---|---|
| `.\.venv\Scripts\python.exe` | استخدام Python الخاصة بالمشروع |
| `odoo\odoo-bin` | تشغيل Odoo من Source Code |
| `-c config\odoo.conf` | استخدام Configuration الخاصة بالمشروع |
| `-d odoo20_training` | تحديد Database التي سنهيئها |
| `-i base` | تثبيت Module الأساسية `base` |
| `--stop-after-init` | إيقاف Odoo بعد اكتمال التهيئة |

أثناء العملية، تنشئ Odoo Tables والبيانات الأساسية المطلوبة.

تحقق من اكتمال العملية دون Errors تمنع التهيئة. بعد انتهاء Process والعودة إلى Prompt، ننتقل إلى تشغيل Odoo بالأمر العادي.

> [!note] تهيئة أولية
> نستخدم `-i base` عند تهيئة Database لأول مرة.
>
> لا نضيفها إلى أمر التشغيل اليومي.

---

#### 18. تشغيل Odoo لأول مرة

شغّل Server بالأمر العادي:

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin `
  -c config\odoo.conf `
  -d odoo20_training
```

بعد اكتمال التحميل وبدء HTTP Server، افتح المتصفح وانتقل إلى:

```text
http://127.0.0.1:8069
```

اترك PowerShell مفتوحة أثناء تشغيل Server.

للإيقاف:

```text
Ctrl + C
```

أكمل الآن الخطوات التالية.

---

### After Starting Odoo

بعد هذه النقطة تصبح الخطوات متشابهة على Fedora وWindows.

---

#### تسجيل الدخول لأول مرة

في Database جديدة تمت تهيئتها بالطريقة السابقة، يستخدم حساب الإدارة الأساسي:

```text
Login: admin
Password: admin
```

بعد الدخول، غيّر Password الحساب من إعدادات المستخدم.

إذا كانت Database موجودة مسبقًا أو مستعادة، استخدم بيانات حسابها الفعلية.

> [!important] Master Password مختلفة
> `admin_passwd` الموجودة داخل `odoo.conf` ليست Password تسجيل الدخول.
>
> هي Master Password مرتبطة بعمليات إدارة Databases.

---

#### تفعيل Developer Mode

Developer Mode تعرض أدوات ومعلومات تقنية مفيدة أثناء التطوير.

من داخل Odoo:

```text
Settings
→ Developer Tools
→ Activate the developer mode
```

![[Developer Tools Setteings.png]]

يمكن أيضًا فتح:

```text
http://127.0.0.1:8069/odoo?debug=1
```

بعد التفعيل، تظهر أدوات تقنية في الصفحات المناسبة:

![[Developer Tools.png]]

وقد تظهر قائمة Developer Tools بحسب الصفحة:

![[Developer Tools 2.png]]

الخيارات المتاحة تعتمد على:

- الصفحة الحالية.
- Modules المثبتة.
- صلاحيات المستخدم.

> [!note] Developer Mode وSecurity
> Developer Mode لا تمنح المستخدم Access Rights إضافية.
>
> تظل العمليات خاضعة لصلاحيات الحساب.

---

#### التحقق من اكتمال الإعداد

قبل متابعة المرجع، تحقق من:

- [ ] Python داخل `.venv` تعمل.
- [ ] تثبيت `requirements.txt` اكتمل.
- [ ] `pip check` لا تعرض تعارضات معلنة.
- [ ] PostgreSQL Connection تعمل باستخدام Role المناسبة.
- [ ] PostgreSQL Server تستخدم Version مناسبة.
- [ ] Configuration تحتوي المسارات الفعلية.
- [ ] Database التدريب تمت تهيئتها.
- [ ] Odoo تعمل بأمر التشغيل العادي.
- [ ] يمكن تسجيل الدخول.
- [ ] تم تغيير Password الإدارة الأولية.
- [ ] Developer Mode متاحة للحساب المناسب.

لا تتوقع ظهور Equipment Menus الآن.

لم ننشئ Module التدريب بعد؛ سنبدأ بها في [[03 Modules & Addons]].

---

## Understanding the Environment

بعد أن جهزنا البيئة، نستطيع فهم وظيفة كل مكوّن والعلاقة بين الأجزاء.

---

### كيف يبدأ Odoo؟

عند تنفيذ أمر التشغيل:

```text
Python داخل .venv
        ↓
تشغّل odoo-bin
        ↓
Odoo تقرأ Configuration
        ↓
تحدد Addons Paths وDatabase Connection
        ↓
تحمّل Modules الخاصة بالـDatabase
        ↓
تبدأ HTTP Server
        ↓
Browser تتصل بها
```

وتحتاج Odoo إلى الوصول إلى مكانين مختلفين للبيانات:

```text
Odoo
├── PostgreSQL → Database
└── data_dir   → Filestore وRuntime Data
```

Directory `data/` ليست مكان تخزين PostgreSQL Database نفسها.

---

### Source Installation

في Source Installation نشغّل Odoo مباشرة من Source Code.

هذا يجعل عناصر مثل الآتية واضحة أمام المطور:

- Version المستخدمة.
- موضع Source Code.
- Configuration.
- Custom Addons.
- أمر التشغيل.
- Logs.
- أدوات Debugging والاختبار.

وجود Source Code على الجهاز لا يكفي وحده؛ تحتاج أيضًا إلى Python وDependencies وPostgreSQL وConfiguration مناسبة.

---

### Git

Git هي **Version Control System**، نستخدمها لإدارة Source Code وتاريخ التعديلات.

تساعدنا على:

- معرفة ما تغير في الكود.
- الرجوع إلى تعديلات سابقة.
- العمل على Branches مختلفة.
- مراجعة التغييرات ومشاركتها مع الفريق.

في خطوات الإعداد، استخدمنا `git clone` لتحميل Odoo Source من Repository الرسمية إلى:

```text
odoo20-dev/odoo/
```

لاحقًا، سنستخدم Git أيضًا لتتبع Custom Modules التي نطورها.

> [!note] الكود والبيانات
> Git تدير ملفات الكود وتاريخها.
>
> حفظ Database وFilestore يحتاج إلى Backup، كما سنشرح في [[30 Backup & Restore]].

---

### Python وpip وVirtual Environment

| المكوّن | وظيفته |
|---|---|
| Python | تشغيل Odoo Server |
| pip | تثبيت Python Packages |
| `.venv` | عزل Packages الخاصة بالمشروع |
| `requirements.txt` | تحديد Dependencies المطلوبة للنسخة |

استخدمنا Python داخل `.venv` صراحةً حتى نعرف Environment التي تشغّل Commands.

مثلًا:

```bash
.venv/bin/python -m pip
```

هذا يربط `pip` بالـInterpreter المحددة.

أما تشغيل `pip` وحدها فقد يستخدم Installation أخرى بحسب `PATH`.

> [!note] تغيير Python
> تحديث Python المثبتة على الجهاز لا يحدّث `.venv` الموجودة تلقائيًا.
>
> إذا تغيرت Interpreter المعتمدة، جهز Environment مناسبة باستخدامها وأعد تثبيت متطلبات المشروع.

---

### System Dependencies وBuild Tools

بعض Dependencies تتكون من Python Code فقط.

وبعضها يحتوي أجزاء تحتاج أثناء البناء إلى:

```text
Compiler
Header Files
System Libraries
Development Libraries
```

لهذا جهزنا:

- Build Tools وDevelopment Packages على Fedora.
- Visual Studio Build Tools وWindows SDK على Windows.

Virtual Environment تعزل Python Packages، لكنها لا تعوض مكونات النظام التي تحتاجها هذه Packages.

---

### PostgreSQL

من المهم التفريق بين ثلاثة أشياء:

| المكوّن | وظيفته |
|---|---|
| PostgreSQL Server | الخدمة التي تستضيف Databases |
| PostgreSQL Role | الحساب المستخدم للاتصال بالخدمة |
| Database | قاعدة بيانات محددة داخل Server |

يمكن لـServer واحدة استضافة أكثر من Database.

في هذا المرجع سنستخدم لاحقًا قواعد منفصلة لأغراض مختلفة:

```text
odoo20_training
→ التدريب والتطوير

odoo20_tests
→ الاختبارات

odoo20_restore_test
→ تجارب الاستعادة والترقية المعزولة
```

في هذا الشابتر أنشأنا الأولى فقط.

---

### PostgreSQL Role وOdoo User

PostgreSQL Role تستخدمها Server للوصول إلى Database.

أما Odoo User فتستخدم لتسجيل الدخول إلى التطبيق، وتخضع لصلاحيات Odoo.

الصورة المبسطة:

```text
Odoo Users
    ↓
Odoo Server
    ↓
PostgreSQL Role
    ↓
Database
```

لا ننشئ PostgreSQL Role لكل مستخدم يتعامل مع واجهة Odoo.

كذلك تختلف Master Password الموجودة في `admin_passwd` عن حساب PostgreSQL وعن حسابات واجهة Odoo.

---

### Odoo Source

داخل Directory المحمّلة ستجد عناصر مثل:

```text
odoo/
├── addons/
├── odoo/
├── odoo-bin
├── requirements.txt
└── ...
```

انتبه إلى تكرار اسم `odoo`:

| المسار من Project Root | المحتوى |
|---|---|
| `odoo/` | Repository المحمّلة كاملة |
| `odoo/odoo/` | Framework Source |
| `odoo/addons/` | Addons تأتي مع Odoo |
| `odoo/odoo-bin` | Entry Point لتشغيل Odoo |
| `odoo/requirements.txt` | Python Dependencies |

توجد Addons أساسية أيضًا داخل Framework Source.

في البداية يكفي أن تعرف وظيفة الأجزاء التي تستخدمها، ثم تستكشف التفاصيل عند الحاجة.

---

### Custom Addons

خصصنا Directory:

```text
custom-addons/
```

لـModules التي سنطورها أو نضيفها إلى المشروع.

هذا يساعد على إبقاء Custom Code منظمة عند:

- مراجعة التغييرات.
- تحديث Odoo.
- تشغيل الاختبارات.
- Deployment.
- Troubleshooting.

اسم `custom-addons` ليس إلزاميًا؛ هو تنظيم اخترناه.

المهم إضافة Directory المناسبة إلى `addons_path`.

سنشرح بنية Module نفسها في [[03 Modules & Addons]].

---

### Configuration

استخدمنا:

```text
config/odoo.conf
```

حتى لا نكرر جميع Options في كل أمر تشغيل.

أهم الإعدادات:

| الإعداد | وظيفته |
|---|---|
| `admin_passwd` | Master Password لإدارة Databases |
| `addons_path` | Directories التي يبحث فيها Odoo عن Modules |
| `data_dir` | موقع Runtime Data |
| `http_interface` | Interface التي تستمع عليها HTTP Server |
| `http_port` | Port الخاصة بـHTTP |
| `db_host` | PostgreSQL Host عند تحديد اتصال TCP |
| `db_port` | PostgreSQL Port |
| `db_user` | PostgreSQL Role |
| `db_password` | Password الخاصة بالـRole |

طريقة الاتصال قد تختلف بين Local Setup ومشروع يستخدم Container أو Database Server منفصلة.

---

### `addons_path`

يمكن أن تحتوي `addons_path` أكثر من Directory.

في إعدادنا استخدمنا:

```text
odoo/addons
custom-addons
```

لاحقًا سيكون لدينا:

```text
custom-addons/
└── equipment_training/
```

نضع `custom-addons` في `addons_path`، وليس Directory كل Module بشكل منفصل.

وجود المسار يجعل Module قابلة للاكتشاف عند استيفاء بنيتها.

تثبيت Module داخل Database إجراء مستقل.

---

### `data_dir`

يستخدمها Odoo لحفظ Runtime Data، ومنها Filestore وSessions.

قد يظهر تنظيم قريب من:

```text
data/
├── filestore/
│   └── odoo20_training/
└── sessions/
```

Filestore قد تحتوي محتوى Attachments المرتبطة بـDatabase.

لذلك لا تحذف `data/` باعتبارها Cache يمكن الاستغناء عنها.

راجع [[30 Backup & Restore]] عند تعلم حفظ Database وFilestore واستعادتهما معًا.

---

### HTTP وPostgreSQL Ports

في إعدادنا:

| الخدمة | Port |
|---|---|
| Odoo HTTP | `8069` |
| PostgreSQL | `5432` |

Browser تتصل بـOdoo على:

```text
http://127.0.0.1:8069
```

أما Odoo فتستخدم إعدادات PostgreSQL للوصول إلى Database.

هاتان خدمتان مختلفتان، ولكل واحدة إعدادات اتصالها.

---

### `.gitignore`

`.gitignore` تحدد الملفات التي لا نريد تتبعها ضمن Repository.

في هذا التنظيم، نستبعد أشياء مثل:

- Virtual Environment.
- Runtime Data.
- Secrets.
- Backups.
- Cache وLogs.

ونحتفظ بالكود وConfiguration Examples المناسبة للمشاركة.

> [!important] الملفات المتتبعة مسبقًا
> إضافة ملف إلى `.gitignore` لا تزيله من Git إذا سبق تتبعه.
>
> راجع الملفات المتتبعة قبل مشاركة Repository، خاصة Configuration وBackups.

---

## Daily Development Workflow

بعد تجهيز البيئة، لا تحتاج إلى إعادة Installation في كل جلسة.

الاستخدام اليومي أبسط من الإعداد الأول.

---

### 1. ابدأ من Project Root

افتح Project في المحرر، ثم Terminal داخل:

```text
odoo20-dev/
```

للتحقق:

#### Fedora

```bash
pwd
```

#### Windows

```powershell
Get-Location
```

يجب أن ترى داخل Current Directory عناصر مثل:

```text
.venv/
odoo/
custom-addons/
config/
```

الأوامر النسبية في المرجع تعتمد على هذا الموضع.

---

### 2. تأكد من PostgreSQL

يجب أن تكون PostgreSQL متاحة قبل تشغيل Odoo.

#### Fedora

```bash
pg_isready
```

#### Windows

```powershell
pg_isready -h 127.0.0.1 -p 5432
```

إذا كانت Service لا تعمل، شغّل Installation الصحيحة.

لا تحتاج إلى إعادة تهيئة PostgreSQL أو إنشاء Role وDatabase في كل جلسة.

---

### 3. شغّل Odoo

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

افتح:

```text
http://127.0.0.1:8069
```

اترك Terminal مفتوحة أثناء تشغيل Server.

---

### 4. أوقف Odoo عند الانتهاء

من Terminal التي تعمل فيها Server:

```text
Ctrl + C
```

المسار اليومي المعتاد:

```text
Project Root
    ↓
Run Odoo
    ↓
Develop وTest
    ↓
Stop Odoo
```

---

### 5. استخدم Odoo Shell عند الحاجة

Odoo Shell توفر Python Console مرتبطة بـDatabase محددة.

تفيد في:

- فحص Records.
- تجربة ORM Queries.
- استكشاف Models.
- تشخيص مشكلة.

#### Fedora

```bash
.venv/bin/python \
  odoo/odoo-bin shell \
  -c config/odoo.conf \
  -d odoo20_training
```

#### Windows

```powershell
.\.venv\Scripts\python.exe `
  odoo\odoo-bin shell `
  -c config\odoo.conf `
  -d odoo20_training
```

داخل Shell، تحقق من Database:

```python
env.cr.dbname
```

المتوقع:

```text
odoo20_training
```

وجرّب:

```python
env.user
```

لا تحتاج إلى فهم `env` بالتفصيل الآن.

سنشرح استخدامها في [[05 ORM]] و[[09 Environment & Context]].

للخروج:

```python
exit()
```

> [!important] صلاحيات Shell والحفظ
> Odoo Shell الافتراضية تبدأ بسياق Superuser.
>
> نجاح عملية فيها لا يثبت أن مستخدم الواجهة العادي يستطيع تنفيذها.
>
> التغييرات غير المثبتة تُلغى عند الخروج. الحفظ الصريح باستخدام `env.cr.commit()` يكون فقط عندما تقصد حفظ تعديل داخل Database التدريب.
>
> أمثلة القراءة السابقة لا تحتاج إلى Commit.

---

### 6. تحقق من Python Environment عند الحاجة

للتحقق من Python:

#### Fedora

```bash
.venv/bin/python --version
```

#### Windows

```powershell
.\.venv\Scripts\python.exe --version
```

ولفحص Dependencies:

#### Fedora

```bash
.venv/bin/python -m pip check
```

#### Windows

```powershell
.\.venv\Scripts\python.exe -m pip check
```

لا تحتاج إلى إعادة تثبيت Requirements يوميًا إذا لم تتغير Environment أو المتطلبات.

---

### 7. أعد التشغيل عند تغيير Configuration

إذا عدلت إعدادات مثل:

- `addons_path`.
- Database Connection.
- `data_dir`.
- HTTP Port.

أوقف Odoo، ثم شغّلها من جديد لقراءة Configuration الحالية.

كذلك تحتاج Process جديدة لقراءة تعديلات Python في المسار المعتاد.

أما إضافة Fields أو تحديث Views وModule Data فقد تحتاج أيضًا إلى **Module Upgrade**.

سنشرح الفرق بين Restart وInstallation وUpgrade عمليًا في [[03 Modules & Addons]].

---

## Existing Company Project

### عندما لا تبدأ من الصفر

في العمل قد تستلم Project تستخدم:

```text
Docker
Odoo.sh
Virtual Machine
Development Server
Managed PostgreSQL
CI/CD
Internal Repositories
```

ابدأ بفهم Environment الموجودة قبل تطبيق خطوات Local Setup.

راجع Documentation مثل:

- README.
- Developer Setup.
- Configuration Examples.
- Docker Files.
- Deployment Scripts.
- متطلبات Environment.
- طريقة إدارة Secrets.

---

### ما الذي تحتاج إلى معرفته؟

قبل أول تعديل، حدد:

| المجال | المعلومات المطلوبة |
|---|---|
| Odoo | Version وEdition وRevision |
| Source | Repositories وBranches المعتمدة |
| Python | Interpreter وDependencies |
| Addons | مواقع Custom Modules و`addons_path` |
| Database | Development Database وطريقة الاتصال |
| Runtime Data | موقع Filestore وStorage |
| Configuration | مصدر الإعدادات وطريقة تطبيقها |
| Integrations | الخدمات الخارجية وبيئات الاختبار |
| Operations | طريقة التشغيل والاختبار وUpgrade |

لا تفترض أن Configuration تأتي دائمًا من `odoo.conf`.

قد تعتمد على Environment Variables أو Scripts أو Secret Manager.

وفي Containers، قد تختلف Paths الداخلية عن Paths الموجودة على جهازك.

---

### التعامل مع بيانات مستعادة

إذا استخدمت نسخة من Database حقيقية، اعمل داخل بيئة اختبار معزولة وفق إجراءات المشروع.

راجع خصوصًا:

- Email.
- Webhooks.
- Scheduled Actions.
- Payment أو SMS Integrations.
- Credentials الخاصة بالخدمات الخارجية.

قد تحمل Database المستعادة إعدادات تشغيلية حقيقية.

راجع [[30 Backup & Restore]] لفهم Neutralization والعزل قبل استخدامها في التجارب.

---

### التحقق قبل بدء التطوير

تأكد من أنك تستطيع:

```text
تشغيل Environment بالطريقة المعتمدة
        ↓
الاتصال بـDevelopment Database
        ↓
تحميل Custom Addons المطلوبة
        ↓
تسجيل الدخول
        ↓
تنفيذ فحص أساسي معروف
```

بعد ذلك ابدأ التعديل على أساس واضح.

---

## Important Notes

- استخدم `requirements.txt` الموجودة داخل Odoo Source التي حمّلتها، ولا تخلط Dependencies من إصدارات مختلفة.
- ميّز بين PostgreSQL Role وحساب المستخدم داخل Odoo وMaster Password الموجودة في `admin_passwd`؛ لكل منها وظيفة مختلفة.
- استخدم Database مخصصة للتطوير، وتجنب إجراء التجارب على Production Database.
- احتفظ بـPasswords وSecrets و`odoo.conf` الحقيقية خارج Git Repository.
- عند استلام مشروع موجود، افهم طريقة تشغيله أولًا قبل تطبيق خطوات Local Setup عليه.

---

> [!tip] الشابتر التالي
> انتقل إلى:
>
> [[03 Modules & Addons]]

---

## Sources

- [Odoo 20 Documentation — Source Installation](https://www.odoo.com/documentation/20.0/administration/on_premise/source.html)
- [Odoo 20 Source — Release and Runtime Versions](https://github.com/odoo/odoo/blob/20.0/odoo/release.py)
- [Odoo 20 Source — Python Requirements](https://github.com/odoo/odoo/blob/20.0/requirements.txt)
- [Odoo 20 Documentation — Command-line Interface](https://www.odoo.com/documentation/20.0/developer/reference/cli.html)
- [Odoo 20 Source — Configuration Options](https://github.com/odoo/odoo/blob/20.0/odoo/tools/config.py)
- [Odoo 20 Source — Initial Users](https://github.com/odoo/odoo/blob/20.0/odoo/addons/base/data/res_users_data.xml)
- [Odoo 20 Source — Odoo Shell](https://github.com/odoo/odoo/blob/20.0/odoo/cli/shell.py)
- [Odoo 20 Documentation — Developer Mode](https://www.odoo.com/documentation/20.0/applications/general/developer_mode.html)