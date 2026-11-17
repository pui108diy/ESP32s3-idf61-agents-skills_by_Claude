
## verify_gpio_unv4.sh

``` bash
#!/usr/bin/env bash
# verify_gpio_unv4.sh (รอบ 5e): ปิดข้อ [GLM]/[INF] ที่เหลือของ gpio.md ด้วยการอ่านซอร์ส IDF v6.1 จริง (อ่านอย่างเดียว)
# Y1 พิสูจน์: ข้อความ "ISR stack size" ใน menuconfig ที่ header ของ gpio.h อ้างถึง มีอยู่จริงและชื่อ config คืออะไร
# Y2 พิสูจน์: ข้อความ "สัญญาณปลุกต้องค้าง 3 รอบ RTC slow clock" มีใน header หรือไม่
# Y3 พิสูจน์: ชื่อ ROM เก่า (gpio_pad_* / gpio_matrix_*) จับคู่กับชื่อ esp_rom_gpio_* ใหม่อย่างไรบน S3
# Y4 พิสูจน์: signature และคำอธิบายของ esp_rom_gpio_* ครบทุกฟังก์ชัน
# Y5-Y8 พิสูจน์: ขา RTC GPIO (0-21) ที่ใช้ gpio_hold_en ต้องเรียก gpio_deep_sleep_hold_en คู่กันหรือไม่ (ข้อ [HW] ที่ค้าง)
# วิธีใช้
#   . $IDF_PATH/export.sh
#   bash verify_gpio_unv4.sh > verify4.log 2>&1
set -u
: "${IDF_PATH:?ต้อง source export.sh ก่อน}"
C="$IDF_PATH/components"

echo "### Y0 รุ่นของ IDF"
(cd "$IDF_PATH" && git describe --tags 2>&1)

echo "### Y1 ข้อความ 'ISR stack' (Kconfig ของ esp_driver_gpio, esp_system, freertos)"
grep -rniE "isr stack|ISR_STACK" "$C/esp_driver_gpio" "$C/esp_system" "$C/freertos" --include=Kconfig* --include=gpio.h
echo "(จบ Y1: ไม่มีบรรทัด = ไม่พบข้อความนี้ในที่ค้น)"

echo "### Y2 'slow clock' ใน esp_sleep.h และ gpio.h"
grep -n -i -B2 -A2 "slow clock" "$C/esp_hw_support/include/esp_sleep.h" "$C/esp_driver_gpio/include/driver/gpio.h"
echo "(จบ Y2: ไม่มีบรรทัด = ไม่พบข้อความนี้)"

echo "### Y3 ชื่อ ROM เก่า ใน linker script ของ S3"
grep -rn -E "gpio_pad_(select_gpio|pullup|unhold|set_drv)|gpio_matrix_(in|out)" "$C/esp_rom/esp32s3" --include=*.ld
echo "(จบ Y3: ทุกบรรทัดควรเป็นรูป PROVIDE ( esp_rom_gpio_xxx = rom_gpio_xxx ) ใช้ดูการจับคู่ชื่อ)"

echo "### Y4 esp_rom_gpio.h ฉบับเต็ม"
sed -n '1,80p' "$C/esp_rom/include/esp_rom_gpio.h"
echo "(จบ Y4)"

echo "### Y5 gpio_deep_sleep_hold_en ใน gpio.c"
sed -n '/^void gpio_deep_sleep_hold_en/,/^}/p' "$C/esp_driver_gpio/src/gpio.c"
echo "(จบ Y5)"

echo "### Y6 rtc_gpio_hold_en ใน rtc_io.c"
sed -n '/^esp_err_t rtc_gpio_hold_en/,/^}/p' "$C/esp_driver_gpio/src/rtc_io.c"
echo "(จบ Y6)"

echo "### Y7 rtcio_ll_hold_en ของ S3"
grep -rn -A8 "rtcio_ll_hold_en" "$C/esp_hal_gpio/esp32s3"
echo "(จบ Y7)"

echo "### Y8 จุดที่ esp_hw_support เรียก hold ของ GPIO/RTC IO ตอนเข้า sleep"
grep -rn -E "gpio_deep_sleep_hold|gpio_hal_deep_sleep_hold|rtc_gpio_force_hold|rtc_gpio_init" "$C/esp_hw_support" --include=*.c --include=*.h
echo "(จบ Y8)"
```

