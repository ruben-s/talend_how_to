Talend ETL tool
===============

Description
-----------
Talend is used to create ETL jobs.
These jobs are then uploaded to SpagoBI server to execute there and schedule.

Logging of ETL Jobs
-------------------
Logging of ETL jobs is done through Talend project properties.
(Through the use of environment variables and System.getenv().get("DB_HOSTNAME") in the database connection field the database to log to can be made dependent on the system/environment - TEST/UAT/PROD - where the jobs are run).

There are 3 specific database tables to log.

DB 'infrastructure' on Postgresql:

- statcatcher
- flowcatcher
- logcatcher

To create these tables for the first time:
https://www.talendforge.org/tutorials/tutorial.php?idTuto=33
(In essence use the tStatCatcher, tFlowmeterCatcher, tLogCatcher components and link them to a t[DB]Output component configured to drop and re-create the table.
Run the job and the tables are created.)


### Empty Welcome page

Error on start-up regarding the internal webbrowser

Fix: [link](https://help.talend.com/display/KB/Exception+while+installing+Talend+Studio+on+a+Linux+system)

Download the source code of your XULRunner package and install it into /usr/local/lib. 

Open TalendOpenStudio-linux-gtk-x86_64.ini and add the last line as below: for 64 bit machines

Code:

- vmargs
    - Xms40m
    - Xmx500m
    - XX:MaxPermSize=256m
    - Dorg.eclipse.swt.browser.XULRunnerPath=/usr/local/lib/xulrunner

https://help.talend.com/display/KB/Exception+while+installing+Talend+Studio+on+a+Linux+system

Restart your machine

### Black background color for tooltips on Linux/Ubuntu/GTK

Use Eclipse 3.6 or higher, this was caused by a bug in SWT (bug 309907). If it still happens check your theme settings. On Ubuntu 10.04, the default color scheme of the 'Radiance' theme for tooltips is white text on black background (see System > Preferences > Appearance > Theme > Colours > Tooltips). Here is the bug on the matter on Ubuntu's Launchpad.

In Ubuntu 11.10 and 12.04 there is no interface to change the color scheme of the theme, so it may be useful to edit the gtkrc file in '/usr/share/themes/<yourtheme>/gtk-2.0/' to set the tooltip background and foreground colors. E.g. 'tooltip_fg_color:#000000' & 'tooltip_bg_color:#E6E6FA'.

You can also install and open gnome-color-chooser: Go to Specific > Tooltips and put black foreground over pale yellow background. 

### Open Issue

- Find out how to make logging of ETL go to different databases when moving from development/uat/production.  
Possible solution: define database name in connections as a java environment parameter.
This could then be retreived by System.getenv().get("LOG_DB").
    - => Not possible if done through the Metadata defined connections as all parameters in the database connection definition are put between double qoutes.
    - Instead: do this using System.getEnv("") directly in the connection properties for Stats&Logs of the project properties.
   
Also see following [link](http://www.etladvisors.com/2012/08/22/managing-multiple-db-env/).

Promoting ETL Jobs to SpagoBi server
------------------------------------

### CAS Proxy

The authentication on SpagoBI makes use of the [Company] CAS server.  
However this is not supported by Talend.  
As such a proxy needs to be running on the Talend workstation that redirects Authentication.
- https://github.com/ptillemans/casproxy

This cas proxy listens on port 3000.
It uses the credential to authenticate as defined in a casproxy.properties file.

### SpagoBi Server definition - CAS Proxy for Authentication

SpagoBi servers need to be defined in Talend Preferences:
Application Menu -> Window -> Preferences -> Talend -> Import/Export -> SpagoBi Server

The only parameter that needs to be defined in the SpagoBi Servers is host & port  
=> localhost, port 3000, the authentication is done through the CAS Proxy.

### Deploy on SpagoBi

Right click on a particular Job allows to "Deploy on SpagoBi".
After this deployment the java code that was built by Talend based on the ETL design, is then present on the SpagoBi server.
(For SpagoBi 3.6 this path is: /var/lib/spagobi/resources/talend/RuntimeRepository/java/[projectname]/[jobname]

However the job (i.e. java prog) is not visible in the "Analytical model -> Documents Development". For this to be the case an XML file is needed that links the Document representing the ETL job and job as present on disk.
 
This XML file is created by Talend and transfered to the SpagoBI server on deployment. 
But adding this XML file for the job to be displayed in the desired place on the tree needs to done manually.

To do this, the by Talend created XML file and already present on the filesystem needs to be uploaded through the SpagoBI web interface.
One can transferred again from the SpagoBI server to the Talend workstation (e.g. by secure copy scp).

To make the ETL job visible: 
In the Documents Develoment (Analytical Model) insert a new object.
Select where in the tree the XML document should be added (this determines where the ETL job is visible), and add upload the XML file.

### Creating ETL flows
- ternary condition in tMap for testing and replacing failing lookups from null to -1
(row1.lookup_dwh_id==null)?-1:row1.lookup_dwh_id
- conversion of datatypes
(row1.UNIT_PRICE==null)?null:row1.UNIT_PRICE.doubleValue() 
    - remark: this is needed since the conversion of a null value results in a NullPointer exception
    - remark2: the true case of the condition can not just return the column itself (e.g. in this case the row1.UNIT_PRICE) as this also results in a java NullPointer exception
    - remarkt3: row1.UNIT_PRICE==null will fail if UNIT_PRICE is an int (java primitive can not be null), in that case == 0 is needed
- using sequences: Numeric.sequence("s1",((max_quoted_price_dwh_id.quoted_prices_dwh_id==null)?0:max_quoted_price_dwh_id.quoted_prices_dwh_id) +1, 1) 
    - when the target table is empty and performing a join between staging and target to identify new records for inserts,  
    then max_xxx_dwh_id is null as the target table is still empty. In that case the null needs to be converted to 0 to prevent a java.lang.nullpointer
- to differentiate between inserts and updates on target:
    - create a join between staging and target with tMap
    - configure tMap for inner join
    - configure 1 output for inner join rejects -> inserts (i.e. no equivalent record on target yet)
        - use tAggregate component to retrieve the max value of integer surrogate primary key 
        - use sequence to create new integer surrogate primary keys

### RE-usable context job
- to enable each job to load the context for a job EXPLICITLY easily [download re-usable context job from TalendForge](http://www.talendbyexample.com/talend-reusable-context-load-job.html)
    - check-out following [page](http://www.talendbyexample.com/talend-load-context-example.html)
- alternative configure implicit context loading in project properties

### Altering the default mapping between Talend and DB

- schema import from oracle database incorrectly translates oracle number datatype to BigDecimal with 0 precision.
Edit schema of Oracle table manually to import numbers as Double with precision.
    - Can be altered permanently in Talend, see [post](http://www.knoetry.com/talend-tips-for-database-developers-part-2-data-type-conversion-errors/)
    - Via Menu: Window -> Preferences -> Talend -> Specific Settings -> Metadata of TalendType -> edit the relevant xml file.    
    
### Limitations experienced

- tMap can not handle any other join buy equality and AND conditions
- tELT(Postgresql)Intput/Output/Map combinations can create more complex join clauses
    - however generated "update" statements are horrible
        - the generated update statement creates subselects for each column
        - the update statement does not seem to use the key as indicated in the schema
            - manually add the src_key = target_key in the where clause of the tELTPostgresqlMap output
        - it seems not possible to only update limited set of columns
            - this might be possble by editing the schema of the table in the tELTPostgresqlOutput (make it a built in property) 
            - however this does not always work. Sometimes the schema edit / make it internal is not accepted -> delete he tELTPostgresqlOutput step and start over
                - if you already a mapping in the tELTPostgresqlMap component you lose this work -> do this last
        - even the key field get's updated ???
    - tELTPostgresqlOutput set to update updates all records in the output table irrespective if the corresponding row to update with (based on key) is present
         - if there is no source row to update with the sub select clause in the generated update statement is empty (i.e. null) -> all records get updated with NULL
         - to avoid above behaviour add a manual where clause in the tELTPostgresqlOutput component to limit the to be updated rows to the rows present in the update source
    - performing a aggregate function with a group by clause is not possible if one wants an update
        - the generated sub select in the generated sql contains all columns as present in the group by, not only the field that needs to be updated
- tELT components do not allow for a automatic truncate 
    - one needs to use a tPostgresqlRow component with a manually entered trunc statement
- tELTPostgresqlMap Map Editor is hard to use:
    - most input fields do not expand, line feeds are not possible ("enter" confirms the edit and leaves the input field)
    - in case of complex join clauses this has to be built manually in an input field
        - this input field next to a certain column name contains then other column references, but this is not clear in the input window as only a very little part is shown
        - care has to be taken with () and the sequence of AND & OR
- when using tMap to perform a left outer join to look-up a dhw_id (e.g. from a dimension), and the look-up does not find any corresponding results -> if the data type of the resulting dwh_id column is an int (a primitive java type) then Talend sets dwh_id = 0 (whilst in effect the dhw_id should be null, as it could not be determined) Solve with (distributor.relation_dwh_id == 0)?-1:distributor.relation_dwh_id
- when using a Talend component that results in a dwh_id surregate key as a type Integer, if no result was found then the object returned is null (if it was an int then it will be set to 0). If the Integer (here max_opp_fact_dwh_id.opportunity_dwh_id) is null then a sequence will return an error if you try Numeric.sequence("s1",1+ max_opp_fact_dwh_id.opportunity_dwh_id,1) .
Note: this happens when the table to fill is still void of any records (thus the get max dwh_id is null).
To account for this use: 
Numeric.sequence("s1",1+ ((max_opp_fact_dwh_id.opportunity_dwh_id == null)?0:max_opp_fact_dwh_id.opportunity_dwh_id),1)
- when in tMap comparing 2 input to find differences (with "Catch lookup inner join reject") for updating, then BigDecimal fields with different length and precision will cause differences (and thus updates) eventhough there is no real difference.
    