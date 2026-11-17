# evidence/gpio-evidence.md

> เวอร์ชัน v0.10 (รอบ 5) · ใช้ประกอบ `references/peripherals/gpio.md` v3.0-rc11 และ `hotlist/GPIO.md` v0.7 · EV-01..40

> จุดประสงค์: เก็บข้อความต้นฉบับที่ใช้ยืนยัน เพื่อให้ตรวจย้อนได้เมื่อไฟล์ต้นทางถูกถอดออกจาก RAG
> ข้อความคัดจากไฟล์ที่อยู่ใน RAG ณ รอบ 4 ย่อด้วย (...) ถ้าสงสัยให้ grep ไฟล์จริงตาม path (relative ต่อ `$IDF_PATH/components/`)
> รายการที่ระบุ "ดึงรอบ 2-3" หมายถึงยืนยันไว้ในรอบก่อน ไม่ได้ดึงข้อความซ้ำในรอบ 4 (ยืนยันซ้ำได้ด้วยสคริปต์ verify)

## EV-01 บล็อก GPIO CAPS ของ S3
- ไฟล์: `soc/esp32s3/include/soc/soc_caps.h`
- พิสูจน์: ขาที่มี, ไม่มี input-only, force hold, glitch filter, HP periph wake, ไม่มี flex/hys/hold-single-IO ในบล็อกนี้ (ถัดจากบล็อกนี้คือ I2C CAPS)
```c
#define SOC_GPIO_PIN_COUNT 49
#define SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER 1
// Force hold is a new function of ESP32-S3
#define SOC_GPIO_SUPPORT_FORCE_HOLD (1)
// 0~48 valid except 22~25
#define SOC_GPIO_VALID_GPIO_MASK (0x1FFFFFFFFFFFFULL & ~(0ULL | BIT22 | BIT23 | BIT24 | BIT25))
// No GPIO is input only
#define SOC_GPIO_VALID_OUTPUT_GPIO_MASK (SOC_GPIO_VALID_GPIO_MASK)
#define SOC_GPIO_IN_RANGE_MAX 48
#define SOC_GPIO_OUT_RANGE_MAX 48
// digital I/O pad powered by VDD3P3_CPU or VDD_SPI(GPIO_NUM_26~GPIO_NUM_48)
#define SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK 0x0001FFFFFC000000ULL
#define SOC_GPIO_CLOCKOUT_BY_IO_MUX (1)
#define SOC_GPIO_CLOCKOUT_CHANNEL_NUM (3)
// GPIO0~21 on ESP32-S3 can support chip deep sleep wakeup
#define SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP (1)
#define SOC_GPIO_SUPPORT_DEEPSLEEP_WAKEUP SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP
#define SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK (0ULL | BIT0 | BIT1 | (...) | BIT20 | BIT21)
```

## EV-02 บล็อก RTCIO CAPS ของ S3
- ไฟล์: `soc/esp32s3/include/soc/soc_caps.h`
```c
#define SOC_RTCIO_PIN_COUNT 22
#define SOC_RTCIO_INPUT_OUTPUT_SUPPORTED 1
#define SOC_RTCIO_HOLD_SUPPORTED 1
#define SOC_RTCIO_WAKE_SUPPORTED 1
// LP IO peripherals have independent clock gating to manage
#define SOC_LP_IO_CLOCK_IS_INDEPENDENT 1
```
- หมายเหตุ: ไม่มี `SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED` ในบล็อกนี้

## EV-03 บล็อก Power Management CAPS ของ S3
- ไฟล์: `soc/esp32s3/include/soc/soc_caps.h` (และ `Kconfig.soc_caps.in`)
```c
#define SOC_PM_SUPPORT_EXT0_WAKEUP (1)
#define SOC_PM_SUPPORT_EXT1_WAKEUP (1)
#define SOC_PM_SUPPORT_EXT_WAKEUP (1)
#define SOC_PM_SUPPORT_WIFI_WAKEUP (1)
#define SOC_PM_SUPPORT_BT_WAKEUP (1)
#define SOC_PM_SUPPORT_TOUCH_SENSOR_WAKEUP (1)
#define SOC_PM_SUPPORT_CPU_PD (1)
```
- พิสูจน์: ไม่มี `SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN` ในบล็อกนี้ (ทั้งฝั่ง .h และ Kconfig)

## EV-04 Kconfig mirror ไม่ตรงกับ soc_caps.h
- ไฟล์: `soc/esp32s3/include/soc/Kconfig.soc_caps.in`
```text
config SOC_GPIO_IN_RANGE_MAX  (ก่อนหน้านี้) hex default 0x1FFFFFFFFFFFF
config SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK
    int
    default 0
```
- พิสูจน์: mask แบบ expression ใน .h ถูกแปลงเป็นค่าที่ไม่ตรง (ค่า hex ด้านบนอยู่ก่อน IN_RANGE_MAX คือ VALID_OUTPUT_GPIO_MASK) (GPIO-28)

## EV-05 Dedicated GPIO และ filter clock
- ไฟล์: `soc/.../soc_caps.h` · `esp_hal_gpio/esp32s3/include/hal/dedic_gpio_caps.h` · `esp_hal_gpio/esp32s3/include/hal/gpio_caps.h`
```c
#define SOC_DEDICATED_GPIO_SUPPORTED 1
#define _DEDIC_GPIO_OUT_CHANS_PER_CPU 8
#define _DEDIC_GPIO_IN_CHANS_PER_CPU 8
#define _GPIO_FILTER_CLK_SUPPORT_APB 1
```

## EV-06 gpio_num.h ของ S3 และมาโครตรวจขา
- ไฟล์: `esp_hal_gpio/esp32s3/include/soc/gpio_num.h` · `esp_hal_gpio/include/hal/gpio_types.h` (มาโครอยู่ใน `hal/gpio_hal`/`gpio_types` ตามที่ปรากฏ)
```c
GPIO_NUM_21 = 21, /*!< GPIO21, input and output */
GPIO_NUM_26 = 26, /*!< GPIO26, input and output */
GPIO_NUM_48 = 48, /*!< GPIO48, input and output */
GPIO_NUM_MAX,
#define GPIO_IS_VALID_GPIO(gpio_num) ((gpio_num >= 0) && (gpio_num < SOC_GPIO_PIN_COUNT) && \
    (((1ULL << (gpio_num)) & SOC_GPIO_VALID_GPIO_MASK) != 0))
```

## EV-07 ISR service และ IRAM
- ไฟล์: `esp_driver_gpio/include/driver/gpio.h`
```c
// gpio_install_isr_service: return
// - ESP_OK Success
// - ESP_ERR_NO_MEM No memory to install this service
// - ESP_ERR_INVALID_STATE ISR service already installed.
// - ESP_ERR_NOT_FOUND No free interrupt found with the specified flags
// - ESP_ERR_INVALID_ARG GPIO error
// This function is incompatible with gpio_isr_register() - if that function is used, a single global ISR is
// registered for all GPIO interrupts. If this function is used, the ISR service provides a global GPIO ISR
// and individual pin handlers are registered via the gpio_isr_handler_add() function.

// gpio_isr_handler_add:
// The pin ISR handlers no longer need to be declared with IRAM_ATTR,
// unless you pass the ESP_INTR_FLAG_IRAM flag when (...)
// - ESP_ERR_INVALID_STATE Wrong state, the ISR service has not been initialized.
// - ESP_ERR_INVALID_ARG Parameter error
typedef void (*gpio_isr_t)(void *arg);
```

## EV-08 gpio.c: ISR service
- ไฟล์: `esp_driver_gpio/src/gpio.c`
```c
    // isr service already installed, free allocated resource
    ret = ESP_ERR_INVALID_STATE;
esp_err_t gpio_isr_register(void (*fn)(void *), void *arg, int intr_alloc_flags, gpio_isr_handle_t *handle)
{   GPIO_CHECK(fn, "GPIO ISR null", ESP_ERR_INVALID_ARG); (...)
esp_err_t gpio_isr_handler_remove(gpio_num_t gpio_num)
{   GPIO_CHECK(gpio_context.gpio_isr_func != NULL, "GPIO isr service is not installed, call gpio_install_isr_service() first", ESP_ERR_INVALID_STATE);
    (...) gpio_intr_disable(gpio_num); (...) gpio_context.gpio_isr_func[gpio_num].fn = NULL;
gpio_intr_enable_on_core(gpio_num, esp_intr_get_cpu(gpio_context.gpio_isr_handle));
```

## EV-09 gpio.c: gpio_config / gpio_set_direction / gpio_reset_pin
- ไฟล์: `esp_driver_gpio/src/gpio.c`
```c
if (pGPIOConfig->pin_bit_mask == 0 || pGPIOConfig->pin_bit_mask & ~SOC_GPIO_VALID_GPIO_MASK) {
    ESP_LOGE(GPIO_TAG, "GPIO_PIN mask error "); return ESP_ERR_INVALID_ARG; }
if (pGPIOConfig->mode & GPIO_MODE_DEF_OUTPUT && pGPIOConfig->pin_bit_mask & ~SOC_GPIO_VALID_OUTPUT_GPIO_MASK) {
    ESP_LOGE(GPIO_TAG, "GPIO can only be used as input mode"); return ESP_ERR_INVALID_ARG; }
// This function will set the pin to a certain mode, instead of adding new modes to current status
// If GPIO_MODE_DEF_OUTPUT flag is not set, output will be disabled; Same for GPIO_MODE_DEF_INPUT flag
gpio_set_intr_type(io_num, pGPIOConfig->intr_type);
if (pGPIOConfig->intr_type) { gpio_intr_enable(io_num); } else { gpio_intr_disable(io_num); }
/* By default, all the pins have to be configured as GPIO pins. */
gpio_hal_func_sel(gpio_context.gpio_hal, io_num, PIN_FUNC_GPIO);
// gpio_set_direction:
if ((GPIO_IS_VALID_OUTPUT_GPIO(gpio_num) != true) && (mode & GPIO_MODE_DEF_OUTPUT)) {
    ESP_LOGE(GPIO_TAG, "io_num=%d can only be input", gpio_num); return ESP_ERR_INVALID_ARG; }
// gpio_reset_pin:
gpio_intr_disable(gpio_num);
if (GPIO_IS_VALID_OUTPUT_GPIO(gpio_num)) { // for powersave reasons, the GPIO should not be floating, select pullup (...) }
gpio_pulldown_dis(gpio_num);
gpio_hal_func_sel(gpio_context.gpio_hal, gpio_num, PIN_FUNC_GPIO);
if (rtc_gpio_is_valid_gpio(gpio_num)) { rtc_gpio_deinit(gpio_num); }
```
- ไฟล์: `esp_driver_gpio/include/driver/gpio.h` (header ของ `gpio_set_direction`): "This function always overwrite all the current modes that have applied on the IO pin"

