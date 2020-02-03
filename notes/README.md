# Instructor Notes
The whole point of this small class it to lead into other discussions. I will explore some of those discussions here like where and how to scale an elastic cluster. I will leave little pieces of information to help facilitate discussion.  

## Primer on Windows Security Events with Winlogbeat

### Prereq & Assumptions
- Configured AWS account at https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/
- Internet connectivity stable enough for RDP and SSH
- Administrator Access to both machines
- ssh or putty (windows)
- rdp
- Git installed
- Code Editor

### EC2 Components
The recommended setup is done prior to class. Every Student will have a windows server and an elastic stack to send their logs.

### Setup Linux

#### Setup linux EC2 instance
Login to AWS https://console.aws.amazon.com/
- Look under `all services` and find the `compute` heading. From there select `EC2`
- on the left pane select `instances` and then `launch instance`
- Select the `Ubuntu Server 16.04 LTS (HVM), SSD Volume Type` option.
- Select `t2.medium` instance type  then select the `Next: Configure Instance Details` button
- unless you anything special needs to be done in this area accept the defaults and select `Next: Add Storage`
- A couple of factors will determine how much storage is required. I have found 250 GB is sufficient for a single node setup that hass all the elastic componenets. As nodes are added then the hard disks could be smaller. once you have decided the size then select `Next: Add Tags`
- If you have several students adding tags may be useful to assist with troublshooting and orginization. Once completed select `Next: Configure Security Group`

- For the very first instance you will set the security group. This security grou can be reused for the subsequent instances.

| Type            | Protocol | Port Range | Source         | Description |
|-----------------|----------|------------|----------------|-------------|
| SSH             | TCP      | 22         | 0.0.0.0/0      | SSH Access  |
| Custom TCP Rule | TCP      | 5044       | 172.31.32.0/20 | Beats       |
| Custom TCP Rule | TCP      | 5601       | 0.0.0.0/0      | Kibana      |
| All ICMP - IPv4 | All      | N/A        | 0.0.0.0/0      | Ping        |

> Why is the addressing setup this way? SSH and Kibana are set to everywhere because we dont know what ip the student is comming from. ICMP facilitates ping. Port 5044 is for winlogbeat. We set a specific IP range as we want to limit it to just us.

- Review the details of the instance and select `launch` when you satisfied with the results

- Repeat as required

#### Install Elastic Components Via APT on linux ec2 instance

This is the recommended start of the class. It's enough the student to get their hands dirty and lead to other discussions. My recommendation is to use a single key for all the instances in the class. For each class the key needs to be changed. My biggest is it gets the users used to authenticating with ssh keys, which are better than a standard username and password. Its also less prone to mistakes as its less complex. Finally, if they use aws in the future they are more comfortable.

- Log into the linux instance via ssh
  - linux users can use  `ssh -i "PathtoCert.pem" ubuntu@some-provided-address.compute-1.amazonaws.com`

  - windows users can follow the instructions here for putty https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html


Installing via rpm instead of an archive is a matter of preference here. In order to simplfy things a little I am using apt to load the required packages. To sart the process we mush add elastic keys with `wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -` Amazon EC2 instances should already have have the `apt-transport-https` package installed but just in case it does not run `sudo apt-get install apt-transport-https`. An entry to the sources list for ubuntu needs to be added. This can be done with `echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list`. We need to update and install the Java environment with `sudo apt-get update && sudo apt-get install default-jre`. Using `default-jre` *could* install an unsupported version of Java. Make sure it installs a supported version of Java by visiting https://www.elastic.co/support/matrix#matrix_jvm. Now that everything is setup we can actually install something. Enter `sudo apt-get install elasticsearch logstash kibana` to install the Elastic components.

#### Logstash
Starting with Logstash, we are going to create the logstash configuration file. Logstash it an optional component. It is not required in this scenario. However, there comes a time when you want add capabilities. If you have Logstash already running then reconfiguration will be much easier.

`sudo vi /etc/logstash/conf.d/logstash.conf`

This file defines the how logstash will input and output information.

Starting with the input we have set this up to receive a beats connection. Logstash opens a port on 5044 to recvieve information for the beats clients. In our case it will be only one but it could be an entire organization of windows machines if needed.

```
input {
  beats {
    port => 5044
  }
}
```

Next we have the output section. Since the elastic node is all-in-one, we specify `localhost`. If we wanted to add additional Elasticsearch nodes this would be a good place to start.


