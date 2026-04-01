# homeassistant_foxess
Script to set a FoxESS Inverter to start/stop charging over WiFi without a modbus

## FoxESS and Octopus Energy in Home Assistant without Modbus
I have Octopus Intelligent Go for my electicity tariff, which has a cheap rate at night, but can also have a variable rate during the day if Octopus deems that certain times are good for charging my EV car. I have a FoxESS solar generation system with batteries. However there are times when charging my car that it drains my home batteries, and I wanted to ensure the car charging came from the grid instead.

I found Home Assistant and got an instance setup in a VirtualBox VM on an Intel NUC. I won't go through all of the details of that, but there are integrations for Octopus and for FoxESS. However, most of the advice for controlling the inverter involves installing a modbus device on your inverter. That isn't really practical for my outdoor inverter (I couldn't run a huge ethernet cable, and had no power nearby to install a wifi dongle).

However, the [Open API from FoxESS](https://www.foxesscloud.com/public/i18n/en/OpenApiDocument.html#set20the20time20segment20information0a3ca20id3dset20the20time20segment20information7193e203ca3e) allows for adjusting settings remotely.

And I have used curl from a shell command within Home Assistant to set the SOC as part of a schedule (this is a v2, since my v1 approach was broken by an API change that no longer allows direct setting of the SOC), and then create some automations based on Octopus Energy's Intelligent Go rates. As a Home Assistant n00b, I'll call out the steps I went through.

### 1) In the File Editor addon, create the shell script below.

I put mine in a new folder: /shell_scripts/foxess_set_soc.sh.

Be sure to replace the APIKEY and SN values with your own.

Note that this needs to go in the config folder, which is wherever your configuration.yaml is, which for me in File Editor is /homeassistant/.

```
#!/bin/bash
# FoxESS H1-G2 MinSoc Setting Script
# Uses Scheduler V1 API (required for H1-G2 with scheduler enabled)
# Usage: ./foxess_set_soc.sh <min_soc_on_grid>
# Example: ./foxess_set_soc.sh 20

if [ $# -ne 1 ]; then
    echo "Usage: $0 <min_soc_on_grid>"
    echo "Example: $0 20  (sets battery reserve to 20%)"
    exit 1
fi

MIN_SOC_GRID="$1"

# Validate (FoxESS requires 10-100 range)
if ! [[ "$MIN_SOC_GRID" =~ ^[0-9]+$ ]] || [ "$MIN_SOC_GRID" -lt 10 ] || [ "$MIN_SOC_GRID" -gt 100 ]; then
    echo "Error: MIN_SOC_GRID must be between 10 and 100"
    exit 1
fi

# Configuration
APIKEY="MY_KEY_HERE"
SN="MY_INVERTER_SERIAL"
BASE_URL="https://www.foxesscloud.com"
URL_PATH="/op/v1/device/scheduler/enable"

# Generate timestamp and signature
TIMESTAMP=$(python3 -c "import time; print(int(time.time() * 1000))")
SIGN_STRING="${URL_PATH}\\r\\n${APIKEY}\\r\\n${TIMESTAMP}"
SIGNATURE=$(printf "%s" "$SIGN_STRING" | md5sum | awk '{print $1}')

# Create JSON body with 24-hour SelfUse schedule
JSON_BODY=$(cat <<EOF
{
  "deviceSN": "${SN}",
  "groups": [
    {
      "enable": 1,
      "startHour": 0,
      "startMinute": 0,
      "endHour": 23,
      "endMinute": 59,
      "workMode": "SelfUse",
      "minSocOnGrid": ${MIN_SOC_GRID},
      "fdSoc": 100,
      "fdPwr": 0,
      "maxSoc": 100
    }
  ]
}
EOF
)

# Make API call
RESPONSE=$(curl -s -w "\n%{http_code}" -X POST "${BASE_URL}${URL_PATH}" \
  -H "Content-Type: application/json" \
  -H "token: ${APIKEY}" \
  -H "signature: ${SIGNATURE}" \
  -H "timestamp: ${TIMESTAMP}" \
  -H "lang: en" \
  -d "$JSON_BODY")

# Parse response
HTTP_CODE=$(echo "$RESPONSE" | tail -n1)
BODY=$(echo "$RESPONSE" | sed '$d')

# Check result
if [ "$HTTP_CODE" -eq 200 ]; then
    ERRNO=$(echo "$BODY" | grep -o '"errno":[0-9]*' | grep -o '[0-9]*' || echo "999")
    if [ "$ERRNO" = "0" ]; then
        echo "✓ Battery reserve (MinSocOnGrid) set to ${MIN_SOC_GRID}%"
        exit 0
    else
        echo "✗ API Error $ERRNO: $BODY"
        exit 1
    fi
else
    echo "✗ HTTP Error $HTTP_CODE"
    exit 1
fi
```

### 2) Using the Advanced SSH & Web Terminal addon you need to do a few things:

- Disable protected mode in the addon's settings
- Type this at the terminal prompt so we can test the script: **docker exec -it homeassistant bash**
- Then navigate to the shell_scripts folder and make the script executable (once): **chmod +x foxess_set_soc.sh**
- Then any time you paste in code in File Editor, run this in terminal to fix the line endings: **dos2unix foxess_set_soc.sh**
- Then you can run it to test it: **./foxess_set_soc.sh 100**
- You can verify in the FoxESSCloud app or website that the battery reserve capacity and system min soc were changed to 100. You can run the script again to change the numbers back again.


### 3) In File Editor, add this shell command to your configuration.yaml:

```
# Custom shell command script to call FoxESS via web to set min soc of batteries shell_command:
 foxess_set_soc: bash shell_scripts/foxess_set_soc.sh {{ soc_on_grid }}
```


### 4) In the Home Assistant UI (Settings > Automations > Scripts), create two scripts:

To start charging:
```
data: soc_on_grid: 100
action: shell_command.foxess_set_soc
```
To stop charging:
```
data: soc_on_grid: 20
action: shell_command.foxess_set_soc
```


### 5) Create an automation to call the script, e.g. when electricity rate is cheap

Settings > Automations & Scenes > Automations

For example, I use Octopus Intelligent Go, so using the current rate electricity entity as the trigger, when is is <0.15 I have an action to call my 'start charging' script above and notify my phone, with another automation when >0.15 to call 'stop charging'.

I'm sure there are some others I could add for free sessions too, but I've not got that far.

<img width="2527" height="961" alt="Screenshot 2026-04-01 212648" src="https://github.com/user-attachments/assets/76dc502f-0895-4901-87de-54a382b4fd07" />


### 6) Restart Home Assistant

You're done! This all relies on FoxCloud, but avoids the need for a modbus to be installed.
