EMPMAINT

This application will allow users to CRUD (create,read,update and delete) the EMP (employees) table that is included in DB2 using a CICS Region.

Note: the application was originally written for the OS/390 V2R10 ADCD image. One notable difference is the way DB2 stores dates.

These directions modify that original documentation to be done using IBMUSER account on z/OS V3R1 Standard Development/Test Image.

DB2 v13   (found via 3.4 searching for **.SDSNLOAD)
CICS TS  v6.1  (found via 3.4 searching for SDFHLOAD)
IBM Enterprise COBOL for z/OS V6R4M0  (found via 3.4 searching for IGY*.**)


1. Allocate three PDS's

      IBMUSER.EMPMAINT.CICS          (COBOL programs & Copy code, BMS maps, JCL)
         Record Format FB
         Record Length 80
         Block size  27920
	 Directory Blocks  20
         Primary Quantity  2 cylinders
	 Secondary Quantity  5 cylinders
	 Data set name type:  PDS

      IBMUSER.EMPMAINT.CICS.LOAD     (Destination for CICS Load modules)
         Record Format U 
         Record Length 0
         Block size  27920
         Directory Blocks 20
         Primary Quantity 3 cylinders
	 Secondary Quantity  5 cylinders
	 Data set name type:  LIBRARY

      IBMUSER.EMPMAINT.DBRMLIB      (DB2 load modules)
         Record Format FB
         Record Length 80
         Block size  27920
         Directory Blocks 20
         Primary Quantity 1 cylinder
	 Secondary Quantity  5 cylinders
	 Data set name type: PDS

2. Make a copy of the IBMUSER.EMP DB2 table to IBMUSER.MY_EMP.  (Since we are IBMUSER, you can leave off the prefix)

=13.S2  (SPUFI)
1 SPUFI
  INPUT DATA SET NAME:  'IBMUSER.EMPMAINT.CICS(SQLTEST)
  OUTPUT DATA SET NAME:  'IBMUSER.SRC.SQLOUT'
  Leave all of the PROCESSING OPTIONS at 'YES'
  Hit ENTER key (Ignore warning message if it pops up)
  On 'CURRENT SPUFI DEFAULTS' screen, Hit ENTER
In the editor that comes up, type the following 3 SQL commands (Without the comments):

CREATE TABLE MY_EMP LIKE EMP;
INSERT INTO MY_EMP SELECT * FROM EMP;                /* (COPIES ALL DATA FROM EMP INTO YOUR OWN VERSION OF THE TABLE - MY_EMP)
SELECT * FROM MY_EMP;                                /* (SHOWS ALL THE DATA IN YOUR COPY OF THE TABLE)

Type SAVE at the command line
Hit F3 to exit the edit session
Hit ENTER key to process the SQL commands previously entered in the editor


3. Make sure that you've included the CICS LOAD library you just created in the SYS1.PROCLIB(CICSTS61) member

//DFHRPL   DD DSN=&INDEX3..SEYULOAD,DISP=SHR       
//         DD DSN=&INDEX2..SDFHLOAD,DISP=SHR       
//*  Begin Ansible 1st Insert Block  *//           
//         DD DSN=BZU.SBZULOAD,DISP=SHR            
//         DD DSN=FEL.SFELLOAD,DISP=SHR            
//         DD DSN=CEE.SCEECICS,DISP=SHR            
//         DD DSN=CEE.SCEERUN2,DISP=SHR            
//         DD DSN=CEE.SCEERUN,DISP=SHR             
//         DD DSN=EQAW.SEQAMOD,DISP=SHR            
//         DD DSN=TCPIP.SEZATCP,DISP=SHR           
//         DD DSN=EQAW.EQAIVP.LOAD,DISP=SHR        
//         DD DSN=SYS1.MIGLIB,DISP=SHR             
//         DD DSN=SYS1.SIEAMIGE,DISP=SHR                           
//         DD DSN=IBMUSER.EMPMAINT.CICS.LOAD,DISP=SHR 

and also make sure you have the DB2 Load libraries included in the STEPLIB portion of the member (it should look like this)

