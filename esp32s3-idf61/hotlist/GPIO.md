# HOT-LIST: กลุ่ม GPIO (ESP32-S3 · ESP-IDF v6.1)

> เวอร์ชันไฟล์: v0.7 · คู่กับ `references/peripherals/gpio.md` v3.0-rc11 · หลักฐานต้นฉบับใน `evidence/gpio-evidence.md`

> รูปแบบ entry และความหมาย tag อยู่ใน PROTOCOL.md A.5 และ B.4
> รหัส GPIO-01..08 มาจากเอกสาร GLM (ค่า `legacy` คือเลขรวมเดิมใน HALLUCINATION-HOT-LIST.md)
> รหัส GPIO-09..23 คือรหัสชั่วคราว `GPIO-P01..P15` ของรอบ 3 ที่จัดเลขจริงแล้ว (ตาราง map อยู่ใน CHANGELOG) และห้ามเปลี่ยนอีก

> รหัส GPIO-24..29 เพิ่มในรอบ 4 · รหัส GPIO-30..34 เพิ่มในรอบ 5
> จำนวน entry ทั้งหมด: 33 (นับจาก heading `### GPIO-` ในไฟล์นี้)
## ส่วนที่ 1: รหัสเดิมจากเอกสาร GLM

### GPIO-01 [COMPILE]
- **ผิด:** ประกาศ ISR เป็น `void isr(int pin)`
- **ถูก:** `void isr(void *arg)` ส่งเลขขาผ่าน `args`
- **หลักฐาน:** [H61] `gpio_isr_t` = `void (*)(void *arg)` (EV-07)
- **ref:** gpio.md §0 ข้อ 7, §4 · **legacy:** เดิม 1 · **verified-in:** v6.1

### GPIO-02 [COMPILE]
- **ผิด:** เรียก `gpio_pad_select_gpio()`
- **ถูก:** ใช้ `gpio_reset_pin()` (ปิด interrupt, เลือก function GPIO, ขา output ได้เปิด pull-up, deinit RTC IO ถ้าเป็น RTC) หรือ `gpio_config()`
- **หลักฐาน:** [VER] grep `-w gpio_pad_select_gpio` ทั้ง `components` และ `docs` ของ v6.1 พบเฉพาะ regex ใน `esp_rom/hints.yml` ไม่มีที่ประกาศหรือเรียกใช้ (EV-33) · [VER] ข้อความ hint ของ IDF: "The rom GPIO functions have been renamed. Please replace with the APIs in 'esp_rom_gpio.h'." (EV-34) · [VER] `esp_rom_gpio.h:24` มี `void esp_rom_gpio_pad_select_gpio(uint32_t iopad_num);` (EV-33) · [C61] พฤติกรรมของ `gpio_reset_pin` (EV-09)
- **หมายเหตุ:** (1) นี่คือการเปลี่ยนชื่อ ไม่ใช่การลบ ข้อความ "removed v6" ของ GLM จึงไม่ตรง และ "deprecated v5.0" ยังเป็น [GLM] (2) การแทนที่ตรงตัวคือ `esp_rom_gpio_pad_select_gpio()` กลุ่มชื่อเก่าที่ hint ครอบคลุม: `gpio_pad_set_drv`, `gpio_pad_pullup`, `gpio_pad_select_gpio`, `gpio_pad_unhold`, `gpio_matrix_in`, `gpio_matrix_out` (3) โค้ดแอปใหม่ให้ใช้ `gpio_reset_pin()` หรือ `gpio_config()` เพราะปิด interrupt, จัด pull, deinit RTC IO ให้ ส่วนฟังก์ชัน ROM ผมคาดว่าเลือกฟังก์ชัน pad อย่างเดียว [INF]
- **ref:** gpio.md §0 ข้อ 21, §5, §13 ข้อ 4-6 · **legacy:** เดิม 10 · **verified-in:** v6.1 (ชื่อเปล่าไม่มีประกาศ [VER])

