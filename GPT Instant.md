GPT-5.1 Instant
ผมจะรีวิวให้แบบ “ถอดรหัสทุกบรรทัด” + ตรวจคุณภาพโค้ด + แนะนำปรับปรุง และอัปเดตเรื่อง Google Maps Routes API ให้ครบเลยนะครับ

1. ภาพรวมการทำงานของสคริปต์
โจทย์ของโค้ดนี้:

รับ shipmentId + rowId
ไปหาแถวที่เกี่ยวข้องในชีตคำนวณ (SHEET_COMPUTED) ด้วย TextFinder (เร็ว)
รวมพิกัด:
จุดเริ่มต้น = คลัง (DEPOT)
จุดส่งของทุกจุดของ shipment นั้น
พร้อม “ระยะทางจากคลัง” ในชีต (ใช้จัดลำดับ)
เลือกปลายทางสุดท้ายเป็น “จุดที่ไกลคลังที่สุด” แล้วจัดเส้นทาง:
เริ่มจากคลัง
จุดกลาง (ตามที่มี)
จบที่จุดไกลสุด
เรียก Google Maps Routes API (Directions v2: computeRoutes)
TRAFFIC_UNAWARE (ไม่สน jam เพื่อให้ตรงใบเสร็จ)
ถ้ามีจุดกลาง > 1 จุด ให้ Google optimize ลำดับเอง
เขียนผลลงชีตผลลัพธ์ (SHEET_RESULT)
สตริงรวม Lat,Lng ทั้งเส้นทาง
ระยะทางรวม (km)
ลิงก์ Google Maps
แยก Lat/Long_ปลายทาง_01–20
รวมแล้ว โครงสร้างดีมาก, แบ่งฟังก์ชันชัด, ตั้งชื่อสื่อความหมายดี

2. อธิบายการทำงานทีละฟังก์ชัน (ละเอียด)
2.1 findOptimalRouteUsingExistingDistance(shipmentId, rowId)
ฟังก์ชันหลัก (entry point)

Workflow:

Log จุดเริ่ม
SpreadsheetApp.openById(SPREADSHEET_ID)
เปิดไฟล์ Google Sheets
prepareWaypointsWithExistingDistance(ss, shipmentId)
ไปหา waypoint ทั้งหมด ของ shipment นี้ (รวมคลัง)
ถ้าจำนวนจุด ≤ 1 (มีแต่คลัง) → throw error
selectFinalDestinationAndSort(waypointsWithDistance)
ใช้ “ระยะทางจากคลัง” ที่มีอยู่แล้ว เพื่อเลือกจุดไกลสุดเป็นปลายทาง
เรียงจุดอื่นตามระยะทางจากคลัง (ไกล → ใกล้)
executeGoogleMapsRoutesAPIOneWay(orderedPoints)
ยิง Google Routes API ขอเส้นทางจริง + optimization
เขียนผลลงชีต:
writeResultsToSheet(resultSheet, rowId, result.orderedWaypoints, ...)
คืนค่าว่า success + ระยะทาง + ลิงก์
ข้อดี:

โฟลว์อ่านง่าย, error ทุกจุดจะถูก catch แล้ว log
ใช้ชื่อฟังก์ชันชัดเจน บอกหน้าที่
2.2 prepareWaypointsWithExistingDistance(ss, shipmentId)
เป้าหมาย: ดึง waypoint ของ shipment หนึ่งชุดจาก SHEET_COMPUTED โดยใช้ TextFinder

ดึงชีต

Copyconst computedSheet = ss.getSheetByName(SHEET_COMPUTED);
อ่าน header แถวที่ 1

Copyconst headerRange = computedSheet.getRange(1, 1, 1, computedSheet.getLastColumn());
const header = headerRange.getValues()[0];
หา index ของคอลัมน์:

"Shipment No"
"จุดส่งสินค้าปลายทาง" (พิกัด)
"ชื่อปลายทาง"
DEPOT_DISTANCE_COLUMN ("ระยะทางจากคลัง_Km")
Copyconst shipmentColIdx = header.indexOf("Shipment No") + 1;
...
if (shipmentColIdx === 0 || latlngColIdx === 0) throw ...
ตรงนี้:

indexOf คืน -1 ถ้าไม่เจอ → +1 แล้วได้ 0
แล้วไปเช็ค === 0 เพื่อจับ error = ถูกต้อง
ใช้ TextFinder หา shipmentId ในคอลัมน์ "Shipment No" (เฉพาะบรรทัดข้อมูล)

Copyconst searchRange = computedSheet.getRange(2, shipmentColIdx, computedSheet.getLastRow(), 1);
const matches = searchRange.createTextFinder(String(shipmentId).trim()).matchEntireCell(true).findAll();
สร้าง array allPoints เริ่มต้นด้วย DEPOT (จุดคลัง)

CopyallPoints.push({
  id: 0,
  name: DEPOT_COORDS.name,
  original: { lat, lng },
  forApi: { location: { latLng: { latitude, longitude } } },
  distance: 0,
  isDepot: true
});
วนแต่ละแถวที่ match shipment นี้

Copyconst latlngRaw = computedSheet.getRange(rowIdx, latlngColIdx).getValue();
...
const cleanedLatLong = String(latlngRaw).trim().replace(/[^\d.,-]/g, "");
เอาเฉพาะตัวเลข, จุด, คอมมา, ลบ
split by ,
parseFloat lat/lng
ถ้า parse ผ่าน → ดึง distance และ name (optional)
distance → เอาจาก ระยะทางจากคลัง_Km ถ้ามี
name → เอาจาก "ชื่อปลายทาง" ถ้ามี
push ลง allPoints พร้อมโครงสร้าง for API
คืน allPoints (DEPOT + จุดลูกค้า)

ข้อดี:

ใช้ TextFinder → FAST+BATCH, ไม่ดึงทั้งชีตมา
รองรับคอลัมน์หายไป (name/distance) ด้วย if > 0
clean string พิกัดได้ดี (ตัดตัวหนังสือไทย/อื่นๆ ออก)
จุดที่ควรปรับปรุง (important – performance):

ตอนวน matches.forEach ใช้ getRange ทีละช่อง:

CopygetRange(rowIdx, latlngColIdx).getValue()
getRange(rowIdx, distanceColIdx).getValue()
getRange(rowIdx, nameColIdx).getValue()
ถ้าชุดข้อมูลเยอะ (เช่น shipment มี 50–100 แถว) → จำนวน call ไป spreadsheet เยอะ → ช้า

แนะนำ:

อ่านทีเดียวเป็น block แล้วใช้ index ใน array แทน
ตัวอย่างปรับ:

Copyconst rows = matches.map(m => m.getRow());
const minRow = Math.min(...rows);
const maxRow = Math.max(...rows);
const numRows = maxRow - minRow + 1;

const dataRange = computedSheet.getRange(minRow, 1, numRows, computedSheet.getLastColumn()).getValues();

// สร้าง map: rowNumber -> rowData
const rowMap = {};
rows.forEach(r => {
  rowMap[r] = dataRange[r - minRow];  // index offset
});

matches.forEach(match => {
  const rowIdx = match.getRow();
  const row = rowMap[rowIdx];
  const latlngRaw = row[latlngColIdx - 1]; // เพราะ array index เริ่ม 0
  ...
});
จะเร็วขึ้นชัดเจนมากเมื่อข้อมูลเยอะ

2.3 selectFinalDestinationAndSort(allPoints)
หน้าที่: ใช้ข้อมูล “ระยะทางจากคลัง” มาตัดสินปลายทาง และเรียงจุด

แยกคลัง (index 0) และจุดปลายทางอื่น

Copyconst depot = allPoints[0];
const destinations = allPoints.slice(1);
เรียงตาม distance มาก → น้อย

