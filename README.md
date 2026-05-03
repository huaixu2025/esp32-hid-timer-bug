
# ESP-IDF `esp_timer` and USB HID Host Conflict Reproduction

This repository reproduces a bug in **ESP-IDF v5.1.2** where `esp_timer_start_periodic` fails after the **USB HID Host driver** is initialized.

- **Symptom**: `esp_timer_start_periodic` returns `ESP_ERR_INVALID_STATE` (0x103 / 259). If `ESP_ERROR_CHECK` is used, it triggers a system abort.
- **Impact**: Any periodic `esp_timer` task started after USB HID Host is running will fail irreversibly.

---

## Hardware Requirements

- ESP32-S3 development board (other ESP32 models with USB-OTG should also work)
- A standard USB keyboard

## Software Environment

- ESP-IDF **v5.1.2** (other v5.1.x versions may also be affected)
- This repository is based on the official `examples/peripherals/usb/host/hid` example

## How It Works

The **only modification** to the official HID example is in the keyboard callback: when the **Enter** key is pressed, the code attempts to create and start a periodic `esp_timer`. This single operation is sufficient to trigger the conflict.

```c
// In key_event_callback(), when Enter (0x28) is pressed:
esp_timer_handle_t test_timer;
esp_timer_create_args_t timer_args = {
    .callback = [](void *arg) { /* empty callback */ },
    .name = "test_timer"
};
esp_timer_create(&timer_args, &test_timer);

esp_err_t err = esp_timer_start_periodic(test_timer, 800 * 1000);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "esp_timer_start_periodic failed: %d", err);
}
```

---


## Steps to Reproduce

1. Hardware setup: Connect a USB keyboard to the ESP32-S3 USB OTG port.
2. Build, flash, and monitor the firmware.
3. Once the board boots and a USB keyboard is connected, press the **Enter** key.
4. Observe the serial monitor output — `esp_timer_start_periodic` fails with `ESP_ERR_INVALID_STATE (0x103)`.

## Clone and Build

```bash
git clone https://github.com/huaixu2025/esp32-hid-timer-bug.git
cd esp32-hid-timer-bug
cd hid
idf.py set-target esp32s3
idf.py build
idf.py flash monitor
```

### Expected Output

```
E (xxxxx) example: esp_timer_start_periodic failed: 259
```

- The program will **not crash** (error handling uses `if` instead of `ESP_ERROR_CHECK`).
- Subsequent keyboard presses may produce **no output** — the USB device appears disconnected.

---

## Getting a Crash (Optional)

To see the full abort and backtrace, replace the error handling with:

```c
ESP_ERROR_CHECK(err);
```

This triggers:

```
ESP_ERROR_CHECK failed: esp_err_t 0x103 (ESP_ERR_INVALID_STATE)
abort() was called at PC 0x4037a55f on core 0
```
## Error Log

With `ESP_ERROR_CHECK`, the full abort and backtrace is:

```
I (1554) example: test_timer created
I (1564) example: test_timer started periodic (800ms)

ESP_ERROR_CHECK failed: esp_err_t 0x103 (ESP_ERR_INVALID_STATE) at 0x42009595
0x42009595: key_event_callback at C:/Users/lihua/Desktop/hid/main/hid_host_example.c:254 (discriminator 1)

file: "./main/hid_host_example.c" line 254
func: key_event_callback
expression: esp_timer_start_periodic(test_timer, 800 * 1000)

abort() was called at PC 0x4037a55f on core 0
0x4037a55f: _esp_error_check_failed at E:/apps/Espressif/frameworks/esp-idf-v5.1.2/components/esp_system/esp_err.c:50



Backtrace: 0x40375922:0x3fca4bd0 0x4037a569:0x3fca4bf0 0x40380532:0x3fca4c10 0x4037a55f:0x3fca4c80 0x42009595:0x3fca4cb0 0x42009636:0x3fca4cf0 0x42009751:0x3fca4d20 0x42009c31:0x3fca4d90 0x4200a52c:0x3fca4db0 0x4200b75e:0x3fca4dd0 0x4200c04f:0x3fca4e00 0x42009b3a:0x3fca4e30 0x4037cb41:0x3fca4e50
0x40375922: panic_abort at E:/apps/Espressif/frameworks/esp-idf-v5.1.2/components/esp_system/panic.c:452

0x4037a569: esp_system_abort at E:/apps/Espressif/frameworks/esp-idf-v5.1.2/components/esp_system/port/esp_system_chip.c:84

0x40380532: abort at E:/apps/Espressif/frameworks/esp-idf-v5.1.2/components/newlib/abort.c:38

0x4037a55f: _esp_error_check_failed at E:/apps/Espressif/frameworks/esp-idf-v5.1.2/components/esp_system/esp_err.c:50

0x42009595: key_event_callback at C:/Users/lihua/Desktop/hid/main/hid_host_example.c:254 (discriminator 1)

0x42009636: hid_host_keyboard_report_callback at C:/Users/lihua/Desktop/hid/main/hid_host_example.c:314

0x42009751: hid_host_interface_callback at C:/Users/lihua/Desktop/hid/main/hid_host_example.c:394

0x42009c31: hid_host_user_interface_callback at C:/Users/lihua/Desktop/hid/managed_components/espressif__usb_host_hid/hid_host.c:298

0x4200a52c: in_xfer_done at C:/Users/lihua/Desktop/hid/managed_components/espressif__usb_host_hid/hid_host.c:710

0x4200b75e: _handle_pending_ep at E:/apps/Espressif/frameworks/esp-idf-v5.1.2/components/usb/usb_host.c:624

0x4200c04f: usb_host_client_handle_events at E:/apps/Espressif/frameworks/esp-idf-v5.1.2/components/usb/usb_host.c:773

0x42009b3a: event_handler_task at C:/Users/lihua/Desktop/hid/managed_components/espressif__usb_host_hid/hid_host.c:156

0x4037cb41: vPortTaskWrapper at E:/apps/Espressif/frameworks/esp-idf-v5.1.2/components/freertos/FreeRTOS-Kernel/portable/xtensa/port.c:162
```
---

## Workarounds Attempted (All Ineffective)

| Attempt | Result |
|---------|--------|
| Move `esp_timer_start_periodic` out of HID callback into `app_main` or a separate task | Same error (259) |
| Delay execution before starting timer | Same error (259) |
| Increase FreeRTOS `Tmr Svc` stack size | Irrelevant (timer never starts) |

**The only viable workaround** is to avoid `esp_timer_start_periodic` entirely and use `esp_timer_get_time()` with software polling instead.

---

## Affected ESP-IDF Versions

- **v5.1.2** — confirmed, 100% reproducible
- Other v5.1.x versions — likely affected (untested)

---

## Official Feedback

Issue submitted to Espressif: [https://github.com/espressif/esp-idf/issues/18546](https://github.com/espressif/esp-idf/issues/18546)

## License

Apache 2.0 — consistent with the official ESP-IDF examples.

---
