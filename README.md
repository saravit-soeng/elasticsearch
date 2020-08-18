# Introduction
Elasticsearch is an open source, full-text search and analysis engine, based on the Apache Lucene search engine. Known for its simple REST APIs, distributed nature, speed, and scalability, Elasticsearch is the central component of the Elastic Stack, a set of open source tools for data ingestion, enrichment, storage, analysis, and visualization. Commonly referred to as the ELK Stack (after Elasticsearch, Logstash, and Kibana), the Elastic Stack now includes a rich collection of lightweight shipping agents known as Beats for sending data to Elasticsearch.

The speed and scalability of Elasticsearch and its ability to index many types of content mean that it can be used for a number of use cases:
- Application search
-	Website search
-	Enterprise search
-	Logging and log analytics
-	Infrastructure metrics and container monitoring
-	Application performance monitoring
-	Geospatial data analysis and visualization
-	Security analytics
-	Business analytics

# Prerequisites
The requirements for Elasticsearch are simple:
-	Any operating systems or platforms (Linux recommended)
-	Download and install Java SE JDK (specific version recommended: Oracle JDK version 1.8.0_131 or later version in the Java 8 release series). 

# Installation
You can install Elasticsearch on your own hardware, or use hosted Elasticsearch service on Elastic cloud.
### Install Elasticsearch on Linux with Debian package
It can be used to install Elasticsearch on any Debian-based system such as Debian and Ubuntu.
- Download and install the public signing key:
```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
```
- You may need to install the apt-transport-https package on Debian before proceeding:
```bash
sudo apt-get install apt-transport-https 
```
- Save the repository definition to __/etc/apt/sources.list.d/elastic-7.x.list__:
```bash
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list
```
- You can install the Elasticsearch Debian package with:
```bash
sudo apt-get update && sudo apt-get install elasticsearch
```
- Running Elasticsearch with systemd:
```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```
### Install Elasticsearch on Linux with RPM
It can be used to install Elasticsearch on any RPM-based system such as OpenSuSE, SLES, Centos, Red Hat, and Oracle Enterprise.
- Download and install the public signing key:
```bash
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
- Create a file called elasticsearch.repo in the __/etc/yum.repos.d/__ directory for RedHat based distributions, or in the __/etc/zypp/repos.d/__ directory for OpenSuSE based distributions, containing:
```
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=0
autorefresh=1
type=rpm-md
```
- You can install the Elasticsearch with one of following commands:
```bash
sudo yum install –enablerepo=elasticsearch elasticsearch 
sudo dnf install --enablerepo=elasticsearch elasticsearch
```
-> Use yum on CentOS and older RedHat based distributions

-> Use dnf on Fedora and other new Red Hat distributions

- Running Elasticsearch with systemd:
```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```
### Checking that Elasticsearch is running
You can test that your elasticsearch node is running by sending an HTTP request on port 9200 on localhost: 
```bash
curl -X GET "localhost:9200/?pretty"
```
which should give you a response something like this:
```
{
  "name" : "elasticsearch",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "xxxx-xxxx-xxxx",
   …
  "tagline" : "You Know, for Search"
}
```
You can change the default configuration such as network host, port, java home, etc… in __/etc/elasticsearch/elasticsearch.yml__.

Anyway, you need to restart elasticsearch service to load the configuration change.

# ETL Tools
Enterprises that use Elasticsearch as a data source often need to extract that data for analysis in other business analytics platforms. And if they use Elasticsearch for backend storage, they need a way to put data pulled from other sources into their Elasticsearch data warehouse. All data operations use ETL (extract, transform, and load) processes to move data into and out of storage. There are some available tools that are capable of working with Elasticsearch such as Logstash, Apache NiFi, Transporter, … that you can choose to work with. In this document we select the two most commonly used tools (Logstash and NiFi).
### Logstash
#### Install with APT based distribution
Assume that you have already installed the public signing key, installed apt-transport-https package, and saved the repository definition to __/etc/apt/sources.list.d/elastic-7.x.list__ (as you have done it in the Elasticsearch installation steps).

Now install Logstash with:
```bash
sudo apt-get install logstash
```
#### Install with YUM based distribution
Assume that you have already done the configuration steps as you have done in the Elasticsearch installation steps.

Now you can install logstash with:
```bash
sudo yum install logstash
```
Running Logstash by using systemd:
```bash
sudo systemctl enable logstash.service
sudo systemctl start logstash.service
```

Now you are having logstash service on your machine. Let’s create a logstash event by ingesting the data from MySQL database into Elasticsearch.
First create a logstash config file named it to __itrc-1f-logstash.conf__ in any directory on your server. The configuration should be like:
```
input {
  jdbc { 
    jdbc_connection_string => "jdbc:mysql://localhost:3306/1F_DATA?serverTimezone=Asia/Seoul&useCursorFetch=true"
    jdbc_user => "db_user"
    jdbc_password => "xxxxxxxxx"
    jdbc_driver_library => "/opt/logstash/mysql-connector-java-5.1.42-bin.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    last_run_metadata_path => "/usr/share/logstash/.logstash_jdbc_last_run_1f"
    schedule => "* * * * * *"
    statement => "select * from ITRC_1F_GROUP where no >:sql_last_value"
    use_column_value => true
    tracking_column => "no"
    }
}
output {
  stdout { codec => json_lines }
  elasticsearch {
    hosts => "http://localhost:9200"
    index => "itrc_1f"
  }
}
```
Make sure that you have downloaded the jdbc driver for MySQL from official website and put it in any directory on server and under __/usr/share/logstash__ create an empty file named __.logstash_jdbc_last_run_1f__ .

Create the logstash event under the pipeline, now edit __/etc/logstash/pipelines.yml__
```
- pipeline.id: itrc_1f
  path.config: "/path_to/itrc-1f-logstash.conf"
