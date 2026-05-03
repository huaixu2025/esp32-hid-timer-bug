
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

## Reproduction Steps

### 1. Clone and Build

```bash
git clone https://github.com/huaixu2025/esp32-hid-timer-bug.git
cd esp32-hid-timer-bug
idf.py set-target esp32s3
idf.py build
```

### 2. Flash and Monitor

```bash
idf.py -p COMx flash monitor
```

### 3. Trigger the Bug

1. Connect a USB keyboard to the board's USB-OTG port.
2. Wait for the keyboard to be recognized (HID logs appear).
3. Press the **Enter** key.
4. Observe the output.

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

Issue submitted to Espressif (link to be added).

## License

Apache 2.0 — consistent with the official ESP-IDF examples.

---
