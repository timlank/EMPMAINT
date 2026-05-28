EMPMAINT

This application will allow users to CRUD (create,read,update and delete) the EMP (employees) table that is included in DB2 using a CICS Region.

Note: the application was originally written for the OS/390 V2R10 ADCD image. 

All done using IBMUSER account on OS/390 V2R10 ADCD

DB2 6.10  (found via 3.4 searching for **.SDSNLOAD)
CICS TS  1.3    (found via 3.4 searching for SDFHLOAD)
IBM COBOL for OS/390 2.1.2   (found via 3.4 searching for IGY*.**)


1. Allocate three PDS's

      IBMUSER.EMPMAINT.CICS          (COBOL programs & Copy code, BMS maps, JCL)
         Record Format FB
         Record Length 80
         Block size  27920
         2 cylinders

      IBMUSER.EMPMAINT.CICS.LOAD     (Destination for CICS Load modules)
         Record Format U 
         Record Length 0
         Block size  27920
         3 cylinders

      IBMUSER.EMPMAINT.DBRMLIB       (DB2 load modules)
         Record Format FB
         Record Length 80
         Block size  27920
         1 cylinder


2. Make a copy of the DSN8610.EMP DB2 table to IBMUSER.MY_EMP just like shown in the HS Tech Channel DB2 videos.


3. Make sure that you've included the CICS LOAD library you just created in the SYS1.PROCLIB(CICSA) member

//DFHRPL   DD DSN=&INDEX2..SDFHLOAD,DISP=SHR          
//         DD DSN=CEE.SCEECICS,DISP=SHR               
//         DD DSN=CEE.SCEERUN,DISP=SHR                    
//         DD DSN=IBMUSER.EMPMAINT.CICS.LOAD,DISP=SHR 

and also make sure you have the DB2 Load library included in the STEPLIB portion of the member

//STEPLIB  DD DSN=&INDEX2..SDFHAUTH,DISP=SHR         
//         DD DSN=CEE.SCEERUN,DISP=SHR               
//         DD DSN=DSN610.SDSNLOAD,DISP=SHR            


4.  go to the console and issue CANCEL CICSA   followed by   START CICSA      (this stops and starts CICSA)


5.  FTP the sources over to IBMUSER.EMPMAINT.CICS

      ftp login to your OS/390 
      ftp>  CD 'IBMUSER.EMPMAINT.CICS'
      ftp>  PROMPT
      ftp>  MPUT *

6.  Compile the following JCL in this order:

MAPJCL1
MAPJCL2
MAPJCL3
MAPJCL4
MAPJCL5
COBJCL1
COBJCL2
COBJCL3
COBJCL4
COBJCL5
COBJCL8
COBJCL9
COBJCL10

(The DCLGEN member "MYEMP" is already included in the sources)

Note:  The SUBMIT CLIST and SUBREXX REXX scripts which are included can be used to perform these compilations for #6 & #7 in a controlled manner from TSO.  They can be located anywhere, but the hlq below and the DD name in the SUBMIT dataset need to match the desired location.
     example:    EXEC 'IBMUSER.SRC.CNTL(SUBMIT)'



7.  Then compile the JCL to perform the DB2 Package and Plan Binding

DB2BIND


8.  Login to CICS in another terminal and issue all of the following commands hitting F3 and then the clear key between successes (put the end of the long ones just below the CEDA DEFINE line to include the entire command)

CEDA DEFINE MAP(EMPMP1) GROUP(EMPMAINT)
CEDA DEFINE MAP(EMPMP2) GROUP(EMPMAINT)
CEDA DEFINE MAP(EMPMP3) GROUP(EMPMAINT)
CEDA DEFINE MAP(EMPMP4) GROUP(EMPMAINT)
CEDA DEFINE MAP(EMPMP5) GROUP(EMPMAINT)
CEDA DEFINE PROGRAM(EMPCRE) GROUP(EMPMAINT)
CEDA DEFINE PROGRAM(EMPCRE2) GROUP(EMPMAINT)
CEDA DEFINE PROGRAM(EMPDEL) GROUP(EMPMAINT)
CEDA DEFINE PROGRAM(EMPDEL2) GROUP(EMPMAINT)
CEDA DEFINE PROGRAM(EMPDISP) GROUP(EMPMAINT)
CEDA DEFINE PROGRAM(EMPMAIN) GROUP(EMPMAINT)
CEDA DEFINE PROGRAM(EMPUPD) GROUP(EMPMAINT)
CEDA DEFINE PROGRAM(EMPUPD2) GROUP(EMPMAINT)
CEDA DEFINE TRANSACTION(EMCR) PROGRAM(EMPCRE) GROUP(EMPMAINT)
CEDA DEFINE TRANSACTION(EMDI) PROGRAM(EMPDISP) GROUP(EMPMAINT)
CEDA DEFINE TRANSACTION(EMDL) PROGRAM(EMPDEL) GROUP(EMPMAINT)
CEDA DEFINE TRANSACTION(EMMN) PROGRAM(EMPMAIN) GROUP(EMPMAINT)
CEDA DEFINE TRANSACTION(EMUP) PROGRAM(EMPUPD) GROUP(EMPMAINT)
CEDA DEFINE DB2CONN(DSN1) GROUP(EMPMAINT) CONNECTERROR(SQLCODE) DB2ID(DSN1) MSGQUEUE1(CSMT) SIGNID(IBMUSER) COMAUTHID(IBMUSER) AUTHID(IBMUSER)
CEDA DEFINE DB2ENTRY(DB2ENTR) GROUP(EMPMAINT) DROLLBACK(YES) PLAN(EMMT) PRIORITY(HIGH) PROTECTNUM(0) THREADLIMIT(10) THREADWAIT(YES) AUTHID(IBMUSER)
CEDA DEFINE DB2TRAN(EMCR) GROUP(EMPMAINT) ENTRY(DB2ENTR) TRANSID(EMCR)
CEDA DEFINE DB2TRAN(EMDI) GROUP(EMPMAINT) ENTRY(DB2ENTR) TRANSID(EMDI)
CEDA DEFINE DB2TRAN(EMDL) GROUP(EMPMAINT) ENTRY(DB2ENTR) TRANSID(EMDL)
CEDA DEFINE DB2TRAN(EMUP) GROUP(EMPMAINT) ENTRY(DB2ENTR) TRANSID(EMUP)
CEDA INSTALL GROUP(EMPMAINT)
CEMT INQUIRE DB2CONN
(hit enter key one time first on this screen then make sure the AUTHID = IBMUSER, COMAUTHID = IBMUSER, SIGNID = IBMUSER and then overtype the Notconnected word in the CONNECTST field with the word CONNECTED and hit enter)
Over in the System Log in M.5, it should show a message like:  "CICS The CICS-DB2 attachment has connected to DB2 subsystem DSN1"

Back in the CICS terminal hit the clear key and type:  EMMN      (after hitting enter, this should start the Employee Maintenance application)
