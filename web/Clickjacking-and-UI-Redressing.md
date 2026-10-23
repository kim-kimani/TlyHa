# Clickjacking and UI Redressing

Clickjacking is dismissed as low until it lands on a one-click state change that matters: account deletion, OAuth consent, fund transfer, permission grant, MFA disable. The trick is that the victim supplies their own session and their own click, so it defeats CSRF tokens entirely. The depth is in the variants beyond a single framed button.

## The mental model

You frame the real application invisibly over your own page and position a lure (a button, a game, a captcha) so the victim's click actually lands on the target's genuine control. Because it is the victim's browser with the victim's cookies clicking the victim's real button, server-side CSRF tokens are present and valid. The only defense is the target refusing to be framed (`X-Frame-Options` / CSP `frame-ancestors`), so the bug exists wherever a sensitive action is framable.

## Where it hides / tell-tales

- **Framable sensitive actions**: any state-changing page that lacks `X-Frame-Options: DENY/SAMEORIGIN` or CSP `frame-ancestors`. Check the headers on the actual sensitive endpoint, not the home page; protection is often inconsistent across routes.
- **Single-click high-impact controls**: "Delete account," "Authorize app" (OAuth consent), "Add funds/payee," "Disable 2FA," "Make public," "Grant admin," "Accept invite." One click equals impact.
- **SPA controls that still render in a frame**: client-side apps that do not set framing headers at the document level.
- **`frame-ancestors` missing while `X-Frame-Options` present (or vice versa)**: old `X-Frame-Options` does not support allow-lists well; a misconfigured CSP may still permit framing from an attacker origin.

## Real material

Basic invisible-overlay PoC:

```html
<style>
  iframe { position:absolute; top:0; left:0; width:1000px; height:800px;
           opacity:0.0001; z-index:2; }
  #lure  { position:absolute; top:410px; left:300px; z-index:1; }
</style>
<div id="lure"><button>Click to win</button></div>
<iframe src="https://app.target.com/settings/delete-account"></iframe>
<!-- align the invisible "Delete" button under the visible lure; victim clicks through -->
```

## Beyond the basics

- **Drag-and-drop and multi-step (classic for data exfiltration)**: lure the victim into dragging a token/text from the framed app into an attacker field, or through a sequence of aligned clicks to complete a multi-step flow (confirm dialogs included). Useful when one click is not enough.
- **Double-click hijacking / cursorjacking**: trick timing so a legitimate first click dismisses something and a second lands on the target, or spoof the cursor position with a fake pointer so the victim aims wrong.
- **Likejacking / tapjacking equivalents**: on mobile web, framing plus touch events; native Android tapjacking overlays a view over another app (see the mobile files).
- **OAuth consent clickjacking**: frame the provider's "Authorize" page; a single click grants an attacker app access to the victim's account. High impact because it chains to full account access, and consent pages are sometimes framable.
- **Combine with other bugs**: clickjacking to trigger a self-XSS form submission (making self-XSS exploitable), to flip a setting that enables a second bug, or to perform the click a `SameSite` cookie would have blocked via normal CSRF.
- **Partial-protection bypasses**: `X-Frame-Options: ALLOW-FROM` is deprecated and ignored by modern browsers (so relying on it may leave framing open); a CSP `frame-ancestors` that includes a wildcard, a takeoverable subdomain (see `Subdomain-Takeover.md`), or an attacker-influenced origin is bypassable.

## What to watch for

- Many programs rate clickjacking as low/informational unless you show a real sensitive-action impact; always demonstrate it on a meaningful one-click control.
- Confirm the sensitive endpoint itself is framable (check its headers), not just a harmless page.
- `SameSite=Lax/Strict` does not stop clickjacking (the request originates from the real framed app), which is exactly why it is still relevant.
- Do not perform irreversible actions on real victims; prove with your own account.

## Hunting checklist

- Enumerate sensitive one-click actions; check each for `X-Frame-Options` and CSP `frame-ancestors`.
- Build an overlay PoC aligning the lure over the real control.
- Consider drag-drop, double-click, and multi-step variants where one click is insufficient.
- Target OAuth consent and settings toggles for highest impact.
- Chain with self-XSS or setting-flips to upgrade severity.
