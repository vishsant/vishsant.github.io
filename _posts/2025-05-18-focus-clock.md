---
layout: distill
title: A Simple Refocus Script
description: When complexity fails, simplicity prevails.
tags: productivity
giscus_comments: true
date: 2025-05-18
featured: true
---

Everyone's chasing the latest app until the app becomes the distraction.

I kept thinking, "`There must be a simpler way`"


So, I built this script: [Refocus clock](https://github.com/vishsant/Scripts-and-Configs/tree/main/scripts/windows/focus)

It doesn't ping you every hour or be for your data.

It simply sits next to your clock, reminding you of the one thing that matters:

`What's to focus now !!!`

No clunky dashboards.

No feature bloat.

No decision fatigue.

`When complexity fails, simplicity prevails.`

## Implementation

### Boilerplate & Environment Setup


```powershell
@echo off
setlocal EnableExtensions EnableDelayedExpansion
```
- `@echo off`: Prevents each command from being printed to the console—so you only see the messages you echo.
- `EnableExtensions` turns on extra cmd.exe features (in most modern Windows it’s already on anyway).
- `EnableDelayedExpansion` lets us safely use !VAR! inside loops or conditional blocks, so changes to variables are seen immediately.
       
### Building the New Clock Format

```powershell

reg query "HKCU\Control Panel\International" /v sShortTime >nul 2>&1

set "PSFILE=%TEMP%\FocusBroadcast.ps1"

> "%PSFILE%" echo Add-Type -Name NativeMethods -Namespace Win32 -MemberDefinition "[System.Runtime.InteropServices.DllImport(\"user32.dll\",SetLastError=true)] public static extern System.IntPtr SendMessageTimeout(System.IntPtr hWnd, uint Msg, System.UIntPtr wParam, string lParam, uint Flags, uint Timeout, out System.UIntPtr result);"

>> "%PSFILE%" echo [void][Win32.NativeMethods]::SendMessageTimeout([System.IntPtr]::Zero,0x001A,[System.UIntPtr]::Zero,'intl',0x0002,100,[ref]([System.UIntPtr]::Zero))

```

```powershell

[Win32.NativeMethods]::SendMessageTimeout(
    [System.IntPtr]::Zero,   # HWND_BROADCAST
    0x001A,                  # ← WM_SETTINGCHANGE
    [System.UIntPtr]::Zero,  # wParam (unused here)
    'intl',                  # lParam: the “international” settings section
    0x0002,                  # SMTO_ABORTIFHUNG
    100,                     # timeout in ms
    [ref]([System.UIntPtr]::Zero)
) | Out-Null

```

- Windows’ short-time format lives in a registry string called `sShortTime`.
- Build a tiny .ps1 file in %TEMP%—one line for the P/Invoke signature, one line for the call.
- `dd-Type … DllImport("user32.dll")` dynamically generates a .NET wrapper so PowerShell can call the native Win32 API.
- `SendMessageTimeout(HWND_BROADCAST, WM_SETTINGCHANGE, …, "intl", SMTO_ABORTIFHUNG, 100ms)` tells all top-level windows (Explorer included) that “international settings changed.”

### Invoke the broadcaster

```powershell

powershell -NoProfile -ExecutionPolicy Bypass -File "%PSFILE%" >nul 2>&1

```

- `-NoProfile` skips loading your PowerShell profile—faster and fewer surprises.
- `-ExecutionPolicy` Bypass ensures the script runs even if your system has a restricted policy.

