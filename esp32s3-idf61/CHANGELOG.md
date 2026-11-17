# CHANGELOG — skill esp32s3-idf61

> บันทึกการเปลี่ยนแปลงและเหตุผลการตัดสินใจ (ไม่ใส่ในไฟล์อื่น) เรียงใหม่สุดก่อน

## รอบ 5i — gpio.md v3.0-rc11, hotlist/GPIO.md v0.7 (GPIO-25, 32, 33 ปรับ), evidence v0.10 (EV-40), PROTOCOL v0.10, SKILL (RC11), tools/verify_gpio_unv8.sh (ใหม่)
**แหล่งหลักฐานใหม่:** `verify7.log.txt` (ผล verify_gpio_unv7.sh Z13-Z17 อ่านดิบ [VER]) + RAG (คำอธิบาย `gpio_hal_hold_*`) ไม่ได้ค้นเว็บ
**ปิดได้:** GPIO-32 บน S3 เกิดเฉพาะ deep sleep (isolate ที่ `sleep_modes.c:862` ใน `if (deep_sleep)`; จุด light sleep ใต้ `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` ซึ่ง S3 ไม่นิยาม) · GPIO-25 ยืนยันจาก docs · docs บรรทัด 616 เป็นของ C2/C3 · `gpio_ll_hold_en/dis/is_digital_io_hold` ของ S3 และ `abort()` เมื่อส่งขา 0-25
**ลดน้ำหนัก:** GPIO-33 กลไก "ขาถูก isolate จนลอย" ยังไม่ครบ เพราะลูป isolate ข้ามขาที่ `esp_gpio_is_reserved` เป็นจริง และคอมเมนต์ `gpio.c:381` บอกว่า output ต้อง reserve IO → คงรหัสไว้ แต่เขียนคำแนะนำ (ต้อง `gpio_hold_en` ทุกขา) เป็นข้อหลัก และกลไกเป็นเงื่อนไข [UNV]
**พบใหม่ (INF):** `gpio_hal_hold_dis` กับขา 0-21 ของ S3 ไม่แตะ `RTC_CNTL_DIG_PAD_HOLD_REG` เพราะ `GPIO_HOLD_MASK[0..21] = 0`
**ข้อผิดพลาดของผู้ช่วย:** (1) Z17/Z2 ใช้ `find -path '*esp32s3*'` ปนผลของ `esp32s31` → PROTOCOL B.5 ข้อ 18 (2) เขียน GPIO-33 ว่า "ลอย" จากโค้ด isolate ฝั่งเดียวโดยไม่ตรวจเงื่อนไข reserve → PROTOCOL B.5 ข้อ 19
**ยังเปิด:** [HW] T1-T5 · Z18-Z22 ของ `verify_gpio_unv8.sh` (ผู้จอง `esp_gpio_reserve`, `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` ทั้ง components, `_IO_WAKEUP_EXT0/1`, stack ตอน isolate) · ข้อ [GLM]: ห้าม printf ใน ISR (GPIO-03), ลำดับ `rtc_gpio_init` (GPIO-07), "deprecated v5.0" ของ `gpio_pad_select_gpio`

