## วิเคราะห์ละเอียด: OPTIMIZED ROUTING SCRIPT (PRO VERSION)

สคริปต์นี้เป็น **Google Apps Script** สำหรับคำนวณเส้นทางส่งสินค้าแบบ one-way จากคลัง → จุดส่งหลายจุด โดยใช้ **Google Maps Routes API** แล้วเขียนผลกลับชีตเบิกส่วนต่าง

---

## 1) จุดประสงค์ของระบบ

| รายการ | รายละเอียด |
|--------|------------|
| งานหลัก | หาลำดับจุดส่ง + ระยะทางรวม จากคลังวังน้อย |
| แหล่งพิกัด | ชีต `SCGนครหลวงJWDภูมิภาค` (แถวที่มี Shipment No ตรงกัน) |
| ปลายทางเขียนผล | ชีต `ทำเบิกส่วนต่างScgวังน้อย` |
| API | `routes.googleapis.com/directions/v2:computeRoutes` |
| โหมดจราจร | `TRAFFIC_UNAWARE` (ระยะทางคงที่ ไม่ผันตามรถติด — ใช้เบิกเงิน) |

**Caller ที่คาดไว้:** ระบบภายนอก (เช่น AppSheet / webhook) เรียก  
`findOptimalRouteUsingExistingDistance(shipmentId, rowId)`

---

## 2) CONFIG (ค่าคงที่)

```text
SPREADSHEET_ID        = 1CYtLpXn6gNYgbGu3oRF8CW5KkGYHQJ6D4jl9u2LiR6o
SHEET_COMPUTED        = SCGนครหลวงJWDภูมิภาค
SHEET_RESULT          = ทำเบิกส่วนต่างScgวังน้อย

DEPOT                 = 14.1646106, 100.6254644
                      ชื่อ: คลังสินค้า เอสซีจี เจดับเบิ้ลยูดี วังน้อย

คอลัมน์ผลลัพธ์
  GoogleMapsRoutesAPI              → สตริงพิกัดเรียงลำดับ
  ระยะทาง_GoogleMapAPI_Km          → ระยะรวม (กม.)
  แสดงแผนที่_GoogleMapsRoutesAPI   → ลิงก์ maps/dir
  ระยะทางจากคลัง_Km                → ใช้เรียง “ไกลสุดเป็นปลายทาง”
```

**หมายเหตุ:** API Key อ่านจาก  
`PropertiesService.getScriptProperties().getProperty("GOOGLE_MAPS_API_KEY")`  
— **ไม่มี hard-code key ในไฟล์** (ดี)

---

## 3) ภาพรวมการทำงาน (Data Flow)

```text
Caller ส่ง (shipmentId, rowId)
        │
        ▼
[1] prepareWaypointsWithExistingDistance
        │  TextFinder หาทุกแถวที่ Shipment No = shipmentId
        │  อ่านพิกัด + ชื่อ + ระยะทางจากคลัง
        │  ใส่ Depot เป็นจุดที่ 0 เสมอ
        ▼
[2] selectFinalDestinationAndSort
        │  เรียงจุดส่งตาม ระยะทางจากคลัง มาก→น้อย
        │  จุดไกลสุด = ปลายทางสุดท้าย
        │  จุดอื่น = intermediate
        ▼
[3] executeGoogleMapsRoutesAPIOneWay
        │  ยิง Routes API (DRIVE + TRAFFIC_UNAWARE)
        │  optimizeWaypointOrder ถ้า intermediate > 1
        │  ได้ ระยะรวม + ลำดับที่ Google แนะนำ + ลิงก์แผนที่
        ▼
[4] writeResultsToSheet
        │  TextFinder หาแถวในชีตผลด้วย rowId
        │  เขียนสตริงพิกัด / ระยะทาง / ลิงก์ / LatLong 01–20
        ▼
Return { Status, CalculatedDistanceKm, GoogleMapsLink }
```

---

## 4) ฟังก์ชันหลักทีละตัว

### 4.1 `findOptimalRouteUsingExistingDistance(shipmentId, rowId)`

**ทำอะไร**
1. เปิดสเปรดชีตด้วย `SPREADSHEET_ID` คงที่  
2. เตรียม waypoints  
3. จัดลำดับ  
4. เรียก API  
5. เขียนผลลงชีตผล ตาม `rowId`  
6. คืน object สำเร็จ หรือ throw error

