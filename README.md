![ProtonRotatoR](https://github.com/user-attachments/assets/3f0b9e42-3b50-4a76-9939-bf8f72823880)

# ProtonRotatoR
Automate the rotation of ProtonVPN exit nodes via cli

Based on the interesting post here where it was done with Mullvad: http://opbible7nans45sg33cbyeiwqmlp5fu7lklu6jd6f3mivrjeqadco5yd.onion//opsec/mullvadvpn-daily-connect/index.html

## Prerequisites
```bash
wget https://repo.protonvpn.com/debian/dists/stable/main/binary-all/protonvpn-stable-release_1.0.8_all.deb
sudo dpkg -i ./protonvpn-stable-release_1.0.8_all.deb && sudo apt update
sudo apt install proton-vpn-cli

# Login
protonvpn signin [username]

# Enable killswitch and check it works
protonvpn config set kill-switch standard
protonvpn connect
protonvpn disconnect
```

## Manually do it with `protonvpn-cli`
```bash
# Region-agnostic version (uses Proton’s country list dynamically)
trap 'protonvpn disconnect >/dev/null 2>&1; break' INT TERM; while :; do c=$(protonvpn countries | awk 'NF>=2 {print $NF}' | grep -Ev 'NZ' | shuf -n1); echo "[rotate] connecting to $c"; protonvpn connect --country "$c"; sleep $((90 + (RANDOM % 300) - 150)); done

# Minimal rotating one-liner (Oceania, exclude NZ, 15 min ±10%)
trap 'protonvpn disconnect >/dev/null 2>&1; break' INT TERM; while :; do c=$(printf "AU FJ FM KI MH NC NF NR PG PW SB TO TV VU WS\n" | tr ' ' '\n' | grep -v NZ | shuf -n1); echo "[rotate] connecting to $c"; protonvpn connect --country "$c"; s=$((90 + (RANDOM % 181) - 90)); echo "[rotate] sleeping ${s}s"; sleep "$s"; done

# Europe-only, no RU/UK, human-ish jitter (12–18 min)
trap 'protonvpn disconnect >/dev/null 2>&1; break' INT TERM; EU="AL AD AM AT AZ BA BE BG BY CH CY CZ DE DK EE ES FI FR GE GR HR HU IE IS IT LT LU LV MD ME MK MT NL NO PL PT RO RS SE SI SK UA"; while :; do c=$(printf "%s\n" $EU | grep -Ev 'RU|UK' | shuf -n1); protonvpn connect --country "$c"; sleep $((90 + RANDOM % 361)); done

# With "don’t repeat last country" (still one-liner)
trap 'protonvpn disconnect >/dev/null 2>&1; break' INT TERM; last=""; while :; do c=$(protonvpn countries | awk 'NF>=2 {print $NF}' | shuf | grep -vx "$last" | head -n1); last="$c"; protonvpn connect --country "$c"; sleep $((90 + RANDOM % 180 - 90)); done
```

## Install the script
1. Clone the script
2. Put it somewhere in your PATH (like `/usr/local/bin/protonrotator`)
3. Run it

### Usage
```bash
  protonrotator [OPTIONS]

SELECTION              
  --mode region|country|city
      region  (default) : random country from --region allowlist (minus exclusions)
      country           : connect to --country (no randomisation)
      city              : connect to --country + --city (if supported by your Proton CLI)
                                            
  --region africa|americas|europe|asia|oceania|world
      Used when --mode=region (default: world)
                                            
  --country CODE                           
      Two-letter code (AU, US, etc). Overrides region selection.
                                                                                        
  --city NAME                                                                           
      City name (only applied if supported by your Proton CLI; script checks `protonvpn connect --help`)
      If --mode=city and --city is omitted, a random city is chosen from `protonvpn cities --country`.
                                            
EXCLUSIONS                  
  --exclude "CODES"                                                                     
      Exclude countries from selection. Accepts space OR comma separated lists:
        --exclude "US UK RU"
        --exclude "US,UK,RU"
        --exclude US,UK RU
                                                                                        
ROTATION CONTROL   
  --once                                                                                
      Pick/connect once and exit (default)
                                            
  --interval DURATION                                                                   
      Keep running and rotate every DURATION.
      If DURATION is a plain integer (e.g. 15), it is treated as MINUTES.
      Duration formats:                                                                 
        15         (minutes)
        900s       (seconds)      
        15m        (minutes)
        2h         (hours)
        1h30m      (combined)
        45s

  --jitter JITTER
      Add randomness to the interval sleep.
      JITTER can be:
        - Percent: 10%     (adds +/- 10% of the interval)
        - Duration: 90s    (adds +/- 90 seconds)
        - Duration: 5m     (adds +/- 5 minutes)

SHUTDOWN BEHAVIOR
  --on-exit leave|disconnect
      leave                : stop rotating and leave the current VPN connection up
      disconnect (default) : run `protonvpn disconnect` on shutdown

STATE / BEHAVIOUR
  --state-file PATH
      Store last-used country (default: /tmp/protonvpn-last-country)
  --no-avoid-repeat
      Allow picking the same country twice in a row

OUTPUT
  -n, --dry-run     Show what would run, do nothing
  -v, --verbose     Extra logs
  -h, --help        Show help
```

Examples:
```bash
# Europe rotation excluding UK + RU
protonrotator --region europe --exclude "UK RU" --once -n

# World rotation excluding US + UK
protonrotator --exclude "UK RU" --interval 30m -n

# Oceania except NZ
protonrotator --region oceania --exclude NZ --interval 15m -n
```

## Create a service that does it in the background
Create `/etc/systemd/system/protonrotator.service`

> [!CAUTION]
> Make sure the `ExecStart` command in this service is what you want to run; this is equivalent to the bash commands above

```bash
[Unit]
Description=ProtonVPN rotate exit node
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/protonrotator --region oceania --exclude NZ --interval 15m
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable it
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now protonrotator.service

# Logs
journalctl -u protonrotator.service -f
```

Edit it to change the options (change only the `ExecStart` line)
```bash
sudo systemctl edit --full protonrotator.service
```

Reload and restart
```bash
sudo systemctl daemon-reload
sudo systemctl restart protonrotator.service
```
