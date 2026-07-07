# Writeups

Security research and reverse engineering — short writeups on things I take apart.

- **axios supply-chain attack (npm)** — unpacking the obfuscated dropper and analysing the Windows (PowerShell) and Linux (Python) RAT: C2, persistence, command set. [→](reversing/malware/axios_compromise/axios_supplychain_attack.md)
- **KeyAuth 1.3 emulator** — getting past Themida/WinLicense anti-VM, hooking the Ed25519 signature check with Frida, redirecting license calls over DNS with a self-signed TLS cert. [→](reversing/auth_emulating/keyauth/keyauth.md)
- **Steam overlay sandbox escape** — a single `document.write()` call bypasses the overlay browser's network restrictions, exposing session cookies and a WebAPI token. [→](random/steam/self-xss.md)