### GPIO-03 [RUNTIME]
- **ผิด:** เรียก `printf`/`ESP_LOGx` ใน ISR
- **ถูก:** ส่งข้อมูลด้วย `xQueueSendFromISR` แล้วพิมพ์ใน task
- **หลักฐาน:** [GLM][INF] (ยังไม่มีข้อความยืนยันใน header)
- **ref:** gpio.md §6 · **legacy:** เดิม 16 · **verified-in:** ยังไม่ยืนยัน

### GPIO-04 [COMPILE]
- **ผิด:** ใช้ GPIO22-25
- **ถูก:** ไม่มีบน S3 (enum `gpio_num_t` ข้ามจาก 21 ไป 26) ใช้ `GPIO_IS_VALID_GPIO(n)` ตรวจ ถ้าใช้เป็นเลขตรงๆ ผลคือ `ESP_ERR_INVALID_ARG` ตอนรัน [C61]
- **หลักฐาน:** [SOC] `SOC_GPIO_VALID_GPIO_MASK` ตัด BIT22-25 (EV-01) · [H61] enum (EV-06)
- **ref:** gpio.md §0 ข้อ 1, §2 · **legacy:** เดิม 62 · **verified-in:** v6.1

### GPIO-05 [RUNTIME]
- **ผิด:** ใช้ `gpio_hold_en()` อย่างเดียวเพื่อคง digital GPIO (26-48) ใน deep sleep
- **ถูก:** ต้องเรียก `gpio_deep_sleep_hold_en()` ด้วย; ขา RTC GPIO 0-21 `gpio_hold_en` ส่งต่อไป `rtc_gpio_hold_en`
- **หลักฐาน:** [H61] คำอธิบาย `gpio_hal_hold_en` ระบุ S3 (EV-11), `gpio_deep_sleep_hold_en` (EV-12) · [C61] การส่งต่อไป RTC
- **ref:** gpio.md §0 ข้อ 11, §7.4 · **legacy:** เดิม 63 · **verified-in:** v6.1 · **[HW]:** รอ T4

### GPIO-06 [SILENT]
- **ผิด:** ปล่อย GPIO3 ลอย
- **ถูก:** GPIO3 ไม่มี internal pull ต้องมีวงจรภายนอกกำหนดระดับ ค่า GPIO3 มีผลเมื่อ EFUSE_STRAP_JTAG_SEL=1 และ DIS_PAD_JTAG=0, DIS_USB_JTAG=0; default JTAG วิ่งผ่าน USB Serial/JTAG และไม่สนค่า GPIO3
- **หลักฐาน:** [DS §3 + Table 3-5] [TRM 8.1-1, 8.5-1] (EV-24)
- **ref:** gpio.md §0 ข้อ 20, §2 · **legacy:** เดิม 64 · **verified-in:** DS v2.2, TRM v1.8

### GPIO-07 [RUNTIME]
- **ผิด:** เปิด `CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP` แล้วใช้ wakeup กับ RTC GPIO โดยไม่เรียก `rtc_gpio_init()` และตั้ง direction เป็น input
- **ถูก:** เรียก `rtc_gpio_init()` + ตั้ง direction เป็น input ก่อน (เพราะ `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown` ไม่แก้ config ขา)
- **หลักฐาน:** [GLM] ส่วน "ไม่แก้ config ขา" เป็น [H61] (EV-16) ส่วนความจำเป็นของ `rtc_gpio_init` ยังไม่ยืนยัน [INF]
- **ref:** gpio.md §7.5 · **legacy:** เดิม 65 · **verified-in:** ยังไม่ยืนยัน

### GPIO-08 [SILENT]
- **ผิด:** เรียก `gpio_hold_dis()` หลังตื่นโดยไม่ตั้ง direction/level ก่อน
- **ถูก:** ตั้ง direction + level ให้เท่าเดิมก่อน ไม่เช่นนั้นขากลับเป็น input และระดับเปลี่ยน
- **หลักฐาน:** [H61] ตัวอย่าง gpio18 ใน header (EV-11)
- **ref:** gpio.md §0 ข้อ 12, §7.4 · **legacy:** เดิม 67 · **verified-in:** v6.1 · **[HW]:** รอ T4

