# hpe-ilo-skill

An [OpenClaw](https://docs.openclaw.ai) skill that teaches your agent to
drive HPE iLO 4/5/6 servers through the Redfish REST API.

Everything runs through plain `curl` + `jq` against
`https://$ILO_HOST/redfish/v1/` — no SDKs, no vendored client libraries.

## Install

Drop `SKILL.md` into any OpenClaw skill root:

```bash
# personal — visible to all agents on this machine
mkdir -p ~/.openclaw/skills/hpe-ilo
curl -fsSL https://raw.githubusercontent.com/eelcoornd/hpe-ilo-skill/main/SKILL.md \
  -o ~/.openclaw/skills/hpe-ilo/SKILL.md

openclaw skills list | grep hpe-ilo
```

Or clone the repo directly:

```bash
git clone https://github.com/eelcoornd/hpe-ilo-skill.git ~/.openclaw/skills/hpe-ilo
```

## Configure

Set three env vars once per shell (or in `~/.openclaw/openclaw.json`
under `env`):

```bash
export ILO_HOST=ilo.example.com
export ILO_USER=Administrator
export ILO_PASS='...'
export ILO_INSECURE=1   # accept self-signed iLO certs
```

## Use

Ask the agent things like:

- "What's the power state of the iLO?"
- "Show me temperatures and fan speeds."
- "Reboot the server gracefully."
- "Set next boot to PXE and restart."
- "Mount http://nas/isos/debian.iso as virtual CD and boot from it."
- "Any critical events in the IML in the last day?"

## Compatibility

Works with iLO 4, iLO 5, iLO 6. The Redfish surface is largely identical;
where iLO-specific extensions exist they're under `.Oem.Hpe.*`.

## License

MIT
