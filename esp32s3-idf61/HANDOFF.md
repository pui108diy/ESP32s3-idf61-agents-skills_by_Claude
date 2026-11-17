# HANDOFF.md — ส่งต่อ session (เขียนทับทุกครั้งที่ปิด session)

> ออกเมื่อสิ้นสุดรอบ 5c · ผู้รับ: session ถัดไป (รอบ 6)

## 1. บริบท
- เป้าหมาย: skill สำหรับ agent เขียนโค้ด ESP32-S3 + ESP-IDF v6.1 ให้ถูกต้องโดยไม่ต้องค้นเว็บบ่อย
- แนวทาง: อิง RAG เป็นหลัก ค้นเว็บเสริม ระบุที่มาทุกข้อ (RAG vs เว็บ vs ผลรันของผู้ใช้ [VER]/[HW])
- สภาพแวดล้อม: AI พิมพ์ไฟล์ในแชท มนุษย์คัดลอก/วาง อัปโหลดเข้า RAG ทุกสิ้น session แก้ไฟล์ด้วย Edit Block (PROTOCOL B.6, ≤5 จุด/ไฟล์ ไม่งั้นออกไฟล์เต็ม)
- แชท/RAG รับ `.txt` และ `.docx` เท่านั้น (ต้นฉบับ `.md` ให้ผู้ใช้คัดลอกเป็น `.txt` ตอนอัปโหลด) RAG รับไฟล์เดี่ยว ไม่รับไดเรกทอรี และจำกัด 10 ไฟล์ จึงต้องมี evidence file ก่อนถอดไฟล์ต้นทาง (PROTOCOL B.5 ข้อ 6)
- การอ่านไฟล์แนบ: ตัวอ่านดึงเฉพาะส่วนตามคำถาม ต้องสั่ง "คัดลอกตรงตัวอักษร ไม่สรุป" (PROTOCOL B.1) รอบ 5 เคยได้ข้อสรุปผิด 2 จุดเพราะถามแบบทั่วไป
- ผู้ใช้: พื้นฐานฮาร์ดแวร์, asm MCU 8 บิต, โค้ดภาษาอื่น (ต้องให้ภาพใหญ่ ภาพย่อย การประยุกต์ใช้)
- บอร์ดผู้ใช้: ESP32-S3 มีจอและปุ่มบนบอร์ด ต้องดู schematic ก่อนเลือก `LAB_BTN_GPIO`/`LAB_HOLD_GPIO` ปัญหา flash ของอุปกรณ์ยังแก้ไม่ได้ จึงยังไม่มีผล [HW] ผู้ใช้ให้ทำ API อื่นไปก่อน

## 2. ไฟล์ในชุด skill ณ ตอนนี้ (ให้ผู้รับนับซ้ำจากไฟล์ก่อนประกาศ)

| ไฟล์                                      | เวอร์ชัน   | สถานะ                                                                                              |
| ----------------------------------------- | ---------- | -------------------------------------------------------------------------------------------------- |
| SKILL.md                                  | v0.4-draft | Router (มีเฉพาะ gpio)                                                                              |
| PROTOCOL.md                               | v0.4-draft | A, B (มี [VER], B.1 เรื่องรูปแบบไฟล์/การอ่านไฟล์, B.2 tools+labs, B.5 ข้อ 1-13, กฎ edit ≤5 จุด)    |
| CHANGELOG.md                              | —          | รอบ 1-5c                                                                                           |
| HANDOFF.md                                | —          | ไฟล์นี้                                                                                            |
| references/peripherals/gpio.md            | v3.0-rc6   | RC รอ [HW] และปิด [UNV] §13                                                                        |
| hotlist/GPIO.md                           | v0.3       | 29 entry (GPIO-01..29) นับจาก heading `### GPIO-`                                                  |
| evidence/gpio-evidence.md                 | v0.4       | EV-01..3[n]4                                                                                       |
| tools/verify_gpio_unv.sh                  | v0.1       | รันแล้ว (ผลเดิมชื่อ verify.log)                                                                    |
| tools/verify_gpio_unv2.sh                 | v0.1       | รันแล้ว (verify2.log)                                                                              |
| tools/verify_gpio_unv3.sh                 | v0.1       | X1-X3 รันแล้ว (verify3.log) · **X4 ใหม่ยังไม่ได้รัน**                                              |
| tools/verify_gpio_unv*.sh_verify*.log_.md | —          | ผู้ใช้จะบันทึกต่อไปตามกฎ B.5 ข้อ 11                                                                |
| labs/gpio_lab01.md                        | v1.2       | ไฟล์เดียว (วิธีทำ+ซอร์ส) T0 ต้อง build ซ้ำหลัง EDIT 2-8, 56-57 · T1 PENDING-RUN · T2-T4 PENDING-HW |