### รหัสข้ามกลุ่มที่ gpio.md เดิมอ้างถึง (เจ้าของคือกลุ่มอื่น ไม่นับในไฟล์นี้)
- **SLEEP-03:** `gpio_wakeup_enable()` ปลุกได้เฉพาะ light sleep
- **SPI-03 / SPI-04:** GPIO33-37 ใช้ได้ตามโมดูล / ป้าย FSPI* เป็น alternate function

## ส่วนที่ 2: รหัสที่จัดเลขแล้ว (เดิม GPIO-P01..P15)

### GPIO-09 [SILENT] (เดิม P01)
- **ผิด:** ใส่ `IRAM_ATTR` ให้ ISR โดยเชื่อว่าจำเป็นเสมอ
- **ถูก:** จำเป็นเมื่อ install ด้วย `ESP_INTR_FLAG_IRAM` เท่านั้น
- **หลักฐาน:** [H61] คำอธิบาย `gpio_isr_handler_add` (EV-07)
- **ref:** gpio.md §0 ข้อ 8, §6 · **verified-in:** v6.1

### GPIO-10 [COMPILE] (เดิม P02)
- **ผิด:** glitch filter config ใช้ `gpio_clock_source_t` และ `int gpio_num`
- **ถูก:** `glitch_filter_clock_source_t clk_src` และ `gpio_num_t gpio_num`
- **หลักฐาน:** [H61] `gpio_filter.h` (EV-21) · [BUILD]
- **ref:** gpio.md §8 · **verified-in:** v6.1

### GPIO-11 [RUNTIME] (เดิม P03)
- **ผิด:** ใช้ `gpio_force_unhold_all()` ปลดขาที่ `gpio_hold_en` ไว้
- **ถูก:** ปลดรายขาด้วย `gpio_hold_dis()` (สัญญาณ hold รวมกันแบบ OR)
- **หลักฐาน:** [H61] (EV-13)
- **ref:** gpio.md §0 ข้อ 13 · **verified-in:** v6.1

### GPIO-12 [RUNTIME] (เดิม P04)
- **ผิด:** เรียก `gpio_force_hold_all()` ในโค้ดที่อยู่ flash
- **ถูก:** ฟังก์ชันนี้และโค้ดหลังจากนั้น (จนกว่าจะ unhold) ต้องอยู่ internal RAM เพราะ hold ขา flash จะหยุด SPI flash และ hold ขา UART จะหยุด log; hold ถูกล้างตอน ROM boot จึงไม่เหมาะคงขาข้าม deep sleep/reset
- **หลักฐาน:** [H61] (EV-13)
- **ref:** gpio.md §0 ข้อ 13 · **verified-in:** v6.1

### GPIO-13 [RUNTIME] (เดิม P05)
- **ผิด:** `ESP_ERROR_CHECK(gpio_install_isr_service(0))`
- **ถูก:** ยอมรับ `ESP_ERR_INVALID_STATE` (component อื่นอาจติดตั้งแล้ว)
- **หลักฐาน:** [C61] (EV-08) · [BUILD]
- **ref:** gpio.md §6 · **verified-in:** v6.1

### GPIO-14 [COMPILE] (เดิม P06)
- **ผิด:** ใช้ `gpio_deep_sleep_wakeup_enable/disable`, `GPIO_IS_DEEP_SLEEP_WAKEUP_VALID_GPIO`, `MAX_PAD_GPIO_NUM`, `MAX_GPIO_NUM`, `DIG_IO_HOLD_BIT_SHIFT`
- **ถูก:** ถูกลบใน v6.0 ใช้ `gpio_wakeup_*_on_hp_periph_powerdown_sleep` และ `GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO()`
- **หลักฐาน:** [MIG] (EV-23)
- **ref:** gpio.md §0 ข้อ 19 · **verified-in:** v6.0+

### GPIO-15 [COMPILE] (เดิม P07)
- **ผิด:** เรียก `gpio_iomux_in/out`
- **ถูก:** เปลี่ยนเป็น `gpio_iomux_input/output` และย้ายเป็น private (`esp_private/gpio.h`); ฟังก์ชัน ROM ใช้ prefix `rom_`
- **หลักฐาน:** [MIG] (EV-23)
- **ref:** gpio.md §0 ข้อ 19 · **verified-in:** v6.0+

