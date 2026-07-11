# Wazuh Decoder Workbench

<img width="2044" height="1210" alt="image" src="https://github.com/user-attachments/assets/2a748c35-082e-4c8a-a55b-b1b631035301" />


**Live demo: <https://abdeygodin.github.io/wazuh-decoder-workbench/>**

A single-file, zero-dependency web tool for security engineers who write custom **decoders** and **rules** for [Wazuh SIEM](https://wazuh.com/).

Paste a raw log line — get ready-to-use `decoder.xml`, `rules.xml`, and an instant `wazuh-logtest`-style check, all in your browser. Nothing is sent anywhere: the whole tool is one static HTML file.

## Features

- **Automatic format detection** — syslog RFC 3164 / RFC 5424 headers, JSON, key=value (FortiGate-style), CEF, and plain positional text.
- **Token recognition** — IPs, ports, users (by context: `user`, `from`, `port`…), emails, URLs, MD5/SHA1/SHA256 hashes, UUIDs, MAC addresses, timestamps. Each token gets a suggested standard Wazuh field name (`srcip`, `dstuser`, `srcport`, …).
- **Fully editable mapping** — toggle fields on/off and rename them; the XML regenerates live.
- **Two regex dialects**:
  - classic **OS_Regex** (works on every Wazuh version, with its quirks accounted for — e.g. a narrow `\w+` instead of the greedy `\.+` for quoted values);
  - **PCRE2** (`type="pcre2"`, Wazuh 4.2+) for more precise patterns.
- **Sensible decoder structure**:
  - syslog logs get a `program_name` parent decoder + a fields child decoder;
  - key=value logs get *one child decoder per field*, so the decoder does not break when the vendor reorders fields;
  - JSON logs use the built-in `JSON_Decoder` plugin instead of regex.
- **rules.xml scaffold** — a base `decoded_as` rule plus an example refined rule with a `<field>` condition taken from your actual log.
- **Built-in logtest simulator** — every pasted line is run through the generated decoders: extracted fields, matched rule ID and level, alert verdict.
- **AI assistant (optional)** — describe what you want in plain language (“rename field3 to hit_count, extract the interface names too”) and let an LLM rewrite the decoder/rules. Works with **local models** (Ollama, LM Studio — nothing leaves your machine) and OpenAI-compatible / Anthropic cloud APIs (bring your own key, stored only in your browser). Every AI reply is **verified by the built-in simulator** against your sample lines before it is shown; failed attempts are sent back to the model with the exact errors (up to 3 tries).
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