## EV-10 get_level / set_level / IRAM ของฟังก์ชันควบคุม
- ไฟล์: `esp_driver_gpio/include/driver/gpio.h`, `esp_driver_gpio/Kconfig`
```c
 * @warning If the pad is not configured for input (or input and output) the returned value is always 0.
 * @note This function is allowed to be executed when Cache is disabled within ISR context, by enabling `CONFIG_GPIO_CTRL_FUNC_IN_IRAM`
```
```text
config GPIO_CTRL_FUNC_IN_IRAM
    bool "Place GPIO control functions into IRAM"
    default n
    help
        Place GPIO control functions (like intr_disable/set_level) into IRAM, so that these functions can be IRAM-safe and able to be called in the other IRAM interrupt context.
```

## EV-11 gpio_hold_en / gpio_hold_dis
- ไฟล์: `esp_driver_gpio/include/driver/gpio.h` · `esp_driver_gpio/src/gpio.c` · `esp_hal_gpio/include/hal/gpio_hal.h`
```c
// gpio.h: When a GPIO is set to hold, its state is latched at that moment and will not change when the internal
// signal or the IO MUX/GPIO configuration is modified (including input enable, output enable, output value,
// function, and drive strength values).
// gpio.c:
esp_err_t gpio_hold_en(gpio_num_t gpio_num) {
    int ret = ESP_OK;
    if (rtc_gpio_is_valid_gpio(gpio_num)) {
#if SOC_RTCIO_HOLD_SUPPORTED
        ret = rtc_gpio_hold_en(gpio_num);
#endif
    } else if (GPIO_HOLD_MASK[gpio_num]) {
        GPIO_CHECK(GPIO_IS_VALID_OUTPUT_GPIO(gpio_num), "Only output-capable GPIO support this function", ESP_ERR_NOT_SUPPORTED);
        (...) gpio_hal_hold_en(gpio_context.gpio_hal, gpio_num); (...)
    } else { ret = ESP_ERR_NOT_SUPPORTED; }
// gpio_hal.h (hold_en):
// On ESP32/S2/C3/S3/C2, the state of digital gpio cannot be held during Deep-sleep, and it will resume the hold
// function when the chip wakes up from Deep-sleep. If the digital gpio also needs to be held during Deep-sleep,
// `gpio_deep_sleep_hold_en` should also be called.
// gpio_hal.h (hold_dis):
// When the chip is woken up from Deep-sleep, the gpio will be set to the default mode, so, the gpio will output
// the default level if this function is called. If you don't want the level changes, the gpio should be configured to
// a known state before this function is called. e.g. If you hold gpio18 high during (...)
```

## EV-12 gpio_deep_sleep_hold_en
- ไฟล์: `esp_driver_gpio/include/driver/gpio.h`
```c
// (...) gpio pads be at the holding state during Deep-sleep. The state of each pad holds is its active configuration (not pad's sleep configuration!).
// 1. For digital IO, this API takes effect only if the corresponding digital IO pad hold function has been enabled. You can enable the GPIO pad hold function by calling `gpio_hold_en`.
// 2. Though this API targets all digital IOs, the pad hold feature only works when the chip is in Deep-sleep mode. When the chip is in active mode, the digital GPIO state can be changed freely even if you have called this function, except for IOs that are already held by `gpio_hold_en`.
// After this API is being called, the digital gpio Deep-sleep hold feature will work during every sleep process. You should call `gpio_deep_sleep_hold_dis` to disable this feature.
void gpio_deep_sleep_hold_en(void);
void gpio_deep_sleep_hold_dis(void);
#endif //!SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP
```

## EV-13 force hold / unhold
- ไฟล์: `esp_driver_gpio/include/driver/gpio.h` · `src/gpio.c`
```c
// GPIO force hold, no matter the chip in active mode or sleep modes.
// This function will immediately cause all pads to latch the current values of input enable, output enable, output value, function, and drive strength values.
// @warning
// 1. This function will hold flash and UART pins as well. Therefore, this function, and all code run afterwards (till calling `gpio_force_unhold_all` to disable this feature), MUST be placed in internal RAM as holding the flash pins will halt SPI flash operation, and holding the UART pins will halt any UART logging.
// 2. The hold state of all pads will be cancelled during ROM boot, so it is not recommended to use this API to hold the pads state during deepsleep and reset.
esp_err_t gpio_force_hold_all(void);
// @note The global hold signal and the hold signal of each IO act on the PAD through 'or' logic, so if a pad has already been configured to hold by `gpio_hold_en`, this API can't release its hold state.
esp_err_t gpio_force_unhold_all(void);
// gpio.c: esp_err_t IRAM_ATTR gpio_force_unhold_all() { (...) gpio_hal_force_unhold_all(); (...) rtc_gpio_force_hold_dis_all(); }
```

## EV-14 gpio_wakeup_enable (light sleep)
- ไฟล์: `esp_driver_gpio/include/driver/gpio.h` · `src/gpio.c`
```c
// gpio.h: @param intr_type GPIO wake-up type. Only GPIO_INTR_LOW_LEVEL or GPIO_INTR_HIGH_LEVEL can be used.
// - ESP_ERR_INVALID_ARG Parameter error
esp_err_t gpio_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type);
// gpio.c (ส่วนที่ย่อ):
if ((intr_type == GPIO_INTR_LOW_LEVEL) || (intr_type == GPIO_INTR_HIGH_LEVEL)) {
#if SOC_RTCIO_WAKE_SUPPORTED
    if (rtc_gpio_is_valid_gpio(gpio_num)) {
#if SOC_LP_IO_CLOCK_IS_INDEPENDENT
        io_mux_enable_lp_io_clock(gpio_num, true);
#endif
        ret = rtc_gpio_wakeup_enable(gpio_num, intr_type); } #endif
    portENTER_CRITICAL(&gpio_context.gpio_spinlock);
    gpio_hal_set_intr_type(gpio_context.gpio_hal, gpio_num, intr_type);
    gpio_hal_wakeup_enable(gpio_context.gpio_hal, gpio_num);
#if CONFIG_ESP_SLEEP_GPIO_RESET_WORKAROUND || CONFIG_PM_SLP_DISABLE_GPIO
    gpio_hal_sleep_sel_dis(gpio_context.gpio_hal, gpio_num);
#endif
} else { ESP_LOGE(GPIO_TAG, "GPIO wakeup only supports level mode, but edge mode set. gpio_num:%u", gpio_num); (...) }
```

## EV-15 wakeup on hp periph powerdown
- ไฟล์: `esp_driver_gpio/include/driver/gpio.h` · `src/gpio.c`
```c
#define GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO(gpio_num) ((gpio_num >= 0) && \
    (((1ULL << (gpio_num)) & SOC_GPIO_HP_PERIPH_PD_SLEEP_WAKEABLE_MASK) != 0))
// Enable GPIO wake-up function on peripheral powerdowned sleep (including deepsleep and peripheral powerdowned lightsleep).
// @param intr_type: - Always supported: GPIO_INTR_LOW_LEVEL, GPIO_INTR_HIGH_LEVEL
//                   - If SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED: GPIO_INTR_POSEDGE, GPIO_INTR_NEGEDGE, (...)
// gpio.c:
if (!GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO(gpio_num)) {
    ESP_LOGE(GPIO_TAG, "GPIO %d does not support wakeup on peripheral powerdown sleep", gpio_num);
    return ESP_ERR_INVALID_ARG; }
#else  // !SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED
if ((intr_type != GPIO_INTR_LOW_LEVEL) && (intr_type != GPIO_INTR_HIGH_LEVEL)) { (...) return ESP_ERR_INVALID_ARG; }
```
- หมายเหตุ "Called by the SDK. User shouldn't call this directly in the APP": ดึงรอบ 2-3 (header ของฟังก์ชัน) ไม่ได้ดึงซ้ำรอบ 4

## EV-16 esp_sleep.h: gpio wakeup
- ไฟล์: `esp_hw_support/include/esp_sleep.h`
```c
// esp_sleep_enable_gpio_wakeup (light sleep):
// Each GPIO supports wakeup function, which can be triggered on either low level or high level. This method can be used with any IO (RTC or digital), whereas external RTC wakeup is limited to RTC GPIOs. It can only be used to wakeup from light sleep though.
// 2. If PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP is enabled (if target supported), this API is unavailable since the GPIO module is powered down during sleep. You can use `esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown` instead, or use EXT1 wakeup source by `esp_sleep_enable_ext1_wakeup_io` to achieve the same function.
// esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown:
// This function enables an IO pin to wake up the chip from peripheral powerdowned sleep. (including deepsleep and peripheral powerdowned lightsleep).
// @note 1.This function does not modify pin configuration. The pins are configured inside `esp_sleep_start`, immediately before entering sleep mode.
// @note You don't need to worry about pull-up or pull-down resistors before using this function because the ESP_SLEEP_GPIO_ENABLE_INTERNAL_RESISTORS option is enabled by default. (...)
esp_err_t esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown(uint64_t gpio_pin_mask, esp_sleep_gpio_wake_up_mode_t mode);
typedef enum { ESP_GPIO_WAKEUP_GPIO_LOW = 0, ESP_GPIO_WAKEUP_GPIO_HIGH = 1,
#if SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED
  ESP_GPIO_WAKEUP_GPIO_POSEDGE = 2, ESP_GPIO_WAKEUP_GPIO_NEGEDGE = 3, ESP_GPIO_WAKEUP_GPIO_ANYEDGE = 4,
#endif } esp_sleep_gpio_wake_up_mode_t;
bool esp_sleep_is_valid_wakeup_gpio(gpio_num_t gpio_num);
```

