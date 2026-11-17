## verify_gpio_unv8.sh

``` bash
#!/usr/bin/env bash
# verify_gpio_unv8.sh (รอบ 5i): ปิดข้อที่ verify_gpio_unv7.sh ยังตอบไม่ครบ (อ่านอย่างเดียว)
# Z18 พิสูจน์: ใครเรียก esp_gpio_reserve/esp_gpio_revoke และตัวฟังก์ชันจองทำอะไร (ตัดสินว่าขา output ที่ตั้งด้วย gpio driver ถูกจองและข้ามการ isolate หรือไม่ ซึ่งกระทบ GPIO-33)
# Z19 พิสูจน์: SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD ถูกนิยามที่ไหนบ้างทั้ง components (Z7 ค้นแค่ soc/esp32s3 และ esp_hal_gpio/esp32s3)
# Z20 พิสูจน์: _IO_WAKEUP_EXT0, _IO_WAKEUP_EXT1 และ RTC_GPIO_TRIG_EN นิยามอย่างไร (เงื่อนไขที่ระบบปลด hold ตอนตื่นจาก deep sleep)
# Z21 พิสูจน์: บริบทของ gpio.c บรรทัด 360-420 (ที่มาของคอมเมนต์ว่า output ต้อง reserve IO)
# Z22 พิสูจน์: esp_sleep_start_safe ถูกเรียกจากไหน และ stack ถูกสลับก่อนถึงจุด isolate หรือไม่ (ตัดสินขอบเขตกฎ stack ของ GPIO-32)
# วิธีใช้
#   . $IDF_PATH/export.sh
#   bash verify_gpio_unv8.sh > verify8.log 2>&1
set -u
: "${IDF_PATH:?ต้อง source export.sh ก่อน}"
C="$IDF_PATH/components"

echo "### Z18 esp_gpio_reserve"
grep -rnE "esp_gpio_(reserve|revoke)[a-z_]*\(" "$C" --include=*.c --include=*.h
echo "--- esp_gpio_reserve.c บรรทัด 1-80"
sed -n '1,80p' "$C/esp_hw_support/esp_gpio_reserve.c"
echo "(จบ Z18)"

echo "### Z19 SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD ทั้ง components"
grep -rn "SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD" "$C"
echo "(จบ Z19)"

echo "### Z20 นิยามเงื่อนไข IO wakeup"
sed -n '40,70p' "$C/esp_hw_support/sleep_gpio.c"
echo "--- RTC_GPIO_TRIG_EN"
grep -rn "RTC_GPIO_TRIG_EN" "$C/soc/esp32s3" "$C/esp_hw_support"
echo "(จบ Z20)"

echo "### Z21 gpio.c บรรทัด 360-420"
sed -n '360,420p' "$C/esp_driver_gpio/src/gpio.c"
echo "(จบ Z21)"

echo "### Z22 esp_sleep_start_safe และจุดเรียก"
grep -n "esp_sleep_start_safe\|esp_deep_sleep_start\|esp_sleep_start(" "$C/esp_hw_support/sleep_modes.c"
echo "--- บริบทของการเรียก esp_sleep_start_safe"
grep -n -B4 -A4 "esp_sleep_start_safe(" "$C/esp_hw_support/sleep_modes.c"
echo "(จบ Z22)"
```

## verify8.log

