Reflection
ภาพรวมการประเมินคำตอบทั้งสามรุ่น

ทั้งสามคำตอบ (gpt-5.1-low, claude-sonnet-4-6, gemini-3.1-pro-preview) วิเคราะห์โค้ด Google Apps Script (GAS) ชุดนี้ในทิศทางที่ตรงกันเป็นส่วนใหญ่ คือชื่นชมโครงสร้างโค้ดว่าดี มีการแบ่ง concern ชัดเจน ใช้ TextFinder และ Batch Writing ซึ่งเป็น best practice ของ GAS แต่แต่ละคำตอบมีจุดเด่นและจุดบอดต่างกัน ซึ่งต้องนำมากลั่นกรองรวมกันเพื่อให้ได้คำตอบที่ครบถ้วนและถูกต้องที่สุด

คำตอบที่ 1 (gpt-5.1-low)
จุดแข็งคือการไล่โค้ดทีละบรรทัดอย่างละเอียดพร้อมอธิบายเหตุผล ทำให้ผู้อ่านเข้าใจ "ทำไม" โค้ดแต่ละส่วนถึงเขียนแบบนี้ได้ง่าย ระบุปัญหาหลักได้ถูกต้องคือ (1) การอ่านข้อมูลทีละเซลล์ใน loop (N+1 read) และเสนอโค้ดแก้ไขที่ใช้การอ่านเป็น block ตาม min-max row ซึ่งเป็นแนวทางที่ใช้ได้จริง (2) การไม่ตรวจสอบ result.routes ก่อนเข้าถึง index 0 (3) เสนอ refactor เรื่อง header-index-map และ formatLatLng helper เพื่อลดโค้ดซ้ำซ้อน ซึ่งเป็นจุดที่คำตอบอื่นพูดถึงน้อยกว่า และ (4) ให้ข้อมูลฟีเจอร์ Routes API ใหม่อย่าง polyline, routeModifiers, TRAFFIC_AWARE_OPTIMAL ได้ถูกต้อง จุดที่ขาดไปคือไม่ได้พูดถึง retry logic, การตรวจสอบขอบเขต lat/lng, และฟีเจอร์ระดับสูงอย่าง computeRouteMatrix หรือ extraComputations (TOLLS/FUEL_CONSUMPTION) ซึ่งเป็นฟีเจอร์ใหม่ที่มีประโยชน์ตรงกับ use case นี้

คำตอบที่ 2 (claude-sonnet-4-6)
เป็นคำตอบที่ลึกและครอบคลุมที่สุดในเชิงเทคนิค ระบุ Silent Error ได้ครบทั้ง 4 กรณีสำคัญ (regex/split edge case, distance parse NaN → 0 ทำให้ sort ผิด, optimizedIntermediateWaypointIndex out-of-bounds ถูกข้ามเงียบๆ, N+1 read) พร้อมโค้ดแก้ไขที่ใช้งานได้จริงและสมบูรณ์กว่าคำตอบอื่น (รวม batch read + guard check + retry logic ในไฟล์เดียว) นอกจากนี้ยังชี้จุดที่คำตอบอื่นไม่ได้พูดถึงคือ ฟีเจอร์ computeRouteMatrix (Distance Matrix v2) ซึ่งตอบโจทย์ปัญหาการพึ่งพาค่าระยะทางที่คำนวณไว้ล่วงหน้าในชีตซึ่งอาจล้าสมัยหรือผิดพลาดได้ตรงจุดที่สุด รวมถึงฟีเจอร์ระดับสูงอย่าง extraComputations (TOLLS, FUEL_CONSUMPTION) และ vehicleInfo.emissionType ซึ่งเป็นฟีเจอร์ที่ค่อนข้างใหม่และถูกต้องตามเอกสาร Google จุดที่ควรระวังคือตารางให้คะแนน (Score Table) ในคำตอบนี้เป็นการประเมินแบบอัตวิสัย ไม่มีเกณฑ์อ้างอิงที่ชัดเจน ไม่ควรนำเสนอเป็นข้อเท็จจริงตายตัว และตัวอย่าง Silent Error #1 เกี่ยวกับ regex ค่อนข้าง contrived (สถานการณ์ที่ยกมาไม่ใช่ปัญหาจริงในทางปฏิบัติทั่วไป) แต่ประเด็นหลักเรื่องไม่ log เมื่อ skip ยังคงถูกต้องและสำคัญ