```
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}"
  }
}
```

  - Example of our config file:
    ```
    input {
      beats {
        port => 5044
      }
    }

    output {
      elasticsearch {
        hosts => ["http://localhost:9200"]
        index => "%{[@metadata][beat]}-%{[@metadata][version]}"
      }
    }

  ```

If there were any special things that needed to be done to logstash it would be performed in `/etc/logstash/logstash.yml`. This can be accomplished by `sudo vi /etc/logstash/logstash.yml` As we have no changes, we will leave this alone.


  - Example of config file:

  ```yml
  #
  # var.PLUGIN_TYPE.PLUGIN_NAME.KEY
  #
  # modules:
  #
  # ------------ Cloud Settings ---------------
  # Define Elastic Cloud settings here.
  # Format of cloud.id is a base64 value e.g. dXMtZWFzdC0xLmF3cy5mb3VuZC5pbyRub3RhcmVhbCRpZGVudGlmaWVy
  # and it may have an label prefix e.g. staging:dXMtZ...
  # This will overwrite 'var.elasticsearch.hosts' and 'var.kibana.host'
  # cloud.id: <identifier>
  #
  # Format of cloud.auth is: <user>:<pass>
  # This is optional
  # If supplied this will overwrite 'var.elasticsearch.username' and 'var.elasticsearch.password'
  # If supplied this will overwrite 'var.kibana.username' and 'var.kibana.password'
  # cloud.auth: elastic:<password>
  #
  # ------------ Queuing Settings --------------
  #
  # Internal queuing model, "memory" for legacy in-memory based queuing and
  # "persisted" for disk-based acked queueing. Defaults is memory
  #
  # queue.type: memory
  #
  # If using queue.type: persisted, the directory path where the data files will be stored.
  # Default is path.data/queue
  #
  # path.queue:
  #
  # If using queue.type: persisted, the page data files size. The queue data consists of
  # append-only data files separated into pages. Default is 64mb
  #
  # queue.page_capacity: 64mb
  #
  # If using queue.type: persisted, the maximum number of unread events in the queue.
  # Default is 0 (unlimited)
  #
  # queue.max_events: 0
  #
  # If using queue.type: persisted, the total capacity of the queue in number of bytes.
  # If you would like more unacked events to be buffered in Logstash, you can increase the
  # capacity using this setting. Please make sure your disk drive has capacity greater than
  # the size specified here. If both max_bytes and max_events are specified, Logstash will pick
  # whichever criteria is reached first
  # Default is 1024mb or 1gb
  #
  # queue.max_bytes: 1024mb
  #
  # If using queue.type: persisted, the maximum number of acked events before forcing a checkpoint
  # Default is 1024, 0 for unlimited
  #
  # queue.checkpoint.acks: 1024
  #
  # If using queue.type: persisted, the maximum number of written events before forcing a checkpoint
  # Default is 1024, 0 for unlimited
  #
  # queue.checkpoint.writes: 1024
  #
  # If using queue.type: persisted, the interval in milliseconds when a checkpoint is forced on the head page
  # Default is 1000, 0 for no periodic checkpoint.
  #
  # queue.checkpoint.interval: 1000
  #
  # ------------ Dead-Letter Queue Settings --------------
  # Flag to turn on dead-letter queue.
  #
  # dead_letter_queue.enable: false

  # If using dead_letter_queue.enable: true, the maximum size of each dead letter queue. Entries
  # will be dropped if they would increase the size of the dead letter queue beyond this setting.
  # Default is 1024mb
  # dead_letter_queue.max_bytes: 1024mb

  # If using dead_letter_queue.enable: true, the directory path where the data files will be stored.
  # Default is path.data/dead_letter_queue
  #
  # path.dead_letter_queue:
  #
  # ------------ Metrics Settings --------------
  #
  # Bind address for the metrics REST endpoint
  #
  # http.host: "127.0.0.1"
  #
  # Bind port for the metrics REST endpoint, this option also accept a range
  # (9600-9700) and logstash will pick up the first available ports.
  #
  # http.port: 9600-9700
  #
  # ------------ Debugging Settings --------------
  #
  # Options for log.level:
  #   * fatal
  #   * error
  #   * warn
  #   * info (default)
  #   * debug
  #   * trace
  #
  # log.level: info
  path.logs: /var/log/logstash
  #
  # ------------ Other Settings --------------
  #
  # Where to find custom plugins
  # path.plugins: []
  #
  # Flag to output log lines of each pipeline in its separate log file. Each log filename contains the pipeline.name
  # Default is false
  # pipeline.separate_logs: false
  #
  # ------------ X-Pack Settings (not applicable for OSS build)--------------
  #
  # X-Pack Monitoring
  # https://www.elastic.co/guide/en/logstash/current/monitoring-logstash.html
  #xpack.monitoring.enabled: false
  #xpack.monitoring.elasticsearch.username: logstash_system
  #xpack.monitoring.elasticsearch.password: password
  #xpack.monitoring.elasticsearch.hosts: ["https://es1:9200", "https://es2:9200"]
  #xpack.monitoring.elasticsearch.ssl.certificate_authority: [ "/path/to/ca.crt" ]
  #xpack.monitoring.elasticsearch.ssl.truststore.path: path/to/file
  #xpack.monitoring.elasticsearch.ssl.truststore.password: password
  #xpack.monitoring.elasticsearch.ssl.keystore.path: /path/to/file
  #xpack.monitoring.elasticsearch.ssl.keystore.password: password
  #xpack.monitoring.elasticsearch.ssl.verification_mode: certificate
  #xpack.monitoring.elasticsearch.sniffing: false
  #xpack.monitoring.collection.interval: 10s
  #xpack.monitoring.collection.pipeline.details.enabled: true
  #
  # X-Pack Management
  # https://www.elastic.co/guide/en/logstash/current/logstash-centralized-pipeline-management.html
  #xpack.management.enabled: false
  #xpack.management.pipeline.id: ["main", "apache_logs"]
  #xpack.management.elasticsearch.username: logstash_admin_user
  #xpack.management.elasticsearch.password: password
  #xpack.management.elasticsearch.hosts: ["https://es1:9200", "https://es2:9200"]
  #xpack.management.elasticsearch.ssl.certificate_authority: [ "/path/to/ca.crt" ]
  #xpack.management.elasticsearch.ssl.truststore.path: /path/to/file
  #xpack.management.elasticsearch.ssl.truststore.password: password
  #xpack.management.elasticsearch.ssl.keystore.path: /path/to/file
  #xpack.management.elasticsearch.ssl.keystore.password: password
  #xpack.management.elasticsearch.ssl.verification_mode: certificate
  #xpack.management.elasticsearch.sniffing: false
  #xpack.management.logstash.poll_interval: 5s

  ```