**Error handling**
- `try/catch` → log แล้ว **throw ต่อ** (caller ภายนอกต้องจับเอง)
- ถ้า waypoints ≤ 1 จุด (มีแค่ depot หรือไม่มีพิกัด) → throw

---

### 4.2 `prepareWaypointsWithExistingDistance(ss, shipmentId)`

**วิธีหาแถว**
- อ่าน header แถว 1 ครั้ง
- หา index คอลัมน์:
  - `Shipment No`
  - `จุดส่งสินค้าปลายทาง`
  - `ชื่อปลายทาง`
  - `ระยะทางจากคลัง_Km`
- ใช้ **TextFinder** เฉพาะคอลัมน์ Shipment No  
  `matchEntireCell(true)` → ต้องตรงทั้งเซลล์

**โครงสร้างจุดแต่ละจุด**
```text
{
  id, name,
  original: { lat, lng },          // ใช้สร้างลิงก์ / สตริง
  forApi: { location: { latLng: { latitude, longitude } } },
  distance,                        // จากคอลัมน์ระยะทางจากคลัง
  isDepot
}
```

**Depot**
- ถูก push เป็นจุดแรกเสมอ (`id: 0`, `distance: 0`)

**ทำความสะอาดพิกัด**
```text
trim → ลบทุกอย่างยกเว้น ตัวเลข . , -
ต้องมีเครื่องหมายจุลภาค 1 ตัว
parseFloat ทั้งคู่ ต้องไม่เป็น NaN
```

**จุดอ่อนที่โค้ดทำจริง**
- อ่านทีละเซลล์ด้วย `getRange(rowIdx, col).getValue()` ต่อ match  
  → ถ้า shipment เดียวมีหลายสิบจุด จะช้ากว่า batch `getValues`
- ถ้าคอลัมน์ชื่อ/ระยะทางไม่มี → ใช้ default (`Point N`, distance 0) ไม่ throw
- ถ้า `Shipment No` หรือพิกัดหาย → throw ตั้งแต่ต้น

---

### 4.3 `selectFinalDestinationAndSort(allPoints)`

**Logic จริงในโค้ด**
1. แยก depot ออก  
2. เรียง destinations ตาม `distance` **มาก → น้อย**  
3. **ตัวแรกหลังเรียง = ปลายทางสุดท้าย** (ไกลจากคลังที่สุด)  
4. ที่เหลือ = intermediate  
5. ลำดับส่ง API:  
   `[depot, ...intermediates, finalDestination]`

**ความหมายทางธุรกิจ**
- เส้นทาง **ขาไปอย่างเดียว** (ไม่กลับคลัง)
- ใช้ระยะทางจากคลังที่มีอยู่แล้วเป็น heuristic เลือกปลายทาง
- ลำดับจุดกลางยังให้ Google optimize ได้อีกชั้น (ถ้า > 1 จุด)

**Edge case**
- ถ้าทุกจุด `distance = 0` หรือว่าง → เรียงไม่มีความหมาย ลำดับขึ้นกับลำดับเดิมหลัง sort เสถียรของ JS
- จุดเดียว: intermediate ว่าง, ปลายทาง = จุดนั้น, `optimizeWaypointOrder = false`

---

### 4.4 `executeGoogleMapsRoutesAPIOneWay(allPoints)`

**Payload ที่ส่งจริง**
```json
{
  "origin": depot,
  "destination": finalDestination,
  "intermediates": [...],
  "travelMode": "DRIVE",
  "routingPreference": "TRAFFIC_UNAWARE",
  "optimizeWaypointOrder": true/false   // true เมื่อ intermediate > 1
}
```

**Field mask ที่ขอ**
```text
routes.duration
routes.distanceMeters
routes.optimizedIntermediateWaypointIndex
```

**หลังได้ response**
- ระยะทาง = `distanceMeters / 1000` (กม.)
- ถ้ามี `optimizedIntermediateWaypointIndex` → จัด intermediate ตาม index ที่ Google ส่ง
- สร้างลิงก์:
  ```text
  https://www.google.com/maps/dir/lat,lng/lat,lng/...
  ```
  แล้ว `encodeURI` ทั้งสตริง

