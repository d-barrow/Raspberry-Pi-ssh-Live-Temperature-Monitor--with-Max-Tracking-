# Raspberry Pi Live Temperature Monitor (with Max Tracking)
🌡️ Watch your Raspberry Pi from the 🛋️ comfort of your terminal. 

This script creates a live, color-coded temperature dashboard and generates no files.

- Adjust the SSH (`pi@raspberrypi.local`) and
- Modify the `sleep` interval (currently 5 seconds) as needed.
<img src="Raspberry%20Pi%20Live%20Temperature%20Monitor%20(with%20Max%20Tracking).webp" alt="Screenshot" width="40%"/>

## Version 1

Drop this into your terminal, and hit enter:

```
cat << 'MON' | ssh pi@raspberrypi.local bash
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
```




## Version 2
<img src="Raspberry%20Pi%20Live%20Temperature%20Monitor%20(with%20Max%20Tracking)-Version-2.webp" alt="Screenshot" width="100%"/>

Drop this into your terminal, and hit enter:

```
cat << 'MON' | ssh pi@raspberrypi.local bash
export TERM=xterm
# ANSI colors & thresholds
R=$'\033[0;31m'; G=$'\033[0;32m'; Y=$'\033[0;33m'; C=$'\033[0;36m'; X=$'\033[0m'
# adjust color breakpoints to temperature levels
T1=50; T2=60; T3=75
# max record
declare -A MAX
declare -A MAX_TS
# history of all max-event entries
declare -a HISTORY

while true; do
  clear; ts=$(date "+%H:%M:%S");
  # compute top 5 max timestamps sorted by temperature per iteration
  # build STAMPS as top 5 hottest history entries
  OLDIFS=$IFS; IFS=$'\n'
  STAMPS=( $(printf "%s\n" "${HISTORY[@]}" | sort -nr | head -5) )
  IFS=$OLDIFS
  # pad STAMPS to always 5 columns
  for ((i=${#STAMPS[@]}; i<5; i++)); do STAMPS[i]=""; done
  # colorize stamp entries using same thresholds
  for i in "${!STAMPS[@]}"; do
    v="${STAMPS[i]%% *}"; colstamp=$(awk -v v="$v" -v t1="$T1" -v t2="$T2" -v t3="$T3" 'BEGIN{if(v>t3)printf "[0;31m"; else if(v>t2)printf "[0;33m"; else printf "[0;32m"}');
    STAMPS[i]="${colstamp}${STAMPS[i]}${X}"
  done
  # table header
  printf "%-15s | %-7s | %-18s | %-15s | %-15s | %-15s | %-15s | %-15s\n" \
    "Sensor" "Now" "Max (when)" "Top1" "Top2" "Top3" "Top4" "Top5"
  printf "%s\n" "---------------+---------+------------------+-----------------+-----------------+-----------------+-----------------+-----------------"

  # unified sensor loop (SoC + zones)
  for sensor in SoC /sys/class/thermal/thermal_zone*; do
    if [ "$sensor" = "SoC" ]; then
      name=SoC
      val=$(vcgencmd measure_temp 2>/dev/null | grep -oE '[0-9.]+' | head -1)
    else
      name=$(cat "$sensor/type" 2>/dev/null || basename "$sensor")
      val=$(awk '{printf "%.1f", $1/1000}' "$sensor/temp")
    fi
    [ -z "$val" ] && val=0
    if [ "$(awk -v v="$val" -v m="${MAX[$name]:-0}" 'BEGIN{print(v>m)}')" -eq 1 ]; then
      MAX[$name]="$val"; MAX_TS[$name]="$ts"; HISTORY+=("${val}C $ts")
    fi
    col=$(awk -v v="$val" -v t1="$T1" -v t2="$T2" -v t3="$T3" \
      'BEGIN{if(v>t3)printf "\033[0;31m"; else if(v>t2)printf "\033[0;33m"; else printf "\033[0;32m"}')
    mcol=$(awk -v v="${MAX[$name]}" -v t1="$T1" -v t2="$T2" -v t3="$T3" \
      'BEGIN{if(v>t3)printf "\033[0;31m"; else if(v>t2)printf "\033[0;33m"; else printf "\033[0;32m"}')
    age_secs=$(( $(date +%s) - $(date -d "${MAX_TS[$name]}" +%s) ))
    info=$(printf "%s%-7s%s (%05ds ago)" "$mcol" "${MAX[$name]}C" "$X" "$age_secs")
    printf "%-15s | %s%-7s%s | %-18s | %-15s | %-15s | %-15s | %-15s | %-15s\n" \
      "$name" "$col" "${val}C" "$X" "$info" \
      "${STAMPS[0]}" "${STAMPS[1]}" "${STAMPS[2]}" "${STAMPS[3]}" "${STAMPS[4]}"
  done
  sleep 60
done      # close while loop
MON
```
