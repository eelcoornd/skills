---
name: hpe-ilo
description: Manage HPE iLO 4/5/6 servers via the Redfish REST API (power, health, thermals, logs, virtual media, firmware).
metadata: { "openclaw": { "requires": { "bins": ["curl", "jq"] }, "primaryEnv": "ILO_HOST" } }
---

# HPE iLO (Redfish)

Manage HPE ProLiant/Synergy/Apollo servers with iLO 4, iLO 5, or iLO 6
through the standard Redfish REST API. All operations are plain HTTPS +
Basic Auth (or a session token) against `https://$ILO_HOST/redfish/v1/`.

## When to use

- Power on/off/cycle/reset a server
- Read server health, temps, fan speeds, power draw
- Fetch iLO Event Log (IEL) or Integrated Management Log (IML)
- Mount / eject a virtual CD/DVD ISO
- Change one-time boot device (PXE, CD, HDD, BiosSetup)
- Kick off firmware updates
- List, add, or remove iLO local users

## Environment

Required:

- `ILO_HOST` — hostname or IP of the iLO (e.g. `ilo-node1.lan` or `10.0.0.42`)
- `ILO_USER` — iLO account username (default admin: `Administrator`)
- `ILO_PASS` — iLO account password

Optional:

- `ILO_INSECURE=1` — accept self-signed certs (adds `-k` to curl). Most iLOs
  ship with a self-signed cert; set this unless you've installed a real one.

Set once per shell:

```bash
export ILO_HOST=10.0.0.42 ILO_USER=Administrator ILO_PASS='...' ILO_INSECURE=1
```

## Cheat sheet — one-liners

