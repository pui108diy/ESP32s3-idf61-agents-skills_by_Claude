## verify_gpio_unv7.sh

``` bash
#!/usr/bin/env bash
# verify_gpio_unv7.sh (รอบ 5h): ปิดข้อที่ verify_gpio_unv6.sh ยังตอบไม่ครบ (อ่านอย่างเดียว)
# Z13 พิสูจน์: sleep_modes.c รอบบรรทัด 862 และ 904 อยู่ในเส้นทาง deep sleep หรือ light sleep (ใช้ตัดสินขอบเขต GPIO-32)
# Z14 พิสูจน์: IO_WAKEUP_ENABLED นิยามอย่างไร (เงื่อนไขปลด hold ตอนตื่นจาก deep sleep)
# Z15 พิสูจน์: esp_gpio_is_reserved ตัดขาใดออกจากการ isolate
# Z16 พิสูจน์: บริบทของ sleep_modes.rst บรรทัด 236-275 และ 596-625 (หัวข้อ/เงื่อนไขชิปของข้อความเรื่อง hold)
# Z17 พิสูจน์: gpio_hal_hold_en / gpio_hal_hold_dis / gpio_hal_is_digital_io_hold บน S3 แยกขา RTC กับ digital อย่างไร
# วิธีใช้
#   . $IDF_PATH/export.sh
#   bash verify_gpio_unv7.sh > verify7.log 2>&1
set -u
: "${IDF_PATH:?ต้อง source export.sh ก่อน}"
C="$IDF_PATH/components"
D="$IDF_PATH/docs/en/api-reference"

echo "### Z13 sleep_modes.c บรรทัด 840-915"
sed -n '840,915p' "$C/esp_hw_support/sleep_modes.c"
echo "(จบ Z13)"

echo "### Z14 IO_WAKEUP_ENABLED"
grep -rn "IO_WAKEUP_ENABLED" "$C/esp_hw_support"
echo "(จบ Z14)"

echo "### Z15 esp_gpio_is_reserved"
grep -rn "esp_gpio_is_reserved" "$C" --include=*.c --include=*.h
echo "(จบ Z15)"

echo "### Z16 sleep_modes.rst"
sed -n '236,275p' "$D/system/sleep_modes.rst"
echo "--- บรรทัด 596-625"
sed -n '596,625p' "$D/system/sleep_modes.rst"
echo "(จบ Z16)"

echo "### Z17 gpio_hal.h และ gpio_ll.h ของ S3"
grep -n -B1 -A6 -E "gpio_hal_hold_en|gpio_hal_hold_dis|gpio_hal_is_digital_io_hold" "$C/esp_hal_gpio/include/hal/gpio_hal.h"
echo "--- gpio_ll.h ของ S3"
find "$C/esp_hal_gpio" -path '*esp32s3*' -name 'gpio_ll.h' -exec grep -n -A8 -E "gpio_ll_hold_en|gpio_ll_hold_dis|gpio_ll_is_digital_io_hold|gpio_ll_get_digital_gpio_hold_mask" {} +
echo "(จบ Z17)"
```

## verify7.log ผลที่ได้ 

