# 📐 وثيقة التصميم التقني — نظام مستودع الملفات (File Repository System TDD)
> **الإصدار:** 1.1 — تطبيق بنود FR-01 → FR-09 من قائمة تدقيق التوافق v1.0 | **المستوى:** 🟡 Production | **الوثائق المرجعية:** MVP Functions **v3.6** (القسمان 3.3 و 4.3.1 · القرارات #20 #21) · **Core Data Layer TDD v2.1 (المرجع الملزم للـ Schema والأمان)** · Compatibility Audit v1.0 · Notifications TDD v1.1 · Attendance TDD · Scheduling & Events TDD v1.1

## سجل التغييرات v1.0 → v1.1

| البند | التغيير |
|---|---|
| FR-01 🔴 | `institution_id` في كل الجداول والفهارس والسياسات؛ `storage_path` يبدأ إلزامياً بـ `{institution_id}/` |
| FR-02 🔴 | جدول `files` الرسمي هو تعريف Core Data v2.1 §4.2-8 (BIGINT identity · `is_contribution` · `file_name`)؛ هذه الوثيقة تضيف أعمدة امتداد فقط |
| FR-03 🟠 | اعتماد ENUM `review_status ('pending','accepted','rejected')` من Core Data (lowercase) |
| FR-04 🔴 | التوحيد على دوال Core Data: `is_enrolled()` · `teaches()` · `has_active_subscription()` — لا دوال مخترعة ولا منطق اشتراك مستقل |
| FR-05 🟠 | `profiles(id)` (UUID) و`courses(id)` (BIGINT) |
| FR-06 🔴 | حذف نمط Outbox كلياً — trigger مباشر إلى `notifications` بأنواع قاموس Notifications TDD §4.5 |
| FR-07 🟠 | نموذج الحمل: 2,000 طالب/مؤسسة × 10 = 20,000؛ إعادة تسعير الكلفة على المستوى الكلي مع حصة تخزين لكل مؤسسة |
| FR-08 🔴 | `sign-download` تفحص تطابق `institution_id` claim مع مؤسسة الملف قبل أي فحص آخر |
| FR-09 🟡 | تحديث المرجعية إلى MVP v3.6 + Core Data v2.1 |

---

## القسم 0: بروتوكول التحليل المتسلسل 🧠

### 0.1 — إعادة صياغة المشكلة

المطلوب بناء مستودع ملفات مركزي مقسّم حسب المادة الدراسية، داخل منصة **متعددة المؤسسات**، يجمع في موضع واحد ملفات المحاضر الرسمية واجتهادات الطلاب المقبولة. الطالب يتصفح ويحمّل يدوياً ويرفع اجتهاداته التي تمر بسير مراجعة (pending → accepted/rejected)، والمحاضر يرفع مباشرة ويراجع الاجتهادات. النظام يدعم استئناف التحميل المنقطع، وفتح PDF والصور دون اتصال، وإدارة التخزين المحلي. **القيد الحاكم:** العزل المؤسسي مطلق — ملف مؤسسة A لا يُرى ولا يُحمَّل من أي مستخدم في مؤسسة B، حتى عبر الروابط الموقّعة.

### 0.2 — الأسئلة الخمسة الحاسمة

1. **من المستخدم الفعلي؟** النموذج الرسمي (قرار #21): **~2,000 طالب و~80 محاضراً لكل مؤسسة × 10 مؤسسات = ~20,800 مستخدم**. ذروة الاستخدام: أول 30 دقيقة بعد نشر ملف محاضرة (إشعار جماعي → حتى 500 تحميل متزامن لنفس الملف داخل المؤسسة) — **وقد تتزامن موجات عبر عدة مؤسسات** (ليلة امتحانات مشتركة)، فميزانية النطاق تُحسب على السيناريو الكلي.
2. **قراءة أم كتابة؟** قراءة بشكل ساحق — نسبة تقريبية **200:1**. هذا يوجّه التصميم نحو CDN/التخزين الكائني وليس تمرير الملفات عبر الخادم.
3. **ماذا لو توقف النظام ساعة؟** إزعاج وليس كارثة — الملفات المحمّلة مسبقاً تعمل أوفلاين. المطلوب: availability بحدود 99.5%.
4. **أكثر البيانات حساسية؟** (أ) **العزل المؤسسي** — تسريب ملف بين مؤسستين = «أخطر فشل ممكن» (MVP §1.1). (ب) الاجتهادات المرفوضة/قيد المراجعة لا يراها إلا صاحبها والمحاضر، وملاحظات المحاضر لا يراها الطالب إطلاقاً. الخطر الأكبر: رابط تحميل يسمح لغير المخوَّلين (مؤسسة أخرى، غير مسجّل، محجوب اشتراكياً) بالوصول.
5. **أضيق عنق زجاجة خلال 6 أشهر؟** كلفة نطاق التحميل (egress): ملف 20MB × 500 طالب = 10GB لكل محاضرة منشورة — **وقد تتزامن عبر 10 مؤسسات = ~100GB في موجة واحدة**. المعالجة في [D-02] (CDN) + مراقبة 6.5 + حصة تخزين لكل مؤسسة (MVP §1.2).

### 0.3 — الافتراضات المعلنة

| # | الافتراض | مستوى الثقة | تأثيره لو كان خاطئاً |
|---|---|---|---|
| A-01 | البنية الخلفية Supabase (Postgres + Storage + RLS) — مثبَّتة بـ Core Data v2.1 | عالٍ — رسمي | — |
| A-02 | الحد الأقصى لحجم الملف: **50MB** للمحاضر (سقف Core Data)، **25MB** لاجتهاد الطالب (قيد أضيق — FR-10 متوافق) | متوسط | تعديل قيمة config واحدة |
| A-03 | الأنواع المسموحة: PDF, PNG, JPG, PPTX, DOCX, XLSX, ZIP (قائمة بيضاء صريحة) | متوسط | توسيع القائمة البيضاء فقط |
| A-04 | إجمالي التخزين المتوقع سنة أولى: ~150GB **لكل مؤسسة** وسطياً → ~1.5TB على مستوى المنصة، مع حصة تخزين لكل مؤسسة تُدار من الإعدادات (MVP §1.2) | متوسط | مراجعة خطة التكلفة — لا تغيير معماري |
| A-05 | حذف الملف من المحاضر = soft delete وليس حذفاً فيزيائياً فورياً | عالٍ | تعديل سلوك endpoint واحد |
| A-06 | حجب الاشتراك (MVP §3.7.2) يُطبَّق: الطالب المحجوب لا يحمّل ولا يرفع — **الحالة تُقرأ حصراً من `has_active_subscription()`** (FR-04) | عالٍ | إزالة شرط واحد من السياسات |
| A-07 | «العنصر» الذي تُربط به الملفات هو حدث من نظام الجدولة أو تصنيف حر يُنشئه المحاضر | متوسط | تعديل مفتاح الربط في `repo_folders` |

### 0.4 — نطاق العمل (Scope Fence)

- ✅ **داخل النطاق:** رفع/تحميل الملفات، سير مراجعة الاجتهادات، تمييز مصدر الملف (`is_contribution`)، استئناف التحميل، الفتح أوفلاين (PDF/صور)، إدارة التخزين المحلي، ملاحظات المحاضر، إنتاج أحداث `file_added` و`contribution_reviewed` لنظام الإشعارات (trigger مباشر)، تطبيق حجب الاشتراك، العزل المؤسسي الكامل.
- ❌ **خارج النطاق:** معاينة PPTX/DOCX داخل التطبيق، التحرير التعاوني، versioning، فحص الفيروسات (دين تقني §2.3)، البحث داخل المحتوى، بنية نظام الإشعارات نفسه، ضغط/تحويل الملفات على الخادم.

---

## القسم 1: الملخص التنفيذي وسياق المشكلة 📋

### 1.1 — بيان المشكلة

ملفات المواد الدراسية اليوم مبعثرة بين مجموعات واتساب وذاكرات USB وبريد إلكتروني، فيضيع على الطالب ملف المحاضرة قبل الامتحان، ويعيد المحاضر إرسال نفس الملف عشرات المرات، وتضيع اجتهادات الطلاب الجيدة لأنه لا قناة رسمية لنشرها. كلفة عدم الحل: ساعات مهدرة أسبوعياً لكل طالب، وغياب أي أرشيف مؤسسي للمحتوى التعليمي.

### 1.2 — الهدف القابل للقياس

تمكين أي طالب من الوصول إلى أي ملف من ملفات مواده المسجّل فيها **خلال أقل من 10 ثوانٍ من فتح التطبيق** (3 نقرات: مادة ← عنصر ← ملف)، مع وصول إشعار الملف الجديد إلى 100% من طلاب المادة النشطين خلال 60 ثانية من الرفع — وبعزل مؤسسي مطلق.

### 1.3 — معايير النجاح

| معيار | القيمة المستهدفة | كيف يُقاس |
|---|---|---|
| زمن بدء التحميل (توليد الرابط الموقّع) p95 | < 500ms | لوحة Supabase / logs |
| نجاح استئناف التحميل المنقطع | ≥ 95% من المحاولات | telemetry من التطبيق |
| فتح PDF محمّل مسبقاً أوفلاين | 100% نجاح، < 2s | اختبار وضع الطيران |
| زمن دورة مراجعة الاجتهاد (رفع → إشعار المحاضر) | < 60s | قياس end-to-end |
| كلفة التخزين + النطاق شهرياً | تُسعَّر على المستوى الكلي (10 مؤسسات) مع حصة لكل مؤسسة — الميزانية التفصيلية في خطة التشغيل، والعتبة التشغيلية في 6.5 (FR-07) | فاتورة المزود + لوحة استهلاك لكل مؤسسة |
| **العزل المؤسسي** | **0 تسريب — `sign-download` يرفض بـ 403 أي ملف من مؤسسة مختلفة** | اختبار تكاملي إلزامي (7.4) |

### 1.4 — ما ليس هذا النظام (Anti-Goals)

- ليس Google Drive: لا مجلدات حرة للطالب، لا مشاركة روابط خارجية، لا تحرير.
- ليس نظام تسليم واجبات: ذلك يمر عبر نظام الجدولة والتصحيح (MVP §4.3.2).
- ليس أرشيفاً بنسخ إصدارات: رفع ملف معدَّل = ملف جديد.
- ليس شبكة اجتماعية: لا إعجابات ولا تعليقات — التعليقات مكانها نظام الدردشة.

---

## القسم 2: القيود التقنية واختيار التقنيات 🔧

### 2.1 — القيود المفروضة (Hard Constraints)

| نوع القيد | التفصيل | مصدره |
|---|---|---|
| تقني | فرض العزل بين الأدوار **والمؤسسات** عبر RLS حصراً | MVP §1.1 · Core Data §4.0 |
| تقني | `institution_id` في كل جدول؛ كل فهرس يبدأ به؛ `storage_path` يبدأ بـ `{institution_id}/` | FR-01 |
| تقني | حالة الاشتراك تُقرأ حصراً من `has_active_subscription()` | MVP §3.7.2 · FR-04 |
| تقني | عمل أوفلاين للملفات المحمّلة (PDF/صور) بعد أول مزامنة | MVP §3.3 |
| تقني | العميل Flutter مع Hive للتخزين المحلي | MVP §3.1.2 |
| ميزانية | فريق صغير — تفضيل الخدمات المُدارة | مبدأ «البساطة المنضبطة» MVP §1.1 |
| تنظيمي | حجب الطالب متأخر الاشتراك عن الميزات (عدا الامتحانات) | MVP §3.7.2 / A-06 |

### 2.2 — مصفوفة اختيار التقنيات

#### [D-01] طبقة: تخزين الملفات الفعلي (Object Storage)

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **Supabase Storage ✅** | تكامل أصيل مع RLS وPostgres؛ روابط موقّعة مدمجة؛ TUS للرفع المستأنَف؛ CDN مدمج | حدود حجم أقل مرونة من S3 الخام؛ vendor lock-in؛ egress يُحتسب على الباقة | قُبل: يحقق A-01 ويلغي طبقة تفويض مستقلة — سياسة RLS واحدة تحكم الجدول والملف معاً |
| S3 مباشرة + CloudFront ❌ | الأرخص على النطاق الكبير | طبقة تفويض كاملة تُكتب وتُدار؛ يكسر توحيد الـ stack | رُفض: كلفة هندسية عالية لفريق صغير |
| Firebase Storage ❌ | قواعد أمان مرنة | يفصل الأمان عن Postgres RLS — نظامان للتفويض = مصدران للخطأ | رُفض: يخالف مبدأ RLS الموحّد |

#### [D-02] طبقة: تسليم الملفات للطالب (Download Delivery)

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **روابط موقّعة قصيرة العمر (TTL = 15 دقيقة) ✅** | التحميل مباشرة من CDN دون المرور بالخادم؛ يدعم HTTP Range للاستئناف؛ التفويض (بما فيه فحص المؤسسة — FR-08) يحدث مرة عند توليد الرابط | الرابط قابل للمشاركة خلال الـ 15 دقيقة (خطر مقبول وموثّق §6.3) | قُبل: الوحيد الذي يفصل التفويض عن نقل البيانات |
| تمرير الملف عبر API الخادم ❌ | تحكم كامل بكل بايت | الخادم عنق زجاجة أمام موجات التحميل المتزامنة عبر 10 مؤسسات | رُفض: يفشل رياضياً |
| ملفات عامة (Public bucket) ❌ | أبسط تنفيذ | يكسر عزل المواد والمؤسسات وحجب الاشتراك معاً | رُفض: مخالفة أمنية مباشرة |

#### [D-03] طبقة: آلية الرفع المستأنَف (Resumable Upload)

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **بروتوكول TUS عبر Supabase Storage ✅** | معيار مفتوح مدعوم أصلاً؛ حزمة `tus_client` لـ Flutter؛ استئناف من نقطة التوقف | تعقيد أعلى قليلاً من رفع بسيط | قُبل: شبكات الطلاب غير مستقرة |
| رفع HTTP بسيط (single PUT) ❌ | أبسط كود | انقطاع عند 90% = إعادة من الصفر | رُفض للملفات > 6MB؛ **يُستخدم كمسار سريع للملفات ≤ 6MB** |
| رفع مجزّأ مخصص ❌ | تحكم كامل | إعادة اختراع TUS | رُفض |

#### [D-04] طبقة: عارض الملفات أوفلاين في التطبيق

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **حزم Flutter محلية (pdfx + عارض الصور) + فهرس Hive ✅** | عرض كامل دون إنترنت؛ لا كلفة خادم | ذاكرة الجهاز محدودة — تعالجها شاشة إدارة التخزين (MVP §3.3) | قُبل: يحقق متطلب الأوفلاين حرفياً |
| عارض ويب مضمّن ❌ | لا حزم إضافية | يتطلب إنترنت | رُفض |
| تحويل على الخادم ❌ | تجربة موحدة | خدمة تُدار + خارج النطاق | رُفض |

### 2.3 — الديون التقنية المقبولة عمداً

| الدين | لماذا الآن | متى يُسدد |
|---|---|---|
| لا فحص فيروسات | قائمة بيضاء صارمة + التحقق من magic bytes | عند فتح أنواع تنفيذية أو نمو كبير |
| لا deduplication | التكرار مقبول بالحجم الحالي — عمود `sha256` جاهز تمهيداً | عند تجاوز كلفة التخزين الكلية عتبتها |
| الرابط الموقّع لا يُلغى قبل انتهاء TTL | حذف الملف يمنع التوليد الجديد؛ الرابط الحي يعمل ≤ 15 دقيقة | مقبول دائماً |

---

## القسم 3: معمارية النظام والمخططات 🏗️

### 3.1 — المخطط العام (System Context)

\`\`\`mermaid
graph TB
    ST[📱 تطبيق الطالب<br/>Flutter + Hive + ملفات محلية]
    LC[📱 تطبيق المحاضر<br/>Flutter]
    subgraph Supabase
        API[PostgREST API<br/>+ RLS مؤسسي]
        EF[Edge Function<br/>finalize-upload / sign-download<br/>فحص institution_id claim أولاً]
        DB[(Postgres<br/>البيانات الوصفية)]
        SO[(Storage Buckets<br/>مسارات institution_id/... + CDN)]
        NTB[(notifications<br/>نظام الإشعارات)]
    end

    ST -->|تصفح / حالة الاجتهادات| API
    ST -->|رفع TUS / تحميل برابط موقّع| SO
    ST -->|طلب رابط تحميل| EF
    LC -->|رفع / مراجعة / ملاحظات| API
    LC -->|رفع TUS| SO
    API --> DB
    EF --> DB
    EF --> SO
    DB -->|trigger مباشر داخل نفس الـ transaction:<br/>file_added / contribution_reviewed| NTB
\`\`\`

### 3.2 — جدول المكونات والمسؤوليات

| المكون | مسؤوليته الوحيدة | ماذا لو سقط؟ | استراتيجية التعافي |
|---|---|---|---|
| Postgres (بيانات وصفية) | سجل الملفات والاجتهادات والملاحظات + فرض RLS المؤسسي | لا تصفح ولا رفع؛ الملفات المحمّلة تعمل أوفلاين | نسخ احتياطي يومي مُدار؛ RPO ساعة واحدة |
| Storage + CDN | حفظ وتسليم البايتات | لا تحميل جديد؛ التصفح يعمل | retry من العميل مع backoff |
| Edge Function `sign-download` | فحص المؤسسة (FR-08) ثم الأهلية (تسجيل + اشتراك) ثم توليد رابط موقّع | لا تحميلات جديدة | stateless — إعادة نشر فورية |
| Edge Function `finalize-upload` | التحقق من اكتمال الرفع + magic bytes + مطابقة بادئة `storage_path` للمؤسسة + إنشاء سجل الملف | ملفات مرفوعة بلا سجل (يتيمة) | مهمة تنظيف دورية تحذف اليتيم > 24 ساعة |
| Trigger الإشعارات | إنتاج `file_added` / `contribution_reviewed` داخل نفس الـ transaction | لا إشعارات جديدة — الملفات سليمة | ملفوف بـ `EXCEPTION WHEN OTHERS` — لا يُفشل العملية الأصلية أبداً |
| فهرس Hive المحلي | تتبّع الملفات المحمّلة | فقدان الفهرس = إعادة تحميل — لا فقدان بيانات | إعادة بناء بمسح مجلد التطبيق |

### 3.3 — تدفق البيانات لأهم عمليتين (Critical Path)

**العملية 1: رفع اجتهاد طالب حتى نشره**

\`\`\`mermaid
sequenceDiagram
    participant S as الطالب
    participant SO as Storage (TUS)
    participant EF as finalize-upload
    participant DB as Postgres
    participant L as المحاضر
    S->>SO: رفع مستأنَف (chunks) إلى مسار {institution_id}/...
    Note over S,SO: نقطة فشل: انقطاع الشبكة → استئناف من آخر chunk
    SO-->>S: اكتمل الرفع (object path)
    S->>EF: finalize(object_path, course_id, folder_id)
    EF->>EF: فحص: بادئة storage_path = institution_id من الـ claim
    EF->>SO: قراءة أول 512 بايت (magic bytes)
    Note over EF: نقطة فشل: النوع لا يطابق الامتداد → حذف الكائن + رفض
    EF->>DB: INSERT file(is_contribution=true, review_status='pending')
    Note over DB: trigger → notifications: إشعار المحاضر «اجتهاد جديد للمراجعة»
    L->>DB: UPDATE review_status='accepted'
    Note over DB: trigger → notifications: file_added لطلاب المادة ACTIVE<br/>+ contribution_reviewed لصاحب الاجتهاد
\`\`\`

**العملية 2: تحميل ملف مع الاستئناف**

\`\`\`mermaid
sequenceDiagram
    participant S as الطالب
    participant EF as sign-download
    participant DB as Postgres
    participant CDN as Storage/CDN
    S->>EF: طلب تحميل file_id
    EF->>DB: فحص 1 (FR-08): institution_id claim = مؤسسة الملف؟
    Note over EF,DB: مؤسسة مختلفة → 403 فوراً قبل أي فحص آخر
    EF->>DB: فحص 2: is_enrolled(course_id)? has_active_subscription()? الملف منشور؟
    Note over EF,DB: محجوب اشتراكياً → 403 SUBSCRIPTION_BLOCKED
    EF-->>S: رابط موقّع (TTL 15min)
    S->>CDN: GET مع Range: bytes=0-
    Note over S,CDN: انقطاع عند 60% → GET جديد بـ Range: bytes=N-
    CDN-->>S: 206 Partial Content
    S->>S: حفظ محلي + تسجيل في فهرس Hive
\`\`\`

### 3.4 — قرارات معمارية جوهرية

#### [D-05] ملفات المحاضر والاجتهادات: جدول واحد أم جدولان؟

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **جدول واحد `files` (تعريف Core Data) بعمود `is_contribution` وعمود `review_status` ✅** | «الاجتهاد المقبول يُنشر في مستودع المادة» (MVP §3.3)؛ استعلام تصفح واحد؛ RLS واحدة | صفوف المحاضر تحمل `review_status` يُملأ accepted تلقائياً | قُبل: يطابق النموذج الذهني ويلغي JOIN في أهم مسار قراءة — **والجدول موجود أصلاً في Core Data (FR-02)** |
| جدولان منفصلان + نسخ عند القبول ❌ | فصل نظيف | ازدواجية ومزامنة | رُفض |
| جدولان + UNION VIEW ❌ | لا نسخ | تعقيد RLS على الـ VIEW | رُفض |

#### [D-06] فرض حجب الاشتراك والعزل المؤسسي: أين؟

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **في Edge Function `sign-download` (المؤسسة أولاً ثم الاشتراك) + سياسات RLS على القراءة والرفع ✅** | نقطة فرض واحدة لكل اتجاه؛ رسائل خطأ واضحة | منطق الحجب في مكانين (function + policy) | قُبل: التحميل لا يمر عبر RLS أصلاً (رابط موقّع) فلا بديل عن الـ function — **وفحص المؤسسة فيها إلزامي (FR-08)** |
| في واجهة التطبيق فقط ❌ | صفر كود خلفي | يُتجاوز بتعديل العميل | رُفض |

---

## القسم 4: نماذج البيانات وتصميم قاعدة البيانات 🗄️

> **قاعدة حاكمة (FR-02):** جدول `files` الرسمي هو تعريف **Core Data v2.1 §4.2-8** (BIGINT identity · `institution_id` · `course_id BIGINT` · `uploader_id → profiles` · `file_name` · `is_contribution BOOLEAN` · `review_status` ENUM · `storage_path` · `size_bytes` · `created_at` · `deleted_at`). هذه الوثيقة تضيف **أعمدة امتداد فقط** عبر migration. اشتقاق المصدر: `is_contribution = false` → ملف محاضر.

### 4.1 — مخطط الكيانات (ERD)

\`\`\`mermaid
erDiagram
    INSTITUTIONS ||--o{ REPO_FOLDERS : owns
    INSTITUTIONS ||--o{ FILES : owns
    COURSES ||--o{ REPO_FOLDERS : contains
    REPO_FOLDERS ||--o{ FILES : groups
    COURSES ||--o{ FILES : scoped_to
    PROFILES ||--o{ FILES : uploaded_by
    PROFILES ||--o{ FILE_NOTES : writes
    FILES ||--o{ FILE_NOTES : annotated_by
    FILES {
        bigint id PK
        uuid institution_id FK
        bigint course_id FK
        bigint folder_id FK "امتداد"
        uuid uploader_id FK "profiles(id)"
        boolean is_contribution "false = ملف محاضر"
        review_status review_status "ENUM: pending | accepted | rejected"
    }
\`\`\`

### 4.2 — تعريف الجداول (Schema) — يحل محل §4.2 في v1.0 (نص التدقيق §4.1 حرفياً)

\`\`\`sql
-- repo_folders: «العنصر» في مسار التصفح (مادة ← عنصر ← ملفات) — MVP §3.3
-- جدول جديد يُضاف كامتداد رسمي لـ Core Data (migration إضافية)
CREATE TABLE repo_folders (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),    -- FR-01
    course_id      BIGINT NOT NULL REFERENCES courses(id),       -- FR-05
    event_id       BIGINT REFERENCES events(id),                 -- ربط اختياري بحدث جدولة (A-07)
    title          TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 120),
    sort_order     INT  NOT NULL DEFAULT 0,
    created_by     UUID NOT NULL REFERENCES profiles(id),        -- FR-05
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ                                   -- soft delete (A-05)
);
CREATE INDEX idx_repo_folders_tenant
    ON repo_folders (institution_id, course_id) WHERE deleted_at IS NULL;

-- جدول files الرسمي هو تعريف Core Data v2.1 §4.2-8 — هذه أعمدة الامتداد فقط (FR-02):
ALTER TABLE files ADD COLUMN folder_id   BIGINT REFERENCES repo_folders(id);  -- NULL = جذر المادة
ALTER TABLE files ADD COLUMN mime_type   TEXT;                    -- بعد التحقق من magic bytes
ALTER TABLE files ADD COLUMN sha256      TEXT;                    -- تمهيد الـ dedup المستقبلي (§2.3)
ALTER TABLE files ADD COLUMN reviewed_at TIMESTAMPTZ;
ALTER TABLE files ADD COLUMN reject_reason TEXT;                  -- اختياري عند الرفض (MVP §3.3)
ALTER TABLE files ADD COLUMN reviewed_by UUID REFERENCES profiles(id);
CREATE UNIQUE INDEX uq_files_storage_path ON files (storage_path); -- idempotency للـ finalize

-- قاعدة مسار التخزين (FR-01): storage_path يبدأ إلزامياً بـ {institution_id}/
ALTER TABLE files ADD CONSTRAINT storage_path_tenant_prefix
    CHECK (storage_path LIKE (institution_id::text || '/%'));

-- trigger: ملف المحاضر يُنشر فوراً؛ اجتهاد الطالب يبدأ pending دائماً (ENUM Core Data — FR-03)
CREATE OR REPLACE FUNCTION enforce_review_status() RETURNS trigger AS $$
BEGIN
    IF NEW.is_contribution = false THEN
        NEW.review_status := 'accepted';
        NEW.reviewed_at   := now();
    ELSE
        NEW.review_status := 'pending';   -- لا يستطيع الطالب حقن accepted
    END IF;
    RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_files_review BEFORE INSERT ON files
FOR EACH ROW EXECUTE FUNCTION enforce_review_status();

-- file_notes: ملاحظات المحاضر — لا يراها الطالب إطلاقاً (MVP §4.3.1)
CREATE TABLE file_notes (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),    -- FR-01
    file_id        BIGINT NOT NULL REFERENCES files(id),
    author_id      UUID NOT NULL REFERENCES profiles(id),        -- محاضر حصراً (RLS)
    body           TEXT NOT NULL CHECK (char_length(body) BETWEEN 1 AND 2000),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_file_notes_tenant ON file_notes (institution_id, file_id);
\`\`\`

**سياسات RLS الموحّدة (بدوال Core Data حصراً + فحص المؤسسة أولاً — FR-01/FR-04):**

\`\`\`sql
ALTER TABLE files ENABLE ROW LEVEL SECURITY;
ALTER TABLE repo_folders ENABLE ROW LEVEL SECURITY;
ALTER TABLE file_notes ENABLE ROW LEVEL SECURITY;

CREATE POLICY files_student_read ON files FOR SELECT USING (
    institution_id = my_institution()
    AND deleted_at IS NULL
    AND (
        (is_contribution = false OR review_status = 'accepted')
        AND is_enrolled(course_id)
        OR uploader_id = auth.uid()
    )
);
CREATE POLICY files_lecturer_read ON files FOR SELECT USING (
    institution_id = my_institution() AND teaches(course_id)
);
CREATE POLICY files_upload ON files FOR INSERT WITH CHECK (
    institution_id = my_institution()
    AND uploader_id = auth.uid()
    AND (
        teaches(course_id)
        OR (is_enrolled(course_id) AND has_active_subscription())
    )
);
CREATE POLICY files_review ON files FOR UPDATE USING (
    institution_id = my_institution() AND teaches(course_id)
);

CREATE POLICY folders_read ON repo_folders FOR SELECT USING (
    institution_id = my_institution() AND deleted_at IS NULL
    AND (is_enrolled(course_id) OR teaches(course_id))
);
CREATE POLICY folders_write ON repo_folders FOR INSERT WITH CHECK (
    institution_id = my_institution() AND teaches(course_id) AND created_by = auth.uid()
);

-- file_notes: محاضرو المادة حصراً — لا سياسة قراءة للطالب أصلاً
CREATE POLICY notes_lecturer ON file_notes FOR ALL USING (
    institution_id = my_institution()
    AND EXISTS (SELECT 1 FROM files f WHERE f.id = file_notes.file_id AND teaches(f.course_id))
);
\`\`\`

**تكامل الإشعارات (FR-06 — يحل محل نمط Outbox الملغى):**

\`\`\`sql
-- trigger مباشر إلى notifications داخل نفس الـ transaction — [D-06] في Notifications TDD
-- الأنواع من قاموس Notifications TDD §4.5:
--   'file_added'            → طلاب المادة حيث enrollments.status = 'ACTIVE' (عند نشر ملف محاضر أو قبول اجتهاد)
--   'contribution_reviewed' → صاحب الاجتهاد (عند accepted أو rejected)
-- النمط الحامي الإلزامي: فشل الإشعار لا يُفشل العملية الأصلية أبداً
CREATE OR REPLACE FUNCTION notify_on_file_change() RETURNS trigger AS $$
BEGIN
    BEGIN
        IF (TG_OP = 'INSERT' AND NEW.is_contribution = false)
           OR (TG_OP = 'UPDATE' AND OLD.review_status = 'pending' AND NEW.review_status = 'accepted') THEN
            INSERT INTO notifications (user_id, type, ref_type, ref_id, course_id, title, body)
            SELECT e.student_id, 'file_added', 'file', NEW.id::text, NEW.course_id,
                   'ملف جديد', 'ملف جديد في مستودع المادة'
            FROM enrollments e
            WHERE e.course_id = NEW.course_id AND e.status = 'ACTIVE'
            ON CONFLICT (user_id, type, ref_id) DO NOTHING;
        END IF;

        IF TG_OP = 'UPDATE' AND OLD.review_status = 'pending'
           AND NEW.review_status IN ('accepted','rejected') THEN
            INSERT INTO notifications (user_id, type, ref_type, ref_id, course_id, title, body)
            VALUES (NEW.uploader_id, 'contribution_reviewed', 'contribution',
                    NEW.id::text, NEW.course_id, 'نتيجة المراجعة',
                    CASE WHEN NEW.review_status = 'accepted'
                         THEN 'اجتهادك: مقبول ✅' ELSE 'اجتهادك: مرفوض' END)
            ON CONFLICT (user_id, type, ref_id) DO NOTHING;
        END IF;
    EXCEPTION WHEN OTHERS THEN
        RAISE WARNING 'notify_on_file_change failed: %', SQLERRM;
    END;
    RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_notify_on_file_change AFTER INSERT OR UPDATE ON files
FOR EACH ROW EXECUTE FUNCTION notify_on_file_change();
\`\`\`

### 4.3 — استراتيجية الفهرسة (كل فهرس يبدأ بـ institution_id — FR-01)

| الفهرس | الاستعلام الذي يخدمه | تكلفته على الكتابة |
|---|---|---|
| `files (institution_id, course_id, folder_id, created_at DESC) WHERE deleted_at IS NULL AND review_status='accepted'` (partial) | التصفح — 90% من قراءات النظام | ضئيلة: partial index يستثني المرفوض/المحذوف |
| `files (institution_id, uploader_id, created_at DESC) WHERE is_contribution` (partial) | «سجل اجتهاداتي وحالاتها» للطالب | ضئيلة |
| `files (institution_id, course_id) WHERE review_status='pending'` (partial) | قائمة «قيد المراجعة» للمحاضر | شبه معدومة — الجدول الجزئي صغير دائماً |
| `file_notes (institution_id, file_id)` | عرض ملاحظات المحاضر على ملف | ضئيلة |
| UNIQUE على `storage_path` | idempotency للـ finalize | إلزامي — يحمي التكامل |

### 4.4 — أسئلة يجب الإجابة عنها صراحة

- **الترحيل (Migrations):** أعمدة الامتداد nullable تُضاف بدون downtime؛ قيود CHECK الجديدة بنمط `NOT VALID` ثم `VALIDATE`.
- **الحذف:** soft delete للسجلات (`deleted_at`)؛ الكائن الفيزيائي يُحذف بمهمة تنظيف بعد 30 يوماً.
- **التوسع:** بحجم 10 مؤسسات (~400K ملف/سنة كلياً) لا حاجة لـ partitioning قبل سنوات؛ إن لزم فعلى `institution_id` بالـ hash.
- **التوافقية:** eventual مقبولة في: عدّاد الملفات الجديدة وإشعارات النشر (≤ 60s). مرفوضة في: حالة الاجتهاد وسياسات الوصول (لحظية بطبيعة RLS).

---

## القسم 5: عقود الـ API والواجهات 🔌

### 5.1 — مبادئ التصميم

**[D-07]** PostgREST للقراءة والعمليات البسيطة المحكومة بـ RLS + **Edge Functions** حصراً للعمليات المركّبة (`finalize-upload`, `sign-download`). النسخ عبر مسار `/v1/`. صيغة الخطأ الموحدة: `{"error": {"code": "STRING_CODE", "message": "...", "details": {}}}`. **`institution_id` يُشتق من JWT claim حصراً — لا يُقبل من العميل في أي عقد.**

### 5.2 — توثيق الـ Endpoints

#### `POST /functions/v1/sign-download`
**الغرض:** فحص المؤسسة ثم الأهلية وتوليد رابط تحميل موقّع | **المصادقة:** JWT (طالب/محاضر) | **Rate Limit:** 30 طلباً/دقيقة/مستخدم

**المدخلات:**
\`\`\`json
{
  "file_id": "bigint — مطلوب"
}
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{
  "url": "https://...signed-url...",
  "expires_at": "ISO-8601 (بعد 15 دقيقة)",
  "size_bytes": 8388608,
  "sha256": "hex — للتحقق من سلامة الملف بعد التحميل"
}
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | `file_id` غير صالح | رسالة تحقق |
| 401 | توكن منتهٍ | تحديث الجلسة / إعادة الدخول |
| 403 `INSTITUTION_MISMATCH` | **الملف من مؤسسة مختلفة عن claim الطالب (FR-08) — يُفحص قبل أي شيء آخر** | إزالة من القائمة المحلية + تسجيل telemetry (لا يحدث في الاستخدام السليم) |
| 403 `NOT_ENROLLED` | غير مسجّل بالمادة (`is_enrolled()` = false) | إخفاء الملف من الواجهة |
| 403 `SUBSCRIPTION_BLOCKED` | `has_active_subscription()` = false (A-06) | شاشة «سدّد اشتراكك» مع رابط الدفعات |
| 404 | الملف محذوف أو غير منشور (pending/rejected لغير صاحبه) | إزالة من القائمة المحلية |
| 429 | تجاوز 30 طلباً/دقيقة | retry مع exponential backoff |
| 500 | فشل توليد التوقيع | retry مرة واحدة ثم رسالة خطأ |

#### `POST /functions/v1/finalize-upload`
**الغرض:** إغلاق عملية رفع TUS مكتملة وإنشاء سجل الملف | **المصادقة:** JWT | **Rate Limit:** 10 طلبات/دقيقة/مستخدم

**المدخلات:**
\`\`\`json
{
  "storage_path": "string — مسار الكائن المرفوع، يجب أن يبدأ بـ {institution_id}/ المطابق للـ claim",
  "course_id": "bigint — مطلوب",
  "folder_id": "bigint — اختياري (NULL = جذر المادة)",
  "file_name": "string — مطلوب، 1-200 حرف",
  "sha256": "hex string — محسوب على العميل، مطلوب"
}
\`\`\`

**المخرجات الناجحة (201):**
\`\`\`json
{
  "file_id": "bigint",
  "review_status": "accepted إن كان الرافع محاضراً، pending إن كان طالباً",
  "created_at": "ISO-8601"
}
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 `INVALID_INPUT` | حقل ناقص أو اسم خارج الحدود | رسالة تحقق |
| 401 | توكن منتهٍ | إعادة الدخول |
| 403 `INSTITUTION_MISMATCH` | بادئة `storage_path` لا تطابق `institution_id` claim — **يُحذف الكائن فوراً** | telemetry — لا يحدث في الاستخدام السليم |
| 403 `NOT_AUTHORIZED` | ليس `teaches()` ولا `is_enrolled()` | إخفاء زر الرفع |
| 403 `SUBSCRIPTION_BLOCKED` | طالب و`has_active_subscription()` = false | شاشة السداد |
| 409 `ALREADY_FINALIZED` | نفس `storage_path` سُجّل سابقاً | اعتبارها نجاحاً — idempotent |
| 413 `FILE_TOO_LARGE` | > 50MB محاضر / > 25MB طالب | رسالة بالحد المسموح |
| 415 `TYPE_MISMATCH` | magic bytes لا تطابق الامتداد أو نوع خارج القائمة البيضاء — **يُحذف الكائن فوراً** | «نوع الملف غير مدعوم» |
| 422 `HASH_MISMATCH` | sha256 المرسل ≠ المحسوب على الخادم | إعادة الرفع |
| 429 | تجاوز الحد | backoff |
| 500 | خطأ داخلي | الكائن يبقى — retry آمن (idempotent عبر 409) |

#### `PATCH /rest/v1/files?id=eq.{file_id}` — مراجعة اجتهاد
**الغرض:** قبول/رفض اجتهاد طالب (محكوم بسياسة RLS `files_review`) | **المصادقة:** JWT محاضر المادة | **Rate Limit:** 60/دقيقة

**المدخلات:**
\`\`\`json
{
  "review_status": "accepted | rejected — مطلوب (ENUM lowercase — FR-03)",
  "reject_reason": "string ≤ 500 حرف — اختياري، له معنى عند rejected فقط"
}
\`\`\`

**حالات الخطأ:** 400 (حالة غير صالحة) · 401 · 403 (RLS: ليس محاضر المادة أو مؤسسة أخرى → صفر صفوف متأثرة، يعامَل كـ 404) · 409 (الملف محذوف) · 429 · 500. القبول يُطلق trigger `notify_on_file_change` مباشرة (لا outbox — FR-06).

**بقية العمليات** (تصفح المجلدات والملفات، سجل اجتهادات الطالب، قائمة pending للمحاضر، `file_notes`, soft delete) تتم عبر PostgREST مباشرة وتحكمها سياسات RLS في §4.2 حرفياً.

### 5.3 — العقود بين الخدمات الداخلية — مُصحَّح FR-06

| العقد | الصيغة | الضمانة |
|---|---|---|
| مستودع الملفات → نظام الإشعارات | **trigger مباشر إلى `notifications` داخل نفس الـ transaction** بأنواع القاموس (`file_added` / `contribution_reviewed`) ملفوفاً بـ `EXCEPTION WHEN OTHERS` | **exactly-once للكتابة** (ذرّية transaction + قيد `uq_notification_dedup` في نظام الإشعارات)؛ فشل الإشعار لا يُفشل العملية الأصلية أبداً |

> نمط Outbox (`notification_outbox` + حدث `FILE_PUBLISHED`) الوارد في v1.0 **ملغى كلياً** — [D-06] في Notifications TDD هو القرار الحاكم لقناة التسليم.

---

## القسم 6: حالات الحافة، أنماط الفشل، والأمان 🛡️

### 6.1 — جرد حالات الحافة

| السيناريو | ماذا يحدث حالياً في التصميم؟ | المعالجة |
|---|---|---|
| طلبان متزامنان يعدلان نفس المورد (محاضران يراجعان نفس الاجتهاد) | UPDATE على صف واحد — آخر كتابة تفوز | مقبول عمداً: `reviewed_by/reviewed_at` يوثّقان مَن حسم أخيراً |
| المستخدم يرسل مدخلات بحجم 10x المتوقع | حجم الملف يُرفض عند TUS (حد الـ bucket) وعند finalize (413)؛ الحقول النصية CHECK | دفاع مزدوج: حد Storage + قيد DB |
| انقطاع الشبكة في منتصف رفع | TUS يستأنف من آخر chunk؛ إن لم يُستدعَ finalize يبقى الكائن يتيماً | idempotency عبر UNIQUE(storage_path) + تنظيف اليتيم > 24 ساعة |
| انقطاع في منتصف التحميل | HTTP Range يستأنف؛ إن انتهى الرابط (> 15 دقيقة) | العميل يطلب `sign-download` جديداً ويكمل بنفس Range |
| خدمة خارجية بطيئة (Storage بطيء أثناء finalize) | قراءة magic bytes قد تعلّق | timeout = 5s → 500 و retry آمن |
| trigger الإشعارات يفشل | خطر إفشال نشر الملف | `EXCEPTION WHEN OTHERS` + `RAISE WARNING` — لا يُفشل العملية أبداً؛ المزامنة عند الفتح في نظام الإشعارات شبكة الأمان |
| الساعة/التوقيت | `expires_at` يُحسب على الخادم حصراً | clock skew بلا أثر؛ كل التوقيتات TIMESTAMPTZ |
| طالب رفع اجتهاداً ثم أُلغي تسجيله قبل المراجعة | `uploader_id = auth.uid()` يُبقي له رؤية اجتهاده | يظل قابلاً للمراجعة؛ إن قُبل يظهر للطلاب النشطين فقط |
| المحاضر يحذف مجلداً يحوي ملفات | soft delete على المجلد | الملفات تبقى وتُعرض في جذر المادة |
| امتلاء ذاكرة جهاز الطالب أثناء التحميل | فشل كتابة محلية | فحص المساحة قبل البدء + شاشة إدارة التخزين (MVP §3.3) |
| محاولة finalize بمسار تخزين من مؤسسة أخرى | تلاعب محتمل بالعميل | فحص بادئة `storage_path` مقابل claim في الدالة + قيد `storage_path_tenant_prefix` في DB — دفاع مزدوج |

### 6.2 — تحليل أنماط الفشل

| المكون | نمط الفشل | الاحتمالية | الأثر | الكشف | التعافي |
|---|---|---|---|---|---|
| Postgres | تعطل كامل | منخفضة | 🔴 لا تصفح/رفع | health check المزود | failover مُدار < 60s؛ الأوفلاين يغطي القراءة المحلية |
| Storage/CDN | تعطل التسليم | منخفضة | 🟠 لا تحميلات جديدة | ارتفاع أخطاء 5xx (telemetry) | retry + backoff؛ الملفات المحمّلة تعمل |
| finalize-upload | فشل بعد الرفع وقبل السجل | متوسطة | 🟡 كائن يتيم + إحباط مستخدم | مهمة التنظيف تحصي اليتامى | retry آمن (idempotent)؛ تنبيه إن تجاوز اليتامى 50/يوم |
| trigger الإشعارات | فشل صامت متكرر | منخفضة | 🟡 ملفات تُنشر بلا إشعار | عدّاد `RAISE WARNING` في اللوجات | إصلاح + المزامنة عند الفتح تعوّض |
| مهمة التنظيف | لا تعمل | متوسطة | 🟢 تضخم تخزين بطيء | metric: عدد الكائنات بلا سجل | تشغيل يدوي؛ لا فقدان بيانات |
| فهرس Hive | تلف محلي | منخفضة | 🟢 يخص جهازاً واحداً | فشل فتح الصندوق | إعادة بناء بمسح مجلد الملفات |

### 6.3 — نموذج التهديدات (STRIDE)

| التهديد | مثال ملموس على هذا النظام | الدفاع المحدد |
|---|---|---|
| Spoofing | طالب ينتحل هوية محاضر ليرفع ملفاً «رسمياً» منشوراً فوراً | `is_contribution` و`review_status` يفرضهما trigger من دور الحساب في DB — لا يُقبلان من العميل |
| Tampering | تعديل جسم finalize لتسجيل ملف على مادة أو مؤسسة أخرى | سياسة RLS `files_upload` + فحص بادئة المسار + قيد DB |
| Repudiation | محاضر ينكر رفض اجتهاد | `reviewed_by/reviewed_at` + سجل تدقيق على تغييرات `review_status` |
| Information Disclosure — **الأخطر** | مستخدم من مؤسسة B يحصل على رابط موقّع لملف مؤسسة A | فحص `INSTITUTION_MISMATCH` أول فحص في `sign-download` (FR-08) + RLS تبدأ بـ `my_institution()` + بادئة المسار المؤسسية |
| Information Disclosure | مشاركة رابط موقّع خارج المنصة | TTL = 15 دقيقة + الرابط لملف واحد؛ خطر متبقٍ مقبول وموثّق [D-02] |
| Information Disclosure | طالب يقرأ ملاحظات المحاضر | RLS على `file_notes`: لا سياسة قراءة للطالب أصلاً |
| Denial of Service | سيل طلبات sign-download لاستنزاف النطاق | rate limit 30/دقيقة + التحميل من CDN لا من الخادم |
| DoS (تخزيني) | طالب يرفع مئات الاجتهادات الضخمة | حد 25MB + حصة **20 اجتهاداً pending كحد أقصى** لكل طالب لكل مادة |
| Elevation of Privilege | طالب يستدعي PATCH review مباشرة | سياسة `files_review` تتطلب `teaches()` — صفر صفوف متأثرة |

### 6.4 — قائمة تدقيق أمنية إلزامية

- [x] المصادقة والتفويض: JWT (Google OAuth) + RLS على كل الجداول؛ Edge Functions تتحقق بدوال Core Data نفسها (`is_enrolled` / `teaches` / `has_active_subscription`)
- [x] **العزل المؤسسي:** `institution_id` من الـ claim حصراً؛ فحصه أول خطوة في `sign-download` و`finalize-upload`؛ بادئة المسار المؤسسية مفروضة بقيد DB
- [x] التشفير: at-rest مُدار (AES-256)؛ in-transit TLS 1.2+ بما فيها روابط CDN
- [x] الأسرار: مفاتيح service-role في أسرار Edge Functions حصراً
- [x] الحقن: PostgREST parameterized بطبيعته؛ لا SQL خام إلا عبر معاملات مربوطة
- [x] سجلات التدقيق: تغييرات review_status وحذف الملفات (مَن/متى)؛ **لا يُسجَّل** محتوى الملفات ولا ملاحظات المحاضر في اللوجات (PII)

### 6.5 — الملاحظة والمراقبة (Observability)

| المقياس | عتبة التنبيه |
|---|---|
| p95 لزمن sign-download | > 800ms لخمس دقائق متواصلة |
| نسبة 403 SUBSCRIPTION_BLOCKED من إجمالي الطلبات | > 20% (قد تعني خللاً في نظام الدفعات) |
| **عدد 403 INSTITUTION_MISMATCH** | **> 0 → تحقيق فوري 🔴 (محاولة اختراق عزل أو خلل عميل)** |
| كائنات يتيمة مكتشفة يومياً | > 50 |
| نسبة 415 TYPE_MISMATCH | > 5% من عمليات finalize |
| استهلاك النطاق الشهري (egress) الكلي | > 80% من سقف الباقة |
| استهلاك تخزين مؤسسة واحدة | > 90% من حصتها (MVP §1.2) |

---

## القسم 7: خطة التنفيذ وخارطة الطريق 🗺️

### 7.1 — ترتيب المخاطر (Risk-First Ordering)

أخطر افتراض تقني: **أن رفع TUS المستأنَف وتحميل HTTP Range يعملان بموثوقية على شبكات جوال متقطعة من Flutter عبر Supabase Storage** — إن فشل هذا، يسقط [D-02] و[D-03] معاً. لذلك هو موضوع المرحلة 0.

### 7.2 — المراحل

#### المرحلة 0: إثبات الجدوى (Spike) — 3 أيام
- **الهدف:** التحقق من الاستئناف (رفعاً وتحميلاً) على شبكة متقطعة
- **المخرج القابل للاختبار:** تطبيق Flutter تجريبي يرفع ملف 25MB بقطع الشبكة مرتين عمداً، ويحمّل ملف 30MB بنفس الشرط، مع تحقق sha256 نهائي
- **معيار النجاح/الفشل:** ≥ 95% نجاح عبر 40 محاولة؛ **إن فشل:** رفع مجزّأ مخصص عبر Edge Function (نقبل التعقيد المرفوض في [D-03])

#### المرحلة 1: نواة البيانات والأمان — 4 أيام
- **المهام:** (1) migrations §4.2 (repo_folders + امتداد files + قيد بادئة المسار) مع triggers (2) سياسات RLS الموحّدة بدوال Core Data (3) الفهارس المؤسسية §4.3
- **الاعتماديات:** Core Data v2.1 مطبَّقة (`files` الأساسي، `profiles`، `institutions`، الدوال المساعدة)
- **المخرج القابل للاختبار:** استعلامات SQL بأدوار ومؤسسات مختلفة تثبت العزل (طالب لا يرى pending غيره، لا يرى file_notes، **مستخدم مؤسسة B لا يرى أي صف لمؤسسة A**)
- **اختبارات المرحلة:** pgTAP عدائية بنمط **دور × عملية × مؤسسة** (النمط الإلزامي في Core Data §1.3)

#### المرحلة 2: مسار الرفع والنشر — 5 أيام
- **المهام:** (1) `finalize-upload` مع magic bytes وحدود الحجم وفحص المؤسسة (2) تكامل TUS في العميل (3) واجهة رفع المحاضر (4) واجهة رفع اجتهاد الطالب + سجل الحالات (5) مهمة تنظيف اليتامى
- **الاعتماديات:** المرحلتان 0 و 1
- **المخرج القابل للاختبار:** محاضر يرفع فيُنشر فوراً؛ طالب يرفع فيظهر pending له وللمحاضر فقط
- **اختبارات المرحلة:** integration على كل أكواد الخطأ (403 INSTITUTION_MISMATCH, 409, 413, 415, 422)

#### المرحلة 3: مسار التحميل والأوفلاين — 5 أيام
- **المهام:** (1) `sign-download` بترتيب الفحوص: مؤسسة ← تسجيل ← اشتراك (2) مدير تحميل بالاستئناف + فهرس Hive (3) عارض PDF/صور أوفلاين (4) شاشة إدارة التخزين المحلي
- **الاعتماديات:** المرحلة 2
- **المخرج القابل للاختبار:** تحميل ملف، وضع الطيران، فتحه بنجاح؛ طالب محجوب يرى شاشة السداد؛ **file_id من مؤسسة أخرى يُرفض بـ 403**
- **اختبارات المرحلة:** unit لمنطق Range/استئناف + اختبار وضع الطيران + اختبار العزل المؤسسي على sign-download

#### المرحلة 4: المراجعة والإشعارات والملاحظات — 4 أيام
- **المهام:** (1) واجهة مراجعة المحاضر (قبول/رفض مع سبب) (2) trigger `notify_on_file_change` المباشر (FR-06) مع نمط `EXCEPTION` (3) file_notes للمحاضر (4) تصفح المجلدات وترتيبها
- **الاعتماديات:** المرحلة 2 + جاهزية جداول نظام الإشعارات (Notifications TDD v1.1 المرحلة 1)
- **المخرج القابل للاختبار:** قبول اجتهاد → يظهر لطلاب المادة النشطين مع تمييز «اجتهاد طالب» + إشعار خلال 60s
- **اختبارات المرحلة:** integration لسير المراجعة الكامل + «فشل الإشعار لا يُفشل النشر» + dedup القيد الفريد

### 7.3 — مخطط الاعتماديات

\`\`\`mermaid
graph LR
    P0[مرحلة 0: Spike الاستئناف] --> P2[مرحلة 2: الرفع والنشر]
    P1[مرحلة 1: البيانات و RLS المؤسسي] --> P2
    P2 --> P3[مرحلة 3: التحميل والأوفلاين]
    P2 --> P4[مرحلة 4: المراجعة والإشعارات]
\`\`\`

### 7.4 — تعريف «الانتهاء» (Definition of Done)

- [ ] كل الاختبارات تمر (تغطية ≥ 85% لمسارات finalize / sign-download / RLS)
- [ ] اختبارات pgTAP العدائية تمر 100% بنمط **دور × عملية × مؤسسة**
- [ ] **اختبار تكاملي إلزامي: `sign-download` يرفض بـ 403 أي `file_id` من مؤسسة مختلفة عن claim الطالب** (من تعريف انتهاء التدقيق)
- [ ] توثيق الـ API محدّث بكل أكواد الخطأ في §5.2
- [ ] مقاييس §6.5 السبعة تعمل والتنبيهات مضبوطة
- [ ] قائمة الأمان §6.4 مراجَعة بالكامل
- [ ] خطة rollback: تعطيل الرفع بـ feature flag مع بقاء التصفح والتحميل — موثقة ومجربة

---

*— نهاية الوثيقة — File Repository System TDD v1.1 —*