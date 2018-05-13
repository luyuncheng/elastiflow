# ElastiFlow&trade;
ElastiFlow&trade; provides network flow data collection and visualization using the Elastic Stack. It supports Netflow v5/v9, sFlow and IPFIX flow types (1.x versions support only Netflow v5/v9).
> Release 3.x is designed for use with the Elastic Stack 6.2 and higher. If you are using an older version of the Elastic Stack, please use version 2.1 or 1.2.

![ElastiFlow&trade;](https://user-images.githubusercontent.com/10326954/39966506-0934e198-56ad-11e8-9f40-c6454b6c6ea7.png)

I was inspired to create ElastiFlow&trade; following the overwhelmingly positive feedback received to an article I posted on Linkedin... [WTFlow?! Are you really still paying for commercial solutions to collect and analyze network flow data?](https://www.linkedin.com/pulse/wtflow-you-really-still-paying-commercial-solutions-collect-cowart)

# Getting Started
ElastiFlow&trade; is built using the Elastic Stack, including Elasticsearch, Logstash and Kibana. To install and configure ElastiFlow&trade;, you must first have a working Elastic Stack environment. The latest release of ElastiFlow&trade; requires version 6.2 or later.

Refer to the following compatibility chart to choose a release of ElastiFlow&trade; that is compatible with the version of the Elastic Stack you are using.

Elastic Stack | ElastiFlow&trade; 1.x | ElastiFlow&trade; 2.x | ElastiFlow&trade; 3.x
:---:|:---:|:---:|:---:
6.2 | &#10003; | &#10003; | &#10003;
6.1 | &#10003; | &#10003; |
6.0 | &#10003; | &#10003; |
5.6 | &#10003; | &#10003; |
5.5 | &#10003; |  |
5.4 | &#10003; |  |

> NOTE: The instructions that follow are for ElastiFlow&trade; 3.x.

## Requirements
Please be aware that in production environments the volume of data generated by many network flow sources can be considerable. It is not uncommon for a core router or firewall to produce 1000s of flow records per second. For this reason it is recommended that ElastiFlow&trade; be given its own dedicated Logstash instance. Multiple instances may be necessary as the volume of flow data increases.

> Due to the way NIC receive queues and the Linux kernel interact, raw UDP packet reception will be bound to a single CPU core and kernel receive buffer. While additional UDP input workers allow Logstash to share the load of processing packets from the receive buffer, it does not scale linearly. As worker threads are increased, so is contention for buffer access. The sweetspot seems to be to use 4-core Logstash instances, adding additional instances as needed for high-volume environments.

## Setting-up-Elasticsearch
Currently there is no specific configuration required for Elasticsearch. As long as Kibana and Logstash can talk to your Elasticsearch cluster you should be ready to go. The index template required by Elasticsearch will be uploaded by Logstash.

At high ingest rates (>10K flows/s), or for data redundancy and high availability, a multi-node cluster is recommended.

## Setting up Logstash
The ElastiFlow&trade; Logstash pipeline is the heart of the solution. It is here that the raw flow data is collected, decoded, parsed, formatted and enriched. It is this processing that makes possible the analytics options provided by the Kibana [dashboards](#dashboards).

Follow these steps to ensure that Logstash and ElastiFlow&trade; are optimally configured to meet your needs. 

### 1. Set JVM heap size.
To increase performance, ElastiFlow&trade; takes advantage of the caching and queueing features available in many of the Logstash plugins. These features increase the consumption of the JVM heap. The JVM heap space used by Logstash is configured in `jvm.options`. It is recommended that Logstash be given at least 1GB of JVM heap. If DNS lookups are enabled (requires version 3.0.10 or later of the DNS filter) increase this to 2GB. This is configured in `jvm.options` as follows:
```
-Xms2g
-Xmx2g
```

### 2. Add and Update Required Logstash plugins
To use ElastiFlow&trade; you will need to install the community supported [sFlow](https://github.com/ashangit/logstash-codec-sflow) codec for Logstash. It is also recommended that you always use the latest version of the [Netflow](https://www.elastic.co/guide/en/logstash/current/plugins-codecs-netflow.html) codec, the [UDP](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-udp.html) input, and the [DNS](https://www.elastic.co/guide/en/logstash/current/plugins-filters-dns.html) filter. This can achieved by running the following commands:

```
LS_HOME/bin/logstash-plugin install logstash-codec-sflow
LS_HOME/bin/logstash-plugin update logstash-codec-netflow
LS_HOME/bin/logstash-plugin update logstash-input-udp
LS_HOME/bin/logstash-plugin update logstash-filter-dns
```

### 3. Copy the pipeline files to the Logstash configuration path.
There are four sets of configuration files provided within the `logstash/elastiflow` folder:
```
logstash
  `- elastiflow
       |- conf.d  (contains the logstash pipeline)
       |- dictionaries (yaml files used to enrich raw flow data)
       |- geoipdbs  (contains GeoIP databases)
       `- templates  (contains index templates)
```

Copy the `elastiflow` directory to the location of your Logstash configuration files (e.g. on RedHat/CentOS or Ubuntu this would be `/etc/logstash/elastiflow` ). If you place the ElastiFlow&trade; pipeline within a different path, you will need to modify the following environment variables to specify the correct location:

Environment Variable | Description | Default Value
--- | --- | ---
ELASTIFLOW_DICT_PATH | The path where the dictionary files are located | /etc/logstash/dictionaries
ELASTIFLOW_TEMPLATE_PATH | The path to where index templates are located | /etc/logstash/templates
ELASTIFLOW_GEOIP_DB_PATH | The path where the GeoIP DBs are located | /etc/logstash/geoipdbs

### 4. Setup environment variable helper files
Rather than directly editing the pipeline configuration files for your environment, environment variables are used to provide a single location for most configuration options. These environment variables will be referred to in the remaining instructions. A [reference](#environment-variable-reference) of all environment variables can be found [here](#environment-variable-reference).

Depending on your environment there may be many ways to define environment variables. The files `profile.d/elastiflow.sh` and `logstash.service.d/elastiflow.conf` are provided to help you with this setup.

Recent versions of both RedHat/CentOS and Ubuntu use systemd to start background processes. When deploying ElastiFlow&trade; on a host where Logstash will be managed by systemd, copy `logstash.service.d/elastiflow.conf` to `/etc/systemd/system/logstash.service.d/elastiflow.conf`. Any configuration changes can then be made by editing this file.

> Remember that for your changes to take effect, you must issue the command `sudo systemctl daemon-reload`.

### 5. Add the ElastiFlow&trade; pipeline to pipelines.yml
Logstash 6.0 introduced the ability to run multiple pipelines from a single Logstash instance. The `pipelines.yml` file is where these pipelines are configured. While a single pipeline can be specified directly in `logstash.yml`, it is a good practice to use `pipelines.yml` for consistency across environments.

Edit `pipelines.yml` (usually located at `/etc/logstash/pipelines.yml`) and add the ElasiFlow&trade; pipeline (adjust the path as necessary).

```
- pipeline.id: elastiflow
  path.config: "/etc/logstash/elastiflow/conf.d/*.conf"
```

### 6. Configure inputs
By default flow data will be recieved on all IPv4 addresses of the Logstash host using the standard ports for each flow type. You can change both the IPs and ports used by modifying the following environment variables:

Environment Variable | Description | Default Value
--- | --- | ---
ELASTIFLOW_NETFLOW_IPV4_HOST | The IP address from which to listen for Netflow messages | 0.0.0.0
ELASTIFLOW_NETFLOW_IPV4_PORT | The UDP port on which to listen for Netflow messages | 2055
ELASTIFLOW_SFLOW_IPV4_HOST | The IP address from which to listen for sFlow messages | 0.0.0.0
ELASTIFLOW_SFLOW_IPV4_PORT | The UDP port on which to listen for sFlow messages | 6343
ELASTIFLOW_IPFIX_TCP_IPV4_HOST | The IP address from which to listen for IPFIX messages via TCP | 0.0.0.0
ELASTIFLOW_IPFIX_TCP_IPV4_PORT | The port on which to listen for IPFIX messages via TCP | 4739
ELASTIFLOW_IPFIX_UDP_IPV4_HOST | The IP address from which to listen for IPFIX messages via UDP | 0.0.0.0
ELASTIFLOW_IPFIX_UDP_IPV4_PORT | The port on which to listen for IPFIX messages via UDP | 4739

Collection of flows over IPv6 is disabled by default to avoid issues on systems without IPv6 enabled. To enable IPv6 rename the following files in the `elastiflow/conf.d` directory, removing `.disabled` from the end of the name: `10_input_ipfix_ipv6.logstash.conf.disabled`, `10_input_netflow_ipv6.logstash.conf.disabled`, `10_input_sflow_ipv6.logstash.conf.disabled`. Similiar to IPv4, IPv6 input can be configured using environment variables:

Environment Variable | Description | Default Value
--- | --- | ---
ELASTIFLOW_NETFLOW_IPV6_HOST | The IP address from which to listen for Netflow messages | [::]
ELASTIFLOW_NETFLOW_IPV6_PORT | The UDP port on which to listen for Netflow messages | 52055
ELASTIFLOW_SFLOW_IPV6_HOST | The IP address from which to listen for sFlow messages | [::]
ELASTIFLOW_SFLOW_IPV6_PORT | The UDP port on which to listen for sFlow messages | 56343
ELASTIFLOW_IPFIX_TCP_IPV6_HOST | The IP address from which to listen for IPFIX messages via TCP | [::]
ELASTIFLOW_IPFIX_TCP_IPV6_PORT | The port on which to listen for IPFIX messages via TCP | 54739
ELASTIFLOW_IPFIX_UDP_IPV6_HOST | The IP address from which to listen for IPFIX messages via UDP | [::]
ELASTIFLOW_IPFIX_UDP_IPV6_PORT | The port on which to listen for IPFIX messages via UDP | 54739

To improve UDP input performance for the typically high volume of flow collection, the default values for UDP input `workers` and `queue_size` is increased. The default values are `2` and `2000` respecitvely. ElastiFlow&trade; increases these to `4` and `4096`. Further tuning is possible using the following environment variables.

Environment Variable | Description | Default Value
--- | --- | ---
ELASTIFLOW_NETFLOW_UDP_WORKERS | The number of Netflow input threads | 4
ELASTIFLOW_NETFLOW_UDP_QUEUE_SIZE | The number of unprocessed Netflow UDP packets the input can buffer | 4096
ELASTIFLOW_SFLOW_UDP_WORKERS | The number of sFlow input threads | 4
ELASTIFLOW_SFLOW_UDP_QUEUE_SIZE | The number of unprocessed sFlow UDP packets the input can buffer | 4096
ELASTIFLOW_IPFIX_UDP_WORKERS | The number of IPFIX input threads | 4
ELASTIFLOW_IPFIX_UDP_QUEUE_SIZE | The number of unprocessed IPFIX UDP packets the input can buffer | 4096

> WARNING! Increasing `queue_size` will increase heap_usage. Make sure have configured JVM heap appropriately as specified in the [Requirementa](#requirements)

### 7. Configure Elasticsearch output
Obviously the data need to land in Elasticsearch, so you need to tell Logstash where to send it. This is done by setting these environment variables:

Environment Variable | Description | Default Value
--- | --- | ---
ELASTIFLOW_ES_HOST | The Elasticsearch host to which the output will send data | 127.0.0.1:9200
ELASTIFLOW_ES_USER | The password for the connection to Elasticsearch | elastic
ELASTIFLOW_ES_PASSWD | The username for the connection to Elasticsearch | changeme

> If you are only using the open-source version of Elasticsearch, it will ignore the username and password. In that case just leave the defaults.

### 8. Enable DNS name resolution (optional)
In the past it was recommended to avoid DNS queries as the latency costs of such lookups had a devastating effect on throughput. While the Logstash DNS filter provides a caching mechanism, its use was not recommended. When the cache was enabled all lookups were performed synchronously. If a name server failed to respond, all other queries were stuck waiting until the query timed out. The end result was even worse performance.

Fortunately these problems have been resolved. Release 3.0.8 of the DNS filter introduced an enhancement which caches timeouts as failures, in addition to normal NXDOMAIN responses. This was an important step as many domain owner intentionally setup their nameservers to ignore the reverse lookups needed to enrich flow data. In addition to this change, I submitted am enhancement which allows for concurrent queries when caching is enabled. The Logstash team approved this change, and it is included in 3.0.10 of the plugin.

With these changes I can finally give the green light for using DNS lookups to enrich the incoming flow data. You will see a little slow down in throughput until the cache warms up, but that usually lasts only a few minutes. Once the cache is warmed up, the overhead is minimal, and event rates averaging 10K/s and as high as 40K/s were observed in testing.

The key to good performance is setting up the cache appropriately. Most likely it will be DNS timeouts that are the source of most latency. So ensuring that a higher volume of such misses can be cached for longer periods of time is most important.

The DNS lookup features of ElastiFlow&trade; can be configured using the following environment variables:

Environment Variable | Description | Default Value
--- | --- | ---
ELASTIFLOW_RESOLVE_IP2HOST | Enable/Disable DNS requests | false
ELASTIFLOW_NAMESERVER | The DNS server to which the dns filter should send requests | 127.0.0.1
ELASTIFLOW_DNS_HIT_CACHE_SIZE | The cache size for successful DNS queries | 25000
ELASTIFLOW_DNS_HIT_CACHE_TTL | The time in seconds successful DNS queries are cached | 900
ELASTIFLOW_DNS_FAILED_CACHE_SIZE | The cache size for failed DNS queries | 75000
ELASTIFLOW_DNS_FAILED_CACHE_TTL | The time in seconds failed DNS queries are cached | 3600

### 9. Configure Application ID enrichment (optional)
Both Netflow and IPFIX allow devices with application identification features to specify the application associated with the traffic in the flow. For Netflow this is the `application_id` field. The IPFIX field is `applicationId`.

The application names which correspond to values of these IDs is vendor-specific. In order for ElastiFlow&trade; to accurately translate the ID values, it must be told the type of device that is exporting the flows. To do so you must edit `elastiflow/dictionaries/app_id_srctype` and specify the source type of your supported device. For example...
```
"192.0.2.1": "cisco_nbar2"
"192.0.2.2": "fortinet"
```
> Currently supported is Cisco's NBAR2 and Fortinet's FortiOS. If you have a device that you would like added, I will need a mapping of Application IDs to names. This can often be extracted from the device's configuration. I would love to be able to build up a large knowledge base of such mappings.

Once configured ElastiFlow&trade; will resolve the ID to an application name, which will be available in the dashboards.
<img width="902" alt="screen shot 2018-05-13 at 12 40 04" src="https://user-images.githubusercontent.com/10326954/39966360-d8e6e420-56aa-11e8-8514-9a9839ca5fb1.png"> 

### 10. Start Logstash
You should now be able to start Logstash and begin collecting network flow data. Assuming you are running a recent version of RedHat/CentOS or Ubuntu, and using systemd, complete these steps:
1. Run `systemctl daemon-reload` to ensure any changes to the environment variables are recognized.
2. Run `systemctl start logstash`
> NOTICE! Make sure that you have already setup the Logstash init files by running `LS_HOME/bin/system-install`. If the init files have not been setup you will receive an error.
To follow along as Logstash starts you can tail its log by running:
```
tail -f /var/log/logstash/logstash-plain.log
```
Logstash takes a little time to start... BE PATIENT!

If using Netflow v9 or IPFIX you will likely see warning messages related to the flow templates not yet being received. They will disappear after templates are received from the network devices, which should happen every few minutes. Some devices can take a bit longer to send templates. Fortinet in particular send templates rather infrequently.

Logstash is setup is now complete. If you are receiving flow data, you should have an `elastiflow-` daily index in Elasticsearch.

## Setting up Kibana
An API (yet undocumented) is available to import and export Index Patterns. The JSON file which contains the Index Pattern configuration is `kibana/elastiflow.index_pattern-json`. To setup the `elastiflow-*` Index Pattern run the following command:
```
curl -X POST -u USERNAME:PASSWORD http://KIBANASERVER:5601/api/saved_objects/index-pattern/elastiflow-* -H "Content-Type: application/json" -H "kbn-xsrf: true" -d @/PATH/TO/elastiflow.index_pattern.json
```

Finally the vizualizations and dashboards can be loaded into Kibana by importing the `elastiflow.dashboards.json` file from within the Kibana UI. This is done from the Management - > Saved Objects page.

## Dashboards
The following dashboards are provided.

> NOTE: The dashboards are optimized for a monitor resolution of 1920x1080.

### Overview
![Overview](https://user-images.githubusercontent.com/10326954/39966471-9b0a40dc-56ac-11e8-8962-78b928c7971f.png)

### Top-N
There are separate Top-N dashboards for Top Talkers, Services, Conversations and Applciations.
![Top-N](https://user-images.githubusercontent.com/10326954/39966477-b52ee92c-56ac-11e8-84eb-4688ddff7754.png)

### Sankey
There are separate Sankey dashboards for Client/Server, Source/Destination and Autonomous System perspectives. The sankey visualizations are built using the new Vega visualization plugin.
> NOTICE! While these visualizations work flawlessly on previous 6.2 versions, there are some anomalies on 6.2.4. For now consider these dashboards as **experimental**.
![Sankey](https://user-images.githubusercontent.com/10326954/39966483-c14a3aa4-56ac-11e8-9319-a56b2bf60d9f.png)

### Geo IP
There are separate Geo Loacation dashboards for Client/Server and Source/Destination perspectives.
![Geo IP](https://user-images.githubusercontent.com/10326954/39966487-cd06acf6-56ac-11e8-9da7-1bff5e822d8d.png)

### AS Traffic
Provides a view of traffic to and from Autonomous Systems (public IP ranges)
![AS Traffic](https://user-images.githubusercontent.com/10326954/39966490-d8d6032e-56ac-11e8-8784-b9903855d4f3.png)

### Exporters
![Flow Exporters](https://user-images.githubusercontent.com/10326954/39966495-e42c14f2-56ac-11e8-8c0e-b4275bfb32eb.png)

### Traffic Details
![Traffic Details](https://user-images.githubusercontent.com/10326954/39966499-ecfa036e-56ac-11e8-98fc-bde7cbbea787.png)

### Flow Records
![Flow Records](https://user-images.githubusercontent.com/10326954/39966504-fafe1446-56ac-11e8-96f3-0f01a01811ca.png)

## Environment Variable Reference
The supported environment variables are:

Environment Variable | Description | Default Value
--- | --- | ---
ELASTIFLOW_DICT_PATH | The path where the dictionary files are located | /etc/logstash/dictionaries
ELASTIFLOW_TEMPLATE_PATH | The path to where index templates are located | /etc/logstash/templates
ELASTIFLOW_GEOIP_DB_PATH | The path where the GeoIP DBs are located | /etc/logstash/geoipdbs
ELASTIFLOW_GEOIP_CACHE_SIZE | The size of the GeoIP query cache | 8192
ELASTIFLOW_GEOIP_LOOKUP | Enable/Disable GeoIP lookups | true
ELASTIFLOW_ASN_LOOKUP | Enable/Disable ASN lookups | true
ELASTIFLOW_KEEP_ORIG_DATA | If set to `false` the original `netflow`, `ipfix` and `sflow` objects will be deleted prior to indexing. This can save disk space without affecting the provided dashboards. However the original flow fields will no longer be available if they are desired for additional analytics. | true
ELASTIFLOW_RESOLVE_IP2HOST | Enable/Disable DNS requests | false
ELASTIFLOW_NAMESERVER | The DNS server to which the dns filter should send requests | 127.0.0.1
ELASTIFLOW_DNS_HIT_CACHE_SIZE | The cache size for successful DNS queries | 25000
ELASTIFLOW_DNS_HIT_CACHE_TTL | The time in seconds successful DNS queries are cached | 900
ELASTIFLOW_DNS_FAILED_CACHE_SIZE | The cache size for failed DNS queries | 75000
ELASTIFLOW_DNS_FAILED_CACHE_TTL | The time in seconds failed DNS queries are cached | 3600
ELASTIFLOW_ES_HOST | The Elasticsearch host to which the output will send data | 127.0.0.1:9200
ELASTIFLOW_ES_USER | The password for the connection to Elasticsearch | elastic
ELASTIFLOW_ES_PASSWD | The username for the connection to Elasticsearch | changeme
ELASTIFLOW_NETFLOW_IPV4_HOST | The IP address from which to listen for Netflow messages | 0.0.0.0
ELASTIFLOW_NETFLOW_IPV4_PORT | The UDP port on which to listen for Netflow messages | 2055
ELASTIFLOW_NETFLOW_IPV6_HOST | The IP address from which to listen for Netflow messages | [::]
ELASTIFLOW_NETFLOW_IPV6_PORT | The UDP port on which to listen for Netflow messages | 52055
ELASTIFLOW_NETFLOW_UDP_WORKERS | The number of Netflow input threads | 4
ELASTIFLOW_NETFLOW_UDP_QUEUE_SIZE | The number of unprocessed Netflow UDP packets the input can buffer | 4096
ELASTIFLOW_NETFLOW_LASTSW_TIMESTAMP | Enable/Disable setting `@timestamp` with the value of netflow.last_switched | false
ELASTIFLOW_NETFLOW_TZ | The timezone of netflow.last_switched | UTC
ELASTIFLOW_SFLOW_IPV4_HOST | The IP address from which to listen for sFlow messages | 0.0.0.0
ELASTIFLOW_SFLOW_IPV4_PORT | The UDP port on which to listen for sFlow messages | 6343
ELASTIFLOW_SFLOW_IPV6_HOST | The IP address from which to listen for sFlow messages | [::]
ELASTIFLOW_SFLOW_IPV6_PORT | The UDP port on which to listen for sFlow messages | 56343
ELASTIFLOW_SFLOW_UDP_WORKERS | The number of sFlow input threads | 4
ELASTIFLOW_SFLOW_UDP_QUEUE_SIZE | The number of unprocessed sFlow UDP packets the input can buffer | 4096
ELASTIFLOW_IPFIX_TCP_IPV4_HOST | The IP address from which to listen for IPFIX messages via TCP | 0.0.0.0
ELASTIFLOW_IPFIX_TCP_IPV4_PORT | The port on which to listen for IPFIX messages via TCP | 4739
ELASTIFLOW_IPFIX_UDP_IPV4_HOST | The IP address from which to listen for IPFIX messages via UDP | 0.0.0.0
ELASTIFLOW_IPFIX_UDP_IPV4_PORT | The port on which to listen for IPFIX messages via UDP | 4739
ELASTIFLOW_IPFIX_TCP_IPV6_HOST | The IP address from which to listen for IPFIX messages via TCP | [::]
ELASTIFLOW_IPFIX_TCP_IPV6_PORT | The port on which to listen for IPFIX messages via TCP | 54739
ELASTIFLOW_IPFIX_UDP_IPV6_HOST | The IP address from which to listen for IPFIX messages via UDP | [::]
ELASTIFLOW_IPFIX_UDP_IPV6_PORT | The port on which to listen for IPFIX messages via UDP | 54739
ELASTIFLOW_IPFIX_UDP_WORKERS | The number of IPFIX input threads | 4
ELASTIFLOW_IPFIX_UDP_QUEUE_SIZE | The number of unprocessed IPFIX UDP packets the input can buffer | 4096

# Attribution
This product includes GeoLite2 data created by MaxMind, available from (http://www.maxmind.com)
