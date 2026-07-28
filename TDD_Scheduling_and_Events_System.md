# 📐 وثيقة التصميم التقني — نظام الجدولة والأحداث
## Academic Hub — Scheduling & Events System (TDD)

| البند | التفاصيل |
|---|---|
| **النظام** | نظام الجدولة والأحداث (Scheduling & Events) |
| **المنتج** | Academic Hub — MVP v3.6 |
| **إصدار الوثيقة** | 2.0 — مواءمة معمارية كاملة مع MVP v3.6 وTDD Core Data Layer v2.1 (الموحّدة) |
| **مستوى العمق المعتمد** | 🟡 Production |
| **الوثائق المرجعية المُلزمة** | MVP Functions v3.6 (الأقسام 1.2، 3.1، 4.1، 5.1، 5.2) · TDD Core Data Layer v2.1 (مالك الـ Schema) · سجل القرارات المعماري (docs/decision-log.md — D-01..D-06) · قالب TDD v1.0 |

---

## سجل الإصدارات (Revision History)

| الإصدار | طبيعة التغيير | أبرز التعديلات |
|---|---|---|
| 1.0 | إصدار أولي | تصميم مستقل على افتراض جامعة واحدة (~5,000 طالب) |
| 1.1 | مواءمة MVP v3.5 | إضافة الأحداث الثقافية (قرار #19) + توحيد افتراض القاعات/المعامل |
| **2.0** | **مواءمة معمارية شاملة** | (1) اعتماد **التعدد المؤسسي** بالكامل: `institution_id` في كل جدول، فهارس مركّبة تبدأ به، RLS من JWT claim (قرار #20 / D-08) · (2) **نقل ملكية الـ Schema** لجداول `events` و`rooms` و`room_requests` إلى Core Data TDD v2.1 — هذه الوثيقة تضيف فقط `event_series` و`event_outbox` وعمود `series_id` · (3) استبدال `event_reviews` بجدول **`room_requests`** المملوك لطبقة البيانات · (4) توحيد التسمية: `starts_at/ends_at`، `submission_mode`، `deadline`، ENUMs بأحرف صغيرة، مفاتيح BIGINT · (5) أرقام الحمل المرجعية: **20,000 متزامن** (قرار #21) · (6) إدراج سياسة **الحجب المالي** عبر `has_active_subscription()` في عرض الجدول (قرار #15) · (7) القاعات والمعامل عبر `rooms.is_lab` بدل افتراض `kind` · (8) التوافرية 99.5% عادي / **99.95% أثناء الامتحانات** · (9) إعادة كتابة ملحق الـ Traceability الذي كان مبتوراً |

---

## القسم 0: بروتوكول التحليل المتسلسل 🧠

### 0.1 — إعادة صياغة المشكلة (Problem Restatement)

المطلوب تصميم نظام جدولة مركزي يعرض للطالب جدولاً موحّداً يضم **7 أنواع أحداث متمايزة** (محاضرة، معمل، سمنار، امتحان، بحث، واجب، حدث ثقافي)، لكل نوع بطاقة عرض ووظائف مختلفة. الأحداث تدخل النظام عبر **أربعة مسارات إنشاء**: (أ) اقتراح من المحاضر يخضع لموافقة إدارية إذا تطلّب مورداً فيزيائياً (عبر `room_requests`)، (ب) نشر مباشر من المحاضر للواجبات والبحوث، (ج) جدولة حصرية من الإداري للامتحانات (قرار #10)، (د) نشر مباشر من الإداري للأحداث الثقافية (قرار #19). يجب أن يعمل الجدول بالكامل دون اتصال بعد أول مزامنة، وأن يُصدر إشعارات استباقية وفورية عند التغييرات.

**القيد المعماري الحاكم الجديد (v2.0):** النظام يعمل ضمن منصة **متعددة المؤسسات (حتى 10 مؤسسات — قرار #20)** ويبني فوق طبقة بيانات مركزية موحّدة (Core Data TDD v2.1) تملك جداول `institutions`, `profiles`, `terms`, `courses`, `enrollments`, `rooms`, `room_requests`, `events`. هذه الوثيقة **لا تعيد تعريف تلك الجداول** — بل توثّق كيف يستهلكها نظام الجدولة، وتضيف كائناتها الخاصة فقط (`event_series`, `event_outbox`).

### 0.2 — الأسئلة الخمسة الحاسمة

1. **من المستخدم الفعلي؟**
   ~20,000 طالب (2,000 × 10 مؤسسات) يقرأون جداولهم عدة مرات يومياً بذروة صباحية، ~200 محاضر لكل مؤسسة تقريبياً ينشئون أحداثاً بمعدل 2–5 أسبوعياً، وإداريون لكل مؤسسة يراجعون طلبات القاعات ويجدولون الامتحانات وينشئون الأحداث الثقافية. كل دور محصور بمؤسسته حصراً (`institution_id`).

2. **ما العملية الأثقل؟**
   **Read-heavy بامتياز** — بنسبة تقريبية 97:3 (قراءة:كتابة). قراءة الجدول تحدث عشرات آلاف المرات يومياً عبر المؤسسات العشر؛ الكتابة بالعشرات لكل مؤسسة. التصميم: فهرس `events (institution_id, course_id, starts_at)` من Core Data، كاش محلي (Hive)، ومزامنة تفاضلية (delta sync) عبر فهرس `events (institution_id, updated_at)`.

3. **ماذا لو توقف النظام ساعة؟**
   الطالب يستمر بالعمل من الكاش المحلي (متطلب صريح في MVP §3.1.2). لكن الكتابة الحرجة لا تتحمل توقفاً وقت الامتحانات. الهدف الموحّد مع Core Data: توافرية **99.5% عادي / 99.95% أثناء نوافذ الامتحانات المجدولة لأي من المؤسسات العشر**، مع منع أي صيانة مجدولة خلالها.

4. **ما البيانات الأكثر حساسية؟**
   بالترتيب: (1) **عزل المؤسسات** — تسريب حدث مؤسسة لمؤسسة أخرى هو الفشل الأخطر على الإطلاق، (2) صحة نطاق الرؤية داخل المؤسسة: طالب يرى أحداث مواد غير مسجّل فيها بحالة ACTIVE (خصوصاً مواعيد امتحانات لم تُنشر لدفعته)، (3) محاضر يعدّل حدث مادة ليست له. الدفاع: RLS بنمط `institution_id = my_institution()` أولاً ثم فحص الدور — على كل جدول بلا استثناء.

5. **أضيق عنق زجاجة متوقع خلال أول 6 أشهر؟**
   استعلام «جدول الطالب الأسبوعي» عند ذروة الصباح عبر 10 مؤسسات متزامنة. قابل للاحتواء بالفهرس المركّب المبدوء بـ `institution_id` (قاعدة D-05) + delta sync يجعل معظم الفتحات شبه فارغة (< 1KB). العنق الثاني: رشقة إشعارات FCM عند تعديل حدث لمادة كبيرة — تُعالج بمعالجة غير متزامنة عبر الـ Outbox.

### 0.3 — الافتراضات المعلنة (Explicit Assumptions)

| # | الافتراض | مستوى الثقة | تأثيره لو كان خاطئاً |
|---|---|---|---|
| A-01 | الحجم المرجعي: 10 مؤسسات × 2,000 طالب = ~20,000 مسجَّل، سقف تصميمي 20,000 متزامن (قرار #21 / D-06) | عالي | أرقام معتمدة رسمياً — أي تغيير قرار عمل جديد |
| A-02 | المنصة: Flutter + Supabase Pro (PostgreSQL + PostgREST + RPC + Realtime) + Supavisor — محسوم في Core Data [D-01], [D-04] | عالي | قرار طبقة البيانات — خارج صلاحية هذه الوثيقة |
| A-03 | لا أحداث متكررة (recurring) بمحرك RRULE — كل تكرار أسبوعي يُنشأ كصفوف مستقلة عبر أداة «تكرار حتى تاريخ» (توليد ثم إدراج دفعة) | متوسط | لو طُلب RRULE حقيقي: دين تقني مسجّل في 2.3 |
| A-04 | جدول `rooms` موحّد للقاعات والمعامل عبر عمود `is_lab BOOLEAN` — مملوك لطبقة البيانات (Core Data §4.2) ويديره نظام إدارة القاعات وظيفياً | عالي | الـ schema موجود فعلاً — لا غموض |
| A-05 | جداول `courses` و`enrollments` (بحالة ACTIVE) و`course_lecturers` موجودة من المرحلة 1 في خارطة Core Data | عالي | اعتمادية صلبة يجب اكتمالها أولاً |
| A-06 | التخزين UTC دائماً (`TIMESTAMPTZ`) والعرض بتوقيت الجهاز؛ كل المقارنات الزمنية بـ `now()` على الخادم | عالي | قاعدة موحّدة مع Core Data §6.1 |
| A-07 | الإشعارات تُسلَّم عبر FCM؛ متطلب التسليم: ≥ 95% خلال 60 ثانية (الإشعار الاستباقي «قبل 30 دقيقة» يُجدول محلياً على الجهاز من بيانات الكاش) | عالي | الإشعارات المحلية تغطي الاستباقي؛ التغييرات تصل عند أول مزامنة |

### 0.4 — نطاق العمل (Scope Fence)

**✅ داخل النطاق:**
- توثيق استهلاك جدول `events` (المملوك لـ Core Data) بأنواعه السبعة وبطاقات القسم 3.1.1 من MVP
- سير الموافقات عبر `room_requests`: اقتراح المحاضر (محاضرة/معمل/سمنار) ← مراجعة الإداري ← إنشاء الحدث المنشور
- النشر المباشر للواجبات والبحوث (نوع التسليم + مكان التسليم الورقي)
- النشر المباشر للأحداث الثقافية من الإداري (قرار #19)
- الجدولة الحصرية للامتحانات من الإداري بما فيها الإدراج الدفعي Top-Down (قرار #10)
- كشف تعارضات القاعة/الوقت (قيد `no_room_overlap` في Core Data + فحص استباقي)
- استعلام جدول الطالب المُفلتر + المزامنة التفاضلية للأوفلاين + سياسة الحجب المالي (قرار #15)
- الكائنات المُضافة: `event_series`, `event_outbox`, عمود `events.series_id`, والـ triggers الخاصة بها
- إطلاق أحداث الإشعارات (published/updated/cancelled) وتسليمها لنظام الإشعارات
- ملاحظة المحاضر على الحدث (أونلاين / مؤجلة)

**❌ خارج النطاق (منع scope creep):**
- تعريف أو تعديل جداول طبقة البيانات الأساسية (`events`, `rooms`, `room_requests`, `courses`, `enrollments`...) — ملكية Core Data TDD v2.1 حصراً
- آلية تسجيل الحضور (BLE/QR) — Attendance TDD؛ نوفّر فقط `event_id`
- محتوى الامتحان وبنك الأسئلة والمراقبة (جداول `exams`, `questions`... في Core Data + وثيقة الامتحانات)
- رفع ملفات تسليم البحوث/الواجبات وتصحيحها (مستودع الملفات + نظام أعمال المحاضر)
- إدارة بيانات القاعات ذاتها (سعة، موقع) — نظام إدارة القاعات
- بنية تسليم الإشعارات (tokens، قنوات) — نظام الإشعارات؛ نحن نُنتج الأحداث فقط
- الأحداث المتكررة بمحرك RRULE (A-03)

---

## القسم 1: الملخص التنفيذي وسياق المشكلة 📋

### 1.1 — بيان المشكلة

اليوم الأكاديمي للطالب مبعثر بين إعلانات ورقية ومجموعات واتساب وجداول Excel متغيرة، فيفوّت الطلاب محاضرات معدَّلة وامتحانات منقولة القاعة، ويستهلك المحاضرون والإداريون ساعات أسبوعياً بالتنسيق اليدوي لحجز القاعات. تكلفة عدم الحل: غيابات ظالمة، تعارضات قاعات تُكتشف لحظة وقوعها، وانعدام مصدر حقيقة واحد تعتمد عليه أنظمة الحضور والامتحانات والنتائج — مضروبة في 10 مؤسسات.

### 1.2 — الهدف القابل للقياس

تمكين 20,000 طالب عبر 10 مؤسسات من رؤية جدولهم الشخصي المُحدَّث (بأنواعه السبعة) خلال أقل من ثانيتين من فتح التطبيق — متصلاً أو غير متصل — مع وصول إشعار أي تغيير إلى ≥ 95% من الطلاب المعنيين خلال 60 ثانية، وتصفير تعارضات القاعات المنشورة داخل كل مؤسسة، وصفر تسريب حدث عبر حدود مؤسسة.

### 1.3 — معايير النجاح (Success Criteria)

| معيار | القيمة المستهدفة | كيف يُقاس |
|---|---|---|
| زمن استجابة استعلام الجدول p95 | < 300ms (server-side) — موحّد مع Core Data §1.3 | pg_stat_statements + APM |
| زمن فحص أي سياسة RLS | < 100ms — موحّد مع Core Data §1.3 | pg_stat_statements |
| زمن فتح الجدول من الكاش (أوفلاين) | < 500ms | تتبّع أداء داخل التطبيق |
| تسليم إشعار التغيير | ≥ 95% خلال 60 ثانية | سجل تسليم FCM |
| تعارضات قاعات في أحداث منشورة | 0 (يمنعها قيد `no_room_overlap`) | استعلام تدقيق يومي |
| تسريب حدث عبر حدود مؤسسة | 0 حالة | اختبارات pgTAP: دور × عملية × مؤسسة |
| نجاح المزامنة التفاضلية | ≥ 99% من الجلسات | تقارير أخطاء التطبيق |
| حجم حمولة المزامنة عند عدم وجود تغييرات | < 1KB | قياس شبكة |

### 1.4 — ما ليس هذا النظام (Anti-Goals)

- **ليس تقويماً شخصياً:** لا أحداث خاصة ينشئها الطالب لنفسه — الجدول انعكاس رسمي لمواده حصراً.
- **ليس نظام حجز ذاتي للقاعات:** المحاضر «يطلب» والإداري «يقرر» — لا حجز فوري تلقائي.
- **ليس محرك تقويم عام:** لا دعوات، لا RSVP، لا تكامل مع تقاويم خارجية.
- **ليس نظام الحضور ولا الامتحانات:** يوفّر لهما «العمود الفقري الزمني» فقط.
- **ليس مالك الـ Schema الأساسي:** جداول البيانات الجوهرية ملك Core Data TDD — هذه الوثيقة توثّق المنطق التشغيلي والإضافات فقط.

---

## القسم 2: القيود التقنية واختيار التقنيات 🔧

### 2.1 — القيود المفروضة (Hard Constraints)

| نوع القيد | التفصيل | مصدره |
|---|---|---|
| تقني | كل جدول محكوم بالمؤسسة يحمل `institution_id`، وكل فهرس أساسي يبدأ به، والقراءة من JWT claim لا subquery | MVP §1.2 (قرار #20) + Core Data §4.0 [D-08] |
| تقني | عمل كامل أوفلاين بعد أول مزامنة عبر Hive | MVP §3.1.2 |
| تقني | فصل صارم بين الأدوار والمؤسسات عبر RLS على مستوى المحرك | MVP §1.1 + Core Data [D-05] |
| تقني | الحجب المالي server-side عبر RLS و`has_active_subscription()` مصدر الحقيقة الوحيد | MVP §3.7.2 (قرار #15) |
| تنظيمي | جدولة الامتحان حصرية للإداري — تُفرض تقنياً (RLS + CHECK على `room_requests`) | قرار #10 |
| تنظيمي | إنشاء الحدث الثقافي حصري للإداري — نشر مباشر | قرار #19 |
| تنظيمي | كل قرار موافقة/رفض يُوثَّق ولا يُحذف (`room_requests` بلا سياسة DELETE) | MVP §5.0 |
| فريق/زمن | فريق صغير (2–3 مطورين) — تفضيل الحلول المُدارة | Core Data A-03 |
| تقني | الاعتمادية على جداول المرحلة 1 من Core Data (institutions, profiles, terms, courses, enrollments) | خارطة بناء Core Data §7.2 |

### 2.2 — مصفوفة اختيار التقنيات

> **ملاحظة v2.0:** قرارات قاعدة البيانات، المصادقة، الاستضافة، والتخزين المحلي محسومة مركزياً في Core Data TDD ([D-01]..[D-04], [D-08]) وتسري على هذه الوثيقة كما هي. تبقى هنا القرارات الخاصة بمنطق الجدولة فقط.

#### [S-01] طبقة: نمذجة أنواع الأحداث السبعة

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **جدول `events` واحد بعمود نوع + حقول مشتركة (Single Table) — كما هو معرّف في Core Data §4.2 ✅** | استعلام «جدول الطالب» = جدول واحد بلا UNION؛ فهرسة موحدة؛ المزامنة التفاضلية أبسط (`updated_at` واحد) | حقول nullable حسب النوع (تُضبط بـ CHECK constraints الموجودة في Core Data) | النظام read-heavy 97:3 والقراءة دائماً «كل الأنواع معاً» |
| جدول لكل نوع ❌ | تحقق صارم لكل نوع | استعلام الجدول = UNION من 7 جداول؛ RLS ×7 | يعاقب مسار القراءة الحرج |
| EAV ❌ | مرونة قصوى | أداء كارثي | إفراط هندسي مرفوض |

#### [S-02] طبقة: سير الموافقات (Approval Workflow)

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **جدول `room_requests` (المملوك لـ Core Data) لدورة الاقتراح/القرار، وإنشاء صف `events` بحالة `approved` عند القبول ✅** | يستخدم الـ schema الموحّد كما هو — صفر ازدواجية؛ `room_requests` يحمل `reviewed_by` و`rejection_reason` فيحقق التوثيق الدائم (MVP §5.0)؛ استعلام «طلباتي وحالتها» للمحاضر بسيط | الحدث لا يظهر في `events` قبل القبول (وهذا مطلوب أصلاً — الطالب لا يرى المقترحات) | القرار المعتمد في v2.0 — يستبدل جدول `event_reviews` من v1.1 الذي كان يكرر نفس الغرض |
| آلة حالات على events + جدول `event_reviews` (تصميم v1.1) ❌ | الحدث كيان واحد طوال حياته | يتعارض مع `room_requests` الموجود في Core Data — جدولان لنفس الغرض = تباعد حتمي | أُلغي في v2.0 لتوحيد المعمارية |
| محرك workflow خارجي ❌ | قوي للمسارات المعقدة | المسار خطوة واحدة | إفراط هندسي |

#### [S-03] طبقة: كشف تعارض القاعات

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **قيد `no_room_overlap` (EXCLUDE USING gist) المعرّف في Core Data §4.2 + فحص استباقي في RPC ✅** | ضمانة نهائية على مستوى المحرك — يستحيل نشر تعارض حتى مع طلبين متزامنين؛ الفحص الاستباقي يعطي رسالة ودّية | رسالة خطأ القيد خام وتحتاج ترجمة في الـ RPC | الفحص التطبيقي وحده يحتوي فجوة سباق (TOCTOU)؛ القيد يغلقها |
| فحص تطبيقي فقط ❌ | بسيط | سباق زمني | يكسر معيار «0 تعارض» |
| قفل تشاؤمي ❌ | يمنع السباق | يُسلسل الكتابات؛ خطر deadlocks | القيد يعطي نفس الضمانة بلا كلفة |

#### [S-04] طبقة: الأوفلاين والمزامنة

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **كاش Hive للقراءة فقط + Delta Sync بفهرس `events (institution_id, updated_at)` مع جلب soft-deleted للتنظيف ✅ (= قرار Core Data [D-06])** | يحقق متطلب MVP حرفياً؛ لا تعارضات مزامنة لأن الطالب لا يكتب على الأحداث أوفلاين؛ حمولة شبه صفرية عند عدم التغيير | يتطلب soft delete (موجود: `events.deleted_at` في Core Data) | أبسط مزامنة صحيحة |
| مزامنة ثنائية الاتجاه (CRDT) ❌ | تدعم كتابة أوفلاين | تعقيد كبير لكتابة لا يحتاجها الطالب هنا | حل لمشكلة غير موجودة |

#### [S-05] طبقة: الأحداث المتكررة

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **توليد الصفوف مسبقاً (Materialized Occurrences) مربوطة بـ `series_id` اختياري ✅** | كل حدث مستقل: يُعدَّل/يُلغى/تُعلَّق عليه ملاحظة منفرداً؛ الحضور يرتبط بصف ملموس | فصل من 15 أسبوعاً × ~300 مادة × ~3 أحداث × 10 مؤسسات ≈ 135K صف/فصل — حجم مقبول تماماً | RRULE يعقّد كل استعلام؛ الحجم لا يبرره |
| RRULE ❌ | تخزين مضغوط | فك التكرار عند كل قراءة؛ استثناءات معقدة | مسجّل في الديون التقنية |

#### [S-06] طبقة: إيصال التغييرات والإشعارات

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **جدول Outbox (`event_outbox`) يملؤه Trigger + Edge Function مجدولة كل 30 ثانية تدفع لنظام الإشعارات (FCM) ✅** | نمط Transactional Outbox: التغيير وسجلّ إشعاره في معاملة واحدة — لا إشعار مفقود؛ at-least-once + إزالة تكرار بـ `outbox.id` في المستهلك (متوافق مع ضمانة Core Data §5.3) | تأخير حتى 30 ثانية (ضمن هدف 60 ثانية)؛ تنظيف دوري | أبسط ضمان تسليم موثوق بلا بنية رسائل خارجية |
| Webhook داخل معاملة الكتابة ❌ | فوري | فشل FCM يُفشل الكتابة أو يضيع الإشعار | يكسر الموثوقية |
| Kafka/RabbitMQ ❌ | تسليم صناعي | بنية تشغيلية كاملة لعشرات الرسائل بالدقيقة | إفراط هندسي |

### 2.3 — الديون التقنية المقبولة عمداً

| الدين | لماذا نقبله الآن | متى يُسدَّد |
|---|---|---|
| لا محرك RRULE — توليد صفوف مسبق [S-05] | الحجم مقبول والاستقلالية مطلوبة | إذا تجاوزت الأحداث ~1M صف/فصل أو طُلبت جداول ديناميكية |
| polling كل 30 ثانية للـ Outbox بدل دفع فوري | ضمن هدف الـ 60 ثانية | إذا شُدّد الهدف إلى < 10 ثوانٍ: pg_notify / Realtime trigger |
| لا history كامل لتعديلات الحدث (آخر حالة + payload الـ outbox فقط) | يكفي متطلبات MVP التوثيقية | عند طلب تدقيق كامل: `event_versions` بنمط append-only |
| الفحص الاستباقي للتعارض يكرّر منطق القيد | تجربة مستخدم أفضل، والقيد هو الضامن | لا يُسدَّد — ازدواج مقصود (defense in depth) |

---

## القسم 3: معمارية النظام والمخططات 🏗️

### 3.1 — المخطط العام (System Context)

\`\`\`mermaid
graph TB
    subgraph Clients["تطبيق Flutter (10 مؤسسات)"]
        ST[👤 الطالب<br/>قراءة الجدول + كاش Hive]
        LC[👤 المحاضر<br/>اقتراح/نشر أحداث]
        AD[👤 الإداري<br/>مراجعة + جدولة امتحانات + أحداث ثقافية]
    end

    subgraph Supabase["Supabase Pro + Supavisor (Core Data Layer)"]
        AUTH[Auth + JWT<br/>الدور + institution_id في الـ claims]
        API[PostgREST + RPC<br/>فحص استباقي للتعارض]
        DB[(PostgreSQL<br/>events + room_requests<br/>RLS: مؤسسة × دور + EXCLUDE)]
        OUTBOX[Edge Function مجدولة<br/>قارئ الـ Outbox]
    end

    NOTIF[نظام الإشعارات<br/>FCM — خارج نطاقنا]
    ATT[نظام الحضور<br/>يستهلك event_id]
    ROOMS[نظام إدارة القاعات<br/>يدير جدول rooms وظيفياً]

    ST --> API
    LC --> API
    AD --> API
    ST -.->|أوفلاين| ST
    API --> AUTH
    API --> DB
    DB -->|trigger يملأ outbox| OUTBOX
    OUTBOX --> NOTIF
    ATT -.->|FK: event_id| DB
    ROOMS -.->|FK: room_id| DB
\`\`\`

### 3.2 — جدول المكونات والمسؤوليات

| المكون | مسؤوليته الوحيدة | ماذا لو سقط؟ | استراتيجية التعافي |
|---|---|---|---|
| PostgreSQL (events + room_requests + القيود) | مصدر الحقيقة الوحيد + ضمانة عدم التعارض + عزل المؤسسات | لا كتابة؛ القراءة تستمر من كاش Hive | failover مُدار؛ RTO ≤ 5 دقائق (موحّد مع Core Data §3.2)؛ PITR |
| PostgREST / RPC | عقود الجدولة + الفحص الاستباقي + الإدراج الدفعي للامتحانات | نفس أثر سقوط DB للعمليات؛ الجدول المتزامن سابقاً متاح أوفلاين | إعادة تشغيل مُدارة؛ العميل يعيد المحاولة بـ exponential backoff |
| RLS Policies (مؤسسة × دور) | فرض نطاق الرؤية والكتابة | policy خاطئة = تسريب — الفشل الأخطر | اختبارات pgTAP إلزامية لكل policy × دور × مؤسسة قبل أي نشر |
| قارئ الـ Outbox | تحويل صفوف outbox إلى دفعات إشعارات | تتراكم الإشعارات دون ضياع | التشغيل التالي يلتقط المتراكم؛ تنبيه إذا عمر أقدم صف > 5 دقائق |
| كاش Hive (على الجهاز) | عرض الجدول أوفلاين + الإشعارات الاستباقية المحلية | فقدان الكاش = إعادة مزامنة كاملة | إعادة بناء تلقائية؛ لا بيانات تُفقد (قراءة فقط) |

### 3.3 — تدفق البيانات لأهم العمليات (Critical Path Sequences)

**العملية 1: اقتراح المحاضر لمحاضرة ← موافقة الإداري ← النشر والإشعار**

\`\`\`mermaid
sequenceDiagram
    participant L as المحاضر
    participant A as RPC
    participant D as PostgreSQL
    participant O as Outbox Worker
    participant N as نظام الإشعارات
    participant S as الطالب

    L->>A: propose_room_event(lecture, room_id, time)
    A->>D: فحص استباقي للتعارض (نتيجة قد تتقادم — القيد يضمن لاحقاً)
    A->>D: INSERT room_requests (status=pending, institution_id من JWT)
    D-->>A: 201 + request_id
    A-->>L: تم الإرسال — قيد المراجعة ⏳

    Note over A,D: --- لاحقاً: مراجعة الإداري (نفس المؤسسة حصراً) ---
    A->>D: BEGIN; UPDATE room_requests SET status=accepted, reviewed_by=...;<br/>INSERT events (status=approved)
    D->>D: قيد no_room_overlap يتحقق نهائياً<br/>(تعارض ظهر بعد الفحص → 409 للإداري + ROLLBACK)
    D->>D: Trigger يُدرج صف event_outbox (published)
    D-->>A: COMMIT
    O->>D: قراءة outbox غير المعالج (كل 30 ثانية)
    O->>N: دفعة إشعارات لطلاب المادة (فشل FCM → retry بـ backoff)
    N-->>S: 🔔 «محاضرة جديدة أُضيفت لجدولك»
\`\`\`

**العملية 2: الطالب يفتح جدوله (المسار الأثقل — مع الأوفلاين والحجب المالي)**

\`\`\`mermaid
sequenceDiagram
    participant S as تطبيق الطالب
    participant H as كاش Hive
    participant A as PostgREST
    participant D as PostgreSQL

    S->>H: اقرأ الجدول فوراً (< 500ms)
    H-->>S: عرض الجدول من الكاش
    S->>A: GET /events?updated_at=gt.<last_sync_at>&...
    alt متصل
        A->>D: SELECT محكوم بـ RLS: institution_id = my_institution()<br/>AND (type='exam' OR has_active_subscription())<br/>AND is_enrolled(course_id) — سياسة events_visibility في Core Data
        D-->>A: الدلتا فقط (غالباً 0 صف، < 1KB)
        A-->>S: 200 + التغييرات + server_time
        S->>H: دمج الدلتا (يشمل soft deletes) + إعادة جدولة الإشعارات المحلية (قبل 30 دقيقة)
    else غير متصل
        S->>S: البقاء على الكاش + شارة «آخر تحديث منذ...»
    end
\`\`\`

**العملية 3: الجدولة الدفعية للامتحانات من الإداري (Top-Down)**

\`\`\`mermaid
sequenceDiagram
    participant AD as الإداري
    participant A as RPC: schedule_exams_bulk
    participant D as PostgreSQL

    AD->>A: schedule_exams_bulk(idempotency_key, exams[])
    A->>A: تحقق: الدور admin حصراً (قرار #10) + كل المواد لنفس institution_id
    A->>D: BEGIN + INSERT events (type=exam, status=approved) + INSERT exams
    alt تعارض في أي صف
        D-->>A: exclusion_violation على الصف X
        A->>D: ROLLBACK (الكل أو لا شيء)
        A-->>AD: 409 + تعداد الصفوف المتعارضة بالاسم والوقت
    else لا تعارض
        D-->>A: COMMIT (+ صفوف outbox لكل امتحان)
        A-->>AD: 201 + ملخص المُدرج
    end
\`\`\`

### 3.4 — قرارات معمارية جوهرية

#### [S-07] Monolith داخل طبقة البيانات — لا Microservices

موحّد مع Core Data [D-05]: المنطق الأمني والمعاملاتي داخل قاعدة البيانات (RLS + RPC). معاملة واحدة تضمن اتساق (القرار + الحدث + الـ outbox). لا خدمة جدولة مستقلة — معاملات موزعة بلا مقابل.

#### [S-08] كتابة متزامنة (Sync) + إشعارات غير متزامنة (Async)

المسار الحرج (إنشاء/موافقة/تعديل) متزامن بالكامل داخل معاملة واحدة — المستخدم يعرف النتيجة فوراً بما فيها التعارض. كل ما هو «إخطار الآخرين» غير متزامن عبر الـ Outbox [S-06]. الحدّ الملموس: أي عملية كتابة تتجاوز 2 ثانية server-side تُعتبر خللاً يستوجب تنبيهاً (القسم 6.5).

---

## القسم 4: نماذج البيانات وتصميم قاعدة البيانات 🗄️

### 4.0 — قواعد التعدد المؤسسي الإلزامية (موروثة من Core Data §4.0 — غير قابلة للاستثناء)

1. كل جدول تضيفه هذه الوثيقة يحمل `institution_id`.
2. كل فهرس أساسي عليه يبدأ بـ `institution_id`.
3. `institution_id` يُقرأ من JWT claim عبر `my_institution()` — لا subquery داخل السياسات.
4. أي مسار Storage مرتبط بالأحداث يبدأ بـ `{institution_id}/`.

### 4.1 — الجداول المستهلَكة (مملوكة لـ Core Data TDD v2.1 — مرجع فقط، لا تُعاد هنا)

| الجدول | ما نستهلكه منه | ملاحظات المواءمة |
|---|---|---|
| `events` | الجدول المركزي: `type` (event_type: lecture/lab/seminar/exam/research/assignment/cultural)، `status` (pending/approved/rejected/cancelled)، `starts_at/ends_at`، `deadline`، `submission_mode` (online/paper)، `paper_submit_location`، `lecturer_note`، `room_id`، `deleted_at`، `updated_at` + قيد `no_room_overlap` + سياسة `events_visibility` (الحجب المالي — قرار #15) | «هل يتطلب حضوراً» يُشتق من النوع (قرار #6): lecture/lab/seminar/exam — لا عمود مستقل |
| `rooms` | `room_id` + `is_lab` للتمييز بين قاعة ومعمل (A-04) | حذف قاعة لها أحداث مستقبلية: `ON DELETE RESTRICT` — مسؤولية نظام القاعات |
| `room_requests` | دورة الاقتراح/القرار كاملة: `requested_type` (CHECK يمنع exam — قرار #10)، `status` (pending/accepted/rejected)، `rejection_reason`، `reviewed_by` | يستبدل `event_reviews` من v1.1؛ append-only (لا سياسة DELETE) |
| `courses`, `enrollments`, `course_lecturers` | نطاق الرؤية (`is_enrolled`) وإسناد المحاضر (`teaches`) | `enrollments.status = 'ACTIVE'` شرط الرؤية |
| `exams` | صف امتحان لكل حدث `type='exam'` (`scheduled_by` إداري حصراً) | الإدراج الدفعي ينشئ الصفين معاً ذرياً |

### 4.2 — الكائنات المُضافة بواسطة هذه الوثيقة (Schema Additions)

\`\`\`sql
-- ============================================================
-- ترحيل نظام الجدولة — يُطبَّق فوق Core Data Schema v2.1
-- ============================================================

-- 1) event_series: تجميع اختياري لتكرارات الحدث الأسبوعي [S-05]
CREATE TABLE event_series (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    created_by     UUID NOT NULL REFERENCES profiles(id),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_event_series_tenant ON event_series (institution_id);

-- 2) إضافة عمود السلسلة إلى events (expand-and-contract: nullable، لا قفل)
ALTER TABLE events ADD COLUMN series_id BIGINT REFERENCES event_series(id);

-- 3) event_outbox: نمط Transactional Outbox للإشعارات [S-06]
CREATE TABLE event_outbox (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    event_id       BIGINT NOT NULL REFERENCES events(id),
    change_kind    TEXT NOT NULL CHECK
                   (change_kind IN ('published','updated','cancelled')),
    payload        JSONB NOT NULL,      -- لقطة الحقول المتغيرة للإشعار
    processed_at   TIMESTAMPTZ,         -- NULL = بانتظار المعالجة
    attempts       INT NOT NULL DEFAULT 0,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- فهرس جزئي لقارئ الـ Outbox (كل 30 ثانية)
CREATE INDEX idx_outbox_pending
    ON event_outbox (institution_id, id) WHERE processed_at IS NULL;

-- RLS: لا وصول لأي دور عميل — للخدمة (service_role) فقط
ALTER TABLE event_outbox ENABLE ROW LEVEL SECURITY;
ALTER TABLE event_series ENABLE ROW LEVEL SECURITY;

CREATE POLICY series_same_institution ON event_series FOR SELECT
    USING (institution_id = my_institution());
CREATE POLICY series_insert_staff ON event_series FOR INSERT
    WITH CHECK (institution_id = my_institution()
                AND my_role() IN ('lecturer','admin')
                AND created_by = auth.uid());

-- 4) Trigger ملء الـ Outbox عند النشر/التعديل/الإلغاء
CREATE OR REPLACE FUNCTION fill_event_outbox() RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.status = 'approved' THEN
        INSERT INTO event_outbox (institution_id, event_id, change_kind, payload)
        VALUES (NEW.institution_id, NEW.id, 'published',
                jsonb_build_object('course_id', NEW.course_id,
                                   'type', NEW.type, 'title', NEW.title,
                                   'starts_at', NEW.starts_at));
    ELSIF TG_OP = 'UPDATE' AND NEW.status = 'cancelled'
          AND OLD.status IS DISTINCT FROM 'cancelled' THEN
        INSERT INTO event_outbox (institution_id, event_id, change_kind, payload)
        VALUES (NEW.institution_id, NEW.id, 'cancelled',
                jsonb_build_object('course_id', NEW.course_id, 'title', NEW.title));
    ELSIF TG_OP = 'UPDATE' AND NEW.status = 'approved' THEN
        INSERT INTO event_outbox (institution_id, event_id, change_kind, payload)
        VALUES (NEW.institution_id, NEW.id, 'updated',
                jsonb_build_object('course_id', NEW.course_id, 'title', NEW.title,
                                   'starts_at', NEW.starts_at, 'ends_at', NEW.ends_at,
                                   'room_id', NEW.room_id,
                                   'lecturer_note', NEW.lecturer_note));
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_events_outbox
AFTER INSERT OR UPDATE ON events
FOR EACH ROW EXECUTE FUNCTION fill_event_outbox();
\`\`\`

> **ملاحظة:** trigger صيانة `updated_at` على `events` معرّف أصلاً في Core Data — لا يُعاد هنا. سياسات RLS على `events` و`room_requests` (بما فيها `events_visibility` مع الحجب المالي) مملوكة لـ Core Data §4.2 أيضاً.

### 4.3 — استراتيجية الفهرسة (Indexing Strategy)

الفهارس على `events` و`enrollments` و`room_requests` معرّفة ومبررة في Core Data §4.3 (وكلها تبدأ بـ `institution_id` وفق قاعدة D-05). الجدول التالي يوثّق ما يخدم مسارات الجدولة تحديداً + الفهارس المُضافة:

| الفهرس | المالك | الاستعلام الذي يخدمه | تكلفته على الكتابة |
|---|---|---|---|
| `events (institution_id, course_id, starts_at) WHERE deleted_at IS NULL` | Core Data | جدول الطالب الأسبوعي — المسار الأثقل | منخفضة — كتابات الأحداث نادرة |
| `events (institution_id, updated_at)` | Core Data | المزامنة التفاضلية [S-04] | منخفضة |
| GiST من قيد `no_room_overlap` | Core Data | كشف التعارض + إشغال القاعة زمنياً | متوسطة — مقبولة لندرة الإدراج |
| `room_requests (institution_id, status)` | Core Data | لوحة الإداري + شاشة «طلباتي» للمحاضر | شبه معدومة |
| `event_outbox (institution_id, id) WHERE processed_at IS NULL` | **هذه الوثيقة** | قارئ الـ Outbox كل 30 ثانية | شبه معدومة |
| `event_series (institution_id)` | **هذه الوثيقة** | تعديل/إلغاء «كل السلسلة» | شبه معدومة |

### 4.4 — الحذف والأرشفة والتوسع (موحّد مع Core Data §4.4)

- **الترحيلات:** expand-and-contract حصراً؛ ممنوع أي `ALTER` هدّام خلال نافذة امتحان مجدولة **لأي من المؤسسات العشر**.
- **الحذف:** soft delete على `events` (`deleted_at` + `status='cancelled'`) — لثلاثة أسباب: (1) نظام الحضور يشير إلى أحداث قد تُلغى بعد تسجيل حضورها، (2) المزامنة التفاضلية تحتاج إيصال «هذا الحدث حُذف» للأجهزة الأوفلاين، (3) قيد MVP «يُوثَّق ولا يُحذف». **hard delete** فقط لصفوف outbox المعالَجة الأقدم من 30 يوماً (pg_cron). `room_requests` لا تُحذف أبداً.
- **التوسع:** ~135K صف أحداث/فصل عبر 10 مؤسسات ≈ 270K/سنة — لا حاجة لتقسيم قبل سنوات. عتبة القرار: > 5M صف أو p95 > 300ms رغم الفهارس ← partitioning بـ RANGE على `starts_at` بمفتاح الفصل. مفتاح sharding المستقبلي جاهز: `institution_id`.
- **التوافقية:** الكتابة الحرجة (النشر، الموافقة، منع التعارض) **strict** — معاملة واحدة + قيد المحرك. **eventual مقبولة** في: وصول الإشعارات (≤ 60 ثانية) وكاش الأجهزة (شارة «آخر تحديث منذ X»).

---

## القسم 5: عقود الـ API والواجهات 🔌

### 5.1 — مبادئ التصميم

- **موحّد مع Core Data [D-07]:** PostgREST (محكوم بـ RLS) للقراءات، و**RPC** لكل عملية كتابة متعددة الخطوات. لا REST يدوي، لا GraphQL.
- **النسخ:** الدوال تُنسَّخ بالاسم (`propose_room_event_v2`) عند التغيير الكاسر؛ القديمة تبقى فصلاً دراسياً كاملاً (الأجهزة الأوفلاين تتأخر بالتحديث).
- **صيغة الخطأ الموحدة (= Core Data §5.1):** `{"error": {"code": "ROOM_CONFLICT", "message_ar": "..."}}`
- **المصادقة:** JWT من Supabase Auth — الدور و`institution_id` ضمن الـ claims ويُعاد فرضهما بـ RLS (دفاع مزدوج).

### 5.2 — توثيق الـ Endpoints

#### قراءة الجدول: `GET /rest/v1/events` (PostgREST مباشرة)
**الغرض:** جلب جدول المستخدم (كامل أو دلتا) — الفلترة الأمنية كلها في سياسة `events_visibility` (مؤسسة + تسجيل ACTIVE + الحجب المالي عدا الامتحانات) | **المصادقة:** كل الأدوار | **Rate Limit:** 60/دقيقة/مستخدم

**المدخلات (Query):** `starts_at=gte.<from>` و`starts_at=lt.<to>` (الأقصى 60 يوماً) — وللدلتا: `updated_at=gt.<last_sync_at>` (تشمل الملغى/المحذوف soft). كل رد يتضمن `server_time` لعزل انحراف ساعة الجهاز.

**حالات الخطأ:** 400 (نطاق > 60 يوماً) · 401 (توكن منتهٍ — البقاء على الكاش أوفلاين) · 429 (backoff) · 500 (البقاء على الكاش).

---

#### `POST /rest/v1/rpc/propose_room_event`
**الغرض:** اقتراح محاضرة/معمل/سمنار — يُدرج في `room_requests` بحالة pending | **المصادقة:** lecturer معيَّن على المادة (`teaches`) | **Rate Limit:** 30/دقيقة

**المدخلات:**

\`\`\`json
{
  "requested_type": "lecture | lab | seminar — مطلوب؛ exam مرفوض بقيد CHECK (قرار #10)",
  "course_id": "bigint — من مواد المحاضر، نفس المؤسسة",
  "title": "string — 3–120 حرفاً",
  "room_id": "bigint — مطلوب",
  "requested_start": "ISO-8601",
  "requested_end": "ISO-8601 — > requested_start",
  "repeat_weekly_until": "ISO-8601 — اختياري: يولّد طلبات السلسلة [S-05]، الحد 30 تكراراً"
}
\`\`\`

**المخرجات الناجحة (201):** `{ "request_id": "bigint", "status": "pending", "conflict_warning": "string | null" }`

**حالات الخطأ:** 400 (حقول ناقصة) · 403 (النوع exam، أو مادة ليست للمحاضر، أو مؤسسة مختلفة — RLS) · 409 (الفحص الاستباقي وجد تعارضاً — تحذير مع اقتراح وقت آخر) · 422 (تكرار > 30) · 429 · 500.

---

#### `POST /rest/v1/rpc/publish_direct_event`
**الغرض:** نشر مباشر — من المحاضر: واجب/بحث؛ من الإداري: حدث ثقافي (قرار #19). يُدرج في `events` بحالة approved فوراً | **المصادقة:** lecturer (research/assignment لمواده) أو admin (cultural حصراً) | **Rate Limit:** 30/دقيقة

**المدخلات:**

\`\`\`json
{
  "type": "research | assignment (lecturer) — cultural (admin حصراً)",
  "course_id": "bigint — مطلوب لغير cultural؛ NULL للثقافي",
  "title": "string",
  "description": "string — اختياري، ≤ 2000 حرف",
  "deadline": "ISO-8601 — مطلوب للواجب/البحث",
  "submission_mode": "online | paper — مطلوب للواجب/البحث",
  "paper_submit_location": "string — مطلوب عند paper",
  "starts_at": "ISO-8601 — مطلوب للثقافي",
  "ends_at": "ISO-8601 — مطلوب للثقافي",
  "room_id": "bigint — اختياري للثقافي (عند غيابه يُدخل المكان نصياً في description)"
}
\`\`\`

**المخرجات الناجحة (201):** `{ "event_id": "bigint", "status": "approved" }`

**حالات الخطأ:** 400 (حقول غير متسقة مع النوع — CHECK constraints مترجمة) · 403 (cultural من غير admin، أو مادة ليست للمحاضر) · 409 (ثقافي مرتبط بقاعة اصطدم بالقيد — منع مباشر لأنه يُنشر فوراً) · 429 · 500.

---

#### `POST /rest/v1/rpc/review_room_request`
**الغرض:** قرار الإداري — القبول يعدّل القاعة/الوقت النهائيين اختيارياً ويُنشئ الحدث المنشور ذرياً؛ الرفض يوثّق السبب | **المصادقة:** admin حصراً، نفس المؤسسة | **Rate Limit:** 60/دقيقة

**المدخلات:**

\`\`\`json
{
  "request_id": "bigint — مطلوب",
  "decision": "accepted | rejected",
  "rejection_reason": "string — إلزامي عند rejected (MVP §5.1.2)، ≤ 500 حرف",
  "final_room_id": "bigint — اختياري عند accepted",
  "final_start": "ISO-8601 — اختياري عند accepted",
  "final_end": "ISO-8601 — اختياري عند accepted"
}
\`\`\`

**المخرجات الناجحة (200):** `{ "request_id": "bigint", "status": "accepted", "event_id": "bigint | null" }`

**حالات الخطأ:** 400 (رفض بلا سبب / أوقات غير متسقة) · 403 (ليس admin — RLS) · 404 (الطلب غير موجود أو لمؤسسة أخرى) · 409 (أ: الطلب ليس pending — قرار مسبق، UPDATE مشروط يحسم السباق؛ ب: قيد `no_room_overlap` اصطدم بتعارض ظهر بعد الفحص ← ROLLBACK كامل) · 429 · 500 (idempotent-safe — التكرار يصطدم بـ 409).

---

#### `POST /rest/v1/rpc/schedule_exams_bulk`
**الغرض:** الجدولة الدفعية للامتحانات (Top-Down) — ذرّية بالكامل: `events` + `exams` معاً | **المصادقة:** admin حصراً (قرار #10) | **Rate Limit:** 10/دقيقة

**المدخلات:**

\`\`\`json
{
  "idempotency_key": "uuid — يمنع الإدراج المزدوج عند إعادة الإرسال",
  "exams": [
    {
      "course_id": "bigint",
      "title": "string",
      "room_id": "bigint",
      "starts_at": "ISO-8601",
      "ends_at": "ISO-8601",
      "kind": "midterm | final",
      "duration_minutes": 90,
      "question_count": 40
    }
  ]
}
\`\`\`

**المخرجات الناجحة (201):** `{ "inserted": 42, "event_ids": ["..."] }`

**حالات الخطأ:** 400 (قائمة فارغة أو > 200 امتحاناً) · 403 (ليس admin — يشمل أي محاولة من محاضر) · 409 (تعارض في أي صف ← ROLLBACK كامل + تعداد المتعارض بالفهرس والتفاصيل / أو idempotency_key مستخدم بنجاح سابقاً — تُعرض النتيجة السابقة) · 429 · 500 (إعادة الإرسال آمنة بفضل idempotency_key).

---

#### `POST /rest/v1/rpc/update_event`
**الغرض:** تعديل حدث (وقت/قاعة/ملاحظة «أونلاين/مؤجلة») — يُطلق إشعار updated | **المصادقة:** منشئ الحدث (محاضر لمواده) أو admin؛ تعديل exam وcultural: admin حصراً | **Rate Limit:** 30/دقيقة

**المدخلات:** `event_id` + أي مجموعة جزئية من حقول الإنشاء + `lecturer_note` + `expected_updated_at` (للتحكم التفاؤلي بالتزامن — نفس نمط optimistic locking في Core Data §6.1).

**حالات الخطأ:** 400 · 403 (ليس المنشئ/الإداري، أو محاضر يعدّل exam/cultural) · 404 (غير موجود/محذوف/مؤسسة أخرى) · 409 (`expected_updated_at` ≠ الحالي — جلب الأحدث والدمج / أو تغيير الوقت اصطدم بقيد التعارض) · 429 · 500.

---

#### `POST /rest/v1/rpc/cancel_event`
**الغرض:** إلغاء حدث (soft: `status='cancelled'` + `deleted_at`) — يُطلق إشعار cancelled | **المصادقة:** كما في update_event | **Rate Limit:** 30/دقيقة

**المخرجات الناجحة (200):** `{ "event_id": "bigint", "status": "cancelled" }` — إذا كان ملغى أصلاً: 409 يعتبره العميل نجاحاً (idempotent عملياً).

### 5.3 — العقود بين الأنظمة الداخلية

| المستهلك | العقد | الضمانة |
|---|---|---|
| نظام الإشعارات | صفوف `event_outbox` بحمولة `{event_id, change_kind, course_id, title, changed_fields}` | **at-least-once** — إزالة التكرار بمفتاح `outbox.id` (= ضمانة Core Data §5.3)؛ الترتيب مضمون لكل حدث (id تسلسلي) |
| نظام الحضور | قراءة `events`: `id`, `type`, `starts_at`, `ends_at`, `status` — الحضور مطلوب للأنواع lecture/lab/seminar/exam (قرار #6) | لا يفتح جلسة إلا لحدث `approved` غير محذوف من نفس المؤسسة |
| نظام إدارة القاعات | يدير `rooms` وظيفياً (سعة، موقع، is_lab)؛ نظامنا FK فقط | حذف قاعة لها أحداث مستقبلية منشورة محظور (`ON DELETE RESTRICT`) |
| نظام الامتحانات | صفا `events` + `exams` يُنشآن ذرياً من `schedule_exams_bulk` | النافذة الزمنية من `events`؛ المحتوى والمحاولات خارج نطاقنا |

---

## القسم 6: حالات الحافة، أنماط الفشل، والأمان 🛡️

### 6.1 — جرد حالات الحافة (Edge Case Inventory)

| السيناريو | ماذا يحدث في التصميم؟ | المعالجة |
|---|---|---|
| طلبان متزامنان يعدلان نفس الحدث | `expected_updated_at` (optimistic locking) | الثاني يستلم 409 ويجلب الأحدث — لا كتابة ضائعة |
| إداريان يراجعان نفس الطلب معاً | UPDATE مشروط بـ `status='pending'` داخل المعاملة | الثاني يجد 0 صف متأثر ← 409 «قرار مسبق» |
| موافقة على طلبين متزامنين لنفس القاعة والوقت | الفحص الاستباقي قد يمرّ للاثنين (TOCTOU) | قيد `no_room_overlap` يُفشل الثاني حتماً عند COMMIT ← 409 [S-03] |
| إداري من مؤسسة يحاول مراجعة طلب مؤسسة أخرى | RLS: `institution_id = my_institution()` | 404 — الصف غير مرئي أصلاً |
| ملف دفعي يخلط مواد من مؤسستين | فحص `institution_id` لكل صف داخل `schedule_exams_bulk` + RLS كطبقة ثانية | ROLLBACK كامل + 403 |
| مدخلات 10x المتوقع | حدود صريحة: title ≤ 120، description ≤ 2000، دفعة ≤ 200، تكرار ≤ 30، نطاق ≤ 60 يوماً، جسم الطلب ≤ 100KB | 400/422 قبل ملامسة DB |
| انقطاع الشبكة في منتصف كتابة | الكتابة معاملة ذرّية؛ العميل لا يعرف النتيجة | `idempotency_key` للدفعي؛ إعادة إرسال review تصطدم بـ 409 الآمن؛ اقتراح مكرر يكشفه المحاضر بشاشة «طلباتي» |
| FCM بطيء (وليس ساقطاً) | قارئ الـ Outbox بـ timeout 10 ثوانٍ لكل دفعة | فشل الدفعة يترك `processed_at=NULL` مع `attempts+1`؛ بعد 5 محاولات ← تنبيه تشغيلي |
| رسالة مكررة من الـ Outbox (at-least-once) | ممكنة بالتصميم | نظام الإشعارات يزيل التكرار بـ `outbox.id`؛ إشعار مكرر مزعج لا خطير |
| المناطق الزمنية / انحراف ساعة الجهاز | `TIMESTAMPTZ` (UTC) حصراً + `server_time` في كل رد | ساعة الجهاز للعرض فقط — لا قرار يُبنى عليها (= Core Data §6.1) |
| طالب أوفلاين أسبوعاً وحدث في كاشه أُلغي | soft delete + الدلتا تشمل المحذوفات | أول مزامنة تُنظّف الكاش وتلغي الإشعارات المحلية المجدولة |
| طالب متأخر عن الاشتراك | سياسة `events_visibility`: يختفي كل شيء **عدا بطاقات الامتحانات** (قرار #15) | server-side عبر RLS حصراً؛ الكاش المحلي يُنظَّف عند أول مزامنة بعد الحجب |
| إلغاء محاضرة بعد تسجيل طلاب حضورهم | FK من الحضور إلى الحدث + soft delete | سجلات الحضور تبقى سليمة مرتبطة بحدث cancelled — قرار احتسابها لنظام النتائج |
| حدث ثقافي بلا مادة | `course_id` nullable؛ إنشاؤه حصري للإداري (قرار #19) | يُعرض لجميع مستخدمي **نفس المؤسسة** فقط |
| `repeat_weekly_until` يتقاطع مع عطلة | لا تقويم عطلات في MVP | إلغاء التكرارات الفردية يدوياً — قيد معروف موثّق |
| 10 امتحانات جماعية بنفس الساعة عبر المؤسسات | ذروة 20,000 متزامن (قرار #21) | التصميم يفترض التداخل الكامل؛ Supavisor + مراجعة Dedicated Compute (Core Data D-04) |

### 6.2 — تحليل أنماط الفشل (Failure Mode Analysis)

| المكون | نمط الفشل | الاحتمالية | الأثر | الكشف | التعافي |
|---|---|---|---|---|---|
| PostgreSQL | تعطل كامل | منخفضة | 🔴 حرج (لا كتابة؛ القراءة تنجو بالكاش) | health check كل 10s | failover مُدار؛ RTO ≤ 5 دقائق؛ PITR |
| PostgreSQL | تدهور أداء (فهرس منسي/خطة سيئة) | متوسطة | 🟡 بطء الجدول بالذروة | تنبيه p95 > 300ms لمدة 5 دقائق | pg_stat_statements + الكاش يمتص الأثر |
| قارئ الـ Outbox | توقف عن المعالجة | متوسطة | 🟡 تأخر الإشعارات (لا ضياع) | تنبيه: أقدم صف غير معالج > 5 دقائق | إعادة تشغيل — الالتقاط تلقائي |
| RLS Policy | ثغرة بعد تعديل (تسريب دور أو مؤسسة) | منخفضة | 🔴 أخطر فشل في النظام | pgTAP في CI: دور × عملية × مؤسسة قبل كل نشر | rollback فوري للـ policy؛ مراجعة سجلات الوصول |
| كاش Hive | تلف ملف الكاش على جهاز | منخفضة | 🟢 جهاز واحد | فشل فتح الصندوق | مسح وإعادة مزامنة كاملة تلقائياً |
| FCM | انقطاع إقليمي | منخفضة | 🟡 لا إشعارات push | ارتفاع أخطاء الدفع | الاستباقي محلي أصلاً (A-07)؛ التغييرات تصل بأول فتح |

### 6.3 — نموذج التهديدات (STRIDE)

| التهديد | مثال ملموس | الدفاع المحدد |
|---|---|---|
| Spoofing | طالب ينتحل JWT محاضر لنشر «إلغاء محاضرة» مزيف | توقيع JWT من Supabase Auth؛ الدور و`institution_id` من claims الخادم — يُحقنان server-side ولا يعدّلهما العميل |
| Tampering | محاضر يعدّل حدث امتحان أو حدث مادة زميله أو حدث مؤسسة أخرى عبر API مباشر | RLS: `institution_id = my_institution()` أولاً، ثم `created_by = auth.uid()` + `teaches()`؛ exam وcultural محصوران بـ admin في policies مستقلة |
| Repudiation | إداري ينكر رفضه طلب قاعة | `room_requests` append-only بـ `reviewed_by` + `rejection_reason` + طابع زمني؛ لا سياسة DELETE لأي دور |
| Information Disclosure | طالب يستعلم امتحانات دفعات أخرى أو أحداث مؤسسة أخرى | سياسة `events_visibility`: المؤسسة + `approved` + (تسجيل ACTIVE ∨ cultural) + الحجب المالي؛ طلبات pending غير مرئية لغير المنشئ والإداري |
| Denial of Service | إغراق قراءة الجدول أو دفعات ضخمة | Rate limits لكل RPC؛ حد 200 صف؛ نطاق 60 يوماً؛ delta sync يجعل الطلب الطبيعي < 1KB |
| Elevation of Privilege | محاضر يستدعي `schedule_exams_bulk` أو ينشئ cultural | فحص الدور في الـ RPC **+** RLS تمنع الإدراج لغير admin — طبقتان مستقلتان (القراران #10 و#19 مفروضان مرتين) |

### 6.4 — قائمة تدقيق أمنية إلزامية

- [x] **المصادقة والتفويض:** Supabase Auth (JWT مع `institution_id` claim) + RLS على كل جدول — `event_outbox` بلا وصول لأي دور عميل (service_role فقط)
- [x] **تشفير البيانات:** in-transit: TLS 1.2+؛ at-rest: AES-256؛ كاش Hive بصندوق مشفّر بمفتاح في التخزين الآمن للجهاز
- [x] **إدارة الأسرار:** مفاتيح FCM وservice_role في أسرار الخادم فقط — لا تغادر الخادم أبداً
- [x] **حقن SQL:** PostgREST مُعلمَن؛ RPCs باستعلامات مُعاملة حصراً؛ ممنوع `EXECUTE` بنص مركّب (= Core Data §6.4)
- [x] **سجلات التدقيق:** كل قرار مراجعة، كل تعديل/إلغاء حدث (من، متى، ماذا في payload الـ outbox). لا يُسجَّل: توكنات، أو PII خارج معرّف المستخدم

### 6.5 — الملاحظة والمراقبة (Observability)

| # | المقياس | عتبة التنبيه |
|---|---|---|
| 1 | p95 لاستعلام جدول الطالب | > 300ms لمدة 5 دقائق متصلة |
| 2 | زمن فحص أي سياسة RLS | > 100ms |
| 3 | عمر أقدم صف outbox غير معالج | > 5 دقائق |
| 4 | معدل أخطاء 5xx على RPCs الجدولة | > 1% في نافذة 10 دقائق |
| 5 | صفوف outbox بـ attempts ≥ 5 | ≥ 1 صف (تحقيق يدوي فوري) |
| 6 | استعلام تدقيق يومي: تعارضات قاعات في approved (canary للقيد) | ≥ 1 = خلل جوهري، تصعيد فوري |
| 7 | استعلام تدقيق يومي: أي join بين حدث ومادة بمؤسستين مختلفتين (canary للعزل) | ≥ 1 = تصعيد فوري |

---

## القسم 7: خطة التنفيذ وخارطة الطريق 🗺️

> **موقعنا في الخارطة الكلية:** نظام الجدولة هو جزء أساسي من **المرحلة 2** في خارطة Core Data §7.2 (الجدولة والملفات والإشعارات) ويعتمد على اكتمال مرحلتها 1 (الهوية والهيكل الأكاديمي والتسجيل). المراحل أدناه تفصيل داخلي لحصة الجدولة من تلك المرحلة وما بعدها.

### 7.1 — ترتيب المخاطر (Risk-First Ordering)

**أخطر افتراض تقني:** أن قيد `no_room_overlap` مع الفهرس الجزئي يمنع فعلاً كل سباقات التعارض تحت حمل متزامن **داخل بيئة multi-tenant**، وأن المزامنة التفاضلية بـ `updated_at` لا تُسقط تغييرات (خصوصاً soft deletes) لأجهزة غابت طويلاً. كلاهما يُختبر في المرحلة 0.

### 7.2 — المراحل

#### المرحلة 0: إثبات الجدوى (Spike) — 3 أيام
- **الهدف:** التحقق من [S-03] و[S-04] على schema Core Data الفعلي
- **المهام:** (1) سكربت يطلق 100 إدراج متزامن لنفس القاعة والوقت من مؤسستين مختلفتين (يجب أن يمر إدراج واحد لكل قاعة — والقاعات معزولة بالمؤسسة عبر `room_id`)، (2) نموذج delta sync مع soft delete وجهاز «غائب» يتحقق من وصول كل التغييرات
- **معيار النجاح/الفشل:** نجاح واحد بالضبط من الإدراجات المتزامنة لكل قاعة + صفر تغيير مفقود. **إن فشل EXCLUDE:** الخطة البديلة `pg_advisory_xact_lock(room_id)` — أبطأ لكنه صحيح

#### المرحلة 1: الكائنات المُضافة والـ RLS — نصف أسبوع
- **المهام:** (1) ترحيل `event_series` + `event_outbox` + `events.series_id` (القسم 4.2)، (2) triggers الـ outbox، (3) اختبارات pgTAP للسياسات المُضافة (دور × عملية × مؤسسة)
- **الاعتماديات:** المرحلة 0 + المرحلة 1 من Core Data (A-05)
- **المخرج:** حزمة اختبارات RLS تمر بالكامل في CI — تشمل محاولات عابرة للمؤسسات

#### المرحلة 2: مسارات الكتابة (اقتراح + مراجعة + نشر مباشر + دفعي) — أسبوع ونصف
- **المهام:** (1) `propose_room_event` بالتحقق الكامل + توليد السلسلة، (2) `publish_direct_event` (واجب/بحث/ثقافي)، (3) `review_room_request` بمعاملة ذرّية، (4) `schedule_exams_bulk` بالـ idempotency (ينشئ events + exams معاً)، (5) `update_event`/`cancel_event` مع optimistic locking، (6) الفحص الاستباقي للتعارض
- **المخرج:** دورة كاملة عبر curl: اقتراح ← موافقة ← نشر ← تعديل ← إلغاء، مع كل أكواد الخطأ الموثقة في 5.2
- **اختبارات:** unit للتحقق لكل نوع + تكامل لسيناريوهات 409 المتزامنة + محاولات عابرة للمؤسسات

#### المرحلة 3: جدول الطالب + الأوفلاين — أسبوع ونصف
- **المهام:** (1) قراءة الجدول عبر PostgREST + دلتا، (2) طبقة Hive والدمج، (3) بطاقات الأنواع السبعة في Flutter، (4) جدولة الإشعارات المحلية «قبل 30 دقيقة» وإعادة بنائها عند كل مزامنة، (5) شارة «آخر تحديث منذ...»، (6) سلوك الحجب المالي في الواجهة (تحسين UX فوق RLS)
- **المخرج:** فتح الجدول بوضع الطيران بعد أول مزامنة + استلام الدلتا عند العودة + طالب متأخر مالياً يرى بطاقات الامتحانات فقط
- **اختبارات:** unit لمنطق الدمج (يشمل soft delete) + E2E «غائب أسبوعاً» + E2E الحجب الانتقائي

#### المرحلة 4: الـ Outbox والإشعارات — أسبوع
- **المهام:** (1) Edge Function المجدولة + retry/attempts، (2) التسليم لواجهة نظام الإشعارات، (3) تنظيف الصفوف المعالجة (pg_cron)، (4) لوحة مقاييس القسم 6.5 وتنبيهاتها
- **المخرج:** تعديل حدث ← وصول push خلال ≤ 60 ثانية، وقياس موثّق
- **اختبارات:** تكامل (فشل FCM المُحاكى لا يُضيع صفاً)

#### المرحلة 5: التقسية والتسليم — أسبوع
- **المهام:** (1) اختبار حمل يحاكي ذروة صباحية عبر 10 مؤسسات على قراءة الجدول والتحقق من p95، (2) مراجعة قائمة 6.4، (3) اختبارا التدقيق اليوميان (canary التعارض + canary العزل)، (4) runbook للتنبيهات، (5) خطة rollback مجربة
- **المخرج:** تقرير حمل + كل بنود Definition of Done ✅

### 7.3 — مخطط الاعتماديات

\`\`\`mermaid
graph LR
    CD1[Core Data مرحلة 1:<br/>الهوية والتسجيل] --> P0[مرحلة 0: Spike<br/>3 أيام]
    P0 --> P1[مرحلة 1: الكائنات المُضافة + RLS<br/>نصف أسبوع]
    P1 --> P2[مرحلة 2: مسارات الكتابة<br/>1.5 أسبوع]
    P2 --> P3[مرحلة 3: الجدول + الأوفلاين<br/>1.5 أسبوع]
    P2 --> P4[مرحلة 4: Outbox + إشعارات<br/>أسبوع]
    P3 --> P5[مرحلة 5: التقسية والتسليم<br/>أسبوع]
    P4 --> P5
\`\`\`

> المرحلتان 3 و4 قابلتان للتوازي بين مطوّرين — الإجمالي التقديري: **5–6 أسابيع** لمطوّرَين (انخفض عن v1.1 لأن الـ schema الأساسي أصبح مملوكاً لـ Core Data).

### 7.4 — تعريف "الانتهاء" (Definition of Done)

- [ ] كل الاختبارات تمر (تغطية ≥ 85% للمسارات الحرجة: التحقق لكل نوع، المراجعة، كشف التعارض، دمج الدلتا)
- [ ] اختبارات pgTAP لكل سياسة مُضافة: **دور × عملية × مؤسسة** — تشمل محاولات عابرة للمؤسسات
- [ ] توثيق الـ RPCs (القسم 5) مطابق للسلوك الفعلي ومحدَّث
- [ ] مقاييس القسم 6.5 السبعة تعمل وتنبيهاتها مضبوطة ومجرّبة
- [ ] قائمة الأمان (6.4) مُراجعة بالكامل مع توقيع المراجع
- [ ] اختبار الحمل يثبت p95 < 300ms تحت ذروة الـ 10 مؤسسات
- [ ] استعلاما التدقيق اليوميان (التعارض + العزل) يعملان ونتيجتهما 0
- [ ] خطة rollback موثقة ومجرّبة (استعادة migration + إبطال policy)

---

## ملحق: خريطة تغطية المتطلبات (Traceability)

| المتطلب / القرار | أين غُطّي في هذه الوثيقة |
|---|---|
| MVP §3.1.1 — بطاقات الأنواع السبعة | استهلاك `events` (4.1) + بطاقات Flutter (المرحلة 3) |
| MVP §3.1.2 — الجدول المُفلتر + أوفلاين + إشعارات | [S-04] + العملية 2 (3.3) + المرحلتان 3 و4 |
| MVP §3.1.2 — سياسة العرض عند الحجب المالي (قرار #15) | سياسة `events_visibility` (4.1) + حالة حافة 6.1 + E2E المرحلة 3 |
| MVP §4.1 — اقتراح المحاضر + النشر المباشر | `propose_room_event` + `publish_direct_event` (5.2) |
| MVP §5.1.1 / قرار #10 — جدولة الامتحان حصرية للإداري | CHECK على `room_requests.requested_type` + `schedule_exams_bulk` (admin حصراً) + STRIDE/EoP |
| MVP §5.1.2 — مراجعة الطلبات بقبول/رفض موثَّق | `review_room_request` + `room_requests` append-only [S-02] |
| MVP §5.2 / قرار #19 — الحدث الثقافي حصري للإداري | `publish_direct_event` (cultural = admin) + RLS طبقة ثانية |
| قرار #6 — الأحداث المتطلبة للحضور | اشتقاق من `type` (4.1) + عقد نظام الحضور (5.3) |
| قرار #20 — تعدد المؤسسات | القسم 4.0 + كل الفهارس والسياسات + canary العزل (6.5) |
| قرار #21 — 20,000 متزامن | A-01 + حالة حافة «10 امتحانات متزامنة» + اختبار حمل المرحلة 5 |
| Core Data [D-05]/[D-08] — RLS + JWT claim | `my_institution()` في كل سياسة مُضافة (4.2) |
| Core Data [D-06] — delta sync | [S-04] + فهرس `(institution_id, updated_at)` |
| Core Data [D-07] — PostgREST + RPC | مبادئ القسم 5.1 وكل الـ endpoints |
| MVP §5.0 — التوثيق وعدم الحذف | soft delete على events + room_requests بلا DELETE (4.4) |

*— نهاية الوثيقة — Scheduling & Events TDD v2.0 (الموحّدة معمارياً) —*