คำตอบที่ 3 (gemini-3.1-pro-preview)
กระชับกว่าสองคำตอบแรก แต่มีมุมมองเฉพาะที่มีคุณค่า คือการตั้งคำถามกับ business logic ของการบังคับ "จุดไกลสุด = ปลายทางสุดท้ายเสมอ" และเสนอแนวทางเลือกคือการทำ Round Trip แล้วให้ Google optimize ทั้งหมดเอง ซึ่งเป็นมุมมองที่ลึกกว่าคำตอบอื่นในเชิงธุรกิจ นอกจากนี้ยังพูดถึงขีดจำกัดจำนวน intermediate waypoints สำหรับการ optimize (ประมาณ 25 จุด) ซึ่งเป็นข้อมูลที่มีประโยชน์แต่ควรระบุว่าควรตรวจสอบกับเอกสารทางการล่าสุดเพราะ Google อาจเปลี่ยนแปลงได้ อย่างไรก็ตาม จุดอ่อนสำคัญของคำตอบนี้คือ ไม่ได้ระบุปัญหา N+1 read ในฟังก์ชัน prepareWaypointsWithExistingDistance อย่างชัดเจน ทั้งที่เป็นปัญหาประสิทธิภาพหลักของโค้ดชุดนี้ กลับไปชื่นชมว่า TextFinder ทำงานได้ดีโดยไม่ได้สังเกตว่าภายใน loop ยังมีการเรียก getRange().getValue() ทีละเซลล์อยู่ ถือเป็นข้อผิดพลาดในการวิเคราะห์ที่ควรแก้ไขในคำตอบสุดท้าย

การตรวจสอบความถูกต้องทางเทคนิค (Verification)

ผู้สังเคราะห์ได้ไล่โค้ดจริงทีละฟังก์ชันเพื่อตรวจสอบว่าข้อกล่าวหาต่าง ๆ ถูกต้องหรือไม่ พบว่า:

N+1 read เป็นปัญหาจริง เพราะใน matches.forEach มีการเรียก computedSheet.getRange(rowIdx, ...).getValue() แยกกันถึง 1-3 ครั้งต่อแถวที่ match ซึ่งยืนยันได้ตรงตามคำตอบของ gpt-5.1 และ Claude
การไม่ตรวจสอบ result.routes ก่อนเข้าถึง [0] เป็นความเสี่ยงจริง เพราะถ้า Google Routes API คืนค่า routes: [] (เช่นหาทางไม่ได้) จะเกิด TypeError ที่ไม่มีข้อความสื่อความหมาย แม้จะถูก catch ที่ชั้นบนแต่ debug ยาก
distance = ... || 0 เมื่อ parseFloat ได้ NaN จะกลายเป็น 0 จริง ซึ่งกระทบ logic การเรียงลำดับปลายทางโดยไม่มีการเตือนใด ๆ — ยืนยันตามคำตอบของ Claude และ Gemini
การข้าม (skip) พิกัดที่ parse ไม่ได้โดยไม่มี Logger.log เป็นจริงตามโค้ด — ยืนยันตามทั้งสามคำตอบ
ประเด็นเรื่อง customersOnly = orderedPoints.slice(1) ตัด depot ออกจากคอลัมน์ 20 ช่อง แต่ resultString (คอลัมน์หลัก) ยังรวม depot ไว้ เป็นข้อสังเกตที่ถูกต้องตามโค้ดจริง (Claude เป็นคนเดียวที่จับจุดนี้ได้) แต่ควรระบุว่าอาจเป็นการออกแบบตั้งใจ ไม่ใช่บั๊กแน่นอน ต้องยืนยันกับความต้องการทางธุรกิจ
ฟีเจอร์ computeRouteMatrix, extraComputations (TOLLS/FUEL_CONSUMPTION), routeModifiers, polyline.encodedPolyline ตรวจสอบแล้วว่ามีอยู่จริงในเอกสาร Google Maps Routes API ปัจจุบัน ถือเป็นข้อมูลที่ถูกต้องและมีประโยชน์
การสังเคราะห์

ไม่มีคำตอบใดผิดพลาดในเชิงเทคนิคอย่างร้ายแรง แต่ต่างกันในเรื่อง "ความครบถ้วน" — Claude ครบถ้วนที่สุดในเชิง Silent Error/Robustness/API ใหม่, gpt-5.1 แข็งในเรื่อง refactor/DRY code และอธิบายละเอียดทีละบรรทัด, Gemini แข็งในมุมมอง business logic แต่พลาดประเด็น performance หลัก ดังนั้นคำตอบสุดท้ายควรรวมจุดแข็งทั้งหมดเข้าด้วยกัน โดยเน้น N+1 read fix, guard check, silent error logging, business logic caveat, DRY refactor, retry logic, และฟีเจอร์ API ใหม่ (โดยเฉพาะ computeRouteMatrix ที่แก้ปัญหาที่มาของ distance ได้ตรงจุด) พร้อมจัดลำดับความสำคัญ (Priority) ให้ผู้ใช้นำไปปรับใช้ได้จริง

