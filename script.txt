#!/bin/vbash
source /opt/vyatta/etc/functions/script-template

ARRAY=()
configure
NUMBER=$(show traffic-control smart-queue QoS download rate | tr -dc '0-9')
echo "Detected download rate of $NUMBER"
download=$NUMBER
maximumspeed=60
minimumspeed=10

while sleep 5; do
  t="$(ping -c 1 8.8.8.8 | sed -ne '/.*time=/{;s///;s/\..*//;p;}')"
  t=$(echo $t | tr -dc '0-9')
  if [ "$t" -gt 60 ]; then
    if [ "$download" -gt $minimumspeed ]; then
      download=$((download*80/100))
      download="${download}mbit"
      set traffic-control smart-queue QoS download rate $download
      commit
      echo "Decreasing your SQM download rate to $download"
      download=$(echo $download | tr -dc '0-9')
    else
      echo "Hit minimum broadband target"
    fi
  else
    echo "Your ping is $t"
    ARRAY+=($t)
    arraylength=${#ARRAY[@]}
    if [ "$arraylength" -gt 5 ]; then
      ARRAY=("${ARRAY[@]:1}")
      printf '%s\n' "${ARRAY[@]}"
      total=0
      for n in ${ARRAY[@]}
      do
        (( total += n ))
      done
      echo "Your total ping over 5 hops is: $total"
      total=$((total/5))
      if [ "$total" -lt 40 ]; then
        if [ "$download" -lt $maximumspeed ]; then
          download=$((download*120/100))
          download="${download}mbit"
          set traffic-control smart-queue QoS download rate $download
          commit
          echo "Your average ping over 5 hops is: $total ms"
          echo "Increasing your SQM download rate to $download"
	  ARRAY=()
          download=$(echo $download | tr -dc '0-9')
        else
          echo "Hit maximum broadband target"
        fi
      else
        echo "Your average ping over 5 hops is: $total ms"
      fi
    else
      printf '%s\n' "${ARRAY[@]}"
    fi
  fi
done