**ข้อจำกัดจากโค้ด**
- ใช้เฉพาะ `routes[0]` — ไม่เปรียบเทียบหลาย route
- ไม่ใช้ `duration` แม้ขอมาแล้ว
- ไม่มี retry / backoff เมื่อ API ล้ม
- ไม่ตรวจโควต้า / rate limit
- ถ้า response ไม่มี `routes` → จะ error ตอนอ่าน `routes[0]`

---

### 4.5 `writeResultsToSheet(resultSheet, rowId, ...)`

**หาแถว**
- TextFinder บนคอลัมน์ `ID_ทำเบิกส่วนต่างScgวังน้อย`  
- ต้อง match ทั้งเซลล์กับ `rowId`

**เขียนค่า**

| คอลัมน์ | ค่าที่เขียน |
|---------|-------------|
| `GoogleMapsRoutesAPI` | `"lat, lng \| lat, lng \| ..."` (มีช่องว่างหลังคอมม่า) |
| `ระยะทาง_GoogleMapAPI_Km` | ตัวเลขกม. |
| `แสดงแผนที่_GoogleMapsRoutesAPI` | URL แผนที่ |
| `Lat/Long_ปลายทาง_01` … `_20` | พิกัดลูกค้าเท่านั้น (ตัด depot) ว่างถ้าไม่ครบ 20 |

**การเขียน**
- คอลัมน์หลัก 3 ตัว: `setValue` ทีละเซลล์  
- 01–20: **batch** `setValues` แถวเดียว 20 คอลัมน์

**เหตุผลในคอมเมนต์**
- เขียนทีละเซลล์เพื่อ “ไม่ทับสูตรค่ายื่นเบิก” — สมเหตุสมผลถ้าชีตมีสูตรปน

---

## 5) สิ่งที่โค้ด **ทำจริง** vs สิ่งที่ชื่อฟังก์ชันอาจทำให้เข้าใจผิด

| ชื่อ / ความคาดหวัง | ความจริงจากโค้ด |
|--------------------|------------------|
| “Optimal route” | Optimize เฉพาะ **ลำดับ intermediate** + เลือกปลายทางด้วยระยะจากคลัง — **ไม่ใช่ TSP เต็มรูปแบบรวม depot กลับ** |
| “Using existing distance” | ใช้ `ระยะทางจากคลัง_Km` แค่เรียงเลือกปลายทาง — **ระยะทางรวมจริงมาจาก Routes API** |
| One-way | ถูกต้อง — ไม่มีขา back-to-depot |
| Format แน่นอน | สตริงผลมีช่องว่างหลัง `,` แต่ลิงก์ไม่มีช่องว่าง |

---

## 6) จุดแข็ง (จากโค้ดจริง)

1. **TextFinder เฉพาะคอลัมน์** — ไม่โหลดทั้งชีตมา filter ในหน่วยความจำ  
2. **API Key จาก Script Properties** — ไม่ฝังในซอร์ส  
3. **TRAFFIC_UNAWARE** — ระยะทาง reproducible สำหรับเบิกเงิน  
4. **Batch เขียน 20 คอลัมน์พิกัด**  
5. **กันพิกัดขยะ** ด้วย regex ทำความสะอาด  
6. **รองรับคอลัมน์ชื่อ/ระยะทางหาย** แบบ soft fallback  

---

## 7) ความเสี่ยง / ข้อจำกัด (Evidence-based)

### CRITICAL / HIGH

| หัวข้อ | รายละเอียด |
|--------|------------|
| Hard-coded Spreadsheet ID | สคริปต์ผูกสมุดบัญชีเดียว — ย้ายไฟล์/คัดลอกโปรเจกต์แล้วพัง |
| ไม่มี retry API | network/5xx ครั้งเดียว = งานล้มทั้ง shipment |
| อ่านทีละเซลล์ต่อ match | shipment ที่มีจุดส่งเยอะ → ช้า + เสี่ยง quota อ่านชีต |
| ไม่ล็อกแถว (LockService) | ถ้ารันขนานสองตัวบน rowId เดียวกัน อาจเขียนทับกัน |

### MEDIUM

