# CEPH Monitoring with ELK

This project is using the <b>ELK (ElasticSearch, Logstash, Kibana)' stack and Grafana</b> to monitor a <b>CEPH cluster (mimic: 13.2.0)</b> where it has deployed 4 OSD hosts. The metrics relevant to physical hosts such as cpu load, memory usage, interface traffic and each osd sata and ssd disks are obtained with the collectd tool using built-in plugin installed on each host. Other metrics relevant to CEPH such as osd disk capacity, read and write latency, pool statistics are obtained with collectd tool based on the ceph-specific plugin developed in other project which is published <a href="https://github.com/rochaporto/collectd-ceph">here</a>. Since the version of CEPH is mimic, the plugins were updated accordingly in this prooject.

## Monitoring metrics

* CEPH Overview
    * # of Monitors, OSD up/down, OSD in/out
    * Cluster IOPS/Throughput
    * RADOS benchmark latency for osd and hdd pools
    * OSD disk apply/commit latency 
    * OSD disk usage/capacity
* OSD Disk Performance
    * OSD disk read/write IOPS/Throughput  
    * OSD SSD read/write IOPS/Throughput  
* OSD Hosts Stats (for each host)
    *  Multiple interface (40G and 10G) rx/tx traffic in terms of packets and octets
    *  Memory usage
    *  Load
*  Pool Statistics (for each pool)
    *  Used space, # of stored Objects, pg/pgp_num
    *  IOPS/Throughput per pool

## Collectd

It was assumed that the collectd tool has already installed on the osd hosts. The tool needs to collect two different part of metrics which are specific for physical hosts and CEPH. Each one should be considered differently and configured accordingly.

### Collectd for Physical Hosts

collectd.conf is a default configuration file for getting the basic measurements about the physical hosts so it has to be applied for all hosts separately. The time interval between the samples, host name and the list of plugins used are set in this file. The name of plugins listed below are enough in our case. 

* disk
* interface
* load
* memory
* network

The collected metrics have been published into <b>LogStash</b> which is running on 25826 port so that the network plugin might be configured as following.

```
<Plugin network>
	<Server "172.16.36.4" "25826">
	</Server>
</Plugin>
```

### Collectd for CEPH

The files under collectd.conf.d folder have been prepared for getting metrics about CEPH usage and performance. The interface traffic as well as disk performance is not enough alone to assess the health of CEPH. Furthermore, there might be many pools where each one might be mapped with different CRUSH map (write HDD or SSD device). The list of configuration files are given below.

* ceph_hdd_latency.conf  
* ceph_monitor.conf  
* ceph_osd.conf  
* ceph_pg.conf  
* ceph_pool.conf  
* ceph_ssd_latency.conf

```
We do not require to configure all CEPH hosts to gather metrics. Instead, a single OSD host which is eligible to run ceph management commands 
enough to get and publish all ceph relevant metrics.
```

The location of the python plugins,  time interval (60 seconds) and name of the cluster (ceph) was set in this configuration files. 

The python files (plugins) are installed under the /usr/lib/collectd/plugins/ceph path. It basically runs ceph cli commands such as ceph mon dump, ceph pg dump and obtains the results in json format. The plugins have been updated the latest version of CEPH (Mimic for now) because there are slightly changes between the releases. 

* If you have multiple pools based on HDD and SSD like our examples, we might prepare two configuration files and plugins and updated the name of metrics properly for each one. For example, hdd and ssd prefix has been added for the latency plugins (hdd_avg_latency, etc). The name of the metrics are important since they will be indexed in ElasticSearch  and Graphane will visualize accordingly.

## Logstash

It is assumed that Logstash, ElasticSearch and Kibana have been installed and worked correctly. The ELK stack has been deployed with docker cluster which is out of this scope. 

LogStash is running on 172.16.36.4 and listening the 25826 port so that its configuration file requires to be set as following. The metrics coming from collectd were tagged with collectdceph to discriminate metrics coming from netflow and ceilometer openstack.

```
input {
  udp {
    port => 25826
    buffer_size => 1452
    codec => collectd { }
    tags => "collectdceph"
    type => collectdceph
  }

}

output {

  if "collectdceph" in [tags] {
    elasticsearch {
                  index => "collectdceph-%{+YYYY.MM.dd}"
                  hosts =>  ["172.16.36.2:9200"]
    }
  }

}

```
Kibana is very useful tool to assess the name of measurements under different plugin and run the specific query. I will strongly recommend to install Kibana and debug the flow and query using it.

![image] (https://gitlab.com/eakkoyun/ceph-monitoring-with-elk/raw/master/grafana/screenshots/kibana-overview.png)

### Grafana

The source data for Grafana is ElasticSearch. The metrics have been collected using Logstash and indexed by collectdceph prefix that we need to set our source in Kibana. Furthermore, the name of the OSD hosts are zula prefix. We have defined the variables comprises of the host names. Grafana allows us that we can picked a specific host and visualize just its metrics on the dashboard.

The json files of dashboard are listed below.

* CEPH_OVERVIEW.json  
* OSD_Disk_Performance.json  
* OSD_HOSTS_STATS.json  
* Pool_Statistics.json

