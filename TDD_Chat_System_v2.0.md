# 💬 وثيقة التصميم التقني — نظام الدردشة والتواصل الداخلي
## Academic Hub — Chat System TDD

---

## بيانات ضبط الوثيقة

| البند | التفاصيل |
|---|---|
| **اسم الوثيقة** | وثيقة التصميم التقني — نظام الدردشة والتواصل الداخلي |
| **رقم الإصدار** | v2.0 |
| **مستوى العمق** | 🟡 Production |
| **الوثائق المرجعية** | MVP Functions v3.6 (§3.4 · §4.4 · §5.3 · قرارات #8 #15 #16) · TDD Core Data Layer v2.1 (المرجع المُلزم للـ schema والأمان) · قالب TDD v1.0 · سجل القرارات المعماري D-01..D-06 |
| **الحالة** | جاهزة للمراجعة الهندسية |
| **مالك الوثيقة** | رامز |

### سجل الإصدارات

| الإصدار | التغيير |
|---|---|
| v1.0 | تصميم أولي — كُتب قبل توحيد Core Data Layer v2.1 وتضمّن تعارضات معمارية |
| v2.0 | إعادة كتابة كاملة للتوافق مع Core Data Layer v2.1 و MVP v3.6: اعتماد جداول core (`chat_posts`/`chat_comments`)، تصحيح أنواع المعرّفات، إضافة أنواع القنوات الأربعة، فرض الحجب المالي server-side، حذف المرفقات من النطاق، توثيق حسم التعارضات |

### سجل حسم التعارضات بين v1.0 والمرجعية المعمارية

| البند المتعارض | v1.0 قالت | المرجعية تقول | القرار في v2.0 والمبرر |
|---|---|---|---|
| نوع `institution_id` | BIGINT | `institutions.id` هو UUID (Core §4.2) | **UUID** — مطابقة المفتاح الأجنبي إلزامية |
| معرّفات الرسائل | UUID لكل شيء | `chat_posts`/`chat_comments` بـ BIGINT IDENTITY | **BIGINT** — الالتزام بجداول core الموجودة فعلاً |
| نموذج الرسائل | جدول واحد `chat_messages` + `parent_message_id` | جدولان منفصلان: منشور وتعليق | **جدولان** — الفصل يطابق فصل الصلاحيات في قرار #16 (المحاضر ينشر، الطالب يعلّق) ويجعل RLS أبسط وأصلب |
| أنواع القنوات | `course` و`support` فقط | core تعرّف 4 أنواع تغطي MVP §5.3 | **الأنواع الأربعة**: `course` · `support` · `admin_lecturer` · `admin_admin` |
| حدود طول النص | 4000 / 2000 | CHECK في core: 8000 للمنشور، 4000 للتعليق | **أرقام core** — القيد موجود على مستوى المحرك أصلاً |
| المرفقات | جدول `chat_message_attachments` | MVP §3.4 نموذج «قراءة + تعليق»؛ الملفات لها مستودع مستقل (§3.3) | **حذف المرفقات من النطاق** — منع ازدواجية مسار الملفات |
| الحجب المالي | غير مذكور | قرار #15: حجب كل الميزات عدا الامتحانات، server-side حصراً | **إضافة `has_active_subscription()` لسياسات قراءة/كتابة الطالب** |
| `institution_id` على الرسائل | موجود (بنوع خاطئ) | core v2.1 لم تضعه على `chat_posts`/`chat_comments` — **فجوة تخالف قاعدة 4.0** | **يُضاف هنا** كامتداد + فهارس مركّبة تبدأ به — ويُرفع كتغذية راجعة لإصدار Core v2.2 |

---

# القسم 0: بروتوكول التحليل المتسلسل 🧠

## 0.1 — إعادة صياغة المشكلة

المطلوب طبقة تواصل مقننة داخل Academic Hub بثلاثة مسارات محسومة في MVP: (أ) قناة واحدة لكل مادة ينشر فيها المحاضر رسمياً ويعلّق الطلاب (قرارا #8 و#16)، (ب) قنوات تنسيق إدارية (إداري↔محاضر، إداري↔إداري)، (ج) مركز دعم فني يستقبل استفسارات الطلاب (MVP §5.3). لا مراسلة 1:1 بين طالب ومحاضر بأي صيغة. النظام يعمل في بيئة 10 مؤسسات معزولة (قرار #20)، والدردشة هي **أثقل مستهلك لحصة Realtime بعد الامتحانات** (MVP §5.3) — أي خطأ في تصميم القنوات أو الفهارس سيظهر كاختناق منصة كاملة لا كمشكلة دردشة.

## 0.2 — الأسئلة الخمسة الحاسمة

1. **من المستخدم الفعلي؟** ~20,000 مستخدم عبر 10 مؤسسات. الطالب يقرأ ويعلّق، المحاضر ينشر في موادّه، الإداري يتواصل مع المحاضرين والإداريين ويرد على الدعم. الذروة: نشر محاضر إعلاناً في قناة مادة كبيرة (حتى 1,000 مسجَّل) → موجة قراءات + إشعارات فورية.
2. **ما هي العملية الأثقل؟** قراءة بامتياز — تقديرياً 90/10. أثقل استعلام: «آخر منشورات القناة + عدد غير المقروء» عند فتح الشاشة الرئيسية. أثقل كتابة: fanout الإشعارات عند منشور جديد في قناة مادة كبيرة (INSERT جماعي واحد لكل المسجَّلين ACTIVE).
3. **ماذا لو توقف النظام ساعة؟** لا فقدان رسائل مقبول إطلاقاً؛ تأخر الوصول مقبول. الدردشة ليست على المسار الحرج للامتحانات — availability المطلوب هو 99.5% العادي (لا يرث نافذة 99.95% الامتحانية).
4. **ما البيانات الأكثر حساسية؟** بالترتيب: (1) دردشات التنسيق إداري↔محاضر (قد تتضمن شؤون طلاب)، (2) محتوى تذاكر الدعم، (3) منشورات المواد. ويعلوها القاعدة العرضية: صفر تسريب عبر حدود مؤسسة.
5. **أضيق عنق زجاجة خلال 6 أشهر؟** حصة اتصالات Realtime المتزامنة عبر 10 مؤسسات. الدفاع: الاشتراك في قناة Realtime واحدة فقط (القناة المفتوحة على الشاشة) لا كل قنوات المستخدم، + fallback إلى polling كل 60 ثانية (نمط Core §3.2).

## 0.3 — الافتراضات المعلنة

| # | الافتراض | مستوى الثقة | تأثيره لو كان خاطئاً |
|---|---|---|---|
| A-01 | قناة المادة تُنشأ تلقائياً (trigger) عند إنشاء صف `courses` | عالي | لو يدوياً: خطوة إدارية إضافية فقط — الـ schema لا يتغير |
| A-02 | قناة الدعم تُنشأ كسولاً (lazy) عند أول استفسار من الطالب | متوسط | لو مسبقاً: 20,000 صف خامل — مقبول لكنه هدر |
| A-03 | متوسط قناة المادة ~200 مسجَّل، الأقصى ~1,000 | متوسط | لو تجاوز: مراجعة fanout الإشعارات نحو صف معالجة async |
| A-04 | لا تعديل على منشور/تعليق بعد النشر — حذف soft فقط | عالي | لو طُلب التعديل: قرار منتج جديد + عمود `edited_at` + سجل تدقيق |
| A-05 | الطالب المحجوب مالياً لا يصل للدعم داخل التطبيق (تبعة مباشرة لنص قرار #15 الحرفي) | متوسط | لو عُدّل القرار: استثناء قناة `support` من الحجب هو تغيير سياسة RLS واحدة |

## 0.4 — نطاق العمل (Scope Fence)

- ✅ **داخل النطاق:** أنواع القنوات الأربعة · منشورات المحاضر وتعليقات الطلاب في قناة المادة · قنوات التنسيق الإدارية · مركز الدعم · حالة القراءة (unread) · تكامل الإشعارات · Realtime + fallback · العزل المؤسسي والحجب المالي
- ❌ **خارج النطاق:** مراسلة 1:1 طالب↔محاضر (قرار #16) · المرفقات (مستودع الملفات MVP §3.3) · تعديل الرسائل · التفاعلات (reactions) · البحث النصي الكامل FTS (دين مؤجل §2.3) · أي قناة عابرة للمؤسسات

---

# القسم 1: الملخص التنفيذي وسياق المشكلة 📋

## 1.1 — بيان المشكلة

بدون قناة رسمية واحدة لكل مادة، يتشتت التواصل الأكاديمي على واتساب وتيليغرام: رسائل رسمية تضيع، طلاب خارج المجموعات لا تصلهم التحديثات، ولا سجل يمكن الرجوع إليه عند نزاع. وبدون قناة تنسيق موثقة بين الإداري والمحاضر، تتخذ قرارات تشغيلية شفهياً بلا أثر. تكلفة عدم الحل: فقدان مصدر الحقيقة للتواصل الرسمي، وتسريب معلومات أكاديمية عبر قنوات لا تخضع لأي عزل مؤسسي.

## 1.2 — الهدف القابل للقياس

تمكين طالب في مؤسسة من فتح قناة مادته وقراءة آخر 30 منشوراً مع عدد غير المقروء خلال أقل من 300ms (p95 server-side)، ووصول منشور المحاضر الجديد إلى الأجهزة المفتوحة على القناة خلال أقل من 5 ثوانٍ، مع صفر تسريب عبر الأدوار أو المؤسسات (مُختبر آلياً بـ pgTAP لكل دور × عملية × مؤسسة).

## 1.3 — معايير النجاح

| معيار | القيمة المستهدفة | كيف يُقاس |
|---|---|---|
| p95 لاستعلام فتح القناة (آخر 30 منشوراً + unread) | < 300ms | pg_stat_statements |
| زمن وصول منشور جديد لجهاز مفتوح على القناة | < 5s | Realtime + قياس من العميل |
| تسريب بين المؤسسات أو الأدوار | 0 حالة | pgTAP: دور × عملية × مؤسسة لكل جدول |
| طالب محجوب مالياً يصل لأي قناة | 0 حالة | اختبار تكامل مع `subscription_status` منتهية |
| تكرار رسالة عند إعادة إرسال أوفلاين | 0 صف مكرر | اختبار idempotency بـ `client_uuid` |
| fanout إشعارات منشور لقناة 1,000 طالب | < 3s لإتمام الـ INSERT الجماعي | قياس زمن الـ trigger |

## 1.4 — ما ليس هذا النظام

- ليس تطبيق مراسلة فورية عام — نموذج «منشور + تعليقات» مقيد بالأدوار
- ليس بديلاً عن نظام الإعلانات («البورد الأسود» للإداري حصراً — MVP §5.2)
- ليس مساراً للملفات — المرفقات الأكاديمية تمر عبر مستودع الملفات حصراً
- لا قرارات آلية — حذف أي رسالة مخالفة قرار بشري (محاضر أو إداري)

---

# القسم 2: القيود التقنية واختيار التقنيات 🔧

## 2.1 — القيود المفروضة (Hard Constraints)

| نوع القيد | التفصيل | مصدره |
|---|---|---|
| تقني | كل جدول tenant-scoped يحمل `institution_id` UUID وكل فهرس يبدأ به | Core §4.0 (قاعدة غير قابلة للاستثناء) |
| تقني | `institution_id` من JWT claim عبر `my_institution()` — لا subquery | Core [D-08] |
| وظيفي | الطالب: قراءة + تعليق فقط؛ المحاضر وحده ينشئ منشوراً في قناة مادته | قرارا #8 و#16 |
| وظيفي | لا مراسلة 1:1 طالب↔محاضر بأي صيغة | MVP §3.4 |
| وظيفي | الحجب المالي server-side عبر RLS و`has_active_subscription()` حصراً | قرار #15 + Core §4.2 |
| وظيفي | الإداري يستقبل إشعارات الدردشات | MVP §5.2 |
| تنظيمي | soft delete دائماً — لا حذف فيزيائياً للرسائل | Core §4.4 |
| سعوي | ميزانية Realtime محسوبة على 10 مؤسسات — رفع الحصة بند D-04 في core | MVP §1.2 |

## 2.2 — مصفوفة اختيار التقنيات

#### [D-01] أساس التخزين: البناء على جداول Core v2.1 بدل schema مستقل

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **امتداد على `chat_channels`/`chat_posts`/`chat_comments` من core ✅** | مصدر حقيقة واحد · RLS والدوال المساعدة (`teaches`, `is_enrolled`) جاهزة · صفر ازدواجية | مقيدون بقرارات core (BIGINT، جدولان) — وهو قيد مقصود | core هي المرجعية المُلزمة؛ إعادة الاختراع هي بالضبط ما أفشل v1.0 |
| schema مستقل للدردشة ❌ | حرية تصميم | ازدواجية RLS + انحراف حتمي عن بقية المنصة | مرفوض — كرر أخطاء v1.0 |
| خدمة دردشة خارجية (Stream/Sendbird) ❌ | ميزات جاهزة | تعيش خارج RLS والعزل المؤسسي كلياً + تكلفة اشتراك | تكسر المبدأ الحاكم للعزل |

#### [D-02] نموذج الرسائل: جدولان (منشور/تعليق) لا جدول واحد

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **`chat_posts` + `chat_comments` ✅** | فصل الصلاحيات يطابق قرار #16 حرفياً · سياسات RLS أبسط وأقل عرضة للخطأ · حدود طول مختلفة (8000/4000) طبيعية | استعلامان عند فتح منشور بتعليقاته | الأمان أولاً — سياسة INSERT للمنشور تخص المحاضر وحده دون شرط شرطي معقد |
| جدول واحد + `parent_message_id` ❌ | استعلام واحد | سياسة INSERT واحدة يجب أن تفرّق داخلياً بين منشور وتعليق — سطح خطأ أمني | مرفوض — نموذج v1.0 المستبدَل |

#### [D-03] التزامن الحي: Realtime على القناة المفتوحة فقط + polling fallback

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **اشتراك Realtime واحد للقناة المفتوحة على الشاشة ✅** | استهلاك حصة = عدد الشاشات المفتوحة لا عدد المستخدمين · fallback polling 60s (نمط Core §3.2) | القنوات غير المفتوحة تعتمد على الإشعارات للتنبيه | حصة Realtime هي عنق الزجاجة المعلن (0.2 س5) |
| اشتراك بكل قنوات المستخدم ❌ | تحديث كل شيء حياً | طالب بـ 8 مواد = 8 اشتراكات × 20,000 مستخدم = انفجار الحصة | مرفوض حسابياً |
| polling فقط ❌ | أبسط | تأخر 60s في قناة مفتوحة تجربة رديئة | مرفوض |

#### [D-04] القنوات غير الصفية (support / admin_lecturer / admin_admin): منشورات مسطّحة بلا تعليقات

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **منشورات مسطّحة — كل مشارك مخوَّل يكتب `chat_post` ✅** | محادثة طبيعية بين أطراف متكافئة · لا حاجة لتمييز منشور/تعليق حيث لا فرق صلاحيات | لا threading داخل قناة الدعم | نموذج المنشور/التعليق وُجد لفرض قرار #16 على قناة المادة فقط؛ خارجها لا مبرر له |
| منشور + تعليقات في كل الأنواع ❌ | اتساق شكلي | من «المنشور» في محادثة إداري↔محاضر؟ غموض بلا قيمة | مرفوض |

#### [D-05] هوية القنوات غير الصفية: عمود `counterpart_id` + قيود تفرّد جزئية

قناة `support` واحدة لكل طالب، وقناة `admin_lecturer` واحدة لكل محاضر (يشاركها كل إداريي المؤسسة)، وقناة `admin_admin` واحدة لكل مؤسسة — تُفرض جميعها بفهارس فريدة جزئية (القسم 4.2) لا بمنطق تطبيق.

## 2.3 — الديون التقنية المقبولة عمداً

| الدين | لماذا نقبله الآن | متى يُسدَّد |
|---|---|---|
| لا بحث نصي كامل (FTS) في الرسائل | MVP لا يطلبه؛ فهرس GIN يرفع كلفة كل كتابة | عند طلب منتج صريح — إضافة GIN على `body` لاحقاً ترحيل آمن |
| عدّاد غير المقروء يُحسب عند الطلب (COUNT فوق `last_read_post_id`) | دقيق وبسيط؛ القنوات قصيرة نسبياً | عند تجاوز p95 المستهدف: عمود denormalized يُحدَّث بـ trigger |
| fanout الإشعارات متزامن داخل الـ trigger | INSERT..SELECT واحد يكفي حتى 1,000 مسجَّل (A-03) | لو تجاوزت القناة ~2,000 مسجَّل: صف معالجة عبر pg_cron |
| لا أرشفة لقنوات الفصول المنتهية | soft delete + `term` عبر `courses` يكفيان | عند 100k رسالة/قناة: partitioning على `created_at` |

---

# القسم 3: معمارية النظام والمخططات 🏗️

## 3.1 — المخطط العام (System Context)

\`\`\`mermaid
graph TB
    subgraph Clients["أجهزة المستخدمين (10 مؤسسات)"]
        ST[📱 طالب — قراءة + تعليق]
        LC[📱 محاضر — نشر في مواده]
        AD[💻 إداري — تنسيق + دعم]
    end

    subgraph Supabase["Supabase Pro + Supavisor"]
        RPC[⚙️ RPCs الدردشة]
        PG[(🗄️ PostgreSQL<br/>chat_channels / chat_posts / chat_comments<br/>RLS: دور × مؤسسة × اشتراك)]
        RT[📡 Realtime<br/>قناة مفتوحة واحدة لكل جهاز]
        NT[🔔 نظام الإشعارات<br/>trigger fanout]
    end

    ST --> RPC
    LC --> RPC
    AD --> RPC
    RPC --> PG
    PG --> RT
    PG --> NT
    RT --> ST
    RT --> LC
    RT --> AD
\`\`\`

## 3.2 — جدول المكونات والمسؤوليات

| المكون | مسؤوليته الوحيدة | ماذا لو سقط؟ | استراتيجية التعافي |
|---|---|---|---|
| `chat_channels` (core + امتداد) | تعريف القناة ونوعها وطرفها المقابل | لا قنوات جديدة؛ القائمة تعمل | ترحيلات إضافية فقط (Core §4.4) |
| `chat_posts` / `chat_comments` (core + امتداد) | مصدر الحقيقة للرسائل | توقف الكتابة؛ القراءة من كاش Hive | PITR + طابور إعادة إرسال على العميل |
| `chat_read_state` (جديد) | آخر منشور مقروء لكل مستخدم/قناة | عدّادات unread خاطئة مؤقتاً | يُعاد بناؤه عند فتح القناة — بيانات قابلة للاشتقاق |
| Realtime | بث حي للقناة المفتوحة | تأخر التحديث الحي فقط | polling كل 60s + إعادة اتصال بـ backoff |
| trigger الإشعارات | fanout إلى `notifications` عند منشور/تعليق | لا تنبيهات؛ الرسائل سليمة في القناة | الضمانة at-least-once — العميل يزيل التكرار بمعرّف الإشعار (Core §5.3) |

## 3.3 — تدفق البيانات لأهم عملية (Critical Path Sequence)

**العملية 1: محاضر ينشر في قناة مادة (مع الـ fanout)**

\`\`\`mermaid
sequenceDiagram
    participant L as 📱 المحاضر
    participant R as ⚙️ RPC create_chat_post
    participant D as 🗄️ PostgreSQL
    participant N as 🔔 notifications
    participant S as 📱 طلاب المادة

    L->>R: create_chat_post(channel_id, body, client_uuid)
    Note over R: نقطة فشل: إعادة إرسال أوفلاين مكررة<br/>الدفاع: UNIQUE(client_uuid) + ON CONFLICT DO NOTHING
    R->>D: تحقق RLS: teaches(course_id) + نفس المؤسسة + طول ≤ 8000
    D->>D: INSERT chat_posts
    D->>N: trigger: INSERT..SELECT إشعار لكل مسجَّل ACTIVE في المادة
    D-->>R: post_id + created_at
    R-->>L: تأكيد
    D--)S: Realtime: يصل للأجهزة المفتوحة على القناة < 5s
    N--)S: إشعار داخل التطبيق للبقية (at-least-once)
\`\`\`

**العملية 2: طالب محجوب مالياً يحاول فتح قناة المادة**

\`\`\`mermaid
sequenceDiagram
    participant S as 📱 طالب (اشتراك منتهٍ)
    participant D as 🗄️ PostgreSQL (RLS)

    S->>D: SELECT على chat_posts للقناة
    Note over D: سياسة القراءة تتطلب للطالب:<br/>is_enrolled(course_id) AND has_active_subscription()
    D-->>S: 0 صف — الحجب على مستوى المحرك (قرار #15)
    Note over S: قفل الواجهة تحسين UX فقط — ليس طبقة أمان
\`\`\`

## 3.4 — قرارات معمارية جوهرية (مرجع سريع)

| القرار | الخلاصة |
|---|---|
| [D-01] | البناء على جداول core — لا schema مستقل |
| [D-02] | منشور/تعليق جدولان لفرض قرار #16 على مستوى RLS |
| [D-03] | Realtime للقناة المفتوحة فقط + polling fallback |
| [D-04] | القنوات غير الصفية منشورات مسطّحة بلا تعليقات |
| [D-05] | `counterpart_id` + فهارس فريدة جزئية لهوية القنوات غير الصفية |
| [D-06] | fanout الإشعارات بـ trigger واحد INSERT..SELECT — الضمانة at-least-once |

---

# القسم 4: نماذج البيانات وتصميم قاعدة البيانات 🗄️

## 4.0 — العلاقة بـ Core Data Layer v2.1

جداول `chat_channels` و`chat_posts` و`chat_comments` **معرَّفة في Core §4.2** — هذه الوثيقة لا تعيد إنشاءها بل تضيف امتدادات (ALTER) وجدولاً جديداً واحداً (`chat_read_state`). كل الامتدادات تلتزم قواعد Core §4.0: `institution_id` UUID، فهارس تبدأ به، RLS من JWT claim.

## 4.1 — مخطط الكيانات (ERD)

\`\`\`mermaid
erDiagram
    INSTITUTIONS ||--o{ CHAT_CHANNELS : "قنواتها"
    COURSES ||--o| CHAT_CHANNELS : "قناة المادة (course)"
    PROFILES ||--o{ CHAT_CHANNELS : "طرف مقابل (support/admin_lecturer)"
    CHAT_CHANNELS ||--o{ CHAT_POSTS : "منشوراتها"
    CHAT_POSTS ||--o{ CHAT_COMMENTS : "تعليقاتها (قنوات course فقط)"
    PROFILES ||--o{ CHAT_POSTS : "يكتب"
    PROFILES ||--o{ CHAT_COMMENTS : "يعلّق"
    CHAT_CHANNELS ||--o{ CHAT_READ_STATE : "حالة قراءة"
    PROFILES ||--o{ CHAT_READ_STATE : "لكل مستخدم"
\`\`\`

## 4.2 — تعريف الامتدادات والجداول (Schema)

\`\`\`sql
-- ============================================================
-- 1) امتداد chat_channels (الجدول الأصلي في Core §4.2):
--    الطرف المقابل للقنوات غير الصفية + قيود شكل القناة
-- ============================================================
ALTER TABLE chat_channels
    ADD COLUMN counterpart_id UUID REFERENCES profiles(id),
    ADD COLUMN created_at TIMESTAMPTZ NOT NULL DEFAULT now();

-- شكل القناة حسب نوعها — على مستوى المحرك لا التطبيق
ALTER TABLE chat_channels ADD CONSTRAINT chan_kind_shape CHECK (
       (kind = 'course'         AND course_id IS NOT NULL AND counterpart_id IS NULL)
    OR (kind IN ('support','admin_lecturer')
                                 AND course_id IS NULL     AND counterpart_id IS NOT NULL)
    OR (kind = 'admin_admin'    AND course_id IS NULL     AND counterpart_id IS NULL)
);

-- [D-05] قناة دعم واحدة لكل طالب، وقناة تنسيق واحدة لكل محاضر
CREATE UNIQUE INDEX one_channel_per_counterpart
    ON chat_channels (institution_id, kind, counterpart_id)
    WHERE kind IN ('support','admin_lecturer');

-- قناة إداريين واحدة لكل مؤسسة
CREATE UNIQUE INDEX one_admin_admin_channel
    ON chat_channels (institution_id)
    WHERE kind = 'admin_admin';

CREATE INDEX idx_chat_channels_tenant ON chat_channels (institution_id, kind);

-- قناة المادة تُنشأ تلقائياً مع المادة (A-01)
CREATE FUNCTION create_course_channel() RETURNS trigger
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
BEGIN
    INSERT INTO chat_channels (institution_id, course_id, kind)
    VALUES (NEW.institution_id, NEW.id, 'course');
    RETURN NEW;
END $$;
CREATE TRIGGER trg_course_channel AFTER INSERT ON courses
    FOR EACH ROW EXECUTE FUNCTION create_course_channel();

-- ============================================================
-- 2) امتداد chat_posts / chat_comments:
--    institution_id (سد فجوة قاعدة Core §4.0) + idempotency
-- ============================================================
ALTER TABLE chat_posts
    ADD COLUMN institution_id UUID NOT NULL REFERENCES institutions(id),
    ADD COLUMN client_uuid UUID UNIQUE;   -- مفتاح idempotency للمزامنة الأوفلاين

ALTER TABLE chat_comments
    ADD COLUMN institution_id UUID NOT NULL REFERENCES institutions(id),
    ADD COLUMN client_uuid UUID UNIQUE;

-- أثقل استعلام: آخر منشورات القناة — فهرس يبدأ بالمؤسسة (قاعدة Core §4.0)
CREATE INDEX idx_chat_posts_channel
    ON chat_posts (institution_id, channel_id, created_at DESC)
    WHERE deleted_at IS NULL;

CREATE INDEX idx_chat_comments_post
    ON chat_comments (institution_id, post_id, created_at)
    WHERE deleted_at IS NULL;

-- ============================================================
-- 3) chat_read_state (جديد): آخر منشور مقروء لكل مستخدم/قناة
--    unread يُحسب عند الطلب: COUNT(*) فوق last_read_post_id (§2.3)
-- ============================================================
CREATE TABLE chat_read_state (
    institution_id    UUID   NOT NULL REFERENCES institutions(id),
    channel_id        BIGINT NOT NULL REFERENCES chat_channels(id),
    user_id           UUID   NOT NULL REFERENCES profiles(id),
    last_read_post_id BIGINT REFERENCES chat_posts(id),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (channel_id, user_id)
);
CREATE INDEX idx_read_state_tenant ON chat_read_state (institution_id, user_id);
\`\`\`

**سياسات RLS (تفصّل وتستبدل السياسات النموذجية في Core §4.2 لهذا النظام):**

\`\`\`sql
ALTER TABLE chat_channels   ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_posts      ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_comments   ENABLE ROW LEVEL SECURITY;
ALTER TABLE chat_read_state ENABLE ROW LEVEL SECURITY;

-- دالة مساعدة: هل يملك المستخدم عضوية هذه القناة؟
-- (الحجب المالي — قرار #15 — يُطبق على الطالب هنا بلا استثناء للدردشة)
CREATE FUNCTION can_access_channel(cid BIGINT) RETURNS BOOLEAN
LANGUAGE sql STABLE SECURITY DEFINER SET search_path = public AS $$
    SELECT EXISTS (
        SELECT 1 FROM chat_channels c
        WHERE c.id = cid
          AND c.institution_id = my_institution()
          AND (
               (c.kind = 'course' AND (
                    teaches(c.course_id)
                    OR my_role() = 'admin'
                    OR (is_enrolled(c.course_id) AND has_active_subscription())
               ))
            OR (c.kind = 'support'        AND (c.counterpart_id = auth.uid()
                                               AND has_active_subscription()
                                               OR my_role() = 'admin'))
            OR (c.kind = 'admin_lecturer' AND (c.counterpart_id = auth.uid()
                                               OR my_role() = 'admin'))
            OR (c.kind = 'admin_admin'    AND my_role() = 'admin')
          )
    )
$$;

-- القنوات: القراءة لمن يملك عضويتها؛ لا INSERT مباشر لأحد (RPC/trigger فقط)
CREATE POLICY chan_read ON chat_channels FOR SELECT
    USING (can_access_channel(id));

-- المنشورات: القراءة بالعضوية
CREATE POLICY posts_read ON chat_posts FOR SELECT
    USING (institution_id = my_institution()
           AND deleted_at IS NULL
           AND can_access_channel(channel_id));

-- [قرار #8 + #16] إنشاء المنشور:
--   قناة course: المحاضر المعيَّن حصراً
--   القنوات الأخرى [D-04]: أي عضو مخوَّل (محادثة مسطّحة)
CREATE POLICY posts_insert ON chat_posts FOR INSERT
    WITH CHECK (
        author_id = auth.uid()
        AND institution_id = my_institution()
        AND EXISTS (
            SELECT 1 FROM chat_channels c
            WHERE c.id = channel_id
              AND c.institution_id = my_institution()
              AND (
                   (c.kind = 'course' AND teaches(c.course_id))
                OR (c.kind <> 'course' AND can_access_channel(c.id))
              )
        )
    );

-- حذف soft للمنشور: كاتبه أو الإداري — عبر UPDATE على deleted_at حصراً
CREATE POLICY posts_soft_delete ON chat_posts FOR UPDATE
    USING (institution_id = my_institution()
           AND (author_id = auth.uid() OR my_role() = 'admin'));
-- لا سياسة DELETE لأحد — الحذف الفيزيائي ممنوع (Core §4.4)

-- التعليقات: قراءة بعضوية قناة المنشور
CREATE POLICY comments_read ON chat_comments FOR SELECT
    USING (institution_id = my_institution()
           AND deleted_at IS NULL
           AND EXISTS (SELECT 1 FROM chat_posts p
                       WHERE p.id = post_id
                         AND can_access_channel(p.channel_id)));

-- [قرار #16] التعليق: الطالب المسجَّل (غير المحجوب)، محاضر المادة، أو الإداري —
-- على منشورات قنوات course فقط ([D-04])، والمنشور غير محذوف
CREATE POLICY comments_insert ON chat_comments FOR INSERT
    WITH CHECK (
        author_id = auth.uid()
        AND institution_id = my_institution()
        AND EXISTS (
            SELECT 1 FROM chat_posts p
            JOIN chat_channels c ON c.id = p.channel_id
            WHERE p.id = post_id
              AND p.deleted_at IS NULL
              AND c.kind = 'course'
              AND can_access_channel(c.id)
        )
    );

CREATE POLICY comments_soft_delete ON chat_comments FOR UPDATE
    USING (institution_id = my_institution()
           AND (author_id = auth.uid() OR my_role() = 'admin'
                OR EXISTS (SELECT 1 FROM chat_posts p
                           JOIN chat_channels c ON c.id = p.channel_id
                           WHERE p.id = post_id AND c.kind = 'course'
                             AND teaches(c.course_id))));

-- حالة القراءة: كل مستخدم يدير صفوفه فقط
CREATE POLICY read_state_own ON chat_read_state FOR ALL
    USING (user_id = auth.uid() AND institution_id = my_institution())
    WITH CHECK (user_id = auth.uid() AND institution_id = my_institution());
\`\`\`

**trigger الإشعارات ([D-06] — التكامل مع نظام الإشعارات):**

\`\`\`sql
-- منشور جديد في قناة course → إشعار لكل مسجَّل ACTIVE (عدا الكاتب)
-- منشور في support/admin_lecturer/admin_admin → إشعار للطرف الآخر/الإداريين
-- تعليق جديد → إشعار لكاتب المنشور (المحاضر) — MVP §4.2
-- الضمانة at-least-once؛ العميل يزيل التكرار بمعرّف الإشعار (Core §5.3)
CREATE FUNCTION notify_chat_post() RETURNS trigger
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
DECLARE ch chat_channels%ROWTYPE;
BEGIN
    SELECT * INTO ch FROM chat_channels WHERE id = NEW.channel_id;
    IF ch.kind = 'course' THEN
        INSERT INTO notifications (user_id, kind, payload)
        SELECT e.student_id, 'chat_post',
               jsonb_build_object('channel_id', NEW.channel_id, 'post_id', NEW.id)
        FROM enrollments e
        WHERE e.course_id = ch.course_id AND e.status = 'ACTIVE'
          AND e.student_id <> NEW.author_id;
    ELSE
        -- القنوات الإدارية والدعم: إشعار كل أعضاء القناة عدا الكاتب
        INSERT INTO notifications (user_id, kind, payload)
        SELECT p.id, 'chat_post',
               jsonb_build_object('channel_id', NEW.channel_id, 'post_id', NEW.id)
        FROM profiles p
        WHERE p.institution_id = ch.institution_id
          AND p.deleted_at IS NULL
          AND p.id <> NEW.author_id
          AND (p.id = ch.counterpart_id OR p.role = 'admin');
    END IF;
    RETURN NEW;
END $$;
CREATE TRIGGER trg_notify_post AFTER INSERT ON chat_posts
    FOR EACH ROW EXECUTE FUNCTION notify_chat_post();

CREATE FUNCTION notify_chat_comment() RETURNS trigger
LANGUAGE plpgsql SECURITY DEFINER SET search_path = public AS $$
BEGIN
    INSERT INTO notifications (user_id, kind, payload)
    SELECT p.author_id, 'chat_comment',
           jsonb_build_object('post_id', NEW.post_id, 'comment_id', NEW.id)
    FROM chat_posts p
    WHERE p.id = NEW.post_id AND p.author_id <> NEW.author_id;
    RETURN NEW;
END $$;
CREATE TRIGGER trg_notify_comment AFTER INSERT ON chat_comments
    FOR EACH ROW EXECUTE FUNCTION notify_chat_comment();
\`\`\`

## 4.3 — استراتيجية الفهرسة

| الفهرس | الاستعلام الذي يخدمه | تكلفته على الكتابة |
|---|---|---|
| `chat_posts (institution_id, channel_id, created_at DESC) WHERE deleted_at IS NULL` | فتح القناة — أثقل استعلام في النظام (0.2 س2) | متوسطة: على مسار كل نشر — ضروري |
| `chat_comments (institution_id, post_id, created_at) WHERE deleted_at IS NULL` | فتح منشور بتعليقاته | متوسطة |
| `chat_channels (institution_id, kind)` | قائمة قنوات المستخدم | شبه معدومة: القنوات نادرة الإنشاء |
| فريد جزئي `(institution_id, kind, counterpart_id)` | منع تكرار قناة دعم/تنسيق [D-05] | شبه معدومة |
| `chat_read_state (institution_id, user_id)` | عدّادات unread لكل قنوات المستخدم | منخفضة: upsert واحد عند فتح قناة |
| `UNIQUE (client_uuid)` على المنشورات والتعليقات | idempotency المزامنة الأوفلاين | منخفضة |

## 4.4 — أسئلة يجب الإجابة عنها صراحة

- **الترحيلات:** ملفات SQL مرقّمة عبر CI — إضافات فقط أثناء التشغيل (Core §4.4)؛ عمود `institution_id` على الرسائل يُضاف nullable ثم backfill من القناة ثم `NOT NULL`.
- **الحذف:** soft delete حصراً (`deleted_at`) — المنشور المحذوف تختفي تعليقاته من العرض (فلترة على منشوره) لكنها لا تُحذف فيزيائياً؛ لا سياسة DELETE على أي جدول دردشة.
- **التوسع:** عند تجاوز `chat_posts` عشرة ملايين صف: partitioning على الشهر — مفتاح sharding المستقبلي جاهز: `institution_id` (نمط Core §4.4).
- **التوافقية:** **Strict** في: إنشاء المنشور/التعليق (idempotent)، تفرّد القنوات. **Eventual مقبولة** في: عدّادات unread، وصول الإشعارات، البث الحي.

---

# القسم 5: عقود الـ API والواجهات 🔌

## 5.1 — مبادئ التصميم

نفس نمط Core [D-07]: **PostgREST محكوم بـ RLS للقراءات** (قائمة القنوات، منشورات القناة، التعليقات)، و**RPC لكل كتابة** لضمان idempotency والتحقق المركّب. لا يمرر العميل `institution_id` أو `role` أبداً — كل شيء من JWT. صيغة الخطأ الموحدة: `{"error": {"code": "...", "message_ar": "..."}}`.

## 5.2 — توثيق الـ RPCs

#### `POST /rest/v1/rpc/create_chat_post`
**الغرض:** نشر منشور (قناة مادة: محاضر حصراً · بقية القنوات: الأعضاء) | **المصادقة:** JWT | **Rate Limit:** 30/دقيقة/مستخدم

**المدخلات:**
\`\`\`json
{
  "channel_id": "bigint — مطلوب",
  "body": "string — مطلوب، 1-8000 حرف",
  "client_uuid": "uuid — مطلوب، يولَّد على الجهاز لكل رسالة"
}
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{ "post_id": 123, "created_at": "ISO-8601" }
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | body فارغ أو > 8000 حرف | رسالة تحقق قبل الإرسال أصلاً |
| 401 | توكن منتهٍ | تجديد الجلسة وإعادة الإرسال من الطابور |
| 403 | طالب يحاول النشر في قناة مادة / قناة مؤسسة أخرى / طالب محجوب مالياً | إخفاء زر النشر (RLS هي الدفاع الفعلي) |
| 404 | القناة غير موجودة أو محذوفة | تحديث قائمة القنوات |
| 409 | `client_uuid` مستخدم سابقاً | اعتباره نجاحاً — حذف من الطابور (idempotent) |
| 429 | تجاوز الحد | retry مع exponential backoff |
| 500 | خطأ داخلي | إبقاء في الطابور وإعادة المحاولة — idempotent |

#### `POST /rest/v1/rpc/add_chat_comment`
**الغرض:** تعليق على منشور في قناة مادة (قرار #16) | **المصادقة:** JWT | **Rate Limit:** 60/دقيقة/مستخدم

**المدخلات:**
\`\`\`json
{
  "post_id": "bigint — مطلوب",
  "body": "string — مطلوب، 1-4000 حرف",
  "client_uuid": "uuid — مطلوب"
}
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{ "comment_id": 456, "created_at": "ISO-8601" }
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | body فارغ أو > 4000 حرف | رسالة تحقق |
| 401 | توكن منتهٍ | تجديد وإعادة إرسال |
| 403 | غير مسجَّل ACTIVE بالمادة / محجوب مالياً / تعليق على قناة غير صفية [D-04] | إخفاء حقل التعليق |
| 404 | المنشور محذوف (soft) أو غير موجود | «هذا المنشور لم يعد متاحاً» + تحديث القناة |
| 409 | `client_uuid` مكرر | نجاح idempotent |
| 429 | تجاوز الحد | backoff |
| 500 | خطأ داخلي | إعادة محاولة آمنة |

#### `POST /rest/v1/rpc/mark_channel_read`
**الغرض:** تحديث آخر منشور مقروء (upsert) | **المصادقة:** JWT | **Rate Limit:** 120/دقيقة/مستخدم

**المدخلات:**
\`\`\`json
{ "channel_id": "bigint — مطلوب", "last_read_post_id": "bigint — مطلوب" }
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{ "updated": true }
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | `last_read_post_id` لا ينتمي للقناة | تجاهل — إعادة الحساب عند الفتح التالي |
| 403 | لا عضوية في القناة | تحديث قائمة القنوات |
| 409 | قيمة أقدم من المخزنة (سباق أجهزة متعددة) | يُحتفظ بالأحدث — GREATEST داخل الدالة، يُعاد 200 |
| 500 | خطأ داخلي | غير حرج — إعادة محاولة كسولة |

#### `POST /rest/v1/rpc/open_support_channel`
**الغرض:** فتح (أو إرجاع) قناة الدعم الخاصة بالطالب — get-or-create idempotent | **المصادقة:** JWT طالب أو محاضر | **Rate Limit:** 10/دقيقة/مستخدم

**المدخلات:**
\`\`\`json
{}
\`\`\`

**المخرجات الناجحة (200):**
\`\`\`json
{ "channel_id": 789, "created": false }
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 401 | توكن منتهٍ | إعادة تسجيل دخول |
| 403 | طالب محجوب مالياً (تبعة قرار #15 — الافتراض A-05) | رسالة «يرجى تسوية الاشتراك» |
| 409 | سباق إنشاء من جهازين | الفهرس الفريد الجزئي يضمن قناة واحدة — تُعاد القناة الموجودة بـ 200 |
| 500 | خطأ داخلي | إعادة محاولة — idempotent |

## 5.3 — العقود بين الأنظمة الداخلية

- **نظام الإشعارات:** triggers القسم 4.2 تكتب في `notifications` — الضمانة at-least-once، وإزالة التكرار على العميل بمعرّف الإشعار (نمط Core §5.3). الإداري يستقبل إشعارات كل الدردشات الموجهة إليه (MVP §5.2).
- **Realtime:** بث `INSERT` على `chat_posts`/`chat_comments` مُفلتراً بالقناة المفتوحة؛ الحصة محسوبة على 10 مؤسسات وترفَع قبل الإطلاق (Core D-04).
- **نظام التسجيل (Enrollment):** عضوية قناة المادة تُشتق حياً من `enrollments` بحالة ACTIVE — لا جدول عضوية مستقل، فتحويل التسجيل إلى COMPLETED أو DROPPED يسحب الوصول تلقائياً بلا مزامنة.

---

# القسم 6: حالات الحافة، أنماط الفشل، والأمان 🛡️

## 6.1 — جرد حالات الحافة

| السيناريو | ماذا يحدث في التصميم؟ | المعالجة |
|---|---|---|
| طالب يحاول إنشاء منشور في قناة مادة | سياسة `posts_insert` ترفضه على مستوى المحرك | 403 — قرار #16 مفروض بـ RLS لا بالواجهة |
| إعادة إرسال أوفلاين مكررة | `UNIQUE(client_uuid)` | 409 يُعامل نجاحاً — صف واحد مضمون |
| تعليق يصل بعد حذف المنشور (مزامنة أوفلاين) | فحص `p.deleted_at IS NULL` في سياسة الإدراج | 404 — العميل يعرض «المنشور لم يعد متاحاً» ويحذف من الطابور |
| طالب انتهى اشتراكه وقناته مفتوحة على الشاشة | القراءة التالية تعود 0 صف؛ Realtime لا يمرر ما لا تسمح به RLS | الحجب فوري server-side (قرار #15) |
| محاضر أُزيل من المادة منتصف الفصل | `teaches()` تُقيَّم حياً عند كل عملية | يفقد النشر فوراً؛ منشوراته السابقة تبقى (سجل رسمي) |
| اعتماد نتيجة المادة (enrollments → COMPLETED) | `is_enrolled` يشترط ACTIVE | الطالب يفقد الوصول للقناة تلقائياً — بلا كود إضافي |
| مستخدم يحاول قراءة قناة مؤسسة أخرى | `institution_id = my_institution()` في كل مسار | 0 صف — طبقتا دفاع (القناة + الرسالة) |
| إنشاء قناة دعم من جهازين معاً | فهرس فريد جزئي [D-05] | الثاني يستلم القناة الموجودة (get-or-create) |
| منشور بحجم 10x المتوقع | CHECK على الطول (8000/4000) في core | مرفوض على مستوى المحرك — لا اعتماد على العميل |
| ساعات ذروة: 10 محاضرين ينشرون لقنوات 1,000 طالب معاً | 10 × INSERT..SELECT ≈ 10,000 صف إشعار | ضمن قدرة القاعدة؛ يُراقب زمن الـ trigger (§6.5) وينقل لـ async عند تجاوز A-03 |
| سباق `mark_channel_read` من جهازين | GREATEST داخل الدالة | يُحتفظ بالأحدث دائماً |

## 6.2 — تحليل أنماط الفشل

| المكون | نمط الفشل | الاحتمالية | الأثر | الكشف | التعافي |
|---|---|---|---|---|---|
| Realtime | انقطاع البث / تجاوز الحصة | متوسطة | 🟢 منخفض — تأخر التحديث الحي | heartbeat من العميل | polling كل 60s + إعادة اتصال بـ backoff (Core §3.2) |
| trigger الإشعارات | بطء fanout لقناة ضخمة | متوسطة | 🟡 متوسط — يبطئ نشر المنشور نفسه | مقياس زمن الـ trigger > 3s | نقل الـ fanout إلى صف pg_cron (دين §2.3 يُسدد مبكراً) |
| PostgreSQL | تعطل كامل | منخفضة | 🟠 عالٍ — لا كتابة | health check (Core §6.2) | القراءة من Hive + طابور إعادة إرسال؛ failover مُدار |
| العميل | فقدان طابور Hive | منخفضة | 🟢 منخفض | — | الرسائل غير المرسلة تُفقد محلياً فقط — تحذير UX قبل مسح الكاش |

## 6.3 — نموذج التهديدات (STRIDE)

| التهديد | مثال ملموس | الدفاع المحدد |
|---|---|---|
| Spoofing | انتحال `author_id` مستخدم آخر | `author_id = auth.uid()` في WITH CHECK — لا يقبل قيمة العميل |
| Spoofing | JWT بمؤسسة مزوّرة | الـ claim يُوقَّع server-side (Core STRIDE) |
| Tampering | تعديل نص منشور بعد نشره | لا سياسة UPDATE على `body` — التحديث المسموح لـ `deleted_at` حصراً (A-04) |
| Repudiation | محاضر ينكر منشوراً رسمياً | لا حذف فيزيائي + `author_id` + `created_at` ثابتان |
| Information Disclosure | طالب يقرأ قناة تنسيق إداري↔محاضر | `can_access_channel` تحصر `admin_lecturer` بالطرفين |
| Information Disclosure | تسريب رسائل بين مؤسستين | `institution_id` على القناة **والرسالة** + فهارس مركّبة + pgTAP لكل مؤسسة |
| Denial of Service | إغراق قناة بالتعليقات | rate limits لكل RPC + حدود الطول على المحرك |
| Elevation of Privilege | طالب يستدعي RPC النشر مباشرة متجاوزاً الواجهة | RLS هي الحكم — الواجهة ليست طبقة أمان (المبدأ الحاكم) |

## 6.4 — قائمة تدقيق أمنية إلزامية

- [x] **المصادقة والتفويض:** JWT (`auth.uid()` + `institution_id` claim) → RLS لكل جدول؛ التحقق في قاعدة البيانات حصراً
- [x] **تشفير البيانات:** TLS 1.2+ in-transit · AES-256 at-rest (موروث من Core §6.4)
- [x] **إدارة الأسرار:** العميل يحمل `anon key` المحكوم بـ RLS فقط — لا أسرار في كود الدردشة
- [x] **حقن SQL:** PostgREST مُعلمَن + RPCs بلا `EXECUTE` نصي مركّب
- [x] **سجلات التدقيق:** حذف الرسائل (من حذف ومتى عبر `deleted_at` + من نفّذ في سجل RPC)؛ **لا يُسجَّل** محتوى رسائل الدعم في اللوغات التشغيلية (PII)

## 6.5 — الملاحظة والمراقبة (Observability)

| المقياس | عتبة التنبيه |
|---|---|
| p95 لاستعلام فتح القناة | > 300ms لمدة 5 دقائق متصلة |
| زمن trigger الـ fanout لمنشور واحد | > 3s |
| نسبة اتصالات Realtime من الحصة (عبر المؤسسات) | > 80% |
| معدل 403 على RPCs الدردشة | ارتفاع مفاجئ > 5x المعدل الطبيعي (محاولة اختراق أو خلل RLS) |
| معدل فشل `create_chat_post` | > 2% خلال ساعة |

---

# القسم 7: خطة التنفيذ وخارطة الطريق 🗺️

> تنفَّذ هذه الوثيقة ضمن **المرحلة 7 من خطة Core Data Layer v2.1** (أسبوعان) — الاعتمادية: المرحلة 1 (الهوية والتسجيل)، والمرحلتان 3 و5 للإشعارات الموجّهة والحجب المالي.

## 7.1 — ترتيب المخاطر (Risk-First Ordering)

**أخطر افتراض:** أن fanout الإشعارات المتزامن داخل trigger يبقى دون 3 ثوانٍ لقناة بـ 1,000 مسجَّل مع 10 منشورات متزامنة عبر المؤسسات. لو فشل، يُنقل الـ fanout إلى صف async — تغيير محصور لا يمس الـ schema. لذلك هو موضوع الـ spike الأول.

## 7.2 — المراحل

#### الخطوة 0: Spike الـ fanout — يومان
- **الهدف:** التحقق من الافتراض أعلاه
- **المخرج القابل للاختبار:** trigger على بيانات اصطناعية (10 مؤسسات × قناة 1,000 مسجَّل) + قياس k6
- **معيار النجاح/الفشل:** < 3s للـ INSERT الجماعي → متابعة · فشل → صف pg_cron من اليوم الأول

#### الخطوة 1: الامتدادات والسياسات — 3 أيام
- **المهام:** (1) ترحيلات ALTER (counterpart_id، institution_id على الرسائل، client_uuid) (2) `chat_read_state` (3) trigger إنشاء قناة المادة (4) كل سياسات RLS (5) pgTAP: دور × عملية × مؤسسة × حالة اشتراك
- **المخرج:** طالب محجوب يرى 0 صف؛ طالب من مؤسسة أخرى يرى 0 صف؛ طالب يفشل في إنشاء منشور — كلها اختبارات آلية خضراء

#### الخطوة 2: RPCs الكتابة والتكامل — 4 أيام
- **المهام:** (1) الـ RPCs الأربعة بكامل أكواد الخطأ (2) triggers الإشعارات (3) طابور إعادة الإرسال على العميل بـ client_uuid
- **المخرج:** دورة كاملة: محاضر ينشر → طالب يستلم إشعاراً ويعلّق → المحاضر يستلم إشعار التعليق
- **اختبارات:** idempotency بإعادة إرسال متعمدة · تعليق على منشور محذوف → 404

#### الخطوة 3: Realtime وحالة القراءة — 3 أيام
- **المهام:** (1) اشتراك القناة المفتوحة + fallback polling (2) `mark_channel_read` وعدّادات unread (3) شاشات القنوات الأربعة
- **المخرج:** رسالة تصل لجهاز مفتوح < 5s؛ انقطاع Realtime يسقط لـ polling تلقائياً

## 7.3 — مخطط الاعتماديات

\`\`\`mermaid
graph LR
    C1[Core مرحلة 1: الهوية والتسجيل] --> K0[خطوة 0: Spike fanout]
    C5[Core مرحلة 5: الاشتراك المالي] --> K1
    K0 --> K1[خطوة 1: امتدادات + RLS]
    K1 --> K2[خطوة 2: RPCs + إشعارات]
    K2 --> K3[خطوة 3: Realtime + قراءة]
\`\`\`

## 7.4 — تعريف "الانتهاء" (Definition of Done)

- [ ] كل الاختبارات تمر (تغطية ≥ 90% للمسارات الحرجة: النشر، التعليق، الحجب، العزل المؤسسي)
- [ ] pgTAP لكل سياسة × دور × عملية × مؤسسة × حالة اشتراك — صفر تسريب
- [ ] توثيق الـ RPCs الأربعة محدّث بكامل أكواد الخطأ
- [ ] مقاييس §6.5 تعمل بعتباتها
- [ ] خطة rollback لكل ترحيل مجرّبة على staging
- [ ] اختبار حمل: منشور لقناة 1,000 طالب × 10 مؤسسات متزامنة — fanout < 3s
- [ ] رفع تغذية راجعة رسمية لـ Core v2.2: إضافة `institution_id` على `chat_posts`/`chat_comments` في المصدر

---

## مصفوفة التوافق مع MVP v3.6 و Core Data Layer v2.1 (تحقق نهائي)

| القرار / القاعدة | أين طُبّق في هذه الوثيقة |
|---|---|
| MVP §3.4 قراءة + تعليق، لا 1:1 | [D-02] + سياسات `posts_insert`/`comments_insert` + نطاق العمل 0.4 |
| MVP قرار #8: نشر المحاضر عبر قناة المادة حصراً | سياسة `posts_insert` (kind='course' → teaches) |
| MVP قرار #15: الحجب المالي server-side | `has_active_subscription()` داخل `can_access_channel` + حالة حافة 6.1 |
| MVP قرار #16: الطالب يعلّق فقط | فصل الجدولين [D-02] + سياسات التعليق |
| MVP §4.4 دردشة المحاضر مع الإدارة | قنوات `admin_lecturer` + [D-04] [D-05] |
| MVP §5.3 دردشات الإداري + مركز الدعم | قنوات `support` و`admin_admin` + `open_support_channel` |
| MVP §5.2 الإداري يستقبل إشعارات الدردشات | trigger `notify_chat_post` (فرع القنوات غير الصفية) |
| MVP §1.2 / قرار #20: تعدد المؤسسات | `institution_id` على كل جدول + فهارس مركّبة + pgTAP بالمؤسسة |
| Core §4.0 قواعد multi-tenancy | القسم 4.0 + 4.2 + سد فجوة الرسائل |
| Core [D-08] الهوية من JWT claim | `my_institution()` في كل سياسة |
| Core §4.4 soft delete — لا حذف فيزيائياً | لا سياسة DELETE + `posts_soft_delete` |
| Core §5.3 إشعارات at-least-once | [D-06] + العقود 5.3 |
| Core §3.2 fallback إلى polling 60s | [D-03] + أنماط الفشل 6.2 |
| v1.6 §2.5 دورة حياة Enrollment | العضوية مشتقة حياً من ACTIVE — حالة حافة COMPLETED في 6.1 |

*— نهاية الوثيقة — Chat System TDD v2.0 —*