Basic-auth curl helper (all examples below assume it's defined):

```bash
ilo() {
  local method=${1:-GET} path=$2; shift 2 || true
  local k=; [[ ${ILO_INSECURE:-0} = 1 ]] && k=-k
  curl -sSL $k -u "$ILO_USER:$ILO_PASS" \
    -H 'Content-Type: application/json' -H 'OData-Version: 4.0' \
    -X "$method" "https://$ILO_HOST$path" "$@"
}
```

### Sanity check

```bash
ilo GET /redfish/v1/ | jq '{Product, Vendor, RedfishVersion}'
```

### Power

```bash
# State
ilo GET /redfish/v1/Systems/1/ | jq '{PowerState, IndicatorLED, Model, SerialNumber}'

# Actions — pick one ResetType
ilo POST /redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
    -d '{"ResetType":"On"}'                # power on
    #   "ForceOff"          — hard off
    #   "GracefulShutdown"  — ACPI shutdown
    #   "GracefulRestart"   — ACPI reboot
    #   "ForceRestart"      — hard reboot
    #   "PushPowerButton"   — toggle power button
    #   "Nmi"               — non-maskable interrupt (crash dump)
```

### Health & inventory

```bash
ilo GET /redfish/v1/Systems/1/ | jq '.Status, .MemorySummary, .ProcessorSummary'
ilo GET /redfish/v1/Chassis/1/Thermal | jq '.Temperatures[] | {Name, ReadingCelsius, Status}'
ilo GET /redfish/v1/Chassis/1/Thermal | jq '.Fans[] | {Name, Reading, Status}'
ilo GET /redfish/v1/Chassis/1/Power   | jq '.PowerControl[] | {Name, PowerConsumedWatts, PowerCapacityWatts}'
```

### Logs

```bash
# iLO Event Log (iLO's own history)
ilo GET '/redfish/v1/Managers/1/LogServices/IEL/Entries?$top=20' \
  | jq '.Members[] | {Created, Severity, Message}'

# Integrated Management Log (server hardware events)
ilo GET '/redfish/v1/Systems/1/LogServices/IML/Entries?$top=20' \
  | jq '.Members[] | {Created, Severity, Message}'

# Clear a log
ilo POST /redfish/v1/Systems/1/LogServices/IML/Actions/LogService.ClearLog -d '{}'
```

### One-time boot override

```bash
ilo PATCH /redfish/v1/Systems/1/ -d '{
  "Boot": { "BootSourceOverrideEnabled": "Once",
            "BootSourceOverrideTarget":  "Pxe" }
}'
# Valid targets: None, Pxe, Floppy, Cd, Usb, Hdd, BiosSetup,
#                Utilities, Diags, UefiTarget, SDCard, UefiHttp
# Then reboot for it to take effect.
```

### Virtual media (mount an ISO from a URL)

```bash
# List virtual media devices (usually 1=Floppy, 2=CD)
ilo GET /redfish/v1/Managers/1/VirtualMedia | jq '.Members'

# Mount an ISO (must be HTTP/HTTPS reachable from the iLO)
ilo POST /redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.InsertMedia \
    -d '{"Image":"http://10.0.0.5/isos/debian-13.iso"}'

# Set it as the next boot device and boot once
ilo PATCH /redfish/v1/Managers/1/VirtualMedia/2 \
    -d '{"Oem":{"Hpe":{"BootOnNextServerReset": true}}}'

# Eject when done
ilo POST /redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.EjectMedia -d '{}'
```

### Firmware update

```bash
# Current versions
ilo GET /redfish/v1/UpdateService/FirmwareInventory | jq '.Members[]."@odata.id"'
ilo GET /redfish/v1/UpdateService/ | jq '{FirmwareVersion, State: .Oem.Hpe.State}'

# Push a signed .fwpkg / .bin from a reachable URL
ilo POST /redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
    -d '{"ImageURI":"http://10.0.0.5/fw/ilo5_305.bin"}'
```

### Users

```bash
ilo GET /redfish/v1/AccountService/Accounts \
  | jq '.Members[]."@odata.id"' -r \
  | xargs -I{} ilo GET {} \
  | jq '{UserName, RoleId, Enabled, Id}'

# Create a read-only monitoring user
ilo POST /redfish/v1/AccountService/Accounts -d '{
  "UserName":"monitor","Password":"'"$NEW_PASS"'","RoleId":"ReadOnly"
}'

# Delete
ilo DELETE /redfish/v1/AccountService/Accounts/5
```

### Session auth (nicer than Basic for repeated calls)

```bash
TOKEN=$(curl -sk -D - -o /dev/null \
  -H 'Content-Type: application/json' \
  -X POST "https://$ILO_HOST/redfish/v1/SessionService/Sessions" \
  -d "{\"UserName\":\"$ILO_USER\",\"Password\":\"$ILO_PASS\"}" \
  | awk -F': ' '/^X-Auth-Token/{print $2}' | tr -d '\r')

curl -sk -H "X-Auth-Token: $TOKEN" "https://$ILO_HOST/redfish/v1/Systems/1/" | jq .PowerState
# Logout when done:
curl -sk -H "X-Auth-Token: $TOKEN" -X DELETE "https://$ILO_HOST/redfish/v1/SessionService/Sessions/<id>"
```

## Discovery

If a path doesn't work, walk the collection — Redfish is self-describing:

```bash
ilo GET /redfish/v1/         | jq '.Systems, .Chassis, .Managers, .UpdateService'
ilo GET /redfish/v1/Systems/ | jq '.Members[]."@odata.id"'   # find the actual system id
```

Older iLO 4 uses `/redfish/v1/Systems/1/`; some iLO 5/6 use the same,
others expose UUIDs. Always start from `/redfish/v1/` and follow `@odata.id`
links — don't hardcode `1`.

## Safety

- **Destructive actions** (`ForceOff`, `ForceRestart`, `ClearLog`, `SimpleUpdate`,
  `InsertMedia` with reboot flag) can take the server down. Confirm with the
  user before running them.
- Firmware updates take several minutes and can render the iLO or BIOS
  unresponsive if interrupted. Never cancel mid-flight.
- Prefer `GracefulShutdown` / `GracefulRestart` over `ForceOff` when the OS
  is running.

## References

- HPE iLO 5 REST API reference: <https://hewlettpackard.github.io/ilo-rest-api-docs/ilo5/>
- Managing HPE Servers Using the RESTful API: <https://support.hpe.com/hpesc/public/docDisplay?docId=a00105236en_us>
- HPE developer portal: <https://developer.hpe.com/platform/ilo-restful-api/home/>
- DMTF Redfish spec: <https://www.dmtf.org/standards/redfish>
