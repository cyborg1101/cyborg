# وثيقة التصميم الفني: نظام تسجيل الحضور
# Technical Design Document — Attendance System

**الحالة:** مسودة 📝
**التاريخ:** مايو 2026
**المسؤول:** رامز
**الوحدة:** Attendance Module
**الإصدارة:** /api/v1/

---

## 1. وصف الوظيفة ونظرة عامة (Overview)

نظام حضور رقمي يحل محل الورق الذي يُضيع وقت الطلاب والمحاضرين في طوابير يومية. مبني على أربع طبقات متكاملة تضمن التحقق من الوجود الفيزيائي داخل القاعة مع أبسط تجربة ممكنة للمحاضر والطالب على حد سواء.

### المشكلة
جمع الحضور الورقي يستهلك 5-10 دقائق من كل محاضرة، ويسبب زحاماً في نهاية الجلسة، والبيانات تضيع أو تتأخر في الإدخال.

### الحل
المحاضر يضغط زراً واحداً — التطبيق يتكفل بالباقي. الطالب يضغط زراً واحداً — حضوره مسجل في ثوانٍ. أربع طبقات أمان تعمل في الخلفية بشكل صامت دون أن يراها المستخدم.

### الأهداف
- تسجيل حضور 60 طالب في أقل من 60 ثانية
- صفر إعداد مطلوب من المحاضر (ما عدا خطوة واحدة عند التسجيل الأول)
- جعل الغش أصعب من حضور المحاضرة الفعلي
- العمل في بيئة إنترنت ضعيف

---

## 2. الطبقات الأربع للنظام (Four-Layer Architecture)

### الطبقة الأولى — BLE Beacon (الأساسي)
التحقق من الوجود الفيزيائي عبر البلوتوث منخفض الطاقة. نطاق 10-15 متر داخل المباني — مثالي لحجم القاعة الدراسية. يغطي 95% من حالات التسجيل.

### الطبقة الثانية — QR Code (الاحتياطي)
للطلاب الذين فشل BLE معهم فقط. المحاضر يعرض QR من هاتفه، الطالب يمسحه. يتجدد كل 30 ثانية لمنع المشاركة.

### الطبقة الثالثة — GPS الحرم (صامتة)
لا يراها المستخدم. تتحقق فقط أن الطالب داخل نطاق 200 متر من الحرم الجامعي. تمنع تشغيل Hotspot مزيف من خارج الجامعة.

### الطبقة الرابعة — Session Security (دائمة)
UUID فريد لكل جلسة، انتهاء تلقائي بالوقت، تسجيل مرة واحدة فقط. كل المنطق في Edge Function — لا ثقة بالـ Client.

---

## 3. تحليل الموارد (Resource Analysis)

### جانب الطالب (Client-Side)

| المورد | الاستهلاك | التفسير |
|---|---|---|
| CPU | 3-5% أثناء المسح | BLE scan يستمر 10 ثوانٍ فقط |
| RAM | 5-10MB | عملية خفيفة جداً |
| البطارية | منخفض | BLE أقل استهلاكاً من WiFi بـ 90% |
| الإنترنت | < 2KB | إرسال UUID + GPS فقط |

### جانب الخادم (Server-Side)

| المورد | الاستهلاك | التفسير |
|---|---|---|
| CPU | 0.05 vCPU (ذروة) | Edge Function تعمل لثوانٍ فقط |
| RAM | < 1MB | جلسة واحدة = بيانات بسيطة |
| DB | 60 صف لكل محاضرة | صغير جداً |
| الذروة | 60 طلب متزامن | كل الطلاب يسجلون معاً |

---

## 4. تدفق البيانات الكامل (Complete Data Flow)

### السيناريو الأساسي — BLE (Happy Path)

