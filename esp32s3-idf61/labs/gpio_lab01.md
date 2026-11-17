# labs/gpio_lab01.md — ห้องทดลอง GPIO บน ESP32-S3 (ESP-IDF v6.1)

> เวอร์ชันแล็บ: v1.2 (v1 + EDIT 1-8 + EDIT 56-57) · คู่กับ `references/peripherals/gpio.md` v3.0-rc6
> ไฟล์นี้เก็บทุกอย่างของโปรเจกต์ไว้ในไฟล์เดียว: จุดประสงค์ วิธีสร้าง วิธีรัน ผลที่คาด สถานะทดสอบ และซอร์สทุกไฟล์ (ภาคผนวก A)
> วัตถุประสงค์หลัก: ใช้ฮาร์ดแวร์จริงยืนยันข้อเท็จจริงที่ gpio.md อ้างจาก header
> ป้ายหลักฐาน ([H61] [SOC] [VER] [INF] ฯลฯ) ความหมายอยู่ใน PROTOCOL.md A.2

## 1. จุดประสงค์

### ภาพใหญ่
GPIO บน ESP32-S3 ไม่ใช่แค่ "ขาเปิด/ปิด" มี 3 เรื่องที่ทำให้โค้ดพังเงียบๆ ได้
- **ความสามารถของชิป:** ขาไหนมีจริง (0-21, 26-48) ขาไหนเป็น RTC GPIO (0-21) ฟีเจอร์ไหนมีหรือไม่มีบน S3
- **การขัดจังหวะ (ISR):** ลำดับ install/add, กฎ IRAM, สิ่งที่ห้ามทำใน ISR
- **การนอนหลับ (sleep):** ปลุกด้วยขาอย่างไร และคงระดับขาข้าม deep sleep ได้อย่างไร

### ภาพย่อย (สิ่งที่โปรเจกต์นี้ตรวจ)
| หัวข้อ | ตรวจด้วย | ยืนยันข้อความใน gpio.md |
|---|---|---|
| ค่า SOC capability จริงของชิป | caps_probe.c | §2, §3 |
| รายการขาที่ valid, RTC, wake ได้ | caps_probe.c ตารางขา | §2 |
| Pin glitch filter ใช้ได้ และ delete ขณะ enable ต้อง error | caps_probe.c | §8 |
| ISR ปุ่มกดทำงาน (ไม่ใช้ IRAM_ATTR เมื่อ install flag 0) | btn_isr.c | §6 |
| Light-sleep ปลุกด้วย GPIO และ bit ของ wake cause | sleep_demos.c | §7 |
| Deep-sleep hold คงระดับขา | sleep_demos.c | §7 |

### การประยุกต์ใช้
- ปุ่มกดแบบ interrupt ที่ไม่กิน CPU: ใช้ pattern ISR + queue ใน btn_isr.c ได้ทันที
- อุปกรณ์แบตเตอรี่: ใช้ light-sleep ปลุกด้วยปุ่ม หรือ deep-sleep แล้วคงระดับ output (เช่นขาคุมรีเลย์หรือ LED)
- เขียนโค้ดที่ปรับตามชิปได้: ใช้แนวคิดใน caps_probe.c ตรวจ SOC_* ด้วย #if แทนการเดา

## 2. สถานะการทดสอบ

| รหัส | ทดสอบอะไร | สถานะ | ผล/หมายเหตุ |
|---|---|---|---|
| T0 | build ผ่านบน v6.1 | PASS (ผู้ใช้รายงาน: ไม่มี error) | v1 + EDIT 1 ผ่านแล้ว; v1.2 (EDIT 2-8, 56-57) ต้อง build ซ้ำ 1 ครั้ง |
| T1 | caps_probe: ค่า SOC_* + ตารางขา + glitch filter | PENDING-RUN | ต้องการแค่บอร์ด ไม่ต้องต่อวงจร (ผู้ใช้ติดปัญหา flash ของอุปกรณ์) |
| T2 | ISR ปุ่มกด (LAB_BTN_GPIO) | PENDING-HW | ต้องต่อปุ่ม หรือใช้ปุ่มบนบอร์ดถ้าตรวจ schematic แล้ว |
| T3 | Light-sleep ปลุกด้วย LAB_BTN_GPIO | PENDING-HW | ต้องมีปุ่ม |
| T4 | Deep-sleep hold LAB_HOLD_GPIO | PENDING-HW | ต้องต่อ LED + R หรือมัลติมิเตอร์ |

ป้ายสถานะ: PASS / FAIL / PENDING-RUN (รันได้เลย) / PENDING-HW (ต้องต่อวงจร) เมื่อทำเสร็จให้แก้คอลัมน์สถานะและกรอกผลใน §9

