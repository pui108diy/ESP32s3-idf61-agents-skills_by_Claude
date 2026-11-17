
## verify_gpio_unv6.sh

``` bash
#!/usr/bin/env bash
# verify_gpio_unv6.sh (รอบ 5g): ปิดข้อที่ verify_gpio_unv5.sh ตอบไม่ครบ (อ่านอย่างเดียว)
# Z7 พิสูจน์: ไฟล์ soc_caps.h ของ S3 อยู่ที่ path นี้จริง และมี SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD หรือไม่ (ใช้ตัดสินว่าลูป isolate ใน sleep_gpio.c ทำงานบน S3 หรือไม่)
# Z8 พิสูจน์: โค้ด sleep_gpio.c ช่วง 190-215 และ 305-420 ฉบับเต็ม (Z1 ตัดไว้แค่บรรทัด 260-320)
# Z9 พิสูจน์: ชื่อฟังก์ชัน hold ใน rtc_io_ll.h ของ S3 (Z3 ว่างเพราะเดาชื่อผิด จึงค้นคำว่า hold แบบกว้าง)
# Z10 พิสูจน์: ใครเรียก esp_sleep_isolate_digital_gpio (ใช้ยืนยันว่า assert เกี่ยวกับ esp_deep_sleep_start)
# Z11 พิสูจน์: เอกสาร sleep_modes.rst และ gpio.rst พูดเรื่อง hold ของ RTC GPIO และ deep sleep อย่างไร
# Z12 พิสูจน์: ข้อความรอบบรรทัด 392 ของ sleep_modes.rst ครบทั้งย่อหน้า
# วิธีใช้
#   . $IDF_PATH/export.sh
#   bash verify_gpio_unv6.sh > verify6.log 2>&1
set -u
: "${IDF_PATH:?ต้อง source export.sh ก่อน}"
C="$IDF_PATH/components"
D="$IDF_PATH/docs/en/api-reference"

echo "### Z7 soc_caps.h ของ S3"
ls -l "$C/soc/esp32s3/include/soc/soc_caps.h"
grep -rn "SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD" "$C/soc/esp32s3" "$C/esp_hal_gpio/esp32s3"
echo "(จบ Z7)"

echo "### Z8 sleep_gpio.c"
sed -n '190,215p' "$C/esp_hw_support/sleep_gpio.c"
echo "--- บรรทัด 305-420"
sed -n '305,420p' "$C/esp_hw_support/sleep_gpio.c"
echo "(จบ Z8)"

echo "### Z9 rtc_io_ll.h ของ S3 บรรทัดที่มีคำว่า hold"
grep -n -i "hold" "$C/esp_hal_gpio/esp32s3/include/hal/rtc_io_ll.h"
echo "--- rtcio_hal_hold_enable ใน esp_hal_gpio ทั้งหมด"
grep -rn -A6 "rtcio_hal_hold_enable" "$C/esp_hal_gpio" --include=*.h --include=*.c
echo "(จบ Z9)"

echo "### Z10 ผู้เรียก esp_sleep_isolate_digital_gpio"
grep -rn "esp_sleep_isolate_digital_gpio" "$C/esp_hw_support"
echo "(จบ Z10)"

echo "### Z11 คำว่า hold ใน sleep_modes.rst และ gpio.rst"
grep -n -i "hold" "$D/system/sleep_modes.rst" "$D/peripherals/gpio.rst"
echo "(จบ Z11)"

echo "### Z12 sleep_modes.rst บรรทัด 380-400"
sed -n '380,400p' "$D/system/sleep_modes.rst"
echo "(จบ Z12)"
```

## verify6.log 