## 3. RAG ที่แนะนำ (จำกัด 10 ไฟล์)
- เก็บ: TRM 2 ไฟล์, Product Overview, migration guide, `esp_sleep.h + soc` (ใช้กับ sleep.md และ soc_caps ของทุก API), `projects_skills.docx`
- ถอดได้หลังอัปโหลด skill pack รอบ 5c: `components_esp_driver_gpio`, `components_esp_hal_gpio`, `gpio.rst`, `gpio.md_reference_api_skill.docx`
- เพิ่ม: skill pack (SKILL, PROTOCOL, CHANGELOG, HANDOFF, gpio.md, hotlist, evidence, labs, ผล verify log รวมเป็นไฟล์เดียวหรือสองไฟล์) + ไฟล์ sleep (ดูงาน B)

## 4. ข้อเท็จจริงที่ปิดแล้ว (ไม่ต้องตรวจซ้ำ)
- ขา 0-21, 26-48 (45 ขา) · RTC GPIO 0-21 · ไม่มี input-only
- S3 ไม่มี (soc_caps.h, [VER]): flex glitch filter, hysteresis, `HOLD_SINGLE_IO_IN_DSLP`, RTC edge wake, EXT1 per-pin, `SOC_GPIO_SUPPORT_ETM` · มี: pin glitch filter, force hold, dedicated GPIO, `gpio_deep_sleep_hold_en/dis`
- IDF ของผู้ใช้ = v6.1 · sdkconfig: `CONFIG_SOC_GPIO_VALID_GPIO_MASK=0x1FFFFFFFFFFFF` ไม่ตรงกับ mask จริง ห้ามใช้
- `gpio_wakeup_enable` ตั้ง intr type เป็น level · `gpio_wakeup_disable` ไม่คืนค่า [VER] (ใช้ `gpio_set_intr_type` ซ้ำ)
- `gpio_pad_select_gpio` ชื่อเปล่าไม่มีประกาศใน v6.1 · IDF hint: ชื่อ ROM GPIO เก่าถูก "เปลี่ยนชื่อ" ไป `esp_rom_gpio.h` (ไม่ใช่ลบ) · ตรงตัว = `esp_rom_gpio_pad_select_gpio(uint32_t)` · โค้ดแอปใหม่ใช้ `gpio_reset_pin`/`gpio_config`
- `esp_sleep_get_ext1_wakeup_status()` ไม่ deprecated · `esp_sleep_get_wakeup_cause()` deprecated · `ESP_EXT1_WAKEUP_ALL_LOW` deprecated · `ESP_PD_DOMAIN_RTC8M` ไม่พบเป็น deprecated (ข้อความสรุปเก่าผิด)
- `esp_sleep_get_wakeup_causes()` คืน `uint32_t` ใช้ `BIT()`; GPIO=bit7, TIMER=bit4
- `GPIO_HOLD_MASK[]` ของ S3: ดัชนี 0-21 = 0, ดัชนี 22.. = BIT(1).. (ดัชนี 26 = BIT(5)) ตรงกับ TRM; ดัชนี 30+ ยังไม่เห็น
- GPIO3 ไม่มี internal pull

## 5. งานถัดไป (เรียงลำดับ)

### งาน A — ปิด gpio (เมื่อมีข้อมูลใหม่)
- รัน `bash tools/run_verify.sh tools/verify_gpio_unv3.sh` เพื่อได้ X4 (อาร์เรย์ `GPIO_HOLD_MASK` ฉบับเต็ม) แล้วแนบผลเป็น `.txt` → ปิด gpio.md §13 ข้อ 3 (ยืนยัน GPIO38 = BIT(17)) ด้วย Edit Block
- build ซ้ำหลังวาง EDIT 2-8, 56-57 → แจ้งผล T0 (v1.2)
- ผล T1 (ไม่ต้องต่อวงจร) → เทียบตาราง "ผลที่คาด" ใน labs/gpio_lab01.md §7; ผล T2-T4 เมื่อบอร์ดพร้อม → ติดป้าย [HW] ใน gpio.md §11 และ hotlist (GPIO-05, 08, 24, 25)
- ออก gpio.md v3.0 final เมื่อ [UNV] ที่กระทบโค้ดปิดหมด (ตอนนี้เหลือ §13 ข้อ 1-2 ที่ต้อง [HW], ข้อ 3, และข้อ 5-6 ที่ไม่กระทบโค้ดแอปหลัก) และตัดคำว่า RC
- งานเสริมเมื่อบอร์ดพร้อม: T5 ทดสอบ `gpio_hold_en` บน RTC GPIO (0-21) ว่าต้องคู่ `gpio_deep_sleep_hold_en` หรือไม่; ทดสอบ GPIO-24 และ GPIO-25
- ตัวเลือกเสริม: เพิ่ม GPIO-30 เรื่องชื่อ ROM เก่ากลุ่ม `gpio_pad_pullup`/`gpio_matrix_in/out` เมื่อยืนยันการจับคู่ชื่อและพารามิเตอร์แล้ว (ตอนนี้เป็น [INF] ใน gpio.md §13 ข้อ 6)