- Start Logstash`sudo systemctl start logstash`

- Check Status of logstash `sudo systemctl status logstash`

#### Elasticsearch
You may have to elevate privelages with `sudo -s`to edit configuration file for Elasticsearch `sudo vi /etc/elasticsearch/elasticsearch.yml`. Like everything else with elastic stack there are a lot of thigns we can tweak. for simplicity we are only going to change a few things.

Similar to logstash the default configuration will work Once you start to have more nodes then there are some options that you will want to take into account. Setting the cluster information can be done here. The cluster will be the same across all the configs.

```yml
---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: my-application
#
```

Below is the section that includes the name for the node. When setting up a cluster each node must have a unique name.

> NOTE: Hostname and Node name need to match

```yml
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
#node.name: node-1
#
```

In the network section you can set ip that elasticsearc will bind too.

```yml
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: 0.0.0.0
#
# Set a custom port for HTTP:
#
#http.port: 9200
#
# For more information, consult the network module documentation.
#
```

As more nodes are added, you will need to open ports 9200 (REST API) and 9300 (Node Communication). As we have everything one one machince and only a single instance of elasticsearch we do not need to accomplish this. Once we reach the point we need additional nodes, the section below is where the configuration will happen.

```yml
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.seed_hosts: ["18.214.204.121"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
#cluster.initial_master_nodes: ["node-1", "node-2"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
```

  - Example a of full config file:

  ```yml
  # ======================== Elasticsearch Configuration =========================
  #
  # NOTE: Elasticsearch comes with reasonable defaults for most settings.
  #       Before you set out to tweak and tune the configuration, make sure you
  #       understand what are you trying to accomplish and the consequences.
  #
  # The primary way of configuring a node is via this file. This template lists
  # the most important settings you may want to configure for a production cluster.
  #
  # Please consult the documentation for further information on configuration options:
  # https://www.elastic.co/guide/en/elasticsearch/reference/index.html
  #
  # ---------------------------------- Cluster -----------------------------------
  #
  # Use a descriptive name for your cluster:
  #
  #cluster.name: my-application
  #
  # ------------------------------------ Node ------------------------------------
  #
  # Use a descriptive name for the node:
  #
  #node.name: node-1
  #
  # Add custom attributes to the node:
  #
  #node.attr.rack: r1
  #
  # ----------------------------------- Paths ------------------------------------
  #
  # Path to directory where to store the data (separate multiple locations by comma):
  #
  path.data: /var/lib/elasticsearch
  #
  # Path to log files:
  #
  path.logs: /var/log/elasticsearch
  #
  # ----------------------------------- Memory -----------------------------------
  #
  # Lock the memory on startup:
  #
  #bootstrap.memory_lock: true
  #
  # Make sure that the heap size is set to about half the memory available
  # on the system and that the owner of the process is allowed to use this
  # limit.
  #
  # Elasticsearch performs poorly when the system is swapping the memory.
  #
  # ---------------------------------- Network -----------------------------------
  #
  # Set the bind address to a specific IP (IPv4 or IPv6):
  #
  #network.host: 0.0.0.0
  #
  # Set a custom port for HTTP:
  #
  #http.port: 9200
  #
  # For more information, consult the network module documentation.
  #
  # --------------------------------- Discovery ----------------------------------
  #
  # Pass an initial list of hosts to perform discovery when this node is started:
  # The default list of hosts is ["127.0.0.1", "[::1]"]
  #
  #discovery.seed_hosts: ["18.214.204.121"]
  #
  # Bootstrap the cluster using an initial set of master-eligible nodes:
  #
  #cluster.initial_master_nodes: ["node-1", "node-2"]
  #
  # For more information, consult the discovery and cluster formation module documentation.
  #
  # ---------------------------------- Gateway -----------------------------------
  #
  # Block initial recovery after a full cluster restart until N nodes are started:
  #
  #gateway.recover_after_nodes: 3
  #
  # For more information, consult the gateway module documentation.
  #
  # ---------------------------------- Various -----------------------------------
  #
  # Require explicit names when deleting indices:
  #
  #action.destructive_requires_name: true

  ```
