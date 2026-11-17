# gpio.md — GPIO · ESP32-S3 · ESP-IDF v6.1

> เวอร์ชัน: v3.0-rc11 · สถานะ: RC (รอผล [HW] T1-T4 ก่อนเป็น v3.0 final)

> เป้าหมาย: ESP32-S3 เท่านั้น · ESP-IDF v6.1 (จุดที่ต่างจาก v5.x ระบุ marker เวอร์ชันไว้)
> คู่กับ: `hotlist/GPIO.md` (รหัสกับดัก GPIO-01..29) · `evidence/gpio-evidence.md` (ข้อความต้นฉบับ EV-xx) · `PROTOCOL.md`
> กฎ: ข้อที่เป็น [UNV] หรือ [GLM] ห้ามใช้เป็นข้อเท็จจริงในโค้ดที่ส่งมอบ

## ป้ายหลักฐาน (ท้ายทุกประโยคที่เป็นข้อเท็จจริง)

| ป้าย | ความหมาย |
|---|---|
| [H61] | header ของ v6.1 (esp_driver_gpio, esp_hal_gpio, esp_sleep.h) |
| [C61] | ซอร์ส .c ของ v6.1 |
| [T61] | test_apps ของ IDF v6.1 (ตัวอย่างการใช้จริง) |
| [SOC] | soc_caps.h, Kconfig.soc_caps.in, ไฟล์ใน soc/esp32s3 ของ v6.1 |
| [TRM] / [DS] | TRM v1.8 / Datasheet v2.2 |
| [MIG] | migration guide v6.x |
| [RST] | gpio.rst (docs) ที่อยู่ใน RAG (ยังไม่ยืนยันว่าตรงแท็ก v6.1) |
| [BUILD] | คอมไพล์และลิงก์ผ่านจริงบน v6.1 (ผู้ใช้รายงาน) |
| [VER] | ผลคำสั่ง grep/sed จาก `tools/verify_*.sh` ที่ผู้ใช้รันบนเครื่องที่ติดตั้ง IDF v6.1 (ต้นไม้ซอร์สจริง) |
| [HW] | ผลรันบนบอร์ดจริง (ยังไม่มีในเอกสารนี้) |
| [INF] | อนุมานจากโค้ด ยังไม่มีข้อความยืนยันตรงๆ |
| [GLM] | ข้อความจากเอกสารเดิม ยังไม่ตรวจซ้ำ |
| [UNV] | ยังไม่ยืนยัน |
| (EV-nn) | อ้างไปยังข้อความต้นฉบับใน `evidence/gpio-evidence.md` |

## 0. TL;DR (ตรวจแล้ว)

1. ขาที่มี = GPIO 0-21 และ 26-48 รวม 45 ขา (`SOC_GPIO_PIN_COUNT` = 49 นับรวมช่องว่าง 22-25) ไม่มีขา input-only [SOC][H61] (EV-01, EV-06)
2. RTC GPIO = GPIO 0-21 (22 ขา) ส่วน GPIO 26-48 เป็น digital IO pad เท่านั้น [SOC] (EV-01, EV-02)
3. `gpio_config()` ให้ `ESP_ERR_INVALID_ARG` เมื่อ `pin_bit_mask` เป็น 0, มีบิตนอก `SOC_GPIO_VALID_GPIO_MASK`, หรือขอโหมด output กับขาที่ output ไม่ได้ ใช้ `1ULL << pin` เสมอ [C61] (EV-09) ดู GPIO-19
4. `gpio_config()` **ตั้งค่าแบบเขียนทับ** (ไม่ใช่เพิ่ม) บังคับ function เป็น GPIO และเปิด interrupt ให้เองเมื่อ `intr_type != GPIO_INTR_DISABLE` [C61] (EV-09) ดู GPIO-29
5. `gpio_set_level()` ได้เฉพาะขา output ที่ valid; `gpio_get_level()` คืน 0 เสมอถ้า pad ไม่ได้เปิด input [H61][C61] (EV-10)
6. ISR service: `gpio_install_isr_service()` ก่อน `gpio_isr_handler_add()` install ซ้ำให้ `ESP_ERR_INVALID_STATE` ใช้ร่วมกับ `gpio_isr_register()` ไม่ได้ [H61][C61] (EV-07, EV-08)
7. `gpio_isr_t` = `void (*)(void *arg)` [H61]
8. handler ที่ผูกผ่าน `gpio_isr_handler_add` **ไม่ต้อง** `IRAM_ATTR` เว้นแต่ install ด้วย `ESP_INTR_FLAG_IRAM` [H61] (EV-07) ดู GPIO-09
9. ISR ที่เรียก `gpio_set_level`/`gpio_intr_disable` ในบริบทที่ cache ปิด ต้องเปิด `CONFIG_GPIO_CTRL_FUNC_IN_IRAM` [H61][Kconfig] (EV-10)
10. `gpio_wakeup_enable()` รับเฉพาะ LOW/HIGH level และ **ตั้ง interrupt type ของขานั้นเป็น level** จึงชนกับ ISR แบบ edge บนขาเดียวกัน [H61][C61] (EV-14) ดู GPIO-24, GPIO-27
11. คง digital GPIO (26-48) ข้าม deep sleep ต้องทำคู่: `gpio_hold_en(pin)` + `gpio_deep_sleep_hold_en()` ส่วน `gpio_hold_en()` บน GPIO 0-21 ส่งต่อไป `rtc_gpio_hold_en()` [H61][C61] (EV-11, EV-12) ดู GPIO-05
12. ปลด hold หลังตื่น: ตั้ง direction + level ให้เท่าเดิมก่อน `gpio_hold_dis()` [H61] (EV-11) ดู GPIO-08
13. `gpio_force_hold_all()` hold ขา flash และ UART ด้วย ฟังก์ชันนี้และโค้ดหลังจากนั้นต้องอยู่ internal RAM และ hold ถูกล้างตอน ROM boot; `gpio_force_unhold_all()` ปลดขาที่ถูก `gpio_hold_en` ไม่ได้ (OR logic) [H61] (EV-13) ดู GPIO-11, GPIO-12
14. S3 **ไม่มี** flex glitch filter และ **ไม่มี** hysteresis (`hys_ctrl_mode`) แต่ **มี** pin glitch filter, force hold, dedicated GPIO [SOC] (EV-01, EV-05)
15. ปลุก light sleep ด้วยขาใดก็ได้: `gpio_wakeup_enable()` + `esp_sleep_enable_gpio_wakeup()` ใช้ไม่ได้ถ้าเปิด `CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP` [H61] (EV-16) ดู GPIO-22
16. ปลุก deep sleep ด้วยขา: เฉพาะ RTC GPIO 0-21 ผ่าน EXT0, EXT1 หรือ `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown()` (ระดับ LOW/HIGH เท่านั้น ไม่มี edge) [H61][SOC] (EV-15, EV-16, EV-17)
17. EXT1 บน S3: โหมด `ESP_EXT1_WAKEUP_ANY_LOW`/`ANY_HIGH` เท่านั้น ตั้ง level ต่างกันรายขาไม่ได้ (ได้ `ESP_ERR_NOT_ALLOWED`) [H61][SOC][INF] (EV-03, EV-17) ดู GPIO-25, GPIO-26
18. `esp_sleep_get_wakeup_causes()` คืน `uint32_t` bitmap ตรวจด้วย `causes & BIT(ESP_SLEEP_WAKEUP_X)` ตัวเอกพจน์ `esp_sleep_get_wakeup_cause()` มี attribute deprecated [H61][T61] (EV-18, EV-22) ดู GPIO-17
19. v6.0 ลบหรือเปลี่ยน: `gpio_deep_sleep_wakeup_enable/disable`, `GPIO_IS_DEEP_SLEEP_WAKEUP_VALID_GPIO`, `MAX_PAD_GPIO_NUM`, `MAX_GPIO_NUM`, `DIG_IO_HOLD_BIT_SHIFT`, `gpio_iomux_in/out` [MIG] (EV-23)
20. GPIO3 ไม่มี internal pull ห้ามปล่อยลอย; strapping = GPIO0, 3, 45, 46; GPIO26-32 ต่อ flash/PSRAM ในแพ็กเกจ ห้ามใช้ [DS][TRM] (EV-24)
21. ห้ามใช้ `gpio_pad_select_gpio` ชื่อเปล่านี้ไม่มีประกาศใน v6.1 (grep `-w` ทั้ง components และ docs พบเพียง regex ใน `esp_rom/hints.yml`) เรียกใช้จะได้ `implicit declaration` หรือ `undefined reference` IDF แจ้ง hint ว่า "The rom GPIO functions have been renamed. Please replace with the APIs in 'esp_rom_gpio.h'" ครอบคลุม `gpio_pad_set_drv`, `gpio_pad_pullup`, `gpio_pad_select_gpio`, `gpio_pad_unhold`, `gpio_matrix_in`, `gpio_matrix_out` การแทนที่ตรงตัวคือ `esp_rom_gpio_pad_select_gpio(uint32_t)` (ประกาศใน `esp_rom_gpio.h`) ส่วนโค้ดแอปใหม่ให้ใช้ `gpio_reset_pin()` หรือ `gpio_config()` เพราะปิด interrupt และจัด pull ให้ (พฤติกรรมของฟังก์ชัน ROM ในส่วนนี้ยังเป็น [INF]) [C61][VER] (EV-09, EV-32, EV-33, EV-34) ดู GPIO-02
22. สัญญาณปลุกด้วยขา GPIO ทุกแบบ (level หรือ pulse) ต้องค้างอย่างน้อย 3 รอบ RTC slow clock มิฉะนั้นอาจไม่ถูกสุ่มตัวอย่าง ใช้กับทั้ง `esp_sleep_enable_gpio_wakeup()` และ `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown()` ความยาวต่อรอบขึ้นกับ `CONFIG_RTC_CLK_SRC` (RC_SLOW ~136 kHz ≈ 7.4 µs, XTAL32K ≈ 30.5 µs) [VER docs] (EV-38) ดู GPIO-31
23. เมื่อเปิด `gpio_deep_sleep_hold_en()` แล้วเรียก `esp_deep_sleep_start()` task ที่เรียกต้องมี stack อยู่ใน internal RAM มิฉะนั้น `assert` ล้ม [VER] (EV-38) ดู GPIO-32
24. เมื่อเปิด `gpio_deep_sleep_hold_en()` ก่อนเข้า deep sleep ระบบ isolate ขา digital (GPIO26-48) ที่ **ไม่ได้ถูก hold** และ **`esp_gpio_is_reserved` เป็นเท็จ**: ปิด input, output, pull-up, pull-down และตั้ง function เป็น GPIO ดังนั้นห้ามเชื่อว่าขาที่ไม่ได้ `gpio_hold_en()` จะคงระดับ ให้ `gpio_hold_en()` ทุกขาที่ต้องคงระดับเสมอ ถ้าไม่เปิด `gpio_deep_sleep_hold_en()` ฟังก์ชัน isolate `return` ทันที ขาที่ `esp_gpio_is_reserved` เป็นจริงจะถูกข้าม ฟังก์ชันตรวจนี้ถูกใช้ใน gpio driver, I2S, Ethernet MAC, PHY แต่ยังไม่เห็นว่าใครเรียก `esp_gpio_reserve` และยังไม่ยืนยันว่าขาที่ตั้งเป็น output ผ่าน gpio driver ถูกจองด้วยหรือไม่ [UNV] [VER] (EV-38 Z1, EV-39 Z8, EV-40 Z15) ดู GPIO-33