### งาน B — sleep.md (เริ่มได้เลย ไม่ต้องรอ [HW])
- ขอจากผู้ใช้: `esp_sleep.h` (มีแล้ว), `esp_hw_support` ส่วน sleep (`sleep_modes.c`, `Kconfig`), `esp_pm` (`esp_pm.h` + Kconfig) ถ้ามีที่ว่าง, เอกสาร `api-reference/system/sleep_modes`
- ใช้ gpio.md §7 เป็นฐาน ย้ายเนื้อหา wake จากขาไปอ้างจาก sleep.md โดยไม่ทำซ้ำ
- ต้องแก้ของ GLM ใน `projects_skills.docx`: รายการ SLEEP เดิม 68 เขียน `& ESP_SLEEP_WAKEUP_*` ไม่มี `BIT()`; ข้อความ "esp_sleep_get_ext1_wakeup_status deprecated" (ผิด [VER]); `esp_sleep_enable_timer_wakeup` (ตรวจ header)
- รหัสกลุ่ม SLEEP: ใช้ของ GLM (SLEEP-01..08) ต่อจากเลขล่าสุด ห้ามชนและห้ามรีนัมเบอร์ ขอส่วน SLEEP เต็มของ `HALLUCINATION-HOT-LIST.md` จาก `projects_skills.docx` ก่อนจัดเลข
- สร้าง `evidence/sleep-evidence.md`, `tools/verify_sleep_unv.sh` (ไม่ตัดผล ใช้ `sed -n`) และ `labs/sleep_lab01.md` (ถ้ามีโปรเจกต์) พร้อมกัน
- ข้อควรตรวจแรกสุด: `esp_sleep_pd_config`, `esp_sleep_enable_timer_wakeup` (พารามิเตอร์/คืนค่า), `esp_light_sleep_start`, `CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP`, ext0/ext1 และ wake ผ่าน `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown`

### งาน C — เครื่องมือ (ทำเมื่อมี gpio + sleep แล้ว)
- `tools/lint.py` ตรวจรหัสสองทาง reference ↔ hotlist, ตรวจ fence, นับ entry สร้าง STATUS
- `index/api-index.tsv` ของ gpio · `dedic_gpio.md`

## 6. ความเสี่ยง/ข้อควรระวัง
- ผล verify ที่ AI อ่านต้องมาจากคำสั่ง verbatim ถ้า AI ได้ข้อความสรุป ห้ามปิด [UNV] จากข้อสรุป "ไม่มี"
- `grep -A` ตัดนิยามกลางอาร์เรย์ได้ (เกิดกับ X3) ใช้ `sed -n 'a,bp'` (PROTOCOL B.5 ข้อ 12)
- gpio.rst ใน RAG ยังไม่ยืนยันว่าตรงแท็ก v6.1
- ข้อ [GLM] ใน gpio.md ที่ยังไม่ตรวจ: ห้าม printf ใน ISR (GPIO-03), ลำดับ `rtc_gpio_init` (GPIO-07), "ต้องค้าง 3 รอบ RTC slow clock", ISR stack ใน menuconfig, "deprecated v5.0" ของ `gpio_pad_select_gpio`
- ข้อมูล Datasheet เรื่อง GPIO47/48 ที่ 1.8 V บนรุ่น R8V/R16V ยังไม่ตรวจซ้ำ
- ยังไม่เคยใช้เว็บใน gpio.md
- ห้ามอ้างว่ามีผล [HW] จนกว่าผู้ใช้จะส่ง
- Edit Block ของไฟล์ที่ผู้ใช้เคยวางไม่ครบอาจค้นไม่เจอ ให้ถามว่าวาง EDIT ไหนแล้ว

## 7. ข้อความเปิด session ถัดไป (คัดลอกไปวาง)

```text
ต่อ session skill esp32s3-idf61 (รอบ 6). ผมอัปโหลด skill pack รอบ 5c เข้า RAG แล้ว และถอดไฟล์ต้นทาง GPIO ออกแล้ว.
ผลที่มี: (แนบ verify_gpio_unv3.sh_verify.log_.txt [มี X4] / ผล build v1.2 / ผล T1-T4 ตามฟอร์ม labs/gpio_lab01.md §9 หรือพิมพ์ว่า "ยังไม่มี")
งานที่ต้องการ: เริ่มงาน B (sleep.md). ไฟล์ที่ผมเพิ่มใน RAG: (ระบุชื่อ)
ช่วยอ่าน PROTOCOL.md และ HANDOFF.md ก่อนเริ่ม และเวลาอ่านไฟล์ผลรันให้คัดลอกตรงตัวอักษร.
```