## รอบ 5h — gpio.md v3.0-rc10, hotlist/GPIO.md v0.6 (GPIO-33), evidence v0.9 (EV-39), PROTOCOL v0.9, SKILL (RC10), tools/verify_gpio_unv7.sh (ใหม่)
**แหล่งหลักฐานใหม่:** `verify6.log.txt` (ผล verify_gpio_unv6.sh Z7-Z12 อ่านดิบ [VER]) + RAG (ยืนยัน `gpio_hal_is_digital_io_hold`, `rtc_gpio_isolate`) ไม่ได้ค้นเว็บ
**ปิดได้:** ชื่อ LL ของ hold ขา RTC (`rtcio_ll_force_hold_enable`, `RTC_CNTL_PAD_HOLD_REG`) · `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` ไม่นิยามบน S3 → ใช้สาขา `#else` · ขา RTC ไม่ต้องคู่ `gpio_deep_sleep_hold_en` (ระดับกลไกโค้ด [VER]; บนบอร์ดรอ T5) · ตอนตื่นจาก deep sleep ระบบปลด hold เฉพาะขาปลุก
**พบใหม่:** GPIO-33 ขา digital ที่ไม่ได้ `gpio_hold_en` ถูก isolate (ลอย) เมื่อเปิด `gpio_deep_sleep_hold_en`
**ข้อผิดพลาดของผู้ช่วย:** (1) รอบก่อนเดาชื่อ `rtcio_ll_hold_enable` ผิดอีกครั้ง → ปรับ PROTOCOL ข้อ 17 และแก้นิสัยการเดาชื่อ (2) GPIO-32 เขียนว่า assert เกิดเฉพาะ `esp_deep_sleep_start` ทั้งที่ผู้เรียก isolate มี 2 จุด → แก้ให้ไม่ฟันธง
**ยังเปิด:** [HW] T1-T5 · Z13-Z17 ของ `verify_gpio_unv7.sh` (บริบท sleep_modes.c:862/904, `IO_WAKEUP_ENABLED`, `esp_gpio_is_reserved`, บริบท docs บรรทัด 272, `gpio_hal_hold_dis` บน S3) · ข้อ [GLM]: ห้าม printf ใน ISR (GPIO-03), ลำดับ `rtc_gpio_init` (GPIO-07), "deprecated v5.0" ของ `gpio_pad_select_gpio`

## รอบ 5g — gpio.md v3.0-rc9, hotlist/GPIO.md v0.5 (GPIO-31, GPIO-32), evidence v0.8 (EV-38), PROTOCOL v0.8, SKILL (RC9), tools/verify_gpio_unv6.sh (ใหม่)
**แหล่งหลักฐานใหม่:** `verify5.log.txt` (ผล verify_gpio_unv5.sh Z0-Z6 อ่านดิบ [VER]) + RAG (ยืนยัน `gpio_hal_deep_sleep_hold_en` และคำอธิบาย hold ใน header) ไม่ได้ค้นเว็บ
**ปิดได้:** ข้อความ "3 รอบ RTC slow clock" อยู่ใน `sleep_modes.rst:392` (ข้อความ GLM ถูก) · `FREERTOS_ISR_STACKSIZE` ช่วง/ค่า default · `gpio_ll_deep_sleep_hold_en/dis/is_en` ของ S3 เลื่อนเป็น [VER]
**พบใหม่:** assert ใน `esp_sleep_isolate_digital_gpio` เมื่อ deep-sleep hold เปิดแต่ stack อยู่ PSRAM (GPIO-32) · ลูป isolate เริ่มจาก mask ของ GPIO26-48 ไม่รวมขา RTC (สนับสนุนว่าขา RTC ไม่ต้องคู่ `gpio_deep_sleep_hold_en`)
**ปิดบางส่วน:** RTC GPIO hold ต้องคู่หรือไม่ → [VER][INF] รอ [HW] T5 และ Z8
**ข้อผิดพลาดของผู้ช่วย:** (1) EDIT 77 (รอบ 5f) สรุปว่าข้อความ 3 รอบ "ห้ามใช้" เพราะค้นแค่ header ซึ่งที่จริงอยู่ใน docs → เพิ่ม PROTOCOL B.5 ข้อ 16 (2) Z3 ว่างเพราะเดาชื่อฟังก์ชัน LL ผิด → ปรับ B.5 ข้อ 15 (3) รอบก่อนผมเสนอ EDIT 94 ที่พิมพ์ assert ผิดเล็กน้อย (must_be) แล้วแก้ในบรรทัดท้ายของ EDIT เดียวกัน
**ยังเปิด:** [HW] T1-T5 · Z7-Z12 ของ `verify_gpio_unv6.sh` (ชื่อ LL hold ของ rtc_io_ll.h, `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` บน S3, ผู้เรียก isolate) · ข้อ [GLM]: ห้าม printf ใน ISR (GPIO-03), ลำดับ `rtc_gpio_init` (GPIO-07), "deprecated v5.0" ของ `gpio_pad_select_gpio`

