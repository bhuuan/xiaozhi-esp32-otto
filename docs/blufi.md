# BluFi Wi-Fi Provisioning (with esp-wifi-connect)

This document explains how to enable and use BluFi (BLE-based Wi-Fi provisioning) in the Xiaozhi firmware, using the built-in `esp-wifi-connect` component for Wi-Fi connection and credential storage. For the official BluFi protocol specification, see [Espressif docs](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/ble/blufi.html).

## Prerequisites

- Chip and firmware must support BLE.
- Enable in `idf.py menuconfig`: `WiFi Configuration Method` → `Esp Blufi` (`CONFIG_USE_ESP_BLUFI_WIFI_PROVISIONING=y`).
  - If using BluFi, the Hotspot option under the same menu must be disabled; otherwise the default is Hotspot provisioning.
- Default NVS and event loop initialization must be kept (already handled in the project's `app_main`).
- `CONFIG_BT_BLUEDROID_ENABLED` and `CONFIG_BT_NIMBLE_ENABLED` are mutually exclusive — enable only one.

## Workflow

1. Phone connects to device via BluFi (official EspBlufi app or custom client) and sends Wi-Fi SSID/password. The phone can also request a list of Wi-Fi networks scanned by the device via the BluFi protocol.
2. Device writes the credentials to `SsidManager` (stored in NVS, part of `esp-wifi-connect`) in the `ESP_BLUFI_EVENT_REQ_CONNECT_TO_AP` handler.
3. `WifiStation` starts scanning and connecting; status is reported back via BluFi.
4. On success the device auto-connects to the new Wi-Fi; on failure it returns a failure status.

## Steps

1. **Configure**: enable `Esp Blufi` in menuconfig. Build and flash firmware.
2. **Trigger provisioning**: device auto-enters provisioning mode on first boot if no saved Wi-Fi exists.
3. **Phone-side**: open EspBlufi app (or other BluFi client), search and connect to device, optionally enable encryption, then input and send Wi-Fi SSID/password.
4. **Check result**:
   - Success: BluFi reports connection success; device auto-connects to Wi-Fi.
   - Failure: BluFi returns failure status; retry or check router.

## Notes

- BluFi provisioning and Hotspot provisioning cannot both be enabled. If Hotspot provisioning is already active, it takes priority. Use menuconfig to select only one provisioning method.
- For repeated testing, clear or overwrite saved SSIDs in the `wifi` NVS namespace to avoid stale config interference.
- Custom BluFi clients must follow the official protocol frame format (see official docs link above).
- The official docs provide a download link for the EspBlufi app.
- Due to BluFi API changes in IDF 5.5.2: IDF 5.5.2 firmware uses the BLE name `"Xiaozhi-Blufi"`; IDF 5.5.1 uses `"BLUFI_DEVICE"`.