## EV-17 esp_sleep.h: EXT0 / EXT1
- ไฟล์: `esp_hw_support/include/esp_sleep.h`
```c
// ESP-S3 (non-ESP32):
typedef enum { ESP_EXT1_WAKEUP_ANY_LOW = 0, ESP_EXT1_WAKEUP_ANY_HIGH = 1,
    ESP_EXT1_WAKEUP_ALL_LOW __attribute__((deprecated("wakeup mode \"ALL_LOW\" is no longer supported after ESP32, please use ESP_EXT1_WAKEUP_ANY_LOW instead"))) = ESP_EXT1_WAKEUP_ANY_LOW } esp_sleep_ext1_wakeup_mode_t;
// ESP32-S3: 0-21 (รายการ RTC GPIO ที่ใช้เป็น io_mask)
// @note This function will be deprecated in release/v6.0. Please switch to use `esp_sleep_enable_ext1_wakeup_io` and `esp_sleep_disable_ext1_wakeup_io`
esp_err_t esp_sleep_enable_ext1_wakeup(uint64_t io_mask, esp_sleep_ext1_wakeup_mode_t level_mode);   // ไม่มี attribute deprecated
// This will append selected IOs to the wakeup IOs, it will not reset previously enabled IOs. To reset all the enabled IOs, call esp_sleep_disable_ext1_wakeup_io(0).
// - ESP_ERR_NOT_ALLOWED when wakeup level will become different between ext1 IOs if !SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN
esp_err_t esp_sleep_enable_ext1_wakeup_io(uint64_t io_mask, esp_sleep_ext1_wakeup_mode_t level_mode);
esp_err_t esp_sleep_disable_ext1_wakeup_io(uint64_t io_mask);
// @note Internal pullups and pulldowns don't work when RTC peripherals are shut down. In this case, external resistors need to be added.
// @return bit mask, if GPIOn caused wakeup, BIT(n) will be set
uint64_t esp_sleep_get_ext1_wakeup_status(void);    // ไม่มี attribute deprecated
#if SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN
esp_err_t esp_sleep_enable_ext1_wakeup_with_level_mask(uint64_t io_mask, uint64_t level_mask) __attribute__((deprecated("please use 'esp_sleep_enable_ext1_wakeup_io' and 'esp_sleep_disable_ext1_wakeup_io' instead")));
#endif
// EXT0: - ESP_ERR_INVALID_ARG if the selected GPIO is not an RTC GPIO, or the mode is invalid
//       - ESP_ERR_INVALID_STATE if wakeup triggers conflict
esp_err_t esp_sleep_enable_ext0_wakeup(gpio_num_t gpio_num, int level);
```

## EV-18 esp_sleep.h: wake cause
- ไฟล์: `esp_hw_support/include/esp_sleep.h`
```c
typedef enum {
    ESP_SLEEP_WAKEUP_UNDEFINED,  //!< In case of deep sleep, reset was not caused by exit from deep sleep
    ESP_SLEEP_WAKEUP_ALL,        //!< Not a wakeup cause, used to disable all wakeup sources with esp_sleep_disable_wakeup_source
    ESP_SLEEP_WAKEUP_EXT0,       //!< Wakeup caused by external signal using RTC_IO
    ESP_SLEEP_WAKEUP_EXT1,       //!< Wakeup caused by external signal using RTC_CNTL
    ESP_SLEEP_WAKEUP_TIMER,      //!< Wakeup caused by timer
    ESP_SLEEP_WAKEUP_TOUCHPAD,   //!< Wakeup caused by touchpad
    ESP_SLEEP_WAKEUP_ULP,        //!< Wakeup caused by ULP program
    ESP_SLEEP_WAKEUP_GPIO,       //!< Wakeup caused by GPIO (light sleep only on ESP32, S2 and S3)
    ESP_SLEEP_WAKEUP_UART0, (...)
} esp_sleep_source_t;
esp_sleep_wakeup_cause_t esp_sleep_get_wakeup_cause(void) __attribute__((deprecated("use esp_sleep_get_wakeup_causes instead")));
// @return The bitmap of the wakeup sources of the last wakeup from sleep. (deep sleep or light sleep)
uint32_t esp_sleep_get_wakeup_causes(void);
```

## EV-19 esp_sleep.h: gpio wakeup status
```c
// @return bit mask of RTC IO that caused wakeup  (ใต้ #if SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP)
uint64_t esp_sleep_get_gpio_wakeup_status(void);
// @param bit RTC IO bit index (0 to SOC_RTCIO_PIN_COUNT-1).
// @return GPIO number, or GPIO_NUM_NC if bit is invalid or has no corresponding GPIO.
gpio_num_t esp_sleep_wakeup_io_bit2num(uint32_t bit);
```

## EV-20 rtc_io.h / rtc_io.c
- ไฟล์: `esp_driver_gpio/include/driver/rtc_io.h` · `src/rtc_io.c`
```c
// rtc_gpio_hold_en: Enabling HOLD function will cause the pad to latch current values of input enable, output enable, output value, function, drive strength values. This function is useful when going into light or deep sleep mode to prevent the pin configuration from changing.
//   - ESP_ERR_INVALID_ARG The IO is not an RTC IO
// rtc_gpio_hold_dis: Disabling hold function will allow the pad receive the values of input enable, output enable, output value, function, drive strength from RTC_IO peripheral.
#if SOC_RTCIO_HOLD_SUPPORTED && SOC_RTCIO_INPUT_OUTPUT_SUPPORTED
// Helper function to disconnect internal circuits from an RTC IO. This function disables input, output, pullup, pulldown, and enables hold feature for an RTC IO. Use this function if an RTC IO needs to be disconnected from internal circuits in deep sleep, to minimize leakage current.
// - ESP_ERR_INVALID_ARG The IO is not an RTC IO
esp_err_t rtc_gpio_isolate(gpio_num_t gpio_num);
#endif
// rtc_io.c (rtc_gpio_wakeup_enable):
#if !SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED
    if (intr_type == GPIO_INTR_POSEDGE || intr_type == GPIO_INTR_NEGEDGE || intr_type == GPIO_INTR_ANYEDGE) {
        return ESP_ERR_INVALID_ARG; // Dont support this mode.
    }
#endif
```

## EV-21 glitch filter
- ไฟล์: `esp_driver_gpio/include/driver/gpio_filter.h` · `src/gpio_pin_glitch_filter.c`
```c
typedef struct { glitch_filter_clock_source_t clk_src; gpio_num_t gpio_num; } gpio_pin_glitch_filter_config_t;
// @note Pin glitch filter parameters are fixed, pulses shorter than two sample clocks (IO-MUX's source clock) will be filtered out. It's independent with "flex" glitch filter.
// - ESP_ERR_INVALID_ARG: Create a pin glitch filter failed because of invalid arguments (e.g. wrong GPIO number)
// - ESP_ERR_NO_MEM / ESP_FAIL
// gpio_pin_glitch_filter.c:
ESP_GOTO_ON_FALSE(config && ret_filter, ESP_ERR_INVALID_ARG, err, TAG, "invalid argument");
ESP_GOTO_ON_FALSE(GPIO_IS_VALID_GPIO(config->gpio_num), ESP_ERR_INVALID_ARG, err, TAG, "invalid gpio number");
soc_module_clk_t clk_src = config->clk_src ? config->clk_src : GLITCH_FILTER_CLK_SRC_DEFAULT;
// enable:  ESP_RETURN_ON_FALSE(filter->fsm == GLITCH_FILTER_FSM_INIT, ESP_ERR_INVALID_STATE, TAG, "filter not in init state");
//          (CONFIG_PM_ENABLE: esp_pm_lock_acquire) gpio_ll_pin_filter_enable(NULL, filter->gpio_num);
// disable: ESP_RETURN_ON_FALSE(filter->fsm == GLITCH_FILTER_FSM_ENABLE, ESP_ERR_INVALID_STATE, TAG, "filter not in enable state");
```

## EV-22 test_apps ของ esp_driver_gpio (test_gpio.c)
```c
gpio_pin_glitch_filter_config_t config = { .clk_src = GLITCH_FILTER_CLK_SRC_DEFAULT, };
TEST_ESP_OK(gpio_new_pin_glitch_filter(&config, &filter));
TEST_ESP_OK(gpio_glitch_filter_enable(filter));
// can't delete filter if it's not disabled
TEST_ESP_ERR(ESP_ERR_INVALID_STATE, gpio_del_glitch_filter(filter));
// light sleep test:
TEST_ESP_OK(gpio_wakeup_enable(TEST_GPIO_INPUT_LEVEL_LOW_PIN, GPIO_INTR_HIGH_LEVEL));
TEST_ESP_OK(esp_sleep_enable_gpio_wakeup());
vTaskDelay(1000 / portTICK_PERIOD_MS);   // Wait for the complete line to be printed before entering sleep
esp_light_sleep_start();
TEST_ASSERT(esp_sleep_get_wakeup_causes() & BIT(ESP_SLEEP_WAKEUP_GPIO));
```

## EV-23 migration guide v6.x
- ไฟล์: migration guide ของ ESP-IDF v6.1 (GPIO)
```text
- gpio_iomux_in and gpio_iomux_out have been replaced by gpio_iomux_input and gpio_iomux_output, and have been moved to esp_private/gpio.h header file as private APIs for internal use only.
- rom_ prefix has been added to all GPIO ROM APIs in components/esp_rom/esp32xx/include/esp32xx/rom/gpio.h. For example, gpio_iomux_in is now rom_gpio_iomux_in.
- MAX_PAD_GPIO_NUM, MAX_GPIO_NUM, and DIG_IO_HOLD_BIT_SHIFT macros have been removed.
- Added the esp_err_t return type to gpio_uninstall_isr_service.
- gpio_deep_sleep_wakeup_enable -> use gpio_wakeup_enable_on_hp_periph_powerdown_sleep ; gpio_deep_sleep_wakeup_disable -> gpio_wakeup_disable_on_hp_periph_powerdown_sleep
- The deprecated macro GPIO_IS_DEEP_SLEEP_WAKEUP_VALID_GPIO() has been removed. Use GPIO_IS_HP_PERIPH_PD_WAKEUP_VALID_IO() instead.
```
- พิสูจน์ด้วย: ไม่มีข้อความ `gpio_pad_select_gpio` ในเอกสารนี้

## EV-24 GPIO3 และ JTAG source
- ไฟล์: ESP32-S3 Datasheet v2.2 §3 "JTAG Signal Source Control" · TRM v1.8 §8.5 และตาราง 8.1-1
```text
The strapping pin GPIO3 can be used to control the source of JTAG signals during the early boot process. This pin does not have any internal pull resistors and the strapping value must be controlled by the external circuit that cannot be in a high impedance state.
Table 3-5 / 8.5-1 JTAG Signal Source (EFUSE_DIS_PAD_JTAG, EFUSE_DIS_USB_JTAG, EFUSE_STRAP_JTAG_SEL, GPIO3):
  USB Serial/JTAG Controller: (0,0,0,Ignored) [default] ; (0,0,1,GPIO3=1) ; (1,0,Ignored,Ignored)
  JTAG pins (MTDI, MTCK, MTMS, MTDO): (0,0,1,GPIO3=0) ; (0,1,Ignored,Ignored)
  JTAG disabled: (1,1,Ignored,Ignored)
```