### ผลตรวจจากซอร์สจริงของ IDF v6.1 (ไม่ใช้บอร์ด) [VER]
ผู้ใช้รันสคริปต์ verify บนเครื่องที่ติดตั้ง IDF v6.1 ได้ผลดังนี้ (รายละเอียดใน `evidence/gpio-evidence.md` EV-32..34)
- ไม่นิยามบน S3 ใน `soc_caps.h`: `SOC_GPIO_SUPPORT_ETM`, `SOC_GPIO_FLEX_GLITCH_FILTER_NUM`, `SOC_GPIO_SUPPORT_PIN_HYS_*`, `SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP`, `SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED`, `SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN`
- นิยามบน S3: `SOC_DEDICATED_GPIO_SUPPORTED 1`, `SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER 1`, `SOC_GPIO_SUPPORT_FORCE_HOLD (1)`
- `gpio_wakeup_disable` ไม่คืน interrupt type เดิม (ถ้ากลับไปใช้ edge ISR ต้องเรียก `gpio_set_intr_type` ซ้ำ)
- ผล T1 จะยืนยันข้อความเหล่านี้บนบอร์ดอีกชั้นหนึ่ง

## 3. สิ่งที่ต้องมี
**ซอฟต์แวร์**
- ESP-IDF v6.1 (ตรวจด้วย `idf.py --version`)
- Terminal ที่ `. $IDF_PATH/export.sh` แล้ว (Windows ใช้ `export.ps1`)

**ฮาร์ดแวร์**
- บอร์ด ESP32-S3 พร้อมสาย USB (ของผู้ใช้เป็นบอร์ดที่มีจอและปุ่มบนบอร์ด)
- ปุ่มกด 1 ตัว (T2, T3) ถ้าใช้ปุ่มบนบอร์ด ต้องดู schematic ของบอร์ดก่อนว่าต่อขาใด
- LED 1 ดวง + ตัวต้านทาน 330 Ω ถึง 1 kΩ (T4) หรือมัลติมิเตอร์
- สายจัมเปอร์

## 4. โครงสร้างไฟล์ (ซอร์สอยู่ในภาคผนวก A)

```text
gpio_lab01/
├── CMakeLists.txt        ระดับโปรเจกต์ (include project.cmake, ตั้งชื่อ gpio_lab)
├── sdkconfig.defaults    ค่าเริ่มต้น (ตอนนี้ไม่บังคับอะไร)
└── main/
    ├── CMakeLists.txt    SRCS + PRIV_REQUIRES esp_driver_gpio esp_hw_support soc
    ├── gpio_lab.h        ประกาศฟังก์ชัน + ตั้งขาของแล็บ (LAB_BTN_GPIO, LAB_HOLD_GPIO)
    ├── main.c            เลือก DEMO_* ทีละโหมด
    ├── caps_probe.c      พิมพ์ SOC_* ตารางขา และทดสอบ glitch filter
    ├── btn_isr.c         ISR ปุ่มกด + queue + task
    └── sleep_demos.c     light-sleep demo และ deep-sleep hold demo
```
ไดเรกทอรี `build/` และไฟล์ `sdkconfig` ถูกสร้างโดย `idf.py` ไม่ต้องเก็บ

## 5. Build และ Flash

```bash
. $IDF_PATH/export.sh
idf.py --version                 # ต้องเป็น v6.1
cd ~/esp/projects/labs/gpio_lab01
idf.py set-target esp32s3        # ทำครั้งเดียว (หรือเมื่อเปลี่ยนชิป)
idf.py build
idf.py -p <PORT> flash monitor   # <PORT> เช่น /dev/ttyACM0 หรือ /dev/ttyUSB0
```
- ออกจาก monitor: กด Ctrl+]
- เก็บ log ลงไฟล์: รัน `script -f probe.log` ก่อน แล้วค่อย `idf.py -p <PORT> monitor`
- เปลี่ยนโหมดทดสอบ: แก้ `DEMO_*` ใน `main/main.c` (ตั้ง 1 เพียงตัวเดียว) แล้ว `idf.py build flash monitor` ใหม่
- เปลี่ยนขา: แก้ `LAB_BTN_GPIO` และ `LAB_HOLD_GPIO` ใน `main/gpio_lab.h` ที่เดียว

## 6. วงจร