Now that we have elasticsearch installed and configured lets go ahead and start elasticsearch `sudo systemctl start elasticsearch`.

#### Kibana
Last but not least on our aws instance is kibana. Just like the rest there is a configuration file. To edit the configuration file, use `sudo vi /etc/kibana/kibana.yml`. This machine does not reside on the same network as the students. When building we have 2 options here. We can bind it to a specific IP address or 0.0.0.0. For production binding it to a specific address is best. These instances are temporary so we can not worry about what IP it decides to use. This can be accomplished with changing the `server.host:` to ` "0.0.0.0"` instead of localhost.

  - Example of our full config file:

  ```yml
  # Kibana is served by a back end server. This setting specifies the port to use.
  #server.port: 5601

  # Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
  # The default is 'localhost', which usually means remote machines will not be able to connect.
  # To allow connections from remote users, set this parameter to a non-loopback address.
  server.host: "0.0.0.0"

  # Enables you to specify a path to mount Kibana at if you are running behind a proxy.
  # Use the `server.rewriteBasePath` setting to tell Kibana if it should remove the basePath
  # from requests it receives, and to prevent a deprecation warning at startup.
  # This setting cannot end in a slash.
  #server.basePath: ""

  # Specifies whether Kibana should rewrite requests that are prefixed with
  # `server.basePath` or require that they are rewritten by your reverse proxy.
  # This setting was effectively always `false` before Kibana 6.3 and will
  # default to `true` starting in Kibana 7.0.
  #server.rewriteBasePath: false

  # The maximum payload size in bytes for incoming server requests.
  #server.maxPayloadBytes: 1048576

  # The Kibana server's name.  This is used for display purposes.
  #server.name: "your-hostname"

  # The URLs of the Elasticsearch instances to use for all your queries.
  #elasticsearch.hosts: ["http://localhost:9200"]

  # When this setting's value is true Kibana uses the hostname specified in the server.host
  # setting. When the value of this setting is false, Kibana uses the hostname of the host
  # that connects to this Kibana instance.
  #elasticsearch.preserveHost: true

  # Kibana uses an index in Elasticsearch to store saved searches, visualizations and
  # dashboards. Kibana creates a new index if the index doesn't already exist.
  #kibana.index: ".kibana"

  # The default application to load.
  #kibana.defaultAppId: "home"

  # If your Elasticsearch is protected with basic authentication, these settings provide
  # the username and password that the Kibana server uses to perform maintenance on the Kibana
  # index at startup. Your Kibana users still need to authenticate with Elasticsearch, which
  # is proxied through the Kibana server.
  #elasticsearch.username: "kibana"
  #elasticsearch.password: "pass"

  # Enables SSL and paths to the PEM-format SSL certificate and SSL key files, respectively.
  # These settings enable SSL for outgoing requests from the Kibana server to the browser.
  #server.ssl.enabled: false
  #server.ssl.certificate: /path/to/your/server.crt
  #server.ssl.key: /path/to/your/server.key

  # Optional settings that provide the paths to the PEM-format SSL certificate and key files.
  # These files validate that your Elasticsearch backend uses the same key files.
  #elasticsearch.ssl.certificate: /path/to/your/client.crt
  #elasticsearch.ssl.key: /path/to/your/client.key

  # Optional setting that enables you to specify a path to the PEM file for the certificate
  # authority for your Elasticsearch instance.
  #elasticsearch.ssl.certificateAuthorities: [ "/path/to/your/CA.pem" ]

  # To disregard the validity of SSL certificates, change this setting's value to 'none'.
  #elasticsearch.ssl.verificationMode: full

  # Time in milliseconds to wait for Elasticsearch to respond to pings. Defaults to the value of
  # the elasticsearch.requestTimeout setting.
  #elasticsearch.pingTimeout: 1500

  # Time in milliseconds to wait for responses from the back end or Elasticsearch. This value
  # must be a positive integer.
  #elasticsearch.requestTimeout: 30000

  # List of Kibana client-side headers to send to Elasticsearch. To send *no* client-side
  # headers, set this value to [] (an empty list).
  #elasticsearch.requestHeadersWhitelist: [ authorization ]

  # Header names and values that are sent to Elasticsearch. Any custom headers cannot be overwritten
  # by client-side headers, regardless of the elasticsearch.requestHeadersWhitelist configuration.
  #elasticsearch.customHeaders: {}

  # Time in milliseconds for Elasticsearch to wait for responses from shards. Set to 0 to disable.
  #elasticsearch.shardTimeout: 30000

  # Time in milliseconds to wait for Elasticsearch at Kibana startup before retrying.
  #elasticsearch.startupTimeout: 5000

  # Logs queries sent to Elasticsearch. Requires logging.verbose set to true.
  #elasticsearch.logQueries: false

  # Specifies the path where Kibana creates the process ID file.
  #pid.file: /var/run/kibana.pid

  # Enables you specify a file where Kibana stores log output.
  #logging.dest: stdout

  # Set the value of this setting to true to suppress all logging output.
  #logging.silent: false

  # Set the value of this setting to true to suppress all logging output other than error messages.
  #logging.quiet: false

  # Set the value of this setting to true to log all events, including system usage information
  # and all requests.
  #logging.verbose: false

  # Set the interval in milliseconds to sample system and process performance
  # metrics. Minimum is 100ms. Defaults to 5000.
  #ops.interval: 5000

  # Specifies locale to be used for all localizable strings, dates and number formats.
  # Supported languages are the following: English - en , by default , Chinese - zh-CN .
  #i18n.locale: "en"

  ```