## EV-25 โดเมนไฟของ GPIO33-37, 47-48 และ VDD_SPI
- ไฟล์: TRM v1.8 §6 (IO MUX) และ §8.4
```text
Note that GPIO33 ~ GPIO37 and GPIO47 ~ GPIO48 can be powered either by VDD_SPI or VDD3P3_CPU.
GPIO45 is used to select the VDD_SPI power supply voltage at reset:
 - GPIO45 = 0, VDD_SPI pin is powered directly from VDD3P3_RTC via resistor RSPI. Typically this voltage is 3.3 V.
 - GPIO45 = 1, VDD_SPI pin is powered from internal 1.8 V LDO.
This functionality can be overridden by setting eFuse bit EFUSE_VDD_SPI_FORCE to 1, (...)
```

## EV-26 Drive strength และค่า reset (TRM ตาราง IO MUX)
```text
Drive Strength "DRV" column shows the drive strength of each pin after reset:
 GPIO17 and GPIO18: 0 ~5 mA; 1 ~20 mA; 2 ~10 mA; 3 ~40 mA
 Other GPIOs:       0 ~5 mA; 1 ~10 mA; 2 ~20 mA; 3 ~40 mA
 (DRV หลัง reset: GPIO17/18 = 2, GPIO19/20 = 3, ขาอื่นที่เห็นในตาราง = 2)
Reset Configurations "RST": 0 IE=0 | 1 IE=1 | 2 IE=1,WPD=1 | 3 IE=1,WPU=1 | 4 OE=(...)
 (RST ที่เห็น: GPIO4-8=0, GPIO9-14=1, GPIO15/16=0, GPIO17/18=1, GPIO19-21=0, GPIO26-29=3, GPIO44=3, GPIO45/46=2, GPIO47/48=1)
```
- ไฟล์ Datasheet note 5 (ค่า default 10 / 40 / 20 mA): ดึงรอบ 2-3

## EV-27 gpio_types.h
- ไฟล์: `esp_hal_gpio/include/hal/gpio_types.h`
```c
GPIO_MODE_DEF_INPUT (BIT0) / GPIO_MODE_DEF_OUTPUT (BIT1) / GPIO_MODE_DEF_OD (BIT2)
GPIO_PULLUP_DISABLE = 0x0, GPIO_PULLUP_ENABLE = 0x1 ; GPIO_PULLDOWN_DISABLE = 0x0, GPIO_PULLDOWN_ENABLE = 0x1
typedef enum { GPIO_PULLUP_ONLY, GPIO_PULLDOWN_ONLY, GPIO_PULLUP_PULLDOWN, GPIO_FLOATING } gpio_pull_mode_t;
GPIO_DRIVE_CAP_0 = 0 /*weak*/, GPIO_DRIVE_CAP_1 = 1 /*stronger*/, GPIO_DRIVE_CAP_2 = 2 /*medium*/, GPIO_DRIVE_CAP_DEFAULT = 2, GPIO_DRIVE_CAP_3 = 3 (...)
typedef struct { gpio_drive_cap_t drv; uint32_t fun_sel : 8; uint32_t sig_out : 16; uint32_t pu : 1; uint32_t pd : 1; uint32_t ie : 1; uint32_t oe : 1; uint32_t oe_ctrl_by_periph : 1; uint32_t oe_inv : 1; uint32_t od : 1; uint32_t slp_sel : 1; (...) } gpio_io_config_t;
```

## EV-28 gpio.rst (docs): glitch filter และการใช้ขาร่วม
- ไฟล์: `docs/en/api-reference/peripherals/gpio.rst` (ฉบับที่อยู่ใน RAG ยังไม่ยืนยันว่าตรงแท็ก v6.1)
```text
Each GPIO can be configured with a glitch filter, which can be used to filter out pulses shorter than two sample clock cycles. The duration of the filter is not configurable. The sample clock is the clock source of the IO_MUX. In the driver, we call this kind of filter as pin glitch filter.
gpio_config is an API that overwrites all the current configurations, so it must be called to set the pin mode to GPIO_MODE_INPUT before calling the LEDC driver API which connects the output signal to the pin. As an alternative, if no other configuration is needed other than making the pin input enabled, gpio_input_enable can be the one to call at any time.
```

## EV-29 หลักฐานเชิงลบ: gpio_pad_select_gpio
- ค้น RAG ด้วยชื่อ `gpio_pad_select_gpio` และ `esp_rom_gpio_pad_select_gpio` ไม่พบ declaration หรือการใช้ใน esp_driver_gpio, esp_hal_gpio, soc หรือ migration guide ที่อยู่ใน RAG
- ข้อความ "deprecated v5.0, removed v6" พบเฉพาะในเอกสาร GLM (`gpio.md`, `projects_skills.docx`)
- ข้อจำกัด: `esp_rom` ไม่อยู่ใน RAG จึงยืนยันไม่ได้ว่า ROM header มีหรือไม่ → ปิดด้วย `tools/verify_gpio_unv.sh` V1, V5

## EV-30 TRM: Digital hold
- ไฟล์: TRM v1.8 บทว่าด้วยการคง GPIO ใน Deep-sleep
```text
each digital pin is controlled by the result of OR operation of the pin's Hold enable signal and the global Hold enable signal.
 - RTC_CNTL_DIG_PAD_HOLD_REG[n], controls the Hold signal of each pin of GPIO26 ~ GPIO48.
 - RTC_CNTL_DG_PAD_FORCE_HOLD, controls the global Hold signal of all digital pins.
To maintain the pin's input/output status in Deep-sleep, set RTC_CNTL_DIG_PAD_HOLD_REG[n] to hold the value of each digital pin, n = 5 ~ 27, corresponding to GPIO26 ~ (...)
```

## EV-31 README ของ esp_driver_gpio
- ไฟล์: `esp_driver_gpio/README.md`
```text
# GPIO API usage in Peripheral Drivers
When a peripheral driver initializes IOs, the general rule is to append necessary configurations onto the IO, instead of using any GPIO APIs that overwrite the current IO setting, such as `gpio_config`, `gpio_reset_pin`, `gpio_set_direction`, `gpio_set_pull_mode`. Use `gpio_pullup_en` and `gpio_pulldown_en` to enable the internal pull-up/pull-down resistors if necessary. Use `gpio_od_enable` to configure an output IO as open-drain mode if necessary.
```

## EV-32 ผล verify_gpio_unv.sh (รันบนเครื่องผู้ใช้ รอบ 5)
- แหล่ง: `verify.log` ของผู้ใช้ (IDF ที่ `~/esp/esp-idf`) ผู้ช่วยอ่านผ่านตัวดึงเอกสาร (ข้อความสรุป ไม่ใช่ log ดิบทั้งไฟล์) path ย่อให้เป็น relative ต่อ `$IDF_PATH`
- ข้อจำกัด: สคริปต์เวอร์ชัน 1 ใช้ `head -20` ใน V1 จึงถูกตัดผล
```text
V0  git describe --tags -> v6.1
V1  grep -rn gpio_pad_select_gpio components docs | head -20  (ผลถูกตัดที่ 20 บรรทัด)
    components/bootloader_support/src/bootloader_common.c:39: esp_rom_gpio_pad_select_gpio(num_pin);
    components/esp_driver_sdmmc/src/sd_host_sdmmc.c:1412: esp_rom_gpio_pad_select_gpio(io_config->cd_io);
    components/esp_driver_sdmmc/src/sd_host_sdmmc.c:1425: esp_rom_gpio_pad_select_gpio(io_config->wp_io);
    components/esp_hw_support/port/esp32/rtc_clk.c:135: esp_rom_gpio_pad_select_gpio(XTAL32K_P_GPIO_NUM);
    components/esp_hw_support/port/esp32/rtc_clk.c:136: esp_rom_gpio_pad_select_gpio(XTAL32K_N_GPIO_NUM);
    components/esp_hw_support/test_apps/rtc_clk/main/test_rtc_clk.c:198-199: esp_rom_gpio_pad_select_gpio(pin_xtal32k_p / pin_xtal32k_n);
    + ประกาศใน .h และ .ld ของ esp32, esp32c2, esp32c3, esp32c5, esp32c6 (ไม่ทราบไฟล์/บรรทัด)
V2  soc_caps.h ของ esp32s3 (ค้น 9 pattern พบ 3):
    47:#define SOC_DEDICATED_GPIO_SUPPORTED 1
    187:#define SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER 1
    190:#define SOC_GPIO_SUPPORT_FORCE_HOLD (1)
    (ไม่พบ: SOC_GPIO_SUPPORT_ETM, SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN, SOC_GPIO_FLEX_GLITCH_FILTER_NUM,
     SOC_GPIO_SUPPORT_PIN_HYS*, SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP, SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED)
V3  sdkconfig: CONFIG_SOC_DEDICATED_GPIO_SUPPORTED=y · CONFIG_SOC_GPIO_PIN_COUNT=49
    CONFIG_SOC_GPIO_SUPPORT_PIN_GLITCH_FILTER=y · CONFIG_SOC_GPIO_SUPPORT_FORCE_HOLD=y
    CONFIG_SOC_GPIO_VALID_GPIO_MASK=0x1FFFFFFFFFFFF · CONFIG_SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP=y
    CONFIG_SOC_RTCIO_HOLD_SUPPORTED=y
V4  gpio_wakeup_disable (สรุป): เช็ค SOC_RTCIO_WAKE_SUPPORTED แล้วเรียก rtc_gpio_wakeup_disable;
    เรียก gpio_hal_wakeup_disable และ gpio_hal_sleep_sel_en  (ในสรุปไม่เห็น gpio_hal_set_intr_type)
V5  esp_rom_gpio.h:
    void esp_rom_gpio_pad_select_gpio(uint32_t iopad_num);
    void esp_rom_gpio_pad_pullup_only(uint32_t iopad_num);
    void esp_rom_gpio_pad_unhold(uint32_t gpio_num);
    void esp_rom_gpio_pad_set_drv(uint32_t iopad_num, uint32_t drv);
    void esp_rom_gpio_connect_in_signal(uint32_t gpio_num, uint32_t signal_idx, bool inv);
    void esp_rom_gpio_connect_out_signal(uint32_t gpio_num, uint32_t signal_idx, bool out_inv, bool oen_inv);
V6  esp_sleep.h (สรุป): ESP_EXT1_WAKEUP_ALL_LOW และ ESP_PD_DOMAIN_RTC8M เป็น deprecated;
    esp_sleep_get_ext1_wakeup_status ระบุว่า "deprecated บางส่วน" (กำกวม ต้องดู raw)
V7  gpio_new_pin_glitch_filter: จองด้วย heap_caps_calloc, ตั้ง clk_src, ตั้ง pm_lock เมื่อ CONFIG_PM_ENABLE
V8  GPIO_HOLD_MASK ใช้ใน gpio.c และ gpio_ll.h ของ esp32s3, esp32, c2, c3, c5, c6, c61, h2, h21 (ไม่แสดงค่า mask)
```
- ข้อสรุปที่ใช้ได้: ใช้แทน EV-29 ว่า `esp_rom_gpio_pad_select_gpio` **มี** ใน v6.1; ชื่อเปล่า `gpio_pad_select_gpio` ยังไม่ยืนยัน (รอ `verify_gpio_unv2.sh` W1)

