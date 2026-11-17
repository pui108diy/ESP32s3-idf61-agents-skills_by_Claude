## verify_gpio_unv.sh
``` bash
#!/usr/bin/env bash
# verify_gpio_unv.sh: ปิดข้อ [UNV] ของ gpio.md โดยไม่ต้องใช้บอร์ด
# วิธีใช้:
#   . $IDF_PATH/export.sh
#   bash verify_gpio_unv.sh > verify.log 2>&1
# แล้วส่งเนื้อหา verify.log กลับมา
set -u
PROJ="${1:-.}"
: "${IDF_PATH:?ต้อง source export.sh ก่อน}"

GPIO_C="$IDF_PATH/components/esp_driver_gpio/src/gpio.c"
FILTER_C="$IDF_PATH/components/esp_driver_gpio/src/gpio_pin_glitch_filter.c"
SOC_CAPS="$IDF_PATH/components/soc/esp32s3/include/soc/soc_caps.h"
SLEEP_H="$IDF_PATH/components/esp_hw_support/include/esp_sleep.h"
ROM_GPIO_H="$IDF_PATH/components/esp_rom/include/esp_rom_gpio.h"

echo "### V0 รุ่นของ IDF"
(cd "$IDF_PATH" && git describe --tags 2>&1 | head -1)

echo "### V1 gpio_pad_select_gpio (ปิด GPIO-02) ใน components และ docs"
grep -rn "gpio_pad_select_gpio" "$IDF_PATH/components" "$IDF_PATH/docs" 2>/dev/null | head -20
echo "(จบ V1: ถ้าไม่มีบรรทัดใดเลย = ไม่พบ symbol นี้ในต้นไม้ v6.1)"

echo "### V2 soc_caps.h ของ S3 (ไม่มีบรรทัดผลลัพธ์ = macro ไม่ถูกนิยามในไฟล์)"
grep -nE "SOC_GPIO_SUPPORT_ETM|SOC_DEDICATED_GPIO_SUPPORTED|SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN|SOC_GPIO_FLEX_GLITCH_FILTER_NUM|SOC_GPIO_SUPPORT_PIN_HYS|SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP|SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED|SOC_GPIO_SUPPORT_FORCE_HOLD|SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER" "$SOC_CAPS"
echo "(จบ V2)"

echo "### V3 sdkconfig ของโปรเจกต์ (ระวัง: ค่า MASK ใน Kconfig ไม่ตรง soc_caps.h ใช้ดูเฉพาะ bool/int)"
grep -nE "CONFIG_SOC_(GPIO|RTCIO|DEDICATED_GPIO|PM_SUPPORT_EXT)" "$PROJ/sdkconfig"
echo "(จบ V3)"

echo "### V4 gpio_wakeup_disable ฝั่ง digital"
grep -n -A16 "^esp_err_t gpio_wakeup_disable(" "$GPIO_C"
echo "(จบ V4)"

echo "### V5 esp_rom_gpio.h"
grep -n "gpio_pad_select_gpio\|esp_rom_gpio_" "$ROM_GPIO_H" 2>&1 | head -20
echo "(จบ V5)"

echo "### V6 deprecated ใน esp_sleep.h"
grep -n "deprecated\|esp_sleep_get_ext1_wakeup_status" "$SLEEP_H" | head -30
echo "(จบ V6)"

echo "### V7 gpio_new_pin_glitch_filter"
grep -n -A45 "^esp_err_t gpio_new_pin_glitch_filter" "$FILTER_C" | head -70
echo "(จบ V7)"

echo "### V8 GPIO_HOLD_MASK (ขา digital ที่ gpio_hold_en ใช้ได้บน S3)"
grep -rn "GPIO_HOLD_MASK" "$IDF_PATH/components/esp_driver_gpio" "$IDF_PATH/components/esp_hal_gpio" 2>/dev/null | head -20
echo "(จบ V8)"
```
## verify.log ผลที่ได้
``` code
### V0 รุ่นของ IDF
v6.1
### V1 gpio_pad_select_gpio (ปิด GPIO-02) ใน components และ docs
/root/esp/esp-idf/components/bootloader_support/src/bootloader_common.c:39:    esp_rom_gpio_pad_select_gpio(num_pin);
/root/esp/esp-idf/components/esp_driver_sdmmc/src/sd_host_sdmmc.c:1412:        esp_rom_gpio_pad_select_gpio(io_config->cd_io);
/root/esp/esp-idf/components/esp_driver_sdmmc/src/sd_host_sdmmc.c:1425:        esp_rom_gpio_pad_select_gpio(io_config->wp_io);
/root/esp/esp-idf/components/esp_hw_support/port/esp32/rtc_clk.c:135:        esp_rom_gpio_pad_select_gpio(XTAL32K_P_GPIO_NUM);
/root/esp/esp-idf/components/esp_hw_support/port/esp32/rtc_clk.c:136:        esp_rom_gpio_pad_select_gpio(XTAL32K_N_GPIO_NUM);
/root/esp/esp-idf/components/esp_hw_support/test_apps/rtc_clk/main/test_rtc_clk.c:198:    esp_rom_gpio_pad_select_gpio(pin_xtal32k_p);
/root/esp/esp-idf/components/esp_hw_support/test_apps/rtc_clk/main/test_rtc_clk.c:199:    esp_rom_gpio_pad_select_gpio(pin_xtal32k_n);
/root/esp/esp-idf/components/esp_rom/esp32/include/esp32/rom/gpio.h:136:void rom_gpio_pad_select_gpio(uint8_t gpio_num);
/root/esp/esp-idf/components/esp_rom/esp32/ld/esp32.rom.api.ld:11:PROVIDE ( esp_rom_gpio_pad_select_gpio    = rom_gpio_pad_select_gpio );
/root/esp/esp-idf/components/esp_rom/esp32/ld/esp32.rom.ld:1510:PROVIDE ( rom_gpio_pad_select_gpio = 0x40009fdc );
/root/esp/esp-idf/components/esp_rom/esp32c2/include/esp32c2/rom/gpio.h:135:void rom_gpio_pad_select_gpio(uint32_t gpio_num);
/root/esp/esp-idf/components/esp_rom/esp32c2/ld/esp32c2.rom.api.ld:15:PROVIDE ( esp_rom_gpio_pad_select_gpio    = rom_gpio_pad_select_gpio );
/root/esp/esp-idf/components/esp_rom/esp32c2/ld/esp32c2.rom.ld:372:rom_gpio_pad_select_gpio = 0x400007b4;
/root/esp/esp-idf/components/esp_rom/esp32c3/include/esp32c3/rom/gpio.h:135:void rom_gpio_pad_select_gpio(uint32_t gpio_num);
/root/esp/esp-idf/components/esp_rom/esp32c3/ld/esp32c3.rom.api.ld:11:PROVIDE ( esp_rom_gpio_pad_select_gpio    = rom_gpio_pad_select_gpio );
/root/esp/esp-idf/components/esp_rom/esp32c3/ld/esp32c3.rom.ld:376:rom_gpio_pad_select_gpio = 0x400005c8;
/root/esp/esp-idf/components/esp_rom/esp32c5/include/esp32c5/rom/gpio.h:92:void rom_gpio_pad_select_gpio(uint32_t gpio_num);
/root/esp/esp-idf/components/esp_rom/esp32c5/ld/esp32c5.rom.api.ld:16:PROVIDE ( esp_rom_gpio_pad_select_gpio    = rom_gpio_pad_select_gpio );
/root/esp/esp-idf/components/esp_rom/esp32c5/ld/esp32c5.rom.ld:267:rom_gpio_pad_select_gpio = 0x40000734;
/root/esp/esp-idf/components/esp_rom/esp32c6/include/esp32c6/rom/gpio.h:135:void rom_gpio_pad_select_gpio(uint32_t gpio_num);
(จบ V1: ถ้าไม่มีบรรทัดใดเลย = ไม่พบ symbol นี้ในต้นไม้ v6.1)
### V2 soc_caps.h ของ S3 (ไม่มีบรรทัดผลลัพธ์ = macro ไม่ถูกนิยามในไฟล์)
47:#define SOC_DEDICATED_GPIO_SUPPORTED    1
187:#define SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER 1
190:#define SOC_GPIO_SUPPORT_FORCE_HOLD      (1)
(จบ V2)
### V3 sdkconfig ของโปรเจกต์ (ระวัง: ค่า MASK ใน Kconfig ไม่ตรง soc_caps.h ใช้ดูเฉพาะ bool/int)
38:CONFIG_SOC_DEDICATED_GPIO_SUPPORTED=y
232:CONFIG_SOC_GPIO_PORT=1
234:CONFIG_SOC_GPIO_PIN_COUNT=49
236:CONFIG_SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER=y
238:CONFIG_SOC_GPIO_SUPPORT_FORCE_HOLD=y
240:CONFIG_SOC_GPIO_VALID_GPIO_MASK=0x1FFFFFFFFFFFF
242:CONFIG_SOC_GPIO_IN_RANGE_MAX=48
244:CONFIG_SOC_GPIO_OUT_RANGE_MAX=48
246:CONFIG_SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK=0x0001FFFFFC000000
248:CONFIG_SOC_GPIO_CLOCKOUT_BY_IO_MUX=y
250:CONFIG_SOC_GPIO_CLOCKOUT_CHANNEL_NUM=3
252:CONFIG_SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP=y
254:CONFIG_SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK=0
332:CONFIG_SOC_RTCIO_PIN_COUNT=22
334:CONFIG_SOC_RTCIO_INPUT_OUTPUT_SUPPORTED=y
336:CONFIG_SOC_RTCIO_HOLD_SUPPORTED=y
338:CONFIG_SOC_RTCIO_WAKE_SUPPORTED=y
444:CONFIG_SOC_PM_SUPPORT_EXT0_WAKEUP=y
446:CONFIG_SOC_PM_SUPPORT_EXT1_WAKEUP=y
448:CONFIG_SOC_PM_SUPPORT_EXT_WAKEUP=y
(จบ V3)
### V4 gpio_wakeup_disable ฝั่ง digital
715:esp_err_t gpio_wakeup_disable(gpio_num_t gpio_num)
716-{
717-    GPIO_CHECK(GPIO_IS_VALID_GPIO(gpio_num), "GPIO number error", ESP_ERR_INVALID_ARG);
718-    esp_err_t ret = ESP_OK;
719-#if SOC_RTCIO_WAKE_SUPPORTED
720-    if (rtc_gpio_is_valid_gpio(gpio_num)) {
721-        ret = rtc_gpio_wakeup_disable(gpio_num);
722-#if SOC_LP_IO_CLOCK_IS_INDEPENDENT
723-        io_mux_enable_lp_io_clock(gpio_num, false);
724-#endif
725-    }
726-#endif
727-    portENTER_CRITICAL(&gpio_context.gpio_spinlock);
728-    gpio_hal_wakeup_disable(gpio_context.gpio_hal, gpio_num);
729-#if CONFIG_ESP_SLEEP_GPIO_RESET_WORKAROUND || CONFIG_PM_SLP_DISABLE_GPIO
730-    gpio_hal_sleep_sel_en(gpio_context.gpio_hal, gpio_num);
731-#endif
(จบ V4)
### V5 esp_rom_gpio.h
24:void esp_rom_gpio_pad_select_gpio(uint32_t iopad_num);
31:void esp_rom_gpio_pad_pullup_only(uint32_t iopad_num);
39:void esp_rom_gpio_pad_unhold(uint32_t gpio_num);
51:void esp_rom_gpio_pad_set_drv(uint32_t iopad_num, uint32_t drv);
63:void esp_rom_gpio_connect_in_signal(uint32_t gpio_num, uint32_t signal_idx, bool inv);
76:void esp_rom_gpio_connect_out_signal(uint32_t gpio_num, uint32_t signal_idx, bool out_inv, bool oen_inv);
(จบ V5)
### V6 deprecated ใน esp_sleep.h
38:    ESP_EXT1_WAKEUP_ALL_LOW __attribute__((deprecated("wakeup mode \"ALL_LOW\" is no longer supported after ESP32, \
99:#define ESP_PD_DOMAIN_RTC8M _Pragma("GCC warning \"'ESP_PD_DOMAIN_RTC8M' enum is deprecated\"") ESP_PD_DOMAIN_RC_FAST
333: * @note This function will be deprecated in release/v6.0. Please switch to use `esp_sleep_enable_ext1_wakeup_io` and `esp_sleep_disable_ext1_wakeup_io`
466:__attribute__((deprecated("please use 'esp_sleep_enable_ext1_wakeup_io' and 'esp_sleep_disable_ext1_wakeup_io' instead")));
619:uint64_t esp_sleep_get_ext1_wakeup_status(void);
778:__attribute__((deprecated("use esp_sleep_get_wakeup_causes instead")));
(จบ V6)
### V7 gpio_new_pin_glitch_filter
93:esp_err_t gpio_new_pin_glitch_filter(const gpio_pin_glitch_filter_config_t *config, gpio_glitch_filter_handle_t *ret_filter)
94-{
95-    esp_err_t ret = ESP_OK;
96-    gpio_pin_glitch_filter_t *filter = NULL;
97-    ESP_GOTO_ON_FALSE(config && ret_filter, ESP_ERR_INVALID_ARG, err, TAG, "invalid argument");
98-    ESP_GOTO_ON_FALSE(GPIO_IS_VALID_GPIO(config->gpio_num), ESP_ERR_INVALID_ARG, err, TAG, "invalid gpio number");
99-
100-    filter = heap_caps_calloc(1, sizeof(gpio_pin_glitch_filter_t), FILTER_MEM_ALLOC_CAPS);
101-    ESP_GOTO_ON_FALSE(filter, ESP_ERR_NO_MEM, err, TAG, "no memory for pin glitch filter");
102-    filter->clk_src = SOC_MOD_CLK_INVALID;
103-    filter->io_mux_clk_acquired = false;
104-
105-    soc_module_clk_t clk_src = config->clk_src ? config->clk_src : GLITCH_FILTER_CLK_SRC_DEFAULT;
106-    ESP_GOTO_ON_ERROR(esp_clk_tree_enable_src(clk_src, true), err, TAG, "enable IOMUX clock source failed");
107-    filter->clk_src = clk_src;
108-    ESP_GOTO_ON_ERROR(io_mux_acquire_clock_source(clk_src), err, TAG, "acquire IOMUX clock source failed");
109-    filter->io_mux_clk_acquired = true;
110-
111-    // create pm lock according to different clock source
112-#if CONFIG_PM_ENABLE
113-    esp_pm_lock_type_t lock_type = ESP_PM_NO_LIGHT_SLEEP;
114-#if GPIO_CAPS_GET(FILTER_CLK_SUPPORT_APB)
115-    if (clk_src == (soc_module_clk_t)GLITCH_FILTER_CLK_SRC_APB) {
116-        lock_type = ESP_PM_APB_FREQ_MAX;
117-    }
118-#endif // GPIO_CAPS_GET(FILTER_CLK_SUPPORT_APB)
119-    sprintf(filter->pm_lock_name, "filter_io_%d", config->gpio_num); // e.g. filter_io_0
120-    ESP_GOTO_ON_ERROR(esp_pm_lock_create(lock_type, 0, filter->pm_lock_name, &filter->pm_lock),
121-                      err, TAG, "create pm_lock failed");
122-#endif // CONFIG_PM_ENABLE
123-
124-    filter->base.gpio_num = config->gpio_num;
125-    filter->base.fsm = GLITCH_FILTER_FSM_INIT;
126-    filter->base.del = gpio_pin_glitch_filter_del;
127-    filter->base.enable = gpio_pin_glitch_filter_enable;
128-    filter->base.disable = gpio_pin_glitch_filter_disable;
129-
130-    *ret_filter = &(filter->base);
131-    return ESP_OK;
132-err:
133-    if (filter) {
134-        gpio_filter_destroy(filter);
135-    }
136-    return ret;
137-}
(จบ V7)
### V8 GPIO_HOLD_MASK (ขา digital ที่ gpio_hold_en ใช้ได้บน S3)
/root/esp/esp-idf/components/esp_driver_gpio/src/gpio.c:786:    } else if (GPIO_HOLD_MASK[gpio_num]) {
/root/esp/esp-idf/components/esp_driver_gpio/src/gpio.c:806:    } else if (GPIO_HOLD_MASK[gpio_num]) {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:473:    SET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:484:    CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32/include/hal/gpio_ll.h:643:    SET_PERI_REG_MASK(RTC_IO_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32/include/hal/gpio_ll.h:654:    CLEAR_PERI_REG_MASK(RTC_IO_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32c2/include/hal/gpio_ll.h:456:        SET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32c2/include/hal/gpio_ll.h:471:        CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32c3/include/hal/gpio_ll.h:453:        SET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32c3/include/hal/gpio_ll.h:468:        CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32c5/include/hal/gpio_ll.h:446:    LP_AON.gpio_hold0.gpio_hold0 |= GPIO_HOLD_MASK[gpio_num];
/root/esp/esp-idf/components/esp_hal_gpio/esp32c5/include/hal/gpio_ll.h:458:    LP_AON.gpio_hold0.gpio_hold0 &= ~GPIO_HOLD_MASK[gpio_num];
/root/esp/esp-idf/components/esp_hal_gpio/esp32c6/include/hal/gpio_ll.h:410:    LP_AON.gpio_hold0.gpio_hold0 |= GPIO_HOLD_MASK[gpio_num];
/root/esp/esp-idf/components/esp_hal_gpio/esp32c6/include/hal/gpio_ll.h:422:    LP_AON.gpio_hold0.gpio_hold0 &= ~GPIO_HOLD_MASK[gpio_num];
/root/esp/esp-idf/components/esp_hal_gpio/esp32c61/include/hal/gpio_ll.h:445:    LP_AON.gpio_hold0.gpio_hold0 = mask | GPIO_HOLD_MASK[gpio_num];
/root/esp/esp-idf/components/esp_hal_gpio/esp32c61/include/hal/gpio_ll.h:458:    LP_AON.gpio_hold0.gpio_hold0 = mask & (~GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32h2/include/hal/gpio_ll.h:456:    LP_AON.gpio_hold0.gpio_hold0 |= GPIO_HOLD_MASK[gpio_num];
/root/esp/esp-idf/components/esp_hal_gpio/esp32h2/include/hal/gpio_ll.h:468:    LP_AON.gpio_hold0.gpio_hold0 &= ~GPIO_HOLD_MASK[gpio_num];
/root/esp/esp-idf/components/esp_hal_gpio/esp32h21/include/hal/gpio_ll.h:437:    LP_AON.gpio_hold0.gpio_hold0 |= GPIO_HOLD_MASK[gpio_num];
/root/esp/esp-idf/components/esp_hal_gpio/esp32h21/include/hal/gpio_ll.h:449:    LP_AON.gpio_hold0.gpio_hold0 &= ~GPIO_HOLD_MASK[gpio_num];
(จบ V8)
```


