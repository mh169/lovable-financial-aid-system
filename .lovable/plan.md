

# سكريبت تهيئة بيانات Excel كنسخة احتياطية قابلة للاستيراد المباشر

## الفكرة
سكريبت Python مستقل يحوّل ملف Excel القديم إلى ملف **نسخة احتياطية SQL** (`legacy-backup.sql`) يمكن تنفيذه مباشرة على قاعدة بيانات SQLite الخاصة بالنظام v7.1 — بدون الحاجة لـ endpoint استيراد أو تعديل النظام.

## لماذا SQL مباشر؟
- النظام يستخدم SQLite محلياً
- لا يحتاج تعديل أي ملف في النظام
- يعمل كـ "استرجاع نسخة احتياطية" حقيقي
- يضمن عدم تأثر عدّاد الترقيم (لأنه يقرأ من جدول منفصل أو من max للأرقام الرقمية فقط)

## المخرجات (في `/mnt/documents/legacy-import/`)

1. **`prepare-legacy-data.py`** — السكريبت الرئيسي
2. **`legacy-backup.sql`** — ملف SQL جاهز للتنفيذ على `database.db`
3. **`prepared-data.json`** — نسخة JSON احتياطية (للمراجعة)
4. **`import-report.txt`** — تقرير عربي مفصّل
5. **`README.md`** — تعليمات الاستخدام بالعربية

## خريطة الأعمدة (Excel ← → النظام)

| عمود Excel | حقل النظام |
|-----------|-----------|
| تاريخ التقرير | reportDate |
| نوع الطلب | requestType |
| جهة الرفع | submittingEntity |
| مضمون التوجيه | directiveContent |
| ملاحظات | notes |
| المرفقات | attachments |
| الرقم الطبي | medicalNo |
| اسم المستفيد | patientName |
| اسم الشهيد | martyrName |
| رقم الباركود | martyrBarcode |
| رقم استمارة المسح | martyrNo |
| الحالة | caseStatus |
| صلة القرابة | relationship |
| العمر / الجنس / المحافظة / المديرية / رقم الهاتف | age, gender, governorate, district, phone |
| نبذة عن الحالة | caseSummary |
| الاحتياج / تفاصيل الاحتياج / البيان | needType, needDetails, statement |
| عرض السعر / الخصم بعد التنسيق | priceQuote, discountAmount |
| **رقم الاستمارة** | **formNumber** (يُحفظ كما هو: `2025_2098`) |
| المبلغ ريال / دولار / كتابة | amountYer, amountUsd, amountWords |
| جهة التسليم / اسم المستلم / ملاحظة / هاتف / الصفة | deliveryEntity, recipientName, recipientNote, recipientPhone, recipientRelation |
| اليوم / الشهر | day, month |
| اسم المشروع / من بند | projectName, budgetItem |
| حالة الطباعة | printStatus |
| رقم مسير الصرف | voucherNo |
| حالة التحصيل | collectionState |
| رقم مسير الإخلاء | evacuationNo |
| ملاحظات حالة الإخلاء | evacuationNotes |
| حالة الإخلاء | evacuationState |

## ميزات السكريبت

1. **قراءة Excel** بـ pandas (xlsx/xls/csv)
2. **تنظيف**: trim، تحويل فراغات لـ NULL، تحويل الأرقام، توحيد التواريخ ISO
3. **علامة `is_legacy = 1`** لكل سجل (لاستبعادها من عدّاد الترقيم في v8 لاحقاً)
4. **حماية الترقيم**: الأرقام القديمة تحتوي `_` (مثل `2025_2098`) — السكريبت يضمن عدم تعارضها مع الترقيم الرقمي الجديد (`000001`, `000002`...)
5. **معالجة الأخطاء**:
   - السجلات بدون formNumber أو patientName → تُسجّل كأخطاء وتُتجاهل
   - أرقام مكررة داخل Excel نفسه → تنبيه
   - تواريخ غير صالحة → محاولة إصلاح
6. **مخرج SQL آمن**: استخدام `INSERT OR IGNORE` + escaping صحيح لمنع SQL injection
7. **تقرير عربي** يبيّن: العدد الكلي، الناجح، المتجاهَل، التحذيرات مع رقم الصف

## طريقة الاستخدام (يكتبها السكريبت في README)

```bash
# 1. تجهيز البيانات
python prepare-legacy-data.py legacy-data.xlsx

# 2. أخذ نسخة احتياطية من قاعدة النظام (احتياط)
cp database.db database.db.backup

# 3. استيراد البيانات
sqlite3 database.db < legacy-backup.sql

# 4. تشغيل النظام والتحقق من ظهور السجلات
```

## ملاحظة مهمة حول schema
سأفحص أولاً `server/init-db.ts` و`shared/schema.ts` من أرشيف v7.1 لمعرفة:
- اسم الجدول الفعلي (`financial_forms` أو غيره)
- أسماء الأعمدة الفعلية في DB (snake_case أو camelCase)
- وجود/عدم وجود عمود `is_legacy`

إن لم يوجد `is_legacy` في schema، سيتم إضافته كـ `ALTER TABLE` في بداية ملف SQL (آمن — لا يمسح بيانات).

## ما لن يتغير
- ❌ لا تعديل على ملفات النظام v7.1
- ❌ لا تأثير على ترقيم الاستمارات الجديدة
- ❌ لا فقد لأي ميزة موجودة
- ✅ فقط إضافة سجلات قديمة كـ "أرشيف" قابل للعرض/البحث في النظام

