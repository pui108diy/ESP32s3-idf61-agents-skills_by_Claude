## verify_gpio_unv2.sh
``` bash
#!/usr/bin/env bash
# verify_gpio_unv2.sh (รอบ 5): ปิดข้อ [UNV] ที่เหลือของ gpio.md โดยไม่ตัดผล
# วิธีใช้:
#   . $IDF_PATH/export.sh
#   bash verify_gpio_unv2.sh > verify2.log 2>&1
set -u
: "${IDF_PATH:?ต้อง source export.sh ก่อน}"

GPIO_C="$IDF_PATH/components/esp_driver_gpio/src/gpio.c"
GPIO_H="$IDF_PATH/components/esp_driver_gpio/include/driver/gpio.h"
ROM_GPIO_H="$IDF_PATH/components/esp_rom/include/esp_rom_gpio.h"
SLEEP_H="$IDF_PATH/components/esp_hw_support/include/esp_sleep.h"
HAL_S3="$IDF_PATH/components/esp_hal_gpio/esp32s3"

echo "### W0 รุ่นของ IDF"
(cd "$IDF_PATH" && git describe --tags 2>&1)

echo "### W1 gpio_pad_select_gpio แบบชื่อเปล่า (-w = ตรงทั้งคำ ไม่นับ esp_rom_gpio_pad_select_gpio)"
grep -rnw "gpio_pad_select_gpio" "$IDF_PATH/components" "$IDF_PATH/docs" 2>/dev/null
echo "(จบ W1: ไม่มีบรรทัดผลลัพธ์ = ไม่มีที่ใดประกาศหรือเรียกชื่อเปล่านี้ใน components และ docs)"

echo "### W2 ชื่อ pad_select ใน header ที่แอปใช้"
grep -n "pad_select_gpio" "$ROM_GPIO_H" "$GPIO_H" 2>&1
echo "(จบ W2)"

echo "### W3 ตัวฟังก์ชัน gpio_wakeup_disable แบบเต็ม"
sed -n '/^esp_err_t gpio_wakeup_disable(/,/^}/p' "$GPIO_C"
echo "--- ตำแหน่งที่เรียก gpio_hal_set_intr_type ใน gpio.c"
grep -n "gpio_hal_set_intr_type" "$GPIO_C"
echo "(จบ W3)"

echo "### W4 GPIO_HOLD_MASK ของ S3"
grep -rn -B1 -A8 "GPIO_HOLD_MASK" "$HAL_S3" 2>&1
echo "--- ใน gpio.c"
grep -n "GPIO_HOLD_MASK" "$GPIO_C"
echo "(จบ W4)"

echo "### W5 esp_sleep.h: บรรทัดที่เป็น deprecated"
grep -n "__attribute__((deprecated" "$SLEEP_H"
echo "--- esp_sleep_get_ext1_wakeup_status พร้อมบรรทัดรอบข้าง"
grep -n -B3 -A2 "esp_sleep_get_ext1_wakeup_status" "$SLEEP_H"
echo "--- esp_sleep_get_wakeup_cause พร้อมบรรทัดรอบข้าง"
grep -n -B2 -A2 "esp_sleep_get_wakeup_cause(" "$SLEEP_H"
echo "(จบ W5)"
```

