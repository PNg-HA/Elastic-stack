[![Elastic Stack version](https://img.shields.io/badge/Elastic%20Stack-8.0.1-00bfb3?style=flat&logo=elastic-stack)](https://www.elastic.co/blog/category/releases)
# Elastic Stack
NT132 (Networks and Systems Administration) Project - ELK stack, referenced by https://www.elastic.co

Elastic Stack, also known as ELK Stack, is a stack that comprises of three popular projects: 

* Elasticsearch
* Logstash
* Kibana
 
and other components such as:
- Beat
- APM

Elastic Stack is used to take data from any source then search by Elasticsearch, analyze and visualize by Kibana. 
Elastic Stack can also be deployed as a Cloud service supported on AWS, Google Cloud, and Azure, or as an on-prem installation.
![image](https://user-images.githubusercontent.com/93396414/199807946-5c29eede-8516-49fe-ae20-ce019d3f6be6.png)


In this repository, I will only go into detail about Elasticsearch and Kibana.

## Contents
1. [Architecture design](#Architecture-design)
2. [Components](#Components)
	* [Elasticsearch](#Elasticsearch)
		* [Nodes and indices](#Nodes-and-indices)
		* [In case of disaster](#In-case-of-disaster)
	* [Kibana](#Kibana)
3. [Setup](#Setup)
	* [Host requirements](#Host-requirements)
	* [Setup steps](#Setup-steps)
4. [Backup and restore](#Backup-and-restore)
	* [Backup](#Backup)
	* [Restore](#Restore)
## Architecture design
![image](https://github.com/PNg-HA/Elastic-Stack/assets/93396414/9ddf29ea-fac1-4ce8-9c54-3526a96a6625)
**Scenario**: VM1 and VM2 are Ubuntu servers. VM2 for Beat agent, sending metrics of VM2 to VM1. VM1 for ELK stack and AWS S3 setup.

## Components
### Elasticsearch
You should think Elasticsearch as the heart of the Elastic Stack, which has near real-time search and analytics for all types of data. Elasticsearch can store and index structured or unstructured text, numerical data, or geospatial data, in a way that supports fast searches. Elasticsearch provides a REST API that enables you to store data in Elasticsearch, retrieve, and analyze it.

![image](https://user-images.githubusercontent.com/93396414/199810719-4a8915fe-8dd3-4d93-aa13-c93f937d1015.png)

You can also use the Elasticsearch clients to access data in Elasticsearch directly from common programming languages, such as Python. Perl, Go, Java, Ruby, and others.

#### Nodes and indices
Complex data structures are serialized as JSON documents, distributed across the cluster (in case you deploy a cluster) and accessed from any node.

![image](https://user-images.githubusercontent.com/93396414/199811956-830049c5-d648-4927-8ae3-945adb947625.png)

Elasticsearch uses a data structure called an inverted **index** to store documents. An inverted index lists every unique word that appears in any document and identifies all of the documents each word occurs in. Index can be thought of as an optimized collection of documents and each document is a collection of fields, which are the key-value pairs that contain your data.

![image](https://user-images.githubusercontent.com/93396414/199812745-ad733a59-7631-4d37-8a72-35c89095bab3.png)

#### In case of disaster
To avoid a single point of failure, Elastic supports **Cross-cluster replication**, which automatically synchronizes indices from your primary cluster to a secondary remote cluster that can serve as a hot backup. If the primary cluster fails, the secondary cluster can take over.

### Kibana
Kibana is the tool to visualize the Elasticsearch data and to manage the Elastic Stack. Kibana is also the home for the Elastic Enterprise Search, Elastic Observability and Elastic Security solutions. Kibana can:
  - Create dashboard
  - Design graph patterns and relationship
  - predict, & detect behavior
  - and so on.	

## Setup
### Host requirements
* [Docker Engine][docker-install] version **18.06.0** or newer (8/12/2023 test on v19.03.9)
* [Docker Compose][compose-install] version **1.26.0** or newer (including [Compose V2][compose-v2]) (8/12/2023 test on v2.21.0)
* More than **6 GB** of RAM (because the host consumes lots of memory when runs ELK multi nodes) (8/12/2023 docker v23 results in error "kernel does not support swap limit capabilities or the cgroup is not mounted. memory limited without swap")
### Setup steps
The filebeat installation is in https://github.com/PNg-HA/ELK-Run-FileBeat.
> **Summary**  
> This document will show how to use Docker and Docker Compose to install 3 
> Elasticsearch nodes in 1 cluser, Kibana to visual and management them, and how to backup 
> and restore the database. The way to install is the final, after hours of fixing bugs during the installation.
> The **demo** is at [here](https://drive.google.com/drive/folders/1b_kFoLlb9tGd0_QrsFezDVBxU-MEX-HW?usp=sharing).

  1. In a Ubuntu server, create a directory and move into it:

	
    $ mkdir docker-ELK && cd $_
    
	
  2. Make a `docker-compose.yml`file with the contents from [`docker-compose.yml`](docker-compose.yml)

   > **Note**  
   > This file is at first referenced from the [docker-compose file][docker-compose-file] of [elastic.co], however I have edited it to fix bugs when build the 
   > docker
   > compose up. What I have edited is setting up JVM heap size in `environment` tag `- ES_JAVA_OPTS=-Xms750m -Xmx750m` in each Elasticsearch node, which
   > prevents the nodes from exiting.

  3. Create `.env` file with the contents from [`.env`](.env)

   > **Note**  
   > This file is at first referenced from the [.env file][.env-file] of [elastic.co], however I have edited it. Beside `password` and `version`, I have also 
   > edited 
   > `MEM_LIMIT` = **6442450944** bytes, which is more than **6 GB**.

  4. Increase the limit of `mmap` (virtual memory of the host):

	$ sudo sysctl -w vm.max_map_count=524288
	
  5. At the docker-ELK directory, run the command:
	
	$ docker compose up -d 

   Wait for about 3 minutes for the ELK to setup.
   > **Note**  
   > If there are problems, run `$ docker compose logs -f <service>` to observe the logs and exit code (search google for it). If you meet the **137** exit 
   > code, I recommend [this][exit-code-137] .
 
  6. When all 3 nodes are healthy, access the Kibana web UI by opening http://IP-address-of-the-host:5601 in a web browser and use the following (default) credentials to log in:

	user: elastic
	password: abc123 
	
  ![image](https://user-images.githubusercontent.com/93396414/199725476-86a1d7e2-3516-4070-90e3-dd0d62a746c9.png)

The successful initial setup should result in:

  ![image](https://user-images.githubusercontent.com/93396414/199725719-a077d76e-78d3-4e93-ba9b-2e35ac8da378.png)

## Backup and restore
### Backup
	
  1. Display the list of ELK containers:
	
	$docker ps
	
  2. Access to **each** container with `root`:

	$docker exec -u 0 -it <esearch container id> /bin/bash
	$apt-get update
	
  3. Install the `nano` text editor:
	
	$apt-get install nano
	
  4. `Ctrl–D` to quit the container and get access to it again with user `elasticsearch`:

	docker exec -it <esearch container id> /bin/bash

  5. Create a backup directory:
	
	mkdir backup_repo

  6. Config the `elasticsearch.yml` file to create a path to the backup directory:

	nano config/elasticsearch.yml

  7. Add `path.repo: /usr/share/elasticsearch/backup_repo` to the file. Save the file with `Ctrl-S` and exit with `Ctrl-X`.
  > **Note**  
  > To know the location of directory `backup_repo`, use command `pwd`
  
Do these above steps for **3** elasticsearch node and restart them. Then go to the Kibana web UI. 

  1. Go to **Stack Management** -> **Snapshot and Restore** -> **Repositories**

  ![image](https://user-images.githubusercontent.com/93396414/199712256-0c617f51-7ee9-4c2f-a9e7-a7355b53738b.png)

  2. Select **Register repository**. 
  3. Name **Repository name** as `demo`.
  4. At **Repository type**, select **Shared file system** and **Next**.
  
  ![image](https://user-images.githubusercontent.com/93396414/199716916-612ef447-49b9-434a-8f51-695472d90408.png)
  
  5. At the `File system location`, type: `/usr/share/elasticsearch/backup_repo`. Ignore other fields and **Save**.
  
  ![image](https://user-images.githubusercontent.com/93396414/199718410-c2adf821-ec40-4035-8dad-773cedf0d0a8.png)
  
  6. Then move to **Policies** -> **Create policy**.
  
  8. At step 1, **Logistics**, type as your choice and **Next**.
  
  ![image](https://user-images.githubusercontent.com/93396414/199718947-6c19bf78-c41c-46da-a914-644f3e32c3dd.png)
  
  8. At step 2, **Snapshot settings**, in **Data streams and indices**, choose only my index: `favor_candy`
  
  ![image](https://user-images.githubusercontent.com/93396414/199720039-dba0cd14-c73b-409f-bb77-da08e789a58d.png)

   > **Note** 
   > If the **Include global state** show up, then turn on it.
   Then **Next**. 
   
  9. Step 3 is not important in this document. I only set the `Maximum count` to 4. Then **Next**.
   
  ![image](https://user-images.githubusercontent.com/93396414/199720548-f2b418f6-6f72-4338-b6b4-39925593b373.png)
   
  10. At step 4, **Review**, check again.
   
  ![image](https://user-images.githubusercontent.com/93396414/199720718-10ef5b4f-f0cd-44a1-9dc5-4748402b2939.png)
   
   If nothing to edit, then **Create policy**.
   
  11. A pop-up shows up, choose **run now**.
   
  ![image](https://user-images.githubusercontent.com/93396414/199721156-c801d522-267b-4f82-8c7a-f53d23b0604c.png)
   
   The successful snapshot will look like this:
   
  ![image](https://user-images.githubusercontent.com/93396414/199721328-d5850f3c-a27a-4267-a437-ded58865db18.png)
   
### Restore:

> **Note**  
> Elastic has strict rules about restoring indices. It will not allow to restore system files nor the other indices unless they are deleted. This part will restore the indices that
> I have written in `Dev Tools`

  1. Now select **Snapshot**
  
  ![image](https://user-images.githubusercontent.com/93396414/199721855-d94ba8f9-9c4d-467a-893b-6ace4f3e4d75.png)

   In the picture, there are many snapshots. Choose one.
  
  2. A pop-up shows up.

  ![image](https://user-images.githubusercontent.com/93396414/199722058-9c843f15-8fb1-42b2-974f-94bba79be647.png)
  
   Then **Restore**.
  
  3. Pass 3 steps and select **Restore snapshot**.
  
  ![image](https://user-images.githubusercontent.com/93396414/199723358-56a249db-5a6b-425f-812f-9b0b3fa759a5.png)
 
   But it refuses to restore because there are one same index named `favor_candy` in my node.
     
  ![image](https://user-images.githubusercontent.com/93396414/199723409-a3fa8ef0-c941-4e7a-adaf-a25c0cf23f48.png)

  4. Go to `Dev Tools` -> type and run **Delete favor_candy**.
  
  ![image](https://user-images.githubusercontent.com/93396414/199722927-e8fb6923-2eb2-4558-9569-1cfd4d2550ab.png)
  
   > **Note**
   > I could delete this index because the kibana_user created it and is permited to delete it. However, there are indices
   > that can not be deleted unless kibanba_user is set to do it. 
  
  5. Do again step 2 and 3. 
  
     The successful restore should be looked like this:
     
  ![image](https://user-images.githubusercontent.com/93396414/199724903-7bd9521f-4c4c-46e4-acc2-b09bfba1123d.png)

  


















[elastic.co]: https://elastic.co
[docker-compose-file]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-file
[.env-file]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-env-file
[elk-stack]: https://www.elastic.co/what-is/elk-stack
[xpack]: https://www.elastic.co/what-is/open-x-pack
[paid-features]: https://www.elastic.co/subscriptions
[es-security]: https://www.elastic.co/guide/en/elasticsearch/reference/current/security-settings.html
[trial-license]: https://www.elastic.co/guide/en/elasticsearch/reference/current/license-settings.html
[license-mngmt]: https://www.elastic.co/guide/en/kibana/current/managing-licenses.html
[license-apis]: https://www.elastic.co/guide/en/elasticsearch/reference/current/licensing-apis.html
[exit-code-137]: https://stackoverflow.com/questions/62006956/elasticsearch-multi-node-cluster-one-node-always-fails-with-docker-compose
[elastdocker]: https://github.com/sherifabdlnaby/elastdocker

[docker-install]: https://docs.docker.com/get-docker/
[compose-install]: https://docs.docker.com/compose/install/
[compose-v2]: https://docs.docker.com/compose/cli-command/
[linux-postinstall]: https://docs.docker.com/engine/install/linux-postinstall/

[bootstrap-checks]: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
[es-sys-config]: https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html
[es-heap]: https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#heap-size-settings

[win-filesharing]: https://docs.docker.com/desktop/windows/#file-sharing
[mac-filesharing]: https://docs.docker.com/desktop/mac/#file-sharing

[builtin-users]: https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
[ls-monitoring]: https://www.elastic.co/guide/en/logstash/current/monitoring-with-metricbeat.html
[sec-cluster]: https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html

[connect-kibana]: https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html
[index-pattern]: https://www.elastic.co/guide/en/kibana/current/index-patterns.html

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[es-docker]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[kbn-docker]: https://www.elastic.co/guide/en/kibana/current/docker.html
[ls-docker]: https://www.elastic.co/guide/en/logstash/current/docker-config.html

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html