```
١. المحاضر يضغط "فتح الحضور"
        ↓
٢. Flutter يرسل: POST /api/v1/attendance/sessions
   { lecture_id, lecturer_id }
        ↓
٣. Supabase يولّد:
   - session_id (UUID)
   - ble_uuid: "ATT-7X9K2M-2026" (عشوائي لهذه الجلسة)
   - closes_at: now() + 15 دقيقة
   - يحفظ في attendance_sessions
        ↓
٤. تطبيق المحاضر يستقبل الـ ble_uuid
   يبدأ BLE Advertising تلقائياً
   يبث: "ATT-7X9K2M-2026" كل ثانية
        ↓
٥. الطالب يضغط "تسجيل حضور"
        ↓
٦. Flutter يفعل في نفس الوقت:
   - BLE Scan (10 ثوانٍ)
   - GPS reading (في الخلفية)
        ↓
٧. BLE Scan يجد: "ATT-7X9K2M-2026" ✅
        ↓
٨. Flutter يرسل: POST /api/v1/attendance/record
   {
     session_id,
     ble_uuid: "ATT-7X9K2M-2026",
     gps: { lat, lng },
     method: "BLE"
   }
        ↓
٩. Edge Function تتحقق:
   ✅ session موجودة وnشطة؟
   ✅ ble_uuid مطابق للجلسة؟
   ✅ الطالب مسجل في هذه المادة؟
   ✅ لم يسجل مسبقاً؟ (unique constraint)
   ✅ GPS داخل نطاق الحرم؟ (200 متر)
        ↓
١٠. كل الشروط ✅ → INSERT في attendance_records
    status: PRESENT, method: BLE
        ↓
١١. تطبيق الطالب: "✅ تم تسجيل حضورك"
    تطبيق المحاضر يتحدث: عداد +1 (via Realtime)
```

### السيناريو الاحتياطي — QR Code

```
١. الطالب يضغط "مشكلة في التسجيل؟"
        ↓
٢. المحاضر يضغط "عرض QR"
   Flutter يطلب QR جديد من Supabase
   يتجدد كل 30 ثانية تلقائياً
        ↓
٣. الطالب يمسح QR بكاميرا التطبيق
        ↓
٤. Flutter يرسل: POST /api/v1/attendance/record
   {
     session_id,
     qr_token,
     gps: { lat, lng },
     method: "QR"
   }
        ↓
٥. Edge Function تتحقق من نفس الشروط
   + تتحقق أن qr_token لم تنتهِ صلاحيته (30 ثانية)
        ↓
٦. ✅ PRESENT مع method: QR
```

### السيناريو — انتهاء الجلسة

```
المحاضر يضغط "إغلاق الحضور"
أو closes_at يصل تلقائياً (pg_cron)
        ↓
attendance_sessions: is_active = false
BLE Advertising يتوقف تلقائياً في Flutter
        ↓
أي محاولة تسجيل بعدها:
Edge Function ترفض: "الجلسة منتهية"
```

---

## 5. هندسة البيانات (Data Schema)

### الجداول

```sql
-- جدول الجلسات
CREATE TABLE attendance_sessions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lecture_id    UUID NOT NULL REFERENCES lectures(id) ON DELETE CASCADE,
  opened_by     UUID NOT NULL REFERENCES profiles(id),
  ble_uuid      TEXT NOT NULL,          -- "ATT-7X9K2M-2026"
  qr_token_hash TEXT,                   -- هاش الـ QR الحالي (يتجدد كل 30 ثانية)
  qr_expires_at TIMESTAMPTZ,            -- وقت انتهاء الـ QR الحالي
  opens_at      TIMESTAMPTZ DEFAULT now(),
  closes_at     TIMESTAMPTZ NOT NULL,   -- opens_at + 15 دقيقة
  is_active     BOOLEAN DEFAULT true,
  created_at    TIMESTAMPTZ DEFAULT now()
);

-- جدول سجلات الحضور
CREATE TABLE attendance_records (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id           UUID NOT NULL REFERENCES attendance_sessions(id),
  student_id           UUID NOT NULL REFERENCES profiles(id),
  status               TEXT NOT NULL CHECK (status IN ('PRESENT', 'ABSENT', 'PENDING')),
  method               TEXT NOT NULL CHECK (method IN ('BLE', 'QR', 'MANUAL')),
  gps_lat              FLOAT,
  gps_lng              FLOAT,
  gps_accuracy_meters  FLOAT,
  submitted_at         TIMESTAMPTZ DEFAULT now(),

  -- منع التسجيل المكرر
  UNIQUE (session_id, student_id)
);
```

