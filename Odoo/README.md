## Download

يمكنك قراءة مرجع **Odoo Development** مباشرة من GitHub، أو تحميله على جهازك لقراءته دون اتصال بالإنترنت.

يوفر المرجع ثلاث طرق للتحميل، حسب طريقة القراءة التي تفضلها.

### 1. With Obsidian (Recommended)

هذه الطريقة مناسبة للحصول على تجربة القراءة الكاملة باستخدام [Obsidian](https://obsidian.md/)، مع ثيم **Catppuccin** وإضافة **RTL** لتحسين عرض النصوص العربية والإنجليزية.

**Requirements**

- [Git](https://git-scm.com/downloads)
    
- [Obsidian](https://obsidian.md/)


**Installation**

1. افتح Terminal على Linux أو macOS، أو PowerShell على Windows.
    
2. انتقل إلى المجلد الذي تريد حفظ المرجع بداخله، مثل مجلد `Documents`.
    
    **Linux / macOS:**
    
    ```
    cd ~/Documents
    ```
    
    **Windows (PowerShell):**
    
    ```
    cd "$HOME\Documents"
    ```
    
    > يمكنك اختيار أي مجلد آخر. سيتم إنشاء مجلد جديد باسم `My-Library` داخل المسار الذي حددته.
    
3. نفّذ الأوامر التالية لتحميل مرجع Odoo مع إعدادات Obsidian:
    
    ```
    git clone --filter=blob:none --sparse https://github.com/marwanalasmi/My-Library.git
    cd My-Library
    git sparse-checkout set Odoo .obsidian
    ```
    
4. افتح Obsidian واختر **Open folder as vault**.
    
5. حدد مجلد `My-Library` الذي تم إنشاؤه.
    
6. من **Settings → Community plugins**، فعّل الإضافات المطلوبة بعد مراجعتها، إذا لم تكن مفعّلة.
    
7. افتح `Odoo/00 Overview.md` للبدء بالقراءة.
    

**Folder Structure**

بعد التحميل، سيظهر المجلد بالشكل التالي:

```
My-Library/
├── .obsidian/
├── Odoo/
│   ├── 00 Overview.md
│   ├── 01 Odoo Architecture.md
│   ├── ...
│   ├── Attachments/
│   └── README.md
└── README.md
```

> **Note:** قد يطلب Obsidian موافقتك على تشغيل Community Plugins. ملفات Markdown قابلة للقراءة حتى دون تفعيل هذه الإضافات.

---

### 2. Without Obsidian

إذا كنت ترغب في قراءة ملفات المرجع باستخدام محرر Markdown آخر، يمكنك تحميل **Odoo فقط** دون إعدادات Obsidian.

**Requirements**

- [Git](https://git-scm.com/downloads)
    
- أي محرر يدعم Markdown


**Installation**

1. افتح Terminal أو PowerShell.
    
2. انتقل إلى المجلد الذي تريد حفظ المرجع فيه، كما هو موضح في الطريقة الأولى.
    
3. نفّذ:
    
    ```
    git clone --filter=blob:none --sparse https://github.com/marwanalasmi/My-Library.git
    cd My-Library
    git sparse-checkout set Odoo
    ```
    
4. افتح مجلد `Odoo` باستخدام محرر Markdown المفضل لديك.
    

سيتم تحميل ملفات الشرح والمرفقات، دون إعدادات Obsidian. وستبقى بعض الملفات الأساسية في جذر المستودع.

> **Note:** بعض الروابط والتنسيقات الخاصة بـObsidian قد لا تعمل بالطريقة نفسها في محررات Markdown الأخرى.

---

### 3. Download ZIP (No Git Required)

إذا كنت تفضل تحميل الملفات مباشرة دون استخدام Terminal أو تثبيت Git:

1. افتح مستودع [My-Library على GitHub](https://github.com/marwanalasmi/My-Library).
    
2. اضغط الزر الأخضر **Code**.
    
3. اختر **Download ZIP**.
    
4. فك ضغط الملف في المكان الذي تريده على جهازك.
    
5. افتح مجلد `Odoo` لقراءة المرجع، أو افتح مجلد المكتبة كاملًا باستخدام Obsidian.
    

> **Note:** هذه الطريقة تحمّل المستودع كاملًا، وليس مجلد Odoo فقط.