ภาพรวมการทำงานของสคริปต์

สคริปต์นี้เป็น Google Apps Script ที่ทำงานผูกกับ Google Sheets เพื่อคำนวณเส้นทางการขนส่งของ SCG-JWD วังน้อยสำหรับใช้ในการเบิกค่าใช้จ่าย โดย flow การทำงานทั้งหมดคือ: รับ shipmentId และ rowId → ค้นหาจุดส่งสินค้าทั้งหมดของ shipment นั้นจากชีตข้อมูล (SHEET_COMPUTED) โดยใช้ TextFinder → เลือก "จุดที่ไกลจากคลังที่สุด" ให้เป็นปลายทางสุดท้ายโดยอ้างอิงค่าระยะทางที่คำนวณไว้ล่วงหน้าในชีต → ยิง Google Maps Routes API (endpoint รุ่นใหม่ computeRoutes) แบบ TRAFFIC_UNAWARE เพื่อให้ระยะทางคงที่เหมาะกับการเบิกเงิน → เขียนผลลัพธ์ (สตริงพิกัดรวม, ระยะทาง, ลิงก์แผนที่, และพิกัดแยกคอลัมน์ Lat/Long_ปลายทาง_01-20) กลับลงชีตผลลัพธ์ (SHEET_RESULT) โดยรวมแล้วโครงสร้างโค้ดจัดเป็นระดับ Pro/Senior เพราะแบ่ง concern ชัดเจนเป็น 5 ส่วนตามคอมเมนต์ในไฟล์

วิเคราะห์การทำงานทีละฟังก์ชัน

ฟังก์ชัน findOptimalRouteUsingExistingDistance ทำหน้าที่เป็น orchestrator หลัก ควบคุมลำดับการทำงานทั้งหมดภายใต้ try...catch เดียว ซึ่งทำให้ error ทุกจุดในกระบวนการถูกจับและ log ได้ครบ ถือเป็นแนวทางที่ดีสำหรับ entry point

ฟังก์ชัน prepareWaypointsWithExistingDistance เป็นฟังก์ชันที่ทำงานหนักที่สุด โดยอ่าน header แถวแรกเพื่อหา index คอลัมน์แบบ dynamic (ไม่ hardcode เลขคอลัมน์ ซึ่งดีมากในเชิง maintainability) จากนั้นใช้ TextFinder ค้นหาเฉพาะในคอลัมน์ "Shipment No" เท่านั้น (ไม่ใช่ทั้งชีต) ทำให้การค้นหาเร็วมาก แต่ปัญหาสำคัญคือ หลังจากเจอแถวที่ match แล้ว โค้ดกลับไปเรียก getRange(rowIdx, ...).getValue() แยกทีละเซลล์สำหรับพิกัด ชื่อ และระยะทาง ซึ่งขัดกับหลักการลด network call ของ Apps Script โดยสิ้นเชิง ถ้า shipment หนึ่งมี 50 จุดส่ง จะเกิด network call ไปยัง Spreadsheet Service ถึง 100-150 ครั้ง ซึ่งเป็นสาเหตุหลักที่ทำให้สคริปต์ช้าลงเมื่อข้อมูลมาก และเสี่ยงชน time limit ของ Apps Script (ปกติ 6 นาทีสำหรับบัญชีทั่วไป)

ฟังก์ชัน selectFinalDestinationAndSort ใช้ business rule ที่ว่า "จุดที่ไกลจากคลังที่สุดต้องเป็นปลายทางสุดท้ายเสมอ" โดยอิงจากค่า ระยะทางจากคลัง_Km ที่มีอยู่แล้วในชีต แล้วเรียงจุดที่เหลือเป็น intermediate ตรงนี้มีข้อสังเกตสำคัญสองเรื่อง คือ (1) ถ้าค่าระยะทางในชีตผิดพลาด ล้าสมัย หรือเป็นค่าประมาณแบบเส้นตรง (ไม่ใช่ระยะทางถนนจริง) การเลือกปลายทางจะผิดไปด้วยโดยไม่มีการเตือนใด ๆ และ (2) การบังคับปลายทางตายตัวเช่นนี้อาจทำให้ Google ไม่สามารถหาลำดับที่สั้นที่สุดจริง ๆ ได้ เพราะถูกจำกัดว่าต้องจบที่จุดนี้เสมอ แม้ในทางภูมิศาสตร์จุดกลางบางจุดอาจอยู่ใกล้ปลายทางมากกว่าที่คิด — อย่างไรก็ตาม การออกแบบเช่นนี้อาจเป็นความตั้งใจทางธุรกิจ (เช่น ต้องการยืนยันจุดที่ไกลสุดเพื่อการเบิกจ่ายที่ตรวจสอบได้) จึงควรยืนยันกับผู้ที่ดูแลกระบวนการเบิกจ่ายว่าตรงกับความต้องการจริงหรือไม่

