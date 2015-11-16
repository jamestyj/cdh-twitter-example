# Analyzing Twitter Data Using CDH

This repository contains an example application for analyzing Twitter data
using a variety of CDH components, including [Flume](http://flume.apache.org),
[Oozie](http://incubator.apache.org/oozie), and [Hive](http://hive.apache.org).

## Getting Started

The easiest way is to download and run the [Cloudera QuickStart
VM](http://www.cloudera.com/content/www/en-us/downloads.html).

### Configuring Flume

1. **Build or Download the custom Flume Source**

   A pre-built version of the custom Flume Source is available
   [here](http://files.cloudera.com/samples/flume-sources-1.0-SNAPSHOT.jar).

   The `flume-sources` directory contains a Maven project with a custom Flume
   source designed to connect to the Twitter Streaming API and ingest tweets in
   a raw JSON format into HDFS.

   To build the flume-sources JAR, from the root of the git repository:

       $ cd flume-sources
       $ mvn package
       $ cd ..

   This will generate a file called `flume-sources-1.0-SNAPSHOT.jar` in the
   `target` directory.

2. **Add the JAR to the Flume classpath**

   Copy `flume-sources-1.0-SNAPSHOT.jar` to
   `/usr/lib/flume-ng/plugins.d/twitter-streaming/lib/flume-sources-1.0-SNAPSHOT.jar`
   and also to
   `/var/lib/flume-ng/plugins.d/twitter-streaming/lib/flume-sources-1.0-SNAPSHOT.jar`,
   just to be sure (actually, refer to Plugin Directories in Cloudera
   Manager->flume->configuration->Agent(Default)). If those places don't exist,
   `sudo mkdir` them.

3. **Configure Flume agent in Cloudera Manager Web UI flume**

    Go to the Flume Service page (by selecting Flume service from the Services
    menu or from the All Services page).

    Pull down the `Configuration` tab, and select `View and Edit`.

    Select the Agent (Default) in the left hand column.

    Set the Agent Name property to `TwitterAgent` whose configuration is
    defined in flume.conf.

    Copy the contents of flume.conf file, in its entirety, into the
    Configuration File field. -- If you wish to edit the keywords and add
    Twitter API related data, now might be the right time to do it.

    Click `Save Changes` button.

### Setting up Hive

1. **Build or Download the JSON SerDe**

   A pre-built version of the JSON SerDe is available
   [here](http://files.cloudera.com/samples/hive-serdes-1.0-SNAPSHOT.jar).

   The `hive-serdes` directory contains a Maven project with a JSON SerDe which
   enables Hive to query raw JSON data.

   To build the hive-serdes JAR, from the root of the git repository:

       $ cd hive-serdes
       $ mvn package
       $ cd ..

   This will generate a file called `hive-serdes-1.0-SNAPSHOT.jar` in the
   `target` directory.

2. **Create the Hive directory hierarchy**

        $ sudo -u hdfs hadoop fs -mkdir /user/hive/warehouse
        $ sudo -u hdfs hadoop fs -chown -R hive:hive /user/hive
        $ sudo -u hdfs hadoop fs -chmod 750 /user/hive
        $ sudo -u hdfs hadoop fs -chmod 770 /user/hive/warehouse

    You'll also want to add whatever user you plan on executing Hive scripts
    with to the hive Unix group:

        $ sudo usermod -a -G hive <username>

3. **Create the tweets table**

    Run `hive`, and execute the following commands:

        ADD JAR <path-to-hive-serdes-jar>;

        CREATE EXTERNAL TABLE tweets (
          id BIGINT,
          created_at STRING,
          source STRING,
          favorited BOOLEAN,
          retweeted_status STRUCT<
            text:STRING,
            user:STRUCT<screen_name:STRING,name:STRING>,
            retweet_count:INT>,
          entities STRUCT<
            urls:ARRAY<STRUCT<expanded_url:STRING>>,
            user_mentions:ARRAY<STRUCT<screen_name:STRING,name:STRING>>,
            hashtags:ARRAY<STRUCT<text:STRING>>>,
          text STRING,
          user STRUCT<
            screen_name:STRING,
            name:STRING,
            friends_count:INT,
            followers_count:INT,
            statuses_count:INT,
            verified:BOOLEAN,
            utc_offset:INT,
            time_zone:STRING>,
          in_reply_to_screen_name STRING
        )
        PARTITIONED BY (datehour INT)
        ROW FORMAT SERDE 'com.cloudera.hive.serde.JSONSerDe'
        LOCATION '/user/flume/tweets';

    The table can be modified to include other columns from the Twitter data,
    but they must have the same name, and structure as the JSON fields
    referenced in the [Twitter
    documentation](https://dev.twitter.com/docs/tweet-entities).

### Prepare the Oozie workflow

1. **Configure Oozie to use MySQL**

    If using Cloudera Manager, Oozie can be reconfigured to use MySQL via the
    service configuration page on the Databases tab. Make sure to restart the
    Oozie service after reconfiguring. You will need to install the MySQL JDBC
    driver in `/usr/lib/oozie/libext`.

    If Oozie was installed manually, Cloudera provides
    [instructions](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH4/4.2.1/CDH4-Installation-Guide/cdh4ig_topic_17_6.html)
    for configuring Oozie to use MySQL.

2. **Create a lib directory and copy any necessary external JARs into it**

    External JARs are provided to Oozie through a `lib` directory in the
    workflow directory. The workflow will need a copy of the MySQL JDBC driver
    and the hive-serdes JAR.

        $ mkdir oozie-workflows/lib
        $ cp hive-serdes/target/hive-serdes-1.0-SNAPSHOT.jar oozie-workflows/lib
        $ cp /var/lib/oozie/mysql-connector-java.jar oozie-workflows/lib

3. **Copy hive-site.xml to the oozie-workflows directory**

    To execute the Hive action, Oozie needs a copy of `hive-site.xml`:

        $ sudo cp /etc/hive/conf/hive-site.xml oozie-workflows
        $ sudo chown <username>: oozie-workflows/hive-site.xml

4. **Copy the oozie-workflows directory to HDFS**

        $ hadoop fs -put oozie-workflows /user/<username>/oozie-workflows

5. **Install the Oozie ShareLib in HDFS**

        $ sudo -u hdfs hadoop fs -mkdir /user/oozie
        $ sudo -u hdfs hadoop fs -chown oozie:oozie /user/oozie

    In order to use the Hive action, the Oozie ShareLib must be installed.
    Installation instructions can be found
    [here](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH4/4.2.1/CDH4-Installation-Guide/cdh4ig_topic_17_6.html).

## Starting the data pipeline

1. **Start the Flume agent**

    Create the HDFS directory hierarchy for the Flume sink. Make sure that it
    will be accessible by the user running the Oozie workflow.

        $ hadoop fs -mkdir /user/flume/tweets
        $ hadoop fs -chown -R flume:flume /user/flume
        $ hadoop fs -chmod -R 770 /user/flume
        $ sudo /etc/init.d/flume-ng-agent start

    If using Cloudera Manager, start Flume agent from Cloudera Manager Web UI.

2. **Adjust the start time of the Oozie coordinator workflow in job.properties**

    You will need to modify the `job.properties` file, and change the
    `jobStart`, `jobEnd`, and `initialDataset` parameters. The start and end
    times are in UTC, because the version of Oozie packaged in CDH4 does not
    yet support custom timezones for workflows. The initial dataset should be
    set to something before the actual start time of your job in your local
    time zone. Additionally, the `tzOffset` parameter should be set to the
    difference between the server's timezone and UTC. By default, it is set to
    -8, which is correct for US Pacific Time.

3. **Start the Oozie coordinator workflow**

        $ oozie job -oozie http://<oozie-host>:11000/oozie -config oozie-workflows/job.properties -run