Copydestinations.sort((a, b) => (b.distance || 0) - (a.distance || 0));
จุดแรกหลัง sort = ไกลสุด = finalDestination
ที่เหลือ = intermediates

สร้างลำดับ:

CopyorderedPoints = [depot, ...intermediates, finalDestination]
ข้อดี:

logic ชัด: เอา “จุดไกลสุด” เป็นปลายทางสุดท้าย
ใช้ค่า distance ที่มีอยู่แล้ว → ไม่ต้องยิง API หา origin-distance เอง
สิ่งที่ควรคิดเพิ่ม:

ถ้า distance ในชีตผิด / ว่าง (0) → อัลกอริทึมจะผิด
ตอนนี้ใช้ (b.distance || 0) → ถ้า distance ว่างทั้งหมด, ลำดับจะเท่ากัน, ก็ใช้ลำดับเดิม
ถ้าต้องการ robust กว่านี้:
ตรวจว่ามีอย่างน้อย 1 จุดที่ distance > 0 มั้ย
ถ้าไม่มี → fallback: เลือกจุดสุดท้ายของ matches เป็นปลายทาง หรือไม่ optimize เลย
2.4 executeGoogleMapsRoutesAPIOneWay(allPoints)
หน้าที่: ยิง Google Maps Routes API แบบ Directions v2: computeRoutes

ดึง API key:

Copyconst GOOGLE_MAPS_API_KEY = PropertiesService.getScriptProperties().getProperty("GOOGLE_MAPS_API_KEY");
แยก origin / destination / intermediates:

Copyconst origin = allPoints[0];
const finalDestination = allPoints[allPoints.length - 1];
const intermediates = allPoints.slice(1, -1);
const shouldOptimize = intermediates.length > 1;
สร้าง payload ตาม spec ของ Routes API:

Copyconst payload = {
  origin: origin.forApi,                         // {location: { latLng: { latitude, longitude } }}
  destination: finalDestination.forApi,
  intermediates: intermediates.map(p => p.forApi),
  travelMode: 'DRIVE',
  routingPreference: 'TRAFFIC_UNAWARE',
  optimizeWaypointOrder: shouldOptimize
};
Request config:

Copyconst apiUrl = "https://routes.googleapis.com/directions/v2:computeRoutes";
const options = {
  method: 'post',
  contentType: 'application/json',
  headers: {
    'X-Goog-Api-Key': GOOGLE_MAPS_API_KEY,
    'X-Goog-FieldMask': 'routes.duration,routes.distanceMeters,routes.optimizedIntermediateWaypointIndex'
  },
  payload: JSON.stringify(payload),
  muteHttpExceptions: true
};
ใช้ FieldMask แค่สิ่งที่ต้องใช้ → ประหยัด quota และเร็วขึ้น
muteHttpExceptions: true แล้วตรวจเอง → ดี
Execute + ตรวจโค้ดตอบกลับ

Copyconst response = UrlFetchApp.fetch(apiUrl, options);
if (response.getResponseCode() !== 200) throw new Error(`API Error: ${response.getContentText()}`);
Parse:

Copyconst result = JSON.parse(response.getContentText());
const route = result.routes[0];
const totalDistance = (route.distanceMeters || 0) / 1000;
เอา order หลัง optimization:

Copyconst orderedWaypoints = [origin];
if (shouldOptimize && route.optimizedIntermediateWaypointIndex) {
  route.optimizedIntermediateWaypointIndex.forEach(index => {
    if (intermediates[index]) orderedWaypoints.push(intermediates[index]);
  });
} else {
  orderedWaypoints.push(...intermediates);
}
orderedWaypoints.push(finalDestination);
สร้าง Link:

Copyconst linkCoordinates = orderedWaypoints.map(p => `${p.original.lat.toFixed(6)},${p.original.lng.toFixed(6)}`);
const rawMapsLink = `https://www.google.com/maps/dir/${linkCoordinates.join('/')}`;
const googleMapsLink = encodeURI(rawMapsLink);
ใช้ .toFixed(6) → ความละเอียด 6 decimal (~0.11m) ดีแล้ว
encodeURI ป้องกันปัญหาตัวอักษรผิด (ถึงจริงๆ ลิงก์นี้มีแต่ตัวเลข , / ก็โอเคอยู่แล้ว)
ข้อดี:

ใช้ Routes API รุ่นใหม่ (Directions v2) ซึ่ง Google แนะนำแทน Directions API เก่า
ใช้ optimization ของ Google เอง → ผลเส้นทางเหมาะสม
ใช้ field mask ถูกต้อง: routes.duration,routes.distanceMeters,routes.optimizedIntermediateWaypointIndex
สิ่งที่ควรปรับปรุง:

ตรวจ result.routes ว่ามี route กลับมาจริงไหม

ตอนนี้ assume ว่ามี routes[0] เสมอ
แต่ถ้า API หาทางไม่ได้ / input เพี้ยน → result.routes อาจเป็น [] หรือ undefined

แนะนำ:

Copyif (!result.routes || result.routes.length === 0) {
  throw new Error(`No route found: ${response.getContentText()}`);
}
const route = result.routes[0];
การจัดการ error จาก Google ให้ชัดขึ้น

ปัจจุบัน throw new Error("API Error: ...") ดีแล้ว
เพิ่ม log พิเศษกรณี quota เกิน, key ผิด, ฯลฯ ก็ได้ โดยอ่านจาก error.status / error.message ใน body

URL ยาวเกิน (edge case):

ถ้า waypoint เยอะมาก (20–25 จุด) → URL /dir/lat,lng/... อาจยาว > 2048 char (บาง browser limit)
ส่วนใหญ่โอเค แต่ถ้าอยากชัวร์:
ถ้า linkCoordinates.length > ~20 → อาจสร้างลิงก์เฉพาะ origin+final หรือใช้ Google Maps short link (ผ่าน Web UI – API ตรงๆ ไม่มี)
ยังไม่ critical ถ้าจำนวนจุดไม่เยอะมาก
2.5 writeResultsToSheet(resultSheet, rowId, orderedPoints, totalDistance, googleMapsLink)
หน้าที่: เขียนผลคำนวณลง SHEET_RESULT

อ่าน header แถว 1

หา index ของ:

'ID_ทำเบิกส่วนต่างScgวังน้อย'
RESULT_COLUMN_NAME ("GoogleMapsRoutesAPI")
DISTANCE_COLUMN_NAME ("ระยะทาง_GoogleMapAPI_Km")
LINK_COLUMN_NAME ("แสดงแผนที่_GoogleMapsRoutesAPI")
'Lat/Long_ปลายทาง_01' (คอลัมน์แรกของ block 20 จุด)
ถ้าไม่เจอ column ID → throw error

ใช้ TextFinder หาบรรทัดด้วย rowId ในคอลัมน์ ID:

Copyconst idRange = resultSheet.getRange(2, idIndex + 1, resultSheet.getLastRow(), 1);
const idMatch = idRange.createTextFinder(String(rowId).trim()).matchEntireCell(true).findNext();
if (!idMatch) throw new Error(`Row not found for ID: ${rowId}`);
สร้าง string รวมพิกัด:

Copyconst resultString = orderedPoints
  .map(p => `${p.original.lat.toFixed(6)}, ${p.original.lng.toFixed(6)}`)
  .join(" | ");
เขียนค่าหลัก (ทีละ cell เพื่อไม่ทับสูตรอื่น):

Copyif (mainResultColIndex !== -1) ...
if (distanceColIndex !== -1) ...
if (linkColIndex !== -1) ...
เขียนชุด Lat/Long_ปลายทาง_01–20 แบบ batch:

Copyconst customersOnly = orderedPoints.slice(1); // ตัด depot ออก
const maxColumns = 20;
const valuesToWrite = new Array(maxColumns).fill('');