## 1. Include และ CMake

```c
#include "driver/gpio.h"         // API หลัก, gpio_num_t; gpio.h include driver/gpio_etm.h และ esp_rom_gpio.h ให้เอง
#include "driver/rtc_io.h"       // rtc_gpio_*
#include "driver/gpio_filter.h"  // glitch filter
#include "esp_sleep.h"           // wakeup API (component esp_hw_support)
#include "soc/soc_caps.h"        // SOC_* (component soc)
```

```cmake
idf_component_register(
    SRCS "main.c"
    INCLUDE_DIRS "."
    PRIV_REQUIRES esp_driver_gpio esp_hw_support soc
)
```
- build ผ่านด้วย CMake ชุดนี้บน v6.1 [BUILD]
- `esp_driver_gpio` ประกาศ `REQUIRES esp_hal_gpio` และ `PRIV_REQUIRES esp_pm` [C61 CMakeLists]
- ไฟล์ที่ build เสมอ: `gpio.c`, `gpio_glitch_filter_ops.c`, `rtc_io.c`
- ไฟล์ที่ build ตามเงื่อนไข: `dedic_gpio.c` (SOC_DEDICATED_GPIO_SUPPORTED), `gpio_pin_glitch_filter.c` (SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER), `gpio_flex_glitch_filter.c` (SOC_GPIO_FLEX_GLITCH_FILTER_NUM > 0), `gpio_etm.c` (SOC_GPIO_SUPPORT_ETM) [C61]
- Kconfig ของ component นี้มีข้อเดียว: `GPIO_CTRL_FUNC_IN_IRAM` (default n) "Place GPIO control functions (like intr_disable/set_level) into IRAM" [Kconfig] (EV-10)

## 2. ข้อเท็จจริงฮาร์ดแวร์ของ S3

| หัวข้อ | ค่า | ป้าย |
|---|---|---|
| ขาที่มี | 0-21 และ 26-48 = 45 ขา; `GPIO_NUM_21` ตามด้วย `GPIO_NUM_26` ใน enum | [SOC][H61] (EV-01, EV-06) |
| ขา input-only | ไม่มี (`VALID_OUTPUT_GPIO_MASK` = `VALID_GPIO_MASK`; ทุกขาในเอกสาร enum เขียน "input and output") | [SOC][H61] |
| Digital IO pad (VDD3P3_CPU หรือ VDD_SPI) | GPIO26-48 (`0x0001FFFFFC000000`) | [SOC] |
| RTC GPIO | GPIO 0-21 (22 ขา) ปลุกและ hold ใน deep sleep ได้ | [SOC][TRM] (EV-02) |
| Clock out | ผูกกับฟังก์ชัน IO_MUX, 3 channel | [SOC] |
| Dedicated GPIO | 8 ช่อง in + 8 ช่อง out ต่อ CPU core (API ใน `driver/dedic_gpio.h` ยังไม่ครอบคลุมในไฟล์นี้) | [SOC][H61] (EV-05) |
| ขา flash/PSRAM ในแพ็กเกจ | GPIO26-32 (SPICS1, SPIHD, SPIWP, SPICS0, SPICLK, SPIQ, SPID) ห้ามใช้อย่างอื่น | [DS] |
| Octal SPI | GPIO33-37 = SPIIO4-7 + SPIDQS เมื่อใช้ flash/PSRAM แบบ 8 line | [DS] |
| โดเมนไฟของ GPIO33-37 และ 47-48 | อาจใช้ไฟจาก VDD_SPI หรือ VDD3P3_CPU; VDD_SPI เลือกด้วย strapping GPIO45 (0 = ต่อ VDD3P3_RTC ผ่านตัวต้านทาน ปกติ 3.3 V, 1 = LDO ภายใน 1.8 V) หรือ eFuse | [TRM §6, §8.4] (EV-25) |
| ขาที่จองให้ interface | GPIO19/20 = USB, GPIO39-42 = JTAG, GPIO43/44 = UART0 | [DS][GLM] |
| Strapping | GPIO0 pull-up(1), GPIO3 ไม่มี internal pull (N/A), GPIO45 pull-down(0), GPIO46 pull-down(0) | [TRM ตาราง 8.1-1] (EV-24) |
| หน้าที่ strapping | GPIO0+GPIO46 = boot mode, GPIO45 = VDD_SPI, GPIO46 = ROM message, GPIO3 = JTAG source | [DS §3] |
| GPIO3 | "does not have any internal pull resistors and the strapping value must be controlled by the external circuit that cannot be in a high impedance state" | [DS §3] (EV-24) |
| JTAG source (Table 3-5) | default (STRAP_JTAG_SEL=0): ใช้ USB Serial/JTAG ไม่สนค่า GPIO3; GPIO3 มีผลเมื่อ DIS_PAD_JTAG=0, DIS_USB_JTAG=0, STRAP_JTAG_SEL=1 (GPIO3=1 → USB, GPIO3=0 → JTAG pins) | [DS][TRM 8.5-1] (EV-24) |
| ค่า reset ของบางขา (คอลัมน์ RST) | 0 = IE=0 (input ปิด), 1 = IE=1, 2 = IE=1+pull-down, 3 = IE=1+pull-up: GPIO4-8 = 0, GPIO9-14 = 1, GPIO15/16 = 0, GPIO17/18 = 1, GPIO19-21 = 0, GPIO26-29 = 3, GPIO44 = 3, GPIO45/46 = 2, GPIO47/48 = 1 (เฉพาะขาที่เห็นในข้อความที่ดึงมา) | [TRM ตาราง IO MUX] (EV-26) |
| Drive strength default หลัง reset | GPIO17/18 = ~10 mA (DRV 2), GPIO19/20 = ~40 mA (DRV 3), ขาอื่น = ~20 mA (DRV 2) | [TRM][DS] (EV-26) |
| ความหมายค่า DRV 0..3 | ขาทั่วไป 5/10/20/40 mA ส่วน GPIO17/18 = 5/20/10/40 mA (ค่า 1 กับ 2 สลับกัน) | [TRM] (EV-26) |
| ชื่อ enum drive | CAP_0 weak, CAP_1 stronger, CAP_2 medium (= DEFAULT), CAP_3 strongest, CAP_MAX (ไม่ใช่ค่าที่ตั้งได้) | [H61] (EV-27) |
| Digital hold ต่อขา | รีจิสเตอร์ `RTC_CNTL_DIG_PAD_HOLD_REG[n]` คุม GPIO26-48; `RTC_CNTL_DG_PAD_FORCE_HOLD` คุมทุกขา (OR กับ hold ต่อขา) | [TRM] (EV-30) |

อย่า hardcode เลข level ของ drive ให้ใช้ `gpio_set_drive_capability()` ด้วยชื่อ enum

## 3. Capability matrix ของ S3 (จาก soc_caps.h)

GPIO CAPS และ RTCIO CAPS อ่านได้เป็นบล็อกต่อเนื่อง "ไม่นิยาม" จึงหมายถึงไม่ปรากฏในบล็อกนั้นจริง (ตรวจซ้ำได้ด้วย `tools/verify_gpio_unv.sh` V2 และ caps_probe T1) (EV-01, EV-02, EV-03)

