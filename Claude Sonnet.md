Claude Sonnet 4.6
🔍 การวิเคราะห์โค้ด Optimized Routing Script (PRO Version) แบบละเอียด
ภาพรวมสถาปัตยกรรมของโค้ด (Architecture Overview):
โค้ดชุดนี้เป็น Google Apps Script ที่ทำงานบน Google Sheets โดยมีวัตถุประสงค์หลักคือการคำนวณเส้นทางขนส่งที่เหมาะสมที่สุดสำหรับระบบโลจิสติกส์ของ SCG-JWD วังน้อย โดยดึงข้อมูลจาก Spreadsheet, เรียก Google Maps Routes API, และเขียนผลลัพธ์กลับเข้า Sheet อีกแผ่น โครงสร้างแบ่งออกเป็น 5 ส่วนชัดเจน ได้แก่ Configuration, Main Function, Data Preparation, API Execution, และ Write Results

Section 1: การวิเคราะห์ทีละฟังก์ชัน (Deep Dive Per Function):
findOptimalRouteUsingExistingDistance (Main Orchestrator)

ฟังก์ชันนี้ทำหน้าที่เป็น Controller หลักที่เรียงลำดับการทำงานทั้งหมด รับพารามิเตอร์ 2 ตัวคือ shipmentId (รหัสใบงาน) และ rowId (หมายเลขแถวใน Result Sheet) การทำงานไหลตามลำดับ 4 ขั้นตอนชัดเจน และมี try-catch ครอบทั้งหมด ซึ่งเป็นแนวทางที่ถูกต้อง

prepareWaypointsWithExistingDistance (Data Layer)

ฟังก์ชันนี้ทำงานหนักที่สุดในระบบ โดยมีขั้นตอนดังนี้:

โหลด Header Row เพื่อหาตำแหน่งคอลัมน์แบบ Dynamic (ไม่ Hardcode เลขคอลัมน์)
ใช้ TextFinder ค้นหาแถวที่ตรงกับ shipmentId ซึ่งเร็วกว่าการดึงข้อมูลทั้งชีตมาก
ทำความสะอาด Coordinate String ด้วย Regex
สร้าง Point Object พร้อม Format ที่พร้อมส่ง API
selectFinalDestinationAndSort (Business Logic)

ฟังก์ชันนี้ใช้ Business Logic ที่ว่า "จุดที่ไกลจากคลังที่สุดคือปลายทางสุดท้ายเสมอ" โดย Sort ตาม distance จากมากไปน้อย แล้วเอาตัวแรก (ไกลสุด) เป็น Final Destination

executeGoogleMapsRoutesAPIOneWay (API Layer)

ฟังก์ชันนี้สร้าง Payload ส่ง Routes API v2, จัดการ Waypoint Optimization, และสร้าง Google Maps Link

writeResultsToSheet (Output Layer)

ฟังก์ชันนี้เขียนผลลัพธ์กลับ Sheet โดยใช้ Batch Writing สำหรับ 20 คอลัมน์ปลายทาง และเขียนทีละเซลล์สำหรับ Summary Fields

Section 2: ปัญหา Silent Errors (เงียบ ไม่ Throw แต่ผลลัพธ์ผิด):
นี่คือส่วนที่อันตรายที่สุดในโค้ด เพราะระบบจะทำงานต่อไปโดยไม่มีการแจ้งเตือน แต่ข้อมูลที่ได้อาจผิดพลาด

🔴 Silent Error #1: Regex ทำลาย Coordinate ที่มีลบ (Negative Lat/Lng)

