## verify_gpio_unv3.sh 

``` bash
#!/usr/bin/env bash
# verify_gpio_unv3.sh (รอบ 5c): ปิดข้อ [UNV] ของ gpio.md ด้วยการอ่านซอร์ส IDF v6.1 จริง (อ่านอย่างเดียว)
# X1 พิสูจน์: IDF บอกว่าชื่อ ROM GPIO เก่า (เช่น gpio_pad_select_gpio) ถูก "เปลี่ยนชื่อ" ไม่ใช่ถูกลบ
# X2 พิสูจน์: gpio_ll_wakeup_disable ของ S3 ไม่แตะ interrupt type
# X3 พิสูจน์: ตำแหน่งอาร์เรย์ GPIO_HOLD_MASK (ผลถูกตัดที่ดัชนี 29 เพราะ -A30 จึงมี X4 ต่อ)
# X4 พิสูจน์: ค่าของ GPIO_HOLD_MASK ครบทั้ง 49 รายการ เพื่อหา bit ของ GPIO38 ที่ใช้ใน T4
# วิธีใช้
#   . $IDF_PATH/export.sh
#   bash verify_gpio_unv3.sh > verify3.log 2>&1
set -u
: "${IDF_PATH:?ต้อง source export.sh ก่อน}"

echo "### X1 hints.yml (ข้อความ hint ของ gpio_pad_select_gpio)"
sed -n '1,14p' "$IDF_PATH/components/esp_rom/hints.yml"
echo "(จบ X1)"

echo "### X2 gpio_ll_wakeup_disable ของ S3"
grep -n -A6 "gpio_ll_wakeup_disable" "$IDF_PATH/components/esp_hal_gpio/esp32s3/include/hal/gpio_ll.h"
echo "(จบ X2)"

echo "### X3 นิยามอาร์เรย์ GPIO_HOLD_MASK (ตัดที่ดัชนี 29 ในรอบก่อน)"
grep -rn -A30 "GPIO_HOLD_MASK\[" "$IDF_PATH/components/soc/esp32s3" "$IDF_PATH/components/esp_hal_gpio/esp32s3" --include=*.c
echo "(จบ X3)"

echo "### X4 อาร์เรย์ GPIO_HOLD_MASK ฉบับเต็ม (บรรทัด 64-130 ของ gpio_periph.c)"
sed -n '64,130p' "$IDF_PATH/components/soc/esp32s3/gpio_periph.c"
echo "(จบ X4)"

```

## verify3.log ผล

```
``` code
### X1 hints.yml (ข้อความ hint ของ gpio_pad_select_gpio)
-
    re: "(error: implicit declaration of function|undefined reference to) '(gpio_pad_set_drv|gpio_pad_pullup|gpio_pad_select_gpio|gpio_pad_unhold|gpio_matrix_in|gpio_matrix_out)'"
    hint: "The rom GPIO functions have been renamed. Please replace with the APIs in 'esp_rom_gpio.h'."

-
    re: "fatal error: (esp_rom_tjpgd.h): No such file or directory"
    hint: "{} was removed. Please use esp_jpeg component from IDF component manager instead.\nPlease look out for component in 'https://components.espressif.com' and add using 'idf.py add-dependency' command.\nRefer to the migration guide for more details."
    match_to_output: True
(จบ X1)
### X2 gpio_ll_wakeup_disable ของ S3
380:static inline void gpio_ll_wakeup_disable(gpio_dev_t *hw, uint32_t gpio_num)
381-{
382-    hw->pin[gpio_num].wakeup_enable = 0;
383-}
384-
385-/**
386-  * @brief Set GPIO pad drive capability
(จบ X2)
### X3 นิยามอาร์เรย์ GPIO_HOLD_MASK (ตัดที่ดัชนี 29 ในรอบก่อน)
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c:64:const uint32_t GPIO_HOLD_MASK[] = {
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-65-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-66-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-67-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-68-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-69-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-70-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-71-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-72-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-73-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-74-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-75-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-76-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-77-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-78-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-79-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-80-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-81-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-82-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-83-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-84-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-85-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-86-    0,
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-87-    BIT(1),
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-88-    BIT(2),
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-89-    BIT(3),
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-90-    BIT(4),
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-91-    BIT(5),
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-92-    BIT(6),
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-93-    BIT(7),
/root/esp/esp-idf/components/soc/esp32s3/gpio_periph.c-94-    BIT(8),
(จบ X3)
### X4 อาร์เรย์ GPIO_HOLD_MASK ฉบับเต็ม (บรรทัด 64-130 ของ gpio_periph.c)
const uint32_t GPIO_HOLD_MASK[] = {
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    0,
    BIT(1),
    BIT(2),
    BIT(3),
    BIT(4),
    BIT(5),
    BIT(6),
    BIT(7),
    BIT(8),
    BIT(9),
    BIT(10),
    BIT(11),
    BIT(12),
    BIT(13),
    BIT(14),
    BIT(15),
    BIT(16),
    BIT(17),
    BIT(18),
    BIT(19),
    BIT(20),
    BIT(21),
    BIT(22),
    BIT(23),
    BIT(24),
    BIT(25),
    BIT(26),
    BIT(27),
};

_Static_assert(sizeof(GPIO_HOLD_MASK) == SOC_GPIO_PIN_COUNT * sizeof(uint32_t), "Invalid size of GPIO_HOLD_MASK");
(จบ X4)
```