customersOnly.forEach((p, i) => {
  if (i < maxColumns) valuesToWrite[i] = `${p.original.lat.toFixed(6)}, ${p.original.lng.toFixed(6)}`;
});

resultSheet
  .getRange(rowIndexInSheet, firstDestColIndex + 1, 1, maxColumns)
  .setValues([valuesToWrite]);
ข้อดี:

แยกการอัปเดตเป็นจุด ๆ → ลดความเสี่ยงเขียนทับสูตรช่องอื่น
ใช้ batch เขียน 20 ช่องในครั้งเดียว → ดีมาก
ใช้ TextFinder หา row แทนการไล่ loop → เร็วและแม่นกว่า
สิ่งที่พอปรับปรุงได้:

จำนวน call ไป getRange ยังค่อนข้างเยอะ (แต่ไม่ซีเรียสมากเท่าฟังก์ชันเตรียมข้อมูล)
อาจใส่ format ระยะทางให้ชัด (เช่นปัดทศนิยม 2 ตำแหน่ง)
3. ประสิทธิภาพ (Performance) & จุดที่อาจเป็น “error เงียบ”
3.1 ประสิทธิภาพภาพรวม
สิ่งที่ทำดีแล้ว:

ใช้ TextFinder แทน getValuesทั้งชีต
Limit FieldMask ของ Routes API ให้เอาเฉพาะที่ใช้
Batch setValues สำหรับ 20 คอลัมน์ Lat/Long ปลายทาง
จุดที่เป็นคอขวดหลักตอนข้อมูลเยอะ:

prepareWaypointsWithExistingDistance:

ใช้ getRange(...).getValue() ทีละช่อง ใน loop ทุก match
ถ้า shipment หนึ่งมี 100 แถว → 100×(1–3 range call) = 100–300 calls
Google Apps Script ช้าเพราะ “จำนวน call ไป Spreadsheet service” เป็นหลัก
แนะนำ: batch read (ตามตัวอย่างที่ให้ไปข้างบน)

writeResultsToSheet:

อ่าน header ทุกครั้งที่เรียก
หาบรรทัดด้วย TextFinder → โอเค แต่ถ้าเรียกบ่อยมากๆ ใน loop ใหญ่ (เช่นหลาย shipment)
อาจ cache header และ map ของ rowId -> rowIndex ไว้ในหน่วยความจำ (สำหรับ run เดียว) เพื่อลดรอบ TextFinder
3.2 error เงียบ (Silent bugs / silent failures)
เท่าที่ดู มี error หลัก ๆ โยนทิ้งหมด (ดีมาก) แต่มี “เหลือ ๆ” บางอย่างที่อาจสร้างพฤติกรรมแปลก ๆ แบบไม่ crash:

route ไม่เจอแต่โค้ด assume มี:

Copyconst route = result.routes[0];
const totalDistance = (route.distanceMeters || 0) / 1000;
ถ้า route ไม่มี → route = undefined → จะ crash (TypeError) ซึ่งยังดีที่ไปเข้า catch ข้างนอก (ไม่เงียบ)

แต่อ่าน log จะไม่รู้สาเหตุชัด ๆ ว่า “หา route ไม่ได้”
เลยแนะนำเพิ่ม check และ throw msg ชัด ๆ ตามข้อที่กล่าวไปแล้ว

การจัดลำดับ selectFinalDestinationAndSort:

ถ้า distance บางจุดเป็น NaN (เช่น parse ไม่ได้) → (b.distance || 0) กลายเป็น 0
ทำให้เรียงผิดได้ แต่ไม่ crash
กรณี header เปลี่ยนชื่อ:

ถ้า "ระยะทางจากคลัง_Km" เปลี่ยนชื่อ → distanceColIdx จะเป็น 0 → มีผล:
Copyconst distance = distanceColIdx > 0 
  ? parseFloat(computedSheet.getRange(rowIdx, distanceColIdx).getValue()) || 0 
  : 0;
