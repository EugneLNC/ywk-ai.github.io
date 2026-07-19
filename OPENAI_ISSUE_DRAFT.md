# Windows: SecureLink process injection crashes Codex App host with 0xC0000005

### What version of the Codex App are you using (From “About Codex” dialog)?

26.715.4045.0 (Microsoft Store package `OpenAI.Codex_26.715.4045.0_x64__2p2nqsd0c76g0`)

### What subscription do you have?

ChatGPT subscription with Codex App access. The exact tier is omitted from this public diagnostic report because it does not affect the local crash.

### What platform is your computer?

Microsoft Windows NT 10.0.26200.0 x64 (Windows 11 Home)

### What issue are you seeing?

Codex App started and then silently exited within seconds. There was no visible crash dialog.

The crashing process was the Chromium-based desktop host, `ChatGPT.exe`, with exit code `3221225477` (`0xC0000005`, access violation). The bundled `codex.exe` child process subsequently exited with code `0`, so this was not a Codex CLI crash.

Codex App's Crashpad generated an approximately 44 MB minidump on each failed launch. A pre-crash module capture showed two non-OpenAI, non-Microsoft DLLs loaded in the `ChatGPT.exe` main process:

```text
C:\Windows\System32\drivers\WsInjtdll64.dll
C:\Windows\System32\drivers\WsInjProctDll64.dll
```

Both files had valid digital signatures from Wangsu Science and Technology Co., Ltd. The machine had SecureLink 3.8.5 installed. Its `SecureLink` service was running with automatic startup, and its `WsInjCoreDrv` kernel driver was running with system startup.

After uninstalling SecureLink with its vendor uninstaller and rebooting Windows:

- the `SecureLink` service was absent;
- the `WsInjCoreDrv` driver was absent;
- no `WsInj*` module was loaded in any `ChatGPT.exe` process;
- Codex App launched and remained stable;
- no new related Application Error / Windows Error Reporting event or Crashpad dump appeared after reboot.

This A/B result identifies SecureLink 3.8.5 process injection as the cause of this specific crash.

### What steps can reproduce the bug?

The issue reproduced consistently in this environment before SecureLink was removed:

1. Use Windows 11 x64 with SecureLink 3.8.5 installed and its `WsInjCoreDrv` driver active.
2. Launch the Microsoft Store Codex App package via the Start menu or `shell:AppsFolder\OpenAI.Codex_2p2nqsd0c76g0!App`.
3. Observe that `ChatGPT.exe` starts and silently exits within seconds with `0xC0000005`.
4. Observe a new dump under the app's Crashpad reports directory.
5. Capture the desktop host's loaded modules before it exits; `WsInjtdll64.dll` and `WsInjProctDll64.dll` are present.
6. Uninstall SecureLink using its supported uninstaller and reboot Windows.
7. Launch Codex App again. The `WsInj*` modules are absent and the app remains stable.

Additional isolation performed before identifying SecureLink:

- Starting with a completely fresh `.codex` directory did not change the crash.
- Disabling an Oray virtual display adapter and rebooting did not change the crash.
- Reinstalling/resetting user state did not address the host-process access violation.

### What is the expected behavior?

Codex App should remain open. If an unsupported third-party process-injection module is detected, the app should ideally surface an actionable compatibility message instead of terminating silently.

### Additional information

- This appears related by symptom to #25376 and #19258, but those reports do not identify SecureLink/Wangsu injection modules. This report has a confirmed before/after root cause.
- The valid Wangsu signatures show that the modules were authentic; they do not guarantee compatibility with the Chromium build embedded in Codex App.
- No raw minidump is attached publicly because process dumps may contain private in-memory data. A sanitized module list or dump can be provided through a private maintainer-approved channel if needed.
- Suggested product improvement: log the presence of non-system injected modules in the desktop crash diagnostics, and document security-client/overlay injection as a troubleshooting branch for Windows `0xC0000005` startup crashes.