## รอบ 5f — gpio.md v3.0-rc8, hotlist/GPIO.md v0.4 (GPIO-30), evidence v0.7 (EV-37), PROTOCOL v0.7, SKILL (RC8), tools/verify_gpio_unv5.sh (ใหม่)
**แหล่งหลักฐานใหม่:** `verify4.log_with_code_fench_.txt` (ผล verify_gpio_unv4.sh Y0-Y8 อ่านดิบ [VER]) + RAG (`gpio_ll_deep_sleep_hold_en`) ไม่ได้ค้นเว็บ
**ปิดได้:** "ISR stack size" = `FREERTOS_ISR_STACKSIZE` · การจับคู่ชื่อ ROM เก่า→ใหม่ครบ 6 คู่ (เปิด GPIO-30) · header ของ `esp_rom_gpio_*` ครบ
**ปิดบางส่วน:** RTC GPIO 0-21 ไม่ต้องคู่ `gpio_deep_sleep_hold_en` (จากโครงสร้างโค้ด [INF] รอ Z1 และ [HW] T5)
**ไม่พบ:** ข้อความ "ค้าง 3 รอบ RTC slow clock" ใน esp_sleep.h และ gpio.h → ถอดป้ายข้อเท็จจริงออกจาก gpio.md §7.1 (ยังไม่ตรวจ docs รอ Z5)
**ข้อผิดพลาดของผู้ช่วย:** (1) Y7 ค้นผิด path จึงว่าง (2) ข้อความตัดสินผลที่เขียนไว้ท้ายหัวข้อสคริปต์ (เช่น "ไม่มีบรรทัด = ไม่พบ") ทำให้สับสนเมื่อผลมีบรรทัดแต่เป็นคนละเรื่อง → เพิ่ม PROTOCOL B.5 ข้อ 15
**ข้อสังเกต:** ผล Y5 มีตัวอักษร `` ` `` แทรกในซอร์ส น่าจะเกิดจากการคัดลอก/แปลงไฟล์ (บันทึกไว้ใน EV-37)
**ยังเปิด:** [HW] T1-T5 · Z1-Z6 ของ `verify_gpio_unv5.sh` · ข้อ [GLM]: ห้าม printf ใน ISR (GPIO-03), ลำดับ `rtc_gpio_init` (GPIO-07), "deprecated v5.0" ของ `gpio_pad_select_gpio`

## รอบ 5e — PROTOCOL v0.6, evidence v0.6 (EV-36), tools/verify_gpio_unv4.sh (ใหม่ รูปแบบเดิม)
**แหล่งหลักฐาน:** ไฟล์บันทึก `verify_gpio_unv.sh_verify.log_.md`, `..unv2..`, `..unv3..` ของผู้ใช้ (อ่านดิบ: เห็นทั้งโค้ดและผล) ไม่ได้ค้นเว็บ
**การตัดสินใจของผู้ใช้:** ยกเลิก `tools/run_verify.sh` (รอบ 5c-5d) ทั้งหมด ไม่ลงใน PROTOCOL กลับไปใช้รูปแบบแรก: ผู้ใช้รัน `bash verify_xxx.sh > verifyN.log 2>&1` แล้วบันทึกโค้ดคู่ผลเองในไฟล์ `.md` (แปลงเป็น `.docx` ก่อนอัปโหลด RAG) เหตุผล: skill จะเผยแพร่สาธารณะ agent อื่นต้องอ่านโค้ดเพื่อเข้าใจว่าพิสูจน์อะไร → PROTOCOL B.5 ข้อ 11 เขียนใหม่ (ไม่มีสคริปต์ห่อ, ห้าม `2>/dev/null` กับผลว่างที่เป็นหลักฐาน, สคริปต์ต้องมีคอมเมนต์วัตถุประสงค์)
**ยกเลิก/แก้จากรอบก่อน:** รอบ 5d เขียนว่า "สคริปต์ v4 มีฟังก์ชัน `run` พิมพ์ `$ คำสั่ง` และ `[exit=N]`" และ B.5 ข้อ 11 อ้างถึง → ยกเลิก (v4 ใหม่ใช้รูปแบบเดิม)
**ข้อผิดพลาดของผู้ช่วย (แก้):** รอบ 5b/EV-33 เขียนว่าข้อสรุปเก่าที่ว่า `ESP_PD_DOMAIN_RTC8M` deprecated "ผิด" แต่ที่ถูกคือข้อสรุปเก่า **ถูก** (`_Pragma` ที่ `esp_sleep.h:99`) สคริปต์ W5 ค้นแคบเกินไป → เพิ่ม PROTOCOL B.5 ข้อ 14 (ตรวจ deprecated สามระดับ) และ EV-36 · HANDOFF รอบ 5-5c ที่ว่า "RTC8M ไม่ต้องตรวจแล้ว" ให้ถือว่าถูกแทนที่ (RTC8M deprecated แทนด้วย `ESP_PD_DOMAIN_RC_FAST`)
**ข้อสังเกตเรื่องไฟล์บันทึก:** `unv3` มีบรรทัดแรก `# !/usr/bin/env bash` (มีช่องว่าง) และ "วิธีใช้" ซ้ำ ไม่กระทบการรัน; V1 ของ `unv` ถูกตัดที่ 20 บรรทัดด้วย `head -20` (เก็บไว้เป็นประวัติ)

