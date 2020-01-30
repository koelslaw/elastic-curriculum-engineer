# Instructor Notes
The whole point of this small class it to lead into other discussions. I will explore some of those discussions here like where and how to scale an elastic cluster. I will leave little pieces of information to help facilitate discussion.  

## Primer on Windows Security Events with Winlogbeat

### Prereq & Assumptions
- Configured AWS account at https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/
- Internet connectivity stable enough for RDP and SSH
- Administrator Access to both machines
- ssh or putty (windows)
- rdp

### EC2 Components
The recommended setup is to do prior to class. The intention is to have a windows server and an elastic machine for every student.

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
Starting with Logstash, we are going to create the logstash configuration file. Logstash it an optional componet

`sudo vi /etc/logstash/conf.d/logstash.conf`

This file defines the how logstash will input and output information.

starting with the input we have set this up to receive a beats connection. Logstash opens a port on 5044 to recvieve information for the beats clients. In our case it will be onyl one but it could be an entire organization of windows machines if needed.

```
input {
  beats {
    port => 5044
  }
}
```

Next we have the output section. Since the elastic node is all-in-one, we specify `localhost`. If we wanted to add additional nodes this would be a good place to start.


```
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}"
  }
}
```

  - Example of config file:
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

- Edit logstash service config `sudo vi /etc/logstash/logstash.yml`
  - Example of config file:
  ```
  ```

- Start Logstash`sudo systemctl start logstash`

- Check Status of logstash `sudo systemctl status logstash`

#### Elasticsearch

- Edit configuration file for Elasticsearch `sudo vi /etc/elasticsearch/elasticsearch.yml`

  - Example of config file:
  ```
  ```
- Start elasticsearch `sudo systemctl start elasticsearch`

#### Kibana

- Edit configuration file for Kibana
  - `sudo vi /etc/kibana/kibana.yml`

  - Example of config file:
  ```
  ```
- Start elasticsearch `sudo systemctl start kibana`

> Ensure the elastic services start without issues as Windows section requires that everything is functioning properly.


### Discussion about Nodes
For simplicity we are going to do everything on one node. However can take a small trek through what it will take to scale to a cluster. This most likely candidate for clustering is Elasticsearch and then Logstash.

  - For Elasticsearh, visit https://www.elastic.co/guide/en/elasticsearch/reference/current/add-elasticsearch-nodes.html.
  - For Logstash, visit https://www.elastic.co/guide/en/logstash/current/deploying-and-scaling.html

There are lots of "It Depends..." here but as this is aimed at a beginner audience the main points are:
- Availability (Shards are distributed across all the nodes, protecting the data)
- Throughput (Reads and writes will not overwhelm a cluster)
- Distubution of effort. (Shifting to dedicated node types: master nodes, ingest, etc.)

## Setup Windows EC2
- Once you have a remote desktop connection to your AWS instance.

- Download the latest 64-bit version Winlogbeat zip file from the downloads page, in our case its 7.5.2. `https://artifacts.elastic.co/downloads/beats/winlogbeat/winlogbeat-7.5.2-windows-x86_64.zip`
- Extract the zip file into `C:\Program Files` folder.
- Rename the `winlogbeat-<version>` folder to `Winlogbeat`.
- Open a PowerShell prompt as an **Administrator** (right-click on the PowerShell icon and select `Run As Administrator`).
- From the PowerShell prompt, run the following commands to install the service.
- To ensure that the script runs without issue we will execute the install of the service under a different execution policy via `PowerShell.exe -ExecutionPolicy UnRestricted -File .\install-service-winlogbeat.ps1`
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
