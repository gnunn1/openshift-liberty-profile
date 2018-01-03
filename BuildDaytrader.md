# Building the Daytrader Application

While this repo includes a pre-built version of Daytrader ready for MySQL, this documents shows the minimal changes that were made in case you want to build it yourself.

* First clone the Daytrader repo as follows:

```git clone https://github.com/WASdev/sample.daytrader7```

* Next we need to add a script to create the database in MySQL, you can do this with the following commands:

```cd daytrader-ee7-web/src/main/webapp/dbscripts```
```mkdir mysql```
```curl https://raw.githubusercontent.com/jamesfalkner/jboss-daytrader/master/javaee6/modules/web/src/main/resources/dbscripts/mysql/Table.ddl -o mysql/Table.ddl```

* Confirm the DDL was downloaded correctly

```cat mysql/Table.ddl```

* In a text editor, open the file ```sample.daytrader7/daytrader-ee7-web/src/main/java/com/ibm/websphere/samples/daytrader/web/TradeConfigServlet.java```. If you navigate to the end of the file, you will see the following code:

```
                    if (dbProductName.startsWith("DB2/")) {// if db is DB2
                        ddlFile = "/dbscripts/db2/Table.ddl";
                    } else if (dbProductName.startsWith("Apache Derby")) { //if db is Derby
                        ddlFile = "/dbscripts/derby/Table.ddl";
                    } else if (dbProductName.startsWith("Oracle")) { // if the Db is Oracle
                        ddlFile = "/dbscripts/oracle/Table.ddl";
                    } else {// Unsupported "Other" Database
                        ddlFile = "/dbscripts/other/Table.ddl";
                        resp.getWriter().println("<BR>TradeBuildDB: **** This Database is unsupported/untested use at your own risk ****</BR>");
                    }
```

Insert the two lines to reference the MySQL DDL that we added:

```
                    if (dbProductName.startsWith("DB2/")) {// if db is DB2
                        ddlFile = "/dbscripts/db2/Table.ddl";
                    } else if (dbProductName.startsWith("Apache Derby")) { //if db is Derby
                        ddlFile = "/dbscripts/derby/Table.ddl";
                    } else if (dbProductName.startsWith("Oracle")) { // if the Db is Oracle
                        ddlFile = "/dbscripts/oracle/Table.ddl";
                    } else if (dbProductName.startsWith("MySQL")) { // if the Db is MySQL
                        ddlFile = "/dbscripts/mysql/Table.ddl";
                    } else {// Unsupported "Other" Database
                        ddlFile = "/dbscripts/other/Table.ddl";
                        resp.getWriter().println("<BR>TradeBuildDB: **** This Database is unsupported/untested use at your own risk ****</BR>");
                    }
```

* In the ```sample.daytrader7``` directory, build the application using Maven:

```
mvn package
```

* By default Daytrader creates 15000 users and 10000 quotes which can take some time. In the interest of expediency I have modified the defaults to 100 users and 1000 quotes in ```sample.daytrader7/daytrader-ee7-web/src/main/webapp/properties/daytrader.properties```:

```
maxUsers=100
maxQuotes=1000
```

Note this can also be changed in the Daytrader UI in the configuration section.