| หัวข้อ | รายละเอียด |
|--------|------------|
| จำกัด 20 จุดปลายทางตอนเขียน | ถ้า > 20 จุด ข้อมูลเกินถูกตัดเงียบ |
| Routes API จำกัด intermediate | Google จำกัดจำนวน waypoint — โค้ดไม่ได้เช็กก่อนยิง |
| `distance = 0` เมื่อคอลัมน์ว่าง | อาจเลือกปลายทางผิดถ้ายังไม่ได้คำนวณระยะจากคลัง |
| throw หลัง log | ถ้า AppSheet ไม่จับ error ผู้ใช้เห็นแค่ fail ทั่วไป |

### LOW / INFO

| หัวข้อ | รายละเอียด |
|--------|------------|
| ใช้ `duration` ไม่ได้ | ขอ field มาแล้วไม่ใช้ |
| ชื่อชีต/คอลัมน์ภาษาไทย | ต้องตรงทุกตัวอักษร รวมวรรค |
| `encodeURI` ทั้ง URL | ปกติใช้ได้กับ lat,lng แต่ถ้ามีอักขระพิเศษอื่นต้องระวัง |

---

## 8) เงื่อนไขที่ต้องมีถึงจะรันสำเร็จ

1. Script Property: `GOOGLE_MAPS_API_KEY`  
2. Routes API เปิดในโปรเจกต์ Google Cloud ของ key นั้น  
3. ชีต `SCGนครหลวงJWDภูมิภาค` มีคอลัมน์:
   - `Shipment No`
   - `จุดส่งสินค้าปลายทาง` (รูปแบบประมาณ `lat,lng`)
4. ชีต `ทำเบิกส่วนต่างScgวังน้อย` มีคอลัมน์:
   - `ID_ทำเบิกส่วนต่างScgวังน้อย` = ค่า `rowId` ที่ส่งเข้ามา
5. อย่างน้อย 1 จุดส่งที่มีพิกัด valid ภายใต้ shipment นั้น  
6. (แนะนำ) มี `ระยะทางจากคลัง_Km` ครบ — ไม่งั้นการเลือกปลายทางไม่มีความหมาย

---

## 9) ตัวอย่างลำดับผลลัพธ์ (จาก logic)

สมมติ shipment มี 3 จุด:

| จุด | ระยะจากคลัง |
|-----|-------------|
| A | 10 กม. |
| B | 40 กม. |
| C | 25 กม. |

หลัง sort: B (40) → C (25) → A (10)  
- **ปลายทางสุดท้าย** = B  
- intermediate เริ่มต้น = C, A  
- Google อาจสลับ C↔A ได้ถ้า `optimizeWaypointOrder = true`  
- เส้นทางสุดท้ายรูปแบบ: **Depot → (C หรือ A) → (อีกจุด) → B**

สตริงที่เขียน:
```text
14.xxxxxx, 100.xxxxxx | 13.xxxxxx, 100.xxxxxx | ... | 14.xxxxxx, 100.xxxxxx
```

---

## 10) ความสัมพันธ์กับระบบ Phaopanya Master (v5.5.6)

| จุดเชื่อม | สถานะ |
|----------|--------|
| ชีต `SCGนครหลวงJWDภูมิภาค` | ใช้ร่วมกับ Master / Workload / Geo |
| `ระยะทางจากคลัง_Km` | คอลัมน์เดียวกับที่ Geo/Maps ชุดอื่นอาจเติม |
| สคริปต์นี้ | **แยกไฟล์** — ไม่ได้อยู่ใน package v5.5.6 |
| ชื่อฟังก์ชัน | ไม่ชนกับ `runMaster` / `runMasterGeo` / `GOOGLEMAPS_*` โดยตรง |
| ความเสี่ยงรวมโปรเจกต์ | ถ้าใส่ไฟล์นี้ใน Apps Script เดียว ต้องระวัง `SPREADSHEET_ID` แข็ง + key คนละตัวกับ Maps service อื่น |

---

## 11) สรุปสั้น

**ระบบนี้ทำอะไร**  
รับรหัส shipment + รหัสแถวเบิก → ดึงทุกจุดส่งของ shipment จากชีตคำนวณ → ตั้งคลังเป็นต้นทาง → เลือกจุดไกลสุดเป็นปลายทาง → ให้ Google จัดลำดับจุดกลาง → ได้ระยะทางรวมแบบไม่คิดรถติด → เขียนกลับชีตเบิก

**จุดที่ออกแบบมาดี**  
TextFinder, TRAFFIC_UNAWARE เพื่อเบิกเงิน, แยก key ออกจากโค้ด, batch เขียนพิกัด 20 ช่อง

