---
layout: default
parent: Training
title: Advanced Kubernetes
nav_order: 6
---
# Advanced Kubernetes

The following provides networking and volume details. 

## Networking

EdgeLake uses <a href="https://kubernetes.io/docs/concepts/services-networking/cluster-ip-allocation/" target="_blank">dynamic ClusterIP</a> 
as it's preferred setup. This means a unique IP address is automatically assigned to the services as they are created 
and ensures load balancing across the pods in the service.

### Configuring the network services on the EdgeLake node

EdgeLake assumes static IPs as the IPs are registered in the EdgeLake Metadata and serve as a directory to locate the 
members of the EdgeLake Network.  
To assign static IPs, specify the host's internal IP as the `Virtual IP` value.

The following chart summarizes the setup:
<table>
  <thead>
    <tr>
      <th style="text-align:center;">Connection Type</th>
      <th style="text-align:center;">External IP</th>
      <th style="text-align:center;">Internal IP</th>
      <th style="text-align:center;">Config Command</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:center;">TCP</td>
      <td style="text-align:center;">External IP</td>
      <td style="text-align:center;">Virtual IP</td>
      <td style="text-align:center;"><code class="language-anylog">run tcp server</code></td>
    </tr>
    <tr>
      <td style="text-align:center;">REST</td>
      <td style="text-align:center;">External IP</td>
      <td style="text-align:center;">Virtual IP</td>
      <td style="text-align:center;"><code class="language-anylog">run REST server</code></td>
    </tr>
    <tr>
      <td style="text-align:center;">Message Broker (TCP)</td>
      <td style="text-align:center;">External IP</td>
      <td style="text-align:center;">Virtual IP</td>
      <td style="text-align:center;"><code class="language-anylog">run message broker</code></td>
    </tr>
  </tbody>
</table>

Additional information on the network configuration are in the <a href="https://github.com/AnyLog-co/documentation/blob/master/network%20configuration.mdn" target="_blank">networking section</a>.

## Peer-to-peer Communication

Kubernetes has 3 base networking configurations:
<ul>
  <li><i>ClusterIP</i> -  Exposes the service on an internal IP within the cluster, making it accessible only within the cluster.</li>
  <li><i>NodePort</i> - Exposes the service on a static port on each node's IP, allowing external traffic to access the service.</li>
  <li><i>LoadBalancer</i> - Exposes the service externally using a cloud provider's load balancer, distributing traffic across multiple nodes.</li>
</ul>

The configuration files are set to use ClusterIP by default, expecting users to enable port-forwarding for ports that 
need to communicate with services outside the Kubernetes network. By default, the deploy_node.sh script enables 
port-forwarding for all the ports the Kubernetes instance will be using. 

**Note**: When using Kubernetes, makes sure ports are open and accessible across your network. 

## Volumes
The base deployment has the same general volumes as a docker deployment, and uses <a href="https://kubernetes.io/docs/concepts/storage/persistent-volumes/" target="_blank">PersistentVolumeClaim</a> - _data_, _blockchain_, _EdgeLake_ and _local-scripts (deployments)_.

While _data_, _blockchain_ and _anylog_ are autogenerated and populated, _local-scripts_ gets downloaded as part of the 
container image. Therefore, we utilize an `if/else` process to make this data persistent. 

**Note**: we copy the scripts to a persistent volume that is created after the initialization of the Pod.

<pre class="code-frame"><code class="language-shell">if [[ -d $EdgeLake_PATH/deployment-scripts ]] && [[ -z $(ls -A $EdgeLake_PATH/deployment-scripts) ]]; then # if directory exists but empty
  git clone -b os-dev https://github.com/EdgeLake-co/deployment-scripts deployment-scripts-tmp
  mv deployment-scripts-tmp/* deployment-scripts
  rm -rf deployment-scripts-tmp
elif [[ ! -d $EdgeLake_PATH/deployment-scripts ]] ; then  # if directory DNE
  git clone -b os-dev https://github.com/EdgeLake-co/deployment-scripts
fi</code></pre>

Once a node is up and running, users can change content in _local-scripts_ using <code class="language-shell">kubectl exec ${POD_NAME} -- /bin/bash</code>s.

Volumes are deployed automatically as part of deploy_node.sh, and remain persistent as long as PersistentVolumeClaims
are not removed. 

## Sample Node Policy for Kubernetes
When a node gets deployed, it either generates a new configuration policy or utilizes an existing one. 

Notes:
1. For EdgeLake Nodes, use static IPs, as these are stored in the shared metadata to serve as a directory to identify and locate nodes in the network.
2. The IP of the EdgeLake Master Node (if used) serves as an identifier of the network (and needs to be static).
3. The configuration policy below calls to scripts that are hosted on the local node. These scripts 
include native EdgeLake commands to declare and connect to database, declare policy, associating the EdgeLake instance with the network
and declaring local monitoring. 

<pre class="code-frame"><code class="language-json">{'config' : {'name' : 'operator-iotech-configs',
    'company' : 'AnyLog Co.',
    'node_type' : 'operator',
    'ip' : '!external_ip',
    'local_ip' : '!ip',
    'port' : '!anylog_server_port.int',
    'rest_port' : '!anylog_rest_port.int',
    'broker_port' : '!anylog_broker_port.int',
    'threads' : '!tcp_threads.int',
    'tcp_bind' : '!tcp_bind',
    'rest_threads' : '!rest_threads.int',
    'rest_timeout' : '!rest_timeout.int',
    'rest_bind' : '!rest_bind',
    'broker_threads' : '!broker_threads.int',
    'broker_bind' : '!broker_bind',
    'script' : [
        'process !local_scripts/database/deploy_database.al',
        'process !local_scripts/policies/cluster_policy.al',
        'process !local_scripts/policies/operator_policy.al',
        'run scheduler 1',
        'process !local_scripts/policies/config_threashold.al',
        'run streamer',
        'if !enable_ha == true then run data distributor',
        'if !enable_ha == true then run data consumer where start_date=!start_data',
        'if !operator_id then run operator where create_table=!create_table and update_tsd_info=!update_tsd_info and compress_json=!compress_file and compress_sql=!compress_sql and archive_json=!archive and archive_sql=!archive_sql and master_node=!ledger_conn and policy=!operator_id and threads=!operator_threads',
        'schedule name=remove_archive and time=1 day and task delete archive where days = !archive_delete',
        'if !monitor_nodes == true then process $ANYLOG_PATH/deployment-scripts/demo-scripts/monitoring_policy.al',
        'if !enable_mqtt == true then process $ANYLOG_PATH/deployment-scrpts/demo-scripts/basic_msg_client.al',
        'if !deploy_local_script == true then process !local_scripts/local_script.al'
    ],
}}</code></pre>