Copy// โค้ดปัจจุบัน - มีปัญหา!
const cleanedLatLong = String(latlngRaw).trim().replace(/[^\d.,-]/g, "");
Regex [^\d.,-] จะเก็บ - ไว้ แต่ถ้าข้อมูลเป็น "14.1646106, 100.6254644" (มีเว้นวรรค) จะกลายเป็น "14.1646106,100.6254644" ซึ่งโอเค แต่ถ้าเป็น "-33.8688, 151.2093" (พิกัดออสเตรเลีย หรือพิกัดใต้เส้นศูนย์สูตร) ก็ยังโอเค อย่างไรก็ตาม ปัญหาที่แท้จริงคือถ้า Input มีหลายคอมม่า เช่น "14.164610,6, 100.625464,4" (ข้อมูลเสีย) จะ Split ได้มากกว่า 2 parts แต่โค้ดเช็คแค่ parts.length === 2 ซึ่งจะ Skip ไปเงียบๆ โดยไม่มี Log

Copy// แนะนำ: เพิ่ม Log เมื่อ Skip
if (parts.length !== 2) {
  Logger.log(`⚠️ SKIP Row ${rowIdx}: Invalid coordinate format "${latlngRaw}"`);
  return;
}
🔴 Silent Error #2: distance = 0 เมื่อ Parse ล้มเหลว ทำให้ Sort ผิด

Copy// โค้ดปัจจุบัน
const distance = distanceColIdx > 0 
  ? parseFloat(computedSheet.getRange(rowIdx, distanceColIdx).getValue()) || 0 
  : 0;
ถ้าค่าในเซลล์ระยะทางเป็น String เช่น "N/A" หรือ "" หรือยังไม่ได้คำนวณ parseFloat จะคืนค่า NaN และ || 0 จะทำให้ Distance กลายเป็น 0 ผลคือจุดนั้นจะถูก Sort ไปอยู่ท้ายสุด และอาจกลายเป็น Final Destination ที่ผิดพลาดโดยไม่มีการเตือน

Copy// แนะนำ: แยก Warning ออกมา
const rawDistance = computedSheet.getRange(rowIdx, distanceColIdx).getValue();
const distance = parseFloat(rawDistance);
if (isNaN(distance)) {
  Logger.log(`⚠️ WARNING Row ${rowIdx} [${name}]: Distance is "${rawDistance}" → defaulting to 0. Sort may be incorrect!`);
}
const safeDistance = isNaN(distance) ? 0 : distance;
🔴 Silent Error #3: optimizedIntermediateWaypointIndex อาจหายไปจาก Response

Copy// โค้ดปัจจุบัน
if (shouldOptimize && route.optimizedIntermediateWaypointIndex) {
  route.optimizedIntermediateWaypointIndex.forEach(index => {
    if (intermediates[index]) orderedWaypoints.push(intermediates[index]);
  });
}
ถ้า API ส่ง optimizedIntermediateWaypointIndex กลับมาแต่มีบาง Index ที่ชี้ไป intermediates[index] ที่ไม่มีอยู่ (Array out of bounds) โค้ดจะ Skip จุดนั้นไปเงียบๆ ผลคือ orderedWaypoints มีจำนวนจุดน้อยกว่าที่ควรจะเป็น แต่ระบบไม่ Throw Error

🔴 Silent Error #4: การดึงข้อมูลทีละเซลล์ใน Loop (N+1 Query Problem)

Copymatches.forEach(match => {
  const rowIdx = match.getRow();
  const latlngRaw = computedSheet.getRange(rowIdx, latlngColIdx).getValue(); // ❌ API Call ทุก Loop!
  // ...
  const distance = computedSheet.getRange(rowIdx, distanceColIdx).getValue(); // ❌ อีก API Call!
  const rawName = computedSheet.getRange(rowIdx, nameColIdx).getValue(); // ❌ อีก API Call!
});
นี่คือ Performance Silent Error ที่สำคัญมาก ถ้ามี 10 จุดส่ง = 30 Spreadsheet API Calls ถ้ามี 50 จุดส่ง = 150 Calls ซึ่งจะทำให้สคริปต์ช้ามากและอาจ Timeout ใน 6 นาที (Limit ของ Apps Script)

