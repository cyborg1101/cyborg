# 🗄️ وثيقة التصميم التقني — طبقة البيانات الأساسية (Core Data Layer)
## منصة Academic Hub — الوثيقة الموحّدة

---

## بيانات ضبط الوثيقة

| البند | التفاصيل |
|---|---|
| **اسم الوثيقة** | التصميم التقني لطبقة البيانات الأساسية (Core Data Layer TDD) — موحّدة |
| **رقم الإصدار** | v2.1 |
| **مستوى العمق** | 🟡 Production |
| **مصادر الدمج** | TDD Core Data Layer v2.0 (الأساس) + تعديل v1.6 (Multi-tenancy + Enrollment) |
| **الوثائق المرجعية** | MVP Functions v3.4 · Attendance TDD · قالب TDD v1.0 |
| **حالة الوثيقة** | جاهزة للمراجعة الهندسية |

### سجل حسم التعارضات بين المصدرين

| البند المتعارض | v2.0 قالت | تعديل v1.6 قال | القرار في v2.1 والمبرر |
|---|---|---|---|
| نطاق المؤسسات | جامعة واحدة، `institution_id` خارج النطاق | 10 مؤسسات multi-tenant إلزامياً | **v1.6 يفوز** — قرار عمل رسمي؛ `institution_id` مضاف لكل جدول tenant-scoped من اليوم الأول |
| حد حجم الملف | 100MB | 50MB | **50MB** — القرار الأحدث والأكثر تحفظاً على التخزين متعدد المؤسسات |
| نموذج الاشتراك | جدول `subscription_status` بعمود `paid_until` | `profiles.payment_status = 'CURRENT'` | **جدول `subscription_status`** — يفصل الحالة المالية عن الهوية؛ دالة `has_active_subscription()` أُعيدت كتابتها عليه |
| Availability | 99.5% عادي / 99.95% أثناء الامتحانات | 99% | **أرقام v2.0** — أدق وأصرم؛ 99% تُعتبر حداً أدنى مطلقاً لا هدفاً |
| Connection pooling | PgBouncer | Supavisor | **Supavisor** — هو الـ pooler المُدار في Supabase فعلياً (transaction mode) |
| ذروة دخول الامتحان | 500 طالب/60 ثانية | 2,000 طالب متزامن لكل مؤسسة | **2,000 لكل مؤسسة، سقف تصميمي 20,000 كلي** — الجداول مستقلة بين المؤسسات ولا آلية تمنع التداخل |
| تسمية الفصول | `terms` | `semesters` | **`terms`** — الاسم الموجود في الـ schema؛ المفهوم واحد |

---

## القسم 0: بروتوكول التحليل المتسلسل 🧠

### 0.1 — إعادة صياغة المشكلة (Problem Restatement)

منصة Academic Hub تتكون من 11 نظاماً رئيسياً (جدولة، حضور، امتحانات، دفعات، ملفات، دردشة...) موزعة على 3 أدوار تشغيلية (طالب، محاضر، إداري)، وتخدم — اعتباراً من هذا الإصدار — **10 مؤسسات تعليمية مستقلة على نفس البنية (multi-tenant)**. جميع الأنظمة تتشارك طبقة بيانات واحدة يجب أن تضمن: (أ) الفصل الصارم بين الأدوار على مستوى الصف الواحد، (ب) **الفصل المطلق بين المؤسسات — تسريب صف واحد عبر حدود مؤسسة هو فشل كارثي**، (ج) العمل دون اتصال مع مزامنة آمنة، (د) عدم تسريب أي درجة قبل الاعتماد النهائي، (هـ) تفرّد تسجيل الحضور، (و) دخول طارئ محدود زمنياً وذاتي الحذف. هذه الوثيقة تصمم المخطط الكامل (Schema)، سياسات الأمان (RLS)، والعقود البرمجية التي ستبني عليها بقية الأنظمة.

### 0.2 — الأسئلة الخمسة الحاسمة

1. **من المستخدم الفعلي؟** ~20,000 طالب (2,000 × 10 مؤسسات) + الكادر الأكاديمي والإداري لكل مؤسسة. ذروة الاستخدام: نافذة تسجيل الحضور (أول 10 دقائق من كل محاضرة) ونافذة دخول الامتحان (دفعة كاملة تدخل خلال 60 ثانية).
2. **ما هي العملية الأثقل؟** النظام Read-Heavy بنسبة تقديرية 85/15. الاستثناء الحرج: كتابة سجلات الحضور ودخول الامتحان — write bursts قصيرة وعالية الكثافة (حتى 2,000 كتابة/دقيقة لمؤسسة واحدة، والتصميم يفترض التداخل الكامل بين المؤسسات لأن جداول الامتحانات تحددها كل مؤسسة باستقلال تام).
3. **ماذا لو توقف النظام ساعة؟** المحاضرات تستمر ورقياً (مقبول)، لكن توقفه أثناء **امتحان جارٍ** كارثي. المطلوب: availability عادي 99.5%، مع نافذة حرجة 99.95% أثناء الامتحانات المجدولة.
4. **ما البيانات الأكثر حساسية؟** بالترتيب: (1) الدرجات قبل الاعتماد، (2) أسئلة الامتحانات قبل انعقادها، (3) صور الإشعارات البنكية، (4) الملاحظات الإدارية الخاصة، (5) رموز الدخول الطارئ. ويعلوها جميعاً كقاعدة عرضية: **أي بيانات مؤسسة يجب ألا تُرى من مؤسسة أخرى**.
5. **أضيق عنق زجاجة خلال 6 أشهر؟** استعلام "جدول الطالب الأسبوعي المُفلتر على مواده" مضروباً في 20,000 مستخدم. يعالج بفهارس مركّبة تبدأ بـ `institution_id` + كاش محلي (Hive) مع مزامنة تفاضلية عبر `updated_at`.

### 0.3 — الافتراضات المعلنة (Explicit Assumptions)

| # | الافتراض | مستوى الثقة | تأثيره لو كان خاطئاً |
|---|---|---|---|
| A-01 | 10 مؤسسات × 2,000 طالب نشط = ~20,000 طالب + كادر أكاديمي/إداري | عالي | لو تجاوز 50k مستخدم: read replicas و partitioning مبكر (القسم 4.4) |
| A-02 | الواجهة الأمامية Flutter (Hive مذكور في MVP §3.1.2) | عالي | لو تغيرت: يتغير عميل المزامنة فقط، لا يتغير الـ schema |
| A-03 | فريق التطوير صغير (1–4 مهندسين) بدون مهندس بنية تحتية متفرغ | متوسط | لو كبر الفريق: إعادة تقييم BaaS مقابل خدمات مخصصة |
| A-04 | حجم ملف واحد ≤ **50MB** (محاضرات PDF/صور) — قرار v1.6 | عالي | لو وُجدت فيديوهات: CDN وتخزين متدرج |
| A-05 | فصل دراسي نشط واحد **لكل مؤسسة** في أي لحظة | عالي | لو تعددت: `term_id` موجود أصلاً في كل الجداول الفصلية |
| A-06 | معاملات الدفع ~1,000/شهر/مؤسسة = ~10,000/شهر إجمالاً | متوسط | لو تضاعفت: مراجعة فهارس لوحة قيادة الدفعات فقط — البنية تتحمل |

### 0.4 — نطاق العمل (Scope Fence)

- ✅ **داخل النطاق:** مخطط قاعدة البيانات الكامل للأنظمة الـ 11 · التعدد المؤسسي (multi-tenancy) بنية وسياسات · نظام Enrollment الوظيفي الكامل · سياسات RLS لكل جدول · عقود الـ API/RPC الحرجة · نموذج المصادقة والدخول الطارئ · استراتيجية المزامنة دون اتصال · تخزين الملفات · الفهرسة والترحيلات
- ❌ **خارج النطاق:** خوارزمية BLE/QR للحضور (Attendance TDD) · واجهات المستخدم · دور المشرف العام Super Admin · القراءة الآلية لإشعارات الدفع بالذكاء الاصطناعي (مؤجلة في MVP §5.5) · تكامل Google Calendar/Drive (مستبعد في MVP §5.4) · التسجيل الذاتي للطالب في المواد (لا وجود له في الـ MVP)

---

## القسم 1: الملخص التنفيذي وسياق المشكلة 📋

### 1.1 — بيان المشكلة

كل نظام في Academic Hub يقرأ ويكتب في نفس البيانات المشتركة. بدون طبقة بيانات واحدة محكمة، سيبني كل نظام افتراضاته الخاصة، وستظهر ثغرات يستحيل إصلاحها لاحقاً: طالب يرى درجة زميله، تسجيل حضور مزدوج، سؤال امتحان يتسرب قبل موعده — أو الأسوأ في بيئة 10 مؤسسات: **بيانات مؤسسة تظهر لمؤسسة أخرى**. تكلفة عدم الحل: إعادة بناء كاملة بعد أول حادثة تسريب، وفقدان ثقة المؤسسات العشر دفعة واحدة.

### 1.2 — الهدف القابل للقياس

تمكين الأنظمة الـ 11 من القراءة والكتابة على مخطط بيانات موحّد يضمن عزل الأدوار **والمؤسسات** بنسبة 100% على مستوى الصف (مُختبر آلياً)، ويخدم 20,000 مستخدم بزمن فحص RLS أقل من 100ms وزمن استجابة p95 أقل من 300ms للاستعلامات الحرجة، ويستوعب دفعة دخول امتحان من 2,000 طالب لمؤسسة واحدة خلال 60 ثانية دون فقدان أي كتابة — مع افتراض التداخل الكامل بين امتحانات المؤسسات العشر (سقف تصميمي 20,000 متزامن).

### 1.3 — معايير النجاح (Success Criteria)

| معيار | القيمة المستهدفة | كيف يُقاس |
|---|---|---|
| زمن فحص RLS لأي سياسة | < 100ms | pg_stat_statements |
| زمن استعلام جدول الطالب الأسبوعي p95 | < 300ms | pg_stat_statements + APM |
| دفعة كتابة الحضور (2,000 طالب/دقيقة/مؤسسة) | صفر فقدان، صفر تكرار | اختبار حمل + عدّ الصفوف مقابل الطلبات |
| اختراق RLS بين الأدوار **أو بين المؤسسات** | 0 حالة | مجموعة اختبارات RLS آلية (pgTAP) لكل جدول: دور × عملية × مؤسسة |
| زمن المزامنة التفاضلية للجدول الأسبوعي | < 2s على شبكة 3G | قياس ميداني من العميل |
| توفر قاعدة البيانات أثناء نوافذ الامتحانات | ≥ 99.95% | مراقبة uptime مقيدة بجداول امتحانات المؤسسات العشر |
| صلاحية رمز الدخول الطارئ | 180 ثانية بالضبط، استخدام واحد | اختبار تكامل آلي |
| استيراد CSV للتسجيل | تقرير سطر-بسطر، صفر تسريب عبر مؤسسة | اختبار تكامل بملف يتعمد خلط مؤسستين |

### 1.4 — ما ليس هذا النظام (Anti-Goals)

