## verify_gpio_unv5.sh

``` bash
#!/usr/bin/env bash
# verify_gpio_unv5.sh (รอบ 5f): ปิดข้อที่ verify_gpio_unv4.sh ยังตอบไม่ได้ (อ่านอย่างเดียว)
# Z1 พิสูจน์: sleep_gpio.c รอบบรรทัด 286 ทำอะไรเมื่อ deep-sleep hold ไม่ถูกเปิด (ใช้ตอบว่า RTC GPIO ต้องคู่ gpio_deep_sleep_hold_en หรือไม่)
# Z2 พิสูจน์: ไฟล์ rtc_io_ll.h ของ S3 อยู่ที่ไหน (Y7 ว่างเพราะ path ที่เดาผิด)
# Z3 พิสูจน์: rtcio_ll_hold_enable ของ S3 แตะรีจิสเตอร์ใด (hold ของ RTC pad)
# Z4 พิสูจน์: gpio_ll_deep_sleep_hold_en ของ S3 แตะรีจิสเตอร์ใด (auto-hold ของ digital pad)
# Z5 พิสูจน์: เอกสาร sleep_modes.rst มีข้อความเรื่องความกว้างพัลส์/จำนวนรอบ slow clock ของ GPIO wakeup หรือไม่
# Z6 พิสูจน์: ค่า default และช่วงของ FREERTOS_ISR_STACKSIZE
# วิธีใช้
#   . $IDF_PATH/export.sh
#   bash verify_gpio_unv5.sh > verify5.log 2>&1
set -u
: "${IDF_PATH:?ต้อง source export.sh ก่อน}"
C="$IDF_PATH/components"

echo "### Z0 รุ่นของ IDF"
(cd "$IDF_PATH" && git describe --tags 2>&1)

echo "### Z1 sleep_gpio.c บรรทัด 260-320 และทุกบรรทัดที่มีคำว่า hold"
sed -n '260,320p' "$C/esp_hw_support/sleep_gpio.c"
echo "--- บรรทัดที่มีคำว่า hold"
grep -n -i "hold" "$C/esp_hw_support/sleep_gpio.c"
echo "(จบ Z1)"

echo "### Z2 ตำแหน่งไฟล์ rtc_io_ll.h ของ S3"
find "$C" -path '*esp32s3*' -name 'rtc_io_ll.h'
echo "(จบ Z2)"

echo "### Z3 rtcio_ll_hold_enable ของ S3"
find "$C" -path '*esp32s3*' -name 'rtc_io_ll.h' -exec grep -n -A10 "rtcio_ll_hold_enable" {} +
echo "(จบ Z3)"

echo "### Z4 gpio_ll_deep_sleep_hold ของ S3"
find "$C" -path '*esp32s3*' -name 'gpio_ll.h' -exec grep -n -A8 "gpio_ll_deep_sleep_hold" {} +
echo "(จบ Z4)"

echo "### Z5 sleep_modes.rst: คำว่า slow clock / clock cycle / pulse"
grep -n -i -E "slow clock|clock cycle|pulse" "$IDF_PATH/docs/en/api-reference/system/sleep_modes.rst"
echo "(จบ Z5)"

echo "### Z6 FREERTOS_ISR_STACKSIZE"
grep -n -B2 -A12 "config FREERTOS_ISR_STACKSIZE" "$C/freertos/Kconfig"
echo "(จบ Z6)"
```


## verify5.log ผลที่ได้