Save the file and Start kibana with `sudo systemctl start kibana`

>NOTE: Ensure the elastic services start without issues as Windows section requires that everything is functioning properly.

To ensure everything is up and running, enter `sudo systemctl status logstash elasticsearch kibana`

```
● logstash.service - logstash
   Loaded: loaded (/etc/systemd/system/logstash.service; disabled; vendor preset: enabled)
   Active: active (running) since Sat 2020-02-01 15:16:04 UTC; 5s ago
 Main PID: 18797 (java)
    Tasks: 14
   Memory: 288.9M
      CPU: 6.955s
   CGroup: /system.slice/logstash.service
           └─18797 /usr/bin/java -Xms1g -Xmx1g -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly -Djava.awt.he

Feb 01 15:16:04 ip-172-31-92-226 systemd[1]: Started logstash.

● elasticsearch.service - Elasticsearch
   Loaded: loaded (/usr/lib/systemd/system/elasticsearch.service; disabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-01-30 20:04:32 UTC; 1 day 19h ago
     Docs: http://www.elastic.co
 Main PID: 14245 (java)
    Tasks: 60
   Memory: 1.3G
      CPU: 6min 10.151s
   CGroup: /system.slice/elasticsearch.service
           ├─14245 /usr/share/elasticsearch/jdk/bin/java -Des.networkaddress.cache.ttl=60 -Des.networkaddress.cache.negative.ttl=10 -XX:+AlwaysPreTouch -Xss1
           └─14339 /usr/share/elasticsearch/modules/x-pack-ml/platform/linux-x86_64/bin/controller

Jan 30 20:04:17 ip-172-31-92-226 systemd[1]: Starting Elasticsearch...
Jan 30 20:04:18 ip-172-31-92-226 elasticsearch[14245]: OpenJDK 64-Bit Server VM warning: Option UseConcMarkSweepGC was deprecated in version 9.0 and will lik
Jan 30 20:04:32 ip-172-31-92-226 systemd[1]: Started Elasticsearch.

● kibana.service - Kibana
   Loaded: loaded (/etc/systemd/system/kibana.service; disabled; vendor preset: enabled)
   Active: active (running) since Sat 2020-02-01 15:16:04 UTC; 5s ago
 Main PID: 18802 (node)
    Tasks: 11
   Memory: 279.0M
      CPU: 4.017s
   CGroup: /system.slice/kibana.service
           └─18802 /usr/share/kibana/bin/../node/bin/node /usr/share/kibana/bin/../src/cli -c /etc/kibana/kibana.yml

Feb 01 15:16:10 ip-172-31-92-226 kibana[18802]: {"type":"log","@timestamp":"2020-02-01T15:16:10Z","tags":["info","plugins","security"],"pid":18802,"message":
```

