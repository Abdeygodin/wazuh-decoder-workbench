# Wazuh Decoder Workbench

<img width="2044" height="1210" alt="image" src="https://github.com/user-attachments/assets/2a748c35-082e-4c8a-a55b-b1b631035301" />


**Live demo: <https://abdeygodin.github.io/wazuh-decoder-workbench/>**

A single-file, zero-dependency web tool for security engineers who write custom **decoders** and **rules** for [Wazuh SIEM](https://wazuh.com/).

Paste a raw log line — get ready-to-use `decoder.xml`, `rules.xml`, and an instant `wazuh-logtest`-style check, all in your browser. Nothing is sent anywhere: the whole tool is one static HTML file.

## Features

- **Automatic format detection** — JSON, key=value (FortiGate-style), CEF, and plain positional text, after the same pre-decoding Wazuh does: a port of its pre-decoder (`OS_CleanMSG`) strips the timestamp formats Wazuh knows (syslog, ISO 8601, proftpd, xferlog, snort, suricata, apache, squid, macOS) and takes the hostname and `program_name` the way Wazuh does — so `%ASA-…` or `firewall,info` are not program names, and RFC 5424 is not parsed (Wazuh does not parse it either). A leading `<PRI>` is removed, as Wazuh's syslog listener does.
- **Token recognition** — IPs, ports, users (by context: `user`, `from`, `port`…), emails, URLs, MD5/SHA1/SHA256 hashes, UUIDs, MAC addresses, timestamps. Each token gets a suggested standard Wazuh field name (`srcip`, `dstuser`, `srcport`, …).
- **Fully editable mapping** — toggle fields on/off and rename them; the XML regenerates live.
- **Two regex dialects**:
  - classic **OS_Regex** (works on every Wazuh version, with its quirks accounted for — e.g. a narrow `\w+` instead of the greedy `\.+` for quoted values);
  - **PCRE2** (`type="pcre2"`, Wazuh 4.2+) for more precise patterns.
- **Sensible decoder structure**:
  - syslog logs get a `program_name` parent decoder + a fields child decoder;
  - key=value logs get *one child decoder per field*, all sharing one name (Wazuh sibling decoders), so every field is extracted and the decoder does not break when the vendor reorders fields;
  - several message variants get one child decoder each, told apart by its own `<prematch>` (Wazuh runs only the first matching child);
  - JSON logs use the built-in `JSON_Decoder` plugin instead of regex.
- **rules.xml scaffold** — a base `decoded_as` rule plus an example refined rule with a condition on a field value taken from your actual log (static fields such as `action` or `srcip` get their own tag — Wazuh refuses `<field name="action">`).
- **Stock decoder check** — every line is also matched against the parent decoders of the stock Wazuh ruleset (embedded in `index.html`, currently 4.14.8). Stock decoders load before `etc/decoders` and Wazuh uses the first parent decoder that matches, so if one takes the line, the generated decoder would never run. The page says which one, links to its file and lists the options: rules on the stock decoder, a child decoder under it, or replacing its file. Checked against a real `wazuh-logtest` 4.14.8 on 63 lines: all agree. To embed another ruleset version: `python3 tools/build-stock-decoders.py --tag v4.x.y`.
- **Built-in logtest simulator** — every pasted line is run through the generated decoders: extracted fields, matched rule ID and level, alert verdict.
- **AI assistant (optional)** — describe what you want in plain language (“rename field3 to hit_count, extract the interface names too”) and let an LLM rewrite the decoder/rules. Works with **local models** (Ollama, LM Studio — nothing leaves your machine) and OpenAI-compatible / Anthropic cloud APIs (bring your own key, stored only in your browser). Every AI reply is checked before it is shown: the decoders are **run through the built-in simulator** against your sample lines, and the rules are **limited to your decoders** — IDs must come from your block (first rule ID + 99), every rule must hang off your own decoders or rules, and anything that could change or silence stock rules (`overwrite`, `if_group`, `if_level`, references to stock rule IDs) is rejected. Log lines are passed to the model as untrusted data. Failed attempts are sent back to the model with the exact errors (up to 3 tries). Rule conditions themselves are not simulated — confirm them with `wazuh-logtest`.
- **Deployment cheat-sheet** — file paths, custom rule ID range (100000–120000), `wazuh-logtest`, restart command.

## Usage

Open the [live demo](https://abdeygodin.github.io/wazuh-decoder-workbench/) or just open `index.html` in any modern browser. That's it.

1. Paste one or more log lines (the first line defines the template; every line is run through the test).
2. Click **Analyze**.
3. Adjust the field mapping and generation settings.
4. Copy `decoder.xml` → `/var/ossec/etc/decoders/local_decoder.xml` and `rules.xml` → `/var/ossec/etc/rules/local_rules.xml` on your manager.
5. Verify with `/var/ossec/bin/wazuh-logtest`, then `systemctl restart wazuh-manager`.

> **Note:** the built-in test is an approximation of the Wazuh engine. Always do the final check with the real `wazuh-logtest`.

## AI assistant setup

- **Ollama (local, recommended):** allow the page's origin, then fully restart Ollama (quit from the tray, start again):
  - Windows (PowerShell): `setx OLLAMA_ORIGINS "https://abdeygodin.github.io"`
  - Linux/macOS: `OLLAMA_ORIGINS=https://abdeygodin.github.io ollama serve`
  - for a locally opened `index.html` use `OLLAMA_ORIGINS=*` (the file origin is `null`)

  Pick “Ollama (local)”, click *Fetch models*, choose a model. Models around 30B+ (or MoE like qwen3 30B-A3B) handle decoder generation noticeably better than 7B ones — the tool detects failures honestly, weak models just get rejected by the verifier more often. If *Fetch models* fails, the app shows this checklist right in the UI.
- **LM Studio (local):** enable the local server (default port 1234) with CORS on.
- **OpenAI-compatible / Anthropic:** paste your base URL and API key. The key never leaves your browser except to call the API you configured.

If your browser blocks calls from the https demo page to `http://localhost`, download `index.html` and open it locally — the tool is fully self-contained.

## Built-in examples

- sshd (syslog, failed password)
- FortiGate traffic log (key=value)
- JSON application log
- CEF (Trend Micro Deep Security)
- Custom application log (positional text)

## Sample regression logs

An anonymized sample corpus lives in `samples/log-samples.json`: FortiGate, Cisco ASA, MikroTik, nginx, Postfix and
Windows Sysmon lines. `test.html` loads the real `index.html` in a hidden frame and runs every sample the way the
Analyze button does — generate decoders, feed the line to the logtest simulator — in both regex dialects.

Each sample has:

- `expected_fields` — what the workbench already extracts correctly; any mismatch fails the page;
- `known_gaps` — what an analyst would expect but the workbench does not do yet, with a note why. They are listed,
  not failed; when one starts to pass, the page asks to move it to `expected_fields`. A gap can be limited to one
  dialect (`"dialect": "os"`); `"expected": null` means the field must not be extracted;
- `stock_decoder` — the stock decoder that takes the line in a real Wazuh with the ruleset version embedded in `index.html` (`null`: none); the page checks the workbench predicts the same;
- `wazuh` — notes from a real `wazuh-logtest` run.

Open `test.html` from a local web server (the frame is not accessible from `file://`):

```bash
python -m http.server 8000
```

Then visit <http://localhost:8000/test.html>. The page title starts with `PASS` or `FAIL`, and `window.regressionResult`
holds the counts for headless runs.

`samples/predecoder-cases.json` holds 102 lines with the `program_name` and message a real `wazuh-logtest` 4.14.8 reported for them; the page checks the workbench's pre-decoder gives the same.

Adding a sample: paste an anonymized line (RFC 5737 IPs, `example.test` domains, fictional users), put the fields the
workbench gets right into `expected_fields` and the rest into `known_gaps`.