| macro | ค่าบน S3 | ผลต่อโค้ด |
|---|---|---|
| SOC_GPIO_PORT | 1U | มี GPIO peripheral 1 ตัว |
| SOC_GPIO_PIN_COUNT | 49 | ใช้เป็นขอบวนลูป (`GPIO_PIN_COUNT`) |
| SOC_GPIO_VALID_GPIO_MASK | `0x1FFFFFFFFFFFF & ~(BIT22..BIT25)` | ตรวจด้วย `GPIO_IS_VALID_GPIO(n)` |
| SOC_GPIO_VALID_OUTPUT_GPIO_MASK | = VALID_GPIO_MASK | ทุกขา output ได้ |
| SOC_GPIO_IN_RANGE_MAX / OUT_RANGE_MAX | 48 / 48 | |
| SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK | 0x0001FFFFFC000000 | digital pad = GPIO26-48 |
| SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER | 1 | ใช้ `gpio_new_pin_glitch_filter` ได้ |
| SOC_GPIO_FLEX_GLITCH_FILTER_NUM | ไม่นิยาม | ห้ามเรียก `gpio_new_flex_glitch_filter` ครอบด้วย `#if ... > 0` |
| SOC_GPIO_SUPPORT_PIN_HYS_FILTER / _HYS_CTRL_BY_EFUSE | ไม่นิยาม | `gpio_config_t` ไม่มี `hys_ctrl_mode` |
| SOC_GPIO_SUPPORT_FORCE_HOLD | (1) | มี `gpio_force_hold_all/unhold_all` |
| SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP | ไม่นิยาม | มี `gpio_deep_sleep_hold_en/dis` (เงื่อนไข `#if !...` เป็นจริง) |
| SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP | (1) | มี `gpio_wakeup_*_on_hp_periph_powerdown_sleep`, `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown` |
| SOC_GPIO_SUPPORT_DEEPSLEEP_WAKEUP | = HP_PERIPH_PD_SLEEP_WAKEUP | ชื่อเดิม (alias) |
| SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK | BIT0..BIT21 | ปลุกได้เฉพาะ GPIO 0-21 |
| SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED | ไม่นิยาม | wake ด้วย edge ใช้ไม่ได้ ได้เฉพาะ LOW/HIGH level |
| SOC_GPIO_CLOCKOUT_BY_IO_MUX / CHANNEL_NUM | (1) / (3) | clock out ผ่าน IO_MUX 3 channel |
| SOC_RTCIO_PIN_COUNT | 22 | `rtc_gpio_*` ใช้ได้กับ 22 ขา |
| SOC_RTCIO_INPUT_OUTPUT_SUPPORTED / HOLD_SUPPORTED / WAKE_SUPPORTED | 1 / 1 / 1 | `rtc_gpio_set_direction`, `rtc_gpio_hold_en`, `rtc_gpio_wakeup_enable`, `rtc_gpio_isolate` ใช้ได้ |
| SOC_LP_IO_CLOCK_IS_INDEPENDENT | 1 | driver เปิด/ปิด clock ของ LP IO เอง |
| SOC_PM_SUPPORT_EXT0_WAKEUP / EXT1_WAKEUP / EXT_WAKEUP | (1) / (1) / (1) | มี EXT0, EXT1 |
| SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN | ไม่นิยาม | EXT1 ตั้ง level ได้ชุดเดียวทุกขา |
| SOC_DEDICATED_GPIO_SUPPORTED | 1 | มี dedicated GPIO [SOC][VER] (EV-32) |
| SOC_GPIO_SUPPORT_ETM | ไม่นิยาม (grep ใน `soc_caps.h` ของ S3 ไม่พบ) | อย่าเรียก `gpio_new_etm_*` [VER] (EV-32) |
| SOC_DEEP_SLEEP_SUPPORTED | 1 | ใช้ deep sleep ได้ |

ผล grep บนเครื่องผู้ใช้ (v6.1) ค้น 9 macro ใน `soc_caps.h` ของ S3 พบเฉพาะ `SOC_DEDICATED_GPIO_SUPPORTED`, `SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER`, `SOC_GPIO_SUPPORT_FORCE_HOLD` ส่วน `SOC_GPIO_SUPPORT_ETM`, `SOC_GPIO_FLEX_GLITCH_FILTER_NUM`, `SOC_GPIO_SUPPORT_PIN_HYS_*`, `SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP`, `SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED`, `SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN` ไม่นิยามบน S3 [VER] (EV-32)

**คำเตือน:** ห้ามใช้ค่าจาก `CONFIG_SOC_*` (Kconfig/sdkconfig) แทน mask ในโค้ด C เพราะ Kconfig mirror เก็บ expression ไม่ได้ เช่น `CONFIG_SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK` เป็น 0 และ `CONFIG_SOC_GPIO_VALID_GPIO_MASK=0x1FFFFFFFFFFFF` (ไม่ตัด 22-25) ยืนยันจาก sdkconfig ของ gpio_lab01 จริง [SOC Kconfig.soc_caps.in][VER] (EV-04, EV-32) ดู GPIO-28

กฎ: โค้ดที่ต้องพอร์ตข้ามชิป ให้ครอบ API ที่ขึ้นกับ capability ด้วย `#if SOC_*` ตามตารางนี้

## 4. Types และ enums

```c
typedef void (*gpio_isr_t)(void *arg);                          // [H61]

typedef struct {
    uint64_t pin_bit_mask;          // 1ULL << n
    gpio_mode_t mode;
    gpio_pullup_t pull_up_en;       // GPIO_PULLUP_DISABLE=0 / GPIO_PULLUP_ENABLE=1
    gpio_pulldown_t pull_down_en;   // GPIO_PULLDOWN_DISABLE=0 / GPIO_PULLDOWN_ENABLE=1
    gpio_int_type_t intr_type;
#if SOC_GPIO_SUPPORT_PIN_HYS_FILTER // ไม่นิยามบน S3 → ฟิลด์นี้ไม่มี
    gpio_hys_ctrl_mode_t hys_ctrl_mode;
#endif
} gpio_config_t;                                                // [H61][SOC]
```

```c
// gpio_mode_t เป็นบิตแฟลก [H61] (EV-27)
// GPIO_MODE_DEF_INPUT = BIT0, GPIO_MODE_DEF_OUTPUT = BIT1, GPIO_MODE_DEF_OD = BIT2
GPIO_MODE_DISABLE          // = 0
GPIO_MODE_INPUT            // INPUT
GPIO_MODE_OUTPUT           // OUTPUT
GPIO_MODE_OUTPUT_OD        // OUTPUT | OD
GPIO_MODE_INPUT_OUTPUT_OD  // INPUT | OUTPUT | OD
GPIO_MODE_INPUT_OUTPUT     // INPUT | OUTPUT
```

- `gpio_int_type_t`: DISABLE=0, POSEDGE=1, NEGEDGE=2, ANYEDGE=3, LOW_LEVEL=4, HIGH_LEVEL=5 [H61]
- `gpio_pull_mode_t` (ใช้กับ `gpio_set_pull_mode`): PULLUP_ONLY, PULLDOWN_ONLY, PULLUP_PULLDOWN, FLOATING [H61] เป็นคนละ enum กับ `gpio_pullup_t`/`gpio_pulldown_t` ใน `gpio_config_t`
- `gpio_drive_cap_t`: CAP_0=0, CAP_1=1, CAP_2=2 (= CAP_DEFAULT), CAP_3=3, CAP_MAX [H61]
- `esp_sleep_ext1_wakeup_mode_t` บน S3: `ESP_EXT1_WAKEUP_ANY_LOW`=0, `ESP_EXT1_WAKEUP_ANY_HIGH`=1, `ESP_EXT1_WAKEUP_ALL_LOW` (มี attribute deprecated, alias ของ ANY_LOW) [H61] (EV-17)
- `esp_sleep_gpio_wake_up_mode_t`: `ESP_GPIO_WAKEUP_GPIO_LOW`=0, `ESP_GPIO_WAKEUP_GPIO_HIGH`=1; ค่า POSEDGE/NEGEDGE/ANYEDGE มีเฉพาะเมื่อ `SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED` (S3 ไม่มี) [H61] (EV-16)
- `gpio_io_config_t` (ใช้กับ `gpio_get_io_config`): `drv`, `fun_sel`, `sig_out`, `pu`, `pd`, `ie`, `oe`, `oe_ctrl_by_periph`, `oe_inv`, `od`, `slp_sel` และฟิลด์อื่นตามชิป [H61] (EV-27)
- ใช้ designated initializer เสมอ เพื่อให้โค้ดไม่พังเมื่อฟิลด์ต่างกันตามชิป [INF]

## 5. API (signature และ return ที่ตรวจแล้ว)