## รอบ 5d — gpio.md v3.0-rc7, evidence v0.5 (EV-35), PROTOCOL v0.5, SKILL (RC7), tools/verify_gpio_unv4.sh (ใหม่)
**แหล่งหลักฐานใหม่:** `verify_gpio_unv3.sh_verify.log_.md` (X4 อ่านดิบ [VER]) ไม่ได้ค้นเว็บ
**ปิดได้:** `GPIO_HOLD_MASK[]` ของ S3 ครบ 49 รายการ (ดัชนี 0-21 = 0, ดัชนี i ≥ 22 = BIT(i-21)) · GPIO38 = BIT(17) · `run_verify.sh` ใช้งานได้จริง (ผู้ใช้รัน)
**การตัดสินใจ:** (1) ผู้ใช้อยากบันทึกผลเองได้ทั้งคำสั่งและผล → PROTOCOL B.5 ข้อ 11 กำหนดรูปแบบกลาง (2) สคริปต์ v4 มีฟังก์ชัน `run` พิมพ์ `$ คำสั่ง` และ `[exit=N]` อัตโนมัติ
**ข้อสังเกตที่ยังไม่แน่ใจ:** ตัวอ่านไฟล์ไม่แสดงบรรทัดคำสั่งในไฟล์ที่ผู้ใช้บันทึกเอง ไม่ทราบว่าไฟล์ไม่มี หรือตัวอ่านตัดทิ้ง
**ยังเปิด:** [HW] T1-T4 · พฤติกรรมฟังก์ชัน ROM · ข้อ [GLM] (รอ Y1-Y2 ของ verify_gpio_unv4.sh) · RTC GPIO ต้องคู่ `gpio_deep_sleep_hold_en` หรือไม่ (รอ Y5-Y8)

