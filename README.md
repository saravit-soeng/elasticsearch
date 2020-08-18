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
Now save the file and restart logstash service. Check the logstash service running log under __/var/log/logstash/logstash-plain.log__. 
If all goes well, a new __“itrc_1f”__ index will be created in Elasticsearch.

###	NiFi
In previous section you have ingested data from MySQL into Elasticsearch using Logstash, now let’s use NiFi for ingesting data from MySQL into Elasticsearch.

You can download and install NiFi by following the [official website](https://nifi.apache.org/docs/nifi-docs/html/getting-started.html#downloading-and-installing-nifi). Now assume that you have installed NiFi on your machine. To get started, open a web browser and navigate to http://localhost:8080/nifi . This will bring up the user interface, which at this point is a blank canvas for orchestrating a dataflow:

![1](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F1.jpg?alt=media&token=ce06baad-1528-470b-8a2a-add84c59c5cf)

Next step is to create a processor for query data from database. Create the processor by dragging the processor icon on toolbar into canvas that should look like:

![2](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F2.jpg?alt=media&token=abdae6c0-5613-4539-9977-f96e6e8f9c7c)

In Filter, search for __QueryDatabaseTable__ and add it, then you will have a processor that look like:
 
![3](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F3.jpg?alt=media&token=81d70c99-95e5-40b2-b08b-e32750906e29)

Configure the processor by double click on it and in the __PROPERTIES__ tab should be like this:
 
![4](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F4.jpg?alt=media&token=60645b3a-51aa-41c4-8f92-b2ad80aa72de)

![5](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F5.jpg?alt=media&token=288c984a-c72b-42b0-ad79-3f80c8de857e)
 
In this step, you have to create __Database connection pooling service__ in advanced. Click on arrow key (→) in _Database Connection Pooling Service_ property and create new controller service, search for __DBCPConnectionPool__ and add it.

Configuration should look like:

![6](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F6.jpg?alt=media&token=98cebe63-267e-4b76-b188-cb4ffaf43943)

After you have configured it, enable it and select it in _Database Connection Pooling Service_ property of the processor. 

Now let’s add __ConvertRecord__ processor for converting Avro format to Json format. And the configuration should look like:

![7](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F7.jpg?alt=media&token=6b2d8722-81a6-4282-96dc-163aa3b556f6)

You have to add two new controller services (_AvroReader_ and _JsonRecordSetWriter_) for _Record Reader_ and _Record Writer_ properties. In _JsonRecordSetWriter_ set the Timestamp Format property to __yyyy-MM-dd'T'HH:mm:ss.SSSZ__ to match with Elasticsearch timestamp format.

The last step is to add __PutElasticsearchHttpRecord__ processor for inserting the data into Elasticsearch, the configuration should be like:

![8](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F8.jpg?alt=media&token=707912ef-8939-4ff9-9c31-d808df2c3fb2)
 
![9](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F9.jpg?alt=media&token=83c27ec8-e7d7-4cd3-b1c9-89fcf52ad144)

You have to create a new controller service called __JsonTreeReader__ for _Recorder Reader_ property and no need to configure.

Now you have created the processors, let’s create the relationship between processors by dragging the arrow icon on one processor to another. It should look like:

![10](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F10.jpg?alt=media&token=032cc2b6-75fa-4de7-bd67-52dc9891c6fe)

Start the NiFi flow by right click on blank screen on canvas and click __Start__. If all goes well, the index will be created in Elasticsearch.

#	Data Visualization
Data visualization and analytics become a very important part in business nowadays. Data visualization provides a lot of business benefits such as data is always in real time. In this document, we choose two possible tools which can work with Elasticsearch to visualize the data in any format such as Kibana and Grafana.

### Kibana
#### Install Kibana on Linux with Debian package
Assume that you have already installed the public signing key, installed apt-transport-https package, and saved the repository definition to __/etc/apt/sources.list.d/elastic-7.x.list__ (as you have done it in the Elasticsearch installation steps).

Now install Kibana with:
```bash
sudo apt-get install kibana
```

#### Install Kibana on Linux with RPM
Assume that you have already done the configuration steps as you have done in the Elasticsearch installation steps.

Now you can install Kibana with:
```bash
sudo yum install kibana
sudo dnf install kibana
```
•	Use yum on CentOS and older Red Hat based distributions
•	Use dnf on Fedora and other new Red Hat distributions

#### Running Elasticsearch with systemd:
```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana.service
sudo systemctl start kibana.service
```

You can change the default configuration such as server host, port, elasticsearch hosts, etc. in __/etc/kibana/kibana.yml__.

Anyway, you need to restart kibana service to load the configuration changes.

Open up Kibana in your browser with: http://localhost:5601. You will be presented with the Kibana home page.

![11](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F11.jpg?alt=media&token=b92b7ed8-fd74-46cc-b60a-eb7eedef47c7)

In Kibana, go to __Management__ → __Stack Management__ → __Index Patterns__ → __Create index pattern__. Kibana should display the __itrc_1f__ index that you have created in Logstash step as below:

![12](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F12.jpg?alt=media&token=772fda3e-7dcd-4ff4-9e76-f88826a80e20)

Enter __“itrc_1f”__ as the index pattern, and in the next step select your Time Filter field.

![13](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F13.jpg?alt=media&token=ecce1596-d51a-4bf5-8a7f-ce21972a44e3)

Hit __Create index pattern__, and you are ready to analyze the data. Go to the __Discover__ tab in Kibana to take a look at the data.

![14](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F14.jpg?alt=media&token=fdd182aa-b4f6-4ede-b2d0-27cd9f3577ae)

Now we have set up the ELK data pipeline using Elasticsearch, Logstash, and Kibana.

Next step is to create Dashboard. Go to __Dashboard__ → __Create dashboard__ → __Create new__ → __Select Timelion__ (You can choose another type of visualization based on your requirement) for building time-series using functional expression as a sample. 
 
![15](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F15.jpg?alt=media&token=b0d19ecf-7981-44d1-b753-1580701ea714)

Then you can edit the Timelion expression as below:

![16](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F16.jpg?alt=media&token=e9a33489-02cf-4acb-b578-ff319eef5c37)

Timelion expression:
```
.es(index=itrc_1f,timefield='date:time',metric=avg:active_power).color(color=green).label(1F).fit(mode=carry).legend(position=ne).lines(width=1,fill=1,show=true)
```

You can save the visualization and return to dashboard that should look like:

![17](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F17.jpg?alt=media&token=a1322175-6035-4cd5-b0e2-a8d17a62a5ec)
 
Save your dashboard. Now you have created a dashboard with Kibana for analyzing your data and you can add new visualization to your dashboard later.

###	Grafana
In previous section you have visualized data from Elasticsearch using Kibana, now let’s use Grafana for data visualization from Elasticsearch.
You can download and install Grafana by following the [official website](https://grafana.com/grafana/download). 

Now assume that you have installed Grafana on your machine. To get started, open a web browser and navigate to http://localhost:3000 . This will bring up the user interface:

![18](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F18.jpg?alt=media&token=ae1a35d6-98b7-48f4-ad2c-dcf3b089b3be)

And default password for Grafana is “admin” for admin user.

After logging in, under __Configuration__ → __Data Sources__ → __Add data source__ → __Select Elasticsearch__ and the configuration should look like:

![19](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F19.jpg?alt=media&token=e3f02b48-9cc0-431d-a648-ed445d886d4a)

__Save & Test__, now you have created the data source for Elasticsearch.

Let’s create Dashboard, under __Create__ → __Dashboard__ → __Add new panel__ and the configuration should look like:

![20](https://firebasestorage.googleapis.com/v0/b/fir-demo-b5359.appspot.com/o/elastic%2F20.jpg?alt=media&token=0bb7d8fd-f405-4476-8c16-b663cec2044d)
 
Apply and Save the change. Now you have created the dashboard with Grafana and you can add the panel to existing dashboard later.

#	References
-	https://www.elastic.co/what-is/elasticsearch
-	https://logz.io/learn/complete-guide-elk-stack
-	https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html
-	https://www.elastic.co/guide/en/logstash/current/installing-logstash.html
-	https://www.elastic.co/guide/en/kibana/current/install.html
-	https://coralogix.com/log-analytics-blog/advanced-guide-to-kibana-timelion-functions/