```c
// ----- ตั้งค่าขา -----
esp_err_t gpio_config(const gpio_config_t *pGPIOConfig);
    // INVALID_ARG: mask==0 / มีบิตนอก VALID_GPIO_MASK / ขอ output กับขาที่ output ไม่ได้ [C61]
    // เขียนทับ input/output/OD/pull ตามโครงสร้าง, บังคับ function = GPIO, ตั้ง intr_type แล้วเปิด/ปิด interrupt ตาม intr_type [C61]
esp_err_t gpio_reset_pin(gpio_num_t gpio_num);
    // ปิด interrupt; ขา output ได้จะเปิด pull-up (เพื่อไม่ให้ลอย) ปิด pull-down; เลือก function GPIO; deinit RTC IO ถ้าเป็น RTC [C61]
esp_err_t gpio_set_direction(gpio_num_t, gpio_mode_t);
    // เขียนทับโหมดเดิมทั้งหมด; INVALID_ARG ถ้าขอ output กับขาที่ output ไม่ได้ [H61][C61]
esp_err_t gpio_input_enable(gpio_num_t);                   // เปิด input อย่างเดียว ไม่เขียนทับค่าอื่น [H61]
esp_err_t gpio_set_level(gpio_num_t, uint32_t level);      // INVALID_ARG ถ้าไม่ใช่ขา output ที่ valid [C61]
int       gpio_get_level(gpio_num_t);                      // 0 เสมอถ้า pad ไม่ได้เปิด input [H61]
esp_err_t gpio_set_pull_mode(gpio_num_t, gpio_pull_mode_t); // เขียนทับ pull เดิม; INVALID_ARG ถ้า mode เกิน GPIO_FLOATING [H61][C61]
esp_err_t gpio_set_drive_capability(gpio_num_t, gpio_drive_cap_t);   // เฉพาะขา output; CAP_MAX → INVALID_ARG [C61][T61]
esp_err_t gpio_get_drive_capability(gpio_num_t, gpio_drive_cap_t *);

// ----- Interrupt -----
esp_err_t gpio_set_intr_type(gpio_num_t, gpio_int_type_t);
esp_err_t gpio_intr_enable(gpio_num_t);
esp_err_t gpio_intr_disable(gpio_num_t);                   // เรียกได้ในบริบท cache ปิดถ้าเปิด CONFIG_GPIO_CTRL_FUNC_IN_IRAM [H61]
esp_err_t gpio_install_isr_service(int intr_alloc_flags);
    // ESP_OK / ESP_ERR_NO_MEM / ESP_ERR_INVALID_STATE (ติดตั้งแล้ว) / ESP_ERR_NOT_FOUND (ไม่มี interrupt ว่างตาม flag) / ESP_ERR_INVALID_ARG [H61]
esp_err_t gpio_uninstall_isr_service(void);                // v6.0+ คืน esp_err_t [MIG][H61]
esp_err_t gpio_isr_handler_add(gpio_num_t, gpio_isr_t, void *args);
    // ESP_ERR_INVALID_STATE ถ้ายังไม่ install; ESP_ERR_INVALID_ARG ถ้าพารามิเตอร์ผิด [H61][C61]
esp_err_t gpio_isr_handler_remove(gpio_num_t);             // INVALID_STATE ถ้ายังไม่ install; ปิด interrupt ของขาและล้าง handler [H61][C61]
esp_err_t gpio_isr_register(void (*fn)(void *), void *arg, int intr_alloc_flags, gpio_isr_handle_t *handle);
    // ISR แบบ global ตัวเดียวสำหรับทุกขา; INVALID_ARG ถ้า fn เป็น NULL; NOT_FOUND ถ้าไม่มี interrupt ว่าง [H61][C61]
    // ใช้ร่วมกับ ISR service ไม่ได้

// ----- Hold -----
esp_err_t gpio_hold_en(gpio_num_t);
    // ขา RTC (0-21): ส่งต่อไป rtc_gpio_hold_en  [C61]
    // ขา digital ที่อยู่ใน GPIO_HOLD_MASK: ต้องเป็นขา output ได้ ไม่งั้น ESP_ERR_NOT_SUPPORTED  [C61]
    // ขาอื่น: ESP_ERR_NOT_SUPPORTED  [C61]
esp_err_t gpio_hold_dis(gpio_num_t);
#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP               // S3: เงื่อนไขเป็นจริง → ฟังก์ชันเหล่านี้มี [SOC]
void gpio_deep_sleep_hold_en(void);                        // hold ทุก digital pad ใน deep sleep ทุกรอบ sleep จนกว่าจะเรียก _dis [H61]
void gpio_deep_sleep_hold_dis(void);
#endif
#if SOC_GPIO_SUPPORT_FORCE_HOLD                            // S3: (1) [SOC]
esp_err_t gpio_force_hold_all(void);                       // hold ทุกขาทันที รวม flash/UART; ฟังก์ชันเป็น IRAM_ATTR [H61][C61]
esp_err_t gpio_force_unhold_all(void);                     // ไม่ปลดขาที่ gpio_hold_en ไว้ (OR); เรียก rtc_gpio_force_hold_dis_all ภายใน [H61][C61]
#endif

// ----- Sleep -----
esp_err_t gpio_sleep_sel_en(gpio_num_t);                   // ใช้ค่า SLP_SEL เปลี่ยนสถานะขาอัตโนมัติใน light sleep [H61]
esp_err_t gpio_sleep_sel_dis(gpio_num_t);
esp_err_t gpio_sleep_set_direction(gpio_num_t, gpio_mode_t);
esp_err_t gpio_sleep_set_pull_mode(gpio_num_t, gpio_pull_mode_t);
esp_err_t gpio_wakeup_enable(gpio_num_t, gpio_int_type_t); // เฉพาะ LOW_LEVEL/HIGH_LEVEL; INVALID_ARG ถ้าเป็น edge [H61][C61]
esp_err_t gpio_wakeup_disable(gpio_num_t);                 // ปิด wakeup และคืนค่า sleep_sel; ไม่เรียก gpio_hal_set_intr_type จึงไม่คืน interrupt type เดิม ถ้ากลับไปใช้ edge ISR ให้เรียก gpio_set_intr_type() ซ้ำ [VER] (EV-33)
#ifdef GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO   
             // macro ตรวจขา: (1ULL << n) & SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK [H61]
esp_err_t gpio_wakeup_enable_on_hp_periph_powerdown_sleep(gpio_num_t, gpio_int_type_t);
esp_err_t gpio_wakeup_disable_on_hp_periph_powerdown_sleep(gpio_num_t);
#endif

// ----- Debug -----
esp_err_t gpio_dump_io_configuration(FILE *out_stream, uint64_t io_bit_mask);   // INVALID_ARG ถ้า stream NULL หรือ mask มีบิตนอก valid [H61][C61] [BUILD]
esp_err_t gpio_get_io_config(gpio_num_t, gpio_io_config_t *out_io_config);      // [H61]
```

**รูปแบบผลลัพธ์ของ `gpio_dump_io_configuration`** (ใช้เทียบ T1) ต่อขาจะพิมพ์: Pullup/Pulldown/DriveCap, InputEn/OutputEn/OpenDrain, FuncSel (GPIO หรือ IOMUX), GPIO Matrix SigOut ID (บอก "simple GPIO output" เมื่อเป็น `SIG_GPIO_OUT_IDX`) และรายการ SigIn ID ที่ต่อกับขานั้น [C61] (EV-09)

## 6. Interrupt: กฎที่ตรวจแล้ว

- ลำดับมาตรฐาน: `gpio_config()` → `gpio_install_isr_service(flags)` → `gpio_isr_handler_add(pin, handler, arg)` [H61]
- `gpio_config()` เปิด interrupt ให้เองเมื่อ `intr_type != GPIO_INTR_DISABLE` ถ้าไม่ต้องการให้ใส่ `GPIO_INTR_DISABLE` [C61]
- ถ้า component อื่นติดตั้ง ISR service ไว้แล้ว `gpio_install_isr_service` คืน `ESP_ERR_INVALID_STATE` จึงห้ามครอบด้วย `ESP_ERROR_CHECK` ให้ยอมรับค่านี้ [C61] (EV-08) ดู GPIO-13
- handler ที่ผ่าน `gpio_isr_handler_add` ทำงานใน ISR และมีข้อจำกัด stack ซึ่งน้อยกว่า ISR แบบ global เพราะมีชั้นเรียกซ้อน [H61] ค่าที่ปรับได้คือ "ISR stack size" ใน menuconfig ซึ่งเป็น Kconfig `FREERTOS_ISR_STACKSIZE` (ใน sdkconfig คือ `CONFIG_FREERTOS_ISR_STACKSIZE`) ช่วง 1536-32768 ค่า default 1536 (ถ้าเปิด coredump: ช่วง 2096-32768 default 2096) เป็นขนาดต่อ 1 core และ S3 มี 2 core จึงใช้ RAM รวมสองเท่า [VER] (EV-37, EV-38)
- `ESP_INTR_FLAG_IRAM`: ตาราง handler ถูกจองใน internal RAM [C61] ISR และฟังก์ชันที่ ISR เรียกต้อง `IRAM_ATTR` ครบ [H61][GLM]
- ถ้าไม่ได้ส่ง `ESP_INTR_FLAG_IRAM` ไม่ต้องใส่ `IRAM_ATTR` ให้ handler: header เขียนว่า "The pin ISR handlers no longer need to be declared with IRAM_ATTR, unless you pass the ESP_INTR_FLAG_IRAM flag" [H61] (EV-07)
- ห้าม `printf`/`ESP_LOGx` ใน ISR ให้ส่งข้อมูลผ่าน `xQueueSendFromISR` แล้วพิมพ์ใน task [GLM][INF] ดู GPIO-03
- ISR service ผูกกับ core ที่ลงทะเบียน interrupt (`gpio_intr_enable_on_core(..., esp_intr_get_cpu(handle))`) [C61][INF]
- **ขาเดียวกันใช้ทั้ง ISR แบบ edge และ light-sleep wake ไม่ได้พร้อมกัน** เพราะ `gpio_wakeup_enable` ตั้ง interrupt type เป็น level (GPIO-24)

```c
// รูปแบบมาตรฐาน: ISR สั้นๆ + queue + task   [BUILD] (btn_isr.c ใน gpio_lab01) / [HW] รอ T2
static QueueHandle_t s_q;

static void btn_isr(void *arg)               // ไม่ใช้ IRAM_ATTR เพราะ install ด้วย flag 0
{
    uint32_t io = (uint32_t)(uintptr_t)arg;
    BaseType_t woken = pdFALSE;
    xQueueSendFromISR(s_q, &io, &woken);
    if (woken) { portYIELD_FROM_ISR(); }
}

void btn_init(void)
{
    s_q = xQueueCreate(8, sizeof(uint32_t));
    const gpio_config_t cfg = {
        .pin_bit_mask = 1ULL << GPIO_NUM_4,
        .mode         = GPIO_MODE_INPUT,
        .pull_up_en   = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type    = GPIO_INTR_NEGEDGE,
    };
    ESP_ERROR_CHECK(gpio_config(&cfg));
    esp_err_t e = gpio_install_isr_service(0);
    if (e != ESP_OK && e != ESP_ERR_INVALID_STATE) { ESP_ERROR_CHECK(e); }
    ESP_ERROR_CHECK(gpio_isr_handler_add(GPIO_NUM_4, btn_isr, (void *)(uintptr_t)GPIO_NUM_4));
}
```

## 7. Sleep, hold และ wake

### 7.1 ตัวเลือกปลุกด้วยขา