## รอบ 5c (ปิด session) — gpio.md v3.0-rc6, hotlist/GPIO.md v0.3, evidence v0.4 (EV-34), PROTOCOL v0.4, SKILL v0.4, labs/gpio_lab01.md (ไฟล์ใหม่), tools/run_verify.sh, tools/verify_gpio_unv3.sh
**แหล่งหลักฐานใหม่:** `verify3.log` (X1-X3 อ่านดิบด้วยคำสั่งคัดลอกตรงตัวอักษร [VER]) ค้น RAG หา `GPIO_HOLD_MASK` ไม่พบ (`gpio_periph.c` ไม่อยู่ใน RAG) ไม่ได้ค้นเว็บ
**ปิดได้:** `gpio_wakeup_disable` ไม่คืน interrupt type (GPIO-24 ยืนยันครบทั้ง C และ LL) · ความหมายของ hint ใน `hints.yml` · โครงสร้าง `GPIO_HOLD_MASK` ของ S3 (ดัชนี 0-21 = 0, ดัชนี 22 เริ่ม BIT(1), ดัชนี 26 = BIT(5))
**ข้อผิดพลาดของผู้ช่วยที่แก้:** รอบ 5 เขียนว่า "ห้ามแนะนำ `esp_rom_gpio_pad_select_gpio` เป็นตัวแทน" (อนุมานเอง) แต่ IDF hint ระบุว่าเป็นการเปลี่ยนชื่อ → แก้ GPIO-02 และ gpio.md §0 ข้อ 21 เป็น 2 ระดับ (ตรงตัว = ชื่อ ROM ใหม่, โค้ดแอปใหม่ = `gpio_reset_pin`/`gpio_config`) · ข้อความ "removed v6" ของ GLM ไม่ตรง (เป็นการ rename)
**ข้อผิดพลาดของกระบวนการ:** (1) `grep -A30` ตัดผล X3 ที่ดัชนี 29 → กฎ PROTOCOL B.5 ข้อ 11 ใช้ `sed -n 'a,bp'` หรือ `-A` ที่ใหญ่พอ (2) Edit Block รอบก่อนของไฟล์ evidence/hotlist ไม่ได้แก้บรรทัดเวอร์ชันหัวไฟล์ → ซิงก์ใน EDIT 44, 47
**การตัดสินใจ (ตามคำสั่งผู้ใช้):** ผลรัน verify บันทึกเป็น `verify_<api>_unv.sh_verify.log_.md` · รวมวิธีสร้าง/รัน/ซอร์สของแล็บไว้ใน `labs/<project>.md` ไฟล์เดียว (RAG รับไดเรกทอรีไม่ได้) · `caps_probe.c` เปลี่ยนไปใช้ `LAB_BTN_GPIO`/`LAB_HOLD_GPIO` แทน GPIO4/18 ตายตัว เพราะบอร์ดผู้ใช้มีจอ
**ยังเปิด:** ผล [HW] T1-T4 (ติดปัญหาอุปกรณ์ flash) · `GPIO_HOLD_MASK` ดัชนี 30+ · พฤติกรรมของฟังก์ชัน ROM `esp_rom_gpio_*` · ข้อ [GLM] ที่ยังไม่ตรวจ (ดู gpio.md §13)

## รอบ 5b — gpio.md v3.0-rc5, hotlist/GPIO.md (GPIO-02, GPIO-24), evidence v0.3 (EV-33)
**แหล่งหลักฐานใหม่:** `verify2.log` (อ่านดิบด้วยการสั่งคัดลอกตรงตัวอักษร) + RAG (`gpio_ll_wakeup_disable`, TRM เรื่อง hold) ไม่ได้ค้นเว็บ
**ปิดได้:** `gpio_pad_select_gpio` ชื่อเปล่าไม่มีประกาศใน v6.1 · `gpio_wakeup_disable` ไม่คืน interrupt type · `esp_sleep_get_ext1_wakeup_status` ไม่ deprecated · ETM ไม่นิยามบน S3 (แก้ §9 ที่ค้าง [INF])
**ยังเปิด:** ข้อความ hint ใน `hints.yml` · `GPIO_HOLD_MASK` ของ S3 · ผล [HW] → `verify3` (X1-X3)
**ข้อผิดพลาดของผู้ช่วยที่พบ:** ข้อความสรุปของตัวอ่านไฟล์รอบก่อนผิด 2 จุด (RTC8M deprecated, ext1 status "บางส่วน") เพราะผู้ช่วยส่งคำถามทั่วไปแทนคำสั่งคัดลอกตรงตัวอักษร → กฎใหม่: อ่านไฟล์ผลรันด้วยคำสั่ง verbatim เสมอ
**แก้คำแนะนำเดิม:** HANDOFF รอบ 5 ให้ตรวจ `ESP_PD_DOMAIN_RTC8M` ว่า deprecated ไม่ต้องตรวจแล้ว (ไม่ปรากฏใน raw)

## รอบ 5 — gpio.md v3.0-rc4, hotlist/GPIO.md (แก้ 2 entry), evidence v0.2 (EV-32), tools/verify_gpio_unv2.sh, ซิงก์เวอร์ชัน PROTOCOL/SKILL
**แหล่งหลักฐานใหม่:** `verify.log` ที่ผู้ใช้รัน `verify_gpio_unv.sh` บน IDF v6.1 (ป้าย [VER]) + RAG ไม่ได้ค้นเว็บ ผู้ช่วยอ่าน log ผ่านตัวดึงเอกสาร (สรุป ไม่ใช่ raw)
**ผลทดสอบ:** T1–T4 ยังไม่มี (ปัญหาอุปกรณ์ flash)