``` code
### Z13 sleep_modes.c บรรทัด 840-915
        s_config.fast_clk_cal_period = rtc_clk_cal(CLK_CAL_RC_FAST, FAST_CLK_SRC_CAL_CYCLES);
    }
#endif
}


static uint32_t call_rtc_sleep_start(uint32_t reject_triggers, uint32_t lslp_mem_inf_fpu, bool dslp);

#if SOC_PMU_SUPPORTED
static esp_err_t FORCE_IRAM_ATTR esp_sleep_start_safe(uint32_t sleep_flags, uint32_t reject_triggers, bool deep_sleep, pmu_sleep_config_t *config)
#else
static esp_err_t FORCE_IRAM_ATTR esp_sleep_start_safe(uint32_t sleep_flags, uint32_t reject_triggers, bool deep_sleep, rtc_sleep_config_t *config)
#endif
{
    esp_err_t result = ESP_OK;
#if CONFIG_ESP_SLEEP_DEBUG
    if (s_sleep_ctx != NULL) {
        s_sleep_ctx->wakeup_triggers = s_config.wakeup_triggers;
    }
#endif
    if (deep_sleep) {
#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
        esp_sleep_isolate_digital_gpio(false);
#endif

#if CONFIG_IDF_TARGET_ESP32P4 && CONFIG_ESP_SLEEP_SET_FLASH_DPD
        if ((sleep_flags & RTC_SLEEP_FLASH_DPD) && (!ESP_CHIP_REV_ABOVE(efuse_hal_chip_revision(), 300))) {
            /* Switch Flash from standby mode to deep powerdown mode */
            /* During bootloader phase following wakeup from deepsleep, flash will exit dpd mode */
            spi_flash_enable_deep_power_down_mode(true, false);
        }
#endif

#if ESP_ROM_SUPPORT_DEEP_SLEEP_WAKEUP_STUB && SOC_DEEP_SLEEP_SUPPORTED
#if SOC_PM_SUPPORT_DEEPSLEEP_CHECK_STUB_ONLY
        esp_set_deep_sleep_wake_stub_default_entry();
#elif !CONFIG_ESP_SYSTEM_ALLOW_RTC_FAST_MEM_AS_HEAP && SOC_RTC_FAST_MEM_SUPPORTED
        /* If not possible stack is in RTC FAST memory, use the ROM function to calculate the CRC and save ~140 bytes IRAM */
        set_rtc_memory_crc();
#endif // SOC_PM_SUPPORT_DEEPSLEEP_CHECK_STUB_ONLY
#endif

#if SOC_DCDC_SUPPORTED
        uint64_t ldo_increased_us = rtc_time_slowclk_to_us(rtc_time_get() - s_config.rtc_ticks_at_ldo_prepare, s_config.rtc_clk_cal_period);
        if (ldo_increased_us < LDO_POWER_TAKEOVER_PREPARATION_TIME_US) {
            esp_rom_delay_us(LDO_POWER_TAKEOVER_PREPARATION_TIME_US - ldo_increased_us);
        }
        pmu_sleep_shutdown_dcdc();
#endif

        // Enter Deep Sleep
#if !ESP_ROM_SUPPORT_DEEP_SLEEP_WAKEUP_STUB || SOC_PM_SUPPORT_DEEPSLEEP_CHECK_STUB_ONLY || !CONFIG_ESP_SYSTEM_ALLOW_RTC_FAST_MEM_AS_HEAP
#if SOC_PMU_SUPPORTED
        result = call_rtc_sleep_start(reject_triggers, config->power.hp_sys.dig_power.mem_dslp, deep_sleep);
#else
        result = call_rtc_sleep_start(reject_triggers, config->lslp_mem_inf_fpu, deep_sleep);
#endif
#else
            /* Otherwise, need to call the dedicated soc function for this */
        result = rtc_deep_sleep_start(s_config.wakeup_triggers, reject_triggers);
#endif
    } else {
#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
        if (sleep_flags & RTC_SLEEP_PD_DIG) {
            esp_sleep_isolate_digital_gpio(true);
        }
#endif
        /* Cache Suspend 1: will wait cache idle in cache suspend */
        sleep_cache_suspend();
        if (!(sleep_flags & RTC_SLEEP_PD_VDDSDIO)) {
#if CONFIG_ESP_SLEEP_SET_FLASH_DPD
            if (sleep_flags & RTC_SLEEP_FLASH_DPD) {
                /* Switch Flash from standby mode to deep powerdown mode */
                spi_flash_enable_deep_power_down_mode(true, false);
            }
#endif
(จบ Z13)
### Z14 IO_WAKEUP_ENABLED
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:57:#define IO_WAKEUP_ENABLED   (RTC_GPIO_TRIG_EN | _IO_WAKEUP_EXT0 | _IO_WAKEUP_EXT1)
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:396:    uint32_t io_wakeup_enabled = pmu_ll_hp_get_wakeup_enable(&PMU) & IO_WAKEUP_ENABLED;
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:398:    uint32_t io_wakeup_enabled = rtc_cntl_ll_get_wakeup_enable() & IO_WAKEUP_ENABLED;
(จบ Z14)
### Z15 esp_gpio_is_reserved
/root/esp/esp-idf/components/esp_driver_gpio/src/gpio.c:381:            // No need to reserve any pin for reading pad level (and do input check first, since output needs reserve IO which will make esp_gpio_is_reserved return true)
/root/esp/esp-idf/components/esp_driver_gpio/src/gpio.c:383:            if (esp_gpio_is_reserved(bit_mask) && gpio_hal_input_is_enabled(gpio_context.gpio_hal, io_num)) {
/root/esp/esp-idf/components/esp_driver_gpio/src/gpio.c:396:                if (esp_gpio_is_reserved(bit_mask) && !gpio_hal_input_is_enabled(gpio_context.gpio_hal, io_num)) {
/root/esp/esp-idf/components/esp_driver_gpio/src/gpio.c:1150:        fprintf(out_stream, "IO[%"PRIu32"]%s -\n", gpio_num, esp_gpio_is_reserved(BIT64(gpio_num)) ? " **RESERVED**" : "");
/root/esp/esp-idf/components/esp_driver_i2s/i2s_common.c:1098:    if (esp_gpio_is_reserved(BIT64(gpio_num))) {
/root/esp/esp-idf/components/esp_eth/src/mac/esp_eth_mac_esp_gpio.c:40:            ESP_RETURN_ON_FALSE(esp_gpio_is_reserved(BIT64(gpio_num)) == false, ESP_ERR_INVALID_STATE,
/root/esp/esp-idf/components/esp_hw_support/esp_gpio_reserve.c:25:bool esp_gpio_is_reserved(uint64_t gpio_mask)
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_gpio_reserve.h:57:bool esp_gpio_is_reserved(uint64_t gpio_mask);
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:318:            !esp_gpio_is_reserved(BIT64(gpio_num))) {
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:346:            !esp_gpio_is_reserved(BIT64(gpio_num))) {
/root/esp/esp-idf/components/esp_phy/src/phy_common.c:197:            if(esp_gpio_is_reserved(config->gpio_cfg[i].gpio_num)) {
(จบ Z15)
### Z16 sleep_modes.rst
        - wake up if any of the selected pins is low (``ESP_EXT1_WAKEUP_ANY_LOW``)

    This wakeup source is controlled by the RTC controller. It supports wakeup even when the RTC peripheral is powered down. Although the power domain of the RTC peripheral, where RTC IOs are located, is powered down during sleep modes, ESP-IDF will automatically lock the state of the wakeup pin before the system enters sleep modes and unlock upon exiting sleep modes. Therefore, the internal pull-up or pull-down resistors can still be configured for the wakeup pin::

        esp_sleep_pd_config(ESP_PD_DOMAIN_RTC_PERIPH, ESP_PD_OPTION_ON);
        rtc_gpio_pullup_dis(gpio_num);
        rtc_gpio_pulldown_en(gpio_num);

    If we turn off the ``RTC_PERIPH`` domain, we will use the HOLD feature to maintain the pull-up and pull-down on the pins during sleep. HOLD feature will be acted on the pin internally before the system enters sleep modes, and this can further reduce power consumption::

        rtc_gpio_pullup_dis(gpio_num);
        rtc_gpio_pulldown_en(gpio_num);

    If certain chips lack the ``RTC_PERIPH`` domain, we can only use the HOLD feature to maintain the pull-up and pull-down on the pins during sleep modes::

        gpio_pullup_dis(gpio_num);
        gpio_pulldown_en(gpio_num);

    :cpp:func:`esp_sleep_enable_ext1_wakeup_io` function can be used to append ext1 wakeup IO and set corresponding wakeup level.

    :cpp:func:`esp_sleep_disable_ext1_wakeup_io` function can be used to remove ext1 wakeup IO.

    .. only:: SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN

        The RTC controller also supports triggering wakeup, allowing configurable IO to use different wakeup levels simultaneously. This can be configured with :cpp:func:`esp_sleep_enable_ext1_wakeup_io`.

    .. only:: not SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN

        .. note::

           Due to hardware limitation, when we use more than one IO for EXT1 wakeup, it is not allowed to configure different wakeup levels for the IOs, and there is corresponding inspection mechanism in :cpp:func:`esp_sleep_enable_ext1_wakeup_io`.

    .. warning::

        - To use the EXT1 wakeup, the IO pad(s) are configured as RTC IO. Therefore, before using these pads as digital GPIOs, users need to reconfigure them by calling the :cpp:func:`rtc_gpio_deinit` function.

        - If the RTC peripherals are configured to be powered down (which is by default), the wakeup IOs will be set to the holding state before entering sleep. Therefore, after the chip wakes up from Light-sleep, please call ``rtc_gpio_hold_dis`` to disable the hold function to perform any pin re-configuration. For Deep-sleep wakeup, this is already being handled at the application startup stage.

.. only:: SOC_ULP_SUPPORTED

--- บรรทัด 596-625

Configuring IOs (Deep-sleep Only)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Some {IDF_TARGET_NAME} IOs have internal pullups or pulldowns, which are enabled by default. If an external circuit drives this pin in Deep-sleep mode, current consumption may increase due to current flowing through these pullups and pulldowns.

.. only:: SOC_RTCIO_HOLD_SUPPORTED and SOC_RTCIO_INPUT_OUTPUT_SUPPORTED

    To isolate a pin to prevent extra current draw, call :cpp:func:`rtc_gpio_isolate` function.

    For example, on ESP32-WROVER module, GPIO12 is pulled up externally, and it also has an internal pulldown in the ESP32 chip. This means that in Deep-sleep, some current flows through these external and internal resistors, increasing Deep-sleep current above the minimal possible value.

    Add the following code before :cpp:func:`esp_deep_sleep_start` to remove such extra current::

        rtc_gpio_isolate(GPIO_NUM_12);

.. only:: esp32c2 or esp32c3

    In Deep-sleep mode:
        - digital GPIOs (GPIO6 ~ 21) are in a high impedance state.
        - RTC GPIOs (GPIO0 ~ 5) can be in the following states, depending on their hold function enabled or not:
            - if the hold function is not enabled, RTC GPIOs will be in a high impedance state.
            - if the hold function is enabled, RTC GPIOs will retain the pin state latched at that hold moment.

.. _enter_sleep:

Entering Sleep
--------------

:cpp:func:`esp_light_sleep_start` or :cpp:func:`esp_deep_sleep_start` functions can be used to enter Light-sleep or Deep-sleep modes correspondingly. After that, the system configures the parameters of RTC controller according to the requested wakeup sources and power-down options.
(จบ Z16)
### Z17 gpio_hal.h และ gpio_ll.h ของ S3
300-  */
301:#define gpio_hal_hold_en(hal, gpio_num) gpio_ll_hold_en((hal)->dev, gpio_num)
302-
303-/**
304-  * @brief Disable gpio pad hold function.
305-  *
306-  * When the chip is woken up from Deep-sleep, the gpio will be set to the default mode, so, the gpio will output
307-  * the default level if this function is called. If you don't want the level changes, the gpio should be configured to
--
316-  */
317:#define gpio_hal_hold_dis(hal, gpio_num) gpio_ll_hold_dis((hal)->dev, gpio_num)
318-
319-/**
320-  * @brief Get whether digital gpio pad is held
321-  *
322-  * @param hal Context of the HAL layer
323-  * @param gpio_num GPIO number, only support output GPIOs
--
332-  */
333:#define gpio_hal_is_digital_io_hold(hal, gpio_num) gpio_ll_is_digital_io_hold((hal)->dev, gpio_num)
334-
335-#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP
336-/**
337-  * @brief Enable all digital gpio pad hold function during Deep-sleep.
338-  *
339-  * When the chip is in Deep-sleep mode, all digital gpio will hold the state before sleep, and when the chip is woken up,
--- gpio_ll.h ของ S3
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:471:static inline void gpio_ll_hold_en(gpio_dev_t *hw, uint32_t gpio_num)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-472-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-473-    SET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-474-}
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-475-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-476-/**
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-477-  * @brief Disable gpio pad hold function.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-478-  *
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-479-  * @param hw Peripheral GPIO hardware instance address.
--
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:482:static inline void gpio_ll_hold_dis(gpio_dev_t *hw, uint32_t gpio_num)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-483-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-484-    CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-485-}
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-486-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-487-/**
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-488-  * @brief Get gpio pad hold status.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-489-  *
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-490-  * @param hw Peripheral GPIO hardware instance address.
--
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:500:static inline bool gpio_ll_is_digital_io_hold(gpio_dev_t *hw, uint32_t gpio_num)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-501-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-502-    uint64_t bit_mask = 1ULL << gpio_num;
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-503-    if (!(bit_mask & SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK)) {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-504-        // GPIO 0-21 are not digital IO pads, 22-25 does not exist
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-505-        abort();
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-506-    }
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-507-    return GET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, bit_mask >> 21);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-508-}
--
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h:88:static inline uint64_t gpio_ll_get_digital_gpio_hold_mask(gpio_dev_t *hw)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-89-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-90-    (void)hw;
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-91-    uint32_t hold0 = LP_SYS.hp_gpio_o_hold_ctrl0.hp_gpio_0_hold_ctrl0;
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-92-    uint32_t hold1 = LP_SYS.hp_gpio_o_hold_ctrl1.hp_gpio_0_hold_ctrl1;
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-93-    return ((uint64_t)hold0 << SOC_RTCIO_PIN_COUNT) |
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-94-           ((uint64_t)hold1 << (32 + SOC_RTCIO_PIN_COUNT));
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-95-}
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-96-
--
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h:580:static inline void gpio_ll_hold_en(gpio_dev_t *hw, uint32_t gpio_num)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-581-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-582-    if (gpio_num < SOC_RTCIO_PIN_COUNT) {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-583-        REG_SET_BIT(LP_SYSTEM_REG_PADCTRL_REG, (0x1 << (gpio_num + SOC_RTCIO_PIN_COUNT)));
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-584-    } else if (gpio_num < SOC_RTCIO_PIN_COUNT + 32) {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-585-        REG_SET_BIT(LP_SYSTEM_REG_HP_GPIO_O_HOLD_CTRL0_REG, (0x1 << (gpio_num - SOC_RTCIO_PIN_COUNT)));
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-586-    } else {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-587-        REG_SET_BIT(LP_SYSTEM_REG_HP_GPIO_O_HOLD_CTRL1_REG, (0x1 << (gpio_num - SOC_RTCIO_PIN_COUNT - 32)));
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-588-    }
--
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h:598:static inline void gpio_ll_hold_dis(gpio_dev_t *hw, uint32_t gpio_num)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-599-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-600-    if (gpio_num < SOC_RTCIO_PIN_COUNT) {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-601-        REG_CLR_BIT(LP_SYSTEM_REG_PADCTRL_REG, (0x1 << (gpio_num + SOC_RTCIO_PIN_COUNT)));
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-602-    } else if (gpio_num < SOC_RTCIO_PIN_COUNT + 32) {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-603-        REG_CLR_BIT(LP_SYSTEM_REG_HP_GPIO_O_HOLD_CTRL0_REG, (0x1 << (gpio_num - SOC_RTCIO_PIN_COUNT)));
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-604-    } else {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-605-        REG_CLR_BIT(LP_SYSTEM_REG_HP_GPIO_O_HOLD_CTRL1_REG, (0x1 << (gpio_num - SOC_RTCIO_PIN_COUNT - 32)));
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-606-    }
--
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h:622:static inline bool gpio_ll_is_digital_io_hold(gpio_dev_t *hw, uint32_t gpio_num)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-623-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-624-    uint64_t bit_mask = 1ULL << gpio_num;
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-625-    if (!(bit_mask & SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK)) {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-626-        // This function only handles digital IOs. If LP IO is queried here, there is a bug in the upper-level code.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-627-        // GPIO 0-7
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-628-        abort();
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-629-    } else {
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/gpio_ll.h-630-        if (gpio_num < 32 + SOC_RTCIO_PIN_COUNT) {
(จบ Z17)
```