//STEPLIB  DD DSN=&INDEX3..SEYUAUTH,DISP=SHR           @D73969A  
//         DD DSN=&INDEX2..SDFHAUTH,DISP=SHR                     
//         DD DSN=&INDEX4..&ACTIVATE,DISP=SHR          @R77578A  
//         DD DSN=CEE.SCEERUN2,DISP=SHR                          
//         DD DSN=CEE.SCEERUN,DISP=SHR                           
//         DD DSN=DB2V13.SDSNLOAD,DISP=SHR                       
//         DD DSN=DB2V13.SDSNLOD2,DISP=SHR                       
//         DD DSN=CICSTS61.CICS.SDFHLINK,DISP=SHR                
//         DD DSN=CICSTS61.SDFHLIC,DISP=SHR                               

Go to the command line and type SAVE  and then F3 to exit editor


4.  go to =13.14 (SDSF) and type / at the COMMAND INPUT line to be able to type console(system) commands and type CANCEL CICSTS61 then hit enter  
    repeat the / to access console(system) commands and type START CICSTS61 and hit enter  (we have just stopped and started CICS to get pick up the new configuration)


5. Ensure unencrypted FTP is configured on z/OS V3R1:
      use =3.4 to go to edit TCPIP.TCPPARMS(FTPSDATA)
	change SECURE_FTP the line near the bottom to read:   
	SECURE_FTP        OPTIONAL         ; FTPS
      then save the dataset and re-ipl your z/OS V3R1


6.  FTP the sources over to IBMUSER.EMPMAINT.CICS
      from a windows command prompt, cd into the CICS directory where you exploded the .zip file containing the code
      ftp login to your z/OS V3R1 with the IBMUSER account and passphrase 
      ftp>  CD 'IBMUSER.EMPMAINT.CICS'
      ftp>  PROMPT
      ftp>  MPUT *

7.  Compile the following JCL in IBMUSER.EMPMAINT.CICS in this order:

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

Note:  The SUBMIT CLIST and SUBREXX REXX scripts which are included can be used to perform these compilations for #7 & #8 in a controlled manner from TSO.  They can be located anywhere, but the hlq below and the DD name in the SUBMIT dataset need to match the desired location.
     example:    EXEC 'IBMUSER.SRC.CNTL(SUBMIT)'


8.  Compile the following JCL in IBMUSER.EMPMAINT.CICS to perform the DB2 Package and Plan Binding

DB2BIND

(the above compile will perform the DB2 Package and Plan Binding)


9.  Login to CICS in another terminal.  CICS commands should be entered in the blank space at the top left of the screen.

Issue all of the following commands hitting F3 and then the clear key between successes 
(put the end of the long commands that take up an entire line on the screen just below the CEDA DEFINE line to include the entire command)

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
CEDA DEFINE DB2CONN(DBD1) GROUP(EMPMAINT) CONNECTERROR(SQLCODE) DB2ID(DBD1) MSGQUEUE1(CSMT) SIGNID(IBMUSER) COMAUTHID(IBMUSER) AUTHID(IBMUSER)
CEDA DEFINE DB2ENTRY(DB2ENTR) GROUP(EMPMAINT) DROLLBACK(YES) PLAN(EMMT) PRIORITY(HIGH) PROTECTNUM(0) THREADLIMIT(10) THREADWAIT(YES) AUTHID(IBMUSER)

CEDA DEFINE DB2TRAN(EMCR) GROUP(EMPMAINT) ENTRY(DB2ENTR) TRANSID(EMCR)
CEDA DEFINE DB2TRAN(EMDI) GROUP(EMPMAINT) ENTRY(DB2ENTR) TRANSID(EMDI)
CEDA DEFINE DB2TRAN(EMDL) GROUP(EMPMAINT) ENTRY(DB2ENTR) TRANSID(EMDL)
CEDA DEFINE DB2TRAN(EMUP) GROUP(EMPMAINT) ENTRY(DB2ENTR) TRANSID(EMUP)
CEDA INSTALL GROUP(EMPMAINT)
CEMT INQUIRE DB2CONN
(hit enter key one time first on this screen then make sure the AUTHID = IBMUSER, COMAUTHID = IBMUSER, SIGNID = IBMUSER and then overtype the Notconnected word in the CONNECTST field with the word CONNECTED and hit enter)
Over in the System Log in 13.14, it should show a message like:  "CICS The CICS-DB2 attachment has connected to DB2 subsystem DBD1"

Back in the CICS terminal hit the clear key and type:  EMMN      (after hitting enter, this should start the Employee Maintenance application)