**ปิดได้:** `SOC_GPIO_SUPPORT_ETM` ไม่นิยามบน S3 · macro อื่นที่ค้นแล้วไม่นิยามตามเดิม · GPIO-28 ยืนยันบน sdkconfig จริง (`CONFIG_SOC_GPIO_VALID_GPIO_MASK=0x1FFFFFFFFFFFF`) · IDF = v6.1
**พบใหม่:** `esp_rom_gpio_pad_select_gpio` มีใน v6.1 (กับดัก: agent อาจเสนอเป็นตัวแทน `gpio_pad_select_gpio`) → เพิ่มคำเตือนใน GPIO-02 แทนรหัสใหม่ เพื่อคงจำนวน 29 entry
**ยังเปิด:** ชื่อเปล่า `gpio_pad_select_gpio` ถูกลบหรือไม่ · `gpio_wakeup_disable` คืน intr type หรือไม่ · `esp_sleep_get_ext1_wakeup_status` deprecated หรือไม่ · ค่า `GPIO_HOLD_MASK` ของ S3 → `verify_gpio_unv2.sh`

**ข้อผิดพลาดของผู้ช่วยที่แก้รอบนี้**
- สคริปต์ verify v1 ใช้ `head -20` ทำให้ผล V1 ถูกตัด → v2 ไม่ตัดผล และเพิ่มกฎใน PROTOCOL B.5 ข้อ 9
- Edit Block รอบ 4 ระบุเวอร์ชัน PROTOCOL/SKILL เป็น v0.2 แต่ไม่ได้แก้บรรทัดเวอร์ชันจริง → ซิงก์เป็น v0.3 (PROTOCOL) และ v0.3 (SKILL)
- กฎ "แก้เกิน 3 จุดให้ออกไฟล์เต็ม" เข้มเกิน ปรับเป็นเกิน 5 จุด หรือเกินประมาณ 30% ของไฟล์

**งานที่เลื่อน:** §9 ใน gpio.md (บรรทัด ETM ยังเป็น [INF]) แก้พร้อมออก v3.0 final

## รอบ 4 — gpio.md v3.0-rc3, hotlist/GPIO.md v0.2, evidence/gpio-evidence.md v0.1, tools/verify_gpio_unv.sh, gpio_lab01 v1.1
**แหล่งหลักฐาน:** RAG ทั้งหมด (esp_driver_gpio, esp_hal_gpio, soc_caps.h + Kconfig, esp_sleep.h, TRM, Datasheet, migration guide, gpio.rst) ไม่ได้ค้นเว็บ
**ผล build:** main.c ใหม่ (EDIT 1 รอบ 3) build ผ่านบน v6.1 (ผู้ใช้รายงาน) · ผล T1-T4 ยังไม่มี (ปัญหาอุปกรณ์ flash)

**ปิด [UNV] ได้:** flex filter/hys/HOLD_SINGLE_IO ไม่มีบน S3 · dedicated GPIO มี · RTC edge wake ไม่มี · EXT1 per-pin ไม่มี · enum wake cause (GPIO=7) · `esp_sleep_enable_ext1_wakeup` ไม่มี attribute deprecated · `gpio_new_pin_glitch_filter` ไม่บังคับ input · `rtc_gpio_isolate` · `esp_sleep_wakeup_io_bit2num` · GPIO3 (DS §3) · drive strength (TRM)
**ยืนยัน:** `gpio_wakeup_enable` ตั้ง intr type ของขา (เหตุผลของ EDIT 1 รอบ 3)
**ที่ยังเป็น [UNV]:** `gpio_pad_select_gpio` ถูกลบหรือไม่ (ไม่พบ symbol ในไฟล์ใดเลย) · body `gpio_wakeup_disable` · ผล [HW] · รายละเอียดใน gpio.md §13