| ต้องการ | API | ขาที่ใช้ได้บน S3 | ข้อควรระวัง |
|---|---|---|---|
| ปลุก light sleep ด้วยระดับสัญญาณ | `gpio_wakeup_enable(pin, LOW/HIGH_LEVEL)` แล้ว `esp_sleep_enable_gpio_wakeup()` | ทุกขา (RTC หรือ digital) | เฉพาะ level และตั้ง intr type ของขา [C61]; ใช้ไม่ได้ถ้าเปิด `CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP` [H61]; รายงานเป็น `ESP_SLEEP_WAKEUP_GPIO` (light sleep เท่านั้นบน S3) [H61]; ถ้าขาอยู่ระดับที่กำหนดอยู่แล้วตอนเข้า sleep จะตื่นทันที [INF] |
| ปลุก deep sleep (และ light sleep แบบ peripheral power-down) | `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown(gpio_pin_mask, mode)` | GPIO 0-21 | mode = `ESP_GPIO_WAKEUP_GPIO_LOW`/`_HIGH` (ไม่มี edge) [H61][SOC]; ไม่แก้ config ขา ขาถูกตั้งภายใน `esp_*_sleep_start` ก่อนเข้า sleep [H61]; คืน INVALID_ARG ถ้า mask มีขาไม่รองรับหรือ mode ผิด [H61] |
| อ่านขาที่ปลุก (วิธีข้างบน) | `esp_sleep_get_gpio_wakeup_status()` → `uint64_t` bitmask ของ RTC IO แล้วแปลงด้วย `esp_sleep_wakeup_io_bit2num(uint32_t bit)` → `gpio_num_t` (คืน `GPIO_NUM_NC` ถ้า bit ไม่ถูกต้อง) | | มีเมื่อ `SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP` [H61] (EV-19) |
| EXT0 (ขาเดียว) | `esp_sleep_enable_ext0_wakeup(gpio, level)` (ใต้ `#if SOC_PM_SUPPORT_EXT0_WAKEUP`) | RTC GPIO 0-21 | INVALID_ARG ถ้าไม่ใช่ RTC GPIO หรือโหมดผิด, INVALID_STATE ถ้า trigger ชนกัน; ใช้ได้เมื่อ RTC peripherals ยังเปิดอยู่ใน sleep [H61] (EV-17) |
| EXT1 (หลายขา) | `esp_sleep_enable_ext1_wakeup_io(io_mask, mode)` (เพิ่มขาต่อจากที่มี ไม่รีเซ็ต) / `esp_sleep_disable_ext1_wakeup_io(io_mask)` (mask = 0 ลบทั้งหมด) | RTC GPIO 0-21 | mode บน S3 = `ANY_LOW` / `ANY_HIGH`; ได้ `ESP_ERR_NOT_ALLOWED` ถ้า level ต่างจากขาที่ตั้งไว้ เพราะ S3 ไม่มี per-pin [H61][INF]; ทำงานแม้ RTC peripherals ปิด แต่ pull ภายในใช้ไม่ได้ ต้องมีตัวต้านทานภายนอก [H61] |
| อ่านขาที่ปลุก EXT1 | `esp_sleep_get_ext1_wakeup_status()` → `uint64_t` BIT(n) = GPIOn | | ไม่พบ attribute deprecated บน declaration นี้ [H61] (EV-17) |
| EXT1 แบบเดิม | `esp_sleep_enable_ext1_wakeup(io_mask, mode)` | | header ระบุ "will be deprecated in release/v6.0" แต่ยังไม่มี attribute (คอมไพล์ได้) และ **รีเซ็ตค่า ext1 เดิมทุกครั้ง** ในโค้ดใหม่ใช้ `_io` [H61] |
| EXT1 ราย-level | `esp_sleep_enable_ext1_wakeup_with_level_mask` | | มีเฉพาะชิปที่ `SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN` (S3 ไม่มี) และมี attribute deprecated [H61][SOC] |

- ตรวจว่าขาใช้เป็นแหล่งปลุกได้หรือไม่: `esp_sleep_is_valid_wakeup_gpio(gpio_num_t)` → `bool` [H61]
- `gpio_wakeup_enable_on_hp_periph_powerdown_sleep()` มี header note "Called by the SDK. User shouldn't call this directly in the APP" แต่ [MIG] แสดงตัวอย่างเรียกตรง ใน APP ให้ใช้ `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown()` หรือ EXT1 เป็นหลัก ถ้าจำเป็นต้องเรียกตรง ตรวจขาด้วย `GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO(pin)` ขานอก 0-21 → `ESP_ERR_INVALID_ARG`, edge บน S3 → `ESP_ERR_INVALID_ARG` [H61][C61] (EV-15)
- `ESP_SLEEP_GPIO_ENABLE_INTERNAL_RESISTORS` เปิดเป็นค่าเริ่มต้น และตั้ง pull ภายในให้ตามโหมด wake ถ้าใช้ตัวต้านทานภายนอกให้ปิดตัวเลือกนี้ [H61] (EV-16)

- สัญญาณปลุกด้วยขา GPIO (level หรือ pulse) ต้องค้างอย่างน้อย 3 รอบ RTC slow clock จึงจะถูกสุ่มตัวอย่างได้แน่นอน ความยาวต่อรอบขึ้นกับ `CONFIG_RTC_CLK_SRC` เช่น RC_SLOW ~136 kHz ≈ 7.4 µs/รอบ (3 รอบ ≈ 22 µs) XTAL32K ≈ 30.5 µs/รอบ (3 รอบ ≈ 92 µs) ตัวเลข 3 รอบรวมถึงค่า µs ต่อรอบมาจาก docs ส่วนผลคูณ 22 µs และ 92 µs ผมคำนวณเอง ใช้กับทั้ง `esp_sleep_enable_gpio_wakeup()` และ `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown()` [VER docs: `sleep_modes.rst:392`] (EV-38) ข้อความนี้ไม่อยู่ใน `esp_sleep.h` และ `gpio.h` ดู GPIO-31 (S3 ไม่รองรับ edge wake ตาม §3 จึงใช้ level เป็นหลัก)

- ถ้าเปิด `CONFIG_ESP_SLEEP_GPIO_RESET_WORKAROUND` หรือ `CONFIG_PM_SLP_DISABLE_GPIO` ฟังก์ชัน enable/disable wakeup ใน `gpio.c` จะเรียก `gpio_hal_sleep_sel_en/dis` ภายใน [C61] (EV-14, EV-15)

### 7.2 ตัวอย่าง light sleep

```c
// [BUILD] sleep_demos.c ใน gpio_lab01 / [HW] รอ T3
const gpio_num_t pin = GPIO_NUM_4;
const gpio_config_t cfg = {
    .pin_bit_mask = 1ULL << pin,
    .mode         = GPIO_MODE_INPUT,
    .pull_up_en   = GPIO_PULLUP_ENABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type    = GPIO_INTR_DISABLE,
};
ESP_ERROR_CHECK(gpio_config(&cfg));
ESP_ERROR_CHECK(gpio_wakeup_enable(pin, GPIO_INTR_LOW_LEVEL));
ESP_ERROR_CHECK(esp_sleep_enable_gpio_wakeup());
vTaskDelay(pdMS_TO_TICKS(200));                 // รอ log ออกให้หมดก่อน (test ของ IDF รอ 1000 ms) [T61]
esp_err_t err = esp_light_sleep_start();
uint32_t causes = esp_sleep_get_wakeup_causes();
bool by_gpio = (causes & BIT(ESP_SLEEP_WAKEUP_GPIO)) != 0;     // ต้องครอบ BIT() [T61]
```

### 7.3 Wake cause

- `esp_sleep_get_wakeup_causes()` คืน `uint32_t` bitmap ของทุกแหล่งที่ปลุก (deep หรือ light sleep) [H61] (EV-18)
- `esp_sleep_get_wakeup_cause()` (เอกพจน์) มี `__attribute__((deprecated("use esp_sleep_get_wakeup_causes instead")))` และคืนแหล่งเดียว (ถ้าปลุกพร้อมกันหลายแหล่งข้อมูลอาจหาย) [H61]
- ลำดับ enum `esp_sleep_source_t` ที่ประกาศ: UNDEFINED, ALL, EXT0, EXT1, TIMER, TOUCHPAD, ULP, GPIO, UART0, ... ไม่กำหนดค่าตรงๆ จึงได้ UNDEFINED=0, ALL=1, EXT0=2, EXT1=3, TIMER=4, TOUCHPAD=5, ULP=6, GPIO=7 ดังนั้น bit ของ GPIO = `0x80` และ TIMER = `0x10` [H61] (EV-18) [HW] รอ T3 ยืนยัน
- `ESP_SLEEP_WAKEUP_GPIO` เป็นของ light sleep เท่านั้นบน S3 (คอมเมนต์ใน enum: "light sleep only on ESP32, S2 and S3") [H61]
- `ESP_SLEEP_WAKEUP_ALL` ไม่ใช่สาเหตุ ใช้ปิด wakeup source ทั้งหมด; `ESP_SLEEP_WAKEUP_UNDEFINED` = บูตปกติ ไม่ใช่การปลุกจาก deep sleep [H61]
- ตัวอย่าง EXT1 (ยังไม่ build): `esp_sleep_enable_ext1_wakeup_io(1ULL << GPIO_NUM_0, ESP_EXT1_WAKEUP_ANY_LOW);` แล้วตั้ง wake source อื่นและเรียก `esp_deep_sleep_start()` [H61 signature][UNV พฤติกรรม]

### 7.4 คงระดับ digital GPIO ข้าม deep sleep

ลำดับ (digital GPIO 26-48 เช่น GPIO38):
1. `gpio_set_direction(pin, GPIO_MODE_OUTPUT)` และ `gpio_set_level(pin, level)`
2. `gpio_hold_en(pin)`
3. `gpio_deep_sleep_hold_en()`
4. ตั้ง wake source แล้ว `esp_deep_sleep_start()`

