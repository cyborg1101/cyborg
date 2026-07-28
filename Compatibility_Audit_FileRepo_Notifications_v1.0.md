# 📋 قائمة تدقيق التوافق — نظاما مستودع الملفات والإشعارات
## مقابل MVP Functions v3.6 و Core Data Layer TDD v2.1 (الموحّدة)

---

## بيانات ضبط الوثيقة

| البند | التفاصيل |
|---|---|
| **اسم الوثيقة** | قائمة تدقيق التوافق — File Repository TDD v1.0 & Notifications TDD v1.0 |
| **رقم الإصدار** | v1.0 |
| **حالة الوثيقة** | ملزمة — تُطبَّق تعديلاتها قبل أي تنفيذ |
| **الوثائق الخاضعة للتدقيق** | tdd_file_repository.md (v1.0) · TDD_Notifications_System_v1.0.md |
| **المرجعية الحاكمة** | MVP Functions v3.6 · TDD Core Data Layer v2.1 (الموحّدة) |
| **النمط المتبع** | نفس منهجية «قائمة تدقيق التوافق v1.6» و«سجل حسم التعارضات» في Core Data v2.1 |

### قاعدة الحسم المعتمدة (تسري على كل بند أدناه)

1. **Core Data v2.1 هي المرجع الملزم للـ Schema والأمان** (تعدد المؤسسات، الفهارس، دوال RLS، أنواع ENUM، `profiles` لا `users`).
2. **MVP v3.6 هو المرجع الوظيفي** (القرارات #1–#23).
3. **Notifications TDD هو المالك الحصري لقناة التسليم** ([D-06] فيها) — أي نظام آخر يُنتج أحداثاً فقط.
4. أي تعارض يُحسم لصالح هذه المرجعية، ويُوثَّق هنا برقم بند.

---

# القسم 1: تدقيق File Repository TDD v1.0 🗂️

## 1.1 — جدول المخالفات والتصحيحات الملزمة

| # | البند | المخالفة المكتشفة | الخطورة | التصحيح الملزم |
|---|---|---|---|---|
| FR-01 | تعدد المؤسسات | لا وجود لـ `institution_id` إطلاقاً: لا في `repo_folders` ولا `files` ولا `file_notes`، لا في الفهارس، لا في مسارات التخزين، لا في سياسات RLS — مخالفة مباشرة لقاعدة Core Data §4.0 وقرار MVP #20 | 🔴 حرج | إضافة `institution_id` لكل الجداول؛ كل فهرس يبدأ به؛ `storage_path` يبدأ إلزامياً بـ `{institution_id}/`؛ كل سياسة RLS تبدأ بـ `institution_id = my_institution()` |
| FR-02 | ازدواجية schema جدول `files` | الوثيقة تعرّف `files` من الصفر (UUID PK، `origin` TEXT، `display_name`) بينما Core Data v2.1 §4.2-8 تعرّفه رسمياً (BIGINT identity، `is_contribution` BOOLEAN، `file_name`، `institution_id`) | 🔴 حرج | **Core Data تفوز.** جدول `files` الرسمي هو المعتمد؛ File Repo TDD تضيف أعمدة امتداد فقط (`folder_id`, `mime_type`, `sha256`, `reviewed_at`) عبر migration — انظر القسم 4.1. `origin` يُشتق: `is_contribution = false` → LECTURER |
| FR-03 | قيم `review_status` | الوثيقة تستخدم `'PENDING'/'ACCEPTED'/'REJECTED'` (uppercase TEXT) بينما Core Data تعرّف ENUM `review_status ('pending','accepted','rejected')` | 🟠 عالٍ | اعتماد ENUM الـ Core Data حرفياً (lowercase) في الـ trigger والسياسات وكل العقود |
| FR-04 | دوال RLS المساعدة | الوثيقة تخترع `is_enrolled_active` / `is_course_lecturer` / `is_subscription_blocked` — دوال غير موجودة في Core Data | 🔴 حرج | التوحيد الإلزامي: `is_enrolled()` (تفحص ACTIVE أصلاً) · `teaches()` · `NOT has_active_subscription()` — الأخيرة **مصدر الحقيقة الوحيد** لحالة الاشتراك (MVP §3.7.2)؛ ممنوع أي منطق اشتراك مستقل |
| FR-05 | مراجع الجداول | `REFERENCES users(id)` و`courses(id)` كـ UUID | 🟠 عالٍ | `profiles(id)` (UUID) و`courses(id)` (BIGINT) حسب Core Data |
| FR-06 | تكامل الإشعارات | نمط Outbox (`notification_outbox` + حدث `FILE_PUBLISHED` + at-least-once) يخالف [D-06] في Notifications TDD (trigger مباشر إلى `notifications` داخل نفس الـ transaction = exactly-once كتابةً) | 🔴 حرج | **حذف الـ outbox كلياً.** trigger مباشر بأنواع قاموس Notifications TDD §4.5: `file_added` عند النشر و`contribution_reviewed` عند القبول/الرفض، ملفوفاً بـ `EXCEPTION WHEN OTHERS` حتى لا يُفشل العملية الأصلية |
| FR-07 | أرقام الحمل | «~2,000 طالب و ~80 محاضراً» + كلفة egress محسوبة لمؤسسة واحدة | 🟠 عالٍ | الأرقام الرسمية: 2,000 طالب **لكل مؤسسة** × 10 = 20,000 (قرار #21)؛ موجة تحميل 10GB/محاضرة قد تتزامن عبر 10 مؤسسات؛ معيار الكلفة (≤ 40$/شهر) يُعاد تسعيره على المستوى الكلي مع حصة تخزين لكل مؤسسة (MVP §1.2) |
| FR-08 | `sign-download` والعزل المؤسسي | التحميل لا يمر عبر RLS (رابط موقّع) والدالة لا تفحص المؤسسة | 🔴 حرج | فحص إلزامي داخل الدالة: `institution_id` من JWT claim = `institution_id` الخاص بالملف، قبل أي فحص أهلية آخر |
| FR-09 | المرجعية | الوثيقة تحيل إلى MVP v3.5 | 🟡 متوسط | تحديث الإحالة إلى MVP v3.6 + Core Data v2.1 |
| FR-10 | حد حجم ملف الطالب 25MB | ليست مخالفة — قيد أضيق من سقف 50MB في Core Data | ✅ متوافق | يبقى؛ يُفرض في `finalize-upload` كما هو |
| FR-11 | Soft delete + TTL الروابط + توقيت الخادم | متوافقة مع Core Data §4.4 ومبدأ TIMESTAMPTZ | ✅ متوافق | لا تعديل |

---

# القسم 2: تدقيق Notifications TDD v1.0 🔔

## 2.1 — جدول المخالفات والتصحيحات الملزمة

| # | البند | المخالفة المكتشفة | الخطورة | التصحيح الملزم |
|---|---|---|---|---|
| NT-01 | تسريب عبر FCM Topics | المواضيع `all_users` / `role_students` / `role_lecturers` **عالمية** — إعلان مؤسسة يصل push لكل مؤسسات المنصة العشر. هذا حرفياً «أخطر فشل ممكن» حسب MVP §1.1 | 🔴 حرج (الأخطر في التدقيق كله) | كل المواضيع مؤسسية حصراً: `inst_{institution_id}_all` / `inst_{institution_id}_students` / `inst_{institution_id}_lecturers`؛ الاشتراك عند الدخول يُشتق من `institution_id` claim في JWT — لا موضوع عالمي بأي صيغة |
| NT-02 | `announcements` بلا `institution_id` | schema الوثيقة لا يحمل العمود بينما Core Data §4.2-13 يعرّفه إلزامياً | 🔴 حرج | اعتماد جدول `announcements` من Core Data كأساس، وأعمدة الوثيقة (`target_type`, `segment_key`, `scheduled_at`, `push_sent_at`, `status`) تُضاف كامتداد — القسم 4.2 |
| NT-03 | ازدواجية schema جدول `notifications` | Core Data: `(kind, payload JSONB, is_pinned)` مقابل الوثيقة: `(type, ref_type, ref_id, course_id, title, body + قيد dedup)` | 🟠 عالٍ | **مخطط Notifications TDD الأغنى يفوز هنا** (هو وثيقة النظام المالك) ويُرحَّل التعديل إلى Core Data v2.2؛ `is_pinned` يُحذف من `notifications` — التثبيت خاصية إعلانات حصراً (MVP §5.2) |
| NT-04 | مراجع وأنواع | `REFERENCES users(id)` · `course_id UUID` · `ref_id UUID` | 🟠 عالٍ | `profiles(id)`؛ `course_id BIGINT`؛ `ref_id` يصبح `TEXT` لأن معرّفات Core Data مختلطة (BIGINT للأحداث/الملفات، UUID للمستخدمين/الدفعات) |
| NT-05 | أرقام الحمل | A-01: «~5,000 طالب + 300 محاضر» لمؤسسة واحدة + «Realtime محدود بـ ~500 اتصال يكفي» | 🔴 حرج | النموذج الرسمي: 20,000 مستخدم عبر 10 مؤسسات (قرار #21)؛ عبارة «500 اتصال تكفي» **تُلغى** — ميزانية Realtime تُحسب على أساس 10 مؤسسات وهي «أثقل مستهلك للحصة بعد الامتحانات» (MVP §1.2)، ورفع الحصة بند مُجدول من اليوم الأول (Core Data [D-04]) |
| NT-06 | فئة `late_payers` | «استعلام ثابت» على جداول الدفعات — منطق اشتراك مستقل | 🔴 حرج | الفئة تُبنى **حصراً** فوق `has_active_subscription()` — MVP §3.7.2 يمنع أي نظام من احتساب حالة الاشتراك بمنطق مستقل |
| NT-07 | تذكير الـ 30 دقيقة | استعلام «الطلاب المسجلون في المادة» بلا فلترة حالة التسجيل | 🟠 عالٍ | فلترة `enrollments.status = 'ACTIVE'` إلزامية (دورة حياة Enrollment في Core Data §4.2-4) — الطالب COMPLETED/DROPPED لا يُذكَّر |
| NT-08 | `announce_level` | TEXT CHECK بنفس القيم بدل ENUM الـ Core Data | 🟡 متوسط | اعتماد ENUM `announce_level` الموجود |
| NT-09 | المرجعية | إحالة إلى MVP v3.5 | 🟡 متوسط | تحديث إلى v3.6 + Core Data v2.1 |
| NT-10 | الطالب المحجوب اشتراكياً يستقبل الإشعارات | قرار سليم لكنه غير مُسند | ✅ يُثبَّت | يُوثَّق كاستثناء صريح ومتعمد من قرار #15: الإشعارات قناة إبلاغ لا «ميزة» — الطالب المحجوب يجب أن يعرف قبول دفعته واقتراب امتحانه |
| NT-11 | fan-out هجين [D-03] + soft delete + توقيت الخادم + idempotency | متوافقة كلياً مع مبادئ Core Data | ✅ متوافق | لا تعديل |

---

# القسم 3: مصفوفة الاتساق المعماري عبر الوثائق الأربع 🏗️

| المحور المعماري | MVP v3.6 | Core Data v2.1 | File Repo v1.0 | Notifications v1.0 |
|---|---|---|---|---|
| تعدد المؤسسات (`institution_id` + فهارس + claims) | ✅ قرار #20 | ✅ §4.0 | ❌ غائب → FR-01 | ❌ غائب → NT-01/02 |
| سيناريو الحمل 20,000 | ✅ قرار #21 | ✅ | ❌ 2,000 → FR-07 | ❌ 5,330 → NT-05 |
| `has_active_subscription()` مصدر وحيد | ✅ §3.7.2 | ✅ | ❌ دالة مستقلة → FR-04 | ❌ استعلام مستقل → NT-06 |
| `profiles` لا `users` · أنواع المعرّفات | — | ✅ | ❌ → FR-05 | ❌ → NT-04 |
| قناة إشعارات مركزية (trigger لا outbox) | — | ✅ §5.3 | ❌ outbox → FR-06 | ✅ [D-06] |
| Soft delete للسجلات الموثِّقة | ✅ §5.0 | ✅ §4.4 | ✅ | ✅ |
| كل التوقيتات بـ `now()` على الخادم | ✅ | ✅ | ✅ | ✅ |
| Idempotency على مسارات الكتابة الحرجة | — | ✅ | ✅ (UNIQUE storage_path) | ✅ (uq_notification_dedup) |
| RLS server-side حصراً (الواجهة UX فقط) | ✅ §3.1.2 | ✅ | ✅ | ✅ |

**الخلاصة:** الوثيقتان سليمتان في المبادئ التشغيلية (idempotency، soft delete، توقيت الخادم، RLS) لكنهما كُتبتا قبل اعتماد التعدد المؤسسي — كل المخالفات الحرجة تعود لهذا الجذر الواحد.

---

# القسم 4: التعديلات الملزمة على مستوى الـ Schema 🗄️

## 4.1 — مستودع الملفات (يحل محل §4.2 في File Repo TDD)

\`\`\`sql
-- repo_folders: جدول جديد يُضاف كامتداد رسمي لـ Core Data (migration إضافية)
CREATE TABLE repo_folders (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    institution_id UUID NOT NULL REFERENCES institutions(id),
    course_id      BIGINT NOT NULL REFERENCES courses(id),
    event_id       BIGINT REFERENCES events(id),
    title          TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 120),
    sort_order     INT  NOT NULL DEFAULT 0,
    created_by     UUID NOT NULL REFERENCES profiles(id),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ
);
CREATE INDEX idx_repo_folders_tenant
    ON repo_folders (institution_id, course_id) WHERE deleted_at IS NULL;

-- جدول files الرسمي هو تعريف Core Data v2.1 §4.2-8 — هذه أعمدة الامتداد فقط:
ALTER TABLE files ADD COLUMN folder_id   BIGINT REFERENCES repo_folders(id);
ALTER TABLE files ADD COLUMN mime_type   TEXT;
ALTER TABLE files ADD COLUMN sha256      TEXT;     -- تمهيد الـ dedup المستقبلي (دين تقني File Repo §2.3)
ALTER TABLE files ADD COLUMN reviewed_at TIMESTAMPTZ;
CREATE UNIQUE INDEX uq_files_storage_path ON files (storage_path); -- idempotency للـ finalize

-- سياسات RLS الموحّدة (بدوال Core Data حصراً + فحص المؤسسة أولاً)
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

-- تكامل الإشعارات: trigger مباشر (يحل محل نمط Outbox الملغى — FR-06)
-- عند review_status → 'accepted' أو ملف محاضر جديد:
--   INSERT INTO notifications (user_id, type, ref_type, ref_id, course_id, ...)
--   بأنواع القاموس: 'file_added' لطلاب المادة ACTIVE · 'contribution_reviewed' لصاحب الاجتهاد
--   ملفوفاً بـ BEGIN...EXCEPTION WHEN OTHERS — فشل الإشعار لا يُفشل العملية الأصلية أبداً
\`\`\`

## 4.2 — الإشعارات (يحل محل §4.2 في Notifications TDD)

\`\`\`sql
-- announcements: أساسه تعريف Core Data v2.1 §4.2-13 (يحمل institution_id أصلاً)
-- أعمدة الامتداد التي تضيفها وثيقة الإشعارات:
ALTER TABLE announcements ADD COLUMN target_type TEXT NOT NULL DEFAULT 'all'
    CHECK (target_type IN ('all','students','lecturers','segment'));
ALTER TABLE announcements ADD COLUMN segment_key TEXT
    CHECK (segment_key IN ('late_payers','frequent_absentees'));
ALTER TABLE announcements ADD COLUMN scheduled_at TIMESTAMPTZ;
ALTER TABLE announcements ADD COLUMN status TEXT NOT NULL DEFAULT 'published'
    CHECK (status IN ('draft','scheduled','published','retracted'));
ALTER TABLE announcements ADD COLUMN push_sent_at TIMESTAMPTZ;
ALTER TABLE announcements ADD CONSTRAINT segment_consistency
    CHECK ((target_type = 'segment') = (segment_key IS NOT NULL));

-- notifications: مخطط وثيقة الإشعارات يفوز (NT-03) — بالتصحيحات NT-04:
--   user_id  UUID REFERENCES profiles(id)
--   course_id BIGINT REFERENCES courses(id)
--   ref_id   TEXT (معرّفات مختلطة BIGINT/UUID عبر الأنظمة)
--   حذف is_pinned (التثبيت خاصية إعلانات حصراً)
--   قيد uq_notification_dedup (user_id, type, ref_id) يبقى كما هو

-- device_tokens و announcement_recipients و announcement_states:
--   profiles(id) بدل users(id) — لا تغيير بنيوياً آخر
\`\`\`

## 4.3 — قاعدة FCM Topics الملزمة (تصحيح NT-01)

| الموضوع الملغى | البديل الإلزامي |
|---|---|
| `all_users` | `inst_{institution_id}_all` |
| `role_students` | `inst_{institution_id}_students` |
| `role_lecturers` | `inst_{institution_id}_lecturers` |

- الاشتراك عند تسجيل الدخول يُشتق من `institution_id` claim في JWT حصراً.
- إلغاء الاشتراك من كل مواضيع المؤسسة عند الخروج أو تعطيل الحساب.
- دالة `dispatch` ترفض إرسال أي رسالة topic لا يبدأ اسمها بـ `inst_` (حارس برمجي ضد الرجوع للنمط القديم).
- Edge Function `sign-download` (مستودع الملفات) تفحص تطابق `institution_id` claim مع مؤسسة الملف قبل توليد أي رابط موقّع (FR-08).

---

# القسم 5: الإجراءات المطلوبة على الوثائق 📌

| الوثيقة | الإجراء | الإصدار الجديد |
|---|---|---|
| tdd_file_repository.md | تطبيق FR-01 → FR-09 (إعادة كتابة §2.1، §3.1، §4.2، §5.2، §6.3 على الأساس المؤسسي) | v1.1 |
| TDD_Notifications_System | تطبيق NT-01 → NT-09 (إعادة كتابة §2.2 [D-02]، §4.2، §5.3، وأرقام §0.2) | v1.1 |
| TDD_Core_Data_Layer | ترحيل مخطط `notifications` المفصّل (NT-03) + جدول `repo_folders` وأعمدة امتداد `files` (FR-02) | v2.2 |
| MVP_Functions | لا تعديل — الوثيقة سليمة ومرجعية | — |

### تعريف «الانتهاء» لهذا التدقيق

- [ ] كل بند 🔴 حرج مُطبَّق في وثيقته قبل كتابة أي migration
- [ ] اختبار pgTAP عدائي لكل سياسة جديدة: دور × عملية × **مؤسسة** (النمط الإلزامي في Core Data §1.3)
- [ ] اختبار تكاملي: إعلان من مؤسسة A لا يصل push ولا Realtime ولا فيداً لأي مستخدم في مؤسسة B
- [ ] اختبار تكاملي: `sign-download` يرفض بـ 403 أي `file_id` من مؤسسة مختلفة عن claim الطالب

---

*— نهاية الوثيقة — Compatibility Audit v1.0 —*