## verify2.log ผลที่ได้
``` code
### W0 รุ่นของ IDF
v6.1
### W1 gpio_pad_select_gpio แบบชื่อเปล่า (-w = ตรงทั้งคำ ไม่นับ esp_rom_gpio_pad_select_gpio)
/root/esp/esp-idf/components/esp_rom/hints.yml:2:    re: "(error: implicit declaration of function|undefined reference to) '(gpio_pad_set_drv|gpio_pad_pullup|gpio_pad_select_gpio|gpio_pad_unhold|gpio_matrix_in|gpio_matrix_out)'"
(จบ W1: ไม่มีบรรทัดผลลัพธ์ = ไม่มีที่ใดประกาศหรือเรียกชื่อเปล่านี้ใน components และ docs)
### W2 ชื่อ pad_select ใน header ที่แอปใช้
/root/esp/esp-idf/components/esp_rom/include/esp_rom_gpio.h:24:void esp_rom_gpio_pad_select_gpio(uint32_t iopad_num);
(จบ W2)
### W3 ตัวฟังก์ชัน gpio_wakeup_disable แบบเต็ม
esp_err_t gpio_wakeup_disable(gpio_num_t gpio_num)
{
    GPIO_CHECK(GPIO_IS_VALID_GPIO(gpio_num), "GPIO number error", ESP_ERR_INVALID_ARG);
    esp_err_t ret = ESP_OK;
#if SOC_RTCIO_WAKE_SUPPORTED
    if (rtc_gpio_is_valid_gpio(gpio_num)) {
        ret = rtc_gpio_wakeup_disable(gpio_num);
#if SOC_LP_IO_CLOCK_IS_INDEPENDENT
        io_mux_enable_lp_io_clock(gpio_num, false);
#endif
    }
#endif
    portENTER_CRITICAL(&gpio_context.gpio_spinlock);
    gpio_hal_wakeup_disable(gpio_context.gpio_hal, gpio_num);
#if CONFIG_ESP_SLEEP_GPIO_RESET_WORKAROUND || CONFIG_PM_SLP_DISABLE_GPIO
    gpio_hal_sleep_sel_en(gpio_context.gpio_hal, gpio_num);
#endif
    portEXIT_CRITICAL(&gpio_context.gpio_spinlock);
    return ret;
}
--- ตำแหน่งที่เรียก gpio_hal_set_intr_type ใน gpio.c
169:    gpio_hal_set_intr_type(gpio_context.gpio_hal, gpio_num, intr_type);
701:        gpio_hal_set_intr_type(gpio_context.gpio_hal, gpio_num, intr_type);
(จบ W3)
### W4 GPIO_HOLD_MASK ของ S3
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-472-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:473:    SET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-474-}
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-475-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-476-/**
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-477-  * @brief Disable gpio pad hold function.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-478-  *
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-479-  * @param hw Peripheral GPIO hardware instance address.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-480-  * @param gpio_num GPIO number, only support output GPIOs
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-481-  */
--
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-483-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:484:    CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-485-}
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-486-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-487-/**
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-488-  * @brief Get gpio pad hold status.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-489-  *
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-490-  * @param hw Peripheral GPIO hardware instance address.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-491-  * @param gpio_num GPIO number, only support output GPIOs
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-492-  *
--- ใน gpio.c
786:    } else if (GPIO_HOLD_MASK[gpio_num]) {
806:    } else if (GPIO_HOLD_MASK[gpio_num]) {
(จบ W4)
### W5 esp_sleep.h: บรรทัดที่เป็น deprecated
38:    ESP_EXT1_WAKEUP_ALL_LOW __attribute__((deprecated("wakeup mode \"ALL_LOW\" is no longer supported after ESP32, \
466:__attribute__((deprecated("please use 'esp_sleep_enable_ext1_wakeup_io' and 'esp_sleep_disable_ext1_wakeup_io' instead")));
778:__attribute__((deprecated("use esp_sleep_get_wakeup_causes instead")));
--- esp_sleep_get_ext1_wakeup_status พร้อมบรรทัดรอบข้าง
616- *
617- * @return bit mask, if GPIOn caused wakeup, BIT(n) will be set
618- */
619:uint64_t esp_sleep_get_ext1_wakeup_status(void);
620-
621-#if SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP
--- esp_sleep_get_wakeup_cause พร้อมบรรทัดรอบข้าง
775- * @return cause of wake up from last sleep (deep sleep or light sleep)
776- */
777:esp_sleep_wakeup_cause_t esp_sleep_get_wakeup_cause(void)
778-__attribute__((deprecated("use esp_sleep_get_wakeup_causes instead")));
779-
(จบ W5)
```