- ليس نظام تحليلات أو BI — التقارير تُصدَّر PDF من استعلامات مباشرة، لا data warehouse
- ليس نظام مراسلة فورية عامة — الدردشة نموذج «منشور + تعليقات» محدود (MVP قرار #16)
- ليس بوابة دفع — لا معالجة مالية فعلية، فقط رفع صور إشعارات وتأكيد بشري (MVP §5.5)
- لا قرارات آلية نهائية — كل عملية آلية تنتهي بقرار بشري (المبدأ الحاكم الأول في MVP §1.1)
- **لا ترحيل تلقائي بين الفصول** — كل فصل جديد يتطلب تسجيلاً إدارياً صريحاً (قرار v1.6 §2.4): الترحيل التلقائي يستلزم محرك قواعد (نجح/رسب/مؤجَّل) لا مكان له في الـ MVP

---

## القسم 2: القيود التقنية واختيار التقنيات 🔧

### 2.1 — القيود المفروضة (Hard Constraints)

| نوع القيد | التفصيل | مصدره |
|---|---|---|
| تقني | فصل الأدوار والمؤسسات يُطبَّق عبر RLS على مستوى قاعدة البيانات | MVP §1.1 + تعديل v1.6 §1.3 |
| تقني | كل جدول محكوم بالمؤسسة يحمل `institution_id` — قاعدة غير قابلة للاستثناء | تعديل v1.6 §1.3 |
| تقني | `institution_id` يُقرأ من JWT claim لا من subquery داخل السياسات | تعديل v1.6 §1.3 — استعلام فرعي × 20,000 مستخدم = الفرق بين 100ms و timeout |
| تقني | العمل الكامل دون اتصال بعد أول مزامنة (Hive على العميل) | MVP §3.1.2 |
| تقني | إغلاق الامتحان بتوقيت الخادم لا الجهاز | MVP §3.6.1 |
| تقني | مصادقة موحدة عبر Google (بدون OTP، بدون تكامل Calendar/Drive) | MVP v3.4 + قرار #1 |
| فريق | فريق صغير بدون مهندس بنية تحتية متفرغ | افتراض A-03 |
| تنظيمي | كل قرار إداري يُوثَّق ولا يُحذف | MVP §5.0 |

### 2.2 — مصفوفة اختيار التقنيات (Stack Decision Matrix)

#### [D-01] طبقة: قاعدة البيانات الرئيسية

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **PostgreSQL 15+ (عبر Supabase) ✅** | RLS أصيلة على مستوى المحرك · قيود تفرّد وتحقق قوية · JSONB لأسئلة الامتحانات · Realtime مدمج | يتطلب انضباطاً في كتابة سياسات RLS · vertical scaling أولاً | RLS مطلب صريح، وSupabase يوفر Auth+Storage+Realtime لفريق صغير (A-03) |
| MySQL ❌ | شائعة، أداء قراءة جيد | لا RLS أصيلة — العزل في طبقة التطبيق (نقطة فشل بشرية) | يخالف القيد التقني الأول مباشرة |
| MongoDB ❌ | مرونة schema | لا قيود علائقية قوية ولا RLS مكافئة | البيانات علائقية بامتياز |

#### [D-02] طبقة: المصادقة

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **Supabase Auth (Google OAuth) ✅** | JWT يحمل `user_id` و`institution_id` (custom claim يُحقَن عند الدخول) يُستهلكان مباشرة في RLS | ربط حساب Google بالسجل الأكاديمي يتطلب جدول ربط مُدار إدارياً | يحقق قرار MVP #1 حرفياً، وصفر كود مصادقة مخصص |
| Firebase Auth ❌ | ناضج وموثوق | يعيش خارج قاعدة البيانات — لا يمكن استهلاك هويته داخل RLS مباشرة | يكسر وحدة الأمان بين المصادقة والبيانات |
| نظام JWT مخصص ❌ | تحكم كامل | سطح هجوم ضخم بلا مبرر لفريق صغير | تكلفة/مخاطرة غير مبررة (A-03) |

#### [D-03] طبقة: التخزين المحلي على العميل

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **Hive ✅** | مذكور صراحة في MVP §3.1.2 · سريع · بدون native dependencies | لا استعلامات علائقية — الفلترة في الذاكرة (مقبول: بيانات الطالب المحلية آلاف الصفوف فقط) | قرار محسوم في MVP |
| SQLite (drift) ❌ | استعلامات SQL كاملة محلياً | تعقيد إضافي بلا حاجة | إفراط هندسي |
| Isar ❌ | أداء ممتاز | مخالفة قرار موثق دون مبرر تشغيلي | الالتزام بالوثيقة المعتمدة |

#### [D-04] طبقة: الاستضافة وتخزين الملفات (صياغة v1.6 النهائية — تلغي كل ما سبق)

- **Supabase Free Tier مرفوض نهائياً حتى كبيئة إطلاق** — كان مرفوضاً أصلاً عند 2,000 متزامن لمؤسسة واحدة، وعند 10 مؤسسات يصبح النقاش منتهياً.
- الخطة: **Pro من اليوم الأول**، مع:
  - رفع حصة اتصالات Realtime المتزامنة قبل أول امتحان جماعي فعلي (تُطلب من Supabase حسب الذروة المتوقعة).
  - تفعيل **Supavisor (transaction pooling)** لكل اتصالات العميل — 20,000 عميل لا يمكن أن يحملوا اتصالات Postgres مباشرة.
  - مراجعة **Dedicated Compute / ترقية الحجم** عند تجاوز 3 مؤسسات نشطة فعلياً — نقطة مراجعة مُجدولة، ليست ردّ فعل.
- بند ميزانية تشغيلية شهرية للبنية يدخل التخطيط المالي للمشروع من الآن.

| الخيار (تخزين الملفات) | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **Supabase Storage (buckets خاصة + Signed URLs) ✅** | سياسات وصول مرتبطة بنفس RLS · روابط موقّعة منتهية الصلاحية · **مسار كل ملف يبدأ بـ `{institution_id}/` وسياسات الـ bucket تتحقق من التطابق مع claim المستخدم** | حصة تخزين لكل مؤسسة تُراقَب إدارياً | نموذج أمان موحد؛ صور الإشعارات البنكية يجب ألا تكون على روابط عامة |
| S3 مباشر ❌ | مرونة قصوى | إدارة IAM يدوياً + خدمة وسيطة للتفويض | ازدواجية أنظمة تفويض = ثغرات محتملة |
| تخزين داخل قاعدة البيانات (bytea) ❌ | معاملات موحدة | ينفخ القاعدة ويدمر النسخ الاحتياطي | غير عملي فوق بضعة ميغابايت |

### 2.3 — الديون التقنية المقبولة عمداً (Accepted Technical Debt)

| الدين | لماذا نقبله الآن | متى يُسدَّد |
|---|---|---|
| لا read replicas | الحمل الحالي لا يبرره بعد الفهرسة المركّبة | عند تجاوز 50k مستخدم نشط أو p95 > 300ms مستمر |
| لا partitioning لجدول الحضور | ~مليون صف/فصل/مؤسسة | عند تجاوز 10 ملايين صف (القسم 4.4) |
| التقارير PDF من استعلامات مباشرة على القاعدة الرئيسية | حجم البيانات صغير | عند بطء ملحوظ: replica مخصصة للتقارير |
| Dedicated Compute مؤجّل | أقل من 3 مؤسسات نشطة عند الإطلاق | مراجعة مُجدولة عند المؤسسة النشطة الرابعة (D-04) |
| المراقبة الصوتية سجل مخالفات فقط دون تحليلات | يحقق قرار MVP #13 بالضبط | لا يُسدَّد — أي توسعة قرار منتج جديد |

---

## القسم 3: معمارية النظام والمخططات 🏗️

### 3.1 — المخطط العام (System Context)

\`\`\`mermaid
graph TB
    subgraph Clients["أجهزة المستخدمين (10 مؤسسات)"]
        ST[📱 تطبيق الطالب<br/>Flutter + Hive]
        LC[📱 تطبيق المحاضر]
        AD[💻 لوحة الإداري]
    end

    subgraph Supabase["منصة الخلفية (Supabase Pro + Supavisor)"]
        AUTH[🔐 Auth<br/>Google OAuth + JWT<br/>+ institution_id claim]
        PG[(🗄️ PostgreSQL<br/>RLS: دور × مؤسسة<br/>على كل جدول)]
        RPC[⚙️ Database Functions<br/>RPC للعمليات الحرجة]
        RT[📡 Realtime<br/>دردشة + إشعارات]
        STG[📦 Storage<br/>مسارات {institution_id}/...]
    end

    ST --> AUTH
    LC --> AUTH
    AD --> AUTH
    ST --> RPC
    LC --> RPC
    AD --> RPC
    RPC --> PG
    ST --> RT
    LC --> RT
    RT --> PG
    ST --> STG
    LC --> STG
    AD --> STG
\`\`\`

### 3.2 — جدول المكونات والمسؤوليات

| المكون | مسؤوليته الوحيدة | ماذا لو سقط؟ | استراتيجية التعافي |
|---|---|---|---|
| PostgreSQL | مصدر الحقيقة الوحيد + تطبيق RLS (دور × مؤسسة) | توقف كامل للكتابة؛ القراءة تستمر من كاش Hive | PITR + failover مُدار، RTO ≤ 5 دقائق |
| Auth | إصدار JWT (مع `institution_id` claim) والتحقق من Google | لا دخول جديد؛ الجلسات القائمة تستمر حتى انتهاء التوكن | الدخول الطارئ QR لا يعتمد على Google (مسار مستقل) |
| Supavisor | تجميع اتصالات 20,000 عميل في transaction mode | استنفاد اتصالات مباشر | مُدار من Supabase؛ مراقبة عتبة 80% |
| Database Functions (RPC) | العمليات متعددة الخطوات ذات المنطق الأمني | تفشل العمليات الحرجة فقط؛ القراءات تستمر عبر PostgREST | stateless — كل RPC حرج idempotent |
| Realtime | بث الدردشة والإشعارات الحية | fallback إلى polling كل 60 ثانية | إعادة اتصال تلقائية مع backoff |
| Storage | حفظ وتقديم الملفات بروابط موقّعة ضمن مسار المؤسسة | لا رفع/تحميل؛ الملفات المحمّلة محلياً تعمل | إعادة محاولة الرفع تلقائياً من العميل |
| Hive (العميل) | كاش القراءة الكامل + طابور الكتابات المعلّقة | فقدانه = إعادة مزامنة كاملة | إعادة بناء تلقائي من الخادم |

### 3.3 — تدفق البيانات لأهم العمليات (Critical Path Sequences)

**العملية 1: دخول الامتحان وتسليمه (الأخطر أمنياً)**

\`\`\`mermaid
sequenceDiagram
    participant S as 📱 الطالب
    participant R as ⚙️ RPC
    participant D as 🗄️ PostgreSQL

    S->>R: start_exam_attempt(exam_id)
    Note over R: نقطة فشل: توقيت الجهاز مزوّر<br/>الدفاع: كل التحقق بـ now() على الخادم
    R->>D: تحقق: النافذة الزمنية + التسجيل ACTIVE بالمادة + لا محاولة سابقة + نفس المؤسسة
    D-->>R: إنشاء attempt (UNIQUE على student+exam) + snapshot الأسئلة
    R-->>S: الأسئلة (بدون answer_key) + ends_at بتوقيت الخادم
    loop أثناء الامتحان
        S->>R: save_answer(attempt_id, ...) — حفظ فوري عند كل اختيار
        Note over R: يُرفض أي حفظ بعد ends_at (server-side)
        S->>R: log_violation(type) عند رصد مخالفة
        R->>D: إدراج في exam_violations (append-only)
        R-->>S: أمر إصدار التنبيه الصوتي من جهاز الطالب
    end
    S->>R: submit_attempt(attempt_id)
    R->>D: قفل المحاولة + التصحيح الآلي داخل معاملة واحدة
    R-->>S: تأكيد استلام فقط — بدون درجة (قرار MVP #7)
\`\`\`

**العملية 2: مزامنة حضور مسجَّل دون اتصال**

\`\`\`mermaid
sequenceDiagram
    participant S as 📱 الطالب (أوفلاين)
    participant Q as 📦 طابور Hive
    participant R as ⚙️ RPC
    participant D as 🗄️ PostgreSQL

    S->>Q: تسجيل حضور محلي + client_uuid ثابت
    Note over S,Q: انقطاع الشبكة
    Q->>R: عند عودة الاتصال: record_attendance(session_id, client_uuid, proof)
    Note over R: نقطة فشل: إعادة إرسال مكررة<br/>الدفاع: UNIQUE(session_id, student_id) + idempotency على client_uuid
    R->>D: INSERT ... ON CONFLICT DO NOTHING
    D-->>R: صف واحد مضمون مهما تكررت المحاولات
    R-->>Q: تأكيد → حذف من الطابور
\`\`\`

**العملية 3: الدخول الطارئ برمز QR (قرار MVP #2)**

\`\`\`mermaid
sequenceDiagram
    participant A as 💻 الإداري
    participant R as ⚙️ RPC
    participant D as 🗄️ PostgreSQL
    participant U as 📱 المستخدم المتعثر

    A->>R: generate_emergency_token(user_id)
    R->>D: تخزين hash(token) فقط + expires_at = now() + 3 دقائق
    R-->>A: QR يعرض التوكن الخام (لا يُخزَّن خاماً أبداً)
    U->>R: redeem_emergency_token(token)
    Note over R: الدفاع: hash مقارنة + used_at IS NULL + expires_at > now()<br/>داخل UPDATE ذري واحد
    R->>D: UPDATE ... SET used_at = now() WHERE ... RETURNING user_id
    R-->>U: جلسة مؤقتة (12 ساعة) بصلاحيات المستخدم نفسه
    D->>D: pg_cron: حذف السجلات المنتهية كل دقيقة
\`\`\`

**العملية 4: الاستيراد بالجملة للتسجيل (CSV) — جديد من v1.6**

\`\`\`mermaid
sequenceDiagram
    participant A as 💻 الإداري
    participant R as ⚙️ RPC
    participant D as 🗄️ PostgreSQL

    A->>R: import_enrollments_csv(course_id, rows[])
    loop لكل صف
        R->>D: تحقق: الطالب موجود وغير محذوف + نفس institution_id للإداري الرافع<br/>+ المادة لنفس المؤسسة والفصل النشط + لا تسجيل ACTIVE مسبق
        alt الصف صالح
            D-->>R: INSERT enrollment (ACTIVE)
        else الصف فاشل
            D-->>R: تسجيل السبب — لا فشل كلي صامت
        end
    end
    R-->>A: تقرير سطر-بسطر (نجح/فشل + السبب)
\`\`\`

### 3.4 — قرارات معمارية جوهرية

#### [D-05] المنطق الأمني داخل قاعدة البيانات (RLS + RPC) وليس في خدمة وسيطة

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **RLS + Database Functions ✅** | نقطة تحقق واحدة يستحيل تجاوزها من أي عميل · لا خادم وسيط يُدار | منطق PL/pgSQL أصعب في الاختبار (يعالج بـ pgTAP) | فريق صغير (A-03) + الفصل الصارم؛ الفهرسة المركّبة بالمؤسسة تُبقي الحد الأدائي أعلى من ذروة الحمل |
| خدمة API وسيطة (Node/Go) ❌ | حرية منطق كاملة | كل endpoint منسي التحقق = ثغرة | يضاعف سطح الهجوم بلا مقابل |

#### [D-06] المزامنة التفاضلية عبر `updated_at` بدل إعادة التحميل الكامل

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **Delta sync على updated_at ✅** | حمل شبكة ضئيل · بسيط التنفيذ | لا يلتقط الحذف الفعلي — يعالج بـ soft delete (القسم 4.4) | يحقق قيد العمل دون اتصال بأقل تعقيد |
| CRDT / محرك مزامنة كامل ❌ | حل نزاعات تلقائي | تعقيد ضخم؛ بيانات الطالب قراءة-فقط في معظمها | إفراط هندسي صريح |

#### [D-08] هوية المؤسسة في JWT claim لا في subquery (من v1.6)

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **`institution_id` كـ custom claim يُحقَن عند الدخول ✅** | قراءة فورية داخل كل سياسة RLS بلا أي وصول لجدول | تغيير مؤسسة المستخدم يتطلب إعادة إصدار التوكن (حدث نادر جداً) | subquery واحد على `profiles` مضروب في 20,000 مستخدم هو الفرق بين 100ms و timeout |
| subquery على `profiles` داخل كل سياسة ❌ | لا حاجة لتخصيص claims | كارثة أداء عند التوسع | مرفوض صراحة في v1.6 §1.3 |

#### [D-09] صف `courses` جديد لكل فصل (من v1.6 §2.3)

المادة الواحدة تحصل على **صف جديد في `courses` لكل فصل تُدرَّس فيه** (نفس `code` المنطقي، صف فيزيائي مختلف عبر `UNIQUE(term_id, code)`). هذا يحل مشكلة "نفس المادة عبر فصلين" جذرياً بدل ترقيعها في الاستعلامات.

---

## القسم 4: نماذج البيانات وتصميم قاعدة البيانات 🗄️

### 4.0 — قواعد التعدد المؤسسي الإلزامية (من v1.6 — غير قابلة للاستثناء)

1. **كل جدول محكوم بالمؤسسة يحمل `institution_id`.**
2. **كل فهرس أساسي على جدول tenant-scoped يبدأ بـ `institution_id`** — أي فحص RLS يمسح جدولاً بلا فهرس مركّب سيخرق سقف 100ms عند 10 مؤسسات.
3. **`institution_id` يُقرأ من JWT claim** ([D-08]) وليس عبر subquery.
4. **Storage:** مسار كل ملف يبدأ بـ `{institution_id}/` وسياسات الـ bucket تتحقق من التطابق مع claim المستخدم.

### 4.1 — مخطط الكيانات (ERD)

\`\`\`mermaid
erDiagram
    INSTITUTIONS ||--o{ PROFILES : "مستخدموها"
    INSTITUTIONS ||--o{ TERMS : "فصولها"
    INSTITUTIONS ||--o{ COURSES : "موادها"
    PROFILES ||--o{ ENROLLMENTS : "يسجَّل في"
    COURSES ||--o{ ENROLLMENTS : "تضم"
    COURSES ||--o{ COURSE_LECTURERS : "يدرّسها"
    PROFILES ||--o{ COURSE_LECTURERS : "معيَّن على"
    COURSES ||--o{ EVENTS : "لها"
    ROOMS ||--o{ EVENTS : "تستضيف"
    EVENTS ||--o{ ROOM_REQUESTS : "قد ينشأ من"
    EVENTS ||--o| ATTENDANCE_SESSIONS : "له جلسة"
    ATTENDANCE_SESSIONS ||--o{ ATTENDANCE_RECORDS : "تضم"
    PROFILES ||--o{ ATTENDANCE_RECORDS : "يسجل"
    EVENTS ||--o{ ABSENCE_EXCUSES : "يُلتمس عنه"
    COURSES ||--o{ FILES : "ملفاتها"
    FILES ||--o{ FILE_NOTES : "ملاحظات عليها"
    COURSES ||--|| CHAT_CHANNELS : "قناتها"
    CHAT_CHANNELS ||--o{ CHAT_POSTS : "منشوراتها"
    CHAT_POSTS ||--o{ CHAT_COMMENTS : "تعليقاتها"
    COURSES ||--o{ QUESTIONS : "بنك أسئلتها"
    EVENTS ||--o{ EXAMS : "امتحان الحدث"
    EXAMS ||--o{ EXAM_QUESTION_EXCLUSIONS : "استبعاداته"
    EXAMS ||--o{ EXAM_ATTEMPTS : "محاولاته"
    EXAM_ATTEMPTS ||--o{ EXAM_ANSWERS : "إجاباتها"
    EXAM_ATTEMPTS ||--o{ EXAM_VIOLATIONS : "مخالفاتها"
    COURSES ||--|| GRADE_WEIGHTS : "أوزانها"
    EVENTS ||--o{ SUBMISSION_GRADES : "درجات تسليماته"
    COURSES ||--o{ FINAL_RESULTS : "نتائجها"
    PROFILES ||--o{ PAYMENT_REQUESTS : "دفعاته"
    PROFILES ||--o{ NOTIFICATIONS : "إشعاراته"
    PROFILES ||--o{ ADMIN_NOTES : "ملاحظات عليه"
    PROFILES ||--o{ EMERGENCY_TOKENS : "رموزه الطارئة"
\`\`\`

### 4.2 — تعريف الجداول (Schema)

\`\`\`sql
-- ============================================================
-- 0) الأنواع المعدودة (ENUMs) — قيم مغلقة من وثيقة MVP + v1.6 حصراً
-- ============================================================
CREATE TYPE user_role         AS ENUM ('student', 'lecturer', 'admin');
CREATE TYPE event_type        AS ENUM ('lecture','lab','seminar','exam','research','assignment','cultural');
CREATE TYPE event_status      AS ENUM ('pending','approved','rejected','cancelled');
CREATE TYPE submission_mode   AS ENUM ('online','paper');            -- MVP §4.1
CREATE TYPE review_status     AS ENUM ('pending','accepted','rejected');
CREATE TYPE question_type     AS ENUM ('mcq','fill_blank','image_labeled'); -- MVP §3.6.2
CREATE TYPE payment_channel   AS ENUM ('university_fees','app_subscription'); -- قرار #14
CREATE TYPE violation_type    AS ENUM ('focus_loss','copy_paste');   -- تصحيح v1.6 §5.1 (انظر الملحق أ)
CREATE TYPE announce_level    AS ENUM ('urgent','important','normal'); -- MVP §5.2
CREATE TYPE enrollment_status AS ENUM ('ACTIVE','DROPPED','COMPLETED'); -- دورة حياة v1.6 §2.5

-- ============================================================
-- 1) institutions: جذر التعدد المؤسسي (v1.6)
-- ============================================================
CREATE TABLE institutions (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name       TEXT NOT NULL,
    is_active  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- 2) profiles: امتداد أكاديمي لهوية Auth — لا كلمات مرور هنا أبداً
-- ============================================================
CREATE TABLE profiles (
    id             UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    institution_id UUID NOT NULL REFERENCES institutions(id), -- قاعدة 4.0 رقم 1
    role           user_role NOT NULL,
    full_name      TEXT NOT NULL,
    university_id  TEXT,                              -- الرقم الجامعي؛ NULL للإداري
    expected_email TEXT NOT NULL,                     -- يُدخله الإداري قبل أول دخول (MVP §5.4)
    college        TEXT,
    department     TEXT,
    batch          TEXT,
    barcode_payload TEXT,                             -- محتوى باركود البطاقة (قرار #3)
    is_active      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ,                       -- soft delete (القسم 4.4)
    -- التفرّد داخل المؤسسة لا عالمياً: نفس الرقم الجامعي قد يتكرر بين مؤسستين
    UNIQUE (institution_id, university_id),
    UNIQUE (institution_id, expected_email),
    UNIQUE (institution_id, barcode_payload)
);
CREATE INDEX idx_profiles_tenant ON profiles (institution_id, role) WHERE deleted_at IS NULL;

-- ============================================================
-- 3) الهيكل الأكاديمي
-- ============================================================
CREATE TABLE terms (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name           TEXT NOT NULL,
    starts_on      DATE NOT NULL,
    ends_on        DATE NOT NULL,
    is_active      BOOLEAN NOT NULL DEFAULT FALSE
);
-- فصل نشط واحد لكل مؤسسة (A-05): قيد جزئي مركّب
CREATE UNIQUE INDEX one_active_term_per_institution
    ON terms (institution_id) WHERE is_active;

CREATE TABLE courses (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    term_id        BIGINT NOT NULL REFERENCES terms(id),
    name           TEXT NOT NULL,
    code           TEXT NOT NULL,
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ,
    -- [D-09] صف جديد لكل فصل: نفس code المنطقي، صف فيزيائي مختلف
    UNIQUE (term_id, code)
);
CREATE INDEX idx_courses_tenant ON courses (institution_id, term_id) WHERE deleted_at IS NULL;

CREATE TABLE course_lecturers (                      -- «الكتاب المشترك» (قرار #5)
    course_id   BIGINT NOT NULL REFERENCES courses(id),
    lecturer_id UUID   NOT NULL REFERENCES profiles(id),
    PRIMARY KEY (course_id, lecturer_id)
);

-- ============================================================
-- 4) enrollments — المواصفة الوظيفية الكاملة (v1.6 §2)
--    من يُسجِّل: الإداري حصرياً (فردي أو CSV بالجملة).
--    لا تسجيل ذاتياً للطالب في الـ MVP. لا ترحيل تلقائياً بين الفصول.
--    لا حذف فيزيائياً أبداً — كشف الدرجات يجب أن يبقى قابلاً
--    للاستخراج بعد سنوات.
-- ============================================================
CREATE TABLE enrollments (
    institution_id UUID NOT NULL REFERENCES institutions(id),
    course_id      BIGINT NOT NULL REFERENCES courses(id),
    student_id     UUID   NOT NULL REFERENCES profiles(id),
    status         enrollment_status NOT NULL DEFAULT 'ACTIVE',
    -- دورة الحياة (v1.6 §2.5):
    --   ACTIVE → DROPPED   : فعل إداري صريح فقط
    --   ACTIVE → COMPLETED : أثر جانبي ذرّي لاعتماد النتيجة النهائية
    --                        (نفس الـ transaction في approve_final_results)
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (course_id, student_id)
);
CREATE INDEX idx_enrollments_tenant
    ON enrollments (institution_id, student_id, course_id);

-- ============================================================
-- 5) القاعات وطلبات الحجز (MVP §5.1)
-- ============================================================
CREATE TABLE rooms (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    name           TEXT NOT NULL,
    location       TEXT,
    capacity       INT  NOT NULL CHECK (capacity > 0),
    is_lab         BOOLEAN NOT NULL DEFAULT FALSE
);
CREATE INDEX idx_rooms_tenant ON rooms (institution_id);

CREATE TABLE room_requests (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    course_id      BIGINT NOT NULL REFERENCES courses(id),
    lecturer_id    UUID   NOT NULL REFERENCES profiles(id),
    requested_type event_type NOT NULL
        CHECK (requested_type IN ('lecture','lab','seminar')), -- الامتحان محظور هنا (قرار #10)
    requested_start TIMESTAMPTZ NOT NULL,
    requested_end   TIMESTAMPTZ NOT NULL CHECK (requested_end > requested_start),
    status         review_status NOT NULL DEFAULT 'pending',
    rejection_reason TEXT,
    reviewed_by    UUID REFERENCES profiles(id),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_room_requests_tenant
    ON room_requests (institution_id, status);

-- ============================================================
-- 6) الأحداث — قلب نظام الجدولة (MVP §3.1)
-- ============================================================
CREATE TABLE events (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    course_id      BIGINT REFERENCES courses(id),     -- NULL للأحداث الثقافية العامة
    type           event_type NOT NULL,
    title          TEXT NOT NULL,
    description    TEXT,
    room_id        BIGINT REFERENCES rooms(id),
    starts_at      TIMESTAMPTZ NOT NULL,
    ends_at        TIMESTAMPTZ NOT NULL CHECK (ends_at > starts_at),
    status         event_status NOT NULL DEFAULT 'approved',
    lecturer_note  TEXT,                              -- «أونلاين / مؤجلة» (MVP §3.1.1)
    submission_mode submission_mode,
    paper_submit_location TEXT,
    deadline       TIMESTAMPTZ,
    created_by     UUID NOT NULL REFERENCES profiles(id),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ,
    CHECK (submission_mode IS DISTINCT FROM 'paper' OR paper_submit_location IS NOT NULL),
    CHECK (type NOT IN ('research','assignment')
           OR (submission_mode IS NOT NULL AND deadline IS NOT NULL))
);
-- منع تعارض القاعات آلياً (MVP §5.1)
ALTER TABLE events ADD CONSTRAINT no_room_overlap
    EXCLUDE USING gist (
        room_id WITH =,
        tstzrange(starts_at, ends_at) WITH &&
    ) WHERE (room_id IS NOT NULL AND status = 'approved' AND deleted_at IS NULL);
-- جدول الطالب الأسبوعي — أثقل استعلام في النظام (0.2 س5)
CREATE INDEX idx_events_tenant
    ON events (institution_id, course_id, starts_at) WHERE deleted_at IS NULL;
CREATE INDEX idx_events_sync ON events (institution_id, updated_at); -- [D-06]

-- ============================================================
-- 7) الحضور (يكمّل Attendance TDD — البنية التخزينية فقط)
-- ============================================================
CREATE TABLE attendance_sessions (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    event_id       BIGINT NOT NULL UNIQUE REFERENCES events(id),
    opened_by      UUID   NOT NULL REFERENCES profiles(id),
    opens_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    closes_at      TIMESTAMPTZ NOT NULL,
    qr_secret      TEXT NOT NULL
);

CREATE TABLE attendance_records (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    session_id     BIGINT NOT NULL REFERENCES attendance_sessions(id),
    student_id     UUID   NOT NULL REFERENCES profiles(id),
    method         TEXT   NOT NULL CHECK (method IN ('ble','agent','qr','manual','excuse_auto')),
    client_uuid    UUID   NOT NULL,                   -- مفتاح idempotency للمزامنة الأوفلاين
    recorded_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (session_id, student_id),                  -- ضمان التفرّد (MVP §3.5)
    UNIQUE (client_uuid)
);
CREATE INDEX idx_attendance_session ON attendance_records (institution_id, session_id);
CREATE INDEX idx_attendance_student ON attendance_records (institution_id, student_id);

CREATE TABLE absence_excuses (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    event_id       BIGINT NOT NULL REFERENCES events(id),
    student_id     UUID   NOT NULL REFERENCES profiles(id),
    category       TEXT   NOT NULL CHECK (category IN ('medical','emergency','other')),
    reason         TEXT,
    document_path  TEXT,                              -- يبدأ بـ {institution_id}/ (قاعدة 4.0)
    status         review_status NOT NULL DEFAULT 'pending',
    reviewed_by    UUID REFERENCES profiles(id),
    rejection_reason TEXT,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (event_id, student_id)
);

-- ============================================================
-- 8) مستودع الملفات والاجتهادات (MVP §3.3, §4.3.1)
-- ============================================================
CREATE TABLE files (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    course_id      BIGINT NOT NULL REFERENCES courses(id),
    event_id       BIGINT REFERENCES events(id),
    uploader_id    UUID   NOT NULL REFERENCES profiles(id),
    storage_path   TEXT   NOT NULL,                   -- {institution_id}/... حصراً
    file_name      TEXT   NOT NULL,
    size_bytes     BIGINT NOT NULL CHECK (size_bytes <= 52428800), -- حد 50MB (A-04 / v1.6)
    is_contribution BOOLEAN NOT NULL DEFAULT FALSE,
    review_status  review_status,
    rejection_reason TEXT,
    reviewed_by    UUID REFERENCES profiles(id),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ,
    CHECK (NOT is_contribution OR review_status IS NOT NULL)
);
CREATE INDEX idx_files_tenant
    ON files (institution_id, course_id) WHERE deleted_at IS NULL;

CREATE TABLE file_notes (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    file_id    BIGINT NOT NULL REFERENCES files(id),
    author_id  UUID   NOT NULL REFERENCES profiles(id),
    note       TEXT   NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- 9) الدردشة: قناة واحدة لكل مادة، منشور + تعليقات (قرار #16)
-- ============================================================
CREATE TABLE chat_channels (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    course_id      BIGINT UNIQUE REFERENCES courses(id),
    kind           TEXT NOT NULL DEFAULT 'course'
        CHECK (kind IN ('course','support','admin_lecturer','admin_admin'))
);

CREATE TABLE chat_posts (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    channel_id BIGINT NOT NULL REFERENCES chat_channels(id),
    author_id  UUID   NOT NULL REFERENCES profiles(id),
    body       TEXT   NOT NULL CHECK (char_length(body) <= 8000),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ
);

CREATE TABLE chat_comments (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    post_id    BIGINT NOT NULL REFERENCES chat_posts(id),
    author_id  UUID   NOT NULL REFERENCES profiles(id),
    body       TEXT   NOT NULL CHECK (char_length(body) <= 4000),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ
);

-- ============================================================
-- 10) بنك الأسئلة والامتحانات (MVP §3.6, §4.3.3 + v1.6 §5.3)
-- ============================================================
CREATE TABLE questions (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    course_id      BIGINT NOT NULL REFERENCES courses(id),
    type           question_type NOT NULL,
    body           JSONB NOT NULL,   -- نص السؤال + الخيارات + مسار الصورة
    answer_key     JSONB NOT NULL,   -- لا يصل للعميل أبداً (RLS + أعمدة منتقاة)
    version        INT NOT NULL DEFAULT 1,           -- optimistic locking (v1.6 §5.3):
                                                     -- نسخة العميل الأقدم تُرفض برسالة
                                                     -- "تم تعديل هذا السؤال من زميل"
    created_by     UUID NOT NULL REFERENCES profiles(id),
    updated_by     UUID REFERENCES profiles(id),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ
);
CREATE INDEX idx_questions_tenant
    ON questions (institution_id, course_id) WHERE deleted_at IS NULL;

CREATE TABLE question_audit_log (                     -- Audit Log كامل غير قابل للحذف
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    question_id BIGINT NOT NULL,
    actor_id    UUID   NOT NULL,
    action      TEXT   NOT NULL CHECK (action IN ('create','update','delete')),
    snapshot    JSONB  NOT NULL,                      -- حالة السؤال قبل التغيير
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE exams (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    event_id       BIGINT NOT NULL UNIQUE REFERENCES events(id),
    course_id      BIGINT NOT NULL REFERENCES courses(id),
    kind           TEXT   NOT NULL CHECK (kind IN ('midterm','final')),
    duration_minutes INT NOT NULL CHECK (duration_minutes BETWEEN 5 AND 300),
    question_count   INT NOT NULL CHECK (question_count > 0),
    scheduled_by   UUID NOT NULL REFERENCES profiles(id) -- إداري حصراً (قرار #10)
);

CREATE TABLE exam_question_exclusions (               -- آلية الاستبعاد لكل امتحان (قرار #5)
    exam_id     BIGINT NOT NULL REFERENCES exams(id),
    question_id BIGINT NOT NULL REFERENCES questions(id),
    excluded_by UUID   NOT NULL REFERENCES profiles(id),
    PRIMARY KEY (exam_id, question_id)
);

CREATE TABLE exam_attempts (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    exam_id        BIGINT NOT NULL REFERENCES exams(id),
    student_id     UUID   NOT NULL REFERENCES profiles(id),
    question_order BIGINT[] NOT NULL,                 -- الترتيب العشوائي المثبّت
    questions_snapshot JSONB NOT NULL,                -- Snapshot عند السحب (v1.6 §5.3):
                                                      -- نص السؤال وخياراته يُنسَخان لحظة
                                                      -- السحب — تعديل السؤال لاحقاً لا يمس
                                                      -- محاولة جارية أو مُسلَّمة
    started_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    ends_at        TIMESTAMPTZ NOT NULL,              -- محسوب على الخادم
    submitted_at   TIMESTAMPTZ,
    auto_score     NUMERIC(5,2),                      -- لا يُعرض للطالب (قرار #7)
    UNIQUE (exam_id, student_id)                      -- محاولة واحدة فقط (MVP §3.6.1)
);
CREATE INDEX idx_attempts_exam ON exam_attempts (institution_id, exam_id);

CREATE TABLE exam_answers (
    attempt_id  BIGINT NOT NULL REFERENCES exam_attempts(id),
    question_id BIGINT NOT NULL REFERENCES questions(id),
    answer      JSONB  NOT NULL,
    saved_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- v1.6 §5.2: كل إجابة تُحفَظ فور اختيارها (ليس عند التسليم فقط)؛
    -- المزامنة عند عودة الاتصال upsert idempotent بهذا المفتاح —
    -- لا إدراج مكرر عند إعادة المحاولة
    PRIMARY KEY (attempt_id, question_id)
);

CREATE TABLE exam_violations (                        -- سجل المراقبة — append-only
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    attempt_id  BIGINT NOT NULL REFERENCES exam_attempts(id),
    type        violation_type NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- 11) الأوزان والدرجات والنتائج (MVP §4.3.4, §5.6 + v1.6 §5.4)
-- ============================================================
CREATE TABLE grade_weights (
    course_id   BIGINT PRIMARY KEY REFERENCES courses(id),
    attendance_pct SMALLINT NOT NULL DEFAULT 0 CHECK (attendance_pct BETWEEN 0 AND 100),
    assignments_pct SMALLINT NOT NULL DEFAULT 0 CHECK (assignments_pct BETWEEN 0 AND 100),
    research_pct   SMALLINT NOT NULL DEFAULT 0 CHECK (research_pct BETWEEN 0 AND 100),
    seminars_pct   SMALLINT NOT NULL DEFAULT 0 CHECK (seminars_pct BETWEEN 0 AND 100),
    midterm_pct    SMALLINT NOT NULL DEFAULT 0 CHECK (midterm_pct BETWEEN 0 AND 100),
    final_pct      SMALLINT NOT NULL DEFAULT 0 CHECK (final_pct BETWEEN 0 AND 100),
    locked_at      TIMESTAMPTZ,                       -- القفل عند الاعتماد (قرار #4)
    updated_by     UUID REFERENCES profiles(id),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- v1.6 §5.4: المجموع = 100% إلزامي — يُرفض الحفظ إن لم يتحقق بالضبط.
    -- الأوزان تُطبَّق مرة واحدة فقط لحظة اعتماد النتيجة؛ قبلها تعديلها بلا
    -- أي أثر جانبي لأن الدرجات الفردية مخزَّنة مستقلة.
    CHECK (attendance_pct + assignments_pct + research_pct
         + seminars_pct + midterm_pct + final_pct = 100)
);
-- القفل يُفرض بـ trigger: أي UPDATE بعد locked_at IS NOT NULL يُرفض على مستوى المحرك

CREATE TABLE submission_grades (
    event_id    BIGINT NOT NULL REFERENCES events(id),
    student_id  UUID   NOT NULL REFERENCES profiles(id),
    score       NUMERIC(5,2) NOT NULL CHECK (score >= 0),
    file_id     BIGINT REFERENCES files(id),
    graded_by   UUID   NOT NULL REFERENCES profiles(id),
    graded_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (event_id, student_id)
);

CREATE TABLE final_results (                          -- الوحيدة المرئية للطالب (قرار #7)
    course_id   BIGINT NOT NULL REFERENCES courses(id),
    student_id  UUID   NOT NULL REFERENCES profiles(id),
    total_score NUMERIC(5,2) NOT NULL,
    breakdown   JSONB NOT NULL,
    approved_by UUID  NOT NULL REFERENCES profiles(id),
    approved_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (course_id, student_id)
);

-- ============================================================
-- 12) الدفعات — قناتان حصراً (قرار #14)
-- ============================================================
CREATE TABLE payment_requests (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    student_id     UUID NOT NULL REFERENCES profiles(id),
    channel        payment_channel NOT NULL,
    amount         NUMERIC(12,2),
    receipt_path   TEXT NOT NULL,                     -- {institution_id}/... في bucket خاص حصراً
    status         review_status NOT NULL DEFAULT 'pending',
    rejection_reason TEXT,
    reviewed_by    UUID REFERENCES profiles(id),
    reviewed_at    TIMESTAMPTZ,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payments_pending
    ON payment_requests (institution_id, status, channel) WHERE status = 'pending';

CREATE TABLE subscription_status (                    -- حالة اشتراك التطبيق (MVP §3.7.2)
    student_id  UUID PRIMARY KEY REFERENCES profiles(id),
    paid_until  DATE NOT NULL,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- 13) الإشعارات والإعلانات والملاحظات (MVP §3.2, §5.2, §4.3.6)
-- ============================================================
CREATE TABLE announcements (                          -- «البورد الأسود»
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    author_id      UUID NOT NULL REFERENCES profiles(id),
    title          TEXT NOT NULL,
    body           TEXT NOT NULL,
    level          announce_level NOT NULL DEFAULT 'normal',
    is_pinned      BOOLEAN NOT NULL DEFAULT FALSE,
    publish_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ
);
CREATE INDEX idx_announcements_tenant
    ON announcements (institution_id, publish_at DESC) WHERE deleted_at IS NULL;

CREATE TABLE notifications (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id    UUID NOT NULL REFERENCES profiles(id),
    kind       TEXT NOT NULL,
    payload    JSONB NOT NULL,
    is_read    BOOLEAN NOT NULL DEFAULT FALSE,
    is_pinned  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notifications_user
    ON notifications (user_id, is_read, created_at DESC);

CREATE TABLE admin_notes (                            -- ملاحظات خاصة غير مرئية لصاحبها
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    subject_id UUID NOT NULL REFERENCES profiles(id),
    author_id  UUID NOT NULL REFERENCES profiles(id),
    note       TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- 14) الدخول الطارئ (قرار #2) — التصميم الآمن
-- ============================================================
CREATE TABLE emergency_tokens (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id     UUID NOT NULL REFERENCES profiles(id),
    token_hash  TEXT NOT NULL UNIQUE,                 -- SHA-256 — الخام لا يُخزَّن أبداً
    created_by  UUID NOT NULL REFERENCES profiles(id),
    expires_at  TIMESTAMPTZ NOT NULL,                 -- now() + 3 دقائق على الخادم
    used_at     TIMESTAMPTZ
);
-- pg_cron كل دقيقة: DELETE FROM emergency_tokens WHERE expires_at < now() OR used_at IS NOT NULL;
\`\`\`

**الدوال المساعدة وسياسات RLS النموذجية:**

\`\`\`sql
-- تفعيل RLS إلزامي على كل جدول بلا استثناء
ALTER TABLE final_results    ENABLE ROW LEVEL SECURITY;
ALTER TABLE questions        ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_posts       ENABLE ROW LEVEL SECURITY;
ALTER TABLE admin_notes      ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_requests ENABLE ROW LEVEL SECURITY;
ALTER TABLE enrollments      ENABLE ROW LEVEL SECURITY;
ALTER TABLE events           ENABLE ROW LEVEL SECURITY;
-- ... (كل الجداول)

-- [D-08] هوية المؤسسة من JWT claim مباشرة — صفر وصول لجداول
CREATE FUNCTION my_institution() RETURNS UUID
LANGUAGE sql STABLE AS
$$ SELECT (auth.jwt() ->> 'institution_id')::uuid $$;

-- دوال مساعدة (SECURITY DEFINER + search_path مثبّت لمنع الانتحال)
CREATE FUNCTION my_role() RETURNS user_role
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public AS
$$ SELECT role FROM profiles WHERE id = auth.uid() $$;

CREATE FUNCTION is_enrolled(cid BIGINT) RETURNS BOOLEAN
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public AS
$$ SELECT EXISTS (SELECT 1 FROM enrollments
                  WHERE course_id = cid AND student_id = auth.uid()
                  AND status = 'ACTIVE') $$;

CREATE FUNCTION teaches(cid BIGINT) RETURNS BOOLEAN
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public AS
$$ SELECT EXISTS (SELECT 1 FROM course_lecturers
                  WHERE course_id = cid AND lecturer_id = auth.uid()) $$;

-- (v1.6 §3) عائلة الدوال الأساسية: أنظمة متعددة (الجدولة، الدفعات) ستبني عليها.
-- آمنة في بيئة multi-tenant لأنها مبنية على auth.uid() — لا تحتاج تمرير institution_id
CREATE OR REPLACE FUNCTION has_active_subscription()
RETURNS boolean AS $$
  SELECT EXISTS (
    SELECT 1 FROM subscription_status
    WHERE student_id = auth.uid() AND paid_until >= CURRENT_DATE
  )
$$ LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public;

-- النمط الإلزامي: كل سياسة على جدول tenant-scoped تبدأ بفحص المؤسسة
-- مثال — جدول الطالب: أحداث مؤسسته فقط، ومع سياسة الحجب المالي (قرار #15 + v1.6 §3):
-- عند تأخر الاشتراك يختفي كل شيء من الجدول باستثناء بطاقات الامتحانات،
-- ويبقى تسجيل حضور الامتحان ودخوله فعّالين بالكامل.
-- Server-side إلزامياً — أي قفل في الواجهة وحده عديم القيمة أمنياً.
CREATE POLICY events_visibility ON events FOR SELECT
    USING (
        institution_id = my_institution()
        AND deleted_at IS NULL
        AND (
            my_role() IN ('admin','lecturer')
            OR type = 'exam'
            OR has_active_subscription()
        )
    );

-- [قرار #7] الطالب يرى نتيجته المعتمدة فقط
CREATE POLICY student_own_final ON final_results FOR SELECT
    USING (student_id = auth.uid() OR my_role() = 'admin'
           OR (my_role() = 'lecturer' AND teaches(course_id)));

-- بنك الأسئلة: محاضرو المادة والإداري فقط — والمؤسسة نفسها حصراً
CREATE POLICY qbank_lecturers ON questions FOR ALL
    USING (institution_id = my_institution()
           AND (my_role() = 'admin' OR teaches(course_id)))
    WITH CHECK (institution_id = my_institution() AND teaches(course_id));

-- التسجيلات: الطالب يرى تسجيلاته ACTIVE فقط؛ الكتابة للإداري حصراً (v1.6 §2.1)
CREATE POLICY enroll_student_read ON enrollments FOR SELECT
    USING (institution_id = my_institution()
           AND (student_id = auth.uid() AND status = 'ACTIVE'
                OR my_role() IN ('admin','lecturer')));
CREATE POLICY enroll_admin_write ON enrollments FOR INSERT
    WITH CHECK (institution_id = my_institution() AND my_role() = 'admin');
CREATE POLICY enroll_admin_update ON enrollments FOR UPDATE
    USING (institution_id = my_institution() AND my_role() = 'admin');
-- لا سياسة DELETE لأحد: الحذف الفيزيائي لسجلات enrollments ممنوع (v1.6 §4)

-- [قرار #16] الدردشة: الطالب يقرأ منشورات مواده، والمحاضر وحده ينشئ منشوراً
CREATE POLICY chat_read ON chat_posts FOR SELECT
    USING (EXISTS (SELECT 1 FROM chat_channels c WHERE c.id = channel_id
                   AND c.institution_id = my_institution()
                   AND (is_enrolled(c.course_id) OR teaches(c.course_id)
                        OR my_role() = 'admin')));
CREATE POLICY chat_post_lecturer_only ON chat_posts FOR INSERT
    WITH CHECK (author_id = auth.uid()
                AND EXISTS (SELECT 1 FROM chat_channels c WHERE c.id = channel_id
                            AND teaches(c.course_id)));

-- الملاحظات الإدارية: غير مرئية لصاحبها نهائياً (MVP §5.4)
CREATE POLICY notes_staff_only ON admin_notes FOR ALL
    USING (my_role() IN ('admin','lecturer') AND subject_id <> auth.uid());

-- الدفعات: الطالب يرى ويضيف دفعاته فقط؛ المراجعة للإداري حصراً
CREATE POLICY pay_student ON payment_requests FOR SELECT
    USING (institution_id = my_institution()
           AND (student_id = auth.uid() OR my_role() = 'admin'));
CREATE POLICY pay_insert ON payment_requests FOR INSERT
    WITH CHECK (institution_id = my_institution() AND student_id = auth.uid());
CREATE POLICY pay_review ON payment_requests FOR UPDATE
    USING (institution_id = my_institution() AND my_role() = 'admin');
\`\`\`

### 4.3 — استراتيجية الفهرسة (Indexing Strategy)

القاعدة الحاكمة (v1.6 §1.3): **كل فهرس أساسي على جدول tenant-scoped يبدأ بـ `institution_id`.**

| الفهرس | الاستعلام الذي يخدمه | تكلفته على الكتابة |
|---|---|---|
| `events (institution_id, course_id, starts_at) WHERE deleted_at IS NULL` | جدول الطالب الأسبوعي (أثقل استعلام — 0.2 س5) | منخفضة: كتابات الأحداث نادرة |
| `events (institution_id, updated_at)` | المزامنة التفاضلية [D-06] | منخفضة |
| GiST على `events (room_id, tstzrange)` | كشف تعارض القاعات آلياً (MVP §5.1) | متوسطة: مقبولة لندرة إدراج الأحداث |
| `enrollments (institution_id, student_id, course_id)` | فحوصات `is_enrolled` في كل سياسة RLS تقريباً | منخفضة: كتابات التسجيل إدارية ونادرة |
| `attendance_records (institution_id, session_id)` | العدّاد الفوري والقائمة الحية للمحاضر | متوسطة: على مسار كتابة الذروة — لكنه ضروري |
| `attendance_records (institution_id, student_id)` | مراجعة الحضور لكل طالب (MVP §4.3.5) | متوسطة: فهرسان فقط على هذا الجدول، لا أكثر |
| `notifications (user_id, is_read, created_at DESC)` | شاشة الإشعارات مع الفلاتر | متوسطة: فهرس مركّب واحد يكفي كل الفلاتر |
| `payment_requests (institution_id, status, channel) WHERE status='pending'` | لوحة قيادة الإداري — فهرس جزئي صغير جداً | شبه معدومة |
| `exam_attempts (institution_id, exam_id)` | مراقبة تقدم الامتحان الجاري | منخفضة |

### 4.4 — الحذف والأرشفة والتوسع (تثبيت مبدأ v1.6 §4)

- **الترحيلات (Migrations):** ملفات SQL مرقّمة تسلسلياً تُطبَّق عبر CI. إضافات فقط أثناء التشغيل (عمود nullable ثم backfill ثم قيد) — ممنوع منعاً باتاً أي `ALTER` هدّام خلال نافذة امتحان مجدولة **لأي من المؤسسات العشر**.
- **الحذف: Soft delete دائماً** (`deleted_at`) للسجلات الأكاديمية — بلا استثناء. `enrollments` تنتقل إلى COMPLETED ولا تُحذف أبداً؛ **كشف الدرجات يجب أن يبقى قابلاً للاستخراج بعد سنوات.**
- **Hard delete** حصراً عبر pg_cron للجداول العملياتية عديمة القيمة طويلة المدى: `emergency_tokens`، جداول provisioning المعلّقة، سجلات rate limiting — والملفات الفعلية في Storage بعد 90 يوماً من soft delete.
- **التوسع:** عند تجاوز `attendance_records` أو `notifications` عشرة ملايين صف: partitioning على `term_id`/الشهر. مفتاح sharding المستقبلي جاهز أصلاً: `institution_id`.
- **التوافقية:** **Strict** (معاملات ذرية) في: الحضور، محاولات الامتحان، اعتماد النتائج (بما فيه تحويل enrollments إلى COMPLETED)، استرداد رمز الطوارئ، قفل الأوزان، استيراد CSV. **Eventual مقبولة** في: العدّاد اللحظي، الإشعارات، منشورات الدردشة.

---

## القسم 5: عقود الـ API والواجهات 🔌

### 5.1 — مبادئ التصميم

**[D-07] PostgREST (توليد تلقائي محكوم بـ RLS) للقراءات، وRPC لكل عملية كتابة متعددة الخطوات.**

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **PostgREST + RPC ✅** | صفر endpoints يدوية للقراءة · RLS تحكم كل شيء | RPC بـ PL/pgSQL أقل ألفة | اتساق أمني كامل مع [D-05] |
| REST يدوي كامل ❌ | حرية شكل الاستجابة | كل endpoint = فرصة نسيان تحقق | مرفوض للسبب ذاته |
| GraphQL ❌ | استعلامات مرنة | تعقيد authorization لكل حقل + N+1 | أشكال البيانات معروفة وثابتة |

- **النسخ:** الدوال تُنسَّخ بالاسم (`start_exam_attempt_v2`) عند تغيير كاسر؛ القديمة تبقى فصلاً كاملاً.
- **صيغة الخطأ الموحدة:** `{"error": {"code": "EXAM_WINDOW_CLOSED", "message_ar": "..."}}`

### 5.2 — توثيق أهم الـ RPCs

#### `POST /rest/v1/rpc/record_attendance`
**الغرض:** تسجيل حضور idempotent | **المصادقة:** JWT طالب | **Rate Limit:** 10/دقيقة/مستخدم

**المدخلات:**
\`\`\`json
{
  "session_id": "bigint — مطلوب",
  "client_uuid": "uuid — مطلوب، يولَّد على الجهاز مرة واحدة لكل تسجيل",
  "method": "string — أحد: ble | agent | qr",
  "proof": "object — حمولة التحقق (تفصيلها في Attendance TDD)"
}
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{ "recorded": true, "recorded_at": "ISO-8601" }
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | proof غير صالح أو حقل ناقص | رسالة تحقق + إبقاء السجل في طابور المحاولة |
| 401 | توكن منتهٍ | تجديد الجلسة ثم إعادة الإرسال من الطابور |
| 403 | غير مسجل ACTIVE بمادة الحدث / الجلسة مغلقة | زر «تقديم التماس عذر» (MVP §3.5.1) |
| 409 | تسجيل سابق موجود | اعتباره نجاحاً — حذف من الطابور (idempotent) |
| 429 | تجاوز الحد | retry مع exponential backoff |
| 500 | خطأ داخلي | إبقاء في الطابور وإعادة المحاولة |

#### `POST /rest/v1/rpc/start_exam_attempt`
**الغرض:** فتح محاولة بتوقيت الخادم + snapshot الأسئلة منزوعة الإجابات | **المصادقة:** JWT طالب | **Rate Limit:** 5/دقيقة

**المدخلات:**
\`\`\`json
{ "exam_id": "bigint — مطلوب" }
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{
  "attempt_id": "bigint",
  "ends_at": "ISO-8601 — توقيت الخادم، العدّاد المحلي عرض فقط",
  "questions": [ { "id": 1, "type": "mcq", "body": { "...": "بدون answer_key نهائياً" } } ]
}
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | exam_id غير موجود | رسالة خطأ عامة |
| 401 | توكن منتهٍ | إعادة توجيه لتسجيل الدخول |
| 403 | خارج النافذة / غير مسجل ACTIVE / لم يسجل حضور الامتحان | إخفاء زر «دخول الامتحان» |
| 409 | محاولة سابقة | غير مسلَّمة وضمن الوقت: استئناف بذات ends_at؛ مسلَّمة: «تم التسليم» |
| 429 | تجاوز الحد | backoff |
| 500 | خطأ داخلي | إعادة محاولة — idempotent |

**ملاحظات (v1.6 §5.2):** `submit_attempt` و`save_answer` يرفضان أي طلب بعد `ends_at` بسماحية 30 ثانية لتذبذب الشبكة فقط. انتهاء الوقت أثناء انقطاع الاتصال: قفل الواجهة محلياً + إغلاق المحاولة server-side عبر pg_cron عند `ends_at` بآخر إجابات محفوظة (نفس نمط جلسات الحضور). العدّاد مبني على وقت بدء صادر من السيرفر — ليس ساعة الجهاز.

#### `POST /rest/v1/rpc/import_enrollments_csv` — جديد (v1.6 §2.2)
**الغرض:** تسجيل بالجملة يربط قائمة طلاب بمادة واحدة دفعة واحدة | **المصادقة:** JWT إداري حصراً | **Rate Limit:** 5/دقيقة

**المدخلات:**
\`\`\`json
{
  "course_id": "bigint — مطلوب",
  "rows": [ { "university_id": "string — الرقم الجامعي للطالب" } ]
}
\`\`\`

**قواعد التحقق لكل صف (تُنفَّذ داخل الدالة، لا في العميل):**
- الطالب موجود وغير محذوف (`deleted_at IS NULL`)
- الطالب ينتمي لنفس `institution_id` الخاص بالإداري الرافع — **حرج في بيئة 10 مؤسسات: ملف CSV خاطئ لا يجوز أن يُسرِّب طالباً عبر حدود مؤسسة أخرى تحت أي ظرف**
- المادة تنتمي لنفس المؤسسة ولنفس الفصل النشط
- لا يوجد تسجيل ACTIVE مسبق لنفس `(student_id, course_id)`

**المخرجات الناجحة (200) — تقرير سطر-بسطر، لا فشل كلي صامت ولا نجاح جزئي صامت:**
\`\`\`json
{
  "total": 120,
  "succeeded": 117,
  "failed": 3,
  "report": [
    { "row": 14, "university_id": "20231045", "ok": false, "reason": "ALREADY_ENROLLED" },
    { "row": 55, "university_id": "20239999", "ok": false, "reason": "STUDENT_NOT_FOUND" },
    { "row": 88, "university_id": "20231777", "ok": false, "reason": "WRONG_INSTITUTION" }
  ]
}
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | صيغة الملف غير صالحة / المادة ليست في الفصل النشط | رسالة تحقق قبل أي إدراج |
| 403 | الفاعل ليس إدارياً / المادة لمؤسسة أخرى | مرفوض من RLS |
| 429 | تجاوز الحد | backoff |
| 500 | خطأ داخلي | المعاملة ذرية لكل صف — إعادة الرفع آمنة (الصفوف الناجحة سابقاً تُرجع ALREADY_ENROLLED) |

#### `POST /rest/v1/rpc/review_absence_excuse`
**الغرض:** قبول/رفض التماس — القبول يسجّل الحضور تلقائياً معاملاتياً (قرار #17) | **المصادقة:** JWT محاضر معيَّن على المادة | **Rate Limit:** 60/دقيقة

**المدخلات:**
\`\`\`json
{
  "excuse_id": "bigint — مطلوب",
  "decision": "string — accepted | rejected",
  "rejection_reason": "string — إلزامي عند الرفض، تتحقق منه الدالة"
}
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{ "status": "accepted", "attendance_recorded": true }
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | رفض بلا سبب | إلزام حقل السبب في الواجهة |
| 401 | توكن منتهٍ | إعادة تسجيل دخول |
| 403 | المحاضر غير معيَّن على مادة الحدث | إخفاء الالتماس أصلاً (RLS) |
| 409 | الالتماس مُراجَع مسبقاً | تحديث العرض بالحالة الحالية |
| 500 | خطأ داخلي | إعادة محاولة — المعاملة ذرية فلا حالة وسطى |

#### `POST /rest/v1/rpc/generate_emergency_token` / `redeem_emergency_token`
**الغرض:** الدخول الطارئ (قرار #2) | **المصادقة:** التوليد: JWT إداري حصراً · الاسترداد: بلا مصادقة (هذا غرضه) | **Rate Limit:** الاسترداد: **5 محاولات/دقيقة/IP**

**مدخلات الاسترداد:**
\`\`\`json
{ "token": "string — 32 بايت عشوائي (base64url) من QR" }
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{ "access_token": "JWT بصلاحيات المستخدم نفسه + institution_id claim", "expires_in": 43200 }
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | صيغة توكن غير صالحة | «رمز غير صالح — راجع الإداري» |
| 401 | منتهٍ / مستخدم سابقاً / غير موجود — **رسالة موحدة عمداً** | نفس الرسالة أعلاه |
| 429 | تجاوز حد المحاولات | قفل مؤقت 5 دقائق |
| 500 | خطأ داخلي | إعادة محاولة |

**ملاحظة (v1.6 §5.5):** بعد الدخول المؤقت عبر QR الطارئ، تغيير حساب Google يتم عبر `admin-update-expected-email` — المسار الدائم المحسوم. لا حاجة لحل جديد.

#### `POST /rest/v1/rpc/approve_final_results`
**الغرض:** اعتماد نتيجة مادة — يجمّد الأوزان وينشر النتائج **ويحوّل تسجيلات المادة من ACTIVE إلى COMPLETED** ذرياً في نفس المعاملة (MVP §5.6.1 + v1.6 §2.5) | **المصادقة:** JWT إداري حصراً | **Rate Limit:** 10/دقيقة

**المدخلات:**
\`\`\`json
{ "course_id": "bigint — مطلوب" }
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{
  "approved": true,
  "students_count": 120,
  "enrollments_completed": 120,
  "weights_locked_at": "ISO-8601"
}
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | أوزان لا تساوي 100 أو درجات ناقصة | عرض قائمة النواقص المُرجعة من الدالة |
| 401 | توكن منتهٍ | إعادة تسجيل دخول |
| 403 | الفاعل ليس إدارياً | مرفوض من RLS قبل الوصول للدالة |
| 409 | النتيجة معتمدة مسبقاً | عرض تاريخ الاعتماد السابق |
| 500 | خطأ داخلي | إعادة محاولة — المعاملة ذرية: إما كل شيء أو لا شيء |

### 5.3 — العقود بين الخدمات الداخلية

لا خدمات متعددة [D-05] — التنسيق الداخلي الوحيد هو **إشعارات مولّدة بـ triggers**. الضمانة: **at-least-once** — العميل يزيل التكرار بمعرّف الإشعار.

---

## القسم 6: حالات الحافة، أنماط الفشل، والأمان 🛡️

### 6.1 — جرد حالات الحافة (Edge Case Inventory)

| السيناريو | ماذا يحدث في التصميم؟ | المعالجة |
|---|---|---|
| محاضران يعدلان نفس السؤال معاً | تعارض إصدارات | عمود `version` + optimistic locking (v1.6 §5.3): نسخة العميل الأقدم تُرفض برسالة "تم تعديل هذا السؤال من زميل" + `question_audit_log` يحفظ snapshot |
| تعديل سؤال أثناء امتحان جارٍ | خطر تغيّر السؤال تحت أقدام الطالب | `questions_snapshot` في المحاولة لحظة السحب — التعديل لا يمس محاولة جارية أو مُسلَّمة (v1.6 §5.3) |
| ملف CSV يتضمن طالباً من مؤسسة أخرى | خطر تسريب عبر حدود المؤسسات | فحص `institution_id` لكل صف داخل `import_enrollments_csv` + سياسة RLS كطبقة ثانية — الصف يفشل بـ WRONG_INSTITUTION في التقرير |
| إداريان يعتمدان نتيجة نفس المادة معاً | سباق محتمل | `SELECT ... FOR UPDATE` على صف `grade_weights` — الثاني يستلم 409 |
| مدخلات بحجم 10x المتوقع | مرفوضة على مستوى المحرك | CHECK على أطوال النصوص وحجم الملف (50MB) — لا اعتماد على تحقق العميل |
| انقطاع الشبكة في منتصف كتابة | خطر تكرار عند إعادة الإرسال | `client_uuid` + `ON CONFLICT DO NOTHING` للحضور؛ UPSERT بمفتاح `(question_id, attempt_id)` لإجابات الامتحان (v1.6 §5.2) — كل المسارات الحرجة idempotent |
| خدمة Google OAuth بطيئة | تعليق شاشة الدخول | timeout 10 ثوانٍ + خيار «دخول طارئ عبر الإداري» بعد فشلين |
| التوقيت (timezones / انحراف ساعة الجهاز) | خطر غش بتغيير الساعة | كل الأعمدة `TIMESTAMPTZ` وكل المقارنات بـ `now()` على الخادم حصراً؛ ساعة الجهاز للعرض فقط |
| طالب متأخر عن الاشتراك (قرار #15 + v1.6 §3) | يجب حجبه إلا عن الامتحانات | `has_active_subscription()` في سياسات RLS: يختفي كل شيء من الجدول (محاضرات، معامل، سمنارات، واجبات، بحوث، ثقافية) **عدا بطاقات الامتحانات**، ويبقى تسجيل حضور الامتحان ودخوله فعّالين بالكامل |
| طالب يفتح الامتحان من جهازين | خطر محاولتين | `UNIQUE(exam_id, student_id)` — الجهاز الثاني يستأنف نفس المحاولة بنفس `ends_at` |
| انتهاء وقت الامتحان أثناء انقطاع الاتصال | خطر محاولة معلّقة للأبد | قفل الواجهة محلياً + إغلاق المحاولة server-side عبر pg_cron عند `ends_at` (v1.6 §5.2) |
| مزامنة أوفلاين تصل بعد إغلاق جلسة الحضور | قبولها الأعمى = ثغرة؛ رفضها الأعمى = ظلم | تُقبل ضمن سماحية `closes_at + 15 دقيقة` إذا كان `proof` صالحاً زمنياً؛ بعدها 403 ويوجَّه الطالب للالتماس |
| 10 امتحانات جماعية في نفس الساعة عبر المؤسسات | ذروة 20,000 متزامن | التصميم يفترض التداخل الكامل (لا آلية تنسيق بين المؤسسات)؛ Supavisor + مراجعة Dedicated Compute المُجدولة (D-04)؛ التشغيل يراقب الواقع |

### 6.2 — تحليل أنماط الفشل (Failure Mode Analysis)

| المكون | نمط الفشل | الاحتمالية | الأثر | الكشف | التعافي |
|---|---|---|---|---|---|
| PostgreSQL | تعطل كامل | منخفضة | 🔴 حرج (كارثي أثناء امتحان لأي مؤسسة) | health check كل 10s | failover مُدار، RTO ≤ 5 دقائق + تمديد وقت الامتحان يدوياً |
| PostgreSQL | استنفاد connections في ذروة امتحانات متداخلة | متوسطة | 🟠 عالٍ | مراقبة `pg_stat_activity` بعتبة 80% | Supavisor (transaction mode) مفعّل من اليوم الأول (D-04) |
| Auth (Google) | انقطاع خارجي | منخفضة | 🟡 متوسط | فشل متكرر للدخول | الجلسات القائمة تستمر + مسار QR الطارئ مستقل تماماً |
| Realtime | انقطاع البث / تجاوز حصة الاتصالات | متوسطة | 🟢 منخفض | heartbeat من العميل | fallback إلى polling كل 60 ثانية + رفع الحصة قبل أول امتحان جماعي (D-04) |
| Storage | فشل رفع إشعار دفع | متوسطة | 🟡 متوسط | خطأ للعميل | طابور إعادة رفع محلي + إعادة الرفع صراحة (MVP §3.7.3) |
| pg_cron | توقف مهام التنظيف/الختم الآلي | منخفضة | 🟡 متوسط | مقياس «عمر أقدم رمز غير محذوف» | التحقق من `expires_at`/`ends_at` يتم عند كل عملية أيضاً — التنظيف دفاع ثانٍ لا أساسي |

### 6.3 — نموذج التهديدات (STRIDE)

| التهديد | مثال ملموس | الدفاع المحدد |
|---|---|---|
| Spoofing | تخمين رمز دخول طارئ | توكن 32 بايت (~10⁷⁷ احتمال) · hash فقط · 180 ثانية · استخدام واحد ذري · 5/دقيقة/IP · رسالة فشل موحدة |
| Spoofing | حساب Google بنفس اسم الطالب | الربط عبر `expected_email` المُدخل إدارياً مسبقاً حصراً — لا ربط ذاتي أبداً |
| Spoofing | JWT بـ `institution_id` مزوّر | الـ claim يُحقَن server-side عند إصدار التوكن ويُوقَّع؛ العميل لا يستطيع تعديله |
| Tampering | تعديل درجة أو وزن بعد الاعتماد | trigger يرفض أي UPDATE بعد `locked_at` — على مستوى المحرك (قرار #4) |
| Tampering | تزوير توقيت تسليم الامتحان | كل الأزمنة `now()` على الخادم؛ `ends_at` محسوب ومخزّن عند البدء |
| Repudiation | محاضر ينكر تعديل سؤال زميله | `question_audit_log` + عمود `version` — RLS تمنع DELETE على الجميع |
| Information Disclosure | تسريب `answer_key` قبل الامتحان | العمود لا يغادر القاعدة أبداً: الطالب بلا SELECT على `questions`، وRPC تُرجع أعمدة منتقاة |
| Information Disclosure | بيانات مؤسسة تظهر لمؤسسة أخرى | `institution_id = my_institution()` في كل سياسة + فهارس مركّبة + مسارات Storage معزولة + فحص CSV لكل صف |
| Information Disclosure | صورة إشعار بنكي على رابط عام | bucket خاص + Signed URL بصلاحية 5 دقائق لصاحب الدفعة أو الإداري فقط |
| Information Disclosure | طالب يرى درجات زملائه | RLS على `final_results` و`submission_grades` بشرط `student_id = auth.uid()` |
| Denial of Service | إغراق `redeem_emergency_token` أو `record_attendance` | rate limits لكل RPC + Supavisor + حدود حجم الطلب |
| Elevation of Privilege | طالب يعدّل `role` أو `institution_id` في صفه | لا سياسة UPDATE على هذه الأعمدة لغير الإداري؛ دوال `SECURITY DEFINER` بـ `search_path` مثبّت |

### 6.4 — قائمة تدقيق أمنية إلزامية

- [x] **المصادقة والتفويض:** Google OAuth → JWT (`auth.uid()` + `institution_id` claim) → RLS؛ التحقق في قاعدة البيانات حصراً — أي قفل في الواجهة وحده عديم القيمة أمنياً
- [x] **تشفير البيانات:** in-transit: TLS 1.2+ · at-rest: AES-256
- [x] **إدارة الأسرار:** `service_role key` في أسرار الخادم فقط؛ العميل يحمل `anon key` المحكوم بـ RLS
- [x] **حقن SQL:** PostgREST مُعلمَن بالكامل؛ ممنوع `EXECUTE` بنص مركّب
- [x] **سجلات التدقيق:** تعديلات بنك الأسئلة، قرارات الدفعات، اعتماد النتائج، رموز الطوارئ، مخالفات الامتحان، **عمليات الاستيراد بالجملة (من رفع، كم صف، كم فشل)**. لا يُسجَّل أبداً: التوكنات الخام، محتوى صور الإشعارات البنكية

### 6.5 — الملاحظة والمراقبة (Observability)

| المقياس | عتبة التنبيه |
|---|---|
| زمن فحص RLS لأي سياسة | > 100ms |
| p95 لاستعلام جدول الطالب | > 300ms لمدة 5 دقائق متصلة |
| نسبة connections المستهلكة (عبر Supavisor) | > 80% |
| معدل فشل `record_attendance` | > 2% خلال نافذة جلسة مفتوحة |
| محاولات استرداد رمز طارئ فاشلة | > 20/ساعة من نفس الـ IP |
| عمر أقدم رمز طارئ غير محذوف | > 10 دقائق (تعطل التنظيف) |
| استهلاك تخزين المؤسسة الواحدة | تجاوز الحصة الإدارية المقررة (D-04) |
| عدد المؤسسات النشطة فعلياً | = 4 → إطلاق مراجعة Dedicated Compute المُجدولة |

---

## القسم 7: خطة التنفيذ وخارطة الطريق 🗺️

### 7.1 — ترتيب المخاطر (Risk-First Ordering)

**أخطر افتراض تقني:** أن RLS (بفحص مؤسسة + دور) + RPC تستوعب دفعة دخول امتحان (2,000 طالب/60 ثانية مع الحفظ التدريجي) دون تجاوز p95 = 300ms وفحص RLS = 100ms. لو فشل، تتغير المعمارية جذرياً ([D-05]) — لذلك هو موضوع المرحلة 0 حصراً.

### 7.2 — المراحل

#### المرحلة 0: إثبات الجدوى (Spike) — أسبوع واحد
- **الهدف:** التحقق من الافتراض أعلاه في بيئة multi-tenant
- **المخرج:** جداول `institutions/profiles/exams/exam_attempts/exam_answers` + `start_exam_attempt` + `save_answer` مع RLS كاملة (مؤسسة + دور)، واختبار حمل k6 يحاكي 2,000 طالب متزامن لمؤسسة واحدة + خلفية تحاكي نشاط 9 مؤسسات أخرى
- **معيار النجاح/الفشل:** p95 < 300ms وصفر فقدان كتابة → متابعة · فشل → الخطة البديلة: نقل مسار الامتحان فقط إلى خدمة Go وسيطة مع إبقاء بقية النظام على [D-05]

#### المرحلة 1: الهوية والهيكل الأكاديمي والتسجيل — 3 أسابيع
- **المهام:** (1) institutions/profiles/terms/courses/enrollments/course_lecturers مع الفهارس المركّبة (2) تدفق Google OAuth + حقن `institution_id` claim + ربط expected_email (3) نظام Enrollment كاملاً: التسجيل الفردي + `import_enrollments_csv` بتقريره السطري (4) RPCs الدخول الطارئ + pg_cron (5) اختبارات pgTAP لعزل الأدوار **والمؤسسات** (6) شاشات إدارة الحسابات
- **الاعتماديات:** المرحلة 0
- **المخرج:** الأدوار الثلاثة من مؤسستين مختلفتين تسجل دخولاً وترى بيانات مؤسستها فقط؛ استيراد CSV يرفض الصف العابر للمؤسسات؛ الدخول الطارئ يفنى خلال 180 ثانية
- **اختبارات:** pgTAP لكل سياسة × دور × مؤسسة · تكامل CSV بملف مختلط متعمد · دورة رمز الطوارئ كاملة

#### المرحلة 2: الجدولة والملفات والإشعارات — 3 أسابيع
- **المهام:** (1) events بأنواعها السبعة + قيد تعارض القاعات (2) room_requests وتدفق الموافقة (3) files (حد 50MB، مسارات `{institution_id}/`) والاجتهادات (4) notifications + triggers (5) delta sync مع Hive (6) `has_active_subscription()` وسياسة الحجب في عرض الأحداث
- **الاعتماديات:** المرحلة 1
- **المخرج:** طالب يرى جدوله الأسبوعي المفلتر ويعمل دون اتصال؛ طالب متأخر مالياً يرى بطاقات الامتحانات فقط
- **اختبارات:** unit للقيود · تكامل تعارض القاعات · مزامنة أوفلاين · الحجب الانتقائي

#### المرحلة 3: الحضور والالتماسات — أسبوعان
- **المهام:** (1) attendance_sessions/records (2) `record_attendance` بمساراتها (3) absence_excuses + `review_absence_excuse` المعاملاتية (4) العدّاد اللحظي عبر Realtime
- **الاعتماديات:** المرحلة 2
- **المخرج:** جلسة حضور كاملة بمزامنة أوفلاين وصفر تكرار؛ قبول التماس يسجّل حضوراً تلقائياً
- **اختبارات:** حمل على دفعة الحضور · idempotency بإعادة إرسال متعمدة · ربط الالتماس/الحضور

#### المرحلة 4: بنك الأسئلة والامتحانات — 3 أسابيع
- **المهام:** (1) questions مع `version` + optimistic locking + audit trigger (2) exclusions (3) دوال المحاولة مع `questions_snapshot` والتسليم والختم الآلي (4) exam_violations بإشارة `focus_loss` الموحدة (الملحق أ) + التنبيه الصوتي (5) الجدولة الإدارية
- **الاعتماديات:** المرحلتان 0 و3
- **المخرج:** امتحان تجريبي كامل مع رصد المخالفات، وتعديل سؤال أثناء الامتحان لا يؤثر على المحاولات
- **اختبارات:** أمنية: انتزاع answer_key من كل مسار · زمنية: رفض الكتابة بعد ends_at · optimistic locking بتعديلين متزامنين

#### المرحلة 5: الدفعات — أسبوعان
- **المهام:** (1) payment_requests بالقناتين + bucket خاص معزول بالمؤسسة (2) لوحة قيادة الإداري (3) subscription_status وربطه بـ `has_active_subscription()` (4) إشعارات التأكيد/الرفض
- **الاعتماديات:** المرحلتان 1 و2
- **المخرج:** دورة دفع كاملة؛ الحجب الانتقائي يعمل end-to-end
- **اختبارات:** الحجب الانتقائي · أمنية: وصول لصورة إشعار طالب آخر أو مؤسسة أخرى → مرفوض

#### المرحلة 6: الأوزان والنتائج والتقارير — أسبوعان
- **المهام:** (1) grade_weights + trigger القفل + قيد مجموع 100% (2) submission_grades وواجهتا التصحيح (3) `approve_final_results` مع تحويل enrollments إلى COMPLETED ذرياً (4) تصدير PDF الأربعة
- **الاعتماديات:** المراحل 3 و4 و5
- **المخرج:** اعتماد مادة كاملة: قفل الأوزان + نشر النتائج + COMPLETED + PDF صحيح
- **اختبارات:** صحة الاحتساب · ذرية الاعتماد (نتيجة + enrollment في معاملة واحدة) · رفض تعديل وزن بعد القفل

#### المرحلة 7: الدردشة والإعلانات — أسبوعان
- **المهام:** (1) القنوات والمنشورات والتعليقات + RLS «قراءة + تعليق» (2) Realtime + fallback (3) البورد الأسود بالجدولة والتثبيت (4) الإشعارات الموجّهة للمتأخرين مالياً ومتكرري الغياب
- **الاعتماديات:** المرحلة 1 (والمرحلتان 3 و5 للإشعارات الموجّهة)
- **المخرج:** قناة مادة حية: المحاضر ينشر، الطالب يعلّق فقط
- **اختبارات:** أمنية: طالب ينشئ منشوراً أو يقرأ قناة مؤسسة أخرى → مرفوض من RLS

### 7.3 — مخطط الاعتماديات

\`\`\`mermaid
graph LR
    P0[مرحلة 0: Spike الامتحانات multi-tenant] --> P1[مرحلة 1: الهوية والتسجيل]
    P1 --> P2[مرحلة 2: الجدولة والملفات]
    P2 --> P3[مرحلة 3: الحضور]
    P0 --> P4[مرحلة 4: الامتحانات]
    P3 --> P4
    P1 --> P5[مرحلة 5: الدفعات]
    P2 --> P5
    P3 --> P6[مرحلة 6: النتائج]
    P4 --> P6
    P5 --> P6
    P1 --> P7[مرحلة 7: الدردشة والإعلانات]
\`\`\`

### 7.4 — تعريف "الانتهاء" (Definition of Done)

- [ ] كل الاختبارات تمر (تغطية ≥ 90% للمسارات الحرجة: الحضور، الامتحانات، الاعتماد، الدخول الطارئ، استيراد CSV)
- [ ] كل جدول عليه RLS مفعّلة مع اختبار pgTAP لكل **دور × عملية × مؤسسة**
- [ ] توثيق كل RPC محدّث بكامل أكواد الخطأ
- [ ] مقاييس القسم 6.5 تعمل والتنبيهات مضبوطة بعتبات موثقة
- [ ] قائمة الأمان (6.4) مُراجعة بالكامل من شخصين مختلفين
- [ ] خطة rollback لكل ترحيل موثقة ومجرّبة على staging
- [ ] اختبار الحمل النهائي: 2,000 طالب متزامن لمؤسسة واحدة على مسار الامتحان مع نشاط خلفي لبقية المؤسسات بنجاح
- [ ] حصة Realtime مرفوعة ومؤكدة من Supabase قبل أول امتحان جماعي فعلي (D-04)

---

## الملحق أ: قرارات مسجَّلة لأنظمة لاحقة (من v1.6 §5 — تُنفَّذ في وثائقها، تُوثَّق هنا لمنع الضياع)

### أ.1 — الامتحانات: إشارة فقدان التركيز (تصحيح القرار #13)

- رصد "سحب ستارة الإشعارات" غير موثوق على Android بدون Accessibility Service (صلاحية ثقيلة قد ترفضها Google Play)، ومستحيل على iOS.
- **القرار:** دمج المخالفتين (سحب الستارة + الخروج من التطبيق) في إشارة واحدة: **فقدان تركيز التطبيق > 5 ثوانٍ** عبر `WidgetsBindingObserver` في Flutter — تعمل على المنصتين بلا صلاحيات إضافية.
- النسخ واللصق تبقى إشارة منفصلة.
- انعكاسه على هذه الوثيقة: `violation_type` أصبح `('focus_loss','copy_paste')` بدل الأنواع الثلاثة القديمة.

### أ.2 — الامتحانات: الانقطاع والمزامنة

- كل إجابة تُحفَظ في Hive فور اختيارها (ليس عند التسليم فقط).
- العدّاد مبني على وقت بدء صادر من السيرفر (ليس ساعة الجهاز).
- المزامنة عند عودة الاتصال: upsert idempotent بمفتاح `(question_id, attempt_id)` — لا إدراج مكرر عند إعادة المحاولة.
- انتهاء الوقت أثناء الانقطاع: قفل الواجهة محلياً + إغلاق المحاولة server-side عبر pg_cron عند `ends_at` (نفس نمط جلسات الحضور).

### أ.3 — بنك الأسئلة

- عمود `version` رقمي لكل سؤال + optimistic locking: نسخة العميل الأقدم تُرفض برسالة "تم تعديل هذا السؤال من زميل".
- **Snapshot عند السحب:** نص السؤال وخياراته يُنسَخان داخل سجل محاولة الطالب لحظة السحب العشوائي — تعديل السؤال لاحقاً لا يمس محاولة جارية أو مُسلَّمة.

### أ.4 — أوزان الدرجات

- المجموع = 100% **إلزامي** — يُرفض الحفظ إن لم يتحقق بالضبط (قيد CHECK على مستوى المحرك).
- الأوزان تُطبَّق مرة واحدة فقط لحظة اعتماد النتيجة (القفل)؛ قبلها تعديلها بلا أي أثر جانبي لأن الدرجات الفردية مخزَّنة مستقلة.

### أ.5 — تغيير حساب Google

- محسوم: `admin-update-expected-email` هو المسار الدائم بعد الدخول المؤقت عبر QR الطارئ. لا حاجة لحل جديد.

---

## مصفوفة التوافق مع وثيقة MVP v3.4 + تعديل v1.6 (تحقق نهائي)

| القرار | أين طُبّق في هذه الوثيقة |
|---|---|
| MVP #1 مصادقة Google موحدة | [D-02] + profiles.expected_email |
| MVP #2 الدخول الطارئ QR (3 دقائق، مرة واحدة) | emergency_tokens + تسلسل 3.3 + عقد 5.2 |
| MVP #4 قفل الأوزان عند الاعتماد | grade_weights.locked_at + trigger + STRIDE/Tampering |
| MVP #5 الكتاب المشترك + الاستبعاد لكل امتحان | course_lecturers + question_audit_log + exclusions + version |
| MVP #6 الأحداث المتطلبة للحضور | attendance_sessions مرتبطة بالحدث |
| MVP #7 النتيجة المعتمدة فقط | final_results + RLS + عدم إرجاع الدرجة عند التسليم |
| MVP #10 جدولة الامتحان حصرية للإداري | CHECK على room_requests + فرض الدور في RPC |
| MVP #13 المراقبة (بصيغة v1.6 المصححة) | violation_type = focus_loss/copy_paste + الملحق أ.1 |
| MVP #14–15 قناتان + سياسة الحجب | payment_channel ENUM + has_active_subscription() + RLS انتقائية |
| MVP #16 قراءة + تعليق | سياسات chat_posts/chat_comments |
| MVP #17 القبول يسجّل الحضور تلقائياً | review_absence_excuse معاملاتية + method='excuse_auto' |
| v1.6 §1 نطاق 10 مؤسسات | الأقسام 0.3، 1.2، 2.2 [D-04]، 4.0 |
| v1.6 §1.3 قواعد RLS والفهارس المؤسسية | القسم 4.0 + كل الفهارس المركّبة + [D-08] |
| v1.6 §2 مواصفة Enrollment الكاملة | enrollments + enrollment_status + import_enrollments_csv + [D-09] |
| v1.6 §3 has_active_subscription() | الدالة في 4.2 + سياسة events_visibility |
| v1.6 §4 الحذف والأرشفة | القسم 4.4 |
| v1.6 §5 قرارات الأنظمة اللاحقة | الملحق أ + انعكاساتها في الـ schema |

*— نهاية الوثيقة — Core Data Layer TDD v2.1 (الموحّدة) —*