ฟังก์ชัน executeGoogleMapsRoutesAPIOneWay ดึง API Key จาก PropertiesService ซึ่งปลอดภัยกว่าการ hardcode ในโค้ด จากนั้นสร้าง payload ส่งไปยัง endpoint รุ่นใหม่ https://routes.googleapis.com/directions/v2:computeRoutes พร้อม FieldMask ที่จำกัดเฉพาะฟิลด์ที่ใช้จริง (duration, distanceMeters, optimizedIntermediateWaypointIndex) ซึ่งช่วยประหยัด quota และเวลาตอบสนอง จุดที่ต้องระวังคือ โค้ดเข้าถึง result.routes[0] ทันทีโดยไม่ตรวจสอบก่อนว่ามี route กลับมาจริงหรือไม่ ถ้า API ตอบกลับด้วย routes: [] (เช่นพิกัดผิดพลาดจนหาทางไม่ได้) จะเกิด TypeError ที่ไม่มีข้อความสื่อความหมาย นอกจากนี้เมื่อนำ optimizedIntermediateWaypointIndex มาจับคู่กับ intermediates[index] หากมี index ที่ชี้ไปนอกขอบเขต array จุดนั้นจะถูกข้ามไปเงียบ ๆ ทำให้จำนวนจุดในเส้นทางสุดท้ายน้อยกว่าที่ควรโดยไม่มีการแจ้งเตือน

ฟังก์ชัน writeResultsToSheet ใช้ TextFinder หาแถวที่ต้องเขียนผลลัพธ์อย่างมีประสิทธิภาพ และเขียนค่าหลักทีละเซลล์เพื่อหลีกเลี่ยงการเขียนทับสูตรอื่นในชีต ซึ่งเป็นแนวทางที่ปลอดภัยดี ส่วนคอลัมน์ Lat/Long_ปลายทาง_01-20 ใช้การเขียนแบบ batch (setValues ครั้งเดียว 20 ช่อง) ซึ่งเป็นจุดแข็งที่ชัดเจนของโค้ดนี้ อย่างไรก็ตามมีข้อสังเกตเล็กน้อยคือ resultString (คอลัมน์หลัก) รวมพิกัดคลังไว้ด้วย ในขณะที่ customersOnly ที่ใช้เขียน 20 คอลัมน์ตัดคลังออก (slice(1)) ซึ่งอาจตั้งใจให้ต่างกันตามจุดประสงค์ของแต่ละคอลัมน์ แต่ควรตรวจสอบยืนยันให้แน่ใจว่าตรงกับที่ผู้ใช้งานปลายทางต้องการ

จุดแข็งของโค้ด (สิ่งที่ทำได้ดีอยู่แล้ว)

โค้ดชุดนี้มีจุดแข็งที่ควรค่าแก่การชื่นชมหลายประการ การใช้ TextFinder แบบเจาะจงเฉพาะคอลัมน์ที่ต้องค้นหา (เช่น ค้นเฉพาะคอลัมน์ "Shipment No" หรือคอลัมน์ ID) แทนการดึงข้อมูลทั้งชีตมาวนลูปเป็นวิธีที่เร็วที่สุดใน Apps Script การใช้ Batch Writing สำหรับคอลัมน์ Lat/Long 20 ช่องด้วย setValues([valuesToWrite]) ครั้งเดียวช่วยลดจำนวน network call ได้มาก การเก็บ API Key ผ่าน PropertiesService ถือเป็นแนวทางความปลอดภัยที่ถูกต้อง การเลือกใช้ endpoint computeRoutes (Directions API v2) ซึ่งเป็นเวอร์ชันล่าสุดที่ Google แนะนำให้ใช้แทน Directions API รุ่นเก่า รวมถึงการจำกัด X-Goog-FieldMask ให้ดึงเฉพาะฟิลด์ที่ใช้จริงเพื่อประหยัด quota ล้วนเป็นแนวทางที่ถูกต้องตามมาตรฐาน และการเขียนผลลัพธ์หลักทีละเซลล์ (ไม่ใช่เขียนทั้งแถว) ช่วยป้องกันการเขียนทับสูตรคำนวณอื่น ๆ ที่อาจมีอยู่ในชีตผลลัพธ์