**การตัดสินใจ**
- จัดเลข `GPIO-P01..P15` เป็น `GPIO-09..23` ครั้งเดียว (map ด้านล่าง) เพิ่ม GPIO-24..29 รวม 29 ข้อ (GLM มี 8 ข้อ)
- ส่วน GLM เดิม 8 ข้อ (เดิม 1, 10, 16, 62-65, 67) คงรหัส GPIO-01..08 และเก็บเลขเดิมไว้เป็น `legacy`
- สร้าง `evidence/` เพื่อให้ถอดไฟล์ต้นทางออกจาก RAG (ช่องจำกัด 10 ไฟล์) ได้โดยไม่เสียหลักฐาน
- สร้างสคริปต์ verify แทนการรอฮาร์ดแวร์สำหรับข้อที่ปิดได้ด้วย grep
- ย้ายขาของแล็บเป็น `LAB_BTN_GPIO` / `LAB_HOLD_GPIO` ใน `gpio_lab.h` เพราะบอร์ดของผู้ใช้ไม่เหมือนบอร์ดอ้างอิง

**แผนที่รหัส (ห้ามเปลี่ยนอีก)**

| เดิม | ใหม่ | เดิม | ใหม่ | เดิม | ใหม่ |
|---|---|---|---|---|---|
| GPIO-P01 | GPIO-09 | GPIO-P06 | GPIO-14 | GPIO-P11 | GPIO-19 |
| GPIO-P02 | GPIO-10 | GPIO-P07 | GPIO-15 | GPIO-P12 | GPIO-20 |
| GPIO-P03 | GPIO-11 | GPIO-P08 | GPIO-16 | GPIO-P13 | GPIO-21 |
| GPIO-P04 | GPIO-12 | GPIO-P09 | GPIO-17 | GPIO-P14 | GPIO-22 |
| GPIO-P05 | GPIO-13 | GPIO-P10 | GPIO-18 | GPIO-P15 | GPIO-23 |

**ข้อขัดแย้งกับเอกสาร GLM ที่พบรอบนี้**
- GPIO-02 ("removed v6"): หาหลักฐานไม่ได้ ติด [GLM][UNV] และเปลี่ยนเป็นคำแนะนำให้ใช้ `gpio_reset_pin`
- `esp_sleep_get_ext1_wakeup_status()` ถูกเขียนใน `projects_skills.docx` ว่า deprecated แต่ declaration ใน `esp_sleep.h` ไม่มี attribute
- รายการ SLEEP เดิม 68 เขียน `& ESP_SLEEP_WAKEUP_*` ไม่มี `BIT()` (ต้องแก้ตอนทำ sleep.md)
- GPIO-04: tag ของ GLM คือ COMPILE (คงไว้) เพราะชื่อ enum `GPIO_NUM_22` ไม่มี; ถ้าใช้เลขตรงๆ จะเป็น RUNTIME
## รอบ 3 — gpio.md v3.0-rc2, hotlist/GPIO.md v0.1, PROTOCOL v0.1, SKILL v0.1, README gpio_lab01
**แหล่งหลักฐานที่เพิ่ม:** `soc_caps.h` ของ S3, `esp_sleep.h`, `esp_driver_gpio` และ `esp_hal_gpio` ทั้ง directory, gpio.rst (RAG)
**ผล build:** gpio_lab (v1) build ผ่านบน v6.1 ไม่มี error (ผู้ใช้รายงาน)

**การเปลี่ยนแปลงใน gpio.md (rc1 → rc2)**
- ปิด [UNV]: flex filter ไม่มีบน S3; hysteresis ไม่มี; pin filter มี; force hold มี; `gpio_deep_sleep_hold_en/dis` มี; HP periph PD wake มี (0–21, level เท่านั้น); RTC edge wake ไม่มี
- เพิ่มตาราง Capability matrix (§3) และตารางตัวเลือกปลุกด้วยขา (§7.1)
- แก้ GPIO3: ใช้ TRM ตาราง 8.1-1 แทนการอ้าง Datasheet ในรอบก่อน
- ใส่ fenced code ครบทุกก้อน (rc1 ไม่มี)
- ย้ายตาราง Hot-List ไปไฟล์แยก `hotlist/GPIO.md`
- เพิ่ม §11 ที่ยังไม่ยืนยัน และตาราง [BUILD]/[HW]

**การแก้ที่ gpio_lab:** ออก EDIT 1 (main.c) แยกโหมด DEMO_* เพราะ light-sleep demo ใช้ GPIO4 ร่วมกับ btn_isr (เหตุผล: `gpio_wakeup_enable` น่าจะเปลี่ยน interrupt type ของขา [INF] ยังไม่เห็นซอร์ส)

