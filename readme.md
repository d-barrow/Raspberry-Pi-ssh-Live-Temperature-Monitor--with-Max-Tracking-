# Raspberry Pi Live Temperature Monitor (with Max Tracking)

This script provides a live, color-coded temperature dashboard for your Raspberry Pi, showing current and maximum temperatures for all available sensors. It is designed for quick diagnostics via SSH and does **not** write any files to disk.



## Features

- **Live monitoring** of SoC and all thermal zones
- **Color-coded output** for easy status recognition (green/yellow/red)
- **Tracks maximum temperature** reached for each sensor (with timestamp)
- **No SD card wear** — runs entirely in RAM, no files created
- **SSH-friendly** — can be run directly from your Mac terminal

## Usage

From your Mac terminal, run:

```
cat << 'MON' | ssh haraldgeisler@raspberrypi.local bash
export TERM=xterm
# ANSI colors & thresholds
R=$'\033[0;31m'; G=$'\033[0;32m'; Y=$'\033[0;33m'; C=$'\033[0;36m'; X=$'\033[0m'
T1=60; T2=70; T3=80
# max record
declare -A MAX
declare -A MAX_TS
while true; do
  clear; ts=$(date "+%H:%M:%S");
  # table header
  printf "%-15s | %-7s | %s\n" "Sensor" "Now" "Max (when)"
  printf "%s\n" "----------------+---------+-------------------"

  # SoC sensor
  name=SoC
  val=$(vcgencmd measure_temp 2>/dev/null | grep -oE '[0-9.]+' | head -1)
  [ -z "$val" ] && val=0
  # update max
  if [ "$(awk -v v="$val" -v m="${MAX[$name]:-0}" 'BEGIN{print(v>m)}')" -eq 1 ]; then
    MAX[$name]="$val"
    MAX_TS[$name]="$ts"
  fi
  # colors
  col=$(awk -v v="$val" -v t1="$T1" -v t2="$T2" -v t3="$T3" 'BEGIN{if(v>t3)printf "\033[0;31m"; else if(v>t2)printf "\033[0;33m"; else printf "\033[0;32m"}')
  mcol=$(awk -v v="${MAX[$name]}" -v t1="$T1" -v t2="$T2" -v t3="$T3" 'BEGIN{if(v>t3)printf "\033[0;31m"; else if(v>t2)printf "\033[0;33m"; else printf "\033[0;32m"}')
  # compute age in seconds since max recorded
  age_secs=$(( $(date +%s) - $(date -d "${MAX_TS[$name]}" +%s) ))
  # include age in seconds ago
  printf "%-15s | %s%-7s%s | %s%-7s%s (%ss ago)\n" \
    "$name" "$col" "${val}C" "$X" "$mcol" "${MAX[$name]}C" "$X" "$age_secs"

  # other zones
  for zone in /sys/class/thermal/thermal_zone*; do
    name=$(cat "$zone/type" 2>/dev/null || basename "$zone")
    val=$(awk '{printf "%.1f", $1/1000}' "$zone/temp")
    [ "$(awk -v v="$val" -v m="${MAX[$name]:-0}" 'BEGIN{print(v>m)}')" -eq 1 ] && { MAX[$name]="$val"; MAX_TS[$name]="$ts"; }
    col=$(awk -v v="$val" -v t1="$T1" -v t2="$T2" -v t3="$T3" 'BEGIN{if(v>t3)printf "\033[0;31m"; else if(v>t2)printf "\033[0;33m"; else printf "\033[0;32m"}')
    mcol=$(awk -v v="${MAX[$name]}" -v t1="$T1" -v t2="$T2" -v t3="$T3" 'BEGIN{if(v>t3)printf "\033[0;31m"; else if(v>t2)printf "\033[0;33m"; else printf "\033[0;32m"}')
    # compute age in seconds since max recorded for this sensor
    zone_age=$(( $(date +%s) - $(date -d "${MAX_TS[$name]}" +%s) ))
    printf "%-15s | %s%-7s%s | %s%-7s%s (%ss ago)\n" \
      "$name" "$col" "${val}C" "$X" "$mcol" "${MAX[$name]}C" "$X" "$zone_age"
  done
  sleep 5
done      # close while loop
MON