```text
T2/T3 ปุ่มกด (ใช้ pull-up ภายใน ไม่ต้องมีตัวต้านทานภายนอก)

   LAB_BTN_GPIO ----+
                    |
                 [ปุ่ม]
                    |
   GND -------------+

T4 LED ตรวจ hold (ไฟต้องไม่ดับระหว่าง deep sleep)

   LAB_HOLD_GPIO ---[R 330Ω–1kΩ]---|>|--- GND
                                  LED (ขายาว = anode ต่อฝั่ง R)
```
- ใช้ไฟ 3.3 V เท่านั้น ห้ามป้อน 5 V เข้าขา GPIO
- ค่าเริ่มต้น `LAB_BTN_GPIO` = GPIO4 และ `LAB_HOLD_GPIO` = GPIO38 **ตรวจ schematic ของบอร์ดคุณก่อน** [GEN]: บอร์ดที่มีจอมักใช้ขาจำนวนมากสำหรับจอ ปุ่ม และ LED ถ้าขาถูกใช้อยู่ให้เปลี่ยนใน `gpio_lab.h`
- ห้ามเลือก: GPIO0/3/45/46 (strapping), GPIO19/20 (USB), GPIO26-32 (flash/PSRAM), GPIO33-37 (ถ้าโมดูลเป็น Octal), GPIO39-42 (JTAG), GPIO43/44 (UART0) [DS][TRM]
- `LAB_HOLD_GPIO` ควรเป็น digital GPIO 26-48 เพื่อทดสอบเส้นทาง `gpio_hold_en` + `gpio_deep_sleep_hold_en` (GPIO0-21 เป็น RTC GPIO ใช้เส้นทาง `rtc_gpio_hold_en` ซึ่งยังไม่ได้ทดสอบ)
- GPIO38 อยู่ในชุดขาที่ hold แบบ digital ได้ ตามรูปแบบของ `GPIO_HOLD_MASK` ที่คาด BIT(17) [INF]
- ถ้าใช้ปุ่ม BOOT ที่ GPIO0: ใช้ได้หลังบูต แต่ห้ามกดค้างตอน reset

## 7. ขั้นตอนทดสอบ

### T1: caps_probe (ไม่ต้องต่อวงจร)
1. ตั้ง `DEMO_BTN_ISR 1` (ค่าเริ่มต้น) แล้ว `idf.py -p <PORT> flash monitor`
2. กดปุ่ม EN/RESET บนบอร์ดเพื่อให้ boot ใหม่ แล้วคัดลอกข้อความตั้งแต่ `===== GPIO CAPS PROBE` ถึง `===== END PROBE =====`
3. วิธีอ่านผล
   - ค่าที่พิมพ์เป็นตัวเลข = macro นิยามไว้ เช่น `49`, `(1)`
   - ค่าที่พิมพ์ซ้ำชื่อ macro เอง (เช่น `SOC_GPIO_FLEX_GLITCH_FILTER_NUM  SOC_GPIO_FLEX_GLITCH_FILTER_NUM`) = macro **ไม่ได้นิยาม**
   - macro ที่เป็น mask จะพิมพ์เป็นนิพจน์ยาว ให้ดูว่ามี BIT22–BIT25 ถูกตัดออก

**ผลที่คาด** (ใช้เทียบกับของจริง)

| macro | คาด | ที่มา |
|---|---|---|
| SOC_GPIO_PIN_COUNT | 49 | [SOC][VER sdkconfig] |
| SOC_GPIO_VALID_GPIO_MASK | นิพจน์ที่ตัด BIT22-25 | [SOC] |
| SOC_GPIO_VALID_OUTPUT_GPIO_MASK | เท่ากับ VALID_GPIO_MASK | [SOC] |
| SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK | 0x0001FFFFFC000000ULL | [SOC] |
| SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER | 1 | [SOC][VER] |
| SOC_GPIO_FLEX_GLITCH_FILTER_NUM | ไม่นิยาม (พิมพ์ชื่อซ้ำ) | [SOC][VER] |
| SOC_GPIO_SUPPORT_PIN_HYS_FILTER | ไม่นิยาม | [SOC][VER] |
| SOC_GPIO_SUPPORT_PIN_HYS_CTRL_BY_EFUSE | ไม่นิยาม | [SOC][VER] |
| SOC_GPIO_SUPPORT_FORCE_HOLD | (1) | [SOC][VER] |
| SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP | ไม่นิยาม | [SOC][VER] |
| SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP | (1) | [SOC] |
| SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK | นิพจน์ BIT0..BIT21 | [SOC] |
| SOC_GPIO_SUPPORT_ETM | ไม่นิยาม | [VER] |
| SOC_DEDICATED_GPIO_SUPPORTED | 1 | [SOC][VER] |
| SOC_RTCIO_PIN_COUNT | 22 | [SOC] |
| SOC_RTCIO_INPUT_OUTPUT_SUPPORTED / HOLD_SUPPORTED / WAKE_SUPPORTED | 1 | [SOC] |
| SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED | ไม่นิยาม | [SOC][VER] |
| SOC_DEEP_SLEEP_SUPPORTED | 1 | [SOC] |
| CONFIG_GPIO_CTRL_FUNC_IN_IRAM | ไม่นิยาม (default n) | [H61 Kconfig] |