## EV-33 ผล verify_gpio_unv2.sh (รันบนเครื่องผู้ใช้ รอบ 5 อ่านดิบ ไม่ใช่สรุป)
- แหล่ง: `verify2.log` ของผู้ใช้ (IDF ที่ `~/esp/esp-idf`) อ่านด้วยการสั่งคัดลอกตรงตัวอักษร path ย่อเป็น relative ต่อ `$IDF_PATH`
```text
W0  git describe --tags -> v6.1
W1  grep -rnw gpio_pad_select_gpio components docs  (ไม่ตัดผล) ได้บรรทัดเดียว:
    components/esp_rom/hints.yml:2:    re: "(error: implicit declaration of function|undefined reference to) '(gpio_pad_set_drv|gpio_pad_pullup|gpio_pad_select_gpio|gpio_pad_unhold|gpio_matrix_in|gpio_matrix_out)'"
    (ไม่มีไฟล์ .c/.h/.rst ที่ประกาศหรือเรียกชื่อเปล่านี้)
W2  components/esp_rom/include/esp_rom_gpio.h:24: void esp_rom_gpio_pad_select_gpio(uint32_t iopad_num);
W3  esp_err_t gpio_wakeup_disable(gpio_num_t gpio_num)
    {   GPIO_CHECK(GPIO_IS_VALID_GPIO(gpio_num), "GPIO number error", ESP_ERR_INVALID_ARG);
        esp_err_t ret = ESP_OK;
    #if SOC_RTCIO_WAKE_SUPPORTED
        if (rtc_gpio_is_valid_gpio(gpio_num)) { ret = rtc_gpio_wakeup_disable(gpio_num);
    #if SOC_LP_IO_CLOCK_IS_INDEPENDENT
            io_mux_enable_lp_io_clock(gpio_num, false);
    #endif } #endif
        portENTER_CRITICAL(&gpio_context.gpio_spinlock);
        gpio_hal_wakeup_disable(gpio_context.gpio_hal, gpio_num);
    #if CONFIG_ESP_SLEEP_GPIO_RESET_WORKAROUND || CONFIG_PM_SLP_DISABLE_GPIO
        gpio_hal_sleep_sel_en(gpio_context.gpio_hal, gpio_num);
    #endif
        portEXIT_CRITICAL(&gpio_context.gpio_spinlock);
        return ret; }
    gpio_hal_set_intr_type ถูกเรียกที่ gpio.c บรรทัด 169 และ 701 เท่านั้น
W4  gpio_ll.h (esp32s3): SET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);   (hold_en, บรรทัด 473)
                         CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]); (hold_dis, บรรทัด 484)
    gpio.c: บรรทัด 786, 806: } else if (GPIO_HOLD_MASK[gpio_num]) {
    (ไม่เห็นนิยามอาร์เรย์ GPIO_HOLD_MASK ในผล)
W5  esp_sleep.h __attribute__((deprecated: บรรทัด 38 (ESP_EXT1_WAKEUP_ALL_LOW), 466 (ข้อความ "please use 'esp_sleep_enable_ext1_wakeup_io' and 'esp_sleep_disable_ext1_wakeup_io' instead"), 778 (esp_sleep_get_wakeup_cause: "use esp_sleep_get_wakeup_causes instead")
    บรรทัด 616-619: /** ... @return bit mask, if GPIOn caused wakeup, BIT(n) will be set */ uint64_t esp_sleep_get_ext1_wakeup_status(void);   (ไม่มี attribute)
```

-  ข้อสรุป: ชื่อเปล่า `gpio_pad_select_gpio` ไม่ถูกประกาศใน v6.1 (ผล W1 ไม่ถูกตัด) · `gpio_wakeup_disable` ไม่คืน interrupt type · `esp_sleep_get_ext1_wakeup_status` ไม่มี attribute deprecated · **แก้ไข:** ข้อความเดิมที่ว่า `ESP_PD_DOMAIN_RTC8M` ไม่ใช่ deprecated เป็นความผิดของผู้ช่วย W5 ค้นเฉพาะ `__attribute__((deprecated` จึงไม่เห็น `_Pragma` ของมัน ดู EV-36 ว่าเป็น deprecated จริง

- ข้อมูลเพิ่มจาก RAG (ไม่ใช่ log): `gpio_ll_wakeup_disable` ฝั่ง digital ใน `esp_hal_gpio` ทำ `hw->pin[gpio_num].wakeup_enable = 0;` อย่างเดียว ส่วน `rtcio_ll_wakeup_disable` ตั้ง `RTCIO.pin[n].wakeup_enable = 0; RTCIO.pin[n].int_type = 0;` (chunk ไม่ระบุชิป [INF]) · TRM: `RTC_CNTL_RTC_PAD_HOLD_REG[n]` (n = 0 ~ 21) คุม GPIO0-21, `RTC_CNTL_DIG_PAD_HOLD_REG` คุม GPIO26-48

## EV-34 ผล verify3 (X1-X3, รันบนเครื่องผู้ใช้ รอบ 5c อ่านดิบด้วยการสั่งคัดลอกตรงตัวอักษร)
- แหล่ง: `verify3.log` ของผู้ใช้ (IDF ที่ `~/esp/esp-idf`, v6.1) path ย่อเป็น relative ต่อ `$IDF_PATH` สคริปต์เทียบเท่าคือ `tools/verify_gpio_unv3.sh`
```text
X1  components/esp_rom/hints.yml (บรรทัด 1-14):
    re: "(error: implicit declaration of function|undefined reference to) '(gpio_pad_set_drv|gpio_pad_pullup|gpio_pad_select_gpio|gpio_pad_unhold|gpio_matrix_in|gpio_matrix_out)'"
    hint: "The rom GPIO functions have been renamed. Please replace with the APIs in 'esp_rom_gpio.h'."
    (รายการถัดไปในไฟล์เป็นเรื่อง esp_rom_tjpgd.h ไม่เกี่ยวกับ GPIO)
X2  components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:380:
    static inline void gpio_ll_wakeup_disable(gpio_dev_t *hw, uint32_t gpio_num)
    {
        hw->pin[gpio_num].wakeup_enable = 0;
    }
X3  components/soc/esp32s3/gpio_periph.c:64:  const uint32_t GPIO_HOLD_MASK[] = {
    บรรทัด 65-86 (22 รายการ, ดัชนี 0-21): 0,
    บรรทัด 87: BIT(1),   (ดัชนี 22)
    บรรทัด 88: BIT(2),   (ดัชนี 23)
    บรรทัด 89: BIT(3),   (ดัชนี 24)
    บรรทัด 90: BIT(4),   (ดัชนี 25)
    บรรทัด 91: BIT(5),   (ดัชนี 26)
    บรรทัด 92: BIT(6),   (ดัชนี 27)
    บรรทัด 93: BIT(7),   (ดัชนี 28)
    บรรทัด 94: BIT(8),   (ดัชนี 29)
    (ผลถูกตัดโดย grep -A30 ที่ดัชนี 29)
```
- ข้อสรุป: (1) IDF ถือว่ากลุ่มชื่อ ROM GPIO เก่า "ถูกเปลี่ยนชื่อ" ไปอยู่ใน `esp_rom_gpio.h` (2) `gpio_wakeup_disable` ไม่คืน interrupt type (ร่วมกับ EV-33 W3) (3) ดัชนี 26 = `BIT(5)` ตรงกับ TRM ดัชนี 30 ขึ้นไปยังไม่เห็น คาดว่า GPIO38 = `BIT(17)` [INF] (ข้อ 3 ถูกแทนด้วย EV-35)

## EV-35 ผล X4 อาร์เรย์ GPIO_HOLD_MASK ฉบับเต็ม (รันบนเครื่องผู้ใช้ รอบ 5d อ่านดิบ)
- แหล่ง: `verify_gpio_unv3.sh_verify.log_.md` (สร้างด้วย `tools/run_verify.sh` หัวไฟล์: IDF = v6.1, IDF_PATH = /root/esp/esp-idf) คำสั่ง `sed -n '64,130p' components/soc/esp32s3/gpio_periph.c`
```text
const uint32_t GPIO_HOLD_MASK[] = {
    0,  (x22 รายการ: ดัชนี 0-21)
    BIT(1),  BIT(2),  BIT(3),  BIT(4),  BIT(5),  BIT(6),  BIT(7),  BIT(8),  BIT(9),  BIT(10),
    BIT(11), BIT(12), BIT(13), BIT(14), BIT(15), BIT(16), BIT(17), BIT(18), BIT(19), BIT(20),
    BIT(21), BIT(22), BIT(23), BIT(24), BIT(25), BIT(26), BIT(27),     (ดัชนี 22-48 รวม 27 รายการ)
};
_Static_assert(sizeof(GPIO_HOLD_MASK) == SOC_GPIO_PIN_COUNT * sizeof(uint32_t), "Invalid size of GPIO_HOLD_MASK");
```
- หมายเหตุ: บล็อกด้านบนย่อ "0," 22 บรรทัดและวางค่า BIT หลายค่าต่อบรรทัดเพื่อความกะทัดรัด ผลดิบเป็นหนึ่งค่าต่อบรรทัดและมีรายการครบ 22 + 27 = 49
- ข้อสรุป: ดัชนี i ≥ 22 → `BIT(i-21)` ดังนั้น GPIO26 = `BIT(5)`, GPIO38 = `BIT(17)`, GPIO48 = `BIT(27)` ตรงกับ TRM (EV-30) และขนาดอาร์เรย์ = 49 ตาม `_Static_assert`




- ข้อสรุป: ชื่อเปล่า `gpio_pad_select_gpio` ไม่ถูกประกาศใน v6.1 (ผล W1 ไม่ถูกตัด) · `gpio_wakeup_disable` ไม่คืน interrupt type · `esp_sleep_get_ext1_wakeup_status` ไม่มี attribute deprecated · **แก้ไข:** ข้อความเดิมที่ว่า `ESP_PD_DOMAIN_RTC8M` ไม่ใช่ deprecated เป็นความผิดของผู้ช่วย W5 ค้นเฉพาะ `__attribute__((deprecated` จึงไม่เห็น `_Pragma` ของมัน ดู EV-36 ว่าเป็น deprecated จริง