## verify4.log ผลที่ได้

``` code
### Y0 รุ่นของ IDF
v6.1
### Y1 ข้อความ 'ISR stack' (Kconfig ของ esp_driver_gpio, esp_system, freertos)
/root/esp/esp-idf/components/esp_driver_gpio/include/driver/gpio.h:364:  * size limit (configurable as "ISR stack size" in menuconfig). This
/root/esp/esp-idf/components/freertos/Kconfig:426:        config FREERTOS_ISR_STACKSIZE
/root/esp/esp-idf/components/freertos/Kconfig:427:            int "ISR stack size"
(จบ Y1: ไม่มีบรรทัด = ไม่พบข้อความนี้ในที่ค้น)
### Y2 'slow clock' ใน esp_sleep.h และ gpio.h
/root/esp/esp-idf/components/esp_hw_support/include/esp_sleep.h-207- * @param time_in_us  time before wakeup, in microseconds
/root/esp/esp-idf/components/esp_hw_support/include/esp_sleep.h-208- * @note  The valid `time_in_us` value depends on the bit width of the lp_timer/rtc_timer counter and the
/root/esp/esp-idf/components/esp_hw_support/include/esp_sleep.h:209: *        current slow clock source selection (Refer RTC clock source configuration in menuconfig).
/root/esp/esp-idf/components/esp_hw_support/include/esp_sleep.h:210: *        Valid values should be positive values less than RTC slow clock period * (2 ^ RTC timer bitwidth).
/root/esp/esp-idf/components/esp_hw_support/include/esp_sleep.h-211- *
/root/esp/esp-idf/components/esp_hw_support/include/esp_sleep.h-212- * @return
(จบ Y2: ไม่มีบรรทัด = ไม่พบข้อความนี้)
### Y3 ชื่อ ROM เก่า ใน linker script ของ S3
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.api.ld:11:PROVIDE ( esp_rom_gpio_pad_select_gpio    = rom_gpio_pad_select_gpio );
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.api.ld:12:PROVIDE ( esp_rom_gpio_pad_pullup_only    = rom_gpio_pad_pullup );
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.api.ld:13:PROVIDE ( esp_rom_gpio_pad_set_drv        = rom_gpio_pad_set_drv );
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.api.ld:14:PROVIDE ( esp_rom_gpio_pad_unhold         = rom_gpio_pad_unhold );
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.api.ld:15:PROVIDE ( esp_rom_gpio_connect_in_signal  = rom_gpio_matrix_in );
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.api.ld:16:PROVIDE ( esp_rom_gpio_connect_out_signal = rom_gpio_matrix_out );
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.ld:458:rom_gpio_matrix_in = 0x40001a94;
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.ld:459:rom_gpio_matrix_out = 0x40001aa0;
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.ld:467:rom_gpio_pad_pullup = 0x40001b00;
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.ld:468:rom_gpio_pad_select_gpio = 0x40001b0c;
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.ld:469:rom_gpio_pad_set_drv = 0x40001b18;
/root/esp/esp-idf/components/esp_rom/esp32s3/ld/esp32s3.rom.ld:470:rom_gpio_pad_unhold = 0x40001b24;
(จบ Y3: ทุกบรรทัดควรเป็นรูป PROVIDE ( esp_rom_gpio_xxx = rom_gpio_xxx ) ใช้ดูการจับคู่ชื่อ)
### Y4 esp_rom_gpio.h ฉบับเต็ม
/*
 * SPDX-FileCopyrightText: 2010-2023 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#pragma once

#ifdef __cplusplus
extern "C" {
#endif

#include <stdint.h>
#include <stdbool.h>
#include "soc/gpio_pins.h"  //for GPIO_MATRIX_CONST_ONE_INPUT, GPIO_MATRIX_CONST_ZERO_INPUT

/**
 * @brief Configure IO Pad as General Purpose IO,
 *        so that it can be connected to internal Matrix,
 *        then combined with one or more peripheral signals.
 *
 * @param iopad_num IO Pad number
 */
void esp_rom_gpio_pad_select_gpio(uint32_t iopad_num);

/**
 * @brief Enable internal pull up, and disable internal pull down.
 *
 * @param iopad_num IO Pad number
 */
void esp_rom_gpio_pad_pullup_only(uint32_t iopad_num);

/**
 * @brief Unhold the IO Pad.
 * @note When the Pad is set to hold, the state is latched at that moment and won't get changed.
 *
 * @param iopad_num IP Pad number
 */
void esp_rom_gpio_pad_unhold(uint32_t gpio_num);

/**
 * @brief Set IO Pad current drive capability.
 *
 * @param iopad_num IO Pad number
 * @param drv Numeric to indicate the capability of current drive
 *      - 0: 5mA
 *      - 1: 10mA
 *      - 2: 20mA
 *      - 3: 40mA
 */
void esp_rom_gpio_pad_set_drv(uint32_t iopad_num, uint32_t drv);

/**
 * @brief Combine a GPIO input with a peripheral signal, which tagged as input attribute.
 *
 * @note There's no limitation on the number of signals that a GPIO can combine with.
 *
 * @param gpio_num GPIO number, especially, `GPIO_MATRIX_CONST_ZERO_INPUT` means connect logic 0 to signal
 *                                          `GPIO_MATRIX_CONST_ONE_INPUT` means connect logic 1 to signal
 * @param signal_idx Peripheral signal index (tagged as input attribute)
 * @param inv  Whether the GPIO input to be inverted or not
 */
void esp_rom_gpio_connect_in_signal(uint32_t gpio_num, uint32_t signal_idx, bool inv);

/**
 * @brief Combine a peripheral signal which tagged as output attribute with a GPIO.
 *
 * @note There's no limitation on the number of signals that a GPIO can combine with.
 * @note Internally, the signal will be connected first, then output will be enabled on the pad.
 *
 * @param gpio_num GPIO number
 * @param signal_idx Peripheral signal index (tagged as output attribute). Particularly, `SIG_GPIO_OUT_IDX` means disconnect GPIO and other peripherals. Only the GPIO driver can control the output level.
 * @param out_inv Whether to signal to be inverted or not
 * @param oen_inv Whether the output enable control is inverted or not
 */
void esp_rom_gpio_connect_out_signal(uint32_t gpio_num, uint32_t signal_idx, bool out_inv, bool oen_inv);

#ifdef __cplusplus
}
#endif
(จบ Y4)
### Y5 gpio_deep_sleep_hold_en ใน gpio.c
void gpio_deep_sleep_hold_en(void)
{
    portENTER_CRITICAL(&gpio_context.gpio_spinlock);
    RTC_CNTL_ATOMIC() {
        gpio_hal_deep_sleep_hold_en(gpio_context.gpio_hal);
    }
    portEXIT_CRITICAL(&gpio_context.gpio_spinlock);
}
(จบ Y5)
### Y6 rtc_gpio_hold_en ใน rtc_io.c
esp_err_t rtc_gpio_hold_en(gpio_num_t gpio_num)
{
    ESP_RETURN_ON_FALSE(rtc_gpio_is_valid_gpio(gpio_num), ESP_ERR_INVALID_ARG, RTCIO_TAG, "RTCIO number error");
    RTCIO_ENTER_CRITICAL();
    rtcio_hal_hold_enable(rtc_io_number_get(gpio_num));
    RTCIO_EXIT_CRITICAL();
    return ESP_OK;
}
(จบ Y6)
### Y7 rtcio_ll_hold_en ของ S3
(จบ Y7)
### Y8 จุดที่ esp_hw_support เรียก hold ของ GPIO/RTC IO ตอนเข้า sleep
/root/esp/esp-idf/components/esp_hw_support/debug_probe/debug_probe.c:57:        ESP_RETURN_ON_ERROR(rtc_gpio_init(pin), TAG, "init RTC GPIO %d failed", pin);
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:286:    if (!gpio_hal_deep_sleep_hold_is_en(&gpio_hal)) {
(จบ Y8)
```