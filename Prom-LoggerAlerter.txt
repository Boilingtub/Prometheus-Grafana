#!/bin/bash

 site='http://localhost:9090/api/v1/query?query='
 CPUQuery='avg(irate(node_cpu_seconds_total[10s]))*100'
 RAMQuery='100*(1-((avg_over_time(node_memory_MemFree_bytes[10s])+%2B+avg_over_time(node_memory_Cached_bytes[10s])+%2B+avg_over_time(node_memory_Buffers_bytes[10s]))/avg_over_time(node_memory_MemTotal_bytes[10s])))'
 HDDQuery='100-((node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"})/node_filesystem_size_bytes{mountpoint="/",fstype!="rootfs"})*100'
 HDCQuery='(node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"})/1000000000'
#FUNCTION /START

  ExtractData () {

    status=$(echo $1 | cut -d ',' -f1)
    status=${status:1}

    pos1=$(echo $1 | grep -bo 'value')
    pos1=${pos1%:*}
    value=$(echo $1 | cut -c $pos1-${#1})
    value=$(echo $value | cut -d '}' -f1) 
    
    Output="$status'\n'$value"
    eval "$2=$Output"
  }
#####################################################################2

  CheckHardDrive () {
    ColBoldRed='\033[1;31m'
    ColBoldBlack='\033[1;30m'   
    ColWhite='\033[0;37m' 
    ColBoldYellow='\033[1;33m'
    UnderlineRed='\033[4;31m'  
    ColReset='\033[0m'
    BackRed='\033[41m'
    HDDThreshold=80
    HDDValue=$(echo $1 | cut -d ',' -f2-)
    HDDValue=${HDDValue%?}
    if (( $( echo "$HDDValue > $HDDThreshold" | bc -l) ))
    then
    echo -e "${ColBoldRed}ALERT ! ALERT ! ALERT ! ALERT ! ALERT ! "
    echo -e "${BackRed}${ColBoldYellow}$HDDValue%${ColReset}${ColBoldYellow} of Hard-Drive is used"
    echo -e "${ColBoldRed}ALERT ! ALERT ! ALERT ! ALERT ! ALERT ! ${ColReset} "
    echo -e ""
    
    EmailData="Subject: $(hostname | cat)'s Disk Usage \n\n $(hostname | cat) : ${HDDValue:0:5}% of Hard-Drive is used"
    echo -e $EmailData | cat > Email.txt
    cat Email.txt | sudo ssmtp -vvv jhbrink59@gmail.com 
    sudo rm Email.txt

    fi
     }


#####################################################################3
  
logQueries () {
    CPUData=$(curl -g $site$CPUQuery)
    RAMData=$(curl -g $site$RAMQuery)
    HDDData=$(curl -g $site$HDDQuery)
    HDCData=$(curl -g $site$HDCQuery)
    Output=''
    echo ''
    echo "================================================================================"
    echo "START   "$(date | cat)
    echo "================================================================================"
    echo 'RUN : Prom-LoggerAlerter.sh in /etc/prometheus'
    echo 'Site = '${site}
    echo ""
    echo "--------------------------------------------------------------------------------"
    echo "CPU" 
    echo ""
    ExtractData $CPUData Output 
    echo -e $Output
    echo ""
    echo "--------------------------------------------------------------------------------"
    echo "RAM"
    echo ""
    ExtractData $RAMData Output
    echo -e $Output
    echo ""
    echo "--------------------------------------------------------------------------------"
    echo "HDD" 
    echo ""
    ExtractData $HDDData Output
    CheckHardDrive $Output
    echo -e $Output
    echo ""
    echo "--------------------------------------------------------------------------------"
    echo "Hard-Drive Free Capacity"
    echo ""
    ExtractData $HDCData Output
    echo -e $Output
    echo ""
    echo "================================================================================"
    echo "END   "$(date | cat)
    echo "================================================================================"
  }

#FUNCTION /END
logQueries | cat >> etc/prometheus/Prom-Log.txt