## EV-36 ผล V6 ของ verify_gpio_unv.sh (รอบ 5 อ่านดิบจากไฟล์ที่ผู้ใช้บันทึก)
- แหล่ง: `verify_gpio_unv.sh_verify.log_.md` คำสั่ง `grep -n "deprecated\|esp_sleep_get_ext1_wakeup_status" components/esp_hw_support/include/esp_sleep.h | head -30` (ได้ 6 บรรทัด ไม่ถึง 30 จึงไม่ถูกตัด)
```text
38:    ESP_EXT1_WAKEUP_ALL_LOW __attribute__((deprecated("wakeup mode \"ALL_LOW\" is no longer supported after ESP32, \
99:#define ESP_PD_DOMAIN_RTC8M _Pragma("GCC warning \"'ESP_PD_DOMAIN_RTC8M' enum is deprecated\"") ESP_PD_DOMAIN_RC_FAST
333: * @note This function will be deprecated in release/v6.0. Please switch to use `esp_sleep_enable_ext1_wakeup_io` and `esp_sleep_disable_ext1_wakeup_io`
466:__attribute__((deprecated("please use 'esp_sleep_enable_ext1_wakeup_io' and 'esp_sleep_disable_ext1_wakeup_io' instead")));
619:uint64_t esp_sleep_get_ext1_wakeup_status(void);
778:__attribute__((deprecated("use esp_sleep_get_wakeup_causes instead")));
```
- ข้อสรุป (สามระดับ ตาม PROTOCOL B.5 ข้อ 14): (1) deprecated ด้วย attribute: `ESP_EXT1_WAKEUP_ALL_LOW` (38), `esp_sleep_enable_ext1_wakeup_with_level_mask` (466), `esp_sleep_get_wakeup_cause` (778) (2) deprecated ด้วย `_Pragma` warning: `ESP_PD_DOMAIN_RTC8M` (99, แทนด้วย `ESP_PD_DOMAIN_RC_FAST`) (3) มีแค่หมายเหตุ "will be deprecated in release/v6.0": `esp_sleep_enable_ext1_wakeup` (333) (4) ไม่มี deprecated: `esp_sleep_get_ext1_wakeup_status` (619)


## EV-37 ผล verify_gpio_unv4.sh (รันบนเครื่องผู้ใช้ รอบ 5f อ่านดิบ: เห็นทั้งโค้ดและผล)
- แหล่ง: `verify4.log.txt` ของผู้ใช้ (IDF v6.1) path ย่อเป็น relative ต่อ `$IDF_PATH`
- หมายเหตุ: ผล Y5 ที่อ่านได้มีตัวอักษร `` ` `` แทรกในคำ `portEXIT_CRITICAL` ถือเป็นความคลาดเคลื่อนจากการคัดลอกหรือแปลงไฟล์ ไม่ใช่ซอร์ส IDF บล็อกด้านล่างเป็นฉบับแก้แล้ว
```text
Y0  v6.1
Y1  components/esp_driver_gpio/include/driver/gpio.h:364:  * size limit (configurable as "ISR stack size" in menuconfig). This
    components/freertos/Kconfig:426:        config FREERTOS_ISR_STACKSIZE
    components/freertos/Kconfig:427:            int "ISR stack size"
Y2  components/esp_hw_support/include/esp_sleep.h:209: *        current slow clock source selection (Refer RTC clock source configuration in menuconfig).
    components/esp_hw_support/include/esp_sleep.h:210: *        Valid values should be positive values less than RTC slow clock period * (2 ^ RTC timer bitwidth).
    (บรรทัด 207-212 อยู่ในคำอธิบาย esp_sleep_enable_timer_wakeup ไม่มีข้อความเรื่องพัลส์ของ GPIO wakeup และไม่มีผลจาก gpio.h)
Y3  components/esp_rom/esp32s3/ld/esp32s3.rom.api.ld:
    11: PROVIDE ( esp_rom_gpio_pad_select_gpio    = rom_gpio_pad_select_gpio );
    12: PROVIDE ( esp_rom_gpio_pad_pullup_only    = rom_gpio_pad_pullup );
    13: PROVIDE ( esp_rom_gpio_pad_set_drv        = rom_gpio_pad_set_drv );
    14: PROVIDE ( esp_rom_gpio_pad_unhold         = rom_gpio_pad_unhold );
    15: PROVIDE ( esp_rom_gpio_connect_in_signal  = rom_gpio_matrix_in );
    16: PROVIDE ( esp_rom_gpio_connect_out_signal = rom_gpio_matrix_out );
    components/esp_rom/esp32s3/ld/esp32s3.rom.ld:
    458: rom_gpio_matrix_in = 0x40001a94;     459: rom_gpio_matrix_out = 0x40001aa0;
    467: rom_gpio_pad_pullup = 0x40001b00;   468: rom_gpio_pad_select_gpio = 0x40001b0c;
    469: rom_gpio_pad_set_drv = 0x40001b18;  470: rom_gpio_pad_unhold = 0x40001b24;
Y4  components/esp_rom/include/esp_rom_gpio.h (ย่อเฉพาะคำอธิบายและ signature):
    void esp_rom_gpio_pad_select_gpio(uint32_t iopad_num);
       "Configure IO Pad as General Purpose IO, so that it can be connected to internal Matrix, then combined with one or more peripheral signals."
    void esp_rom_gpio_pad_pullup_only(uint32_t iopad_num);   "Enable internal pull up, and disable internal pull down."
    void esp_rom_gpio_pad_unhold(uint32_t gpio_num);   "Unhold the IO Pad. @note When the Pad is set to hold, the state is latched at that moment and won't get changed."
    void esp_rom_gpio_pad_set_drv(uint32_t iopad_num, uint32_t drv);   "0: 5mA  1: 10mA  2: 20mA  3: 40mA"
    void esp_rom_gpio_connect_in_signal(uint32_t gpio_num, uint32_t signal_idx, bool inv);
       "There's no limitation on the number of signals that a GPIO can combine with." GPIO_MATRIX_CONST_ZERO_INPUT / GPIO_MATRIX_CONST_ONE_INPUT = ต่อ logic 0 / 1 เข้า signal
    void esp_rom_gpio_connect_out_signal(uint32_t gpio_num, uint32_t signal_idx, bool out_inv, bool oen_inv);
       "Internally, the signal will be connected first, then output will be enabled on the pad." SIG_GPIO_OUT_IDX = disconnect GPIO and other peripherals. Only the GPIO driver can control the output level.
Y5  void gpio_deep_sleep_hold_en(void)
    {
        portENTER_CRITICAL(&gpio_context.gpio_spinlock);
        RTC_CNTL_ATOMIC() {
            gpio_hal_deep_sleep_hold_en(gpio_context.gpio_hal);
        }
        portEXIT_CRITICAL(&gpio_context.gpio_spinlock);
    }
Y6  esp_err_t rtc_gpio_hold_en(gpio_num_t gpio_num)
    {
        ESP_RETURN_ON_FALSE(rtc_gpio_is_valid_gpio(gpio_num), ESP_ERR_INVALID_ARG, RTCIO_TAG, "RTCIO number error");
        RTCIO_ENTER_CRITICAL();
        rtcio_hal_hold_enable(rtc_io_number_get(gpio_num));
        RTCIO_EXIT_CRITICAL();
        return ESP_OK;
    }
Y7  (ว่าง: grep ใน components/esp_hal_gpio/esp32s3 ไม่พบ rtcio_ll_hold_en ซึ่งเป็น path ที่ผู้ช่วยเดาผิด ไม่ใช่หลักฐานว่าไม่มี)
Y8  components/esp_hw_support/debug_probe/debug_probe.c:57:        ESP_RETURN_ON_ERROR(rtc_gpio_init(pin), TAG, "init RTC GPIO %d failed", pin);
    components/esp_hw_support/sleep_gpio.c:286:    if (!gpio_hal_deep_sleep_hold_is_en(&gpio_hal)) {
```
- ข้อมูลเพิ่มจาก RAG (ไม่ใช่ log): `gpio_ll_deep_sleep_hold_en` ใน `esp_hal_gpio` ทำ `CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_FORCE_UNHOLD); SET_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_AUTOHOLD_EN_M);` (chunk ไม่ระบุชิป [INF])
- ข้อสรุป: (1) "ISR stack size" ใน menuconfig = `FREERTOS_ISR_STACKSIZE` (2) ไม่พบข้อความ "3 รอบ slow clock" ใน esp_sleep.h และ gpio.h (ภายหลังพบใน docs ดู EV-38 Z5) (3) ชื่อ ROM ใหม่ชี้ไปที่ `rom_gpio_*` ครบ 6 คู่ (GPIO-30) (4) RTC GPIO hold ใช้ `rtcio_hal_hold_enable` ส่วน `gpio_deep_sleep_hold_en` ใช้ digital auto-hold คนละกลไก (INF) (5) บริบทของ `sleep_gpio.c:286` ดู EV-38 Z1

## EV-38 ผล verify_gpio_unv5.sh (รันบนเครื่องผู้ใช้ รอบ 5g อ่านดิบ: คัดลอกตรงตัวอักษร)
- แหล่ง: `verify5.log.txt` ของผู้ใช้ (IDF v6.1) path ย่อเป็น relative ต่อ `$IDF_PATH` และตัดคำนำหน้า path ยาวของบรรทัด Z4 ออก
- ข้อจำกัด: Z1 แสดงเฉพาะบรรทัด 260-320 ของ `sleep_gpio.c` ส่วนบรรทัด 311-353 และ 360-394 ยังไม่เห็นครบ
```text
Z0  v6.1
Z1  components/esp_hw_support/sleep_gpio.c (บรรทัด 260-320 เริ่มกลางฟังก์ชัน esp_sleep_restore_isolated_digital_gpio):
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
        if (do_backup) { s_gpio_isolate_backup.backuped = 0; ... pu = pd = ie = oe = 0; }
        uint64_t hold_mask = gpio_ll_get_digital_gpio_hold_mask(gpio_hal.dev);
        gpio_io_config_t isolate_config = { .pu = 0, .pd = 0, .ie = 0, .oe = 0, .fun_sel = PIN_FUNC_GPIO };
        while (pad_mask) {
            gpio_num_t gpio_num = (gpio_num_t)__builtin_ctzll(pad_mask);
            if (!(hold_mask & (1ULL << gpio_num)) &&
                !esp_gpio_is_reserved(BIT64(gpio_num))) {
                if (do_backup) {
                    gpio_io_config_t io_config;
    (บรรทัด 260-320 ตัดที่นี่)
    --- บรรทัดที่มีคำว่า hold ใน sleep_gpio.c:
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
Z2  components/esp_hal_gpio/esp32s3/include/hal/rtc_io_ll.h
    components/esp_hal_gpio/esp32s31/include/hal/rtc_io_ll.h
Z3  (ว่าง: ชื่อ rtcio_ll_hold_enable ที่ผู้ช่วยเดาไม่พบ ไม่ใช่หลักฐานว่าไม่มีฟังก์ชัน hold)
Z4  components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:
    424: static inline void _gpio_ll_deep_sleep_hold_en(gpio_dev_t *hw)
    426:     CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_FORCE_UNHOLD);
    427:     SET_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_AUTOHOLD_EN_M);
    430-432: /// use a macro to wrap the function, force the caller to use it in a critical section ... PERIPH_RCC_ATOMIC() must be used to ensure atomicity.
    433: #define gpio_ll_deep_sleep_hold_en(...) (void)__DECLARE_RCC_ATOMIC_ENV; _gpio_ll_deep_sleep_hold_en(__VA_ARGS__)
    440: static inline void _gpio_ll_deep_sleep_hold_dis(gpio_dev_t *hw)
    442:     CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_AUTOHOLD_EN_M);
    460: static inline bool gpio_ll_deep_sleep_hold_is_en(gpio_dev_t *hw)
    462:     return !GET_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_FORCE_UNHOLD) && GET_PERI_REG_MASK(RTC_CNTL_DIG_ISO_REG, RTC_CNTL_DG_PAD_AUTOHOLD_EN_M);