### الفهارس (Indexes)

```sql
-- جلب جلسات محاضرة معينة
CREATE INDEX idx_sessions_lecture
  ON attendance_sessions(lecture_id, is_active);

-- جلب حضور طالب معين
CREATE INDEX idx_records_student
  ON attendance_records(student_id, submitted_at DESC);

-- جلب حضور جلسة معينة (لتقرير المحاضر)
CREATE INDEX idx_records_session
  ON attendance_records(session_id);
```

### إغلاق تلقائي بـ pg_cron

```sql
-- كل دقيقة: أغلق الجلسات المنتهية
SELECT cron.schedule(
  'close-expired-sessions',
  '* * * * *',
  $$
    UPDATE attendance_sessions
    SET is_active = false
    WHERE is_active = true
    AND closes_at < now();
  $$
);
```

---

## 6. سياسات الأمان — RLS

```sql
-- الطالب يسجل حضوره فقط في مواد مسجل فيها
CREATE POLICY "student_insert_attendance"
ON attendance_records FOR INSERT
WITH CHECK (
  -- الطالب هو من يسجل
  student_id = auth.uid()
  AND
  -- الجلسة نشطة
  session_id IN (
    SELECT id FROM attendance_sessions
    WHERE is_active = true
    AND closes_at > now()
  )
  AND
  -- الطالب مسجل في المادة
  session_id IN (
    SELECT s.id
    FROM attendance_sessions s
    JOIN lectures l ON s.lecture_id = l.id
    JOIN enrollments e ON l.course_id = e.course_id
    WHERE e.student_id = auth.uid()
    AND e.status = 'ACTIVE'
  )
);

-- الطالب يرى سجل حضوره فقط
CREATE POLICY "student_read_own_attendance"
ON attendance_records FOR SELECT
USING (student_id = auth.uid());

-- المحاضر يرى حضور طلاب موادّه فقط
CREATE POLICY "lecturer_read_attendance"
ON attendance_records FOR SELECT
USING (
  session_id IN (
    SELECT s.id
    FROM attendance_sessions s
    JOIN lectures l ON s.lecture_id = l.id
    JOIN courses c ON l.course_id = c.id
    WHERE c.lecturer_id = auth.uid()
  )
);

-- المحاضر يفتح ويغلق جلسات موادّه فقط
CREATE POLICY "lecturer_manage_sessions"
ON attendance_sessions FOR ALL
USING (
  opened_by = auth.uid()
  AND lecture_id IN (
    SELECT l.id FROM lectures l
    JOIN courses c ON l.course_id = c.id
    WHERE c.lecturer_id = auth.uid()
  )
);
```

---

## 7. Edge Function — منطق التحقق

