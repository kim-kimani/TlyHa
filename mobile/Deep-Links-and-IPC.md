# Deep Links and IPC

Deep links and inter-process communication are how other apps (and sometimes web pages) reach into an app. They are a remote-ish attack surface that bypasses the app's own UI and trust assumptions. The bugs are handlers that trust their input, links that leak tokens, and components that act as confused deputies for whoever calls them.

## The mental model

An app advertises entry points: custom URL schemes (`myapp://`), Android App Links / iOS Universal Links (`https://app.target.com/...` routed into the app), and IPC components (Android intents, content providers, services; iOS `NSUserActivity`, extensions). Anything that can craft a link or an intent, another installed app, a web page, a QR code, an NFC tag, can drive these entry points. The vulnerability is when the handler performs sensitive actions, trusts attacker-controlled parameters, or forwards the request to internal components.

## Where it hides / tell-tales

- **Scheme handlers that act on parameters**: `myapp://reset?token=...`, `myapp://addAccount?server=...`, `myapp://pay?to=...&amount=...`. If the handler executes the action without re-authenticating or validating origin, any app or web page that opens the link triggers it.
- **Token delivery over links**: OAuth/magic-link/reset tokens delivered to a custom scheme or universal link. If a custom scheme carries the token, a malicious app that also registers that scheme (Android allows scheme collisions) can intercept it.
- **Unvalidated redirects into WebViews**: a deep link whose parameter becomes a URL loaded in the app's WebView (`myapp://open?url=https://evil.tld`), giving the attacker content inside the app's origin and bridges (see `Android-App-Testing.md`/`iOS-App-Testing.md`).
- **Android intent surface**: exported activities/services/receivers invoked with crafted extras; `intent://` URLs from a web page that launch components; intent-redirection where the app forwards an attacker-supplied nested intent to a private component.
- **Universal/App Link verification gaps**: if domain verification (`assetlinks.json` on Android, `apple-app-site-association` on iOS) is misconfigured, links may fall back to the browser (leaking tokens to the web) or be claimable by another app.

## Real material

Triggering handlers:

```
# Android: launch a deep link via adb (simulates another app / browser)
adb shell am start -W -a android.intent.action.VIEW -d "myapp://reset?user=victim&token=X"
# Android: intent:// from a web page the victim visits
<a href="intent://pay?to=attacker&amount=1000#Intent;scheme=myapp;package=com.target.app;end">go</a>
# iOS: open a scheme
open "myapp://addAccount?server=evil.tld&autoconfirm=1"
```

Scheme-collision token theft (Android, two apps register `myapp://callback`): the malicious app may receive the OAuth code/token if the system lets the user pick or if ordering favors it. Universal/App Links are the fix precisely because they are domain-verified; a custom scheme for token delivery is the tell.

Intent redirection (confused deputy) on Android:

```java
// vulnerable: app forwards a nested intent to a private component
Intent inner = getIntent().getParcelableExtra("forward");
startActivity(inner);   // attacker supplies 'forward' targeting a non-exported internal activity
```

WebView-loading deep link to in-app content injection:

```
myapp://webview?url=https://evil.tld/page_with_bridge_calls.html
```

## Beyond the basics

- **Enumerate every entry point first**: parse the manifest (`<data android:scheme=...>`, exported components) / `Info.plist` (`CFBundleURLSchemes`, associated domains) to list schemes, hosts, and paths. Each is a handler to test for trust and action.
- **Map handler to action to authorization**: for each deep link, determine what it does and whether it re-checks identity and validates parameters. The dangerous ones perform state changes (login, link account, reset, pay, change settings) or load content. A deep link that logs you in or links an account without confirmation is account takeover when the victim opens an attacker link.
- **Token interception via scheme collision and web fallback**: prefer-universal-links reasoning: if auth tokens ride a custom scheme, a malicious app can register the same scheme (Android) to steal them; if a universal link falls back to the browser due to verification failure, the token leaks to a web context (and via `Referer`). Both are real takeover paths.
- **Intent redirection and PendingIntent**: covered in `Android-App-Testing.md`; from the deep-link angle, an `intent://` URL or a forwarded intent reaches internal, non-exported components, bypassing the exported-surface boundary. Mutable PendingIntents given to other components are hijackable.
- **Parameter injection through links**: deep-link parameters flow into WebView URLs (XSS/content injection), into SQL (if they reach a provider query), into file paths (traversal), or into further intents. Treat every link parameter as untrusted input to whatever sink it reaches.
- **Cross-app data exposure**: content providers and exported services reachable via IPC can leak or modify data for any app on the device; combine with deep links for a full local attack chain.
- **QR/NFC/clipboard as link delivery**: the "attacker who sends a link" can be a QR code, an NFC tag, a chat message, or a clipboard entry, widening who can trigger a vulnerable handler.

## What to watch for

- Custom-scheme attacks often require the victim to open a crafted link or have a malicious app installed; frame impact by the delivery requirement, but token interception and one-click account linking are frequently high severity.
- Android App Links / iOS Universal Links with correct domain verification are much safer than custom schemes; confirm whether verification is actually enforced before assuming interception works.
- A deep link that only opens a benign screen is low; the finding is when it acts or loads attacker content.
- Test on your own device and accounts; do not phish real users beyond controlled proof.

## Hunting checklist

- Enumerate all URL schemes, universal/app links, and exported IPC components from the manifest/plist.
- For each handler, determine the action and whether it re-authenticates and validates input.
- Test state-changing deep links (login, link, reset, pay) for one-click abuse via an attacker link.
- Check whether auth tokens ride a custom scheme (interceptable) and whether universal links fall back to the browser.
- Test intent redirection and PendingIntent hijacking to reach internal components (Android).
- Trace deep-link parameters into WebView/SQL/file/intent sinks for injection.