**จุดที่ต้องรู้ก่อนใช้จริง**  
ผูก spreadsheet เดียว, ไม่มี retry/lock, อ่านทีละเซลล์, ตัดที่ 20 จุด, และลำดับ “optimal” ขึ้นกับทั้ง heuristic ระยะจากคลัง + Google optimize เฉพาะจุดกลาง

ถ้าต้องการขั้นถัดไป สามารถไล่เป็น:
1. แผนทดสอบเคสขอบ (1 จุด / 2 จุด / >20 จุด / พิกัดเสีย)  
2. เทียบว่าจะรวมเข้า package v5.5.6 แบบ namespace หรือแยกโปรเจกต์  
3. ชี้จุดที่ควรแก้ก่อน production (retry, batch read, lock)

## สรุปตรวจโค้ด Routing PRO — ประสิทธิภาพ / Error เงียบ / ซ้ำซ้อน / สิ่งที่ควรปรับ / API

ตรวจจากโค้ดจริงทั้งไฟล์ + เทียบแนวทาง Google Routes API ปัจจุบัน (computeRoutes ยังเป็น API ที่แนะนำ ไม่ใช่ Directions API เก่า)

---

## 1) ประสิทธิภาพโดยรวม

| ส่วน | ระดับ | เหตุผลจากโค้ด |
|------|--------|----------------|
| หาแถว Shipment | **ดี** | TextFinder เฉพาะคอลัมน์ + `matchEntireCell` |
| อ่านพิกัด/ชื่อ/ระยะทาง | **อ่อน** | หลังหา match แล้ว `getValue()` **ทีละเซลล์ 2–3 ครั้งต่อจุด** |
| เรียงลำดับปลายทาง | **เบา** | sort ในหน่วยความจำ ไม่หนัก |
| เรียก Routes API | **เหมาะสม** | ยิงครั้งเดียวต่อ shipment + field mask ไม่กว้างเกิน |
| เขียนผล | **ปานกลาง** | คอลัมน์หลัก 3 ช่อง `setValue` ทีละเซลล์ / พิกัด 01–20 batch ดี |
| Concurrent | **ไม่รองรับ** | ไม่มี `LockService` |

**ประมาณ I/O ชีตเมื่อ shipment มี N จุด**

```text
TextFinder 1 ครั้ง
+ getValue × (1 ถึง 3) × N     ← จุดช้าหลัก
+ เขียน 3–4 ครั้ง + batch 1 ครั้ง
```

ถ้า N = 5 ยังโอเค  
ถ้า N = 30–50 ต่อ shipment และรันถี่ → จะช้าและกิน quota อ่านชีตโดยไม่จำเป็น

**สรุปประสิทธิภาพ:** โครงสร้างดีสำหรับ shipment เล็ก–กลาง แต่ **จุดอ่านทีละเซลล์ทำให้ไม่ใช่ “จรวด” จริง** ตามที่คอมเมนต์บอก

---

## 2) Error เงียบ (Silent / Soft-fail)

### 2.1 มี throw ชัดเจน (ไม่เงียบ)

- ไม่เจอ waypoint valid
- ขาดคอลัมน์ Shipment / พิกัด
- ไม่มี API Key
- API ตอบ ≠ 200
- ไม่เจอ `rowId` ในชีตผล
- ไม่มีคอลัมน์ ID บนชีตผล

### 2.2 เงียบหรือกลืนข้อมูล (สำคัญ)

| จุด | พฤติกรรมจริง | ผลกระทบ |
|-----|--------------|----------|
| พิกัดว่าง / parse ไม่ได้ | `return` ข้ามจุดนั้น | shipment อาจเหลือจุดน้อยลงโดยไม่มี log ว่าข้ามกี่จุด |
| คอลัมน์ผลบางตัวไม่มี | ข้าม `setValue` | ดูเหมือนสำเร็จ แต่คอลัมน์นั้นว่าง |
| คอลัมน์ `ชื่อปลายทาง` / `ระยะทางจากคลัง_Km` ไม่มี | ใช้ชื่อ `Point N` และ distance = 0 | ลำดับปลายทางอาจผิดโดยไม่เตือน |
| จุดส่ง > 20 | ตัดเงียบที่ช่อง 20 | ข้อมูลหายโดยไม่ error |
| `result.routes` ว่าง | จะพังตอน `routes[0]` (มี throw ทางอ้อม) | ข้อความ error อาจอ่านยาก |
| `optimizedIntermediateWaypointIndex` ไม่ครบ | ใช้ลำดับเดิม | ไม่ fail แต่ลำดับอาจไม่ optimal |
| ระยะทางจากคลังเป็นข้อความ/ว่าง | `parseFloat(...) \|\| 0` | กลายเป็น 0 เงียบ → เรียงผิด |

