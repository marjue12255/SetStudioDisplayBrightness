# Set-StudioDisplayBrightness – Description & Guide

Nov 8, 2026 · Marcus Jüttner

## How it works

The script sets the brightness of an Apple Studio Display on Windows. It sends an HID feature report straight to the display, using only what ships with Windows.

Sequence:

1. `Add-Type` compiles a small C# declaration in memory at runtime. Through it, PowerShell calls the Windows DLLs `hid.dll`, `setupapi.dll` and `kernel32.dll` via P/Invoke.
2. `SetupDiGetClassDevs` and `SetupDiEnumDeviceInterfaces` enumerate all HID devices.
3. For each device, `HidD_GetAttributes` reads the vendor and product ID, and `HidP_GetCaps` returns the usage page and feature report length.
4. Among matches with VID `05AC` and the right PID, the script tries the collection with usage page `0x80` (monitor control) first, then interface `mi_07`, then the rest.
5. `HidD_SetFeature` sends the report `[0x01, LSB, MSB, 0, 0, 0, 0]`. The brightness value ranges from 400 (0 %) to 60000 (100 %) and is sent as a 16-bit value.

The script writes nothing to disk and changes no system settings. It only sets the display's current brightness.

## Origin

The script is based on the open-source project [himbeles/studi](https://github.com/himbeles/studi) on GitHub. It is written in Rust and uses the `hidapi` library.

Its README documents the report format (report ID 1, brightness as LSB/MSB) and the 400–60000 value range, both determined from a Wireshark capture. The PowerShell script reproduces the same thing directly against the Windows HID API, without `hidapi` and without a compiled program.

## Requirements and use without install rights

The script also runs on locked-down company PCs where you cannot install software. It only needs what Windows already includes:

- Windows PowerShell 5.1 and .NET Framework 4.x (preinstalled since Windows 10)
- `csc.exe` from the .NET Framework, which `Add-Type` calls internally
- the system DLLs `hid.dll`, `setupapi.dll` and `kernel32.dll`
- standard user rights, no administrator rights

The display must be connected via Thunderbolt or USB-C **with USB data**. HDMI or DisplayPort adapters carry only the picture, so there is no HID channel.

Limits on managed PCs:

- **Execution policy:** `-ExecutionPolicy Bypass` on the command line applies to that one process only and changes no setting. If group policy blocks that too, paste the content into an interactive PowerShell console.
- **AppLocker/WDAC or Constrained Language Mode:** If these block `Add-Type` or `csc.exe`, the script will not work. This cannot be bypassed from PowerShell and should be raised with IT.

## Parameters

`-Percent` and `-List` are mutually exclusive. One of them is required.

| Parameter | Type / values | Default | Purpose |
| --- | --- | --- | --- |
| `-Percent` | Integer 0–100 | – | Sets the brightness in percent |
| `-List` | Switch | – | Shows all Apple HID collections (VID, PID, interface, usage page, FeatureLen, path) and changes nothing |
| `-ProductId` | Hex, e.g. `1114` or `0x1116` | `1114` | Product ID of the display. Known values are 1114 (default) and 1116 as a tested alternative |
| `-Verbose` | Switch | off | Shows every attempt with its interface and Win32 error code |

On the test PC the display reports PID `1116`, so `-ProductId 1116` must be passed there. Setting the brightness works with it, so 1116 is a confirmed alternative to 1114.

## Step-by-step usage

1. Copy `Set-StudioDisplayBrightness.ps1` into a folder, e.g. `Documents\Display`.
2. Open a command prompt or PowerShell in that folder. No administrator rights are needed.
3. Check that Windows sees the display as a USB device. The PID appears in the InstanceId column, e.g. `PID_1116`.

   ```powershell
   Get-PnpDevice | Where-Object InstanceId -match 'VID_05AC' | Format-Table Status,Class,FriendlyName,InstanceId -AutoSize
   ```

4. List the HID collections:

   ```
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Set-StudioDisplayBrightness.ps1 -List
   ```

5. Set the brightness, adding `-ProductId` if the PID differs:

   ```
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\Set-StudioDisplayBrightness.ps1 -Percent 50 -ProductId 1116
   ```

6. If it fails, repeat the same call with `-Verbose`. The output shows which interface failed with which error code.

Handy for daily use: one shortcut or `.cmd` file per brightness level, e.g. `Bright.cmd` with `-Percent 100` and `Dim.cmd` with `-Percent 20`.

## Troubleshooting

The script prints its messages in German. The first column quotes them as they appear.

| Message | Likely cause | Fix |
| --- | --- | --- |
| „Kein passendes Studio Display … gefunden“ with a list of Apple collections (no matching display found) | Display reports a different PID | Pass the PID from the list with `-ProductId` |
| „Windows sieht ueberhaupt kein Apple-HID-Geraet“ (Windows sees no Apple HID device) | No USB data connection | Use a Thunderbolt or data-capable USB-C cable directly to the PC |
| „Studio Display gefunden, aber keine Collection hat den Feature-Report angenommen“ (display found, report rejected) | Different report ID or value range | Check the output of `-List` and `-Verbose`, then adjust the script |
| Error from `Add-Type` | AppLocker, WDAC or Constrained Language Mode | Ask IT whether `Add-Type` can be allowed |
| Running scripts is disabled on this system | Execution policy | Call with `-ExecutionPolicy Bypass` or paste the content into the console |

Error code 5 means “access denied”, error code 87 “invalid parameter”. The latter usually points to a wrong report length or report ID.