หลังตื่น:
1. ตั้ง direction + level ให้เท่าเดิม
2. `gpio_hold_dis(pin)`
3. `gpio_deep_sleep_hold_dis()`

- `gpio_hold_en` latch ค่า input enable, output enable, output value, function และ drive strength ณ ตอนนั้น [H61] (EV-11)
- header ของ HAL ระบุชื่อ S3 ตรงๆ ว่าบน "ESP32/S2/C3/S3/C2" สถานะของ digital GPIO ถูก hold ใน deep sleep ไม่ได้ ต้องเรียก `gpio_deep_sleep_hold_en` ด้วย [H61] (EV-11)
- ถ้าไม่ตั้งสถานะก่อนปลด ขากลับเป็น input (ค่าเริ่มต้น) และระดับเปลี่ยน header ยกตัวอย่าง gpio18 ที่ hold ค่า high แล้วตื่นมาเป็น low [H61] (EV-11)
- `gpio_deep_sleep_hold_en` สถานะที่ถูก hold คือ active configuration ไม่ใช่ sleep configuration; กลไก hold ทำงานเฉพาะตอนอยู่ใน deep sleep ตอน active ขายังเปลี่ยนค่าได้ ยกเว้นขาที่ถูก `gpio_hold_en`; ค้างอยู่ทุกรอบ sleep จนกว่าจะเรียก `_dis` [H61] (EV-12)

- ขา RTC (0-21): `gpio_hold_en` ส่งต่อไป `rtc_gpio_hold_en` [C61] ซึ่งเรียก `rtcio_hal_hold_enable` = macro ของ `rtcio_ll_force_hold_enable(rtcio_num)` ตั้งบิต `rtc_io_desc[rtcio_num].hold_force` ใน `RTC_CNTL_PAD_HOLD_REG` [VER] (EV-37, EV-39 Z9) ส่วน `gpio_deep_sleep_hold_en()` ตั้ง `RTC_CNTL_DG_PAD_AUTOHOLD_EN_M` ใน `RTC_CNTL_DIG_ISO_REG` [VER] (EV-38 Z4) เป็นคนละรีจิสเตอร์ และลูป isolate เริ่มจาก `SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK` = GPIO26-48 ไม่รวม 0-21 [VER][SOC] (EV-01, EV-38 Z1) จึงสรุปในเชิงกลไกโค้ดได้ว่าขา RTC **ไม่ต้อง** คู่กับ `gpio_deep_sleep_hold_en()` พฤติกรรมบนบอร์ดยังไม่ทดสอบ [HW] T5

- ถ้าเรียก `gpio_deep_sleep_hold_en()` ไว้ `esp_sleep_isolate_digital_gpio` จะทำงานก่อนเข้า deep sleep ถ้าไม่เรียก ฟังก์ชันนั้น `return` ทันที (บรรทัด 286) [VER] (EV-38 Z1) ฟังก์ชันถูกเรียกจาก `esp_sleep_start_safe` ที่ `sleep_modes.c:862` ในสาขา `if (deep_sleep)` [VER] (EV-40 Z13) บน S3 `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` ไม่พบใน `soc/esp32s3` และ `esp_hal_gpio/esp32s3` จึงใช้สาขา `#else` [VER] (EV-39 Z7) ทุกขาใน `pad_mask` ที่ไม่ได้ถูก `gpio_hold_en` และ `esp_gpio_is_reserved` เป็นเท็จ จะถูก `sleep_gpio_isolate_one_pin` สั่ง input disable, output disable, pullup dis, pulldown dis, function = GPIO [VER] (EV-39 Z8) ส่วนขาที่ `esp_gpio_is_reserved` เป็นจริงจะถูกข้าม [VER] (EV-40 Z15) ยังไม่ทราบว่าขาที่ตั้งเป็น output ผ่าน gpio driver ถูกจองหรือไม่ (คอมเมนต์ `gpio.c:381` บอกว่า "output needs reserve IO") ถ้าถูกจอง ขานั้นจะไม่ถูก isolate [UNV] ดู GPIO-33
- ตอนตื่นจาก deep sleep ระบบ (`esp_sleep_startup_init`) เรียก `esp_deep_sleep_wakeup_io_reset()` เมื่อ reset reason = DEEPSLEEP และ `io_wakeup_enabled` ไม่เป็นศูนย์ โดย `IO_WAKEUP_ENABLED = (RTC_GPIO_TRIG_EN | _IO_WAKEUP_EXT0 | _IO_WAKEUP_EXT1)` [VER] (EV-39 Z8, EV-40 Z14) ฟังก์ชันนี้ปลด hold ให้ **เฉพาะขาที่เป็นแหล่งปลุก**: ขา EXT1 ผ่าน `rtcio_hal_hold_disable` และขา HP periph PD (GPIO0-21) ผ่าน `gpio_hal_hold_dis` [VER] (EV-39 Z8) ขาอื่นที่ hold ไว้ (เช่น GPIO38 ใน T4 ที่ปลุกด้วย timer) แอปต้อง `gpio_hold_dis` เอง [INF] เพราะ timer ไม่อยู่ในนิยามที่เห็นของ `IO_WAKEUP_ENABLED` (ยังไม่เห็นนิยาม `_IO_WAKEUP_EXT0/1` รอ Z20) ข้อสังเกต: บน S3 `GPIO_HOLD_MASK[0..21] = 0` (EV-35) และ `gpio_ll_hold_dis` ล้างบิตตาม mask นั้น ดังนั้น `gpio_hal_hold_dis` กับขา 0-21 ไม่แตะ `RTC_CNTL_DIG_PAD_HOLD_REG` [VER] (EV-40 Z17) ว่า hold ของ RTC pad ในเส้นทาง HP periph PD wake ถูกปลดที่อื่นหรือไม่ ยังไม่ทราบ [UNV] หลังตื่นจาก **light sleep** ด้วย EXT1 ให้เรียก `rtc_gpio_hold_dis` ก่อนตั้งค่าขาใหม่ เพราะถ้า RTC peripherals ถูกปิด (ค่าเริ่มต้น) ขาปลุกถูก hold ก่อนเข้า sleep ส่วน deep sleep ระบบจัดการตอน startup [VER docs: `sleep_modes.rst:272`] (EV-40 Z16)
- `gpio_ll_is_digital_io_hold` ของ S3 เรียก `abort()` ถ้าขาไม่อยู่ใน `SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK` (คือขา 0-25) จึงห้ามส่งขา RTC หรือขาที่ไม่มีจริงเข้า `gpio_hal_is_digital_io_hold` ในโค้ดระดับต่ำ [VER] (EV-40 Z17) ฟังก์ชัน driver ระดับบนอย่าง `gpio_hold_en` ตรวจชนิดขาก่อน (ขา RTC ไปที่ `rtc_gpio_hold_en`) [C61]
- ข้อกำหนด stack: เมื่อ deep-sleep hold เปิดอยู่ ฟังก์ชัน isolate ตรวจ `esp_ptr_internal(esp_cpu_get_sp())` ด้วย `assert("If hold digital IO, the stack of the task calling esp_deep_sleep_start must be in internal ram!")` [VER] (EV-38 Z1) บน S3 ฟังก์ชันนี้ถูกเรียกเฉพาะในเส้นทาง deep sleep (`sleep_modes.c:862`) ส่วนจุดเรียกใน light sleep (`:904`) อยู่ใต้ `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` ซึ่ง S3 ไม่นิยาม [VER] (EV-39 Z7, EV-40 Z13) ยังไม่ทราบว่าตอนเรียก isolate stack ยังเป็นของ task ที่เรียก `esp_deep_sleep_start` (รอ Z22) ดู GPIO-32

- ขา digital: `GPIO_HOLD_MASK[]` ของ S3 (`components/soc/esp32s3/gpio_periph.c`) มี 49 รายการ (ขนาดตรวจด้วย `_Static_assert` = `SOC_GPIO_PIN_COUNT`) ดัชนี 0-21 เป็น `0` (ขา RTC ใช้เส้นทาง `rtc_gpio_hold_en` ไม่ผ่านอาร์เรย์นี้) ดัชนี 22 = `BIT(1)` ไล่ต่อเนื่องถึงดัชนี 48 = `BIT(27)` คือดัชนี i ≥ 22 ได้ `BIT(i-21)` ตัวอย่าง GPIO26 = `BIT(5)`, GPIO38 = `BIT(17)`, GPIO48 = `BIT(27)` [VER] ตรงกับ TRM ที่ว่า GPIO26-48 ใช้บิต 5-27 ของ `RTC_CNTL_DIG_PAD_HOLD_REG` [TRM] (EV-30, EV-35) ดัชนี 22-25 มีค่าในอาร์เรย์แม้ไม่มีขาจริง แต่ `gpio_hold_en` ตรวจ valid output ต่อ จะได้ `ESP_ERR_NOT_SUPPORTED` [C61]
```c
// [BUILD] sleep_demos.c / [HW] รอ T4
#define HOLD_GPIO GPIO_NUM_38
// ก่อนหลับ
ESP_ERROR_CHECK(gpio_set_direction(HOLD_GPIO, GPIO_MODE_OUTPUT));
ESP_ERROR_CHECK(gpio_set_level(HOLD_GPIO, 1));
ESP_ERROR_CHECK(gpio_hold_en(HOLD_GPIO));
#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP
gpio_deep_sleep_hold_en();
#endif
ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup(10ULL * 1000000ULL));
esp_deep_sleep_start();
// หลังตื่น (causes & BIT(ESP_SLEEP_WAKEUP_TIMER))
ESP_ERROR_CHECK(gpio_set_direction(HOLD_GPIO, GPIO_MODE_OUTPUT));
ESP_ERROR_CHECK(gpio_set_level(HOLD_GPIO, 1));
ESP_ERROR_CHECK(gpio_hold_dis(HOLD_GPIO));
#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP
gpio_deep_sleep_hold_dis();
#endif
```