### GPIO-16 [COMPILE] (เดิม P08)
- **ผิด:** ใช้ `hys_ctrl_mode`, `gpio_new_flex_glitch_filter` หรือ `gpio_force_hold_all` โดยไม่ตรวจ capability
- **ถูก:** บน S3 `hys_ctrl_mode` และ flex filter **ไม่มี** (compile error) ส่วน force hold มี ครอบด้วย `#if SOC_*` เสมอ
- **หลักฐาน:** [SOC][H61] (EV-01)
- **ref:** gpio.md §3 · **verified-in:** v6.1

### GPIO-17 [SILENT] (เดิม P09)
- **ผิด:** `causes & ESP_SLEEP_WAKEUP_X`
- **ถูก:** `causes & BIT(ESP_SLEEP_WAKEUP_X)` และ `causes` เป็น `uint32_t`
- **หลักฐาน:** [H61][T61] (EV-18, EV-22)
- **ref:** gpio.md §0 ข้อ 18, §7.3 · **verified-in:** v6.1
- **หมายเหตุ:** รายการ SLEEP เดิม 68 ใน HALLUCINATION-HOT-LIST.md เขียน `& ESP_SLEEP_WAKEUP_*` ไม่มี `BIT()` ให้แก้ตอนทำ sleep.md

### GPIO-18 [RUNTIME] (เดิม P10)
- **ผิด:** ใช้ GPIO26-32 (หรือ 33-37 เมื่อใช้ Octal) เป็น GPIO ทั่วไปั
- **ถูก:** ห้ามใช้เมื่อต่อ flash/PSRAM ในแพ็กเกจ
- **หลักฐาน:** [DS] "Do not use the pins connected to in-package flash/PSRAM for any other purposes"
- **ref:** gpio.md §2, §12 · **verified-in:** DS v2.2

### GPIO-19 [SILENT] (เดิม P11)
- **ผิด:** `1UL << pin` ใน `pin_bit_mask`
- **ถูก:** `1ULL << pin` (mask เป็น `uint64_t` และมีขาเกิน 31)
- **หลักฐาน:** [H61] · `gpio_config` ตรวจ mask 64 บิต (EV-09)
- **ref:** gpio.md §0 ข้อ 3 · **verified-in:** v6.1

### GPIO-20 [RUNTIME] (เดิม P12)
- **ผิด:** `gpio_get_level()` บน pad ที่ไม่ได้เปิด input แล้วคาดว่าจะอ่านค่า output
- **ถูก:** ได้ 0 เสมอ ใช้ `GPIO_MODE_INPUT_OUTPUT` ถ้าต้องอ่านค่าที่ขับอยู่
- **หลักฐาน:** [H61] "If the pad is not configured for input (or input and output) the returned value is always 0" (EV-10)
- **ref:** gpio.md §0 ข้อ 5 · **verified-in:** v6.1

### GPIO-21 [RUNTIME] (เดิม P13)
- **ผิด:** เรียก `gpio_wakeup_enable_on_hp_periph_powerdown_sleep` ด้วยขานอก 0-21 หรือ edge type บน S3
- **ถูก:** ได้ `ESP_ERR_INVALID_ARG` ใช้เฉพาะ GPIO 0-21 และ LOW/HIGH level; ใน APP ควรใช้ `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown()` หรือ EXT1
- **หลักฐาน:** [C61][H61][SOC] (EV-15)
- **ref:** gpio.md §7.1 · **verified-in:** v6.1

### GPIO-22 [RUNTIME] (เดิม P14)
- **ผิด:** เรียก `esp_sleep_enable_gpio_wakeup()` ขณะเปิด `CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP`
- **ถูก:** API ใช้ไม่ได้ (GPIO module ถูกปิดไฟ) ให้ใช้ `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown()` หรือ EXT1 (`esp_sleep_enable_ext1_wakeup_io`)
- **หลักฐาน:** [H61] (EV-16)
- **ref:** gpio.md §0 ข้อ 15, §7.1 · **verified-in:** v6.1