```typescript
// supabase/functions/record-attendance/index.ts

import { serve } from "https://deno.land/std/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js";

const CAMPUS_CENTER = {
  lat: 15.5527,  // إحداثيات الحرم الجامعي — تُضبط للجامعة الفعلية
  lng: 32.5324
};
const CAMPUS_RADIUS_METERS = 200;

serve(async (req) => {
  const { session_id, ble_uuid, qr_token, gps, method } =
    await req.json();

  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  );

  // ١. التحقق من الجلسة
  const { data: session } = await supabase
    .from("attendance_sessions")
    .select("*")
    .eq("id", session_id)
    .eq("is_active", true)
    .gt("closes_at", new Date().toISOString())
    .single();

  if (!session) {
    return Response.json(
      { error: "الجلسة غير موجودة أو منتهية" },
      { status: 400 }
    );
  }

  // ٢. التحقق حسب الطريقة
  if (method === "BLE") {
    if (ble_uuid !== session.ble_uuid) {
      return Response.json(
        { error: "إشارة البلوتوث غير صحيحة" },
        { status: 400 }
      );
    }
  }

  if (method === "QR") {
    const qrValid =
      hashToken(qr_token) === session.qr_token_hash &&
      new Date() < new Date(session.qr_expires_at);

    if (!qrValid) {
      return Response.json(
        { error: "رمز QR منتهي أو غير صحيح" },
        { status: 400 }
      );
    }
  }

  // ٣. التحقق من GPS (الحرم الجامعي فقط)
  if (gps) {
    const distance = calculateDistance(
      gps.lat, gps.lng,
      CAMPUS_CENTER.lat, CAMPUS_CENTER.lng
    );

    if (distance > CAMPUS_RADIUS_METERS) {
      return Response.json(
        { error: "أنت خارج الحرم الجامعي" },
        { status: 400 }
      );
    }
  }

  // ٤. تسجيل الحضور
  const { error } = await supabase
    .from("attendance_records")
    .insert({
      session_id,
      student_id: req.headers.get("x-user-id"),
      status: "PRESENT",
      method,
      gps_lat: gps?.lat,
      gps_lng: gps?.lng,
      gps_accuracy_meters: gps?.accuracy
    });

  // unique constraint يمنع التسجيل المكرر تلقائياً
  if (error?.code === "23505") {
    return Response.json(
      { error: "تم تسجيل حضورك مسبقاً" },
      { status: 409 }
    );
  }

  return Response.json({ status: "PRESENT" });
});

// حساب المسافة بين نقطتين (Haversine Formula)
function calculateDistance(
  lat1: number, lng1: number,
  lat2: number, lng2: number
): number {
  const R = 6371000; // نصف قطر الأرض بالمتر
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLng = (lng2 - lng1) * Math.PI / 180;
  const a =
    Math.sin(dLat/2) * Math.sin(dLat/2) +
    Math.cos(lat1 * Math.PI / 180) *
    Math.cos(lat2 * Math.PI / 180) *
    Math.sin(dLng/2) * Math.sin(dLng/2);
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
}
```

---

## 8. كود Flutter — جانب الطالب

```dart
class AttendanceService {

  // ١. تسجيل الحضور عبر BLE
  Future<AttendanceResult> recordWithBLE(String sessionId) async {

    // مسح BLE لمدة 10 ثوانٍ
    final devices = await FlutterBluePlus.startScan(
      timeout: Duration(seconds: 10)
    );

    // البحث عن UUID الجلسة
    final sessionDevice = devices.firstWhereOrNull(
      (d) => d.advertisementData.localName
               .startsWith("ATT-")
    );

    if (sessionDevice == null) {
      return AttendanceResult.bleNotFound;
    }

    final bleUuid = sessionDevice.advertisementData.localName;

    // قراءة GPS في الخلفية
    final position = await Geolocator.getCurrentPosition(
      desiredAccuracy: LocationAccuracy.medium
    );

    // إرسال للـ Edge Function
    final response = await supabase.functions.invoke(
      'record-attendance',
      body: {
        'session_id': sessionId,
        'ble_uuid': bleUuid,
        'method': 'BLE',
        'gps': {
          'lat': position.latitude,
          'lng': position.longitude,
          'accuracy': position.accuracy
        }
      }
    );

    if (response.status == 200) return AttendanceResult.success;
    return AttendanceResult.failed;
  }

  // ٢. تسجيل الحضور عبر QR (الاحتياطي)
  Future<AttendanceResult> recordWithQR(
    String sessionId,
    String qrToken
  ) async {

    final position = await Geolocator.getCurrentPosition(
      desiredAccuracy: LocationAccuracy.medium
    );

    final response = await supabase.functions.invoke(
      'record-attendance',
      body: {
        'session_id': sessionId,
        'qr_token': qrToken,
        'method': 'QR',
        'gps': {
          'lat': position.latitude,
          'lng': position.longitude,
          'accuracy': position.accuracy
        }
      }
    );

    if (response.status == 200) return AttendanceResult.success;
    return AttendanceResult.failed;
  }
}
```