**ตารางขา** (49 แถว คอลัมน์ `valid out rtc hp_wake`)
- `valid` และ `out` = 1 ทั้งหมด 45 ขา (0-21 และ 26-48) และ = 0 ที่ 22-25
- `rtc` = 1 เฉพาะขา 0-21
- `hp_wake` = 1 เฉพาะขา 0-21 นอกนั้น 0 และไม่มีค่า -1 (ถ้ามี -1 แปลว่า macro ไม่นิยามบน S3)

**Pin glitch filter** (ทดสอบบน `LAB_BTN_GPIO` ชั่วคราว แล้วลบทิ้ง)
- ต้องเห็น `pin filter: del while enabled -> ESP_ERR_INVALID_STATE (expect ESP_ERR_INVALID_STATE)` [H61][T61]
- ต้องเห็น `flex filter: NOT supported` [SOC][VER]

**IO dump:** ต้องมีบล็อก `IO DUMP` ของ `LAB_BTN_GPIO` และ `LAB_HOLD_GPIO`

**ถ้าโปรแกรม reboot/abort ตอนทดสอบ glitch filter:** ส่งบรรทัด error แรกมา ซอร์ส v6.1 ของ `gpio_new_pin_glitch_filter` ตรวจแค่ config/ret_filter ไม่เป็น NULL และขา valid ไม่ได้บังคับตั้งขาเป็น input [C61] ดังนั้นไม่ควร abort จากเรื่องนี้

### T2: ISR ปุ่มกด (PENDING-HW)
1. ต่อปุ่มตามวงจร ตั้ง `DEMO_BTN_ISR 1` แล้ว flash
2. กดปุ่ม 1 ครั้ง
3. **คาด:** `I (...) btn: GPIO<n> falling edge, level=0` ขึ้น 1 บรรทัดต่อการกด
   - `level` อาจเป็น 1 ถ้าปล่อยปุ่มเร็วมาก เพราะอ่านค่าใน task หลัง ISR
   - กดซ้ำเร็วกว่า 50 ms จะถูกทิ้ง (debounce ใน task)
4. **ถ้าไม่ตรง:** ไม่มี log เลย → ตรวจว่าปุ่มต่อขาและ GND; log ขึ้นหลายบรรทัดต่อการกดครั้งเดียว → ปุ่มเด้งนานกว่า 50 ms (เพิ่ม delay ใน btn_isr.c)
5. **ยืนยันข้อความ:** gpio.md §6 (install flag 0 ไม่ต้อง IRAM_ATTR, ISR signature `void (*)(void *)`) [H61]

### T3: Light-sleep wake (PENDING-HW)
1. ตั้ง `DEMO_LIGHT_SLEEP 1` ต่อปุ่มตามวงจร แล้ว flash
2. **คาด:** log `light sleep; pull GPIO<n> low to wake` แล้วบอร์ดเงียบ (บางบอร์ดที่ใช้ USB native พอร์ต USB อาจหลุดชั่วคราว [GEN])
3. กดปุ่ม **คาด:** `woke: err=0 causes=0x80 gpio_bit=1`
   - `0x80` = BIT(7) = ESP_SLEEP_WAKEUP_GPIO เพราะ enum ประกาศเรียง UNDEFINED, ALL, EXT0, EXT1, TIMER, TOUCHPAD, ULP, GPIO โดยไม่กำหนดค่าตรงๆ จึงได้ 0..7 [H61] ผลนี้ใช้ยืนยันบนฮาร์ดแวร์ [HW]
4. **ถ้าตื่นทันทีโดยไม่กดปุ่ม:** ขาเป็น LOW อยู่ตอนเข้า sleep (ปุ่มค้างหรือสายผิด)
5. **ยืนยันข้อความ:** gpio.md §7 (gpio_wakeup_enable + esp_sleep_enable_gpio_wakeup, causes เป็น bitmap ต้องครอบ BIT())
6. **ข้อควรระวัง (GPIO-24):** `gpio_wakeup_enable` ตั้ง interrupt type ของขาเป็น level และ `gpio_wakeup_disable` ไม่คืนค่าเดิม [VER] ถ้าจะกลับไปใช้ edge ISR ต้องเรียก `gpio_set_intr_type(pin, GPIO_INTR_NEGEDGE)` ซ้ำ