ปัญหา Silent Error ที่ต้องระวัง

ปัญหาที่อันตรายที่สุดของโค้ดชุดนี้คือจุดที่ระบบ "ทำงานต่อไปได้โดยไม่ error แต่ผลลัพธ์อาจผิดพลาด" ซึ่งพบทั้งหมด 6 จุดสำคัญดังนี้

จุดแรกคือการข้ามพิกัดที่ parse ไม่ได้โดยไม่มีการบันทึก log ใด ๆ ในฟังก์ชัน prepareWaypointsWithExistingDistance หากค่าพิกัดว่าง, format ผิด (เช่น split แล้วได้มากกว่า 2 ส่วน), หรือ isNaN จะถูก return/skip ไปเฉย ๆ ทำให้พนักงานไม่รู้ว่ามีจุดส่งของบางจุดหายไปจากเส้นทางโดยไม่รู้ตัว ซึ่งกระทบต่อความถูกต้องของค่าเบิกจ่ายโดยตรง

จุดที่สองคือการ default ค่าระยะทางเป็น 0 เมื่อ parse ไม่สำเร็จ ผ่านโค้ด parseFloat(...) || 0 หากค่าในเซลล์เป็นค่าว่างหรือ string ที่ parse ไม่ได้ (เช่น "N/A") ระบบจะถือว่าจุดนั้นมีระยะทาง 0 กม. จากคลัง ซึ่งจะทำให้จุดนั้นถูกจัดลำดับไปอยู่ท้ายสุดของการ sort อาจกลายเป็นการเลือกปลายทางที่ผิดพลาดโดยไม่มีการเตือนเลย

จุดที่สามคือการไม่ตรวจสอบ result.routes ก่อนเข้าถึง index [0] ในฟังก์ชัน executeGoogleMapsRoutesAPIOneWay หาก Google API ตอบกลับด้วย array ว่าง (ไม่พบเส้นทาง) จะเกิด TypeError ที่ไม่มีข้อความสื่อความหมาย แม้จะถูกจับใน try...catch ชั้นบนแต่การ debug จะยากเพราะไม่รู้สาเหตุที่แท้จริง

จุดที่สี่คือการจับคู่ optimizedIntermediateWaypointIndex กับ intermediates[index] โดยไม่ตรวจสอบขอบเขต หาก index ที่ Google ส่งกลับมาชี้ไปยังตำแหน่งที่ไม่มีอยู่จริง จุดนั้นจะถูกข้ามไปเงียบ ๆ ทำให้จำนวนจุดในเส้นทางสุดท้ายน้อยกว่าที่ควรเป็น

จุดที่ห้าคือการไม่ตรวจสอบขอบเขตของค่า latitude (-90 ถึง 90) และ longitude (-180 ถึง 180) ทำให้ค่าพิกัดที่เป็นตัวเลขแต่ไม่สมเหตุสมผลทางภูมิศาสตร์ (เช่นพิมพ์ผิดตำแหน่งทศนิยม) จะผ่านการตรวจสอบไปได้โดยไม่มีการเตือน

จุดที่หกคือการ N+1 read ในฟังก์ชันเตรียมข้อมูล ซึ่งไม่ใช่ error ที่มองเห็นได้ทันที แต่เป็น "ปัญหาเงียบ" ในเชิงประสิทธิภาพที่จะแสดงผลชัดเมื่อข้อมูลมากขึ้นเรื่อย ๆ จนอาจทำให้สคริปต์ทำงานช้าเกินไปหรือ timeout โดยไม่มีสัญญาณเตือนล่วงหน้า

ปัญหาด้านประสิทธิภาพ (Performance)

ปัญหาหลักที่ควรแก้ไขเร่งด่วนที่สุดคือ N+1 Read Problem ในฟังก์ชัน prepareWaypointsWithExistingDistance ที่เรียก getRange().getValue() ทีละเซลล์ภายใน loop ของแต่ละ match ถ้ามี 10 จุดส่ง อาจเกิด network call ถึง 30 ครั้ง และถ้ามี 50 จุดส่งอาจถึง 150 ครั้ง ซึ่งควรแก้ด้วยการอ่านข้อมูลทั้งบล็อกในครั้งเดียว (batch read) แล้วนำมากรองในหน่วยความจำ (in-memory filtering) แทน