Z5  docs/en/api-reference/system/sleep_modes.rst:392:        Any GPIO/IO wakeup signal -- whether level or edge -- must be held (or have a pulse width) of at least 3 RTC slow-clock cycles to be reliably sampled by the wakeup logic. The duration of one slow-clock cycle depends on :ref:`CONFIG_RTC_CLK_SRC` (e.g. RC_SLOW @ ~136 kHz ~= 7.4 us/cycle, XTAL32K @ 32.768 kHz ~= 30.5 us/cycle). This applies to both :cpp:func:`esp_sleep_enable_gpio_wakeup` and :cpp:func:`esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown`.
Z6  components/freertos/Kconfig:
    426:        config FREERTOS_ISR_STACKSIZE
    427:            int "ISR stack size"
    428:            range 2096 32768 if ESP_COREDUMP_ENABLE
    429:            default 2096 if ESP_COREDUMP_ENABLE
    430:            range 1536 32768
    431:            default 1536
    432:            help
    433:                The interrupt handlers have their own stack. The size of the stack can be defined here. Each processor
    434:                has its own stack, so the total size occupied will be twice this.
```
- หมายเหตุเรื่องการย่อ: บล็อก Z1 ด้านบนรวมเนื้อหาจากสองส่วนของผลดิบ (ตัวฟังก์ชันบรรทัด 284-320 และรายการบรรทัดที่มีคำว่า hold) บรรทัดโค้ดที่ตัดด้วย `...` ในส่วน backup เป็นการย่อโดยผู้ช่วยเพื่อความกะทัดรัด ไม่ใช่ซอร์สต้นฉบับ ถ้าต้องใช้ข้อความเต็มให้ดู Z8 ใน `verify_gpio_unv6.sh`
- ข้อสรุป: (1) ข้อความของ GLM เรื่อง 3 รอบ slow clock **ถูก** (docs v6.1) (2) `FREERTOS_ISR_STACKSIZE` default 1536 ช่วง 1536-32768 (3) deep-sleep hold เปิด → ต้อง stack internal RAM (assert) (4) `gpio_deep_sleep_hold_en` = digital auto-hold ใน `RTC_CNTL_DIG_ISO_REG` และลูป isolate เริ่มจาก mask ของ GPIO26-48 จึงไม่ครอบคลุมขา RTC 0-21 (INF) (5) ชื่อฟังก์ชัน LL ของ hold ใน `rtc_io_ll.h` ยังไม่เห็น (รอ Z9) (ข้อ 4 และ 5 ถูกแทนด้วย EV-39)

## EV-39 ผล verify_gpio_unv6.sh (รันบนเครื่องผู้ใช้ รอบ 5h อ่านดิบ: คัดลอกตรงตัวอักษร)
- แหล่ง: `verify6.log.txt` ของผู้ใช้ (IDF v6.1 ที่ `/root/esp/esp-idf`) path ย่อเป็น relative ต่อ `$IDF_PATH` ไฟล์นี้มีเฉพาะผล ไม่มีโค้ดสคริปต์ (โค้ดอยู่ใน `tools/verify_gpio_unv6.sh`)
- การย่อ: เครื่องหมาย `(...)` คือส่วนที่ผู้ช่วยย่อเพื่อความกะทัดรัด ไม่ใช่ซอร์สต้นฉบับ
```text
Z7  -rw-r--r--. 1 root root 25529 Oct 12 11:51 components/soc/esp32s3/include/soc/soc_caps.h
    (grep -rn SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD ใน components/soc/esp32s3 และ components/esp_hal_gpio/esp32s3: ไม่มีบรรทัดผลลัพธ์)