### GPIO-23 [RUNTIME] (เดิม P15)
- **ผิด:** ใช้ `esp_sleep_get_wakeup_cause()` (deprecated attribute) หรือ `esp_sleep_enable_ext1_wakeup()` ในโค้ดใหม่ v6.x
- **ถูก:** ใช้ `esp_sleep_get_wakeup_causes()` (bitmap) และ `esp_sleep_enable_ext1_wakeup_io()` + `esp_sleep_disable_ext1_wakeup_io()` (ตัวเดิมรีเซ็ต ext1 ทุกครั้ง)
- **หลักฐาน:** [H61] (EV-17, EV-18) · `esp_sleep_get_ext1_wakeup_status()` ไม่พบ attribute deprecated จึงยังใช้ได้
- **ref:** gpio.md §7.1, §7.3 · **verified-in:** v6.1

## ส่วนที่ 3: รหัสใหม่ รอบ 4

### GPIO-24 [RUNTIME]
- **ผิด:** ใช้ขาเดียวกันเป็นทั้ง ISR แบบ edge (`gpio_isr_handler_add`) และ light-sleep wake (`gpio_wakeup_enable`)
- **ถูก:** `gpio_wakeup_enable` เรียก `gpio_hal_set_intr_type` ตั้ง interrupt type ของขาเป็น LOW/HIGH level จึงชนกับ edge ISR (ISR อาจถูกเรียกรัวขณะระดับค้าง) ส่วน `gpio_wakeup_disable` **ไม่** เรียก `gpio_hal_set_intr_type` จึงไม่คืน interrupt type เดิม ถ้าจะกลับไปใช้ edge ISR ต้องเรียก `gpio_set_intr_type(pin, GPIO_INTR_NEGEDGE)` ซ้ำเสมอ หรือแยกขา (ผลกระทบต่อ ISR เป็น [INF])
- **หลักฐาน:** [C61] `gpio_wakeup_enable` (EV-14) · [VER] ตัวฟังก์ชัน `gpio_wakeup_disable` ทั้งหมด และ `gpio_hal_set_intr_type` ถูกเรียกที่บรรทัด 169, 701 เท่านั้น (EV-33) · [VER] `gpio_ll_wakeup_disable` ของ S3 ทำแค่ `hw->pin[gpio_num].wakeup_enable = 0;` (EV-34)
- **ref:** gpio.md §0 ข้อ 10, §6 · **verified-in:** v6.1

### GPIO-25 [RUNTIME]
- **ผิด:** เรียก `esp_sleep_enable_ext1_wakeup_io()` หลายครั้งด้วย level ต่างกันบน S3
- **ถูก:** ได้ `ESP_ERR_NOT_ALLOWED` เพราะ S3 ไม่มี per-pin mode ให้ใช้ level เดียวกันทุกขาใน EXT1 หรือแยกบางขาไปใช้ EXT0 / `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown`
- **หลักฐาน:** [H61] เงื่อนไข `!SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN` (EV-17) · [SOC][VER] ไม่มี macro นี้บน S3 (EV-03, EV-32) · [VER docs] `sleep_modes.rst` ใต้ `.. only:: not SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN`: "when we use more than one IO for EXT1 wakeup, it is not allowed to configure different wakeup levels for the IOs, and there is corresponding inspection mechanism in esp_sleep_enable_ext1_wakeup_io" (EV-40 Z16)
- **ref:** gpio.md §0 ข้อ 17, §7.1 · **verified-in:** v6.1 · **[HW]:** ยังไม่ทดสอบ

### GPIO-26 [COMPILE]
- **ผิด:** ใช้ `ESP_EXT1_WAKEUP_ALL_LOW` บน S3
- **ถูก:** ใช้ `ESP_EXT1_WAKEUP_ANY_LOW` (ค่า ALL_LOW มี attribute deprecated และเป็น alias ของ ANY_LOW; โหมด "ทุกขาต้อง low" ใช้ได้เฉพาะ ESP32)
- **หลักฐาน:** [H61] (EV-17)
- **ref:** gpio.md §0 ข้อ 17, §4 · **verified-in:** v6.1