---

## 9. كود Flutter — جانب المحاضر

```dart
class LecturerAttendanceService {

  // فتح جلسة الحضور
  Future<AttendanceSession> openSession(String lectureId) async {

    // إنشاء الجلسة في Supabase
    final response = await supabase
      .from('attendance_sessions')
      .insert({
        'lecture_id': lectureId,
        'opened_by': supabase.auth.currentUser!.id,
        'closes_at': DateTime.now()
          .add(Duration(minutes: 15))
          .toIso8601String()
      })
      .select()
      .single();

    final session = AttendanceSession.fromJson(response);

    // بدء BLE Advertising تلقائياً
    await FlutterBluePlus.startAdvertising(
      AdvertiseData(localName: session.bleUuid)
    );

    // مراقبة الحضور في الوقت الفعلي
    _listenToAttendance(session.id);

    return session;
  }

  // مراقبة لحظية عبر Supabase Realtime
  void _listenToAttendance(String sessionId) {
    supabase
      .from('attendance_records')
      .stream(primaryKey: ['id'])
      .eq('session_id', sessionId)
      .listen((records) {
        // تحديث عداد الحاضرين في الواجهة
        presentCount.value = records
          .where((r) => r['status'] == 'PRESENT')
          .length;
      });
  }

  // إغلاق الجلسة
  Future<void> closeSession(String sessionId) async {
    await supabase
      .from('attendance_sessions')
      .update({'is_active': false})
      .eq('id', sessionId);

    // إيقاف BLE
    await FlutterBluePlus.stopAdvertising();
  }
}
```

---

## 10. واجهات المستخدم (UI States)

### شاشة المحاضر — حالات الجلسة

```
┌─────────────────────────────────────────┐
│  حضور: مقدمة في البرمجة              │
│  الجلسة مفتوحة — 12:43 دقيقة متبقية  │
│                                         │
│         32 / 60 طالب              │
│         ████████░░░░  53%               │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ أحمد محمد          ✅ BLE       │   │
│  │ فاطمة علي          ✅ QR        │   │
│  │ محمد الأمين        ⏳ لم يسجل   │   │
│  └─────────────────────────────────┘   │
│                                         │
│  [عرض QR للمشكلات]  [إغلاق الجلسة]   │
└─────────────────────────────────────────┘
```

### شاشة الطالب — حالات التسجيل

```
حالة المسح:
┌─────────────────────────┐
│  🔵 جاري البحث...       │
│  يبحث عن إشارة المحاضر │
│  ████░░░░░░  40%        │
└─────────────────────────┘

حالة النجاح:
┌─────────────────────────┐
│  ✅ تم تسجيل حضورك     │
│  مقدمة في البرمجة      │
│  الساعة 10:23 صباحاً   │
└─────────────────────────┘

حالة الفشل:
┌─────────────────────────┐
│  ❌ لم يتم التسجيل     │
│  تأكد من وجودك داخل    │
│  القاعة                 │
│                          │
│  [مسح QR بدلاً منه]    │
└─────────────────────────┘
```

---

## 11. حالات الحافة (Edge Cases)