```
Now save the file and restart logstash service. Check the logstash service running log under /var/log/logstash/logstash-plain.log. 
If all goes well, a new “itrc_1f” index will be created in Elasticsearch.

###	NiFi
In previous section you have ingested data from MySQL into Elasticsearch using Logstash, now let’s use NiFi for ingesting data from MySQL into Elasticsearch.
You can download and install NiFi by following the official website. Now assume that you have installed NiFi on your machine. To get started, open a web browser and navigate to http://localhost:8080/nifi . This will bring up the user interface, which at this point is a blank canvas for orchestrating a dataflow:
 
Next step is to create a processor for query data from database. Create the processor by dragging the processor icon on toolbar into canvas that should look like:
 
In Filter, search for QueryDatabaseTable and add it, then you will have a processor that look like:
 

Configure the processor by double click on it and in the PROPERTIES tab should be like this:
 
 
In this step, you have to create Database connection pooling service in advanced. Click on arrow key (→) in Database Connection Pooling Service property and create new controller service, search for DBCPConnectionPool and add it.
Configuration should look like: 
 
After you have configured it, enable it and select it in Database Connection Pooling Service property of the processor. 
Now let’s add ConvertRecord processor for converting Avro format to Json format. And the configuration should look like:
 
You have to add two new controller services (AvroReader and JsonRecordSetWriter) for Record Reader and Record Writer properties. In JsonRecordSetWriter set the Timestamp Format property to yyyy-MM-dd'T'HH:mm:ss.SSSZ to match with Elasticsearch timestamp format.
The last step is to add PutElasticsearchHttpRecord processor for inserting the data into Elasticsearch, the configuration should be like:
 
 
You have to create a new controller service called JsonTreeReader for Recorder Reader property and no need to configure.
Now you have created the processors, let’s create the relationship between processors by dragging the arrow icon on one processor to another. It should look like:
 
Start the NiFi flow by right click on blank screen on canvas and click Start. If all goes well, the index will be created in Elasticsearch.
V.	Data Visualization
Data visualization and analytics become a very important part in business nowadays. Data visualization provides a lot of business benefits such as data is always in real time. In this document, we choose two possible tools which can work with Elasticsearch to visualize the data in any format such as Kibana and Grafana.
1.	Kibana
Install Kibana on Linux with Debian package
Assume that you have already installed the public signing key, installed apt-transport-https package, and saved the repository definition to /etc/apt/sources.list.d/elastic-7.x.list (as you have done it in the Elasticsearch installation steps).
Now install Kibana with:
sudo apt-get install kibana
Install Kibana on Linux with RPM
Assume that you have already done the configuration steps as you have done in the Elasticsearch installation steps.
Now you can install Kibana with:
sudo yum install kibana
sudo dnf install kibana
•	Use yum on CentOS and older Red Hat based distributions
•	Use dnf on Fedora and other new Red Hat distributions
Running Elasticsearch with systemd:
sudo systemctl daemon-reload
sudo systemctl enable kibana.service
sudo systemctl start kibana.service
You can change the default configuration such as server host, port, elasticsearch hosts, etc. in /etc/kibana/kibana.yml.
Anyway, you need to restart kibana service to load the configuration changes.
Open up Kibana in your browser with: http://localhost:5601. You will be presented with the Kibana home page.
 
In Kibana, go to Management → Stack Management → Index Patterns → Create index pattern. Kibana should display the itrc_1f index that you have created in Logstash step as below:
 
Enter “itrc_1f” as the index pattern, and in the next step select your Time Filter field.
 
Hit Create index pattern, and you are ready to analyze the data. Go to the Discover tab in Kibana to take a look at the data.
 
Now we have set up the ELK data pipeline using Elasticsearch, Logstash, and Kibana.
Next step is to create Dashboard. Go to Dashboard → Create dashboard → Create new → Select Timelion (You can choose another type of visualization based on your requirement) for building time-series using functional expression as a sample. 
 

Then you can edit the Timelion expression as below:
 
Timelion expression:
.es(index=itrc_1f,timefield='date:time',metric=avg:active_power).color(color=green).label(1F).fit(mode=carry).legend(position=ne).lines(width=1,fill=1,show=true)
You can save the visualization and return to dashboard that should look like:
 
Save your dashboard. Now you have created a dashboard with Kibana for analyzing your data and you can add new visualization to your dashboard later.
2.	Grafana
In previous section you have visualized data from Elasticsearch using Kibana, now let’s use Grafana for data visualization from Elasticsearch.
You can download and install Grafana by following the official website. Now assume that you have installed Grafana on your machine. To get started, open a web browser and navigate to http://localhost:3000 . This will bring up the user interface:
 
And default password for Grafana is “admin” for admin user.
After logging in, under Configuration → Data Sources → Add data source → Select Elasticsearch and the configuration should look like:
 
Save & Test. Now you have created the data source for Elasticsearch.
Let’s create Dashboard, under Create → Dashboard → Add new panel and the configuration should look like:
 
Apply and Save the change. Now you have created the dashboard with Grafana and you can add the panel to existing dashboard later.
VI.	References
-	https://www.elastic.co/what-is/elasticsearch
-	https://logz.io/learn/complete-guide-elk-stack
-	https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html
-	https://www.elastic.co/guide/en/logstash/current/installing-logstash.html
-	https://www.elastic.co/guide/en/kibana/current/install.html
-	https://coralogix.com/log-analytics-blog/advanced-guide-to-kibana-timelion-functions/