### GPIO-27 [RUNTIME]
- **ผิด:** `gpio_wakeup_enable(pin, GPIO_INTR_NEGEDGE/POSEDGE/ANYEDGE)` หรือ `rtc_gpio_wakeup_enable` ด้วย edge บน S3
- **ถูก:** ได้ `ESP_ERR_INVALID_ARG` ปลุกด้วยได้เฉพาะ `GPIO_INTR_LOW_LEVEL` / `GPIO_INTR_HIGH_LEVEL` (S3 ไม่มี `SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED`)
- **หลักฐาน:** [H61][C61] (EV-14, EV-20) · [SOC] (EV-02)
- **ref:** gpio.md §0 ข้อ 10, §7.1 · **verified-in:** v6.1

### GPIO-28 [SILENT]
- **ผิด:** อ่านค่า mask/capability จาก `CONFIG_SOC_*` (Kconfig/sdkconfig) แทน `SOC_*` ใน `soc_caps.h`
- **ถูก:** ใช้ `SOC_*` จาก `soc/soc_caps.h` Kconfig mirror เก็บ expression ไม่ได้ (เช่น `CONFIG_SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK` = 0, VALID_OUTPUT mask = `0x1FFFFFFFFFFFF` ไม่ตัด 22-25)
-  **หลักฐาน:** [SOC] `Kconfig.soc_caps.in` เทียบ `soc_caps.h` (EV-04) · [VER] sdkconfig ของ gpio_lab01 บน v6.1 ได้ `CONFIG_SOC_GPIO_VALID_GPIO_MASK=0x1FFFFFFFFFFFF` ซึ่งไม่ตัด BIT22-25 (EV-32)
- **ref:** gpio.md §3 · **verified-in:** v6.1

### GPIO-29 [SILENT]
- **ผิด:** เรียก `gpio_config`/`gpio_reset_pin`/`gpio_set_direction`/`gpio_set_pull_mode` บนขาที่ต่อสัญญาณ peripheral แล้ว (หลัง init ของ peripheral)
- **ถูก:** API เหล่านี้เขียนทับ config เดิม (`gpio_config` บังคับ function เป็น GPIO) เรียกก่อน init peripheral หรือใช้ `gpio_input_enable`, `gpio_pullup_en`, `gpio_pulldown_en`, `gpio_od_enable` ที่ "เพิ่ม" config
- **หลักฐาน:** [C61] (EV-09) · [RST] ตัวอย่าง LEDC (EV-28) · README ของ esp_driver_gpio (EV-31)
- **ref:** gpio.md §0 ข้อ 4, §9 · **verified-in:** v6.1

### GPIO-30 [COMPILE]
- **ผิด:** เรียกชื่อ ROM GPIO เก่าตรงๆ: `gpio_pad_pullup`, `gpio_pad_set_drv`, `gpio_pad_unhold`, `gpio_matrix_in`, `gpio_matrix_out` (ส่วน `gpio_pad_select_gpio` ดู GPIO-02)
- **ถูก:** `#include "esp_rom_gpio.h"` แล้วใช้ชื่อใหม่ตามตาราง ชื่อใหม่ไม่ตรงชื่อเก่าทุกตัว (pullup ได้ `_only`, matrix ได้ `connect_*_signal`) จึงห้ามเดาจากชื่อเก่า ให้ใช้ signature จาก `esp_rom_gpio.h`

| ชื่อเก่า | ชื่อใหม่ | หน้าที่ตาม header |
|---|---|---|
| `gpio_pad_select_gpio` | `esp_rom_gpio_pad_select_gpio(uint32_t iopad_num)` | ตั้ง IO Pad เป็น GPIO เพื่อต่อกับ matrix |
| `gpio_pad_pullup` | `esp_rom_gpio_pad_pullup_only(uint32_t iopad_num)` | เปิด pull-up และปิด pull-down |
| `gpio_pad_set_drv` | `esp_rom_gpio_pad_set_drv(uint32_t iopad_num, uint32_t drv)` | ตั้ง drive capability |
| `gpio_pad_unhold` | `esp_rom_gpio_pad_unhold(uint32_t gpio_num)` | ปลด hold ของ pad |
| `gpio_matrix_in` | `esp_rom_gpio_connect_in_signal(uint32_t gpio_num, uint32_t signal_idx, bool inv)` | ต่อสัญญาณ input ของ peripheral เข้า GPIO |
| `gpio_matrix_out` | `esp_rom_gpio_connect_out_signal(uint32_t gpio_num, uint32_t signal_idx, bool out_inv, bool oen_inv)` | ต่อสัญญาณ output ของ peripheral ออก GPIO |