Z8  sleep_gpio.c (บรรทัด 197-):
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
    (ท้ายของ esp_sleep_isolate_digital_gpio สาขา #else)
        (void)do_backup;
        while (pad_mask) {
            gpio_num_t gpio_num = (gpio_num_t)__builtin_ctzll(pad_mask);
            if (!gpio_hal_is_digital_io_hold(&gpio_hal, gpio_num) &&
                !esp_gpio_is_reserved(BIT64(gpio_num))) {
                sleep_gpio_isolate_one_pin(gpio_num);
            }
            pad_mask &= pad_mask - 1;
        }
    static void esp_deep_sleep_wakeup_io_reset(void)   (ใต้ #if SOC_DEEP_SLEEP_SUPPORTED)
    {
    #if SOC_PM_SUPPORT_EXT1_WAKEUP
        uint32_t rtc_io_mask = rtc_hal_ext1_get_wakeup_pins();
        // Disable ext1 wakeup before releasing hold, such that wakeup status can reflect the correct wakeup pin
        rtc_hal_ext1_clear_wakeup_pins();
        for (int gpio_num = 0; gpio_num < SOC_GPIO_PIN_COUNT && rtc_io_mask != 0; ++gpio_num) {
            int rtcio_num = rtc_io_num_map[gpio_num];
            if ((rtc_io_mask & BIT(rtcio_num)) == 0) { continue; }
    #if SOC_RTCIO_HOLD_SUPPORTED
            rtcio_hal_hold_disable(rtcio_num);
    #endif
            rtc_io_mask &= ~BIT(rtcio_num);
        }
    #endif
    #if SOC_GPIO_SUPPORT_HP_PERIPH_PD_SLEEP_WAKEUP
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
    #endif
    #if CONFIG_ESP_SLEEP_GPIO_RESET_WORKAROUND || CONFIG_PM_SLP_DISABLE_GPIO
        // Configure to isolate (disable the Input/Output/Pullup/Pulldown function of the pin) all GPIO pins in sleep state
        esp_sleep_config_gpio_isolate();
        // Enable automatic switching of GPIO configuration
        esp_sleep_enable_gpio_switch(true);
    #endif
        return ESP_OK;
    }
Z9  components/esp_hal_gpio/esp32s3/include/hal/rtc_io_ll.h:
    289: * Enable force hold function on an RTC IO pad.
    298: static inline void rtcio_ll_force_hold_enable(int rtcio_num)
    300:     SET_PERI_REG_MASK(RTC_CNTL_PAD_HOLD_REG, rtc_io_desc[rtcio_num].hold_force);
    306: * @note If disable the pad hold, the status of pad maybe changed in sleep mode.
    309: static inline void rtcio_ll_force_hold_disable(int rtcio_num)
    311:     CLEAR_PERI_REG_MASK(RTC_CNTL_PAD_HOLD_REG, rtc_io_desc[rtcio_num].hold_force);
    324: static inline void rtcio_ll_force_hold_all(void)
    326:     SET_PERI_REG_MASK(RTC_CNTL_PWC_REG, RTC_CNTL_PAD_FORCE_HOLD_M);
    335: static inline void rtcio_ll_force_unhold_all(void)
    337:     CLEAR_PERI_REG_MASK(RTC_CNTL_PWC_REG, RTC_CNTL_PAD_FORCE_HOLD_M);
    components/esp_hal_gpio/include/hal/rtc_io_hal.h:239:#define rtcio_hal_hold_enable(rtcio_num) rtcio_ll_force_hold_enable(rtcio_num)
Z10 components/esp_hw_support/include/esp_private/esp_sleep_internal.h:112:void esp_sleep_isolate_digital_gpio(bool do_backup);
    components/esp_hw_support/sleep_gpio.c:281:IRAM_ATTR void esp_sleep_isolate_digital_gpio(bool do_backup)
    components/esp_hw_support/sleep_modes.c:862:        esp_sleep_isolate_digital_gpio(false);
    components/esp_hw_support/sleep_modes.c:904:            esp_sleep_isolate_digital_gpio(true);
Z11 docs/en/api-reference/system/sleep_modes.rst:272:        - If the RTC peripherals are configured to be powered down (which is by default), the wakeup IOs will be set to the holding state before entering sleep. Therefore, after the chip wakes up from Light-sleep, please call ``rtc_gpio_hold_dis`` to disable the hold function to perform any pin re-configuration. For Deep-sleep wakeup, this is already being handled at the application startup stage.
    sleep_modes.rst:602: .. only:: SOC_RTCIO_HOLD_SUPPORTED and SOC_RTCIO_INPUT_OUTPUT_SUPPORTED
    sleep_modes.rst:616: - RTC GPIOs (GPIO0 ~ 5) can be in the following states, depending on their hold function enabled or not:
    sleep_modes.rst:617: - if the hold function is not enabled, RTC GPIOs will be in a high impedance state.
    sleep_modes.rst:618: - if the hold function is enabled, RTC GPIOs will retain the pin state latched at that hold moment.
    (บรรทัด 244, 249 เกี่ยวกับ HOLD feature เมื่อ RTC_PERIPH domain ปิด ข้อความเต็มดูไฟล์ log)
Z12 sleep_modes.rst (ส่วนท้าย):
    - Deep-sleep mode (always) / - Light-sleep mode when CONFIG_PM_POWER_DOWN_PERIPHERAL_IN_LIGHT_SLEEP is enabled
    .. only:: SOC_RTC_GPIO_EDGE_WAKEUP_SUPPORTED  (ข้อความเรื่อง edge wakeup และ ANYEDGE)
    .. note:: Only GPIOs powered by the VDD3P3_RTC power domain (RTC IOs) can be used with this API.
    .. note:: Any GPIO/IO wakeup signal -- whether level or edge -- must be held (or have a pulse width) of at least 3 RTC slow-clock cycles ... (ตรงกับ EV-38 Z5)
```
- ข้อสรุป: (1) ชื่อ LL ของ hold ขา RTC คือ `rtcio_ll_force_hold_enable` ใช้ `RTC_CNTL_PAD_HOLD_REG` คนละรีจิสเตอร์กับ `RTC_CNTL_DIG_ISO_REG` ของ `gpio_deep_sleep_hold_en` (2) `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` ไม่นิยามใน soc_caps.h ของ S3 (ค้นเฉพาะ `soc/esp32s3` และ `esp_hal_gpio/esp32s3`) จึงใช้สาขา `#else` (3) ขา digital ที่ไม่ได้ hold ถูก isolate (4) ตอนตื่นจาก deep sleep ระบบปลด hold เฉพาะขาปลุก (5) ผู้เรียก isolate สองจุดใน `sleep_modes.c` ยังไม่เห็นบริบท (6) บรรทัด 616 ของ docs พูดถึง "GPIO0 ~ 5" ซึ่งไม่ใช่ S3 ห้ามนำมาใช้ (ข้อ 3 ต้องอ่านร่วมกับเงื่อนไข reserve ใน EV-40 ข้อ 5 และข้อ 5 ถูกแทนด้วย EV-40)

## EV-40 ผล verify_gpio_unv7.sh (รันบนเครื่องผู้ใช้ รอบ 5i อ่านดิบ: คัดลอกตรงตัวอักษร)
- แหล่ง: `verify7.log.txt` ของผู้ใช้ (IDF v6.1 ที่ `/root/esp/esp-idf`) path ย่อเป็น relative ต่อ `$IDF_PATH` ไฟล์นี้มีเฉพาะผล ไม่มีโค้ดสคริปต์ (โค้ดอยู่ใน `tools/verify_gpio_unv7.sh`)
- ข้อบกพร่องของสคริปต์: Z17 ใช้ `find -path '*esp32s3*'` ซึ่งตรงกับไดเรกทอรี `esp32s31` ด้วย ผลของ `esp32s31` ถูกตัดทิ้งจากบล็อกด้านล่าง (ไม่ใช่ S3)
- การย่อ: `(...)` คือส่วนที่ผู้ช่วยย่อ ไม่ใช่ซอร์สต้นฉบับ
```text
Z13 components/esp_hw_support/sleep_modes.c (บรรทัด 849-863 และ 890-906):
    849: static esp_err_t FORCE_IRAM_ATTR esp_sleep_start_safe(uint32_t sleep_flags, uint32_t reject_triggers, bool deep_sleep, ...)
    860:     if (deep_sleep) {
    861: #if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
    862:         esp_sleep_isolate_digital_gpio(false);
    863: #endif
    (...)
    890:         // Enter Deep Sleep
    901:     } else {
    902: #if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
    903:         if (sleep_flags & RTC_SLEEP_PD_DIG) {
    904:             esp_sleep_isolate_digital_gpio(true);
    905:         }
    906: #endif
    907:         /* Cache Suspend 1: will wait cache idle in cache suspend */
Z14 components/esp_hw_support/sleep_gpio.c:
    57:#define IO_WAKEUP_ENABLED   (RTC_GPIO_TRIG_EN | _IO_WAKEUP_EXT0 | _IO_WAKEUP_EXT1)
    396:    uint32_t io_wakeup_enabled = pmu_ll_hp_get_wakeup_enable(&PMU) & IO_WAKEUP_ENABLED;
    398:    uint32_t io_wakeup_enabled = rtc_cntl_ll_get_wakeup_enable() & IO_WAKEUP_ENABLED;
Z15 components/esp_driver_gpio/src/gpio.c:
    381:            // No need to reserve any pin for reading pad level (and do input check first, since output needs reserve IO which will make esp_gpio_is_reserved return true)
    383:            if (esp_gpio_is_reserved(bit_mask) && gpio_hal_input_is_enabled(gpio_context.gpio_hal, io_num)) {
    396:                if (esp_gpio_is_reserved(bit_mask) && !gpio_hal_input_is_enabled(gpio_context.gpio_hal, io_num)) {
    1150:        fprintf(out_stream, "IO[%"PRIu32"]%s -\n", gpio_num, esp_gpio_is_reserved(BIT64(gpio_num)) ? " **RESERVED**" : "");
    components/esp_driver_i2s/i2s_common.c:1098:    if (esp_gpio_is_reserved(BIT64(gpio_num))) {
    components/esp_eth/src/mac/esp_eth_mac_esp_gpio.c:40:            ESP_RETURN_ON_FALSE(esp_gpio_is_reserved(BIT64(gpio_num)) == false, ESP_ERR_INVALID_STATE,
    components/esp_hw_support/esp_gpio_reserve.c:25:bool esp_gpio_is_reserved(uint64_t gpio_mask)
    components/esp_hw_support/include/esp_private/esp_gpio_reserve.h:57:bool esp_gpio_is_reserved(uint64_t gpio_mask);
    components/esp_hw_support/sleep_gpio.c:318:            !esp_gpio_is_reserved(BIT64(gpio_num))) {
    components/esp_hw_support/sleep_gpio.c:346:            !esp_gpio_is_reserved(BIT64(gpio_num))) {
    components/esp_phy/src/phy_common.c:197:            if(esp_gpio_is_reserved(config->gpio_cfg[i].gpio_num)) {
    (grep ค้นเฉพาะชื่อ esp_gpio_is_reserved ไม่ได้ค้น esp_gpio_reserve)
Z16 docs/en/api-reference/system/sleep_modes.rst (ส่วน EXT1):
    "...ESP-IDF will automatically lock the state of the wakeup pin before the system enters sleep modes and unlock upon exiting sleep modes."
    ".. only:: not SOC_PM_SUPPORT_EXT1_WAKEUP_MODE_PER_PIN" / ".. note::" "Due to hardware limitation, when we use more than one IO for EXT1 wakeup, it is not allowed to configure different wakeup levels for the IOs, and there is corresponding inspection mechanism in :cpp:func:`esp_sleep_enable_ext1_wakeup_io`."
    "- If the RTC peripherals are configured to be powered down (which is by default), the wakeup IOs will be set to the holding state before entering sleep. Therefore, after the chip wakes up from Light-sleep, please call ``rtc_gpio_hold_dis`` to disable the hold function to perform any pin re-configuration. For Deep-sleep wakeup, this is already being handled at the application startup stage."
    ".. only:: SOC_RTCIO_HOLD_SUPPORTED and SOC_RTCIO_INPUT_OUTPUT_SUPPORTED" "To isolate a pin to prevent extra current draw, call :cpp:func:`rtc_gpio_isolate` function. (...) Add the following code before :cpp:func:`esp_deep_sleep_start` to remove such extra current:: rtc_gpio_isolate(GPIO_NUM_12);"
    ".. only:: esp32c2 or esp32c3" "In Deep-sleep mode: - digital GPIOs (GPIO6 ~ 21) are in a high impedance state. - RTC GPIOs (GPIO0 ~ 5) can be in the following states, depending on their hold function enabled or not: (...)"
Z17 components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h:
    471: static inline void gpio_ll_hold_en(gpio_dev_t *hw, uint32_t gpio_num)
    473:     SET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
    482: static inline void gpio_ll_hold_dis(gpio_dev_t *hw, uint32_t gpio_num)
    484:     CLEAR_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, GPIO_HOLD_MASK[gpio_num]);
    500: static inline bool gpio_ll_is_digital_io_hold(gpio_dev_t *hw, uint32_t gpio_num)
    502:     uint64_t bit_mask = 1ULL << gpio_num;
    503:     if (!(bit_mask & SOC_GPIO_VALID_DIGITAL_IO_PAD_MASK)) {
    504:         // GPIO 0-21 are not digital IO pads, 22-25 does not exist
    505:         abort();
    506:     }
    507:     return GET_PERI_REG_MASK(RTC_CNTL_DIG_PAD_HOLD_REG, bit_mask >> 21);
    components/esp_hal_gpio/include/hal/gpio_hal.h:
    301: #define gpio_hal_hold_en(hal, gpio_num) gpio_ll_hold_en((hal)->dev, gpio_num)
    317: #define gpio_hal_hold_dis(hal, gpio_num) gpio_ll_hold_dis((hal)->dev, gpio_num)
    333: #define gpio_hal_is_digital_io_hold(hal, gpio_num) gpio_ll_is_digital_io_hold((hal)->dev, gpio_num)
```
- ข้อมูลเพิ่มจาก RAG (ไม่ใช่ log): คำอธิบายของ `gpio_hal_hold_dis` ใน `esp_hal_gpio` ว่า "When the chip is woken up from Deep-sleep, the gpio will be set to the default mode" (ตรงกับ EV-11)
- ข้อสรุป: (1) isolate ถูกเรียกใน deep sleep ที่บรรทัด 862 และใน light sleep เฉพาะเมื่อ `SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD` นิยาม ซึ่ง S3 ไม่นิยามตาม EV-39 Z7 จึงไม่ครอบ light sleep บน S3 (ข้อจำกัด: ค้นสองไดเรกทอรี) (2) `IO_WAKEUP_ENABLED` รวม `RTC_GPIO_TRIG_EN`, `_IO_WAKEUP_EXT0`, `_IO_WAKEUP_EXT1` ไม่เห็น timer ในนิยามนี้ (ยังไม่เห็นนิยามของ `_IO_WAKEUP_EXT0/1`) (3) ผู้เรียก `esp_gpio_is_reserved` พบใน gpio driver, I2S, Ethernet MAC, PHY และ `sleep_gpio.c` (4) ขาที่ถูกจองถูกข้ามในลูป isolate (5) ยังไม่เห็นผู้เรียก `esp_gpio_reserve` จึงยังไม่ทราบว่าขา output ที่ตั้งผ่าน gpio driver ถูกจองหรือไม่ (6) docs ยืนยันข้อจำกัด EXT1 level เดียวบนชิปที่ไม่มี per-pin ซึ่งรวม S3 (7) บรรทัด 616 ของ docs เป็นของ C2/C3 (8) บน S3 `GPIO_HOLD_MASK[0..21] = 0` ทำให้ `gpio_hal_hold_dis` กับขา 0-21 ไม่แตะ `RTC_CNTL_DIG_PAD_HOLD_REG`