### T4: Deep-sleep hold (PENDING-HW)
1. ต่อ LED ตามวงจร ตั้ง `DEMO_DEEP_SLEEP_HOLD 1` แล้ว flash
2. **คาด:** log `entering deep sleep; GPIO<n> held high` LED ติด และต้อง **ติดค้างตลอด 10 วินาที** ที่ deep sleep
3. ครบ 10 วินาทีบอร์ดบูตใหม่ **คาด:** `woke from deep sleep; GPIO<n> unheld, level kept high` และ LED ไม่กะพริบดับ
4. **ยืนยันข้อความ:** gpio.md §7.4 (hold_en + deep_sleep_hold_en, ตั้ง direction/level ก่อน hold_dis)
5. **ถ้า LED ดับระหว่างหลับ:** ส่งผลมา ข้อความที่เกี่ยวข้องคือ GPIO-05 และ GPIO-08

## 8. ปัญหาที่พบบ่อย

| อาการ | สาเหตุที่น่าจะเป็น | วิธีแก้ |
|---|---|---|
| ไม่เห็น probe log | พิมพ์ก่อนต่อ monitor | กด RESET แล้วดู |
| `#error` ตอน build | DEMO_* ไม่ได้ตั้งเป็น 1 เพียงตัวเดียว | แก้ main.c |
| Boot ไม่ขึ้นหลังต่อขา | ต่อขา strapping (0, 3, 45, 46) ผิด | ย้ายไปขาอื่น |
| ใช้ขา 26-32 แล้วเครื่องค้าง | ขาเหล่านี้ต่อ flash/PSRAM ในแพ็กเกจ [DS] | ห้ามใช้ |
| จอหรืออุปกรณ์บนบอร์ดทำงานผิดหลังต่อปุ่ม/LED | ขาที่เลือกชนกับจอ | ดู schematic แล้วเปลี่ยนใน `gpio_lab.h` |
| monitor ต่อไม่ได้ | พอร์ตผิด หรือ USB หลุดตอน sleep | ตรวจ `ls /dev/ttyACM* /dev/ttyUSB*` |
| flash ไม่ผ่าน | ปัญหาสาย/พอร์ต/โหมดบูตของบอร์ด | ลองสาย USB อื่น กด BOOT ค้างแล้วกด RESET ก่อน flash [GEN] |

## 9. ฟอร์มสำหรับส่งผลกลับ (คัดลอกไปกรอก)

```text
IDF version: 
Board/โมดูล (เช่น DevKitC-1 N16R8):
LAB_BTN_GPIO / LAB_HOLD_GPIO ที่ใช้:
T0: build v1.2 ผ่านหรือไม่ + warning (ถ้ามี):
T1: แนบข้อความจาก "===== GPIO CAPS PROBE" ถึง "===== END PROBE ====="
T2: PASS/FAIL + log ที่เห็น
T3: PASS/FAIL + บรรทัด "woke: ..." ที่เห็น
T4: PASS/FAIL + LED ติดค้างหรือไม่ + log ตอนตื่น
```

## 10. แบบฝึกต่อยอด
- เพิ่ม `ESP_INTR_FLAG_IRAM` ใน `gpio_install_isr_service()` แล้วใส่ `IRAM_ATTR` ให้ `btn_isr` ดูว่าต้องทำอะไรเพิ่ม (gpio.md §6)
- ปลุก deep sleep ด้วยปุ่ม (ต้องเป็น RTC GPIO 0-21) ผ่าน `esp_sleep_enable_ext1_wakeup_io` แล้วอ่านขาที่ปลุกด้วย `esp_sleep_get_ext1_wakeup_status()` (gpio.md §7.1)
- ทำ negative control ของ T4: ปิดการเรียก `gpio_deep_sleep_hold_en()` แล้วดูว่า LED ดับหรือไม่ (ยืนยัน GPIO-05)
- ทดสอบ GPIO-24: หลัง T3 เรียก `gpio_wakeup_disable` แล้วตรวจ interrupt type ด้วย `gpio_dump_io_configuration` ก่อนและหลังเรียก `gpio_set_intr_type`

## ภาคผนวก A: ซอร์สทั้งหมด (v1.2)

### A.1 CMakeLists.txt (ระดับโปรเจกต์)
```cmake
cmake_minimum_required(VERSION 3.22)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
idf_build_set_property(MINIMAL_BUILD ON)
project(gpio_lab)
```

### A.2 sdkconfig.defaults
```text
# ตั้งชิปด้วยคำสั่ง: idf.py set-target esp32s3
# เปิดเฉพาะเมื่อ ISR ของคุณเรียก gpio_set_level / gpio_intr_disable ในบริบท IRAM
# CONFIG_GPIO_CTRL_FUNC_IN_IRAM=y
```

