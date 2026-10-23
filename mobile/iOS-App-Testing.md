# iOS App Testing

iOS is more locked down than Android, which changes the toolkit but not the questions. You still ask what the app stores, what it exposes, what it trusts on a device the user controls, and how its backend behaves. The biggest differences are the need for a jailbroken device (or sideload tricks) for deep dynamic analysis, and Apple-specific storage and IPC mechanisms.

## The mental model

An iOS app ships as an IPA (a zip) containing a compiled Mach-O binary (often encrypted on-device by FairPlay), resources, and an `Info.plist` declaring capabilities, URL schemes, and entitlements. The sandbox is strong, so local escalation is harder than on Android, but the same client-trust failures apply: secrets in the binary, insecure Keychain/`NSUserDefaults`/file storage, weak transport, exposed URL schemes and universal links, and backend bugs that are identical to web.

## Where it hides / tell-tales

- **Insecure storage**: secrets or tokens in `NSUserDefaults` (plaintext plist), unprotected Core Data/SQLite, files written without a strong `NSFileProtection` class, or Keychain items with weak accessibility (`kSecAttrAccessibleAlways`) that persist unlocked.
- **Secrets in the binary**: hardcoded API keys, endpoints, and encryption keys recoverable with `strings` and a disassembler once the binary is decrypted.
- **Transport**: App Transport Security (ATS) exceptions in `Info.plist` (`NSAllowsArbitraryLoads`) permitting cleartext; no certificate pinning, or pinning bypassable with SSL Kill Switch/objection.
- **URL schemes and universal links**: custom schemes (`myapp://`) and associated-domain universal links that reach sensitive actions; the scheme handler trusting its input (see `Deep-Links-and-IPC.md`).
- **Pasteboard and backgrounding leaks**: sensitive data copied to the general pasteboard (readable by other apps), or the app snapshot (taken when backgrounded) capturing sensitive screens to disk.
- **WebView (WKWebView/UIWebView)**: JS bridges (`WKScriptMessageHandler`), loading untrusted content, and `UIWebView` (deprecated, more dangerous) with file access.
- **Jailbreak detection and client checks**: enforced only client-side, bypassable at runtime.

## Real material

Static triage of an IPA:

```
unzip app.ipa                              # Payload/<App>.app/
# read capabilities, URL schemes, ATS exceptions:
plutil -p Payload/App.app/Info.plist | grep -iE 'URLScheme|NSAllowsArbitraryLoads|Associated'
# decrypt the binary on a jailbroken device (App Store binaries are FairPlay-encrypted):
#   frida-ios-dump / flexdecrypt  -> then analyze in Ghidra/Hopper/radare2
strings Payload/App.app/App | grep -iE 'https?://|key|secret|token'
otool -L Payload/App.app/App                # linked libraries
```

Inspecting storage on a jailbroken device:

```
# app sandbox:
/var/mobile/Containers/Data/Application/<UUID>/
  Library/Preferences/<bundle>.plist        # NSUserDefaults (plaintext)
  Documents/   Library/Caches/              # files, often sensitive
# Keychain dump (objection or keychain_dumper):
objection -g com.target.app explore
  ios keychain dump
  ios nsuserdefaults get
  ios cookies get
```

Triggering a URL scheme to test the handler:

```
# from Safari or another app, or via simulator:
open "myapp://reset?user=victim&token=..."
# universal link:
https://app.target.com/path   (if associated domains route it into the app)
```

## Beyond the basics

- **Runtime instrumentation with Frida/objection**: the core dynamic technique. Bypass jailbreak detection and SSL pinning, dump the Keychain and `NSUserDefaults`, hook Objective-C/Swift methods to reveal decrypted secrets and flip client-side checks, and call internal methods directly. On a non-jailbroken device you can sometimes repackage the IPA with a Frida gadget and sideload.
- **Decrypt before you analyze**: App Store binaries are encrypted; static analysis of the encrypted binary yields little. Dump the decrypted binary (frida-ios-dump) first, then the strings and disassembly become meaningful.
- **Keychain accessibility matters**: a token in the Keychain is fine if `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`, but `kSecAttrAccessibleAlways`/non-`ThisDeviceOnly` means it survives in backups and is available when locked, which is a finding. Check the accessibility attribute, not just "it is in the Keychain."
- **Pasteboard and snapshot leaks**: sensitive fields that allow copy to the general pasteboard leak to any app; the automatic backgrounding snapshot can persist a screen showing tokens or PII to disk unless the app blurs/clears it. Both are real, often-missed privacy bugs.
- **URL scheme and universal link abuse**: a scheme handler that performs actions (log in, reset, pay, add account) based on untrusted parameters is a local/remote attack depending on who can trigger it. Universal links that carry auth tokens can leak via the web fallback. See `Deep-Links-and-IPC.md`.
- **WKWebView bridges**: enumerate `WKScriptMessageHandler` names and the native methods they reach; if the WebView loads attacker-influenced content, the bridge is an entry to native functionality.
- **The backend is usually the bigger prize**: proxy the app (bypass pinning) and test the API with the full web methodology (BOLA, mass assignment, auth). iOS client bugs are real but the API is where account takeover and data theft usually live (see `Mobile-API-and-Transport-Security.md`).

## What to watch for

- Deep dynamic analysis generally needs a jailbroken test device or a re-signed/gadget IPA; plan for that, and use your own device.
- The strong sandbox means many "local" attacks require the attacker to already have another app installed or physical access; frame impact by who can actually trigger it.
- A string in the binary might be a restricted/public key; confirm it works and matters.
- Keychain/biometric-gated secrets may be genuinely protected; distinguish them from weakly-stored ones.

## Hunting checklist

- Unzip the IPA; read `Info.plist` for URL schemes, associated domains, and ATS exceptions.
- Decrypt the binary, then `strings`/disassemble for secrets and endpoints.
- Dump Keychain, `NSUserDefaults`, cookies, and sandbox files; check Keychain accessibility classes.
- Test pasteboard and backgrounding-snapshot leakage.
- Trigger URL schemes and universal links against their handlers.
- Use objection/Frida to bypass pinning and jailbreak detection and to dump runtime secrets.
- Proxy and test the backend API with full web methodology.