``` code
### Z18 esp_gpio_reserve
/root/esp/esp-idf/components/app_trace/port/port_uart.c:211:        uint64_t r = esp_gpio_reserve(gpio_mask);
/root/esp/esp-idf/components/app_trace/port/port_uart.c:307:    esp_gpio_revoke(gpio_mask);
/root/esp/esp-idf/components/bt/porting_btdm/transport/driver/uart/hci_driver_uart_config.c:47:    esp_gpio_revoke(BIT64(uart_param->hci_uart_tx_pin));
/root/esp/esp-idf/components/bt/porting_btdm/transport/driver/uart/hci_driver_uart_config.c:48:    esp_gpio_revoke(BIT64(uart_param->hci_uart_rx_pin));
/root/esp/esp-idf/components/bt/porting_btdm/transport/driver/uart/hci_driver_uart_config.c:50:        esp_gpio_revoke(BIT64(uart_param->hci_uart_cts_pin));
/root/esp/esp-idf/components/bt/porting_btdm/transport/driver/uart/hci_driver_uart_config.c:54:        esp_gpio_revoke(BIT64(uart_param->hci_uart_rts_pin));
/root/esp/esp-idf/components/bt/porting/transport/driver/uart/hci_driver_uart_config.c:47:    esp_gpio_revoke(BIT64(uart_param->hci_uart_tx_pin));
/root/esp/esp-idf/components/bt/porting/transport/driver/uart/hci_driver_uart_config.c:48:    esp_gpio_revoke(BIT64(uart_param->hci_uart_rx_pin));
/root/esp/esp-idf/components/bt/porting/transport/driver/uart/hci_driver_uart_config.c:50:        esp_gpio_revoke(BIT64(uart_param->hci_uart_cts_pin));
/root/esp/esp-idf/components/bt/porting/transport/driver/uart/hci_driver_uart_config.c:54:        esp_gpio_revoke(BIT64(uart_param->hci_uart_rts_pin));
/root/esp/esp-idf/components/driver/twai/twai.c:321:    uint64_t busy_mask = esp_gpio_reserve(gpio_mask);
/root/esp/esp-idf/components/driver/twai/twai.c:346:    esp_gpio_revoke(gpio_mask);
/root/esp/esp-idf/components/esp_driver_gpio/src/gpio.c:390:                uint64_t old_busy_mask = esp_gpio_reserve(bit_mask);
/root/esp/esp-idf/components/esp_driver_gpio/src/gpio.c:513:    esp_gpio_revoke(BIT64(gpio_num));
/root/esp/esp-idf/components/esp_driver_i2c/i2c_common.c:315:    uint64_t old_gpio_rsv_mask = esp_gpio_reserve(BIT64(handle->sda_num) | BIT64(handle->scl_num));
/root/esp/esp-idf/components/esp_driver_i2c/i2c_common.c:364:    uint64_t old_gpio_rsv_mask = esp_gpio_reserve(BIT64(handle->sda_num) | BIT64(handle->scl_num));
/root/esp/esp-idf/components/esp_driver_i2c/i2c_common.c:436:    esp_gpio_revoke(BIT64(handle->sda_num));
/root/esp/esp-idf/components/esp_driver_i2c/i2c_common.c:437:    esp_gpio_revoke(BIT64(handle->scl_num));
/root/esp/esp-idf/components/esp_driver_i2s/i2s_common.c:1090: *       global GPIO reserve map (no esp_gpio_reserve), to avoid esp_gpio_revoke()
/root/esp/esp-idf/components/esp_driver_i2s/i2s_common.c:1111:    if (esp_gpio_reserve(BIT64(gpio_num)) & BIT64(gpio_num)) {
/root/esp/esp-idf/components/esp_driver_i2s/i2s_common.c:1126:    esp_gpio_revoke(revoke_mask);
/root/esp/esp-idf/components/esp_driver_ledc/src/ledc.c:817:    uint64_t old_gpio_rsv_mask = esp_gpio_reserve(BIT64(gpio_num));
/root/esp/esp-idf/components/esp_driver_ledc/src/ledc.c:848:        esp_gpio_revoke(BIT64(gpio_num));
/root/esp/esp-idf/components/esp_driver_mcpwm/src/mcpwm_gen.c:71:    uint64_t old_gpio_rsv_mask = esp_gpio_reserve(BIT64(config->gen_gpio_num));
/root/esp/esp-idf/components/esp_driver_mcpwm/src/mcpwm_gen.c:108:        esp_gpio_revoke(BIT64(gen->gen_gpio_num));
/root/esp/esp-idf/components/esp_driver_rmt/src/rmt_tx.c:217:        esp_gpio_revoke(BIT64(tx_channel->base.gpio_num));
/root/esp/esp-idf/components/esp_driver_rmt/src/rmt_tx.c:337:    uint64_t old_gpio_rsv_mask = esp_gpio_reserve(BIT64(config->gpio_num));
/root/esp/esp-idf/components/esp_driver_rmt/src/rmt_tx.c:1147:            esp_gpio_revoke(BIT64(tx_chan->base.gpio_num));
/root/esp/esp-idf/components/esp_driver_rmt/src/rmt_tx.c:1151:        uint64_t old_gpio_rsv_mask = esp_gpio_reserve(BIT64(gpio_num));
/root/esp/esp-idf/components/esp_driver_sdm/src/sdm.c:336:        esp_gpio_revoke(BIT64(chan->gpio_num));
/root/esp/esp-idf/components/esp_driver_sdm/src/sdm.c:370:    uint64_t old_gpio_rsv_mask = esp_gpio_reserve(BIT64(config->gpio_num));
/root/esp/esp-idf/components/esp_driver_spi/src/gpspi/spi_common.c:613:    uint64_t orig_occupied_map = esp_gpio_reserve(BIT64(gpio_num));
/root/esp/esp-idf/components/esp_driver_spi/src/gpspi/spi_common.c:831:            esp_gpio_revoke(BIT64(bus_cfg->iocfg[i]));
/root/esp/esp-idf/components/esp_driver_spi/src/gpspi/spi_common.c:883:        esp_gpio_revoke(BIT64(cs_gpio_num));
/root/esp/esp-idf/components/esp_driver_touch_sens/common/touch_sens_common.c:48:    if (esp_gpio_reserve(BIT64(pin)) & BIT64(pin)) {
/root/esp/esp-idf/components/esp_driver_twai/esp_twai_onchip.c:139:    uint64_t busy_mask = esp_gpio_reserve(reserve_mask);
/root/esp/esp-idf/components/esp_driver_twai/esp_twai_onchip.c:152:    esp_gpio_revoke(node->gpio_reserved);
/root/esp/esp-idf/components/esp_driver_uart/src/uart.c:858:    esp_gpio_revoke(uart_context[uart_num].io_reserved_mask & released_io_mask);
/root/esp/esp-idf/components/esp_driver_uart/src/uart.c:1018:    uint64_t old_busy_mask = esp_gpio_reserve(io_reserve_mask);
/root/esp/esp-idf/components/esp_eth/src/mac/esp_eth_mac_esp_gpio.c:48:            ESP_RETURN_ON_FALSE((esp_gpio_reserve(BIT64(gpio_num)) & BIT64(gpio_num)) == 0, ESP_ERR_INVALID_STATE,
/root/esp/esp-idf/components/esp_eth/src/mac/esp_eth_mac_esp_gpio.c:57:            ESP_RETURN_ON_FALSE((esp_gpio_reserve(BIT64(gpio_num)) & BIT64(gpio_num)) == 0, ESP_ERR_INVALID_STATE,
/root/esp/esp-idf/components/esp_eth/src/mac/esp_eth_mac_esp_gpio.c:89:            ESP_RETURN_ON_FALSE((esp_gpio_reserve(BIT64(iomux_info->gpio_num)) & BIT64(iomux_info->gpio_num)) == 0, ESP_ERR_INVALID_STATE,
/root/esp/esp-idf/components/esp_eth/src/mac/esp_eth_mac_esp_gpio.c:342:            esp_gpio_revoke(BIT64(gpio_num));
/root/esp/esp-idf/components/esp_hw_support/debug_probe/debug_probe.c:75:    esp_gpio_revoke(unit->pin_bit_mask);
/root/esp/esp-idf/components/esp_hw_support/debug_probe/debug_probe.c:113:    uint64_t old_gpio_rsv_mask = esp_gpio_reserve(pin_bit_mask);
/root/esp/esp-idf/components/esp_hw_support/esp_gpio_reserve.c:15:uint64_t esp_gpio_reserve(uint64_t gpio_mask)
/root/esp/esp-idf/components/esp_hw_support/esp_gpio_reserve.c:20:uint64_t esp_gpio_revoke(uint64_t gpio_mask)
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_gpio_reserve.h:39:uint64_t esp_gpio_reserve(uint64_t gpio_mask);
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_gpio_reserve.h:47:uint64_t esp_gpio_revoke(uint64_t gpio_mask);
/root/esp/esp-idf/components/esp_lcd/rgb/esp_lcd_panel_rgb.c:867:    uint64_t busy_mask = esp_gpio_reserve(gpio_reserve_mask);
/root/esp/esp-idf/components/esp_lcd/rgb/esp_lcd_panel_rgb.c:994:        esp_gpio_revoke(rgb_panel->gpio_reserve_mask);
/root/esp/esp-idf/components/esp_psram/device/esp_psram_impl_ap_quad.c:281:    esp_gpio_reserve(BIT64(cs1_io) | BIT64(wp_io));
/root/esp/esp-idf/components/esp_psram/esp32/esp_psram_impl_quad.c:835:    esp_gpio_reserve(BIT64(psram_io->flash_clk_io)        |
/root/esp/esp-idf/components/esp_psram/esp32s2/esp_psram_impl_quad.c:386:    esp_gpio_reserve(BIT64(psram_io.flash_clk_io)        |
/root/esp/esp-idf/components/esp_psram/esp32s3/esp_psram_impl_octal.c:312:    esp_gpio_reserve(BIT64(OCT_PSRAM_CS1_IO));
/root/esp/esp-idf/components/spi_flash/flash_ops.c:177:    esp_gpio_reserve(reserve_pin_mask);
--- esp_gpio_reserve.c บรรทัด 1-80
/*
 * SPDX-FileCopyrightText: 2023-2024 Espressif Systems (Shanghai) CO LTD
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <stdatomic.h>
#include "soc/soc_caps.h"
#include "esp_types.h"
#include "esp_bit_defs.h"
#include "esp_private/esp_gpio_reserve.h"

static _Atomic uint64_t s_reserved_pin_mask = ATOMIC_VAR_INIT(~(SOC_GPIO_VALID_GPIO_MASK));

uint64_t esp_gpio_reserve(uint64_t gpio_mask)
{
    return atomic_fetch_or(&s_reserved_pin_mask, gpio_mask);
}

uint64_t esp_gpio_revoke(uint64_t gpio_mask)
{
    return atomic_fetch_and(&s_reserved_pin_mask, ~gpio_mask);
}

bool esp_gpio_is_reserved(uint64_t gpio_mask)
{
    return atomic_load(&s_reserved_pin_mask) & gpio_mask;
}

// TODO: IDF-6968 reserve the pins that not fanned out regarding the SiP version
(จบ Z18)
### Z19 SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD ทั้ง components
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_sleep_internal.h:101:#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_sleep_internal.h:130:#endif // !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/linker.lf:89:    if (SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP = n || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD = y):
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:197:#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:200:#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:213:#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:279:#endif // SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:303:#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:353:#endif //!SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:861:#if !SOC_GPIO_SUPPORT_HOLD_SINGLE_IO_IN_DSLP || SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:902:#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:931:#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:993:#if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/soc/esp32s31/include/soc/Kconfig.soc_caps.in:642:config SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD
/root/esp/esp-idf/components/soc/esp32s31/include/soc/soc_caps.h:229:#define SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD        (1)
/root/esp/esp-idf/components/spi_flash/linker.lf:85:        if SOC_GPIO_NEED_SOFT_ISOLATE_DURING_PD = y:
(จบ Z19)
### Z20 นิยามเงื่อนไข IO wakeup
#include "esp_private/sleep_gpio.h"
#include "esp_private/spi_flash_os.h"
#include "esp_private/startup_internal.h"
#include "bootloader_flash.h"

ESP_LOG_ATTR_TAG(TAG, "sleep_gpio");

#if SOC_PM_SUPPORT_EXT0_WAKEUP
#define _IO_WAKEUP_EXT0     RTC_EXT0_TRIG_EN
#else
#define _IO_WAKEUP_EXT0     0
#endif
#if SOC_PM_SUPPORT_EXT1_WAKEUP
#define _IO_WAKEUP_EXT1     RTC_EXT1_TRIG_EN
#else
#define _IO_WAKEUP_EXT1     0
#endif
#define IO_WAKEUP_ENABLED   (RTC_GPIO_TRIG_EN | _IO_WAKEUP_EXT0 | _IO_WAKEUP_EXT1)

#if CONFIG_IDF_TARGET_ESP32
/* On ESP32, for IOs with RTC functionality, setting SLP_PU, SLP_PD couldn't change IO status
 * from FUN_PU, FUN_PD to SLP_PU, SLP_PD at sleep.
 */
typedef struct gpio_slp_mode_cfg {
    volatile uint32_t fun_pu;
    volatile uint32_t fun_pd;
} gpio_slp_mode_cfg_t;

static DRAM_ATTR gpio_slp_mode_cfg_t gpio_cfg = {};

void esp_sleep_gpio_pupd_config_workaround_apply(void)
--- RTC_GPIO_TRIG_EN
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_pmu.h:65:#define RTC_GPIO_TRIG_EN            (PMU_GPIO_WAKEUP_EN | PMU_LP_GPIO_WAKEUP_EN)      //!< GPIO & LP_GPIO wakeup
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_pmu.h:67:#define RTC_GPIO_TRIG_EN            (PMU_GPIO_WAKEUP_EN)
/root/esp/esp-idf/components/esp_hw_support/include/esp_private/esp_pmu.h:145:                               RTC_GPIO_TRIG_EN         | \
/root/esp/esp-idf/components/esp_hw_support/port/esp32/include/soc/rtc.h:588:#define RTC_GPIO_TRIG_EN    BIT(2)  //!< GPIO wakeup (light sleep only)
/root/esp/esp-idf/components/esp_hw_support/port/esp32/include/soc/rtc.h:602:#define RTC_SLEEP_REJECT_MASK (RTC_GPIO_TRIG_EN | RTC_SDIO_TRIG_EN)
/root/esp/esp-idf/components/esp_hw_support/port/esp32/rtc_sleep.c:281:    if (reject_opt & RTC_GPIO_TRIG_EN) {
/root/esp/esp-idf/components/esp_hw_support/port/esp32/rtc_sleep.c:309:    if (reject_opt & RTC_GPIO_TRIG_EN) {
/root/esp/esp-idf/components/esp_hw_support/port/esp32c2/include/soc/rtc.h:600:#define RTC_GPIO_TRIG_EN            BIT(2)  //!< GPIO wakeup
/root/esp/esp-idf/components/esp_hw_support/port/esp32c2/include/soc/rtc.h:613:#define RTC_SLEEP_REJECT_MASK (RTC_GPIO_TRIG_EN         | \
/root/esp/esp-idf/components/esp_hw_support/port/esp32c3/include/soc/rtc.h:657:#define RTC_GPIO_TRIG_EN            BIT(2)  //!< GPIO wakeup
/root/esp/esp-idf/components/esp_hw_support/port/esp32c3/include/soc/rtc.h:670:#define RTC_SLEEP_REJECT_MASK (RTC_GPIO_TRIG_EN         | \
/root/esp/esp-idf/components/esp_hw_support/port/esp32s2/include/soc/rtc.h:687:#define RTC_GPIO_TRIG_EN    BIT(2)  //!< GPIO wakeup (light sleep only)
/root/esp/esp-idf/components/esp_hw_support/port/esp32s2/include/soc/rtc.h:706:                               RTC_GPIO_TRIG_EN         | \
/root/esp/esp-idf/components/esp_hw_support/port/esp32s3/include/soc/rtc.h:673:#define RTC_GPIO_TRIG_EN    BIT(2)  //!< GPIO wakeup (light sleep only)
/root/esp/esp-idf/components/esp_hw_support/port/esp32s3/include/soc/rtc.h:692:                               RTC_GPIO_TRIG_EN         | \
/root/esp/esp-idf/components/esp_hw_support/sleep_gpio.c:57:#define IO_WAKEUP_ENABLED   (RTC_GPIO_TRIG_EN | _IO_WAKEUP_EXT0 | _IO_WAKEUP_EXT1)
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:1056:    if ((sleep_flags & RTC_SLEEP_PD_DIG) && (s_config.wakeup_triggers & RTC_GPIO_TRIG_EN)) {
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:1815:    } else if (CHECK_SOURCE(source, ESP_SLEEP_WAKEUP_GPIO, RTC_GPIO_TRIG_EN)) {
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:1820:        s_config.wakeup_triggers &= ~RTC_GPIO_TRIG_EN;
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:2365:    s_config.wakeup_triggers |= RTC_GPIO_TRIG_EN;
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:2381:    s_config.wakeup_triggers |= RTC_GPIO_TRIG_EN;
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:2523:    } else if (wakeup_cause & RTC_GPIO_TRIG_EN) {
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:2603:    if (wakeup_cause_raw & RTC_GPIO_TRIG_EN) {
/root/esp/esp-idf/components/esp_hw_support/sleep_modes.c:2906:        if (s_config.wakeup_triggers & (RTC_EXT0_TRIG_EN | RTC_GPIO_TRIG_EN)) {
(จบ Z20)
### Z21 gpio.c บรรทัด 360-420

    if (pGPIOConfig->pin_bit_mask == 0 ||
            pGPIOConfig->pin_bit_mask & ~SOC_GPIO_VALID_GPIO_MASK) {
        ESP_LOGE(GPIO_TAG, "GPIO_PIN mask error ");
        return ESP_ERR_INVALID_ARG;
    }

    if (pGPIOConfig->mode & GPIO_MODE_DEF_OUTPUT &&
            pGPIOConfig->pin_bit_mask & ~SOC_GPIO_VALID_OUTPUT_GPIO_MASK) {
        ESP_LOGE(GPIO_TAG, "GPIO can only be used as input mode");
        return ESP_ERR_INVALID_ARG;
    }

    do {
        if (((gpio_pin_mask >> io_num) & BIT(0))) {
            // This function will set the pin to a certain mode, instead of adding new modes to current status
            // If GPIO_MODE_DEF_OUTPUT flag is not set, output will be disabled; Same for GPIO_MODE_DEF_INPUT flag
            // Also the func sel will always be set to GPIO function in this function
            // so IO conflict check is tight here
            uint64_t bit_mask = BIT64(io_num);
            bool conflict = false;
            // No need to reserve any pin for reading pad level (and do input check first, since output needs reserve IO which will make esp_gpio_is_reserved return true)
            // but if the pin has been used as an IOMUX input, neither it can be configured to GPIO function, nor input can be disabled
            if (esp_gpio_is_reserved(bit_mask) && gpio_hal_input_is_enabled(gpio_context.gpio_hal, io_num)) {
                conflict = true;
            }
            // Currently, we have no way to know if an IO has been used as a GPIO matrix input (no reserve for this case),
            // so we cannot check such case and skip if input will be disabled
            if ((pGPIOConfig->mode) & GPIO_MODE_DEF_OUTPUT) {
                // need to reserve the pin if it is used as an output
                uint64_t old_busy_mask = esp_gpio_reserve(bit_mask);
                if (old_busy_mask & bit_mask) {
                    conflict = true;
                }
            } else {
                // gpio output will be disabled, so should skip for any reserved output pins
                if (esp_gpio_is_reserved(bit_mask) && !gpio_hal_input_is_enabled(gpio_context.gpio_hal, io_num)) {
                    conflict = true;
                }
            }
            if (conflict) {
                ESP_LOGW(GPIO_TAG, "conflict found for GPIO[%"PRIu32"]", io_num);
                // Right now, we just give a warning
                // Later, we should skip the pin and continue with the next one
            }

#if SOC_RTCIO_PIN_COUNT > 0
            if (rtc_gpio_is_valid_gpio(io_num)) {
                rtc_gpio_deinit(io_num);
            }
#endif

            if ((pGPIOConfig->mode) & GPIO_MODE_DEF_INPUT) {
                input_en = 1;
                gpio_input_enable(io_num);
            } else {
                gpio_input_disable(io_num);
            }

            if ((pGPIOConfig->mode) & GPIO_MODE_DEF_OD) {
                od_en = 1;
(จบ Z21)
### Z22 esp_sleep_start_safe และจุดเรียก
513:    esp_deep_sleep_start();
849:static esp_err_t FORCE_IRAM_ATTR esp_sleep_start_safe(uint32_t sleep_flags, uint32_t reject_triggers, bool deep_sleep, pmu_sleep_config_t *config)
851:static esp_err_t FORCE_IRAM_ATTR esp_sleep_start_safe(uint32_t sleep_flags, uint32_t reject_triggers, bool deep_sleep, rtc_sleep_config_t *config)
1005:        // Code outside of esp_sleep_start_safe may be linked to FLASH if CONFIG_PM_SLP_IRAM_OPT is false,
1017:static esp_err_t SLEEP_FN_ATTR esp_sleep_start(uint32_t sleep_flags, esp_sleep_mode_t mode, bool allow_sleep_rejection)
1186:        result = esp_sleep_start_safe(sleep_flags, reject_triggers, deep_sleep, &config);
1347:    if (esp_sleep_start(sleep_flags, ESP_SLEEP_MODE_DEEP_SLEEP, allow_sleep_rejection) == ESP_ERR_SLEEP_REJECT) {
1380:void FORCE_IRAM_ATTR esp_deep_sleep_start(void)
1409:    esp_err_t reject = esp_sleep_start(sleep_flags, ESP_SLEEP_MODE_LIGHT_SLEEP, true);
1793:    // will be updated by esp_sleep_start().
--- บริบทของการเรียก esp_sleep_start_safe
845-
846-static uint32_t call_rtc_sleep_start(uint32_t reject_triggers, uint32_t lslp_mem_inf_fpu, bool dslp);
847-
848-#if SOC_PMU_SUPPORTED
849:static esp_err_t FORCE_IRAM_ATTR esp_sleep_start_safe(uint32_t sleep_flags, uint32_t reject_triggers, bool deep_sleep, pmu_sleep_config_t *config)
850-#else
851:static esp_err_t FORCE_IRAM_ATTR esp_sleep_start_safe(uint32_t sleep_flags, uint32_t reject_triggers, bool deep_sleep, rtc_sleep_config_t *config)
852-#endif
853-{
854-    esp_err_t result = ESP_OK;
855-#if CONFIG_ESP_SLEEP_DEBUG
--
1182-#if CONFIG_PM_ESP_SLEEP_POWER_DOWN_CPU && !CONFIG_FREERTOS_UNICORE && SOC_PM_CPU_RETENTION_BY_SW
1183-        esp_sleep_cpu_skip_retention();
1184-#endif
1185-    } else {
1186:        result = esp_sleep_start_safe(sleep_flags, reject_triggers, deep_sleep, &config);
1187-    }
1188-#if CONFIG_ESP_SLEEP_CACHE_SAFE_ASSERTION
1189-    if (sleep_flags & RTC_SLEEP_PD_VDDSDIO) {
1190-        /* Cache Suspend 2: If previous sleep powerdowned the flash, suspend cache here so that the
(จบ Z22)
```