### A.3 main/CMakeLists.txt
```cmake
idf_component_register(
    SRCS "main.c" "caps_probe.c" "btn_isr.c" "sleep_demos.c"
    INCLUDE_DIRS "."
    PRIV_REQUIRES esp_driver_gpio esp_hw_support soc
)
```

### A.4 main/gpio_lab.h
```c
#pragma once

#include "driver/gpio.h"

// ===== ขาที่ใช้ในแล็บ: แก้ที่นี่ที่เดียวให้ตรงกับบอร์ดของคุณ =====
// LAB_BTN_GPIO  : ปุ่มกด (ต่อลง GND ใช้ pull-up ภายใน) ใช้ใน T2 และ T3
//                 ต้องเป็นขาที่ไม่ชนจอหรืออุปกรณ์บนบอร์ด
//                 ถ้าใช้ปุ่ม BOOT ที่ GPIO0: ใช้ได้หลังบูต แต่ห้ามกดค้างตอน reset
// LAB_HOLD_GPIO : ขา output ของ T4 (LED หรือมัลติมิเตอร์)
//                 ควรเป็น digital GPIO 26-48 ที่ไม่ใช่ RTC เพื่อทดสอบเส้นทาง
//                 gpio_hold_en + gpio_deep_sleep_hold_en
//                 ห้ามใช้: 19/20 (USB), 26-32 (flash/PSRAM), 33-37 (ถ้าใช้ Octal),
//                          39-42 (JTAG), 43/44 (UART0), 45/46 (strapping)
#define LAB_BTN_GPIO   GPIO_NUM_4
#define LAB_HOLD_GPIO  GPIO_NUM_38

void caps_probe_run(void);
void btn_isr_init(void);
void light_sleep_gpio_demo(void);
void deep_sleep_hold_demo(void);
```

### A.5 main/main.c
```c
#include "gpio_lab.h"

// เลือกโหมดเดียวเท่านั้น ตั้งเป็น 1 หนึ่งตัว ที่เหลือเป็น 0
// แยกโหมดเพราะ light-sleep demo ใช้ขาเดียวกับ btn_isr (LAB_BTN_GPIO)
// และ gpio_wakeup_enable() ตั้ง interrupt type ของขาเป็น LOW/HIGH level
// (ยืนยันจาก gpio.c v6.1: gpio_hal_set_intr_type) จึงชนกับ ISR แบบ edge
#define DEMO_BTN_ISR           1
#define DEMO_LIGHT_SLEEP       0
#define DEMO_DEEP_SLEEP_HOLD   0

#if (DEMO_BTN_ISR + DEMO_LIGHT_SLEEP + DEMO_DEEP_SLEEP_HOLD) != 1
#error "ตั้ง DEMO_* ให้เป็น 1 เพียงตัวเดียว"
#endif

void app_main(void)
{
    caps_probe_run();
#if DEMO_BTN_ISR
    btn_isr_init();
#elif DEMO_LIGHT_SLEEP
    light_sleep_gpio_demo();
#else
    deep_sleep_hold_demo();
#endif
}
```