``` code
### Z7 soc_caps.h ของ S3
-rw-r--r--. 1 root root 25529 Oct 12 11:51 /root/esp/esp-idf/components/soc/esp32s3/include/soc/soc_caps.h
(จบ Z7)
### Z8 sleep_gpio.c
        } else {
            gpio_sleep_sel_dis(gpio_num);
        }
    }
}
#endif

#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
IRAM_ATTR static void sleep_gpio_isolate_one_pin(gpio_num_t gpio_num)
{
#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
    gpio_io_config_t isolate_config = { .pu = 0, .pd = 0, .ie = 0, .oe = 0, .fun_sel = PIN_FUNC_GPIO };
    gpio_ll_set_pad_config_for_sleep_isolate(gpio_num, &isolate_config);
#else
    gpio_hal_context_t gpio_hal = { .dev = GPIO_HAL_GET_HW(GPIO_PORT_0) };
    gpio_hal_input_disable(&gpio_hal, gpio_num);
    gpio_hal_output_disable(&gpio_hal, gpio_num);
    gpio_hal_pullup_dis(&gpio_hal, gpio_num);
    gpio_hal_pulldown_dis(&gpio_hal, gpio_num);
    gpio_hal_func_sel(&gpio_hal, gpio_num, PIN_FUNC_GPIO);
#endif
}

#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
typedef struct {
    uint64_t backuped;
--- บรรทัด 305-420
        s_gpio_isolate_backup.backuped = 0;
        s_gpio_isolate_backup.pu = 0;
        s_gpio_isolate_backup.pd = 0;
        s_gpio_isolate_backup.ie = 0;
        s_gpio_isolate_backup.oe = 0;
    }

    uint64_t hold_mask = gpio_ll_get_digital_gpio_hold_mask(gpio_hal.dev);
    gpio_io_config_t isolate_config = { .pu = 0, .pd = 0, .ie = 0, .oe = 0, .fun_sel = PIN_FUNC_GPIO };

    while (pad_mask) {
        gpio_num_t gpio_num = (gpio_num_t)__builtin_ctzll(pad_mask);
        if (!(hold_mask & (1ULL << gpio_num)) &&
            !esp_gpio_is_reserved(BIT64(gpio_num))) {
            if (do_backup) {
                gpio_io_config_t io_config;
                gpio_ll_backup_pad_config_for_sleep_isolate(gpio_num, &io_config);
                if (io_config.pu) {
                    s_gpio_isolate_backup.pu |= (1ULL << gpio_num);
                }
                if (io_config.pd) {
                    s_gpio_isolate_backup.pd |= (1ULL << gpio_num);
                }
                if (io_config.ie) {
                    s_gpio_isolate_backup.ie |= (1ULL << gpio_num);
                }
                if (gpio_ll_output_is_enabled(gpio_hal.dev, gpio_num)) {
                    s_gpio_isolate_backup.oe |= (1ULL << gpio_num);
                }
                s_gpio_isolate_backup.fun_sel[gpio_num] = (uint8_t)io_config.fun_sel;
                s_gpio_isolate_backup.backuped |= (1ULL << gpio_num);
            }
            gpio_ll_set_pad_config_for_sleep_isolate(gpio_num, &isolate_config);
        }
        pad_mask &= pad_mask - 1;
    }
#else
    (void)do_backup;
    while (pad_mask) {
        gpio_num_t gpio_num = (gpio_num_t)__builtin_ctzll(pad_mask);
        if (!gpio_hal_is_digital_io_hold(&gpio_hal, gpio_num) &&
            !esp_gpio_is_reserved(BIT64(gpio_num))) {
            sleep_gpio_isolate_one_pin(gpio_num);
        }
        pad_mask &= pad_mask - 1;
    }
#endif
}
#endif //!SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD

#if SOC_DEEP_SLEEP_SUPPORTED
static void esp_deep_sleep_wakeup_io_reset(void)
{
#if SOC_PM_SUPPORT_EXT1_WAKEUP
    uint32_t rtc_io_mask = rtc_hal_ext1_get_wakeup_pins();
    // Disable ext1 wakeup before releasing hold, such that wakeup status can reflect the correct wakeup pin
    rtc_hal_ext1_clear_wakeup_pins();
    for (int gpio_num = 0; gpio_num < SOC_GPIO_PIN_COUNT && rtc_io_mask != 0; ++gpio_num) {
        int rtcio_num = rtc_io_num_map[gpio_num];
        if ((rtc_io_mask & BIT(rtcio_num)) == 0) {
            continue;
        }
#if SOC_RTCIO_HOLD_SUPPORTED
        rtcio_hal_hold_disable(rtcio_num);
#endif
        rtc_io_mask &= ~BIT(rtcio_num);
    }
#endif

#if SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP
    gpio_hal_context_t gpio_hal = { .dev = GPIO_HAL_GET_HW(GPIO_PORT_0) };
    uint64_t dslp_io_mask = SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK;
    while (dslp_io_mask) {
        int gpio_num = __builtin_ctzll(dslp_io_mask);
        bool wakeup_io_enabled = gpio_hal_wakeup_is_enabled_on_hp_periph_powerdown_sleep(&gpio_hal, gpio_num);
        if (wakeup_io_enabled) {
            // Disable the wakeup before releasing hold, such that wakeup status can reflect the correct wakeup pin
            gpio_hal_wakeup_disable_on_hp_periph_powerdown_sleep(&gpio_hal, gpio_num);
            gpio_hal_hold_dis(&gpio_hal, gpio_num);
        }
        dslp_io_mask &= dslp_io_mask - 1;
    }
#endif
}
#endif

ESP_SYSTEM_INIT_FN(esp_sleep_startup_init, SECONDARY, BIT(0), 105)
{
#if SOC_DEEP_SLEEP_SUPPORTED
    // Need to unhold the IOs that were hold right before entering deep sleep, which are used as wakeup pins
#if SOC_PMU_SUPPORTED
    uint32_t io_wakeup_enabled = pmu_ll_hp_get_wakeup_enable(&PMU) & IO_WAKEUP_ENABLED;
#else
    uint32_t io_wakeup_enabled = rtc_cntl_ll_get_wakeup_enable() & IO_WAKEUP_ENABLED;
#endif
    if (esp_reset_reason() == ESP_RST_DEEPSLEEP && io_wakeup_enabled) {
        esp_deep_sleep_wakeup_io_reset();
    }
#endif  //#if SOC_DEEP_SLEEP_SUPPORTED

#if CONFIG_ESP_SLEEP_GPIO_RESET_WORKAROUND || CONFIG_PM_SLP_DISABLE_GPIO
    // Configure to isolate (disable the Input/Output/Pullup/Pulldown
    // function of the pin) all GPIO pins in sleep state
    esp_sleep_config_gpio_isolate();
    // Enable automatic switching of GPIO configuration
    esp_sleep_enable_gpio_switch(true);
#endif
    return ESP_OK;
}

/**
 * @brief Convert RTC IO status bit number to GPIO number (for sleep wakeup status translation).
 *
 * @param bit RTC IO intr status bit index (0 to SOC_RTCIO_PIN_COUNT-1).
 * @return GPIO number, or GPIO_NUM_NC if bit is invalid or has no corresponding GPIO.
 */
(จบ Z8)
### Z9 rtc_io_ll.h ของ S3 บรรทัดที่มีคำว่า hold
289: * Enable force hold function on an RTC IO pad.
291: * Enabling HOLD function will cause the pad to lock current status, such as,
298:static inline void rtcio_ll_force_hold_enable(int rtcio_num)
300:    SET_PERI_REG_MASK(RTC_CNTL_PAD_HOLD_REG, rtc_io_desc[rtcio_num].hold_force);
304: * Disable hold function on an RTC IO pad.
306: * @note If disable the pad hold, the status of pad maybe changed in sleep mode.
309:static inline void rtcio_ll_force_hold_disable(int rtcio_num)
311:    CLEAR_PERI_REG_MASK(RTC_CNTL_PAD_HOLD_REG, rtc_io_desc[rtcio_num].hold_force);
315: * Enable force hold function on all RTC IO pads.
317: * Enabling HOLD function will cause the pad to lock current status, such as,
324:static inline void rtcio_ll_force_hold_all(void)
326:    SET_PERI_REG_MASK(RTC_CNTL_PWC_REG, RTC_CNTL_PAD_FORCE_HOLD_M);
330: * Disable hold function on all RTC IO pads.
332: * @note If disable the pad hold, the status of pad maybe changed in sleep mode.
335:static inline void rtcio_ll_force_unhold_all(void)
337:    CLEAR_PERI_REG_MASK(RTC_CNTL_PWC_REG, RTC_CNTL_PAD_FORCE_HOLD_M);
--- rtcio_hal_hold_enable ใน esp_hal_gpio ทั้งหมด
/root/esp/esp-idf/components/esp_hal_gpio/include/hal/rtc_io_hal.h:239:#define rtcio_hal_hold_enable(rtcio_num) rtcio_ll_force_hold_enable(rtcio_num)
/root/esp/esp-idf/components/esp_hal_gpio/include/hal/rtc_io_hal.h-240-
/root/esp/esp-idf/components/esp_hal_gpio/include/hal/rtc_io_hal.h-241-/**
/root/esp/esp-idf/components/esp_hal_gpio/include/hal/rtc_io_hal.h-242- * Disable hold function on an RTC IO pad.
/root/esp/esp-idf/components/esp_hal_gpio/include/hal/rtc_io_hal.h-243- *
/root/esp/esp-idf/components/esp_hal_gpio/include/hal/rtc_io_hal.h-244- * @note If disable the pad hold, the status of pad maybe changed in sleep mode.
/root/esp/esp-idf/components/esp_hal_gpio/include/hal/rtc_io_hal.h-245- * @param rtcio_num The index of rtcio. 0 ~ SOC_RTCIO_PIN_COUNT.
(จบ Z9)
### Z10 ผู้เรียก esp_sleep_isolate_digital_gpio
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_sleep_internal.h:112:void esp_sleep_isolate_digital_gpio(bool do_backup);
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_sleep_internal.h:118: * otherwise full pad isolate like esp_sleep_isolate_digital_gpio. Intended after SPI bus is idle
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_sleep_internal.h:126: * Reverses esp_sleep_isolate_digital_gpio / esp_sleep_isolate_mspi_gpio. Call after wake before
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:281:IRAM_ATTR void esp_sleep_isolate_digital_gpio(bool do_backup)
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:862:        esp_sleep_isolate_digital_gpio(false);
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:904:            esp_sleep_isolate_digital_gpio(true);
(จบ Z10)
### Z11 คำว่า hold ใน sleep_modes.rst และ gpio.rst
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:244:    If we turn off the ``RTC_PERIPH`` domain, we will use the HOLD feature to maintain the pull-up and pull-down on the pins during sleep. HOLD feature will be acted on the pin internally before the system enters sleep modes, and this can further reduce power consumption::
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:249:    If certain chips lack the ``RTC_PERIPH`` domain, we can only use the HOLD feature to maintain the pull-up and pull-down on the pins during sleep modes::
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:272:        - If the RTC peripherals are configured to be powered down (which is by default), the wakeup IOs will be set to the holding state before entering sleep. Therefore, after the chip wakes up from Light-sleep, please call ``rtc_gpio_hold_dis`` to disable the hold function to perform any pin re-configuration. For Deep-sleep wakeup, this is already being handled at the application startup stage.
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:413:    **Mode 0 (UART_WK_MODE_ACTIVE_THRESH) - Active Edge Threshold Wakeup**
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:415:        When all clocks are powered down, the chip can be woken up by toggling the RXD pin for a certain number of cycles. The chip wakes up when the number of rising edges is greater than or equal to threshold value. The threshold value can be configured using the ``rx_edge_threshold`` field in :cpp:type:`uart_wakeup_cfg_t` structure.
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:419:    **Mode 1 (UART_WK_MODE_FIFO_THRESH) - RX FIFO Threshold Wakeup**
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:421:        Since the UART Core clock remains active, the UART RX can still receive data and store it in the RX FIFO. The chip can be woken up from Light-sleep when the number of bytes in the RX FIFO exceeds the configured threshold. The threshold value can be configured using the ``rx_fifo_threshold`` field in :cpp:type:`uart_wakeup_cfg_t` structure.
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:439:    After waking-up from UART wakeup mode 0, you should send some extra data through the UART port or reset the UART module in Active mode, otherwise, the next UART wake-up would trigger with two less rising edges than the configured threshold value.
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:449:    LP_UART can wake up the ULP LP core coprocessor. LP_UART supports the same wakeup modes as the HP UART described above, including active edge threshold wakeup, RX FIFO threshold wakeup, start bit detection wakeup, and character sequence detection wakeup.
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:602:.. only:: SOC_RTCIO_HOLD_SUPPORTED and SOC_RTCIO_INPUT_OUTPUT_SUPPORTED
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:616:        - RTC GPIOs (GPIO0 ~ 5) can be in the following states, depending on their hold function enabled or not:
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:617:            - if the hold function is not enabled, RTC GPIOs will be in a high impedance state.
/root/esp/esp-idf/docs/en/api-reference/system/sleep_modes.rst:618:            - if the hold function is enabled, RTC GPIOs will retain the pin state latched at that hold moment.
(จบ Z11)
### Z12 sleep_modes.rst บรรทัด 380-400

            - Deep-sleep mode (always)
            - Light-sleep mode when :ref:`CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP` is enabled

            .. only:: SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED

                On {IDF_TARGET_NAME} the API also accepts edge-triggered wakeup modes: ``ESP_GPIO_WAKEUP_GPIO_POSEDGE`` (rising edge), ``ESP_GPIO_WAKEUP_GPIO_NEGEDGE`` (falling edge), and ``ESP_GPIO_WAKEUP_GPIO_ANYEDGE`` (both edges). For ``ESP_GPIO_WAKEUP_GPIO_ANYEDGE``, the internal pull-up/pull-down is disabled (when :ref:`CONFIG_ESP_SLEEP_GPIO_ENABLE_INTERNAL_RESISTORS` is enabled) because the idle level is ambiguous; an external pull or an already-stable line is recommended.

            .. note::
                Only GPIOs powered by the VDD3P3_RTC power domain (RTC IOs) can be used with this API. The exact set of supported pins can be checked in the `datasheet <{IDF_TARGET_DATASHEET_EN_URL}>`__ > Section IO Pins.

    .. note::
        Any GPIO/IO wakeup signal -- whether level or edge -- must be held (or have a pulse width) of at least 3 RTC slow-clock cycles to be reliably sampled by the wakeup logic. The duration of one slow-clock cycle depends on :ref:`CONFIG_RTC_CLK_SRC` (e.g. RC_SLOW @ ~136 kHz ~= 7.4 us/cycle, XTAL32K @ 32.768 kHz ~= 30.5 us/cycle). This applies to both :cpp:func:`esp_sleep_enable_gpio_wakeup` and :cpp:func:`esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown`.

.. only:: esp32h2

    GPIO Wakeup
    ^^^^^^^^^^^

    Any IO can be used as the external input to wake up the chip from Light-sleep. Each pin can be individually configured using :cpp:func:`gpio_wakeup_enable` (low/high level only). Then the :cpp:func:`esp_sleep_enable_gpio_wakeup` function should be called to enable this wakeup source.

(จบ Z12)
```