- **หมายเหตุ:** (1) ผลถ้าใช้ชื่อเก่า: `implicit declaration` หรือ `undefined reference` และ IDF แสดง hint "The rom GPIO functions have been renamed. Please replace with the APIs in 'esp_rom_gpio.h'." (2) การจับคู่ชื่อ ยืนยันจาก linker script ว่าชื่อใหม่ชี้ไปที่สัญลักษณ์ ROM `rom_gpio_*` ส่วนพารามิเตอร์ของชื่อเก่าไม่ได้ตรวจ ให้ใช้พารามิเตอร์ของชื่อใหม่ (3) โค้ดแอปควรใช้ driver ก่อน เช่น `gpio_set_pull_mode`, `gpio_set_drive_capability`, `gpio_hold_dis` แต่ความเทียบเท่าแบบ 1:1 ไม่ได้ตรวจ [INF] (4) header ของ `esp_rom_gpio_pad_set_drv` ระบุ 0/1/2/3 = 5/10/20/40 mA โดยไม่กล่าวถึงข้อยกเว้นของ GPIO17/18 ที่ TRM ระบุว่าค่า 1 กับ 2 สลับกัน ให้ใช้ `gpio_set_drive_capability()` ด้วยชื่อ enum แทน (5) `esp_rom_gpio_connect_out_signal` ถ้าส่ง `SIG_GPIO_OUT_IDX` จะเป็นการตัดการเชื่อมต่อ peripheral ให้ driver GPIO คุมระดับเอง [VER]
- **หลักฐาน:** [VER] `esp32s3.rom.api.ld` บรรทัด 11-16 (`PROVIDE ( esp_rom_gpio_xxx = rom_gpio_xxx )`), `esp32s3.rom.ld` บรรทัด 458-470, `esp_rom_gpio.h` ฉบับเต็ม, `hints.yml` (EV-34, EV-37) · [MIG] ฟังก์ชัน ROM GPIO เพิ่ม prefix `rom_` (EV-23)
- **ref:** gpio.md §0 ข้อ 21, §13 ข้อ 4 · **verified-in:** v6.1

### GPIO-31 [SILENT]
- **ผิด:** ส่งสัญญาณปลุกด้วยขา GPIO (level หรือ pulse) สั้นมาก เช่น พัลส์ระดับ µs แล้วคาดว่าบอร์ดจะตื่น
- **ถูก:** สัญญาณต้องค้างอย่างน้อย 3 รอบ RTC slow clock ความยาวต่อรอบขึ้นกับ `CONFIG_RTC_CLK_SRC` (RC_SLOW ~136 kHz ≈ 7.4 µs/รอบ → 3 รอบ ≈ 22 µs, XTAL32K ≈ 30.5 µs/รอบ → ≈ 92 µs; ผลคูณผมคำนวณเอง) ถ้าสั้นกว่านี้อาจไม่ถูกสุ่มตัวอย่าง ไม่มี error ใด ๆ แค่ไม่ตื่น ใช้กับทั้ง `esp_sleep_enable_gpio_wakeup()` และ `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown()`
- **หลักฐาน:** [VER] `docs/en/api-reference/system/sleep_modes.rst:392` (EV-38) ข้อความไม่ปรากฏใน `esp_sleep.h` และ `gpio.h`
- **ref:** gpio.md §0 ข้อ 22, §7.1 · **verified-in:** v6.1 · **[HW]:** ยังไม่ทดสอบ

### GPIO-32 [RUNTIME]
- **ผิด:** เรียก `gpio_deep_sleep_hold_en()` แล้วเรียก `esp_deep_sleep_start()` จาก task ที่ stack อยู่ใน external RAM (PSRAM)
- **ถูก:** ต้องให้ task ที่เรียก `esp_deep_sleep_start()` มี stack ใน internal RAM เมื่อ deep-sleep hold เปิดอยู่ มิฉะนั้น `assert(esp_ptr_internal(esp_cpu_get_sp()) && "If hold digital IO, the stack of the task calling esp_deep_sleep_start must be in internal ram!")` ล้ม (ฟังก์ชันตรวจนี้ `return` ก่อนถึง assert ถ้า deep-sleep hold ไม่ได้เปิด) ผลของ assert ขึ้นกับระดับ assertion ที่ตั้งใน menuconfig [INF]