**สรุป:** ไม่ใช่สคริปต์ที่ “กลืนทุก error” แต่มี **soft-skip ที่ทำให้ผลดูสำเร็จทั้งที่ข้อมูลไม่ครบ** โดยเฉพาะพิกัดเสียและจุดเกิน 20

---

## 3) ฟังก์ชันซ้ำซ้อน / โครงสร้าง

| รายการ | สถานะ |
|--------|--------|
| ฟังก์ชันหลัก 5 ตัว | **ไม่ซ้ำ** แยกหน้าที่ชัด |
| หา header + index คอลัมน์ | ทำ 2 ที่ (computed / result) คล้ายกัน แต่ยังไม่ถึงขั้น duplicate อันตราย |
| ทำความสะอาด lat,lng | มีที่เดียว — ดี |
| ไม่มี helper ร่วมสำหรับ “หาแถวด้วย TextFinder” | เขียนสองรอบ (shipment / rowId) |

**สรุป:** ไม่มี dead function / ฟังก์ชันซ้ำทำงานสองทาง  
มีแค่ **pattern ซ้ำ** เรื่องอ่าน header + TextFinder ที่รวมเป็น helper ได้

---

## 4) สิ่งที่ควรปรับปรุง (เรียงความสำคัญ)

### ควรทำก่อนใช้หนัก (HIGH)

1. **Batch อ่านแถวที่ match**  
   หลัง `findAll()` เก็บ row index → อ่านช่วงคอลัมน์ที่ต้องใช้ทีเดียวด้วย `getValues()`  
   ลด `getValue` ต่อจุด

2. **นับจุดที่ข้าม + log / คืนค่า**  
   เช่น `skippedInvalidLatLng`, `usedPoints`  
   ถ้า valid points = 0 ค่อย throw  
   ตอนนี้ข้ามเงียบได้

3. **เช็กจำนวน intermediate ก่อนยิง API**  
   Routes API รองรับ intermediate ประมาณ **25 จุด** (พิกัด)  
   ถ้าเกินควรถือเป็น error ชัด ไม่ใช่พึ่ง API ปฏิเสธ

4. **Retry เบา ๆ เมื่อ API 5xx / timeout**  
   1–2 ครั้ง + หน่วงสั้น ๆ  
   ตอนนี้ล้มครั้งเดียวจบ

5. **LockService ตอนเขียนแถวผล**  
   กันรันขนานทับ `rowId` เดียวกัน

### ควรทำเพื่อความถูกต้อง (MEDIUM)

6. **เตือนเมื่อ `ระยะทางจากคลัง_Km` เป็น 0 ทั้งชุด**  
   เพราะ heuristic เลือกปลายทางพัง

7. **อย่าตัด >20 เงียบ**  
   อย่างน้อย log หรือเขียนสถานะ `TRUNCATED_20`

8. **ตรวจ `routes && routes[0]` ก่อนใช้**  
   ข้อความ error ชัดกว่า

9. **อย่า hard-code Spreadsheet ID** ถ้าจะใช้หลายไฟล์  
   อ่านจาก `SpreadsheetApp.getActive()` หรือ Script Properties

10. **คืนสถานะรายละเอียดให้ caller**  
    ไม่ใช่แค่ Success + ระยะทาง  
    เช่น จำนวนจุดที่ใช้ / จุดที่ข้าม / ถูก optimize หรือไม่

### ทำทีหลังได้ (LOW)

11. รวม helper `findRowByValue_(sheet, colName, value)`  
12. cache header index ต่อ execution  
13. ใช้ `duration` ถ้าอนาคตต้องโชว์เวลา (ตอนนี้ขอแล้วไม่ใช้ — ไม่ผิดถ้าไม่ต้องการ)

---

## 5) Google Maps API — ควรย้าย / มีของใหม่ไหม?

### สิ่งที่สคริปต์ใช้อยู่ตอนนี้