### Discussion about Nodes
For simplicity we are going to do everything on one node. However can take a small trek through what it will take to scale to a cluster. This most likely candidate for clustering is Elasticsearch and then Logstash.

  - For Elasticsearh, visit https://www.elastic.co/guide/en/elasticsearch/reference/current/add-elasticsearch-nodes.html.
  - For Logstash, visit https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html

There are lots of "It Depends..." here but as this is aimed at a beginner audience the main points are:
- Availability - (Shards are distributed across all the nodes, protecting the data)
- Throughput - (Reads and writes will not overwhelm a cluster)
- Distribution of effort - (Shifting to dedicated node types: master nodes, ingest, etc.)

## Setup Windows EC2
Select your Windows EC2 instance and select `Connect`. That will open a small window. Click on `Get Password` to start the password retrieval process. It will ask you for the key in order to decrypt the password for the instance. Select `Elasticsearch.pem` and then `Decrypt Password`. You will be presented with the password to your instance. use the information provided to access the windows EC2 instance.

Once you have a remote desktop connection to your AWS instance. Download the latest 64-bit version Winlogbeat zip file from the downloads page, in our case its 7.5.2. `https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-7.5.2-windows-x86_64.zip` From your downloads folder, extract the zip file into `C:\Program Files` folder. Rename the extracted file with the name `winlogbeat-<some_version>` folder to `Winlogbeat`. Open a PowerShell prompt as an **Administrator**. From start menu, right-click on the PowerShell icon and select `Run As Administrator`. To ensure that the script runs without issue we will execute the install of the service under a different execution policy via `PowerShell.exe -ExecutionPolicy UnRestricted -File .\install-service-winlogbeat.ps1`
- Edit the config file for Winlogbeat notepad
  - Example of winlogbeat config file
  ```
  ```
- Comment out any logs that you do not require.
- disable the elasticsearch output for Beats by commenting out the following line

```
```
- Set the output to use logstash instead

```yml
#----------------------------- Logstash output --------------------------------
output.logstash:
  hosts: ["<<<IPofLogstashNode>>>:5044"]
```  
- Since we are sending out data to logstash instead of elasticsearch then we need to manually setup the index template.

```
.\winlogbeat.exe setup --index-management -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["<<<IPofLogstashNode>>>:9200"]'
```

- Setup dashboards for .To load dashboards when the Logstash output is enabled, you need to temporarily disable the Logstash output and enable Elasticsearch. To connect to a secured Elasticsearch cluster, you also need to pass Elasticsearch credentials.
```
.\winlogbeat.exe setup -e `
  -E output.logstash.enabled=false `
  -E output.elasticsearch.hosts=['<<<IPofElasticsearchNode>>>:9200'] `
  -E setup.kibana.host=<<<IPofKibanaNode>>>:5601
```
- Test the winlogbeat config `C:\Program Files\Winlogbeat> .\winlogbeat.exe test config -c .\winlogbeat.yml -e`