ที่นี่ distanceColIdx > 0 = false → distance = 0 ทั้งหมด → ลำดับปลายทางจะไม่แม่น
แต่โค้ดไม่ throw → เงียบ (แต่ยัง run)
แนะนำ:

ถ้าคุณถือว่า "ระยะทางจากคลัง_Km" เป็นคอลัมน์สำคัญมาก ให้บังคับต้องมี:
Copyif (distanceColIdx === 0) throw new Error("Missing column ระยะทางจากคลัง_Km");
การ clean พิกัด:

CopycleanedLatLong = String(latlngRaw).trim().replace(/[^\d.,-]/g, "");
ถ้า format แปลก ๆ เช่นมี | หลายคู่ หรือ Lat:14.12, Lng:100.12 → น่าจะถูก clean ได้
แต่ถ้าข้อมูลเป็น 14.12,100.12 / 13.99,99.88 (หลายพิกัดในช่องเดียว) → โค้ดนี้จะกลายเป็น 14.12,100.12,13.99,99.88 → split แล้ว parts.length = 4 → ไม่เข้าเงื่อนไข parts.length === 2 → ข้ามทั้งเซลล์นี้ (เงียบ)
ถ้ารู้อยู่แล้วว่าในชีตคุณเก็บ “1 ช่อง = 1 พิกัด” ตลอด ก็ไม่มีปัญหา
ถ้ามี case ผสมกัน อาจอยาก log warning

4. ฟังก์ชัน/โค้ดที่ซ้ำซ้อนหรือปรับได้
โดยรวมโค้ดไม่ได้ซ้ำซ้อนเยอะ โครงสร้างโอเค แต่มีจุดที่ refactor แล้วอ่านง่าย/ใช้ซ้ำได้มากขึ้น:

ส่วนที่ “หา column index จาก header” ทำซ้ำใน 2 ฟังก์ชัน

prepareWaypointsWithExistingDistance
writeResultsToSheet
อาจแยกเป็น helper:

Copyfunction getHeaderIndexMap(sheet) {
  const header = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  const map = {};
  header.forEach((name, idx) => { map[name] = idx + 1; }); // +1 for sheet index
  return map;
}
แล้วในแต่ละฟังก์ชันเรียก:

Copyconst headerMap = getHeaderIndexMap(computedSheet);
const shipmentColIdx = headerMap["Shipment No"] || 0;
...
การแปลงพิกัดให้เป็น string Lat,Lng:

ทำซ้ำหลายที่: ตอนสร้าง resultString, ตอนสร้าง link, ตอนเขียน 20 ช่อง
อาจทำเป็น helper
Copyfunction formatLatLng(p, withSpace = false) {
  const sep = withSpace ? ', ' : ',';
  return `${p.original.lat.toFixed(6)}${sep}${p.original.lng.toFixed(6)}`;
}
ก็จะสั้นลงและลดโอกาสพิมพ์ผิด

5. เรื่อง Google Maps Routes API – ฟีเจอร์ใหม่/ทางเลือก
คุณใช้ endpoint รุ่นใหม่แล้ว:
https://routes.googleapis.com/directions/v2:computeRoutes → ดีมาก

สิ่งที่สามารถใช้เพิ่มได้ (ถ้าอนาคตต้องการ):

polyline / geometry
ถ้าสักวันอยากวาดเส้นบน map เอง (เช่นใน Web App)
สามารถเพิ่มใน FieldMask:

Copy'routes.distanceMeters,routes.duration,routes.polyline.encodedPolyline'
RouteModifiers
ถ้าต้องการเลี่ยงทางด่วน / ทางดิน / เฉพาะถนนใหญ่:

CopyrouteModifiers: {
  avoidTolls: true,
  avoidHighways: false,
  avoidFerries: true
}
TRAFFIC_AWARE / TRAFFIC_AWARE_OPTIMAL
ตอนนี้คุณใช้ TRAFFIC_UNAWARE เพราะเหตุผลด้านใบเสร็จ → ok
แต่อาจมี use case:

ใช้ TRAFFIC_AWARE_OPTIMAL เพื่อวางแผนปฏิบัติการจริง (ETA)
แล้ว TRAFFIC_UNAWARE เฉพาะตอนทำเอกสารเบิก
สามารถแตกเป็น 2 ฟังก์ชันหรือใส่ parameter เลือกโหมดได้

Units / Locale
Routes API จะคืน meter → คุณแปลงเป็น km เองแล้ว ถูกต้อง
ถ้าต้องการเวลาแบบละเอียด สามารถใช้ routes.duration ได้ (ตอนนี้ดึงมาแต่ไม่ได้ใช้)

Batching / Rate limits
ถ้าในอนาคตคุณยิง API หลายร้อยครั้งในหนึ่ง run:

ควรมี queue + delay เล็กน้อย (sleep) หรือใช้ UrlFetchApp.fetchAll (แต่ต้องระวัง quota)
ตอนนี้ฟังก์ชันออกแบบมาใช้ทีละ shipment ซึ่งโอเคอยู่
6. สรุปสิ่งที่ดี + สิ่งที่แนะนำให้แก้/เพิ่ม (แบบหัวข้อ)
จุดแข็งของโค้ด
แยกโครงสร้างเป็นฟังก์ชันตามหน้าที่ชัดเจน
ใช้ TextFinder เพื่อลดเวลาอ่านชีต
ใช้ Routes API รุ่นใหม่ถูกต้อง
ใช้ FieldMask ฉลาด (เฉพาะ field ที่ใช้)
เขียนค่าลงชีตแบบ batch (20 ช่องปลายทาง)
มี log และใช้ try/catch ที่ชั้นบนสุดของฟังก์ชันหลัก
การตั้งชื่อตัวแปรและคอนฟิกอ่านง่ายมาก
สิ่งที่ “ควร” ปรับ (Priority สูง)
Batch อ่านข้อมูลใน prepareWaypointsWithExistingDistance
ลดจำนวน getRange().getValue() ใน loop (ช่วยเรื่อง performance อย่างชัดเจน)

ตรวจ no-route case จาก Google Routes API

Copyif (!result.routes || result.routes.length === 0) {
  throw new Error(`No route returned from API: ${response.getContentText()}`);
}
บังคับคอลัมน์ที่ critical ถ้าต้องใช้แน่นอน

เช่น ถ้า ระยะทางจากคลัง_Km ไม่ควรหาย:

Copyif (distanceColIdx === 0) throw new Error("Missing ระยะทางจากคลัง_Km in computed sheet");
แยก helper สำหรับ format พิกัด + header index map

ลด duplication และ reduce bug เวลาแก้อะไรทีเดียว
สิ่งที่ “อาจ” ปรับ (Optional / แล้วแต่ use-case)
เพิ่ม log รายละเอียดกรณี API error เช่น quota, invalid key
คุมจำนวนจุดกลางสูงสุดเพื่อป้องกัน URL ลิงก์ยาวเกิน
ถ้าใช้บ่อยใน loop หลาย shipment → cache header ที่อ่านแล้วใน global variable สำหรับ run นั้น
ถ้าคุณอยาก ผมสามารถ:

เขียน “เวอร์ชันปรับปรุงแล้ว” ให้ครบทั้งไฟล์ (รวม refactor ที่แนะนำ)
หรือออกแบบให้รองรับ 2 โหมด:
โหมดเบิกเงิน (TRAFFIC_UNAWARE)
โหมดวางแผนงานจริง (TRAFFIC_AWARE_OPTIMAL)
คุณอยากให้ผมช่วยต่อในทิศทางไหนก่อน: ปรับโค้ดให้ หรือออกแบบโครงสร้างเพิ่ม (เช่นรองรับหลายคลัง, หลาย provider) ครับ?