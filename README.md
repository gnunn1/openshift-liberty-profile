# Liberty Profile on OpenShift Demo

This is a demo of running Websphere Liberty Profile JEE server on OpenShift v3. This demo uses a slightly modified version of the [JEE7 DayTrader](https://github.com/WASdev/sample.daytrader7) application to use MySQL as the database back-end. This demo includes the EAR already built for you, however if you want to see exactly what changes were made they are documented [here](BuildDaytrader.md).

This demo was tested on OpenShift 3.7 using ```oc cluster up``` however it should work in any 3.6 or later version.

The following sections describe the steps needed to run the application.

### Start OpenShift and Create the Project

* Start your openshift environment if not already started. I use ```oc cluster up``` but minishift, the Red Hat CDK or other environment will work equally as well.

* Create a project for the demo

```oc new-project liberty```

### Create the database

Daytrader requires a database to work, so in this first step we create that database. Note that the step below is using a persistent MySQL template which means the database will persistent between container restarts. You could substitute an epheremal MySQL template instead though if you wish.

```oc new-app --template=mysql-persistent -p MYSQL_USER=daytrader -p MYSQL_PASSWORD=daytrader -p MYSQL_DATABASE=daytraderdb -e MYSQL_LOWER_CASE_TABLE_NAMES=1```

Note that the schema uses lowercase table names hence we set the environment variable ```MYSQL_LOWER_CASE_TABLE_NAMES```.

### Create the Build Image

This demo uses OpenShift's Source-2-Image (s2i) technology to create the daytrader container. The s2i allows you to create new container images using source code or binary artifacts (i.e. ear, war, etc) without having to write a dockerfile. Fortunately, there is an existing [S2I Liberty](https://github.com/redhat-cop/containers-quickstarts/tree/master/s2i-liberty) builder available for OpenShift which we can leverage. Unfortunately it has one minor [issue](https://github.com/redhat-cop/containers-quickstarts/issues/59) that needed correcting so this demo will use a [forked](https://github.com/gnunn1/containers-quickstarts) version of that image.

Please review the documentation of the s2i-liberty image to understand better how it works.

OpenShift is capable of building docker images directly, and that is what we will be doing here:

```oc new-build websphere-liberty:javaee7~https://github.com/gnunn1/containers-quickstarts --context-dir=s2i-liberty --name=s2i-liberty --strategy=docker```

This builds a new docker image using the official ```websphere-liberty:javaee7``` as the base and layering on the dockerfile from ```https://github.com/gnunn1/containers-quickstarts``` to add the s2i capabilities.

Please note one difference above versus the s2i-liberty example is that we are using the javaee7 tag rather than webProfile7. This is because the Daytrader application requires the full JEE profile as it is dependant on EJB, JMS, etc to function correctly. You can view more information about the different versions of the [websphere-liberty](https://hub.docker.com/_/websphere-liberty) image on Docker Hub.

### Create the Daytrader Image

Prior to executing this step, make sure the previous build has completed:

```oc get builds```

You should see a build of ```s2i-liberty-1``` with a status of ```Complete```. If the status is ```Running``` wait a few seconds and then check again.

In this step we create the Daytrader application image using an s2i binary build process. As mentioned previously, s2i allows you to either build based on source code or by passing the pre-built artifacts. In a binary build you are not restricted to just passing the single artifact, but rather you can pass a whole directory which can include many different things beyond the application artifact such as server configuration. In this case we will leverage this capability to upload a complete server configuration.

In this repo, there is a directory called deployments which contains the directory that we will use for our binary build. It includes the application EAR file, the MySQL JDBC driver jar file as well as the ```server.xml``` needed to configure liberty for the application. Note that the daytrader build generates these files for you in the ```daytrader-ee7-wlpcfg``` module. I've just extracted the server.xml and modified it for MySQL instead of Derby.

To build the Daytrader application, run the following command from where you cloned this repo:

```
oc new-build s2i-liberty --name=daytrader --binary=true
oc start-build daytrader --from-dir=deployments
```

The first command creates the build definition in OpenShift, the second starts the build using the ```deployments``` directory as the source.

### Create the Daytrader Application

Once the build has completed (check with ```oc get builds``` again), and only once the build has completed, run the following command to create the Daytrader application:

```
oc new-app daytrader
```

This will automatically link the application with the build we created previously since they share the same name. In the OpenShift console, you should see the Daytrader pod starting.

The next step is we need to expose the application outside of the OpenShift environment by creating a route:

```oc expose svc daytrader --port=9080```

If you look at the OpenShift console, you should see a URL now appear above the Daytrader application. In my case the URL is ```http://daytrader-liberty2.127.0.0.1.nip.io/``` but yours may be different depending on your OpenShift configuration. Click the URL and you should see Websphere Liberty screen.

![Daytrader Config](https://github.com/gnunn1/openshift-liberty-profile/raw/master/images/liberty-splash.png)

### Initialize Daytrader Database

At this point, if you examine the daytrader pod logs you will notice some database errors. This is because the database hasn't been initialized with the schema and data so this is the next step.

Append ```/daytrader``` to the end of the URL in the browser to view the Daytrader application. Click the configuration tab and you will see a screen as per below:

![Daytrader Config](https://github.com/gnunn1/openshift-liberty-profile/raw/master/images/daytrader-config.png)

Click the link ```(Re)-create  DayTrader Database Tables and Indexes``` to initialize the database. Once it is complete you will see a message ```Please Stop and Re-start your Daytrader application (or your application server)```. To do this simply scale the daytrader pod down to 0 and back-up to 1.

Once that is done, go back to the configuration tab and select the ```(Re)-populate  DayTrader Database``` link. This will populate the database with users and quotes. In the interest of expediency I modified the application to lower the users and quotes to 100 and 1000 from the defaults of 15000 and 10000. You can change the values used in ```Configure DayTrader run-time parameters```.

### Use the Application

At this point you can use the application, go to the ```Trading and Portfolio``` tab, login using the default credential and start trading away.