- **หลักฐาน:** [VER] `components/esp_hw_support/sleep_gpio.c` บรรทัด 284-299 (EV-38 Z1) · [VER] ผู้เรียก `esp_sleep_isolate_digital_gpio` คือ `sleep_modes.c:862` (ด้วย `false` ในสาขา `if (deep_sleep)` ของ `esp_sleep_start_safe`) และ `:904` (ด้วย `true` ในสาขา light sleep ใต้ `#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD`) (EV-39 Z10, EV-40 Z13) บน S3 macro นั้นไม่นิยามตามที่ค้นใน `soc/esp32s3` และ `esp_hal_gpio/esp32s3` (EV-39 Z7) จึงเกิดเฉพาะเส้นทาง deep sleep ยังไม่ได้ค้นทั้ง `components/` (รอ Z19) และยังไม่ทราบว่าตอนเรียก isolate stack ถูกสลับหรือไม่ (รอ Z22) [UNV]

- **ref:** gpio.md §0 ข้อ 23, §7.4 · **verified-in:** v6.1 (ตัวโค้ด) · **[HW]:** ยังไม่ทดสอบ

### GPIO-33 [SILENT]
- **ผิด:** เรียก `gpio_deep_sleep_hold_en()` แล้วคาดว่าขา digital ที่ขับอยู่ (เช่น LED, รีเลย์) จะคงระดับใน deep sleep โดยไม่ได้เรียก `gpio_hold_en(pin)` ให้ขานั้น

- **ถูก:** ต้อง `gpio_hold_en(pin)` กับทุกขา digital ที่ต้องคงระดับเสมอ (คำแนะนำนี้ถูกในทุกกรณี) กลไกที่เห็นจากโค้ด: เมื่อเปิด `gpio_deep_sleep_hold_en()` ขา digital (GPIO26-48) ที่ไม่ได้ถูก hold และ `esp_gpio_is_reserved` เป็นเท็จ จะถูกสั่ง input disable, output disable, pull-up disable, pull-down disable และ function = GPIO ก่อนเข้า deep sleep [VER] ผลคือขาไม่ขับและไม่มี pull ซึ่งน่าจะลอย (high-Z) [INF] ขาที่ `esp_gpio_is_reserved` เป็นจริงจะถูกข้าม ยังไม่ทราบว่าขาที่ตั้งเป็น output ผ่าน gpio driver ถูกจองหรือไม่ ถ้าถูกจอง ข้อความเรื่อง "ขาถูก isolate" ใช้ไม่ได้กับขาเหล่านั้น [UNV] ถ้าไม่เปิด `gpio_deep_sleep_hold_en()` ฟังก์ชัน isolate `return` ทันทีและไม่แตะขา
- **หลักฐาน:** [VER] `sleep_gpio.c` ฟังก์ชัน `sleep_gpio_isolate_one_pin` สาขา `#else` และลูป `pad_mask` (EV-39 Z8) · `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` ไม่พบใน soc_caps.h ของ S3 (EV-39 Z7) · [VER] `esp_gpio_is_reserved` ถูกใช้เป็นเงื่อนไขข้ามขาในลูป isolate (`sleep_gpio.c:318`) และนิยามที่ `esp_hw_support/esp_gpio_reserve.c:25` · คอมเมนต์ `gpio.c:381` "output needs reserve IO which will make esp_gpio_is_reserved return true" (EV-40 Z15) ยังไม่เห็นผู้เรียก `esp_gpio_reserve` (รอ Z18, Z21)
- **ref:** gpio.md §0 ข้อ 24, §7.4 · **verified-in:** v6.1 (ตัวโค้ด) · **[HW]:** ยังไม่ทดสอบ (T4 ใช้ `gpio_hold_en` ครบจึงไม่น่าเจอ)