```text
POST https://routes.googleapis.com/directions/v2:computeRoutes
travelMode: DRIVE
routingPreference: TRAFFIC_UNAWARE
optimizeWaypointOrder: true (เมื่อ intermediate > 1)
```

### ข้อสรุปจากเอกสารปัจจุบัน

| หัวข้อ | คำแนะนำ |
|--------|----------|
| ยังควรใช้ Routes API อยู่ไหม? | **ใช่** — เป็นตัวแทน Directions API (Legacy) ที่ Google แนะนำให้ migrate มา |
| ต้องเปลี่ยนไป API ใหม่กว่านี้ไหม? | **ยังไม่จำเป็น** สำหรับงานเบิกเงินแบบระยะทางคงที่ |
| `TRAFFIC_UNAWARE` | **เหมาะกับงานเบิก** ที่ต้องการตัวเลข reproducible ไม่ผันตามรถติด |
| `TRAFFIC_AWARE` / `TRAFFIC_AWARE_OPTIMAL` | ใช้เมื่อต้องการ ETA จริงตามจราจร — **ไม่เหมาะเป็นหลักสำหรับใบเบิกที่ต้องล็อกตัวเลข** |
| `optimizeWaypointOrder` | ใช้ถูกทางแล้ว และ **ห้ามใช้คู่กับ `TRAFFIC_AWARE_OPTIMAL`** ตามเอกสาร Google — โค้ดคุณใช้ `TRAFFIC_UNAWARE` จึงเข้าเงื่อนไข |
| Field mask | ใช้แล้ว ดี (ลด cost / latency) |
| Routes Preferred / Waypoint Optimization แบบ experimental | ไม่จำเป็นสำหรับเคสนี้ ซับซ้อนและ timeout สูงขึ้น |

**ไม่แนะนำตอนนี้**
- ถอยกลับ Directions API เก่า
- เปลี่ยนเป็น `TRAFFIC_AWARE_OPTIMAL` เพื่อ “ความแม่นยำ” ในงานเบิกเงิน (ตัวเลขจะเปลี่ยนตามเวลา)
- ใช้ Route Matrix แทน computeRoutes สำหรับเส้นทางเดียวหลายจุด (ไม่ตรงงาน)

**ทางเลือกเพิ่มในอนาคต (ไม่บังคับ)**
- เก็บ `routes.duration` แยกคอลัมน์ “เวลาโดยประมาณ” สำหรับวางแผน — คนละเรื่องกับตัวเลขเบิก
- ถ้ามีเคสต้องกลับคลัง → ออกแบบขา 2 (ไป–กลับ) คนละฟังก์ชัน อย่าผสมใน one-way

---

## 6) ตารางตัดสินใจสั้น ๆ

| คำถาม | คำตอบ |
|-------|--------|
| มีประสิทธิภาพดีไหม? | **ใช้ได้** กับ shipment เล็ก–กลาง แต่ I/O ชีตยังไม่ optimal |
| มี error เงียบไหม? | **มี** — ข้ามพิกัดเสีย, ตัด 20 จุด, ขาดคอลัมน์ผล, distance=0 |
| ฟังก์ชันซ้ำซ้อนไหม? | **แทบไม่มี** — โครงแยกดี |
| ควรแก้ก่อนใช้หนักไหม? | **ควร** — batch read, นับจุดข้าม, เช็กขีดจำกัด waypoint, retry, lock |
| ต้องเปลี่ยน Google API ไหม? | **ไม่ต้อง** — Routes API + TRAFFIC_UNAWARE ถูกทางสำหรับเบิกเงิน |

---

## 7) ลำดับแก้ที่คุ้มที่สุด (ถ้าจะปรับ)

1. Batch อ่านค่าจากแถวที่ TextFinder เจอ  
2. นับ/รายงานจุดที่ข้ามเพราะพิกัดเสีย  
3. จำกัดและแจ้งเมื่อ intermediate เกินเพดาน API / เกิน 20 ช่องเขียน  
4. Retry API สั้น ๆ + ตรวจ `routes[0]`  
5. Lock ตอนเขียนแถวผล  

ไม่จำเป็นต้อง rewrite ทั้งก้อน — โครงปัจจุบันใช้ต่อได้ ถ้าปิดช่อง silent-fail และลด chatty `getValue()`