นอกจากนี้ getLastRow() ถูกเรียกซ้ำหลายครั้งในหลายฟังก์ชัน (ทั้งใน prepareWaypointsWithExistingDistance และ writeResultsToSheet) ซึ่งแต่ละครั้งเป็นการเรียก Spreadsheet Service เพิ่มเติม ควร cache ค่าไว้ในตัวแปรก่อนใช้งานซ้ำ และในกรณีที่มีจุดส่งจำนวนมาก (15-20 จุดขึ้นไป) ลิงก์ Google Maps ที่สร้างขึ้นจากการต่อพิกัดทั้งหมดอาจยาวเกินขีดจำกัดที่บางเบราว์เซอร์หรือแอปพลิเคชันรองรับ (ประมาณ 2,000 ตัวอักษร) ทำให้ลิงก์เปิดไม่ได้หรือข้อมูลขาดหาย ควรพิจารณาจำกัดจำนวนจุดในลิงก์หรือหาวิธีย่อ URL

โค้ดซ้ำซ้อนและจุดที่ควร Refactor

แม้โค้ดโดยรวมไม่ได้ซ้ำซ้อนมาก แต่มี 2 pattern ที่ทำซ้ำในหลายฟังก์ชันซึ่งควรแยกเป็น helper function เพื่อลดโอกาสเกิด bug และง่ายต่อการดูแลรักษาในระยะยาว

รูปแบบแรกคือการหา index คอลัมน์จาก header ซึ่งทำซ้ำใน prepareWaypointsWithExistingDistance และ writeResultsToSheet ควรแยกเป็นฟังก์ชันกลาง เช่น

Copyfunction getHeaderIndexMap(sheet) {
  const header = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
  const map = {};
  header.forEach((name, idx) => { map[name] = idx + 1; }); // +1 สำหรับ getRange (1-based)
  return map;
}
รูปแบบที่สองคือการแปลงพิกัดเป็น string "lat,lng" ซึ่งทำซ้ำในหลายที่ (ตอนสร้าง resultString, ตอนสร้างลิงก์, ตอนเขียน 20 คอลัมน์) ควรแยกเป็น helper เดียว

Copyfunction formatLatLng(point, withSpace) {
  const sep = withSpace ? ', ' : ',';
  return `${point.original.lat.toFixed(6)}${sep}${point.original.lng.toFixed(6)}`;
}
โค้ดตัวอย่างสำหรับแก้ไขจุดสำคัญ (Critical Fixes)

ต่อไปนี้เป็นตัวอย่างการแก้ไขที่รวมจุดสำคัญที่สุดเข้าด้วยกัน คือการอ่านข้อมูลแบบ batch แทน N+1, การเพิ่ม log เมื่อ skip ข้อมูล, และการตรวจสอบ routes ก่อนใช้งาน