### A.6 main/caps_probe.c
ใช้แทน `soc_caps.h` ที่ขาด โดยพิมพ์ค่าจริงที่ compiler เห็น ถ้า macro ไม่ได้นิยาม มันจะพิมพ์ชื่อ macro ซ้ำ ซึ่งแปลว่า "ไม่มี"
```c
#include <stdio.h>
#include "sdkconfig.h"
#include "esp_err.h"
#include "soc/soc_caps.h"
#include "driver/gpio.h"
#include "driver/gpio_filter.h"
#include "driver/rtc_io.h"
#include "gpio_lab.h"

#define STR_(x) #x
#define STR(x)  STR_(x)
#define PROBE(m) printf("%-52s %s\n", #m, STR(m))

#if SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER
static void test_pin_glitch_filter(void)
{
    gpio_glitch_filter_handle_t f = NULL;
    const gpio_pin_glitch_filter_config_t cfg = {
        .clk_src  = GLITCH_FILTER_CLK_SRC_DEFAULT,
        .gpio_num = LAB_BTN_GPIO,
    };
    ESP_ERROR_CHECK(gpio_new_pin_glitch_filter(&cfg, &f));
    ESP_ERROR_CHECK(gpio_glitch_filter_enable(f));
    esp_err_t e = gpio_del_glitch_filter(f);
    printf("pin filter: del while enabled -> %s (expect ESP_ERR_INVALID_STATE)\n", esp_err_to_name(e));
    ESP_ERROR_CHECK(gpio_glitch_filter_disable(f));
    ESP_ERROR_CHECK(gpio_del_glitch_filter(f));
}
#endif

#if SOC_GPIO_FLEX_GLITCH_FILTER_NUM > 0
static void test_flex_glitch_filter(void)
{
    gpio_glitch_filter_handle_t f = NULL;
    const gpio_flex_glitch_filter_config_t cfg = {
        .clk_src         = GLITCH_FILTER_CLK_SRC_DEFAULT,
        .gpio_num        = GPIO_NUM_4,
        .window_width_ns = 100,
        .window_thres_ns = 75,
    };
    ESP_ERROR_CHECK(gpio_new_flex_glitch_filter(&cfg, &f));
    ESP_ERROR_CHECK(gpio_del_glitch_filter(f));
    printf("flex filter: create/delete OK\n");
}
#endif

static void print_pin_table(void)
{
    printf("\npin valid out rtc hp_wake(-1=macro absent)\n");
    for (int i = 0; i < GPIO_PIN_COUNT; i++) {
        int hp = -1;
#ifdef GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO
        hp = GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO(i) ? 1 : 0;
#endif
        printf("%3d %5d %3d %3d %5d\n", i,
               GPIO_IS_VALID_GPIO(i) ? 1 : 0,
               GPIO_IS_VALID_OUTPUT_GPIO(i) ? 1 : 0,
               rtc_gpio_is_valid_gpio((gpio_num_t)i) ? 1 : 0,
               hp);
    }
}

void caps_probe_run(void)
{
    printf("\n===== GPIO CAPS PROBE (ESP-IDF v6.1, esp32s3) =====\n");
    PROBE(SOC_GPIO_PIN_COUNT);
    PROBE(SOC_GPIO_VALID_GPIO_MASK);
    PROBE(SOC_GPIO_VALID_OUTPUT_GPIO_MASK);
    PROBE(SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK);
    PROBE(SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER);
    PROBE(SOC_GPIO_FLEX_GLITCH_FILTER_NUM);
    PROBE(SOC_GPIO_SUPPORT_PIN_HYS_FILTER);
    PROBE(SOC_GPIO_SUPPORT_PIN_HYS_CTRL_BY_EFUSE);
    PROBE(SOC_GPIO_SUPPORT_FORCE_HOLD);
    PROBE(SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP);
    PROBE(SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP);
    PROBE(SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK);
    PROBE(SOC_GPIO_SUPPORT_ETM);
    PROBE(SOC_DEDICATED_GPIO_SUPPORTED);
    PROBE(SOC_RTCIO_PIN_COUNT);
    PROBE(SOC_RTCIO_INPUT_OUTPUT_SUPPORTED);
    PROBE(SOC_RTCIO_HOLD_SUPPORTED);
    PROBE(SOC_RTCIO_WAKE_SUPPORTED);
    PROBE(SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED);
    PROBE(SOC_DEEP_SLEEP_SUPPORTED);
    PROBE(CONFIG_SOC_GPIO_FLEX_GLITCH_FILTER_NUM);
    PROBE(CONFIG_GPIO_CTRL_FUNC_IN_IRAM);

    print_pin_table();

#if SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER
    test_pin_glitch_filter();
#else
    printf("pin glitch filter: NOT supported\n");
#endif
#if SOC_GPIO_FLEX_GLITCH_FILTER_NUM > 0
    test_flex_glitch_filter();
#else
    printf("flex glitch filter: NOT supported\n");
#endif
    gpio_dump_io_configuration(stdout, (1ULL << LAB_BTN_GPIO) | (1ULL << LAB_HOLD_GPIO));
    printf("===== END PROBE =====\n");
}
```

