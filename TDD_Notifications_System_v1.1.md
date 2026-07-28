# 🔔 وثيقة التصميم التقني — نظام الإشعارات والإعلانات
## Academic Hub — Notifications & Announcements System TDD

---

## بيانات ضبط الوثيقة

| البند | التفاصيل |
|---|---|
| **اسم الوثيقة** | وثيقة التصميم التقني — نظام الإشعارات والإعلانات |
| **رقم الإصدار** | v1.1 — تطبيق بنود NT-01 → NT-10 من قائمة تدقيق التوافق v1.0 |
| **مستوى العمق** | 🟡 Production |
| **الوثائق المرجعية** | MVP Functions **v3.6** (الأقسام 3.2 · 4.2 · 5.2 · القرارات #15 #20 #21) · **Core Data Layer TDD v2.1 (المرجع الملزم للـ Schema والأمان)** · Compatibility Audit v1.0 · Attendance TDD · Scheduling & Events TDD v1.1 |
| **الحالة** | معتمدة بعد حسم التعارضات |
| **مالك الوثيقة** | رامز |

### سجل التغييرات v1.0 → v1.1

| البند | التغيير |
|---|---|
| NT-01 🔴 | إلغاء المواضيع العالمية (`all_users`…) — كل مواضيع FCM مؤسسية: `inst_{institution_id}_*` + حارس برمجي في dispatch |
| NT-02 🔴 | `announcements` أساسه تعريف Core Data v2.1 §4.2-13 (يحمل `institution_id`) وأعمدة هذه الوثيقة امتداد عليه |
| NT-03 🟠 | مخطط `notifications` المفصّل في هذه الوثيقة هو المعتمد ويُرحَّل إلى Core Data v2.2؛ لا `is_pinned` على الإشعارات الفردية |
| NT-04 🟠 | `profiles(id)` بدل `users(id)` · `course_id BIGINT` · `ref_id TEXT` |
| NT-05 🔴 | نموذج الحمل: 20,000 مستخدم عبر 10 مؤسسات (قرار #21)؛ إلغاء عبارة «500 اتصال Realtime تكفي» |
| NT-06 🔴 | فئة `late_payers` تُبنى حصراً فوق `has_active_subscription()` (MVP §3.7.2) |
| NT-07 🟠 | تذكير الـ 30 دقيقة يفلتر `enrollments.status = 'ACTIVE'` إلزامياً |
| NT-08 🟡 | اعتماد ENUM `announce_level` من Core Data بدل TEXT CHECK |
| NT-09 🟡 | تحديث المرجعية إلى MVP v3.6 + Core Data v2.1 |
| NT-10 ✅ | تثبيت استثناء الطالب المحجوب اشتراكياً كقرار صريح موثَّق |

---

# القسم 0: بروتوكول التحليل المتسلسل 🧠

## 0.1 — إعادة صياغة المشكلة

المطلوب تصميم نظام إشعارات وإعلانات موحّد يخدم منصة **متعددة المؤسسات** (10 مؤسسات — قرار MVP #20/#21) بثلاثة أدوار بصلاحيات متمايزة داخل كل مؤسسة: **الطالب** مستقبِل قراءة فقط (إعلانات البورد الأسود، تغييرات الجدول، تحديثات الملفات، دردشات مواده، تنبيهات الدفعات، وتذكير استباقي قبل 30 دقيقة من الأحداث الحضورية)، **المحاضر** مستقبِل للإعلانات المؤسسية والتنبيهات النظامية، و**الإداري** هو الناشر الوحيد داخل مؤسسته إضافة إلى استقباله إشعارات الدردشات.

**القيد الحاكم الجديد (من التدقيق):** العزل المؤسسي مطلق — إعلان مؤسسة A لا يصل push ولا Realtime ولا فيداً لأي مستخدم في مؤسسة B. هذا حرفياً «أخطر فشل ممكن» حسب MVP §1.1.

نقطة الغموض الوحيدة المكتشفة: **من يملك منطق «التذكير قبل 30 دقيقة»؟** — الحدث نفسه ملك نظام الجدولة، لكن التسليم ملك هذا النظام. القرار: نظام الجدولة **مصدر حدث (Event Producer)** وهذا النظام **قناة تسليم حصرية** — لا يولّد أي نظام آخر إشعاراته بنفسه (موثّق في [D-06]).

## 0.2 — الأسئلة الخمسة الحاسمة

1. **من المستخدم الفعلي؟** النموذج الرسمي (قرار #21): **~2,000 طالب لكل مؤسسة × 10 مؤسسات = 20,000 طالب** + ~80 محاضراً/مؤسسة (~800) + ~30 إدارياً/مؤسسة (~300). ذروة الاستخدام: نشر جدول امتحانات أو إعلان عاجل → **fan-out إلى ~2,100 جهاز داخل المؤسسة الواحدة خلال ثوانٍ** — وقد تتزامن موجات عبر عدة مؤسسات (بداية الفصل الدراسي مشتركة)، فالميزانية تُحسب على السيناريو الكلي لا المؤسسة الواحدة.
2. **قراءة أم كتابة أثقل؟** قراءة بشدة — نسبة تقريبية **100:1**. الكتابة الوحيدة الثقيلة هي fan-out الإشعارات الفردية، وقد حُيّدت بنموذج هجين ([D-03]) يجعل الإعلان العام صفاً واحداً مهما كان عدد المستقبلين.
3. **ماذا لو توقف النظام ساعة؟** لا خسارة بيانات (المصادر تكتب في Postgres والإشعارات تُبنى منها) — الأثر: تأخر تسليم فقط. الاستثناء الحرج: تذكيرات ما قبل الحدث وإشعارات تغيير قاعة الامتحان. availability المستهدف للتسليم: **99.5% شهرياً**، مع مسار «تغيير حدث خلال < ساعتين من موعده» كمسار حرج له مراقبة مستقلة (القسم 6.5).
4. **البيانات الأكثر حساسية؟** أمران: (أ) الإشعارات الموجّهة للفئات (**قائمة المتأخرين مالياً ومتكرري الغياب**) — تسريب انتماء طالب لإحداها = كشف معلومات مالية/سلوكية. (ب) **العزل المؤسسي نفسه** — تسريب إعلان أو بيانات مؤسسة لأخرى يفشل المنصة تجارياً. الدفاع: الفئة تُحسب لحظة الإرسال ولا تُخزَّن كوسم، وكل سياسة RLS تبدأ بـ `institution_id = my_institution()`.
5. **أضيق عنق زجاجة خلال 6 أشهر؟** جدول `notifications` الفردي: بمعدل ~20 إشعاراً فردياً/طالب/أسبوع × 20,000 طالب → **~26 مليون صف/فصل دراسي** على المستوى الكلي. المعالجة: فهرسة على `(user_id, created_at)` + سياسة أرشفة بعد 180 يوماً (القسم 4.4) + **تفعيل range partitioning على `created_at` مُجدول من السنة الأولى** (لا مؤجّلاً كما في v1.0 — الأرقام الكلية تتجاوز عتبة الـ 20 مليون خلال فصلين).

## 0.3 — الافتراضات المعلنة

| # | الافتراض | مستوى الثقة | تأثيره لو كان خاطئاً |
|---|---|---|---|
| A-01 | حجم المنصة: 10 مؤسسات × (~2,000 طالب + ~80 محاضراً + ~30 إدارياً) = ~21,100 مستخدم (قرار MVP #21) | عالٍ — رقم رسمي | لو زاد عدد المؤسسات: البنية المؤسسية للمواضيع والفهارس تستوعب التوسع خطياً |
| A-02 | مستوى العمق 🟡 Production | عالٍ | لو Prototype: يمكن إسقاط الجدولة المؤجلة والمقاييس من النطاق الأول |
| A-03 | المنصة: Flutter + Supabase (Postgres/RLS/Realtime) — متسقة مع Core Data v2.1 | عالٍ | لو stack مختلف: تُستبدل الأقسام 2 و3 مع بقاء نموذج البيانات صالحاً |
| A-04 | تطبيق موبايل حصراً (Android أولوية) — لا نسخة ويب في الـ MVP | متوسط | لو وُجد ويب: تُضاف قناة Web Push وتتعقد إدارة التوكنات |
| A-05 | اتصال الإنترنت لدى الطلاب متقطع (سياق محلي) | عالٍ | يعزز قرار الاعتماد على المزامنة عند الفتح لا على الدفع الفوري وحده |
| A-06 | فئة `frequent_absentees` تعتمد استعلاماً على جداول نظام الحضور؛ فئة `late_payers` تُبنى **حصراً** فوق `has_active_subscription()` (NT-06) | عالٍ | لو غير جاهزة: تُبنى الفئات يدوياً (قائمة user_ids يلصقها الإداري) كحل مؤقت |

## 0.4 — نطاق العمل (Scope Fence)

- ✅ **داخل النطاق:**
  - إعلانات البورد الأسود: نشر / تصنيف (🔴🟡🟢 عبر ENUM `announce_level`) / تثبيت / جدولة نشر لاحق / استهداف (الكل، طلاب، محاضرون، فئة ديناميكية) — **داخل حدود المؤسسة حصراً**
  - الإشعارات الفردية النظامية الواردة من الأنظمة الأخرى (جدولة، ملفات، دردشة، دفعات، حضور/التماسات)
  - تذكير استباقي قبل 30 دقيقة من الأحداث الحضورية (التسليم فقط — الأحداث ملك نظام الجدولة)
  - التسليم عبر 3 قنوات: Push (FCM بمواضيع مؤسسية) + داخل التطبيق (Realtime) + مزامنة عند الفتح (Hive)
  - تصفية الطالب: الكل / غير مقروء / مثبّت / حسب المادة — وتصفية المحاضر: الكل / غير مقروء
  - مركز إشعارات موحّد لكل دور، مع حالة قراءة لكل عنصر
- ❌ **خارج النطاق:**
  - محتوى الدردشة نفسه (نظام مستقل — نستقبل منه أحداثاً فقط)
  - البريد الإلكتروني والرسائل النصية كقنوات تسليم
  - تفضيلات إشعارات لكل مستخدم (كتم قنوات/مواد) — دين تقني مقبول (القسم 2.3)
  - القراءة الآلية لإشعارات التحويل البنكي (خارج نطاق الإصدار وفق MVP 5.5)
  - تحليلات فتح الإعلانات (open-rate dashboards)

---

# القسم 1: الملخص التنفيذي وسياق المشكلة 📋

## 1.1 — بيان المشكلة

تعتمد الجامعات حالياً على مجموعات واتساب ولوحات ورقية لإيصال المعلومات، فتضيع الإعلانات الحرجة (تغيير قاعة امتحان، استحقاق مالي) وسط الضجيج، ولا يمكن إثبات وصولها ولا استهداف فئة بعينها. كل نظام في Academic Hub (جدولة، دفعات، حضور، ملفات، دردشة) يحتاج إيصال أحداثه للمستخدم المناسب في وقتها — وبناء قناة تسليم داخل كل نظام على حدة يعني 5 تطبيقات متكررة وغير متسقة لنفس المنطق. تكلفة عدم الحل: طالب يفوته امتحان بسبب تغيير قاعة لم يصله = فشل مباشر في القيمة الجوهرية للمنصة. وعلى مستوى المنصة متعددة المؤسسات: أي تسريب بين مؤسستين = فشل تجاري كامل.

## 1.2 — الهدف القابل للقياس

تمكين أي نظام في المنصة من إيصال حدث إلى المستخدم المستهدف خلال **≤ 10 ثوانٍ (p95)** من وقوعه للأجهزة المتصلة، وضمان ظهوره في مركز الإشعارات خلال **≤ 3 ثوانٍ من أول فتح للتطبيق** للأجهزة غير المتصلة — مع قدرة إداري المؤسسة على بث إعلان لكل مستخدمي مؤسسته (~2,100) بعملية كتابة واحدة، وبعزل مؤسسي مطلق.

## 1.3 — معايير النجاح

| معيار | القيمة المستهدفة | كيف يُقاس |
|---|---|---|
| زمن التسليم (حدث → إشعار على جهاز متصل) p95 | < 10 ثوانٍ | طابع زمني في payload يقارن عند الاستلام (عينة أجهزة اختبار) |
| نسبة نجاح تسليم Push | ≥ 97% من التوكنات الفعّالة | تقارير FCM + سجل delivery_log |
| زمن تحميل مركز الإشعارات p95 | < 400ms لآخر 50 عنصراً | قياس زمن الاستعلام من جانب العميل |
| بث إعلان عام | صف واحد في DB مهما كان عدد المستقبلين | مراجعة الكود / خطة الاستعلام |
| فقدان الإشعارات عند انقطاع الشبكة | 0% (المزامنة عند الفتح تلتقط كل شيء) | اختبار تكاملي: قطع الشبكة → أحداث → إعادة اتصال |
| دقة تنفيذ الإعلان المجدول | ± 60 ثانية من الموعد | مقارنة `scheduled_at` بـ `published_at` |
| **العزل المؤسسي** | **0 تسريب — إعلان مؤسسة A لا يصل بأي قناة لمستخدم في مؤسسة B** | اختبار تكاملي إلزامي (تعريف الانتهاء 7.4) |

## 1.4 — ما ليس هذا النظام (Anti-Goals)

- **ليس نظام مراسلة:** لا يُنشئ محادثات ولا ردوداً — الدردشة نظام مستقل يرسل إلينا أحداثاً فقط
- **ليس ضامن تسليم Push بنسبة 100%:** FCM لا يضمن ذلك لأي أحد — الضمانة الحقيقية هي المزامنة عند الفتح، وPush مجرد تسريع
- **ليس محرك قرارات:** لا يقرر «من هو متكرر الغياب» ولا «من هو متأخر مالياً» — يستعلم تعريفاً تملكه أنظمة الحضور/الدفعات (`has_active_subscription()` حصراً للاشتراك)
- **ليس أرشيفاً دائماً:** الإشعارات الفردية تُؤرشف بعد 180 يوماً؛ الإعلانات فقط تُحفظ دائماً (soft delete)

---

# القسم 2: القيود التقنية واختيار التقنيات 🔧

## 2.1 — القيود المفروضة (Hard Constraints)

| نوع القيد | التفصيل | مصدره |
|---|---|---|
| تقني | المنصة Flutter + Supabase (Postgres, RLS, Realtime, Edge Functions) | Core Data v2.1 |
| تقني | **تعدد المؤسسات: `institution_id` في كل جدول وكل فهرس وكل سياسة RLS** | Core Data §4.0 · قرار MVP #20 |
| تقني | العمل دون اتصال بعد أول مزامنة (Hive) | MVP 3.1.2 |
| تقني | حالة الاشتراك تُقرأ حصراً من `has_active_subscription()` | MVP §3.7.2 |
| فريق | فريق صغير (1–3 مطورين) — لا طاقة لتشغيل بنية رسائل مستقلة (Kafka/RabbitMQ) | سياق المشروع |
| ميزانية | خطة Supabase Pro + FCM مجاني — **مع بند مُجدول من اليوم الأول لرفع حصة Realtime** (Core Data [D-04]) | سياق المشروع |
| تنظيمي | قرارات الإداري تُوثَّق ولا تُحذف (MVP 5.0) → soft delete إلزامي للإعلانات | MVP v3.6 |
| وظيفي | الإداري هو الناشر الوحيد داخل مؤسسته؛ الطالب قراءة فقط؛ المحاضر قراءة فقط ضمن هذا النظام | Roles Matrix — MVP القسم 7 |

## 2.2 — مصفوفة اختيار التقنيات

### [D-01] البنية الأساسية للنظام

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **Supabase: Postgres + Realtime + Edge Functions ✅** | RLS أصلي يطبّق فصل الأدوار والعزل المؤسسي بلا كود إضافي؛ Realtime يغطي التسليم داخل التطبيق؛ نفس بنية بقية الأنظمة (تكامل عبر triggers مباشرة) | ميزانية اتصالات Realtime تُحسب على أساس 10 مؤسسات — Realtime هو «أثقل مستهلك للحصة بعد الامتحانات» (MVP §1.2)، ورفع الحصة بند مُجدول من اليوم الأول؛ Edge Functions لها cold start ~200-400ms | يقبل: صفر بنية إضافية، وتكامل الأنظمة الأخرى يصبح trigger واحداً |
| Firebase (Firestore + Functions) ❌ | تكامل FCM أسهل | يقسم البيانات بين قاعدتين → فقدان الـ triggers والـ joins والـ RLS الموحد | رُفض: كسر وحدة مصدر الحقيقة |
| خدمة Node.js مستقلة + Redis Pub/Sub ❌ | تحكم كامل، أداء أعلى نظرياً | مكوّنان إضافيان للتشغيل والمراقبة | رُفض: تعقيد تشغيلي لا يبرره الحجم |

### [D-02] قناة الدفع (Push Delivery) — مُصحَّح NT-01

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **FCM (توكنات فردية + Topics مؤسسية حصراً) ✅** | مجاني بلا حدود عملية؛ Topic مؤسسي واحد يحل بث الإعلان لمؤسسة كاملة برسالة API واحدة | لا ضمان تسليم (best-effort)؛ إدارة دورة حياة التوكنات علينا؛ **الانضباط في تسمية المواضيع مسؤوليتنا — يفرضه حارس برمجي في dispatch** | يقبل: المعيار الفعلي لـ Android، والمزامنة عند الفتح تعوّض نقص الضمان |
| OneSignal ❌ | لوحة إدارة جاهزة | طرف ثالث يرى بيانات المستخدمين (حساسية الفئات المالية) | رُفض: تسريب PII لطرف ثالث |
| Self-hosted (ntfy / Gotify) ❌ | سيادة كاملة | WebSocket دائم → استهلاك بطارية؛ لا يعمل موثوقاً مع قيود Android | رُفض: غير عملي على الموبايل |

**قاعدة المواضيع الملزمة (تصحيح NT-01 — لا استثناء):**

| الموضوع الملغى (v1.0) | البديل الإلزامي (v1.1) |
|---|---|
| `all_users` | `inst_{institution_id}_all` |
| `role_students` | `inst_{institution_id}_students` |
| `role_lecturers` | `inst_{institution_id}_lecturers` |

- الاشتراك عند تسجيل الدخول يُشتق من `institution_id` claim في JWT **حصراً** — العميل لا يمرر اسم موضوع أبداً.
- إلغاء الاشتراك من كل مواضيع المؤسسة عند تسجيل الخروج أو تعطيل الحساب.
- دالة `dispatch` **ترفض** إرسال أي رسالة topic لا يبدأ اسمها بـ `inst_` (حارس برمجي ضد الرجوع للنمط القديم — اختبار وحدة إلزامي عليه).

### [D-03] نموذج التوزيع (Fan-out Model) — القرار الأهم في الوثيقة

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **هجين: fan-out-on-read للإعلانات + fan-out-on-write للإشعارات الفردية ✅** | الإعلان المؤسسي = صف واحد + جدول حالة قراءة يُملأ عند التفاعل فقط؛ الإشعار الفردي = صف لكل مستقبل؛ استعلام «غير المقروء» يبقى بسيطاً في الحالتين | منطقان مختلفان في العميل؛ حساب عدّاد غير المقروء يتطلب استعلامين (يُدمجان في view) | يقبل: يجمع كفاءة الكتابة مع بساطة القراءة |
| fan-out-on-write للجميع ❌ | نموذج واحد بسيط | إعلان مؤسسي واحد = ~2,100 صف؛ على مستوى المنصة ملايين الصفوف الزائدة/فصل بلا قيمة | رُفض: تضخم كتابي بلا مبرر |
| fan-out-on-read للجميع ❌ | صفر تضخم | الاستهداف الديناميكي يتطلب إعادة حساب الفئة عند كل قراءة → نتائج غير مستقرة زمنياً | رُفض: الفئات يجب أن تتجمد لحظة الإرسال (قرار أمني — 0.2 س4) |

### [D-04] جدولة الإعلانات المؤجلة

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **pg_cron (داخل Postgres) كل 60 ثانية ✅** | مدمج في Supabase؛ لا مكوّن خارجي؛ transactional مع بيانات الإعلان | دقة ± 60 ثانية (مقبولة — معيار 1.3)؛ يتطلب دالة نشر idempotent | يقبل: أبسط حل يحقق الدقة المطلوبة |
| Edge Function مع مجدول خارجي ❌ | مرونة أكبر | مكوّن إضافي + مشكلة تزامن | رُفض |
| جدولة من جهاز الإداري ❌ | صفر بنية خلفية | تفشل لو أُغلق التطبيق | رُفض: موثوقية معدومة |

### [D-05] التخزين المحلي والعمل دون اتصال

**Hive** (مفروض من MVP — لا مقارنة): صندوقان `announcements_box` و`notifications_box`، مزامنة تفاضلية عند الفتح عبر `?since=last_sync_at`، وحالة القراءة تُخزن محلياً وتُزامَن كدفعة عند عودة الاتصال (القسم 6.1 — idempotent).

## 2.3 — الديون التقنية المقبولة عمداً

| الدين | لماذا الآن | متى يُسدد |
|---|---|---|
| لا تفضيلات إشعارات للمستخدم (كتم مادة/قناة) | يقلل نطاق الـ MVP | عند تجاوز متوسط 10 إشعارات/يوم/طالب أو أول موجة شكاوى |
| الفئات الديناميكية = فئتان ثابتتان فقط (`late_payers` فوق `has_active_subscription()` · `frequent_absentees` فوق استعلام نظام الحضور) — لا محرك segments عام | فئتان فقط مطلوبتان في MVP 5.2 | عند طلب فئة ثالثة |
| لا retry queue مستقل لفشل FCM — المزامنة عند الفتح شبكة الأمان | يوفر بناء طابور رسائل كامل | إذا هبطت نسبة تسليم Push دون 90% |
| أرشفة الإشعارات = حذف بعد 180 يوماً (لا أرشيف بارد) | بيانات عابرة؛ الإعلانات محفوظة دائماً | عند ظهور متطلب تدقيق يتجاوز 180 يوماً |

---

# القسم 3: معمارية النظام والمخططات 🏗️

## 3.1 — المخطط العام (System Context)

\`\`\`mermaid
graph TB
    subgraph Producers["🏭 الأنظمة المصدرة للأحداث"]
        SCH[نظام الجدولة والأحداث]
        PAY[نظام الدفعات]
        ATT[نظام الحضور والالتماسات]
        FIL[مستودع الملفات]
        CHT[نظام الدردشة]
        HAL[إدارة القاعات]
    end

    subgraph Core["🔔 نواة نظام الإشعارات — Supabase"]
        TRG[DB Triggers<br/>التقاط الأحداث]
        NTB[(notifications<br/>إشعارات فردية)]
        ANB[(announcements<br/>إعلانات البورد — institution_id)]
        CRON[pg_cron<br/>مجدول: نشر مؤجل + تذكير 30 دق + أرشفة]
        EF[Edge Function: dispatch<br/>بناء وإرسال Push — حارس inst_ فقط]
        RT[Supabase Realtime]
    end

    ADMIN[👔 إداري المؤسسة<br/>الناشر الوحيد داخل مؤسسته] -->|نشر / جدولة / تثبيت / استهداف| ANB
    SCH & PAY & ATT & FIL & CHT & HAL --> TRG
    TRG --> NTB
    NTB --> EF
    ANB -->|عند النشر| EF
    CRON --> ANB
    CRON -->|تذكير الأحداث| NTB
    EF --> FCM[Firebase Cloud Messaging<br/>topics: inst_id_all / _students / _lecturers]
    NTB & ANB --> RT

    FCM --> APP[📱 تطبيق Flutter<br/>Hive Cache]
    RT --> APP
    APP -->|مزامنة عند الفتح ?since=| NTB
    APP -->|مزامنة عند الفتح| ANB
\`\`\`

## 3.2 — جدول المكونات والمسؤوليات

| المكون | مسؤوليته الوحيدة | ماذا لو سقط؟ | استراتيجية التعافي |
|---|---|---|---|
| DB Triggers | تحويل حدث نظام مصدر إلى صف إشعار موحّد | لا تُنشأ إشعارات جديدة (البيانات الأصلية سليمة) | جزء من Postgres |
| جدولا notifications/announcements | مصدر الحقيقة الوحيد لكل ما يُعرض | توقف كامل للنظام | نسخ Supabase الاحتياطي اليومي + PITR |
| Edge Function `dispatch` | قراءة الصفوف غير المرسلة → التحقق من بادئة `inst_` → إرسال FCM → تحديث حالة الإرسال | يتوقف Push فقط — التطبيق يلتقط كل شيء عند الفتح | stateless — إعادة تشغيل تلقائية؛ عمود `push_sent_at` يمنع الإرسال المزدوج |
| pg_cron | نشر المجدول + توليد تذكيرات الـ 30 دقيقة + الأرشفة | تتأخر الإعلانات المجدولة والتذكيرات | مهام idempotent — التشغيل التالي يلتقط المتراكم |
| FCM | إيصال Push للأجهزة | تأخر تنبيه فقط — لا فقدان بيانات | الاعتماد الجوهري على المزامنة عند الفتح |
| Realtime | تحديث حي لمركز الإشعارات المفتوح | يتراجع العميل إلى polling كل 60 ثانية | fallback مدمج في العميل |
| Hive Cache | عرض دون اتصال + تقليل الاستعلامات | إعادة مزامنة كاملة عند فتح التطبيق (مقبول) | الكاش قابل للتفريغ دائماً — DB هي الحقيقة |

## 3.3 — تدفق البيانات للعمليات الحرجة

### المسار 1: إداري المؤسسة ينشر إعلاناً عاجلاً على البورد الأسود

\`\`\`mermaid
sequenceDiagram
    participant AD as 👔 إداري المؤسسة
    participant DB as Postgres
    participant EF as Edge Fn: dispatch
    participant F as FCM
    participant ST as 📱 أجهزة مستخدمي المؤسسة

    AD->>DB: INSERT announcement (institution_id, level=urgent, target=all, status=published)
    Note over DB: RLS: role=admin + institution_id = my_institution()<br/>⚠️ نقطة فشل: رفض RLS → 403
    DB->>EF: Webhook (database trigger → invoke)
    Note over EF: ⚠️ نقطة فشل: cold start أو timeout<br/>→ push_sent_at=NULL → الـ sweeper يلتقطه خلال 60 ثا
    EF->>EF: حارس: الموضوع يبدأ بـ inst_ وإلا رفض
    EF->>F: send to topic "inst_{institution_id}_all" (رسالة API واحدة)
    F-->>ST: push خلال 1–5 ثوانٍ (best-effort — مؤسسة واحدة فقط)
    EF->>DB: UPDATE push_sent_at = now()
    DB-->>ST: Realtime broadcast (قناة مؤسسية — للتطبيقات المفتوحة)
    ST->>DB: عند الفتح لاحقاً: GET announcements?since=last_sync (تحت RLS المؤسسي)
    Note over ST: الطبقة الثالثة تضمن الوصول 100%<br/>حتى لو فشل Push وRealtime معاً
\`\`\`

### المسار 2: نظام مصدر يولّد إشعاراً فردياً (مثال: رفض دفعة)

\`\`\`mermaid
sequenceDiagram
    participant PAY as نظام الدفعات
    participant TR as Trigger
    participant DB as notifications
    participant EF as dispatch
    participant F as FCM
    participant ST as 📱 جهاز الطالب

    PAY->>TR: UPDATE payment SET status='rejected'
    TR->>DB: INSERT notification (user_id, type='payment_rejected', ref_id)
    Note over TR,DB: داخل نفس الـ transaction<br/>= استحالة ضياع الإشعار إذا نجحت العملية الأصلية
    DB->>EF: trigger invoke
    EF->>F: send to device tokens الخاصة بالطالب
    F-->>ST: push
    Note over F,ST: ⚠️ فشل token (منتهٍ/أُلغي)<br/>→ يُعلَّم is_valid=false ويُحذف بعد 30 يوماً
\`\`\`

### المسار 3: تذكير الـ 30 دقيقة قبل حدث حضوري — مُصحَّح NT-07

\`\`\`mermaid
sequenceDiagram
    participant CR as pg_cron (كل 5 دقائق)
    participant SCH as جداول نظام الجدولة (قراءة فقط)
    participant DB as notifications
    participant EF as dispatch

    CR->>SCH: SELECT الأحداث الحضورية التي تبدأ خلال [30, 35] دقيقة<br/>× الطلاب حيث enrollments.status = 'ACTIVE' حصراً<br/>(الطالب COMPLETED/DROPPED لا يُذكَّر)
    SCH-->>CR: قائمة (event_id × الطلاب النشطون في المادة)
    CR->>DB: INSERT ... ON CONFLICT (user_id, type, ref_id) DO NOTHING
    Note over CR,DB: ⚠️ التشغيل المزدوج آمن — القيد الفريد يمنع التكرار
    DB->>EF: dispatch → FCM
\`\`\`

## 3.4 — قرارات معمارية جوهرية

### [D-06] المركزية: قناة تسليم واحدة أم كل نظام يشعّر بنفسه؟

| الخيار | المزايا | العيوب | لماذا قُبل/رُفض |
|---|---|---|---|
| **قناة مركزية واحدة — الأنظمة تنتج أحداثاً فقط ✅** | نقطة واحدة لإدارة التوكنات والقنوات والمراقبة والعزل المؤسسي؛ إضافة نوع إشعار جديد = trigger + سطر في قاموس الأنواع | الـ triggers توزّع منطق الالتقاط على جداول الأنظمة الأخرى (يُدار بتسمية موحدة `notify_on_*` وتوثيق مركزي في 4.5) | يقبل: يمنع 6 تطبيقات متكررة لنفس المنطق — **وهذا القرار ملزم لكل الأنظمة (حُسم في التدقيق FR-06: لا outbox ولا إرسال مباشر من أي نظام آخر)** |
| كل نظام يرسل FCM مباشرة ❌ | استقلالية الأنظمة | تكرار إدارة التوكنات 6 مرات؛ لا مركز إشعارات موحّداً | رُفض |
| Message broker وسيط ❌ | فصل نظيف نظرياً | بنية إضافية كاملة لحمل لا يبررها | رُفض: over-engineering |

### [D-07] الإرسال: متزامن أم غير متزامن؟

**غير متزامن بالكامل** — كتابة الصف في DB هي «نجاح» العملية من منظور المصدر؛ الإرسال Push مسؤولية `dispatch` اللاحقة. **المقايضة المقبولة:** تأخر يصل إلى 60 ثانية في أسوأ حالة مقابل أن فشل FCM لا يفشل أبداً العملية الأصلية.

---

# القسم 4: نماذج البيانات وتصميم قاعدة البيانات 🗄️

> **قاعدة حاكمة (من التدقيق):** جدول `announcements` أساسه تعريف **Core Data v2.1 §4.2-13** (يحمل `institution_id` و`author_id` و`title/body` وENUM `announce_level` أصلاً) — هذه الوثيقة تضيف **أعمدة امتداد فقط**. مخطط `notifications` المفصّل أدناه هو المعتمد ويُرحَّل إلى Core Data v2.2 (NT-03).

## 4.1 — مخطط الكيانات (ERD)

\`\`\`mermaid
erDiagram
    PROFILES ||--o{ NOTIFICATIONS : "يستقبل"
    PROFILES ||--o{ DEVICE_TOKENS : "يمتلك"
    PROFILES ||--o{ ANNOUNCEMENT_STATES : "حالة قراءته"
    PROFILES ||--o{ ANNOUNCEMENTS : "ينشر (إداري فقط)"
    INSTITUTIONS ||--o{ ANNOUNCEMENTS : "تملك"
    ANNOUNCEMENTS ||--o{ ANNOUNCEMENT_STATES : "لها"
    ANNOUNCEMENTS ||--o{ ANNOUNCEMENT_RECIPIENTS : "استهداف فئوي مجمّد"

    ANNOUNCEMENTS {
        uuid id PK
        uuid institution_id FK "من Core Data — إلزامي"
        uuid author_id FK "profiles(id)"
        text title
        text body
        announce_level level "ENUM من Core Data: urgent|important|normal"
        text target_type "all|students|lecturers|segment — امتداد"
        text segment_key "late_payers|frequent_absentees|null — امتداد"
        boolean is_pinned
        timestamptz scheduled_at "امتداد"
        timestamptz published_at
        text status "draft|scheduled|published|retracted — امتداد"
        timestamptz push_sent_at "امتداد"
        timestamptz deleted_at "soft delete"
    }

    NOTIFICATIONS {
        bigint id PK
        uuid user_id FK "profiles(id)"
        text type "قاموس الأنواع 4.5"
        text ref_type
        text ref_id "TEXT — معرّفات مختلطة BIGINT/UUID عبر الأنظمة"
        bigint course_id FK "courses(id) BIGINT — للتصفية حسب المادة"
        text title
        text body
        timestamptz created_at
        timestamptz read_at
        timestamptz push_sent_at
    }

    ANNOUNCEMENT_STATES {
        uuid announcement_id PK,FK
        uuid user_id PK,FK "profiles(id)"
        timestamptz read_at
    }

    ANNOUNCEMENT_RECIPIENTS {
        uuid announcement_id PK,FK
        uuid user_id PK,FK "profiles(id)"
    }

    DEVICE_TOKENS {
        uuid id PK
        uuid user_id FK "profiles(id)"
        text fcm_token UK
        text platform
        boolean is_valid
        timestamptz last_seen_at
    }
\`\`\`

## 4.2 — تعريف الجداول (Schema) — يحل محل §4.2 في v1.0

\`\`\`sql
-- announcements: أساسه تعريف Core Data v2.1 §4.2-13 (institution_id + author_id + title + body
-- + level من ENUM announce_level + is_pinned + published_at + created_at + deleted_at موجودة أصلاً).
-- أعمدة الامتداد التي تضيفها هذه الوثيقة (migration إضافية — NT-02):
ALTER TABLE announcements ADD COLUMN target_type TEXT NOT NULL DEFAULT 'all'
    CHECK (target_type IN ('all','students','lecturers','segment'));
ALTER TABLE announcements ADD COLUMN segment_key TEXT
    CHECK (segment_key IN ('late_payers','frequent_absentees'));
ALTER TABLE announcements ADD COLUMN scheduled_at TIMESTAMPTZ;             -- non-null = نشر مؤجل [D-04]
ALTER TABLE announcements ADD COLUMN status TEXT NOT NULL DEFAULT 'published'
    CHECK (status IN ('draft','scheduled','published','retracted'));
ALTER TABLE announcements ADD COLUMN push_sent_at TIMESTAMPTZ;             -- NULL = يلتقطه الـ sweeper
ALTER TABLE announcements ADD CONSTRAINT segment_consistency
    CHECK ((target_type = 'segment') = (segment_key IS NOT NULL));
ALTER TABLE announcements ADD CONSTRAINT scheduled_needs_time
    CHECK (status != 'scheduled' OR scheduled_at IS NOT NULL);

-- الفهرس المؤسسي للبورد الأسود (كل فهرس يبدأ بـ institution_id — Core Data §4.0):
CREATE INDEX idx_announcements_board
    ON announcements (institution_id, published_at DESC)
    WHERE status = 'published' AND deleted_at IS NULL;
CREATE INDEX idx_announcements_scheduled
    ON announcements (scheduled_at)
    WHERE status = 'scheduled';

-- announcement_recipients: تجميد الاستهداف الفئوي لحظة النشر (قرار أمني — 0.2 س4)
CREATE TABLE announcement_recipients (
    announcement_id UUID NOT NULL REFERENCES announcements(id),
    user_id         UUID NOT NULL REFERENCES profiles(id),        -- NT-04: profiles لا users
    PRIMARY KEY (announcement_id, user_id)
);
CREATE INDEX idx_ann_recipients_user ON announcement_recipients (user_id);

-- announcement_states: حالة قراءة المستخدم للإعلان — fan-out-on-read
CREATE TABLE announcement_states (
    announcement_id UUID NOT NULL REFERENCES announcements(id),
    user_id         UUID NOT NULL REFERENCES profiles(id),        -- NT-04
    read_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (announcement_id, user_id)                        -- تكرار وسم القراءة idempotent
);

-- notifications: الإشعارات الفردية النظامية — fan-out-on-write
-- هذا المخطط المفصّل هو المعتمد (NT-03) ويُرحَّل إلى Core Data v2.2
-- لا is_pinned هنا — التثبيت خاصية إعلانات حصراً (MVP §5.2)
-- العزل المؤسسي يتحقق بنيوياً: كل إشعار مملوك لمستخدم واحد وRLS تفرض user_id = auth.uid()
CREATE TABLE notifications (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id       UUID NOT NULL REFERENCES profiles(id),          -- NT-04
    type          TEXT NOT NULL,                                  -- من قاموس الأنواع (4.5)
    ref_type      TEXT NOT NULL,                                  -- الكيان المصدر: event/payment/file/post...
    ref_id        TEXT NOT NULL,                                  -- NT-04: TEXT — معرّفات Core Data مختلطة
                                                                  -- (BIGINT للأحداث/الملفات، UUID للمستخدمين/الدفعات)
    course_id     BIGINT REFERENCES courses(id),                  -- NT-04: BIGINT — تصفية «حسب المادة»
    title         TEXT NOT NULL,
    body          TEXT NOT NULL CHECK (char_length(body) <= 500),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at       TIMESTAMPTZ,                                    -- NULL = غير مقروء
    push_sent_at  TIMESTAMPTZ,
    -- منع التكرار المطلق (تذكير مزدوج، trigger أعيد تنفيذه):
    CONSTRAINT uq_notification_dedup UNIQUE (user_id, type, ref_id)
);

-- device_tokens: دورة حياة توكنات FCM
CREATE TABLE device_tokens (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID NOT NULL REFERENCES profiles(id),           -- NT-04
    fcm_token    TEXT NOT NULL UNIQUE,                            -- UPSERT يعيد ربطه عند جهاز مشترك
    platform     TEXT NOT NULL DEFAULT 'android' CHECK (platform IN ('android','ios')),
    is_valid     BOOLEAN NOT NULL DEFAULT true,                   -- false عند رفض FCM → حذف بعد 30 يوماً
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
\`\`\`

**سياسات RLS الحاكمة (النمط الإلزامي — كل سياسة إعلانات تبدأ بالمؤسسة):**

\`\`\`sql
ALTER TABLE announcements ENABLE ROW LEVEL SECURITY;
ALTER TABLE announcement_recipients ENABLE ROW LEVEL SECURITY;
ALTER TABLE announcement_states ENABLE ROW LEVEL SECURITY;
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;
ALTER TABLE device_tokens ENABLE ROW LEVEL SECURITY;

-- قراءة الإعلانات: مؤسستي أولاً، ثم مطابقة الاستهداف لدوري أو لوجودي في recipients
CREATE POLICY ann_read ON announcements FOR SELECT USING (
    institution_id = my_institution()
    AND status = 'published' AND deleted_at IS NULL
    AND (
        target_type = 'all'
        OR (target_type = 'students'  AND my_role() = 'student')
        OR (target_type = 'lecturers' AND my_role() = 'lecturer')
        OR (target_type = 'segment' AND EXISTS (
            SELECT 1 FROM announcement_recipients r
            WHERE r.announcement_id = announcements.id AND r.user_id = auth.uid()))
        OR my_role() = 'admin'
    )
);

-- النشر/التعديل: إداري مؤسستي فقط
CREATE POLICY ann_write ON announcements FOR INSERT WITH CHECK (
    institution_id = my_institution() AND my_role() = 'admin' AND author_id = auth.uid()
);
CREATE POLICY ann_update ON announcements FOR UPDATE USING (
    institution_id = my_institution() AND my_role() = 'admin'
);

-- recipients: المستخدم يرى صفه فقط — الدفاع ضد أخطر Information Disclosure (6.3)
CREATE POLICY recipients_self ON announcement_recipients FOR SELECT USING (
    user_id = auth.uid()
);

-- الإشعارات الفردية وحالات القراءة والتوكنات: ملكية شخصية صرفة
CREATE POLICY notif_self ON notifications FOR SELECT USING (user_id = auth.uid());
CREATE POLICY states_self ON announcement_states FOR ALL USING (user_id = auth.uid())
    WITH CHECK (user_id = auth.uid());
CREATE POLICY tokens_self ON device_tokens FOR ALL USING (user_id = auth.uid())
    WITH CHECK (user_id = auth.uid());
\`\`\`

## 4.3 — استراتيجية الفهرسة

| الفهرس | الاستعلام الذي يخدمه | تكلفته على الكتابة |
|---|---|---|
| `notifications (user_id, created_at DESC)` | مركز الإشعارات: آخر 50 عنصراً — الاستعلام الأكثر تكراراً في النظام كله | ~10% إبطاء على INSERT — مقبول لأن القراءة 100:1 |
| `notifications (user_id) WHERE read_at IS NULL` (partial) | عدّاد غير المقروء (badge) | ضئيلة — ينكمش ذاتياً مع القراءة |
| `notifications (user_id, course_id)` | تصفية الطالب «حسب المادة» | ~8% على INSERT — مقبول |
| `announcements (institution_id, published_at DESC) WHERE status='published' AND deleted_at IS NULL` (partial) | قائمة البورد الأسود لمؤسسة | شبه معدومة — كتابة الإعلانات نادرة |
| `announcements (scheduled_at) WHERE status='scheduled'` (partial) | مسح pg_cron كل 60 ثانية | معدومة عملياً |
| `device_tokens (user_id) WHERE is_valid` (partial) | جمع توكنات مستخدم عند dispatch | ضئيلة |
| `announcement_recipients (user_id)` | RLS: «هل هذا الإعلان الفئوي موجّه لي؟» | ضئيلة — كتابة دفعة واحدة عند النشر |

## 4.4 — الأسئلة الإلزامية

- **الترحيلات (Migrations):** عبر Supabase migrations المتسلسلة؛ التغييرات إضافية فقط — لا `ALTER` مانع للكتابة على `notifications` بعد الإطلاق.
- **الحذف:** الإعلانات **soft delete حصراً** (`deleted_at` + status=`retracted`). الإشعارات الفردية **hard delete بالأرشفة** بعد 180 يوماً (مهمة cron أسبوعية بدفعات 10,000 صف).
- **التوسع (مُصحَّح NT-05):** بحجم 20,000 مستخدم، `notifications` يصل **~26 مليون صف/فصل** — range partitioning على `created_at` شهرياً **يُفعَّل من السنة الأولى** (لا مؤجّلاً)، والمفتاح طبيعي لأن الأرشفة والقراءة كلاهما زمنيان.
- **التوافقية:** الكتابة الأصلية (حدث المصدر + صف الإشعار) **strong consistency** — نفس الـ transaction عبر trigger. التسليم (Push/Realtime) **eventual** بسقف 60 ثانية ([D-07]).

## 4.5 — قاموس أنواع الإشعارات (Event Type Registry)

المرجع الموحّد لكل الأنظمة المصدرة — إضافة نوع جديد تتطلب صفاً هنا + trigger في نظام المصدر. `ref_id` دائماً TEXT (يحمل BIGINT أو UUID مُسلسلاً):

| type | المصدر | المستقبل | ref_type | مثال النص |
|---|---|---|---|---|
| `event_reminder` | pg_cron ← جداول الجدولة (ACTIVE فقط — NT-07) | طالب | event | «محاضرة تشريح تبدأ خلال 30 دقيقة — قاعة 3» |
| `event_changed` / `event_cancelled` | نظام الجدولة | طلاب المادة النشطون | event | «تغيّرت قاعة امتحان الفسيولوجي إلى المدرج الكبير» |
| `file_added` | مستودع الملفات (trigger مباشر — FR-06) | طلاب المادة النشطون | file | «ملف جديد في مادة الباطنة» |
| `contribution_reviewed` | مستودع الملفات | طالب | contribution | «اجتهادك: مقبول ✅» |
| `chat_post` / `chat_comment` / `chat_reply` | نظام الدردشة | طلاب المادة / محاضر / إداري | post | «منشور جديد من د. أحمد في قناة الجراحة» |
| `payment_confirmed` / `payment_rejected` / `payment_due` | نظام الدفعات | طالب | payment | «تم تأكيد دفعة رسوم الجامعة» |
| `excuse_result` | نظام الحضور | طالب | excuse | «التماس عذر الغياب: مقبول — سُجّل حضورك» |
| `hall_request_result` | إدارة القاعات | محاضر | hall_request | «طلب قاعة المحاضرة: مقبول — قاعة 5، الأحد 10ص» |
| `final_result_published` | التقارير والنتائج | طلاب المادة | course_result | «نتيجتك النهائية في مادة X متاحة الآن» |
| `support_message` | الدعم الفني | طالب / إداري | ticket | «رد جديد على استفسارك» |

---

# القسم 5: عقود الـ API والواجهات 🔌

## 5.1 — مبادئ التصميم

**[D-08]** الوصول عبر **PostgREST المدمج في Supabase** (قراءة الجداول والـ views تحت RLS) + **Edge Functions للعمليات المركّبة** (نشر إعلان فئوي، dispatch). البدائل المرفوضة: REST API مخصص كامل (❌)، GraphQL (❌). النسخ: مسار `/functions/v1/`. صيغة الخطأ الموحدة: `{"error": {"code": "...", "message": "..."}}`.

## 5.2 — توثيق الـ Endpoints

### `POST /functions/v1/announcements`
**الغرض:** نشر أو جدولة إعلان داخل مؤسسة الإداري (يشمل تجميد الفئة عند الاستهداف الفئوي) | **المصادقة:** JWT + role=admin — **`institution_id` يُشتق من الـ claim حصراً ولا يُقبل من الجسم** | **Rate Limit:** 30/دقيقة/إداري

**المدخلات:**
\`\`\`json
{
  "title": "string — مطلوب، 3-150 حرفاً",
  "body": "string — مطلوب، حتى 5000 حرف",
  "level": "urgent | important | normal — ENUM announce_level، افتراضي normal",
  "target_type": "all | students | lecturers | segment",
  "segment_key": "late_payers | frequent_absentees — مطلوب فقط عند segment",
  "is_pinned": "boolean — افتراضي false",
  "scheduled_at": "ISO-8601 UTC — اختياري؛ وجوده = جدولة، غيابه = نشر فوري",
  "idempotency_key": "uuid — مطلوب (القسم 6.1)"
}
\`\`\`

**المخرجات الناجحة (201):**
\`\`\`json
{
  "id": "uuid",
  "status": "published | scheduled",
  "recipients_count": 1840,
  "published_at": "ISO-8601 | null"
}
\`\`\`

**حالات الخطأ:**
| كود | متى تحدث | استجابة العميل المتوقعة |
|---|---|---|
| 400 | حقل ناقص/تجاوز حدود/`segment_key` بلا `target_type=segment` | عرض رسالة التحقق على الحقل |
| 401 | توكن منتهٍ | إعادة توجيه لتسجيل الدخول |
| 403 | الدور ليس admin | شاشة «غير مصرّح» — الدفاع في العمق |
| 409 | `idempotency_key` مكرر | اعتبار العملية ناجحة وعرض الإعلان الموجود |
| 422 | `scheduled_at` في الماضي (بتوقيت الخادم) | «اختر وقتاً مستقبلياً» |
| 429 | تجاوز rate limit | retry مع exponential backoff |
| 500 | خطأ داخلي | «حدث خطأ — أعد المحاولة»؛ الإعادة آمنة بفضل idempotency_key |

> **تجميد فئة `late_payers` (NT-06):** داخل دالة النشر، الفئة تُحسب **حصراً** بـ `SELECT p.id FROM profiles p WHERE p.institution_id = <inst> AND p.role = 'student' AND NOT has_active_subscription(p.id)` — ممنوع أي استعلام مستقل على جداول الدفعات (MVP §3.7.2: مصدر الحقيقة الوحيد لحالة الاشتراك).

### `GET /rest/v1/rpc/my_feed`
**الغرض:** القائمة الموحدة (إعلانات + إشعارات) مع التصفية — يغطي MVP 3.2 و4.2 | **المصادقة:** JWT (أي دور) | **Rate Limit:** 120/دقيقة/مستخدم

**المدخلات (query):** `filter=all|unread|pinned` · `course_id=bigint` (طلاب فقط) · `since=ISO-8601` (المزامنة التفاضلية) · `limit≤50` · `cursor`

**المخرجات الناجحة (200):**
\`\`\`json
{
  "items": [
    {
      "kind": "announcement | notification",
      "id": "uuid|bigint",
      "title": "...", "body": "...",
      "level": "urgent|important|normal|null",
      "is_pinned": false,
      "course_id": "bigint|null",
      "ref_type": "payment", "ref_id": "text",
      "created_at": "ISO-8601",
      "is_read": true
    }
  ],
  "next_cursor": "opaque-string|null",
  "unread_count": 7
}
\`\`\`

**حالات الخطأ:** 400 (فلتر غير معروف / limit>50) → تجاهل وإعادة الافتراضي · 401 → تسجيل دخول · 429 → إبطاء polling · 500 → عرض كاش Hive مع شارة «غير محدّث».

> ملاحظة RLS مدمجة: الدالة تُرجع تلقائياً — إعلانات **مؤسسة المستخدم** المطابقة لدوره أو الموجّهة له في `announcement_recipients`، وإشعاراته الفردية فقط. لا معامل `user_id` ولا `institution_id` في العقد إطلاقاً (منع IDOR).

### `POST /rest/v1/rpc/mark_read`
**الغرض:** وسم عناصر كمقروءة (دفعة — يخدم المزامنة بعد الأوفلاين) | **المصادقة:** JWT | **Rate Limit:** 60/دقيقة

**المدخلات:** `{"announcement_ids": ["uuid"], "notification_ids": [123]}` — حتى 200 عنصر/طلب
**المخرجات (200):** `{"updated": 14}`
**حالات الخطأ:** 400 (تجاوز 200 عنصر) → تقسيم الدفعة · 401 · 429 · 500 → إعادة المحاولة آمنة (idempotent). عناصر لا تخص المستخدم تُتجاهل صامتة (لا 403 — منع استكشاف المعرّفات).

### `POST /rest/v1/device_tokens` (UPSERT)
**الغرض:** تسجيل/تحديث توكن FCM عند كل إقلاع للتطبيق | **المصادقة:** JWT | **Rate Limit:** 10/دقيقة
**المدخلات:** `{"fcm_token": "string", "platform": "android|ios"}`
**المخرجات (201/200):** `{"id": "uuid"}` — التوكن الموجود لمستخدم آخر يُعاد ربطه بالمستخدم الحالي (جهاز مشترك).
**حالات الخطأ:** 400 (توكن فارغ) · 401 · 429 · 500 → إعادة عند الإقلاع التالي.

### `POST /functions/v1/announcements/{id}/retract` و `PATCH .../pin`
**الغرض:** سحب إعلان (soft) / تبديل التثبيت — داخل مؤسسة الإداري فقط | **المصادقة:** admin | **Rate Limit:** 30/دقيقة
**الأخطاء المميزة:** 404 (غير موجود أو محذوف **أو من مؤسسة أخرى** — لا تمييز في الرد لمنع الاستكشاف) · 409 عند retract لإعلان `retracted` أصلاً.

## 5.3 — العقود بين الخدمات الداخلية — مُصحَّح NT-01

| العقد | الصيغة | الضمانة |
|---|---|---|
| نظام مصدر → notifications | Trigger داخل نفس transaction العملية الأصلية (لا outbox — [D-06] + FR-06) | **exactly-once للكتابة** (ذرّية transaction + قيد `uq_notification_dedup`) |
| notifications/announcements → dispatch | Database webhook (invoke) + cron sweeper كل 60 ثا لصفوف `push_sent_at IS NULL` | **at-least-once** — الإرسال المزدوج النادر مقبول |
| dispatch → FCM | HTTP v1 API؛ **topics مؤسسية حصراً** (`inst_{institution_id}_all` / `_students` / `_lecturers`)، توكنات فردية للباقي؛ **حارس برمجي: رفض أي topic لا يبدأ بـ `inst_`** | best-effort — شبكة الأمان: المزامنة عند الفتح |
| اشتراك Topics | العميل يشترك عند تسجيل الدخول في مواضيع مؤسسته المشتقة من `institution_id` claim حصراً؛ إلغاء الاشتراك من كل مواضيع المؤسسة عند الخروج أو تعطيل الحساب | idempotent من طرف FCM |

---

# القسم 6: حالات الحافة، أنماط الفشل، والأمان 🛡️

## 6.1 — جرد حالات الحافة

| السيناريو | ماذا يحدث في التصميم؟ | المعالجة |
|---|---|---|
| طلبان متزامنان يعدلان نفس المورد (إداريان يثبّتان/يسحبان نفس الإعلان) | آخر كتابة تكسب على `is_pinned` (غير ضار)؛ `retract` محمي | `UPDATE ... WHERE status='published'` — الطلب الثاني يصطدم بـ 409 |
| المستخدم يرسل مدخلات 10x المتوقع | قيود CHECK على مستوى DB + حد 200 عنصر في العقد | رفض 400 قبل أي معالجة |
| انقطاع الشبكة في منتصف نشر إعلان | الإداري لا يعرف هل نجح | `idempotency_key` إلزامي — إعادة الإرسال تُرجع 409 مع الإعلان الموجود |
| انقطاع الشبكة أثناء mark_read | حالة القراءة محلية فقط في Hive | طابور محلي يُعاد إرساله عند الاتصال — idempotent بالكامل |
| خدمة خارجية بطيئة وليست ساقطة (FCM يستجيب في 30 ثا) | dispatch قد يتجاوز timeout الدالة | timeout صريح 10 ثوانٍ/دفعة FCM + دفعات 500 توكن؛ الفاشل يلتقطه الـ sweeper |
| رسالة مكررة (trigger أعيد تنفيذه / cron مزدوج) | خطر إشعار مكرر | `uq_notification_dedup` + `ON CONFLICT DO NOTHING` |
| التوقيت (timezones / انحراف ساعة الجهاز) | `scheduled_at` قد يُفسَّر بتوقيت الجهاز | كل التخزين والمقارنات **UTC بتوقيت الخادم حصراً**؛ رفض 422 لأي جدولة ماضية |
| مستخدم عُطّل حسابه وله توكنات فعّالة | قد يستمر وصول Push | dispatch يستبعد بـ join على `profiles.status='active'`؛ التعطيل يُبطل التوكنات فوراً (trigger) + إلغاء اشتراك مواضيع المؤسسة |
| إعلان مجدول والفئة تغيّرت قبل موعد النشر (طالب سدّد متأخراته) | من يستقبل؟ | **قرار صريح:** الفئة تُجمَّد **لحظة النشر الفعلي** — `announcement_recipients` تُملأ داخل مهمة النشر بنداء `has_active_subscription()` لحظتها |
| جهاز مشترك بين طالبين (توكن FCM واحد) — **وقد يكونان من مؤسستين مختلفتين** | إشعارات المستخدم السابق تصل للحالي | UPSERT يعيد ربط التوكن بآخر من سجّل دخولاً + إبطاله عند الخروج + **إلغاء اشتراك مواضيع المؤسسة السابقة قبل الاشتراك في الجديدة** |
| طالب محجوب لتأخر الاشتراك (MVP 3.7.2) | هل تصله الإشعارات؟ | **نعم — استثناء صريح ومتعمد من قرار #15 (مثبَّت بالتدقيق NT-10):** الإشعارات قناة إبلاغ لا «ميزة» — الطالب المحجوب يجب أن يعرف قبول دفعته واقتراب امتحانه؛ الحجب يطبَّق على الميزات الأخرى |
| نقل مستخدم بين مؤسستين (حالة إدارية نادرة) | إعلانات المؤسسة القديمة في كاش Hive | تغيّر `institution_id` claim → تفريغ الكاش وإعادة الاشتراك بالمواضيع عند أول دخول |

## 6.2 — تحليل أنماط الفشل

| المكون | نمط الفشل | الاحتمالية | الأثر | الكشف | التعافي |
|---|---|---|---|---|---|
| Postgres | تعطل كامل | منخفضة | 🔴 حرج — النظام كله | Supabase health + تنبيه | إدارة Supabase + PITR؛ RTO ~15 دق |
| Edge Fn dispatch | فشل/timeout متكرر | متوسطة | 🟡 تأخر Push ≤ 60 ثا | مقياس `push_lag` (6.5) | sweeper cron يلتقط `push_sent_at IS NULL` |
| pg_cron | توقف صامت | منخفضة | 🟡 تأخر مجدولات وتذكيرات | heartbeat كل دقيقة + تنبيه عند غياب 5 دق | إعادة تفعيل؛ المهام idempotent |
| FCM | انقطاع إقليمي | منخفضة | 🟢 لا Push مؤقتاً | هبوط معدل النجاح | لا فعل — المزامنة عند الفتح تغطي |
| Realtime | انقطاع القناة / تجاوز حصة الاتصالات على مستوى المنصة | متوسطة | 🟢 المركز المفتوح لا يتحدث حياً | كشف انقطاع في العميل + مقياس اتصالات كلي | fallback تلقائي إلى polling كل 60 ثا؛ رفع الحصة بند مُجدول (NT-05) |
| device_tokens | تراكم توكنات ميتة | عالية (طبيعي) | 🟢 هدر إرسال | ردود FCM `UNREGISTERED` | وسم `is_valid=false` فوراً + حذف بعد 30 يوماً |
| Triggers | خطأ في trigger يفشل العملية الأصلية | منخفضة | 🔴 رفض دفعة يفشل لأن الإشعار فشل! | اختبارات تكاملية لكل trigger | كل trigger يلف الإدراج بـ `BEGIN...EXCEPTION WHEN OTHERS` — **فشل الإشعار لا يُفشل العملية الأصلية أبداً** |
| dispatch | رجوع خاطئ لموضوع عالمي (regression) | منخفضة | 🔴 تسريب عابر للمؤسسات | الحارس البرمجي يرفض + تنبيه فوري | اختبار وحدة دائم على الحارس |

## 6.3 — نموذج التهديدات (STRIDE)

| التهديد | مثال ملموس على هذا النظام | الدفاع المحدد |
|---|---|---|
| Spoofing | طالب ينتحل دور إداري لنشر إعلان مزيف | JWT + فحص role=admin في RLS **وفي** Edge Function (دفاع مزدوج) |
| Tampering | تعديل إعلان منشور لتغيير موعد امتحان؛ أو تمرير `institution_id` مزيف في الجسم | RLS: UPDATE للإداري داخل مؤسسته فقط؛ `institution_id` يُشتق من JWT claim حصراً ولا يُقبل من العميل أبداً |
| Repudiation | إداري ينكر نشر إعلان مسيء | `author_id` + `created_at` غير قابلين للتعديل + soft delete حصراً |
| Information Disclosure — **الأخطر (أ)** | مستخدم من مؤسسة B يصل إعلانات/إشعارات مؤسسة A (push أو فيد أو Realtime) | topics مؤسسية + حارس `inst_` + RLS تبدأ بـ `institution_id = my_institution()` + اختبار تكاملي إلزامي (7.4) |
| Information Disclosure — **الأخطر (ب)** | طالب يستعلم `announcement_recipients` ليعرف زملاءه المتأخرين مالياً | RLS: `user_id = auth.uid()` حصراً؛ نص الإشعار الفئوي لا يذكر اسم الفئة؛ `recipients_count` للإداري فقط |
| Denial of Service | سيل طلبات mark_read أو تسجيل توكنات وهمية | Rate limits لكل endpoint + حد 200 عنصر/دفعة + حد 10 أجهزة/مستخدم |
| Elevation of Privilege | محاضر يستدعي دالة النشر مباشرة | التحقق من الدور والمؤسسة داخل الدالة من JWT — لا ثقة بأي شيء يرسله العميل |

## 6.4 — قائمة التدقيق الأمنية

- [x] **المصادقة والتفويض:** Supabase Auth (Google) + RLS على كل الجداول الخمسة؛ التحقق من الدور والمؤسسة في DB لا في العميل
- [x] **العزل المؤسسي:** `institution_id` من JWT claim حصراً؛ مواضيع FCM مؤسسية؛ حارس `inst_` في dispatch؛ اختبار pgTAP عدائي لكل سياسة: دور × عملية × **مؤسسة**
- [x] **التشفير:** in-transit عبر TLS 1.2+؛ at-rest عبر AES-256 على بنية Supabase
- [x] **الأسرار:** مفتاح خدمة FCM في Supabase Secrets — لا يلمسه العميل ولا المستودع أبداً
- [x] **الحقن:** parameterized queries حصراً؛ لا SQL ديناميكي في أي trigger
- [x] **سجلات التدقيق:** يُسجَّل: نشر/سحب/تثبيت الإعلانات. **لا يُسجَّل:** محتوى FCM payload في اللوجات (PII)، ولا قوائم أعضاء الفئات خارج `announcement_recipients` المحمي

## 6.5 — الملاحظة والمراقبة (Observability)

| المقياس | التعريف | عتبة التنبيه |
|---|---|---|
| `push_lag_p95` | الفارق بين `created_at` و`push_sent_at` | > 90 ثانية لمدة 5 دقائق → 🔴 |
| `push_success_rate` | نجاح FCM ÷ توكنات is_valid المستهدفة (نافذة ساعة) | < 90% → 🟡؛ < 75% → 🔴 |
| `pending_dispatch_count` | صفوف `push_sent_at IS NULL` أقدم من دقيقتين | > 100 → 🔴 |
| `cron_heartbeat_gap` | آخر نبضة pg_cron | > 5 دقائق → 🔴 |
| `feed_query_p95` | زمن استعلام `my_feed` | > 400ms لمدة 10 دقائق → 🟡 |
| `realtime_connections_total` | إجمالي اتصالات Realtime عبر كل المؤسسات | > 80% من حصة الخطة → 🟡 (مؤشر رفع الحصة — NT-05) |
| `non_inst_topic_rejections` | محاولات dispatch مرفوضة من حارس `inst_` | > 0 → 🔴 فوري (regression أمني) |

---

# القسم 7: خطة التنفيذ وخارطة الطريق 🗺️

## 7.1 — ترتيب المخاطر (Risk-First Ordering)

**أخطر افتراض تقني:** أن سلسلة «trigger → Edge Function → FCM Topic مؤسسي» توصل إعلاناً لآلاف الأجهزة خلال ≤ 10 ثوانٍ p95، وأن Push يصل فعلاً على أجهزة Android منخفضة التكلفة مع أوضاع توفير الطاقة العدوانية. إن فشل هذا، ينقلب التصميم من «push أولاً» إلى «polling أولاً». لذلك هو موضوع المرحلة 0.

## 7.2 — المراحل

### المرحلة 0: إثبات الجدوى (Spike) — 3 أيام
- **الهدف:** التحقق من زمن ونسبة تسليم المسار الكامل trigger → dispatch → FCM Topic مؤسسي
- **المخرج القابل للاختبار:** سكربت يُدرج إعلاناً ويقيس وصوله إلى 5 أجهزة Android حقيقية (بينها جهاز اقتصادي بوضع توفير طاقة) + 2,000 توكن محاكى لمؤسسة واحدة — **مع التحقق أن توكنات «مؤسسة ثانية» محاكاة لا يصلها شيء**
- **معيار النجاح/الفشل:** p95 ≤ 10 ثوانٍ ونسبة وصول ≥ 90% + صفر تسريب عبر المؤسسات. **إن فشل التسليم:** Push «أفضل جهد» + polling خلفي كل 15 دقيقة (WorkManager) وتحديث معايير 1.3 رسمياً

### المرحلة 1: النواة — Schema + RLS + مركز الإشعارات (قراءة) — أسبوع
- **المهام:** (1) migrations (امتداد `announcements` + الجداول الأربعة والفهارس المؤسسية) (2) سياسات RLS + اختباراتها العدائية بنمط **دور × عملية × مؤسسة** (3) دالة `my_feed` مع الفلاتر والـ cursor (4) شاشة مركز الإشعارات في Flutter مع Hive (5) `mark_read` + الطابور المحلي الأوفلاين
- **الاعتماديات:** Core Data v2.1 مطبَّقة (جداول `profiles` / `institutions` / `announcements` الأساسي / دوال `my_institution` / `my_role` / `has_active_subscription`)
- **المخرج القابل للاختبار:** صفوف تُدرج يدوياً تظهر في التطبيق، تُقرأ، وتُفلتر — أونلاين وأوفلاين؛ ومستخدم مؤسسة B لا يرى شيئاً من مؤسسة A
- **اختبارات المرحلة:** pgTAP عدائية — تغطية 100% للسياسات بمصفوفة المؤسسات

### المرحلة 2: النشر الإداري — أسبوع
- **المهام:** (1) دالة النشر مع idempotency واشتقاق `institution_id` من الـ claim (2) واجهة الإداري: إنشاء/تصنيف/تثبيت/سحب (3) الجدولة المؤجلة عبر pg_cron + الـ sweeper (4) تجميد الفئتين لحظة النشر — `late_payers` فوق `has_active_subscription()` حصراً (NT-06)
- **الاعتماديات:** المرحلة 1
- **المخرج القابل للاختبار:** إداري ينشر إعلاناً عاجلاً مثبّتاً مجدولاً بعد 5 دقائق → يظهر لمستخدمي مؤسسته فقط في موعده ± 60 ثا
- **اختبارات المرحلة:** integration للجدولة + idempotency النشر + تجميد الفئات + عزل المؤسسة

### المرحلة 3: قنوات التسليم الحية — Push + Realtime — أسبوع
- **المهام:** (1) device_tokens بدورة حياتها الكاملة (2) دالة dispatch بدفعات 500 + timeout + **حارس `inst_`** (3) اشتراكات Topics المؤسسية المشتقة من الـ claim (4) Realtime مع fallback إلى polling (5) deep-linking من نقرة الإشعار عبر `ref_type/ref_id`
- **الاعتماديات:** المرحلتان 0 و2
- **المخرج القابل للاختبار:** إعلان يُنشر → push يصل خلال ≤ 10 ثا لأجهزة المؤسسة المتصلة فقط، والنقر يفتح الشاشة الصحيحة
- **اختبارات المرحلة:** e2e على أجهزة حقيقية + اختبار توكن ميت + اختبار جهاز مشترك (بما فيه عبر مؤسستين) + اختبار وحدة على حارس `inst_`

### المرحلة 4: ربط الأنظمة المصدرة — أسبوع (تتوزع مع تسليم كل نظام مصدر)
- **المهام:** trigger لكل نوع في قاموس 4.5 مع نمط `EXCEPTION` الحامي + مهمة تذكير الـ 30 دقيقة (بفلتر `enrollments.status='ACTIVE'` — NT-07) + مهمة الأرشفة (180 يوماً / حذف توكنات ميتة 30 يوماً)
- **الاعتماديات:** المرحلة 3 + جاهزية جداول كل نظام مصدر
- **المخرج القابل للاختبار:** رفض دفعة تجريبية → إشعار يصل؛ حدث جدولة بعد 30 دق → تذكير يصل مرة واحدة فقط للطلاب النشطين فقط
- **اختبارات المرحلة:** integration لكل trigger + «فشل الإشعار لا يُفشل المصدر» + dedup + فلتر ACTIVE

### المرحلة 5: التقسية والمراقبة — 3 أيام
- **المهام:** المقاييس السبعة (6.5) بعتباتها + rate limits + مراجعة قائمة 6.4 + اختبار حمل: بث متزامن لمؤسستين (2 × 2,000 توكن محاكى) وقياس p95 والعزل
- **المخرج القابل للاختبار:** لوحة مقاييس حية + تقرير اختبار الحمل مطابقاً لمعايير 1.3

## 7.3 — مخطط الاعتماديات

\`\`\`mermaid
graph LR
    P0[مرحلة 0: Spike تسليم FCM مؤسسي<br/>3 أيام] --> P3
    P1[مرحلة 1: النواة + RLS المؤسسي + المركز<br/>أسبوع] --> P2[مرحلة 2: النشر الإداري<br/>أسبوع]
    P2 --> P3[مرحلة 3: Push + Realtime<br/>أسبوع]
    P3 --> P4[مرحلة 4: ربط الأنظمة المصدرة<br/>أسبوع موزّع]
    P3 --> P5[مرحلة 5: التقسية والمراقبة<br/>3 أيام]
    P4 --> P5
\`\`\`

> المرحلتان 0 و1 تعملان بالتوازي — إجمالي المسار الحرج: **~4.5 أسابيع**.

## 7.4 — تعريف «الانتهاء» (Definition of Done)

- [ ] كل الاختبارات تمر — تغطية ≥ 85% للمسارات الحرجة (RLS، dispatch، dedup، الجدولة)
- [ ] اختبارات RLS العدائية تمر بنسبة 100% بنمط **دور × عملية × مؤسسة** (Core Data §1.3)
- [ ] **اختبار تكاملي إلزامي: إعلان من مؤسسة A لا يصل push ولا Realtime ولا فيداً لأي مستخدم في مؤسسة B** (من تعريف انتهاء التدقيق)
- [ ] اختبار وحدة دائم: dispatch يرفض أي topic لا يبدأ بـ `inst_`
- [ ] توثيق الـ API (القسم 5) مطابق للتنفيذ الفعلي
- [ ] المقاييس تعمل والتنبيهات مضبوطة على عتبات 6.5
- [ ] قائمة الأمان 6.4 مراجعة بالكامل مع توقيع المراجع
- [ ] اختبار الحمل: بث مؤسستين متزامنتين يحقق معايير 1.3
- [ ] خطة rollback: migrations قابلة للعكس + kill switch لتعطيل dispatch بمتغير بيئة دون إيقاف بقية المنصة

---

*— نهاية الوثيقة — Notifications & Announcements TDD v1.1 —*