Copyfunction prepareWaypointsWithExistingDistance(ss, shipmentId) {
  const computedSheet = ss.getSheetByName(SHEET_COMPUTED);
  const lastRow = computedSheet.getLastRow();
  const lastCol = computedSheet.getLastColumn();

  // อ่านข้อมูลทั้งหมดครั้งเดียว (1 network call แทน N calls)
  const allData = computedSheet.getRange(1, 1, lastRow, lastCol).getValues();
  const header = allData[0];

  const shipmentColIdx = header.indexOf("Shipment No");
  const latlngColIdx = header.indexOf("จุดส่งสินค้าปลายทาง");
  const nameColIdx = header.indexOf("ชื่อปลายทาง");
  const distanceColIdx = header.indexOf(DEPOT_DISTANCE_COLUMN);

  if (shipmentColIdx === -1 || latlngColIdx === -1) {
    throw new Error("Missing required columns in computed sheet");
  }

  const allPoints = [{
    id: 0, name: DEPOT_COORDS.name,
    original: { lat: DEPOT_COORDS.lat, lng: DEPOT_COORDS.lng },
    forApi: { location: { latLng: { latitude: DEPOT_COORDS.lat, longitude: DEPOT_COORDS.lng } } },
    distance: 0, isDepot: true
  }];

  let idCounter = 1;
  const targetShipment = String(shipmentId).trim();

  for (let i = 1; i < allData.length; i++) {
    const row = allData[i];
    if (String(row[shipmentColIdx]).trim() !== targetShipment) continue;

    const latlngRaw = row[latlngColIdx];
    if (!latlngRaw) {
      Logger.log(`⚠️ SKIP Row ${i + 1}: พิกัดว่าง`);
      continue;
    }

    const cleaned = String(latlngRaw).trim().replace(/[^\d.,-]/g, "");
    const parts = cleaned.split(",");
    if (parts.length !== 2) {
      Logger.log(`⚠️ SKIP Row ${i + 1}: รูปแบบพิกัดผิด "${latlngRaw}"`);
      continue;
    }

    const lat = parseFloat(parts[0].trim());
    const lng = parseFloat(parts[1].trim());
    const validRange = lat >= -90 && lat <= 90 && lng >= -180 && lng <= 180;

    if (isNaN(lat) || isNaN(lng) || !validRange) {
      Logger.log(`⚠️ SKIP Row ${i + 1}: พิกัดไม่ถูกต้อง lat=${lat}, lng=${lng}`);
      continue;
    }

    const rawDistance = distanceColIdx >= 0 ? row[distanceColIdx] : null;
    const distance = parseFloat(rawDistance);
    if (isNaN(distance)) {
      Logger.log(`⚠️ WARNING Row ${i + 1}: ระยะทาง "${rawDistance}" ไม่ถูกต้อง → ใช้ 0 (อาจกระทบการเรียงลำดับ)`);
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
Copy
และสำหรับฟังก์ชันเรียก API ควรเพิ่ม guard check และ retry logic ดังนี้

Copyfunction fetchRoutesApiWithRetry(apiUrl, options, maxRetries) {
  let response;
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    response = UrlFetchApp.fetch(apiUrl, options);
    const code = response.getResponseCode();
    if (code === 200) return response;
    if (code === 429 || code >= 500) {
      if (attempt < maxRetries) {
        const waitMs = Math.pow(2, attempt) * 1000;
        Logger.log(`⚠️ API Error ${code}, Retry ${attempt}/${maxRetries} หลัง ${waitMs}ms`);
        Utilities.sleep(waitMs);
        continue;
      }
    }
    throw new Error(`API Error ${code}: ${response.getContentText()}`);
  }
  throw new Error(`API failed after ${maxRetries} retries`);
}

// ในฟังก์ชัน executeGoogleMapsRoutesAPIOneWay แทนที่การเรียก fetch เดิม:
const response = fetchRoutesApiWithRetry(apiUrl, options, 3);
const result = JSON.parse(response.getContentText());

if (!result.routes || result.routes.length === 0) {
  throw new Error(`Routes API ไม่พบเส้นทาง. Response: ${response.getContentText()}`);
}
const route = result.routes[0];
ฟีเจอร์ Google Maps Routes API ใหม่ที่น่าสนใจ

โค้ดปัจจุบันใช้ endpoint computeRoutes (Directions API v2) ซึ่งเป็นเวอร์ชันล่าสุดและถูกต้องแล้ว แต่ยังมีฟีเจอร์เพิ่มเติมที่น่านำมาพิจารณาใช้งานตามความเหมาะสม

ฟีเจอร์แรกที่ควรพิจารณามากที่สุดคือ computeRouteMatrix (Distance Matrix v2) ที่ endpoint https://routes.googleapis.com/distanceMatrix/v2:computeRouteMatrix ซึ่งสามารถส่ง origin (คลัง) 1 จุดไปพร้อมกับ destination หลายจุดในครั้งเดียว แล้วได้ระยะทางจริงจากถนนกลับมาทั้งหมด ฟีเจอร์นี้จะช่วยแก้ปัญหารากที่แท้จริงของ business logic ในโค้ดนี้ คือการที่ระบบพึ่งพาคอลัมน์ ระยะทางจากคลัง_Km ที่คำนวณไว้ล่วงหน้าในชีต ซึ่งอาจล้าสมัยหรือเป็นค่าประมาณ หากใช้ computeRouteMatrix คำนวณสดทุกครั้งจะได้ค่าที่แม่นยำกว่าสำหรับใช้เลือกปลายทางสุดท้าย

ฟีเจอร์ที่สองคือการเพิ่ม routes.polyline.encodedPolyline ใน FieldMask หากในอนาคตต้องการวาดเส้นทางบนแผนที่ในเว็บแอปหรือ Google My Maps จะสามารถนำ encoded polyline นี้ไปใช้ได้ทันที

ฟีเจอร์ที่สามคือ routeModifiers เช่น avoidTolls, avoidHighways, avoidFerries ซึ่งมีประโยชน์มากถ้ารถบรรทุกที่ใช้จริงมีข้อจำกัดเรื่องขนาด/น้ำหนักที่ขึ้นทางด่วนไม่ได้ จะช่วยให้ระยะทางที่คำนวณตรงกับพฤติกรรมการขับจริงมากขึ้น ซึ่งส่งผลตรงต่อความถูกต้องของค่าเบิกจ่าย

ฟีเจอร์ที่สี่คือการรองรับสองโหมดคู่กัน คือคง TRAFFIC_UNAWARE ไว้สำหรับการเบิกเงิน (เพื่อความคงที่ของระยะทาง) แต่เพิ่มโหมด TRAFFIC_AWARE_OPTIMAL พร้อมพารามิเตอร์ departureTime สำหรับ use case อื่นที่ต้องการ ETA จริง เช่นแอปติดตามคนขับ โดยแยกเป็นฟังก์ชันหรือ parameter ทางเลือก

ฟีเจอร์ที่ห้าคือฟีเจอร์ระดับสูงที่ Google เพิ่มเข้ามาในปีหลัง ๆ ได้แก่ extraComputations ที่รองรับค่า TOLLS (ประมาณค่าทางด่วน), FUEL_CONSUMPTION (ประมาณการใช้น้ำมันโดยต้องระบุ vehicleInfo.emissionType), และ TRAFFIC_ON_POLYLINE ฟีเจอร์เหล่านี้มีประโยชน์มากสำหรับระบบเบิกจ่ายที่ต้องการความแม่นยำเรื่องต้นทุนมากขึ้น แต่ควรตรวจสอบว่าโปรเจกต์ Google Cloud ของคุณเปิดใช้งานผลิตภัณฑ์ที่เกี่ยวข้องและมีรูปแบบการเก็บค่าบริการที่เหมาะสมก่อนนำไปใช้จริง

ฟีเจอร์ที่หกคือการเพิ่ม routes.legs.distanceMeters และ routes.legs.duration ใน FieldMask เพื่อดูระยะทางและเวลาแยกเป็นช่วง ๆ ระหว่างจุดต่อจุด ซึ่งมีประโยชน์มากสำหรับการตรวจสอบและ debug รายจุด

สุดท้าย ควรตรวจสอบเอกสารทางการล่าสุดของ Google เรื่องขีดจำกัดจำนวน intermediate waypoints สำหรับการใช้ optimizeWaypointOrder (ในอดีตมีการระบุขีดจำกัดไว้ประมาณ 25 จุด) และเพิ่ม guard ในโค้ดเพื่อโยน error ที่สื่อความหมายชัดเจนหากจำนวนจุดเกินขีดจำกัดนั้น แทนที่จะให้ API ตอบกลับ error ที่เข้าใจยาก

สรุป Priority Action Items

ระดับ Critical (ควรแก้ก่อน): แก้ N+1 read ในฟังก์ชันเตรียมข้อมูลด้วยการอ่านแบบ batch, เพิ่ม guard check สำหรับ result.routes ก่อนเข้าถึง index 0, และเพิ่ม Logger.log ทุกจุดที่มีการ skip ข้อมูล (พิกัดผิด, ระยะทาง parse ไม่ได้) เพื่อไม่ให้เกิด silent data loss ในระบบที่เกี่ยวข้องกับการเบิกจ่ายเงิน

ระดับ Important (ควรแก้ในลำดับถัดไป): เพิ่ม retry logic แบบ exponential backoff สำหรับ API call ที่ล้มเหลวชั่วคราว, เพิ่มการตรวจสอบขอบเขต lat/lng, cache ค่า getLastRow(), และ refactor โค้ดซ้ำซ้อน (header index map, format lat/lng) ให้เป็น helper function

ระดับ Nice-to-have (ปรับปรุงระยะยาว): พิจารณาใช้ computeRouteMatrix แทนการพึ่งพาคอลัมน์ระยะทางที่คำนวณไว้ล่วงหน้าในชีตเพื่อความแม่นยำที่ดีขึ้น, เพิ่มฟีเจอร์ routeModifiers/extraComputations ตามความจำเป็นทางธุรกิจ, ทบทวน business rule เรื่องการบังคับจุดไกลสุดเป็นปลายทางสุดท้ายร่วมกับผู้ดูแลกระบวนการเบิกจ่ายว่าตรงกับเจตนาจริงหรือไม่ และพิจารณาวิธีจัดการลิงก์ Google Maps ที่อาจยาวเกินไปเมื่อมีจุดส่งจำนวนมาก