# Odoo Development Notes  
  
مرجع تعليمي باللغة العربية لتطوير **Odoo 20**، يجمع الشروحات والأمثلة العملية والملاحظات التي أعدّها أثناء التعلم.  
  
يغطي المحتوى موضوعات تبدأ بفهم بنية Odoo وتجهيز بيئة التطوير، وتمتد إلى بناء Modules واختبارها ونشرها وصيانتها.  
  
## Table of Contents  
  
- [About](#about)  
- [Who Is This For?](#who-is-this-for)  
- [Content](#content)  
- [How to Read](#how-to-read)  
- [Download](#download)  
- [Sources](#sources)  
  
---  
  
## About  
  
يهدف هذا المرجع إلى تنظيم مفاهيم تطوير Odoo وشرحها بأسلوب واضح، مع أمثلة عملية تساعد على فهمها وتطبيقها.  
  
الشروحات باللغة العربية، مع الاحتفاظ بالمصطلحات التقنية باللغة الإنجليزية لتسهيل ربطها بالكود والوثائق الرسمية.  
  
المحتوى مقسم إلى شابترات مرقمة، ويمكن قراءتها بالتسلسل أو الرجوع إلى موضوع محدد عند الحاجة.  
  
---  
  
## Who Is This For?  
  
المحتوى مناسب لمن:  
  
- **يبدأ تعلم تطوير Odoo:** ويريد فهم المفاهيم الأساسية ومتابعة الموضوعات بترتيب واضح.  
- **يحتاج إلى مرجع أثناء التطوير:** ويريد الرجوع إلى شرح مرتب للمفاهيم والأمثلة عند التطبيق أو مراجعة موضوع معين.  
  
---  
  
## Content  
  
| المجال | الموضوعات |  
|---|---|  
| الأساسيات | Architecture، Development Environment، Modules وAddons |  
| البيانات | Models، Fields، ORM، Recordsets، Relationships وDomains |  
| الواجهة | Views، XML، Actions، Menus، Buttons وWidgets |  
| منطق العمل | Business Logic، Computed Fields، Onchange، Constraints، Inheritance وWizards |  
| الصلاحيات | Security وMulti-Company |  
| التواصل والتقارير | QWeb، Reports، Chatter، Activities وAutomation |  
| التكامل والواجهة الأمامية | HTTP Controllers، Portal، OWL وAPIs |  
| الاختبار والتحسين | Testing، Debugging وPerformance |  
| التشغيل والصيانة | Deployment، Backup، Migration وMaintenance |  
| اللغات | Translations وLocalization |  
  
---  
  
## How to Read  
  
1. ابدأ بملف [00 Overview.md](00%20Overview.md) للتعرف على خريطة المرجع.
2. تابع الشابترات حسب ترتيبها الرقمي عند القراءة الأولى.  
3. طبّق الأمثلة أثناء التعلم، وارجع إلى الموضوعات السابقة عند الحاجة.  
4. استخدم مصادر كل شابتر للتوسع ومراجعة التفاصيل.  
  
ملفات المرجع مكتوبة بصيغة Markdown باستخدام **Obsidian**. يُنصح باستخدامه لعرض الروابط الداخلية والصور والتنسيقات كما أُعدّت.  
  
يمكن أيضًا تصفح الملفات على GitHub أو قراءتها بمحرر آخر، مع احتمال اختلاف عرض بعض تنسيقات Obsidian.  
  
---  
  
## Download  
  
يمكنك تصفح ملفات مرجع **Odoo Development** على GitHub، أو تحميلها لقراءتها على جهازك دون اتصال بالإنترنت.  
  
اختر الطريقة المناسبة لك من الخيارات التالية.  
  
### 1. With Obsidian (Recommended)  
  
هذه الطريقة مناسبة لقراءة المرجع داخل **Obsidian**، مع إعدادات المكتبة المرفقة وثيم **Catppuccin** وإضافة **RTL** لتحسين عرض النصوص العربية والإنجليزية.  
  
**Requirements**  
  
- [Git](https://git-scm.com/downloads)  
- [Obsidian](https://obsidian.md/)  
  
**Installation**  
  
1. افتح Terminal على Linux أو macOS، أو PowerShell على Windows.  
  
2. انتقل إلى المجلد الذي تريد حفظ المرجع بداخله، مثل `Documents`.  
  
**Linux / macOS**  
  
```bash  
cd ~/Documents  
```  
  
**Windows — PowerShell**  
  
```powershell  
cd "$HOME\Documents"  
```  
  
يمكنك اختيار أي مسار آخر. سيتم إنشاء مجلد باسم `My-Library` داخل المسار الحالي.  
  
3. نفّذ الأوامر التالية لتحميل مجلد Odoo وإعدادات Obsidian، مع الملفات الأساسية الموجودة في جذر المستودع:  
  
```bash  
git clone --filter=blob:none --sparse https://github.com/marwanalasmi/My-Library.git  
cd My-Library  
git sparse-checkout set Odoo-Development .obsidian  
```  
  
هذه الأوامر صالحة أيضًا في PowerShell.  
  
4. افتح Obsidian واختر **Open folder as vault**.  
  
5. حدد مجلد **My-Library** الذي تم إنشاؤه، ليتم تحميل إعدادات المكتبة الموجودة داخله.  
  
6. إذا ظهرت رسالة تتعلق بإضافات المجتمع، راجع الإضافات وفعّل ما تحتاج إليه من:  
  
**Settings → Community plugins**  
  
7. افتح ملف **Odoo-Development/00 Overview.md** للبدء بالقراءة.  
  
**Folder Structure**  
  
سيكون تنظيم الملفات الرئيسية قريبًا من:  
  
```text  
My-Library/  
├── .obsidian/  
├── Odoo-Development/  
│ ├── 00 Overview.md  
│ ├── 01 Odoo Architecture.md  
│ ├── ...  
│ ├── Attachments/  
│ └── README.md  
└── README.md  
```  
  
> [!NOTE]  
> إعدادات Obsidian اختيارية، ويمكن قراءة ملفات Markdown الأساسية دون تفعيل إضافات المجتمع. راجع الإضافات قبل تفعيلها.  
  
---  
  
### 2. Markdown Files Only  
  
هذه الطريقة مناسبة إذا كنت تفضل استخدام محرر Markdown آخر، أو تريد الحصول على ملفات **Odoo Development** دون إعدادات Obsidian المرفقة.  
  
**Requirements**  
  
- [Git](https://git-scm.com/downloads)  
- محرر Markdown أو Text Editor.  
  
**Installation**  
  
1. افتح Terminal أو PowerShell.  
  
2. انتقل إلى المجلد الذي تريد حفظ المرجع بداخله، كما هو موضح في الطريقة الأولى.  
  
3. نفّذ الأوامر التالية:  
  
```bash  
git clone --filter=blob:none --sparse https://github.com/marwanalasmi/My-Library.git  
cd My-Library  
git sparse-checkout set Odoo-Development  
```  
  
هذه الأوامر صالحة أيضًا في PowerShell.  
  
4. افتح مجلد **Odoo-Development** باستخدام المحرر الذي تفضله.  
  
ستجد ملفات الشرح والصور والمرفقات داخل مجلد `Odoo-Development`، بالإضافة إلى الملفات الأساسية الموجودة في جذر المستودع.  
  
> [!NOTE]  
> تحتفظ الملفات بتنسيقاتها الأصلية المكتوبة باستخدام Obsidian، لذلك قد يختلف عرض بعض الروابط الداخلية والصور والتنبيهات حسب المحرر المستخدم.  
  
---  
  
### 3. Download ZIP (No Git Required)  
  
هذه الطريقة مناسبة إذا كنت تريد تحميل الملفات مباشرة دون استخدام Terminal أو تثبيت Git.  
  
1. افتح [My-Library على GitHub](https://github.com/marwanalasmi/My-Library).  
  
2. اضغط **Code → Download ZIP**.  
  
3. فك ضغط الملف في المكان الذي تريده على جهازك.  
  
4. افتح مجلد **Odoo-Development** لقراءة ملفات المرجع باستخدام المحرر الذي تفضله.  
  
5. لفتح المكتبة داخل Obsidian، اختر **Open folder as vault** وحدد المجلد المستخرج الذي يحتوي على `Odoo-Development` و`.obsidian`.  
  
**هذه الطريقة تحمّل ملفات المستودع كاملًا، بما فيها أي مواضيع أخرى موجودة، ولا تقتصر على Odoo Development.**  
  
---  

### Updates

إذا حمّلت المرجع باستخدام Git، يمكنك الحصول على أحدث التعديلات دون الحاجة إلى إعادة تحميله.

افتح Terminal أو PowerShell داخل مجلد `My-Library`، ثم نفّذ:

```bash
git pull
```

سيجلب Git آخر التحديثات من المستودع ويطبّقها على نسختك المحلية، ما لم توجد تعديلات محلية متعارضة.

أما إذا استخدمت **Download ZIP**، فستحتاج إلى تحميل نسخة جديدة للحصول على التحديثات.

---

## Sources  
  
تعتمد الشروحات التقنية على **وثائق Odoo الرسمية** والكود المصدري الرسمي عند الحاجة.  
  
توجد روابط المصادر المستخدمة في نهاية كل شابتر.  
  
- [Odoo 20 — Developer Documentation](https://www.odoo.com/documentation/20.0/developer.html)  
- [Odoo 20 — Official Source Code](https://github.com/odoo/odoo/tree/20.0)