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

#--- Use sed to replace variables in the START script, config.xml, and
#--- properties files based on the server this script is running on

case ${RPENV} in
   st)
      logLevel='5'
      db2UserId='IARPDEV'
      db2Password='XXXXXXXX'
      db2System='//vcond1.delta.com:50004/DB2Z'
      db2DbName='EBKDD30'
      ;;
   dvl)
      logLevel='5'
      db2UserId='IARPST'
      db2Password='XXXXXXXX'
      db2System='//vcond1.delta.com:50004/DB2U'
      db2DbName='EBKDM30'
      ;;
   int)
      logLevel='3'
      db2UserId='IARPSI'
      db2Password='XXXXXXXX'
      db2System='//mvsu-mvss-ddvipa.delta.com:454/DBVG'
      db2DbName='EBKDS03'
      ;;
   prd)
      logLevel='3'
      db2UserId='IARPPROD'
      db2Password='XXXXXXXX'
      db2System='//mvsq-mvsr-ddvipa.delta.com:455/DBRG'
      db2DbName='EBKDP03'
      ;;
   *)
      echo "Unknown host type... " ${RPENV} >&2
      exit;;
esac

checkDir ${APPLICATION_SRC_LOG_DIR}/Logs
checkDir ${APPLICATION_SRC_LOG_DIR}/Archive
checkDir ${APPLICATION_DIR}/Config


#LD_LIBRARY_PATH=/opt/IBM/db2/V8.1/lib:/apps/revpipe/rpjava/%version%:$LD_LIBRARY_PATH
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

echo "connecting... $database -- $db2userid"
#db2 connect to $database user $db2userid using $db2passwd 
YEAR=`date +%Y`
MON=`date +%m`
DAY=`date +%d`
curDate=$YEAR'-'$MON'-'$DAY

    sql="SELECT rptg_cal_strt_dt, rptg_cal_end_dt, fncl_cal_strt_dt, fncl_cal_end_dt, prir_perd_ct FROM ebkdp03.utr_rev_actg_rptg_cal WHERE fncl_cal_end_dt = '$curDate';"
echo "$sql"
   getResult=$(db2 -x -t "$sql")
      


	sqlupdate1="UPDATE ebkdp03.mt_cpn_utr_st set lgcy_hub_rrgn_cd = '$DAY' WHERE INIT_UTR_EVT_DT > '' and INIT_UTR_EVT_DT <= '' and cut_evt_dt <= '' and utr_rev_rctn_stt_cd = 'Y' and utr_rev_rctn_elg_ind = 'Y' and lgcy_hub_rrgn_cd is null;"
	
	sqlupdate2="UPDATE ebkdp03.mt_cpn_utr_st set utr_rev_rctn_stt_cd = 'X', utr_rev_rctn_elg_ind = 'N', LST_UPDT_USR_ID='TD14187', LST_UPDT_GTS='' WHERE INIT_UTR_EVT_DT > '' and INIT_UTR_EVT_DT <= '' and cut_evt_dt <= '' and utr_rev_rctn_stt_cd = 'Y' and utr_rev_rctn_elg_ind = 'Y' and mt_txt_doc_typ_cd <> 'TKT';"
	
	
	
	
	
	
	
	
	
	
	
	


exitProcess 0