Section 3: ปัญหาด้านประสิทธิภาพ (Performance Issues):
🟡 Issue #1: N+1 Read Problem (ปัญหาหลัก)

อย่างที่กล่าวไป การเรียก getRange().getValue() ทีละเซลล์ใน Loop เป็นสิ่งที่ควรหลีกเลี่ยงอย่างยิ่งใน Apps Script เพราะทุก Call มี Network Overhead ไปยัง Google Servers

Copy// ✅ วิธีที่ถูกต้อง: ดึงข้อมูลทั้งบล็อกครั้งเดียว
const allData = computedSheet.getRange(2, 1, computedSheet.getLastRow() - 1, computedSheet.getLastColumn()).getValues();

// แล้วหา Row Index จาก TextFinder แล้ว Map กลับมาที่ Array
matches.forEach(match => {
  const rowIdx = match.getRow() - 2; // แปลงเป็น 0-based index ของ allData
  const latlngRaw = allData[rowIdx][latlngColIdx - 1];
  const distance = allData[rowIdx][distanceColIdx - 1];
  const rawName = allData[rowIdx][nameColIdx - 1];
});
🟡 Issue #2: getLastRow() ถูกเรียกซ้ำหลายครั้ง

Copy// โค้ดปัจจุบันเรียก getLastRow() หลายครั้ง
const searchRange = computedSheet.getRange(2, shipmentColIdx, computedSheet.getLastRow(), 1);
// และใน writeResultsToSheet ก็เรียกอีก
const idRange = resultSheet.getRange(2, idIndex + 1, resultSheet.getLastRow(), 1);
ควร Cache ค่าไว้ในตัวแปรก่อน:

Copyconst lastRow = computedSheet.getLastRow();
const searchRange = computedSheet.getRange(2, shipmentColIdx, lastRow, 1);
🟡 Issue #3: encodeURI ไม่เพียงพอสำหรับ URL ที่มีอักขระพิเศษ

Copyconst googleMapsLink = encodeURI(rawMapsLink);
encodeURI จะไม่ Encode :, /, , ซึ่งในกรณีนี้โอเคเพราะ Coordinate ใช้อักขระเหล่านี้ แต่ถ้าชื่อสถานที่ถูกใส่เข้าไปใน URL ในอนาคต จะเกิดปัญหา ควรใช้ encodeURIComponent สำหรับแต่ละ Segment แทน

Section 4: ปัญหาด้านความถูกต้องของ Logic (Logic Correctness Issues):
🔴 Logic Issue #1: Business Rule "ไกลสุด = ปลายทาง" อาจไม่ถูกเสมอไป

Copydestinations.sort((a, b) => (b.distance || 0) - (a.distance || 0));
const finalDestination = destinations[0]; // ไกลสุดเสมอ
สมมติว่ามีเส้นทาง A(50km) → B(80km) → C(60km) จากคลัง ระบบจะเลือก B เป็นปลายทาง แต่ถ้าเส้นทางที่เหมาะสมจริงๆ คือ A → C → B (ไปตามทาง) การบังคับให้ B เป็นปลายทางอาจทำให้ Google ต้องหาเส้นทางย้อนกลับ ซึ่งระยะทางรวมอาจมากกว่าการปล่อยให้ optimizeWaypointOrder: true จัดการทั้งหมด

🔴 Logic Issue #2: shouldOptimize Condition อาจทำให้ไม่ Optimize เมื่อมีแค่ 1 Intermediate

Copyconst shouldOptimize = intermediates.length > 1;
ถ้ามีเพียง 2 จุดส่ง (1 intermediate + 1 final destination) shouldOptimize จะเป็น false ซึ่งโอเคสำหรับ 2 จุด แต่ Logic ยังคงถูกต้องอยู่ เพียงแต่ควร Comment อธิบายให้ชัดว่าทำไมถึงไม่ Optimize กรณีนี้