### 7.5 RTC GPIO (0-21)

```c
esp_err_t rtc_gpio_init(gpio_num_t);                                  // ย้ายขาไปโดเมน RTC [C61]
esp_err_t rtc_gpio_deinit(gpio_num_t);                                // กลับเป็น digital GPIO [C61]
esp_err_t rtc_gpio_set_direction(gpio_num_t, rtc_gpio_mode_t);        // [H61]
esp_err_t rtc_gpio_set_level(gpio_num_t, uint32_t level);             // [H61]
uint32_t  rtc_gpio_get_level(gpio_num_t);                             // [H61]
esp_err_t rtc_gpio_hold_en(gpio_num_t);                               // INVALID_ARG ถ้าไม่ใช่ RTC IO [H61][C61]
esp_err_t rtc_gpio_hold_dis(gpio_num_t);                              // ปลดแล้ว pad รับค่าจาก RTC_IO อีกครั้ง ค่าใน sleep อาจเปลี่ยน [H61]
esp_err_t rtc_gpio_force_hold_en_all(void);                           // ใช้คู่กับ gpio_force_hold_all [C61]
esp_err_t rtc_gpio_force_hold_dis_all(void);
esp_err_t rtc_gpio_isolate(gpio_num_t);                               // ปิด input/output/pull-up/pull-down แล้วเปิด hold ลด leakage ใน deep sleep; INVALID_ARG ถ้าไม่ใช่ RTC IO [H61][C61]
esp_err_t rtc_gpio_wakeup_enable(gpio_num_t, gpio_int_type_t);        // S3: เฉพาะ LOW/HIGH level; edge → INVALID_ARG [C61][H61]
esp_err_t rtc_gpio_wakeup_disable(gpio_num_t);
bool      rtc_gpio_is_valid_gpio(gpio_num_t);                         // [BUILD]
```
- `rtc_gpio_hold_en` latch ค่า input enable, output enable, output value, function, drive strength ของ pad [H61] (EV-20)
- `rtc_gpio_isolate` มีเงื่อนไข `SOC_RTCIO_HOLD_SUPPORTED && SOC_RTCIO_INPUT_OUTPUT_SUPPORTED` ซึ่งเป็นจริงบน S3 ตัวอย่างใน header พูดถึง ESP32-WROVER GPIO12 (ไม่เกี่ยวกับ S3) [H61][SOC] (EV-20)
- เปิด `CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP` แล้วควรเรียก `rtc_gpio_init()` ก่อนตั้ง wakeup [GLM] ดู GPIO-07

## 8. Glitch filter (S3 มีเฉพาะ pin filter)

```c
// [BUILD] caps_probe.c (compile ผ่าน) / [HW] รอ T1 (พฤติกรรมตอนรัน)
typedef struct {
    glitch_filter_clock_source_t clk_src;   // ใช้ GLITCH_FILTER_CLK_SRC_DEFAULT (ถ้าเป็น 0 driver ใช้ DEFAULT ให้เอง)
    gpio_num_t gpio_num;
} gpio_pin_glitch_filter_config_t;          // [H61][C61]

gpio_glitch_filter_handle_t f = NULL;
const gpio_pin_glitch_filter_config_t cfg = {
    .clk_src  = GLITCH_FILTER_CLK_SRC_DEFAULT,
    .gpio_num = GPIO_NUM_4,
};
ESP_ERROR_CHECK(gpio_new_pin_glitch_filter(&cfg, &f));
ESP_ERROR_CHECK(gpio_glitch_filter_enable(f));
// ... ใช้งาน ...
ESP_ERROR_CHECK(gpio_glitch_filter_disable(f));
ESP_ERROR_CHECK(gpio_del_glitch_filter(f));
```
- ตัด pulse ที่สั้นกว่า 2 sample clock ของ IO-MUX ค่าตายตัว เป็นการกรองสัญญาณระดับ ns ไม่ใช่ debounce ปุ่ม [H61] (EV-21)
- `gpio_new_pin_glitch_filter`: `INVALID_ARG` ถ้า config/ret_filter เป็น NULL หรือขาไม่ valid; `NO_MEM` ถ้าจองหน่วยความจำไม่ได้; ไม่บังคับตั้งขาเป็น input ก่อน [C61][H61]
- ฟิลเตอร์ปิดเป็นค่าเริ่มต้น ต้องเรียก `gpio_glitch_filter_enable` [RST]
- สถานะ: สร้าง = INIT → enable ได้เฉพาะตอน INIT → disable ได้เฉพาะตอน ENABLE; `gpio_del_glitch_filter` ขณะยัง enable → `ESP_ERR_INVALID_STATE`; `gpio_glitch_filter_enable` ซ้ำ → `ESP_ERR_INVALID_STATE` [C61][T61] (EV-21, EV-22)
- ถ้าเปิด `CONFIG_PM_ENABLE` ตอน enable จะ acquire PM lock และ release ตอน disable [C61]
- ตัวทดสอบของ IDF สร้าง config โดยไม่ใส่ `gpio_num` (เป็น GPIO0 โดยปริยาย) ห้ามลอกรูปแบบนั้นมาใช้ [T61]
- ชนิดที่ถูกต้อง: `glitch_filter_clock_source_t clk_src` และ `gpio_num_t gpio_num` [H61]
- Flex glitch filter ไม่มีบน S3 บล็อกด้านล่างนี้ใช้เฉพาะชิปที่ `SOC_GPIO_FLEX_GLITCH_FILTER_NUM > 0`:

```c
#if SOC_GPIO_FLEX_GLITCH_FILTER_NUM > 0
typedef struct {
    glitch_filter_clock_source_t clk_src;
    gpio_num_t gpio_num;
    uint32_t window_width_ns;   // ความกว้าง sample window
    uint32_t window_thres_ns;   // ห้ามมากกว่า window_width_ns
} gpio_flex_glitch_filter_config_t;   // [H61]
#endif
```

## 9. ใช้ขาร่วมกับ peripheral และฟีเจอร์เสริม

- driver ของ peripheral ตั้งค่าขาที่จำเป็นให้เองเมื่อต่อสัญญาณ peripheral ผู้ใช้รับผิดชอบเฉพาะการตั้งขาเป็น input/output ธรรมดา [RST]
- `gpio_config`, `gpio_reset_pin`, `gpio_set_direction`, `gpio_set_pull_mode` เป็น API ที่ **เขียนทับ** config เดิม (และ `gpio_config` บังคับ function = GPIO) จึงต้องเรียกก่อนเรียก API ของ peripheral ที่ต่อสัญญาณเข้าขา; ตัวอย่างใน gpio.rst: เรียก `gpio_config` เป็น `GPIO_MODE_INPUT` ก่อนเรียก LEDC แล้วอ่านขานั้นเป็น input ได้ [RST][C61] (EV-28, EV-31)
- ถ้าต้องการแค่เปิด input ให้เรียก `gpio_input_enable()` ได้ทุกเมื่อโดยไม่เขียนทับค่าอื่น [RST][H61]
- ภายใน driver ของ peripheral ให้ "เพิ่ม" config ด้วย `gpio_pullup_en`, `gpio_pulldown_en`, `gpio_od_enable` แทน API เขียนทับ [README ของ esp_driver_gpio] (EV-31)
- Dedicated GPIO (8 in / 8 out ต่อ core) ใช้ `driver/dedic_gpio.h`; ยังไม่ครอบคลุมในไฟล์นี้ [UNV] (ควรมีไฟล์ `dedic_gpio.md` แยก)
- ETM ของ GPIO (`gpio_etm.h`): บน S3 `SOC_GPIO_SUPPORT_ETM` ไม่นิยามใน `soc_caps.h` จึงอย่าใช้ [VER] (EV-32)

## 10. ดัชนี Hot-List ของกลุ่ม GPIO

ตัวรายการเต็ม (ผิด/ถูก/หลักฐาน) อยู่ใน `hotlist/GPIO.md` ตารางนี้ใช้ตรวจสองทาง (ทุกรหัสในไฟล์นี้ต้องมีใน hotlist และกลับกัน)