| الحالة | السلوك المتوقع |
|---|---|
| الطالب يضغط مرتين | unique constraint يمنعه — "تم تسجيل حضورك مسبقاً" |
| الجلسة منتهية | Edge Function ترفض — "انتهى وقت التسجيل" |
| BLE غير متاح في الهاتف | التطبيق يعرض QR مباشرة |
| GPS غير متاح | التسجيل يكمل بدون GPS مع تسجيل ملاحظة |
| الطالب خارج الحرم | Edge Function ترفض — "أنت خارج الحرم الجامعي" |
| QR منتهي الصلاحية | "انتهت صلاحية الرمز — اطلب من المحاضر تجديده" |
| انقطع الإنترنت أثناء التسجيل | يُعاد المحاولة تلقائياً 3 مرات عند عودة الاتصال |
| المحاضر يفتح جلستين بالخطأ | الثانية تلغي الأولى تلقائياً |

---

## 12. مؤشرات الأداء (Performance Targets)

| العملية | الهدف | الطريقة |
|---|---|---|
| فتح جلسة (المحاضر) | < 500ms | INSERT بسيط |
| BLE Scan | 5-10 ثوانٍ | طبيعي للـ BLE |
| تسجيل حضور | < 1 ثانية | Edge Function خفيفة |
| تحديث عداد المحاضر | < 200ms | Supabase Realtime |
| QR Scan | < 2 ثانية | Camera + HTTP |

---

## 13. سيناريوهات الاختبار (Test Scenarios)

### اختبارات وظيفية

```
✅ طالب مسجل في المادة يسجل عبر BLE داخل القاعة → PRESENT
✅ طالب يحاول التسجيل مرتين → رسالة "تم التسجيل مسبقاً"
✅ طالب خارج الحرم → رفض GPS
✅ BLE UUID خاطئ → رفض
✅ QR منتهي الصلاحية → رفض
✅ جلسة منتهية → رفض
✅ طالب غير مسجل في المادة → رفض RLS
✅ تسجيل ناجح عبر QR كطريقة احتياطية
✅ إغلاق يدوي من المحاضر → BLE يتوقف + pg_cron
✅ إغلاق تلقائي بعد 15 دقيقة → pg_cron
```

### اختبارات الحمل

```
60 طالب يسجلون في نفس الوقت → كل الطلبات تنجح
لا deadlock في unique constraint
Realtime عداد المحاضر يتحدث لكل 60 طلب
```

---

## 14. الاعتماديات (Dependencies)

| الوحدة | السبب |
|---|---|
| Auth Module | التحقق من هوية الطالب والمحاضر |
| Enrollments | التحقق أن الطالب مسجل في المادة |
| Lectures | ربط الجلسة بمحاضرة محددة |
| Courses | التحقق أن المحاضر يدير هذه المادة |

### Flutter Packages المطلوبة

```yaml
dependencies:
  flutter_blue_plus: ^1.31.0   # BLE Scan + Advertise
  geolocator: ^11.0.0          # GPS
  mobile_scanner: ^5.0.0       # QR Scanner
  supabase_flutter: ^2.0.0     # Supabase Client
```

---

## 15. ملاحظات التنفيذ (Implementation Notes)

**١. إحداثيات الحرم الجامعي**
يجب ضبط `CAMPUS_CENTER` في Edge Function بالإحداثيات الحقيقية للجامعة قبل الإطلاق. الإدارة تُدخل هذه البيانات في Supabase Studio.

**٢. مدة الجلسة**
15 دقيقة افتراضية — قابلة للتغيير من قِبل المحاضر عند فتح الجلسة (5-30 دقيقة).

**٣. بيانات GPS الحساسة**
إحداثيات الطلاب في `attendance_records` تُقرأ من المحاضر والإدارة فقط عبر RLS. لا تظهر للطلاب.

**٤. تقرير الحضور**
بعد إغلاق الجلسة، المحاضر يقدر يصدّر تقرير CSV من التطبيق مباشرة. يُضاف في المرحلة الثانية.

---

*نهاية الوثيقة — Attendance System TDD v1.0*
