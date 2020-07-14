#!/bin/ksh
#######################################################################
#
# Copyright [2018] Delta Air Lines, Inc./Delta Technology, Inc.
#                       All Rights Reserved
#               Access, Modification, or Use Prohibited
# Without Express Permission of Delta Air Lines or Delta Technology.
#
#######################################################################

checkDir()
{
  if [ -d $1 ]
    then
      printf "%s exists\n" $1
    else
      printf "Directory %s does not exist. Creating it...\n\n" $1
      mkdir -p $1
  fi
}

function setEnv
{
      database="%database%"
      schema="%schema%"
      db2userid="%db2userid%"
      db2passwd="%db2passwd%"
      serverid="%serverid%"
}

#######################################################################
#                     Script starts here
#######################################################################
echo ""

#### CHECK IF ALL REQUIRED ARGUMENTS ARE PASSED ####

if [ "$1" = "" ]
  then
    printf "Install version required. Exiting...\n\n"
    exit
fi

if [ "$2" = "" ]
  then
    printf "Install environment required. Exiting...\n\n"
    exit
fi

if [ "$3" = "" ]
  then
    printf "User Id required. Exiting...\n\n"
    exit
fi

#### IF ARGUMENTS ARE PASSED THEN SET THE PASSED VALUES TO THE REQUIRED VARIABLES ####

COMPONENT_VER="$1"
RPENV=$2
USERID=$3

#### ECHO THE VERSION AND RPENV PASSED ####

echo ${COMPONENT_VER}
echo ${RPENV}

#### SET THE INSTALL DIRECTORY ####

INSTALL_DIR=${BASEDIR}/${USERID}/install/revrec/rpjava/${COMPONENT_VER}

echo "        -------------------------------------------------------------"
echo "        ---------------- Begin iar/update_recrev_date ----------------"
echo "        -------------------------------------------------------------"
echo "\nInstalling Version: ${COMPONENT_VER} ..."

APPLICATION_DIR=${BASEDIR}/${USERID}3/revrec/update_recrev_date
echo "APPLICATION_DIR: ${APPLICATION_DIR}"

APPLICATION_SRC_LOG_DIR=${SRC_LOG_BASEDIR}/${USERID}3/revrec/update_recrev_date

checkDir ${APPLICATION_DIR}
checkDir ${APPLICATION_SRC_LOG_DIR}

cd ${APPLICATION_DIR}

SED_TEMP_FILE=${APPLICATION_DIR}/_sed.tmp

checkDir ${APPLICATION_SRC_LOG_DIR}/Logs
checkDir ${APPLICATION_SRC_LOG_DIR}/Archive
checkDir ${APPLICATION_DIR}/Config

LD_LIBRARY_PATH=${IBM_DB2_SHLIB_PATH}:${VERSION_DIR}:$LD_LIBRARY_PATH

export LD_LIBRARY_PATH

export CLASSPATH=.:${XERCES_DIR}/xerces.jar:${JAXB_ENV}:${FRAME_WORK_DIR}/mwMsgTestImpl.jar:${FRAME_WORK_DIR}/mwSvrFW.jar:${SAPI_BASE_DIR}/mwMsg.jar:${SAPI_BASE_DIR}/mwBeans.jar:${SAPI_BASE_DIR}/mwLog.jar:${IAPI_BASE}/iapibase.jar:${IBM_DB2}/db2jcc.jar:${IBM_DB2}/db2jcc_license_cisuz.jar:${IBM_DB2}/db2jcc_license_cu.jar:${IBM_DB2}/db2java.zip:${IBM_DB2}/Common.jar:${IBM_DB2}/sqlj.zip:${IBM_DB2}/db2fs.jar:${IBM_DB2}/db2policy.jar:${IBM_DB2}/db2qgjava.jar:${RP_COMMON_JAR}:${VERSION_DIR}/itext-2.0.1.jar:${VERSION_DIR}/pdfschema.jar:${VERSION_DIR}/pdflayoutmanager.jar:${SAPI_MQ_DIR}/mwMsgMqImpl.jar:${SAPI_MQ_DIR}/QueueService.jar:${SAPI_TUX_DIR}/mwMsgJoltImpl.jar:${MQM_JAR}:${TUXEDO_DIR}/jolt.jar:${APPLICATION_JAR}:${LOG4J_JAR}

