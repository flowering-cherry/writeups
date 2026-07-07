# Writeups

Security research and reverse engineering — short writeups on things I take apart.

- **axios supply-chain attack (npm)** — unpacking the obfuscated dropper and analysing the Windows (PowerShell) and Linux (Python) RAT: C2, persistence, command set. [→](reversing/malware/axios_compromise/axios_supplychain_attack.md)
- **KeyAuth 1.3 emulator** — getting past Themida/WinLicense anti-VM, hooking the Ed25519 signature check with Frida, redirecting license calls over DNS with a self-signed TLS cert. [→](reversing/auth_emulating/keyauth/keyauth.md)
- **Eyezy (stalkerware)** — an open Firebase Storage bucket on a ~3M-user app: anonymous read/write/delete, enabling DoS and code execution in the app's WebView. [→](random/eyezy/eyezy.md)
- **Steam overlay sandbox escape** — a single `document.write()` call bypasses the overlay browser's network restrictions, exposing session cookies and a WebAPI token. [→](random/steam/self-xss.md)