``` code
### Z0 รุ่นของ IDF
v6.1
### Z1 sleep_gpio.c บรรทัด 260-320 และทุกบรรทัดที่มีคำว่า hold
    }
}

IRAM_ATTR void esp_sleep_restore_isolated_digital_gpio(void)
{
    uint64_t backuped = s_gpio_isolate_backup.backuped;
    while (backuped) {
        gpio_num_t gpio_num = (gpio_num_t)__builtin_ctzll(backuped);
        gpio_io_config_t io_config = {
            .pu = (s_gpio_isolate_backup.pu >> gpio_num) & 0x1,
            .pd = (s_gpio_isolate_backup.pd >> gpio_num) & 0x1,
            .ie = (s_gpio_isolate_backup.ie >> gpio_num) & 0x1,
            .oe = (s_gpio_isolate_backup.oe >> gpio_num) & 0x1,
            .fun_sel = (uint32_t)s_gpio_isolate_backup.fun_sel[gpio_num],
        };
        gpio_ll_set_pad_config_for_sleep_isolate(gpio_num, &io_config);
        backuped &= backuped - 1;
    }
}
#endif // SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD

IRAM_ATTR void esp_sleep_isolate_digital_gpio(bool do_backup)
{
    gpio_hal_context_t gpio_hal = { .dev = GPIO_HAL_GET_HW(GPIO_PORT_0) };
#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP
    /* no need to do isolate if digital IOs are not being held in deep sleep */
    if (!gpio_hal_deep_sleep_hold_is_en(&gpio_hal)) {
        return;
    }
#endif

    /**
     * there is a situation where we cannot isolate digital IO before deep sleep:
     * - task stack is located in external ram(mspi ram), since we will isolate mspi io
     *
     * assert here instead of returning directly, because if digital IO is not isolated,
     * the bottom current of deep sleep will be higher than light sleep, and there is no
     * reason to use deep sleep at this time.
     */
    assert(esp_ptr_internal(esp_cpu_get_sp()) && "If hold digital IO, the stack of the task calling esp_deep_sleep_start must be in internal ram!");

    DRAM_ATTR static volatile uint64_t s_pad_mask = SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK;
    uint64_t pad_mask = s_pad_mask;
#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
    if (do_backup) {
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
--- บรรทัดที่มีคำว่า hold
197:#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
284:#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP
286:    if (!gpio_hal_deep_sleep_hold_is_en(&gpio_hal)) {
299:    assert(esp_ptr_internal(esp_cpu_get_sp()) && "If hold digital IO, the stack of the task calling esp_deep_sleep_start must be in internal ram!");
312:    uint64_t hold_mask = gpio_ll_get_digital_gpio_hold_mask(gpio_hal.dev);
317:        if (!(hold_mask & (1ULL << gpio_num)) &&
345:        if (!gpio_hal_is_digital_io_hold(&gpio_hal, gpio_num) &&
353:#endif //!SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
360:    // Disable ext1 wakeup before releasing hold, such that wakeup status can reflect the correct wakeup pin
367:#if SOC_RTCIO_HOLD_SUPPORTED
368:        rtcio_hal_hold_disable(rtcio_num);
381:            // Disable the wakeup before releasing hold, such that wakeup status can reflect the correct wakeup pin
383:            gpio_hal_hold_dis(&gpio_hal, gpio_num);
394:    // Need to unhold the IOs that were hold right before entering deep sleep, which are used as wakeup pins
(จบ Z1)
### Z2 ตำแหน่งไฟล์ rtc_io_ll.h ของ S3
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/rtc_io_ll.h
/root/esp/esp-idf/components/esp_hal_gpio/esp32s31/include/hal/rtc_io_ll.h
(จบ Z2)
### Z3 rtcio_ll_hold_enable ของ S3
(จบ Z3)
### Z4 gpio_ll_deep_sleep_hold ของ S3
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:424:static inline void _gpio_ll_deep_sleep_hold_en(gpio_dev_t *hw)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-425-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-426-    CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_FORCE_UNHOLD);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-427-    SET_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_AUTOHOLD_EN_M);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-428-}
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-429-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-430-/// use a macro to wrap the function, force the caller to use it in a critical section
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-431-/// the critical section needs to declare the __DECLARE_RCC_ATOMIC_ENV variable in advance
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-432-/// When operating RTC_CNTL_DIG_ISO_REG, PERIPH_RCC_ATOMIC() must be used to ensure atomicity.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:433:#define gpio_ll_deep_sleep_hold_en(...) (void)__DECLARE_RCC_ATOMIC_ENV; _gpio_ll_deep_sleep_hold_en(__VA_ARGS__)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-434-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-435-/**
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-436-  * @brief Disable all digital gpio pad hold function during Deep-sleep.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-437-  *
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-438-  * @param hw Peripheral GPIO hardware instance address.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-439-  */
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:440:static inline void _gpio_ll_deep_sleep_hold_dis(gpio_dev_t *hw)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-441-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-442-    CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_AUTOHOLD_EN_M);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-443-}
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-444-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-445-/// use a macro to wrap the function, force the caller to use it in a critical section
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-446-/// the critical section needs to declare the __DECLARE_RCC_ATOMIC_ENV variable in advance
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-447-/// When operating RTC_CNTL_DIG_ISO_REG, PERIPH_RCC_ATOMIC() must be used to ensure atomicity.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:448:#define gpio_ll_deep_sleep_hold_dis(...) (void)__DECLARE_RCC_ATOMIC_ENV; _gpio_ll_deep_sleep_hold_dis(__VA_ARGS__)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-449-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-450-/**
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-451- * @brief  Get deep sleep hold status
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-452- *
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-453- * @param  hw Peripheral GPIO hardware instance address.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-454- *
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-455- * @return
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-456- *     - true  deep sleep hold is enabled
--
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:460:static inline bool gpio_ll_deep_sleep_hold_is_en(gpio_dev_t *hw)
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-461-{
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-462-    return !GET_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_FORCE_UNHOLD) && GET_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_AUTOHOLD_EN_M);
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-463-}
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-464-
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-465-/**
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-466-  * @brief Enable gpio pad hold function.
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-467-  *
/root/esp/esp-idf/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h-468-  * @param hw Peripheral GPIO hardware instance address.
(จบ Z4)
### Z5 sleep_modes.rst: คำว่า slow clock / clock cycle / pulse
392:        Any GPIO/IO wakeup signal -- whether level or edge -- must be held (or have a pulse width) of at least 3 RTC slow-clock cycles to be reliably sampled by the wakeup logic. The duration of one slow-clock cycle depends on :ref:`CONFIG_RTC_CLK_SRC` (e.g. RC_SLOW @ ~136 kHz ~= 7.4 us/cycle, XTAL32K @ 32.768 kHz ~= 30.5 us/cycle). This applies to both :cpp:func:`esp_sleep_enable_gpio_wakeup` and :cpp:func:`esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown`.
(จบ Z5)
### Z6 FREERTOS_ISR_STACKSIZE
424-            default y
425-
426:        config FREERTOS_ISR_STACKSIZE
427-            int "ISR stack size"
428-            range 2096 32768 if ESP_COREDUMP_ENABLE
429-            default 2096 if ESP_COREDUMP_ENABLE
430-            range 1536 32768
431-            default 1536
432-            help
433-                The interrupt handlers have their own stack. The size of the stack can be defined here. Each processor
434-                has its own stack, so the total size occupied will be twice this.
435-
436-        config FREERTOS_INTERRUPT_BACKTRACE
437-            # Todo: Consider removing this. Not sure when users will ever want it to be disabled (IDF-4986)
438-            bool "Enable backtrace from interrupt to task context"
(จบ Z6)
```


