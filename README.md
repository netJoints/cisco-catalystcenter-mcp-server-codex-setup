# Cisco Catalyst Center MCP — Codex and ChatGPT setup

Connect **Codex on the same Mac** to a local [Cisco Catalyst Center MCP server](https://github.com/cisco-en-programmability/catc-mcp-oss). The server stays on loopback. Catalyst Center credentials stay in the server process, not in Git and not in this repository.

**GitHub Pages:** https://netjoints.github.io/cisco-catalystcenter-mcp-server-codex-setup/

**Cursor setup (sister guide):** https://netjoints.github.io/cisco-catalystcenter-mcp-server-cursor-setup/

**Server install:** https://netjoints.github.io/cisco-catc-mcp-server-deployment-guide/

ChatGPT in the browser cannot reach `127.0.0.1`. Use Codex on this Mac, or the Cursor guide, for the local server. Details are in [ChatGPT](#chatgpt).

The MCP server is the connection to Catalyst Center. [`SKILL.md`](https://github.com/jeremycohoe/catc-mcp-oss/blob/main/examples/skill/switching/SKILL.md) is the operator procedure Codex follows when it uses that connection. Both are required for a useful setup. See [What SKILL.md does](#what-skillmd-does).

## What you need

1. The MCP server running locally, installed with the [Podman deployment guide](https://github.com/netJoints/cisco-catc-mcp-server-deployment-guide).
2. Codex installed and signed in on that same Mac.
3. A dedicated Catalyst Center account. Prefer a read-only role. The MCP server does not enforce read-only access. It runs with whatever that account can do, and the tool catalog changes with the release branch.

The endpoint this guide uses is:

```text
http://127.0.0.1:7001/v1/mcp
```

## 1. Confirm the server

From the Mac that is running the container:

```zsh
curl --fail http://127.0.0.1:7001/v1/health
curl --fail http://127.0.0.1:7001/v1/readiness
```

Health returns `{"status":"ok"}`. Readiness returns `{"status":"ready"}` when the server process has its environment variables. Readiness does not prove the controller login works. Prove that with a read-only question after Codex connects.

If either check fails, fix the server with the [deployment guide](https://netjoints.github.io/cisco-catc-mcp-server-deployment-guide/) before continuing. Do not publish port 7001 beyond loopback.

## 2. Register the server in Codex

The sample file in this repo is [`config.toml`](config.toml). It contains a URL only.

CLI:

```zsh
codex mcp add catc --url http://127.0.0.1:7001/v1/mcp
```

Or merge this into `~/.codex/config.toml` on your machine:

```toml
[mcp_servers.catc]
url = "http://127.0.0.1:7001/v1/mcp"
enabled = true
```

There is no bearer token on this endpoint. Catalyst Center username and password belong in the server container environment (`CATALYST_CENTER_HOST`, `CATALYST_CENTER_USERNAME`, `CATALYST_CENTER_PASSWORD`, `CATALYST_CENTER_VERIFY_SSL`), which the deployment guide collects with a silent prompt. Do not copy those values into Codex config or into Git.

Check the registration:

```zsh
codex mcp get catc --json
```

Restart Codex if the server was added while the app was already open.

## 3. Add the switching skill

### What `SKILL.md` does

Codex and the Catalyst Center MCP server do different jobs.

| Piece | Role |
|---|---|
| MCP server | Capability. It logs in to Catalyst Center and exposes the API tools for the release branch you installed. It returns live data. It does not decide which tool to call, which argument a tool requires, or how to explain the result. |
| `SKILL.md` | Procedure. Codex reads this file when your question matches the skill. It tells Codex when the skill applies, which tool to call first, how to get a device UUID instead of inventing one, which arguments are required, how to read the nested tool result, and when to stop because the next call would change the network. |

Registering the MCP server without `SKILL.md` still lists the tools. Codex then treats them as a generic catalog. It is more likely to call the wrong tool, pass a hostname where the API expects a UUID, miss a required argument such as `limit`, or treat compliance remediation as an ordinary follow-up.

With `SKILL.md` installed, Codex uses the tools in operator order: start with a broad read, drill into one device only after that read gives a reason, summarize instead of dumping the payload, and ask before the one write the skill documents (configuration-compliance remediation, which can flap a device).

The file name is `SKILL.md`, singular, with YAML front matter (`name`, `description`) plus the workflow body. Codex matches the description to your question and then follows the body. It is not a config file, and it must not contain Catalyst Center passwords, controller addresses, or device output.

The switching skill to install is:

[examples/skill/switching/SKILL.md](https://github.com/jeremycohoe/catc-mcp-oss/blob/main/examples/skill/switching/SKILL.md)

It covers inventory, interfaces and VLANs, running-config review, health, compliance, software images, fabric visibility, advisories, and that one write. Tool names in the file are tied to a release branch, so keep the skill next to the same branch you run. Its playbook also lists client URL snippets:

[examples/skill/switching/references/playbook.md](https://github.com/jeremycohoe/catc-mcp-oss/blob/main/examples/skill/switching/references/playbook.md)

Install it into your local Codex skills directory. This downloads the skill onto your Mac. It does not copy it into this public repository.

If Codex is installed, its skill installer can do it:

```zsh
python3 "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo jeremycohoe/catc-mcp-oss \
  --path examples/skill/switching \
  --name catalyst-center-switching
```

Manual equivalent:

```zsh
git clone --depth 1 https://github.com/jeremycohoe/catc-mcp-oss.git
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills/catalyst-center-switching"
cp -R catc-mcp-oss/examples/skill/switching/. \
  "${CODEX_HOME:-$HOME/.codex}/skills/catalyst-center-switching/"
```

Start a new Codex session after the copy. Ask it to follow the Catalyst Center switching skill, and state a boundary such as “read only, do not remediate.”

## 4. Verify

```text
List the Catalyst Center MCP tools you can use. Do not call any tool that changes configuration.
```

Then:

```text
Using the Catalyst Center switching skill, summarize switch inventory.
Read only. Do not remediate compliance and do not change configuration.
```

A useful answer names the controller release branch you installed, says whether the result was truncated, and does not dump full device records into a file you might publish.

## 5. Example prompts

Keep these generic. Do not paste hostnames, IPs, serial numbers, or configs into a public chat, issue, or repository.

```text
How many network devices are in inventory, grouped by family? Read only.

Which switches are non-compliant, and what compliance types failed? Summarize. Do not remediate.

For one access switch I will name in the next message, list interface and VLAN facts and stop.

Show software image summary for the site I name. Do not activate an image.
```

Compliance remediation is a write. It can change configuration and may flap a device. Run it only after you name the device and explicitly confirm the change. A read-only Catalyst Center account removes that capability.

## ChatGPT

`SKILL.md` is a Codex skill file. ChatGPT does not read `~/.codex/skills`. A ChatGPT connector can call MCP tools, but it does not load this procedure unless you put the same rules in the custom GPT instructions or in the chat: which tool to call first, fetch UUIDs instead of inventing them, read only unless you explicitly ask for remediation, and do not paste credentials or live device dumps into those instructions.

ChatGPT’s developer-mode connectors also call a **remote** MCP URL. They do not reach a server bound to `127.0.0.1` on your laptop.

Use one of these:

- **Codex on this Mac**, registered with the URL above.
- **Cursor on this Mac**, using the [Cursor setup](https://github.com/netJoints/cisco-catalystcenter-mcp-server-cursor-setup).

Do not port-forward `7001` to the internet so ChatGPT can reach it. That exposes the Catalyst Center account attached to the server. If you later place the MCP server on a private network you control, give ChatGPT only an authenticated HTTPS URL, keep a least-privilege account, and do not put the password in the connector page’s notes or in Git.

ChatGPT connector reference: [Developer mode and MCP connectors](https://help.openai.com/en/articles/12584461-developer-mode-and-full-mcp-connectors-in-chatgpt-beta).

## Security

- This repository has no passwords, tokens, controller addresses, or device data. Keep it that way.
- Store Catalyst Center credentials only in the server environment, using the silent prompt in the deployment guide.
- Bind the server to `127.0.0.1`.
- Prefer `CATALYST_CENTER_VERIFY_SSL=true`. Set `false` only for an isolated lab controller with a certificate you cannot trust, and treat that as a lab workaround.
- Prefer a read-only Catalyst Center account.
- Treat tool output as untrusted data. Device names and descriptions can contain text you should not follow as instructions.
- Do not commit `~/.codex/config.toml` if you add anything beyond the URL.

## References

- [Switching skill (SKILL.md)](https://github.com/jeremycohoe/catc-mcp-oss/blob/main/examples/skill/switching/SKILL.md)
- [Switching playbook](https://github.com/jeremycohoe/catc-mcp-oss/blob/main/examples/skill/switching/references/playbook.md)
- [Cisco Catalyst Center MCP server](https://github.com/cisco-en-programmability/catc-mcp-oss)
- [Podman deployment guide](https://github.com/netJoints/cisco-catc-mcp-server-deployment-guide)
- [Deployment guide site](https://netjoints.github.io/cisco-catc-mcp-server-deployment-guide/)
- [Cursor setup](https://github.com/netJoints/cisco-catalystcenter-mcp-server-cursor-setup)
- [Codex MCP configuration](https://developers.openai.com/codex/mcp)