echo "CLASPPATH:$CLASSPATH"

#misc initial setup
EVENT=Appl_Event
HOST="app_Host='`hostname`'"
SCRIPT=`basename $0`
SCRIPT=`echo $SCRIPT | sed s/\.sh// `
LOGDATE=`date +%Y%m%d`
LOG_FILE="${APPLICATION_DIR}/Logs/${SCRIPT}_${LOGDATE}.log"

echo "Update script execution start" > ${APPLICATION_DIR}/Logs/update_recrev_date.log
logger "$0 started"

setEnv
 
source=./installRevRecMonthend.sh
echo "connecting... $database -- $db2userid"
#db2 connect to $database user $db2userid using $db2passwd 
YEAR=`date +%Y`
MON=`date +%m`
DAY=`date +%d`
curDate=$YEAR'-'$MON'-'$DAY
echo "Current Date :-- $curDate"
    
selectQryCount="SELECT count(*) FROM ebkdp03.utr_rev_actg_rptg_cal WHERE fncl_cal_end_dt = '$curDate';"
getCount=$(db2 -x -t "$selectQryCount")
	echo '--Select Query count----'$getCount
	echo '--Select Query count----'$getCount  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
	if [ $getCount == 1 ]; then
		echo "Inside the logic after query result getting One to proceed --"
		selectResult="SELECT to_char(rptg_cal_strt_dt,'YYYY-MM-DD') as rptg_cal_strt_dt, to_char(rptg_cal_end_dt,'YYYY-MM-DD') as rptg_cal_end_dt, 
						to_char(fncl_cal_strt_dt,'YYYY-MM-DD') as fncl_cal_strt_dt, to_char(fncl_cal_end_dt,'YYYY-MM-DD') as fncl_cal_end_dt,  prir_perd_ct 
						FROM ebkdp03.utr_rev_actg_rptg_cal 
						WHERE fncl_cal_end_dt = '$curDate';"
		echo "Executing.. ---- $selectResult"
		echo "Executing.. ---- $selectResult"  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		
		db2 -x $selectResult | while read rptg_cal_strt_dt rptg_cal_end_dt fncl_cal_strt_dt fncl_cal_end_dt prir_perd_ct ; do
			echo "rptg_cal_strt_dt:--$rptg_cal_strt_dt, rptg_cal_end_dt:--$rptg_cal_end_dt,  fncl_cal_strt_dt:--$fncl_cal_strt_dt, fncl_cal_end_dt:--$fncl_cal_end_dt, prir_perd_ct:--$prir_perd_ct"
			rptgCalStrtDt=$rptg_cal_strt_dt
			rptgCalEndDt=$rptg_cal_end_dt
			fnclCalStrtDt=$fncl_cal_strt_dt
			fnclCalEndDt=$fncl_cal_end_dt
			prirPerdCt=$prir_perd_ct			
		done
		
		#Starting Update query for B
		prirdaysToAdd=30
		addPrirPerdCt30days=$(($prirPerdCt + $prirdaysToAdd))
		initUtrEvtDt=`$(date --date="${rptgCalStrtDt} -${addPrirPerdCt30days} day" +%Y-%m-%d)`
		qryCountUpdateB="SELECT count(*) FROM ebkdp03.mt_cpn_utr_st WHERE INIT_UTR_EVT_DT > '$initUtrEvtDt' and INIT_UTR_EVT_DT <= '$rptgCalEndDt' and cut_evt_dt <= '$rptgCalEndDt' and utr_rev_rctn_stt_cd = 'Y' and utr_rev_rctn_elg_ind = 'Y' and lgcy_hub_rrgn_cd is null;"
		echo "Checking count before executing update Query-B ---- $qryCountUpdateB"
		echo "Checking count before executing update Query-B ---- $qryCountUpdateB"  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		
		getCountB=$(db2 -x -t "$qryCountUpdateB")
		echo '--Select Query count getCountB----'$getCountB
		echo '--Select Query count getCountB----'$getCountB  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		
		resultBeforeUpdateB="SELECT * FROM ebkdp03.mt_cpn_utr_st WHERE INIT_UTR_EVT_DT > '$initUtrEvtDt' and INIT_UTR_EVT_DT <= '$rptgCalEndDt' and cut_evt_dt <= '$rptgCalEndDt' and utr_rev_rctn_stt_cd = 'Y' and utr_rev_rctn_elg_ind = 'Y' and lgcy_hub_rrgn_cd is null;"
		echo "selecting results before executing updateB ---- $resultBeforeUpdateB"
		echo "selecting results before executing updateB ---- $resultBeforeUpdateB"  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		db2 -x $resultBeforeUpdateB > ${APPLICATION_DIR}/Logs/updateQryResult.txt
		
		echo "updating query B starts..."
		updateQryB="UPDATE ebkdp03.mt_cpn_utr_st set lgcy_hub_rrgn_cd = '$DAY' WHERE INIT_UTR_EVT_DT > '$initUtrEvtDt' and INIT_UTR_EVT_DT <= '$rptgCalEndDt' and cut_evt_dt <= '$rptgCalEndDt' and utr_rev_rctn_stt_cd = 'Y' and utr_rev_rctn_elg_ind = 'Y' and lgcy_hub_rrgn_cd is null;"
		echo "update-B query...:$updateQryB"
		getResultUpdatedB=`$(db2 -x -t "$updateQryB") << BEOF |grep -v '^$'|grep -v selected
							COMMIT
							BEOF`
		echo " No.of records updated in query UpdateB:-- $getResultUpdatedB"
		echo " No.of records updated in query UpdateB:-- $getResultUpdatedB"  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		
		
		#sStarting Update query for C
		qryCountUpdateC="SELECT count(*) FROM ebkdp03.mt_cpn_utr_st WHERE INIT_UTR_EVT_DT > '$initUtrEvtDt' and INIT_UTR_EVT_DT <= '$rptgCalEndDt' and cut_evt_dt <= '$rptgCalEndDt' and utr_rev_rctn_stt_cd = 'Y' and utr_rev_rctn_elg_ind = 'Y' and mt_txt_doc_typ_cd <> 'TKT';"
		echo "Checking count before executing update Query-C ---- $qryCountUpdateC"
		getCountB=$(db2 -x -t "$qryCountUpdateC")
		echo '--Select Query count getCountB----'$getCountB
		echo '--Select Query count getCountB----'$getCountB  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		
		resultBeforeUpdateC="SELECT * FROM ebkdp03.mt_cpn_utr_st WHERE INIT_UTR_EVT_DT > '$initUtrEvtDt' and INIT_UTR_EVT_DT <= '$rptgCalEndDt' and cut_evt_dt <= '$rptgCalEndDt' and utr_rev_rctn_stt_cd = 'Y' and utr_rev_rctn_elg_ind = 'Y' and mt_txt_doc_typ_cd <> 'TKT';"
		echo "selecting results before executing updateC ---- $resultBeforeUpdateC"
		echo "selecting results before executing updateC ---- $resultBeforeUpdateC"  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		db2 -x $resultBeforeUpdateC  > ${APPLICATION_DIR}/Logs/updateQryResult.txt
		
		echo "updating query C starts..."
		updateQryC="UPDATE ebkdp03.mt_cpn_utr_st set utr_rev_rctn_stt_cd = 'X', utr_rev_rctn_elg_ind = 'N', LST_UPDT_USR_ID='TD14187', LST_UPDT_GTS='' WHERE INIT_UTR_EVT_DT > '$initUtrEvtDt' and INIT_UTR_EVT_DT <= '$rptgCalEndDt' and cut_evt_dt <= '$rptgCalEndDt' and utr_rev_rctn_stt_cd = 'Y' and utr_rev_rctn_elg_ind = 'Y' and mt_txt_doc_typ_cd <> 'TKT';"
		echo "update-C query...:$updateQryC"
		echo "update-C query...:$updateQryC" > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		
		getResultUpdatedC=`$(db2 -x -t "$updateQryC") << BEOF |grep -v '^$'|grep -v selected
							COMMIT
							BEOF`
		echo " No.of records updated in query UpdateB:-- $getResultUpdatedC"
		echo " No.of records updated in query UpdateB:-- $getResultUpdatedC"  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
		
	else
		echo "Select query returned result not equal to 1 which is not expected to proceed.."
		echo "Select query returned result not equal to 1 which is not expected to proceed.."  > ${APPLICATION_DIR}/Logs/update_recrev_date.log
	fi

exitProcess 0