🟡 Logic Issue #3: Depot ถูกรวมใน orderedWaypoints แล้วส่งไป Batch Write

Copyconst customersOnly = orderedPoints.slice(1); // ตัด Depot ออก
ตรงนี้โอเค แต่ใน resultString ยังรวม Depot ไว้:

Copyconst resultString = orderedPoints.map(p => `${p.original.lat.toFixed(6)}, ${p.original.lng.toFixed(6)}`).join(" | ");
ต้องตรวจสอบว่า Column GoogleMapsRoutesAPI ต้องการพิกัด Depot รวมอยู่ด้วยหรือไม่ ถ้าไม่ต้องการก็ควรใช้ orderedPoints.slice(1) เช่นกัน

Section 5: ปัญหาด้านความทนทาน (Robustness Issues):
🔴 Robustness Issue #1: ไม่มี Retry Logic สำหรับ API Call

ถ้า API ล้มเหลวชั่วคราว (HTTP 429 Rate Limit หรือ 500 Server Error) ระบบจะ Throw Error ทันทีโดยไม่ลองใหม่ ควรเพิ่ม Exponential Backoff:

Copyfunction fetchWithRetry(url, options, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const response = UrlFetchApp.fetch(url, options);
    if (response.getResponseCode() === 200) return response;
    if (response.getResponseCode() === 429 || response.getResponseCode() >= 500) {
      if (attempt < maxRetries) {
        const waitMs = Math.pow(2, attempt) * 1000; // 2s, 4s, 8s
        Logger.log(`⚠️ API Error ${response.getResponseCode()}, Retry ${attempt}/${maxRetries} after ${waitMs}ms`);
        Utilities.sleep(waitMs);
      }
    } else {
      throw new Error(`API Error ${response.getResponseCode()}: ${response.getContentText()}`);
    }
  }
  throw new Error(`API failed after ${maxRetries} retries`);
}
🔴 Robustness Issue #2: ไม่มีการตรวจสอบ routes Array ก่อนเข้าถึง

Copyconst route = result.routes[0]; // ถ้า routes เป็น [] จะได้ undefined
const totalDistance = (route.distanceMeters || 0) / 1000; // TypeError!
ควรเพิ่ม Guard:

Copyif (!result.routes || result.routes.length === 0) {
  throw new Error(`Routes API returned no routes. Response: ${JSON.stringify(result)}`);
}
const route = result.routes[0];
🟡 Robustness Issue #3: ไม่มี Validation ว่า lat/lng อยู่ในช่วงที่ถูกต้อง