**การตัดสินใจ**
- ใช้รหัสชั่วคราว `GPIO-Pnn` สำหรับรายการใหม่ เพราะยังไม่ได้ GPIO section เต็มของ HALLUCINATION-HOT-LIST.md เดิม (เลี่ยงเลขชนกัน)
- เพิ่ม tag [BUILD] และ [HW] แยกจาก [H61]: build ผ่านไม่ได้พิสูจน์พฤติกรรม
- ไม่ค้นเว็บในรอบนี้ เพราะ RAG มีหลักฐานครบสำหรับ gpio

## รอบ 2 — gpio.md v3.0-rc1 และโปรเจกต์ gpio_lab (v1)
- ร่าง gpio.md v3.0-rc1 ตามโครงสร้างใหม่ (TL;DR → CMake → HW → API → กับดัก → ตัวอย่าง → ยังไม่ยืนยัน)
- สร้างโปรเจกต์ gpio_lab: CMakeLists (โปรเจกต์ + main), caps_probe.c, btn_isr.c, sleep_demos.c
- ถอนคำยืนยันที่เคยอ้างว่าตรวจ soc_caps.h/esp_sleep.h (ไม่อยู่ใน RAG ตอนนั้น)
- **ข้อผิดใน gpio.md ของ GLM (v2.3) ที่พบ (11 จุด)**

| # | ของ GLM | ที่ถูกต้อง |
|---|---|---|
| 1 | glitch config ใช้ `gpio_clock_source_t`, `int gpio_num` | `glitch_filter_clock_source_t`, `gpio_num_t` |
| 2 | flex filter ใช้ `unsigned int` | `uint32_t` |
| 3 | บอก ISR ต้อง IRAM_ATTR (ขัดกับตัวอย่างของตัวเอง) | ต้องเฉพาะเมื่อใช้ `ESP_INTR_FLAG_IRAM` |
| 4 | `gpio_force_unhold_all` ไม่มีข้อจำกัด | ปลดขาที่ `gpio_hold_en` ไม่ได้ (OR) |
| 5 | `gpio_force_hold_all` ไม่มี warning | ต้องอยู่ IRAM; hold flash/UART; ROM boot ล้าง hold |
| 6 | ไม่พูดถึง `hys_ctrl_mode` | มีเฉพาะชิปที่ `SOC_GPIO_SUPPORT_PIN_HYS_FILTER` (S3 ไม่มี) |
| 7 | drive strength เน้นเฉพาะ GPIO17/18 | default ต่อขา: 17/18 = 10 mA, 19/20 = 40 mA, อื่น 20 mA |
| 8 | เสนอ `gpio_wakeup_enable_on_hp_periph_powerdown_sleep` ตรงๆ | header: "Called by the SDK" แต่ MIG มีตัวอย่างเรียกตรง |
| 9 | สำเนาใน projects_skills.docx: `uint64_t causes` และไม่มี `BIT()` | `uint32_t` และต้อง `BIT()` |
| 10 | GPIO3 ระบุ Pull-up (สำเนาใน projects_skills.docx) | ไม่มี internal pull |
| 11 | `gpio_pad_select_gpio` ถูกลบใน v6 | ไม่พบในเอกสาร migration [UNV] |

## รอบ 1 — วิเคราะห์โครงสร้างเดิมของ GLM และเสนอโครงสร้างใหม่
- เก็บ: Router บางๆ, template คงที่, Hot-List แยกและผูกรหัส, tag COMPILE/RUNTIME/SILENT, หัวข้อ "จุดตรวจซ้ำ"
- ปัญหาที่พบ: จำนวนข้อ/เวอร์ชันไม่ตรงกันข้ามไฟล์ (58/105/111; v2.2/v2.3; v3.2/v3.4), รหัสผูกทางเดียว, มีเลขเก่าค้าง ("เดิม 62"), ไม่แยกระดับหลักฐาน, ไม่มีหลักฐาน build
- เสนอ: ป้ายหลักฐานต่อประโยค, รหัสห้ามรีนัมเบอร์, ตัวอย่างต้อง build จริง, lint ตรวจสองทาง (แผน)