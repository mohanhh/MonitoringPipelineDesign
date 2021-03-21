**Describe a system/product/app you or your team built.**

I led the team which designed and built a Log Analytics and Monitoring Pipeline for my previous company. This was an ambitious project with a plan to replace Splunk used in the company with available resources. We also committed ourselves to using Open-Source software where-ever possible.

![Diagram, schematic Description automatically
generated](MonitoringPipelineDesign.png)
Mobile Applications are the biggest sources of revenue for the company. Consequently, monitoring the performance of these applications and doing root cause analysis for any bugs/problems in these applications is very important for the Management. My team used the available Hortonworks cluster and Elastic Search clusters to monitor these applications.

Mobile apps streamed their logs (using https GET and POST) to an external endpoint hosted on Heroku. The log messages with timestamps were primarily response times for transactions like payments or response times for search and query or response times to load pages and images. It also sent some MIS type of response times like time for cold start of the application along with OS and phone types. The applications combined several log statements in to one chunk and then posted the data either on a timer or when the log buffer was full to conserve bandwidth.

On Heroku, on a variable node cluster, a simple python application logged all these statements to a file. We then used a Heroku feature called logdrain which monitored this logfile and posted the messages to our Proxy Server.

Proxy server passed on this data to our 3 Node Flume cluster. We customized this Flume server and implemented our own code called Flue Interceptor which separated the log messages in to individual json messages and stored them in a Kafka Topic.

Our Kafka cluster was again a 3 Node cluster with a Zookeeper instance. There was no hashing setup and we let Kafka put the message in to any partition. We setup replication for each topic and matched number of partitions to number of executors available on our Spark cluster. Kafka retained data for 72 hours.

Next a 40 node Spark cluster ran a series of Streaming jobs to parse these logs and generate metrics. First streaming job which was called Preprocessor ran every minute and ran preconfigured Regular Expressions on each log to extract log type and response times. These extracted metrics were stored in a second Kafka Topic. Aggregation job ran every 15 minutes read these metrics and calculated 50th, 75th,90th and 95th percentiles and stored them in another Kafka topic. Alert job was triggered next which sent alerts to our Production Monitoring team if the percentile values were above the threshold set. We heavily used Spark checkpoints instead of using Kafka&#39;s enable.auto.committo ensure that every log statement was read at least once.

Raw logs and computed metrics from Kafka were shipped to a 25 node Elastic Cluster for Log Analysis and MIS reports. 3 Node Logstash cluster was used to ship logs from Kafka to Elastic.

Elastic Search cluster used Kibana to do Dashboarding.

Apart from this cluster, the team also spent time on developing a UI which helped setup the Regular Expressions for the Preprocessor to parse. The UI also had Graphical Interface which showed charts and logs associated with those points on the chart. The UI was developed in Backbone.js and used Java Spring boot backend.

We setup Hot and Warm nodes in Elastic Search cluster to improve responsiveness. Hot nodes are those which ingest the data and can return search results at the same time. For this Hot Nodes have fast SSD disks, while Warm Nodes hold older data which could be queried but are no longer being updated.

We also used this cluster to do Log aggregations for other Microservice owners. Ran light weight Elastic Log Agents on the production servers which shipped logs to our Elastic cluster through Logstash. Microservice owners could look at their logs on Kibana Dashboard.

Another use of the cluster was monitoring cloud costs of the company. We setup tasks to pull up billing data from the Cloud Provider and designed Dashboards to show the Cloud Usage and cloud costs per project.

Elastic Search:

3 Master Nodes:

Master Node: Tracks health of the cluster, decides which shards to allocate to which nodes.

Data Nodes: Hold data and perform data related operations such as Create, Read, Update, Delete (CRUD) search and aggregations.

Co-ordinating Node: Forwards search requests to the Data Nodes and collects the search results back, aggregates the results and sends it back to the client.

Elastic Search: Each index was broken in to 5 shards and 1 replica. Each index had a format

&quot;indexname&quot;-%yyyy-MM-dd

A day-old index was moved from Hot nodes to Warm nodes using a utility called Curator. Curator also deleted logs which were 3 months old.

Evaluating the design:

Some of the infrastructure decisions were made for us. Company already had a Hortonworks cluster which was paid for, being used by other groups and had a dedicated IT staff. We setup external Heroku cluster when we discovered the Proxy server had a limitation as to how many TCP connections it would accept. Flume served us better as it could write data to variety of sinks like Kafka, HDFS and we could also run Flume clients on other servers and ship the logs to our Flume cluster. Kafka was a solid backbone holding the pipeline together when downstream tasks had problem. Spark gave us a reliable Analytics Engine. We moved from Apache Solr to Elastic Search as Elastic had better interface and was faster.

The UI was customized according to specifications from our Production Support team. They liked the way it allowed them to paste a certain log statement and test Regular Expression. They also liked how it showed the graphs and allowed them to navigate to corresponding logs. We used Kibana more and more later on when Elastic enhanced Kibana features.

**Performance Testing:**

Each individual part of the pipeline was load tested. Flume cluster was tested by tool called NetStorm. Our performance team tested the flume end point by pumping in stored logs. We monitored CPU, load, memory, network, process and latency by using APM tools first and then monitored server health by loading Metric Beat from Elastic Search. We setup limit on memory and CPU usage to 80 % and 90 % respectively.

Next was Kafka. Kafka was optimized for availability by having 3 nodes and replication factor set to 2. It was optimized for high throughput by increasing number of partitions to 72. In our test basically Flume and Spark clusters were the limiting factor. Kafka was never really a problem as it easily could handle our 20000 messages per second rate. To increase durability, we used Acks = all for Kafka.

We tested Spark cluster by increasing the number of executors until the preprocessor job could handle 20000 messages per second within a minute. More than this, the Streaming job would start falling back. We benchmarked that for 20000 messages per second, the preprocessor job would finish on an average within 40 to 45 seconds.

Benchmarking Elastic search. Benchmarking 20000 writes per second using NetStorm and Elastic Search Put document command.