### A.7 main/btn_isr.c
ตัวอย่างนี้ใช้ `gpio_install_isr_service(0)` ISR จึง**ไม่ใส่** `IRAM_ATTR` ตาม header [H61]
```c
#include <inttypes.h>
#include <stdint.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_err.h"
#include "esp_log.h"
#include "driver/gpio.h"
#include "gpio_lab.h"

#define BTN_GPIO LAB_BTN_GPIO

static const char *TAG = "btn";
static QueueHandle_t s_evt_q;

static void btn_isr(void *arg)
{
    uint32_t io = (uint32_t)(uintptr_t)arg;
    BaseType_t hp_woken = pdFALSE;
    xQueueSendFromISR(s_evt_q, &io, &hp_woken);
    if (hp_woken) {
        portYIELD_FROM_ISR();
    }
}

static void btn_task(void *arg)
{
    uint32_t io;
    TickType_t last = 0;
    for (;;) {
        if (xQueueReceive(s_evt_q, &io, portMAX_DELAY) == pdTRUE) {
            TickType_t now = xTaskGetTickCount();
            if ((now - last) < pdMS_TO_TICKS(50)) {   // debounce
                continue;
            }
            last = now;
            ESP_LOGI(TAG, "GPIO%" PRIu32 " falling edge, level=%d", io, gpio_get_level((gpio_num_t)io));
        }
    }
}

void btn_isr_init(void)
{
    s_evt_q = xQueueCreate(8, sizeof(uint32_t));
    configASSERT(s_evt_q);

    const gpio_config_t cfg = {
        .pin_bit_mask = 1ULL << BTN_GPIO,
        .mode         = GPIO_MODE_INPUT,
        .pull_up_en   = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type    = GPIO_INTR_NEGEDGE,
    };
    ESP_ERROR_CHECK(gpio_config(&cfg));

    esp_err_t err = gpio_install_isr_service(0);
    if (err != ESP_OK && err != ESP_ERR_INVALID_STATE) {   // ติดตั้งแล้วโดย component อื่นได้
        ESP_ERROR_CHECK(err);
    }
    ESP_ERROR_CHECK(gpio_isr_handler_add(BTN_GPIO, btn_isr, (void *)(uintptr_t)BTN_GPIO));
    xTaskCreate(btn_task, "btn", 3072, NULL, 5, NULL);
}
```

### A.8 main/sleep_demos.c
```c
#include <inttypes.h>
#include <stdint.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_err.h"
#include "esp_log.h"
#include "esp_bit_defs.h"
#include "esp_sleep.h"
#include "driver/gpio.h"
#include "soc/soc_caps.h"
#include "gpio_lab.h"

static const char *TAG = "sleep";

void light_sleep_gpio_demo(void)
{
    const gpio_num_t pin = LAB_BTN_GPIO;   // ต่อปุ่ม ถ้าขาเป็น LOW อยู่ตอนเข้า sleep จะตื่นทันที
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

    ESP_LOGI(TAG, "light sleep; pull GPIO%d low to wake", (int)pin);
    vTaskDelay(pdMS_TO_TICKS(200));   // รอ log ออกให้หมดก่อนเข้า sleep
    esp_err_t err = esp_light_sleep_start();

    uint32_t causes = (uint32_t)esp_sleep_get_wakeup_causes();
    ESP_LOGI(TAG, "woke: err=%d causes=0x%" PRIx32 " gpio_bit=%d",
             (int)err, causes, (causes & BIT(ESP_SLEEP_WAKEUP_GPIO)) ? 1 : 0);
}

#define HOLD_GPIO LAB_HOLD_GPIO

void deep_sleep_hold_demo(void)
{
    uint32_t causes = (uint32_t)esp_sleep_get_wakeup_causes();

    if (causes & BIT(ESP_SLEEP_WAKEUP_TIMER)) {
        // ตื่นจาก deep sleep: ตั้งสถานะให้เท่าเดิมก่อน แล้วค่อยปลด hold
        ESP_ERROR_CHECK(gpio_set_direction(HOLD_GPIO, GPIO_MODE_OUTPUT));
        ESP_ERROR_CHECK(gpio_set_level(HOLD_GPIO, 1));
        ESP_ERROR_CHECK(gpio_hold_dis(HOLD_GPIO));
#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP
        gpio_deep_sleep_hold_dis();
#endif
        ESP_LOGI(TAG, "woke from deep sleep; GPIO%d unheld, level kept high", (int)HOLD_GPIO);
        return;
    }

    ESP_ERROR_CHECK(gpio_set_direction(HOLD_GPIO, GPIO_MODE_OUTPUT));
    ESP_ERROR_CHECK(gpio_set_level(HOLD_GPIO, 1));
    ESP_ERROR_CHECK(gpio_hold_en(HOLD_GPIO));
#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP
    gpio_deep_sleep_hold_en();
#endif
    ESP_ERROR_CHECK(esp_sleep_enable_timer_wakeup(10ULL * 1000000ULL));   // 10 s
    ESP_LOGI(TAG, "entering deep sleep; GPIO%d held high", (int)HOLD_GPIO);
    esp_deep_sleep_start();   // ไม่ return
}
```
ฟังก์ชัน sleep ทั้งสองถูก compile ทุกครั้ง การ build จึงตรวจ API ให้ด้วย แต่ไม่ถูกเรียกจนกว่าจะตั้ง `DEMO_*` ไม่ได้ใช้ ext0/ext1 ในแล็บนี้ เพราะยังไม่ได้ build/ทดสอบ