| รหัส    | tag     | หัวข้อ                                                                    | หัวข้อในไฟล์นี้      |
| ------- | ------- | ------------------------------------------------------------------------- | -------------------- |
| GPIO-01 | COMPILE | ISR ต้อง `void f(void *arg)`                                              | §0 ข้อ 7, §4         |
| GPIO-02 | COMPILE | `gpio_pad_select_gpio` (ใช้ `gpio_reset_pin`)                             | §0 ข้อ 21, §5        |
| GPIO-03 | RUNTIME | ห้าม printf/ESP_LOG ใน ISR                                                | §6                   |
| GPIO-04 | COMPILE | GPIO22-25 ไม่มีบน S3                                                      | §0 ข้อ 1, §2         |
| GPIO-05 | RUNTIME | hold_en อย่างเดียวคงขา digital ใน deep sleep ไม่ได้                       | §0 ข้อ 11, §7.4      |
| GPIO-06 | SILENT  | GPIO3 ไม่มี internal pull                                                 | §0 ข้อ 20, §2        |
| GPIO-07 | RUNTIME | เปิด PM power-down แล้วลืม rtc_gpio_init                                  | §7.5                 |
| GPIO-08 | SILENT  | gpio_hold_dis ก่อนตั้งสถานะ                                               | §0 ข้อ 12, §7.4      |
| GPIO-09 | SILENT  | ใส่ IRAM_ATTR ให้ ISR โดยเชื่อว่าจำเป็นเสมอ                               | §0 ข้อ 8, §6         |
| GPIO-10 | COMPILE | ชนิดใน glitch filter config ผิด                                           | §8                   |
| GPIO-11 | RUNTIME | force_unhold_all ปลด gpio_hold_en ไม่ได้                                  | §0 ข้อ 13            |
| GPIO-12 | RUNTIME | force_hold_all ในโค้ดที่อยู่ flash                                        | §0 ข้อ 13            |
| GPIO-13 | RUNTIME | ESP_ERROR_CHECK ครอบ install_isr_service                                  | §6                   |
| GPIO-14 | COMPILE | API ที่ถูกลบใน v6.0                                                       | §0 ข้อ 19            |
| GPIO-15 | COMPILE | gpio_iomux_in/out                                                         | §0 ข้อ 19            |
| GPIO-16 | COMPILE | ใช้ฟีเจอร์ที่ S3 ไม่มีโดยไม่ครอบ #if SOC_*                                | §3                   |
| GPIO-17 | SILENT  | wake causes ไม่ครอบ BIT()                                                 | §0 ข้อ 18, §7.3      |
| GPIO-18 | RUNTIME | ใช้ GPIO26-32 (หรือ 33-37 เมื่อ Octal)                                    | §2                   |
| GPIO-19 | SILENT  | `1UL << pin`                                                              | §0 ข้อ 3             |
| GPIO-20 | RUNTIME | gpio_get_level บน pad ที่ไม่ได้เปิด input                                 | §0 ข้อ 5             |
| GPIO-21 | RUNTIME | gpio_wakeup_*_on_hp_periph_powerdown_sleep ด้วยขา/edge ผิด                | §7.1                 |
| GPIO-22 | RUNTIME | esp_sleep_enable_gpio_wakeup ขณะเปิด PM power-down                        | §0 ข้อ 15, §7.1      |
| GPIO-23 | RUNTIME | ใช้ API ที่ deprecated ใน v6.x                                            | §7.1, §7.3           |
| GPIO-24 | RUNTIME | gpio_wakeup_enable ชนกับ ISR edge บนขาเดียวกัน                            | §0 ข้อ 10, §6        |
| GPIO-25 | RUNTIME | EXT1 ตั้ง level ต่างกันรายขาบน S3                                         | §0 ข้อ 17, §7.1      |
| GPIO-26 | COMPILE | ESP_EXT1_WAKEUP_ALL_LOW บน S3                                             | §0 ข้อ 17, §4        |
| GPIO-27 | RUNTIME | ใช้ edge type กับ wakeup บน S3                                            | §0 ข้อ 10, §7.1      |
| GPIO-28 | SILENT  | ใช้ CONFIG_SOC_* (Kconfig) แทน mask จาก soc_caps.h                        | §3                   |
| GPIO-29 | SILENT  | เรียก API เขียนทับหลัง peripheral init บนขาเดียวกัน                       | §0 ข้อ 4, §9         |
| GPIO-30 | COMPILE | ชื่อ ROM GPIO เก่า (gpio_pad_pullup, gpio_matrix_in ฯลฯ)                  | §0 ข้อ 21, §13 ข้อ 4 |
| GPIO-31 | SILENT  | สัญญาณปลุกด้วยขาสั้นกว่า 3 รอบ RTC slow clock                             | §0 ข้อ 22, §7.1      |
| GPIO-32 | RUNTIME | deep sleep พร้อม digital hold แต่ stack อยู่ใน PSRAM (assert)             | §0 ข้อ 23, §7.4      |
| GPIO-33 | SILENT  | เปิด deep-sleep hold แต่ไม่ gpio_hold_en ขาที่ต้องคงระดับ (ขาถูก isolate) | §0 ข้อ 24, §7.4      |

## 11. ตัวอย่างและสถานะทดสอบ

โปรเจกต์ตัวอย่าง `gpio_lab01` (วิธีสร้าง วิธีรัน ผลที่คาด สถานะทดสอบ และซอร์สทั้งหมดอยู่ในไฟล์เดียว `labs/gpio_lab01.md`) ปรับขาที่ `main/gpio_lab.h`

| รหัส | เนื้อหา | build | รันจริง |
|---|---|---|---|
| T1 | caps_probe | [BUILD] | PENDING-RUN |
| T2 | ISR ปุ่ม (LAB_BTN_GPIO) | [BUILD] | PENDING-HW |
| T3 | light-sleep wake (LAB_BTN_GPIO) | [BUILD] | PENDING-HW |
| T4 | deep-sleep hold (LAB_HOLD_GPIO) | [BUILD] | PENDING-HW |

## 12. เช็คลิสต์เลือกขาให้บอร์ดของคุณ

| ขา | ปัญหา | ทำอย่างไร |
|---|---|---|
| 0, 3, 45, 46 | strapping | อย่าให้วงจรภายนอกดึงระดับผิดตอน reset; GPIO3 ห้ามลอย [TRM][DS] |
| 19, 20 | USB (D-/D+) | ห้ามใช้ถ้าใช้ USB-Serial-JTAG ตัวบอร์ดเป็นช่องบันทึก/แฟลช [DS][GLM] |
| 26-32 | flash/PSRAM ในแพ็กเกจ | ห้ามใช้ [DS] |
| 33-37 | Octal flash/PSRAM | ห้ามใช้ถ้าโมดูลเป็น Octal; ตรวจรุ่นโมดูลของคุณ [DS] |
| 39-42 | JTAG | หลีกเลี่ยงถ้าจะดีบัก JTAG [DS][GLM] |
| 43, 44 | UART0 (log) | หลีกเลี่ยง [DS][GLM] |
| ขาของจอ/ปุ่ม/LED บนบอร์ด | ขึ้นกับรุ่นบอร์ด | ดู schematic ของบอร์ดคุณ ห้ามเดา [GEN] |

- สำหรับ T3 (light-sleep wake) ขาปุ่มใดก็ได้ที่ valid; สำหรับปลุก deep sleep ต้องเป็น RTC GPIO 0-21
- สำหรับ T4 ใช้ขา digital 26-48 ที่ไม่ชนข้อห้ามข้างบน ตัวอย่างคือ GPIO38 ตรวจกับ schematic ว่าไม่ต่ออุปกรณ์ในบอร์ด

## 13. ที่ยังไม่ยืนยัน (ปิดแล้วลบออกจากรายการ)

1. EXT1 ราย-level บน S3 (ยืนยันด้วยผลของ `esp_sleep_enable_ext1_wakeup_io` สองครั้งที่ level ต่างกัน) [HW]


2. `gpio_hold_en` บน GPIO 0-21 ใน deep sleep ต้องคู่ `gpio_deep_sleep_hold_en()` หรือไม่: โค้ดบอกว่าเป็นคนละรีจิสเตอร์และไม่ต้องคู่ [VER] (ดู §7.4) ผลบนบอร์ดรอ [HW] T5
3. ข้อความ "deprecated v5.0" ของ `gpio_pad_select_gpio` ยังเป็น [GLM] (ไม่กระทบโค้ด เพราะชื่อเปล่าใช้ไม่ได้อยู่แล้ว)
4. พฤติกรรมของ `esp_rom_gpio_pad_select_gpio` เทียบกับ `gpio_reset_pin`: header บอกเพียง "Configure IO Pad as General Purpose IO, so that it can be connected to internal Matrix" [VER] (EV-37) ไม่บอกว่าปิด interrupt หรือจัด pull ตัวฟังก์ชันอยู่ใน ROM ตรวจซอร์สไม่ได้ จึงยังเป็น [INF] ว่าไม่ทำสองอย่างนั้น ในโค้ดแอปใช้ `gpio_reset_pin()` หรือ `gpio_config()`

5. ผู้จองจริงของ `esp_gpio_reserve` และขาที่ตั้งเป็น output ผ่าน gpio driver ถูกจองหรือไม่ (กระทบกลไกของ GPIO-33 และว่าขานั้นถูกข้ามการ isolate) รัน `verify_gpio_unv8.sh` Z18, Z21
6. `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` ทั้ง `components/` (Z19), นิยาม `_IO_WAKEUP_EXT0/_EXT1` และ `RTC_GPIO_TRIG_EN` (Z20), stack ตอนเรียก isolate (Z22), และ `gpio_hal_hold_dis` กับขา 0-21 บน S3 (mask = 0 ไม่แตะ `RTC_CNTL_DIG_PAD_HOLD_REG`) ว่า hold ของ RTC pad ในเส้นทาง HP periph PD wake ถูกปลดที่ใด [INF]


7.  เดิม 8 . หัวข้อ TRM `RTC_CNTL_EXT_WAKEUP1_SEL[17:0]` (18 บิต) ขัดกับ RTC IO 22 ขา ให้ยึด `esp_sleep.h` (0-21) [UNV]
8.  เดิม 9. GPIO47/48 ที่ 1.8 V บนรุ่น R8V/R16V (TRM ยืนยันแค่โดเมนไฟ) ตรวจรุ่นโมดูล
9.  เดิม 10. gpio.rst ใน RAG ตรงกับแท็ก v6.1 หรือไม่ (ใช้เป็นหลักฐานรอง)
10.  เดิม 11. ผล [HW] ของ T1-T4

## 14. บันทึกการตรวจ (สรุป)

- รอบ 2: ตรวจ gpio.md ของ GLM (v2.3) กับ header v6.1 พบข้อผิด 11 จุด (ดู CHANGELOG)
- รอบ 3: ปิด [UNV] ด้านความสามารถของ S3; build ผ่านบน v6.1 [BUILD]
- รอบ 4: ปิด [UNV] เพิ่ม (ETM, dedicated GPIO, EXT1 per-pin, enum cause, glitch filter, rtc_gpio_isolate, ISR API, gpio_wakeup_enable); เพิ่ม Hot-List เป็น 29 ข้อ; เก็บข้อความต้นฉบับใน `evidence/gpio-evidence.md` เพื่อให้ถอดไฟล์ต้นทางออกจาก RAG ได้
- ยังไม่ได้ค้นเว็บใน gpio.md นี้ (ทุกข้อมาจาก RAG)