Copyconst lat = parseFloat(parts[0].trim());
const lng = parseFloat(parts[1].trim());
if (!isNaN(lat) && !isNaN(lng)) { // เช็คแค่ว่าเป็นตัวเลข
ควรเพิ่มการตรวจสอบ Range:

Copyconst isValidLat = lat >= -90 && lat <= 90;
const isValidLng = lng >= -180 && lng <= 180;
if (!isNaN(lat) && !isNaN(lng) && isValidLat && isValidLng) {
  // ...
} else {
  Logger.log(`⚠️ Invalid coordinate range: lat=${lat}, lng=${lng} at Row ${rowIdx}`);
}
Section 6: ฟีเจอร์ Google Maps Routes API ใหม่ที่ควรพิจารณา:
🆕 Feature #1: computeRouteMatrix API (แนะนำอย่างยิ่ง)

ปัจจุบันโค้ดดึงค่า ระยะทางจากคลัง_Km จาก Spreadsheet ที่คำนวณไว้ล่วงหน้า แต่ถ้าข้อมูลนั้นเก่าหรือผิด ผลการ Sort จะผิดด้วย Routes API มี Endpoint ใหม่ที่คำนวณระยะทางระหว่างจุดหลายๆ คู่พร้อมกัน:

CopyPOST https://routes.googleapis.com/distanceMatrix/v2:computeRouteMatrix
สามารถส่ง Origin 1 จุด (Depot) และ Destinations หลายจุด แล้วได้ระยะทางกลับมาพร้อมกันทั้งหมด ทำให้ไม่ต้องพึ่งข้อมูล Pre-computed ที่อาจเก่า

🆕 Feature #2: TRAFFIC_AWARE และ TRAFFIC_AWARE_OPTIMAL Routing Preference

โค้ดปัจจุบันใช้ TRAFFIC_UNAWARE ซึ่งเหมาะสำหรับการเบิกเงินเพราะระยะทางคงที่ แต่ถ้าต้องการระบบ ETA จริงๆ สำหรับ Driver App สามารถเพิ่ม Parameter departureTime เพื่อให้ API คำนวณด้วยข้อมูลจราจรจริง:

Copy// สำหรับ Use Case การวางแผนล่วงหน้า
const payload = {
  // ...existing fields...
  routingPreference: 'TRAFFIC_AWARE_OPTIMAL',
  departureTime: new Date(Date.now() + 3600000).toISOString() // ออกเดินทางใน 1 ชั่วโมง
};
🆕 Feature #3: X-Goog-FieldMask ที่ละเอียดขึ้น

Copy// โค้ดปัจจุบัน
'X-Goog-FieldMask': 'routes.duration,routes.distanceMeters,routes.optimizedIntermediateWaypointIndex'
สามารถเพิ่ม routes.legs เพื่อดูระยะทางแต่ละ Leg (ระหว่างจุดต่อจุด) ซึ่งมีประโยชน์มากสำหรับการตรวจสอบและ Debug:

Copy'X-Goog-FieldMask': 'routes.duration,routes.distanceMeters,routes.optimizedIntermediateWaypointIndex,routes.legs.distanceMeters,routes.legs.duration'
🆕 Feature #4: vehicleInfo และ extraComputations (2024-2025 Features)

Routes API เพิ่ม Support สำหรับ:

TOLLS - คำนวณค่าทางด่วนอัตโนมัติ
FUEL_CONSUMPTION - ประมาณการใช้น้ำมัน (ต้องระบุ vehicleInfo.emissionType)
TRAFFIC_ON_POLYLINE - แสดงสภาพจราจรบน Polyline
Copyconst payload = {
  // ...existing fields...
  extraComputations: ['TOLLS', 'FUEL_CONSUMPTION'],
  vehicleInfo: { emissionType: 'DIESEL' }
};
Section 7: สรุปคะแนนและ Action Items:
หมวด	ระดับปัญหา	คะแนน (10)
Silent Errors	🔴 สูง	5/10
Performance	🟡 กลาง	6/10
Robustness	🔴 สูง	5/10
Code Quality	🟢 ดี	8/10
Architecture	🟢 ดี	8/10
Priority 1 (แก้ทันที - Critical):

เพิ่ม Guard Check สำหรับ result.routes[0] ก่อนเข้าถึง
แก้ N+1 Read Problem ใน prepareWaypointsWithExistingDistance ให้ดึงข้อมูลเป็น Block
เพิ่ม Warning Log เมื่อ Distance Parse ล้มเหลวและ Default เป็น 0
Priority 2 (ควรแก้เร็วๆ - Important):

เพิ่ม Retry Logic ด้วย Exponential Backoff
เพิ่ม Coordinate Range Validation
เพิ่ม Log เมื่อ Skip Row เนื่องจาก Invalid Format
Cache getLastRow() ไว้ในตัวแปร
Priority 3 (ปรับปรุงระยะยาว - Enhancement):

พิจารณาใช้ computeRouteMatrix แทนการพึ่ง Pre-computed Distance
เพิ่ม routes.legs ใน FieldMask เพื่อ Debug ง่ายขึ้น
พิจารณาเพิ่ม TOLLS Computation สำหรับระบบเบิกเงินที่แม่นยำขึ้น
Section 8: โค้ดเวอร์ชันปรับปรุง (Critical Fixes Only):
Copy// ✅ FIX 1: Batch Read แทน N+1 Reads
function prepareWaypointsWithExistingDistance(ss, shipmentId) {
  const computedSheet = ss.getSheetByName(SHEET_COMPUTED);
  const lastRow = computedSheet.getLastRow();
  const lastCol = computedSheet.getLastColumn();
  
  // โหลดข้อมูลทั้งหมดครั้งเดียว (1 API Call แทน N Calls)
  const allData = computedSheet.getRange(1, 1, lastRow, lastCol).getValues();
  const header = allData[0];
  
  const shipmentColIdx = header.indexOf("Shipment No");
  const latlngColIdx = header.indexOf("จุดส่งสินค้าปลายทาง");
  const nameColIdx = header.indexOf("ชื่อปลายทาง");
  const distanceColIdx = header.indexOf(DEPOT_DISTANCE_COLUMN);
  
  if (shipmentColIdx === -1 || latlngColIdx === -1) 
    throw new Error("Missing required columns in computed sheet");
  
  const allPoints = [{
    id: 0, name: DEPOT_COORDS.name,
    original: { lat: DEPOT_COORDS.lat, lng: DEPOT_COORDS.lng },
    forApi: { location: { latLng: { latitude: DEPOT_COORDS.lat, longitude: DEPOT_COORDS.lng } } },
    distance: 0, isDepot: true
  }];
  
  let idCounter = 1;
  const targetShipment = String(shipmentId).trim();
  
  // วนลูปใน Memory (ไม่มี API Call เพิ่มเติม)
  for (let i = 1; i < allData.length; i++) {
    const row = allData[i];
    if (String(row[shipmentColIdx]).trim() !== targetShipment) continue;
    
    const latlngRaw = row[latlngColIdx];
    if (!latlngRaw) {
      Logger.log(`⚠️ SKIP Row ${i + 1}: Empty coordinate`);
      continue;
    }
    
    const cleanedLatLong = String(latlngRaw).trim().replace(/[^\d.,-]/g, "");
    const parts = cleanedLatLong.split(",");
    
    if (parts.length !== 2) {
      Logger.log(`⚠️ SKIP Row ${i + 1}: Invalid format "${latlngRaw}"`);
      continue;
    }
    
    const lat = parseFloat(parts[0].trim());
    const lng = parseFloat(parts[1].trim());
    
    if (isNaN(lat) || isNaN(lng) || lat < -90 || lat > 90 || lng < -180 || lng > 180) {
      Logger.log(`⚠️ SKIP Row ${i + 1}: Invalid coordinates lat=${lat}, lng=${lng}`);
      continue;
    }
    
    const rawDistance = distanceColIdx >= 0 ? row[distanceColIdx] : null;
    const distance = parseFloat(rawDistance);
    if (isNaN(distance)) {
      Logger.log(`⚠️ WARNING Row ${i + 1}: Distance "${rawDistance}" invalid → using 0. Sort may be affected!`);
    }
    
    const name = (nameColIdx >= 0 && row[nameColIdx]) 
      ? String(row[nameColIdx]).trim() 
      : `Point ${idCounter}`;
    
    allPoints.push({
      id: idCounter++, name,
      original: { lat, lng },
      forApi: { location: { latLng: { latitude: lat, longitude: lng } } },
      distance: isNaN(distance) ? 0 : distance,
      isDepot: false
    });
  }
  
  return allPoints;
}

// ✅ FIX 2: Guard Check + Retry สำหรับ API
function executeGoogleMapsRoutesAPIOneWay(allPoints) {
  const GOOGLE_MAPS_API_KEY = PropertiesService.getScriptProperties().getProperty("GOOGLE_MAPS_API_KEY");
  if (!GOOGLE_MAPS_API_KEY) throw new Error("Google Maps API Key not set in Script Properties");
  
  const origin = allPoints[0];
  const finalDestination = allPoints[allPoints.length - 1];
  const intermediates = allPoints.slice(1, -1);
  const shouldOptimize = intermediates.length > 1;

  const payload = {
    origin: origin.forApi,
    destination: finalDestination.forApi,
    intermediates: intermediates.map(p => p.forApi),
    travelMode: 'DRIVE',
    routingPreference: 'TRAFFIC_UNAWARE',
    optimizeWaypointOrder: shouldOptimize
  };
  
  const apiUrl = "https://routes.googleapis.com/directions/v2:computeRoutes";
  const options = {
    method: 'post', contentType: 'application/json',
    headers: {
      'X-Goog-Api-Key': GOOGLE_MAPS_API_KEY,
      'X-Goog-FieldMask': 'routes.duration,routes.distanceMeters,routes.optimizedIntermediateWaypointIndex,routes.legs.distanceMeters'
    },
    payload: JSON.stringify(payload),
    muteHttpExceptions: true
  };
  
  // Retry Logic
  let response;
  for (let attempt = 1; attempt <= 3; attempt++) {
    response = UrlFetchApp.fetch(apiUrl, options);
    if (response.getResponseCode() === 200) break;
    if (attempt < 3) {
      Logger.log(`⚠️ API attempt ${attempt} failed (${response.getResponseCode()}), retrying...`);
      Utilities.sleep(Math.pow(2, attempt) * 1000);
    } else {
      throw new Error(`API Error after 3 attempts: ${response.getContentText()}`);
    }
  }
  
  const result = JSON.parse(response.getContentText());
  
  // ✅ Guard Check ก่อนเข้าถึง routes[0]
  if (!result.routes || result.routes.length === 0) {
    throw new Error(`Routes API returned no routes. Full response: ${JSON.stringify(result)}`);
  }
  
  const route = result.routes[0];
  const totalDistance = (route.distanceMeters || 0) / 1000;
  
  const orderedWaypoints = [origin];
  if (shouldOptimize && route.optimizedIntermediateWaypointIndex?.length > 0) {
    route.optimizedIntermediateWaypointIndex.forEach(index => {
      if (intermediates[index]) {
        orderedWaypoints.push(intermediates[index]);
      } else {
        Logger.log(`⚠️ WARNING: Optimized index ${index} out of bounds (intermediates.length=${intermediates.length})`);
      }
    });
    // Fallback ถ้า Optimization ให้ Index ไม่ครบ
    if (orderedWaypoints.length - 1 < intermediates.length) {
      Logger.log(`⚠️ WARNING: Optimization returned ${orderedWaypoints.length - 1} points but expected ${intermediates.length}. Using original order.`);
      orderedWaypoints.splice(1); // ล้าง intermediates ที่เพิ่มไป
      orderedWaypoints.push(...intermediates);
    }
  } else {
    orderedWaypoints.push(...intermediates);
  }
  orderedWaypoints.push(finalDestination);
  
  const linkCoordinates = orderedWaypoints.map(p => `${p.original.lat.toFixed(6)},${p.original.lng.toFixed(6)}`);
  const googleMapsLink = encodeURI(`https://www.google.com/maps/dir/${linkCoordinates.join('/')}`);
  
  return { orderedWaypoints, totalDistance, googleMapsLink };
}
Copy
โดยรวมแล้วโค้ดชุดนี้มีโครงสร้างที่ดีมาก มีการแบ่ง Concern ชัดเจน และใช้ TextFinder ได้อย่างชาญฉลาด แต่ปัญหาหลักที่ต้องแก้ด่วนคือ N+1 Read Problem และการขาด Guard Check บน API Response ซึ่งอาจทำให้ระบบล่มกลางคันโดยไม่มีสาเหตุชัดเจนในสภาพแวดล้อม Production ครับ