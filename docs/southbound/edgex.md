# EdgeX connection 

EdgeX allows to send data _REST_, _MQTT_ and _Kafka_ which requires `run msg client` on the EdgeLake 
side to accept data. 

## Requirements 
1. EdgeLake deployment to accept data - directions can be found [here](https://github.com/EdgeLake/docker-compose)
2. EdgeX & Corresponding plugins 
    * EdgeX 
    * EdgeX MQTT broker 
    * EdgeX Random Data Generator

## Accepting Data from EdgeX into EdgeLake
1. When starting EdgeLake, make sure to begin with message broker enable (`ANYLOG_BROKER_PORT` config value)

2. Begin inserting data - the sample code uses RandomInt and Modbus
<pre>
    <code>
[
    {
      "id": "fb68440c-0dea-49be-b2b2-8e9003ab78c2",
      "pushed": 1656093207769,
      "device": "Random-Integer-Generator01",
      "created": 1656093207759,
      "modified": 1656093207771,
      "origin": 1656093207757297700,
      "readings": [
        {
          "id": "95fa6063-9c6d-4a31-8237-9732c51ec3f7",
          "created": 1656093207759,
          "origin": 1656093207757240800,
          "device": "Random-Integer-Generator01",
          "name": "RandomValue_Int16",
          "value": "-12830",
          "valueType": "Int16"
        }
      ]
    },
    {
          "id": "f2acb1fb-f785-4cf4-8c80-8b97ab7f6056",
          "created": 1656092885317,
          "origin": 1656092885312948700,
          "device": "Modbus TCP test device",
          "name": "FanSpeed",
          "value": "Low",
          "valueType": "String"
        }
      ]
    }
]
    </code>
</pre>

3. Configure EdgeX `app-service-mqtt`, make sure to update the following params
    * MQTT_IP_ADDRESS
    * MQTT_PORT
    * MQTT_TOPIC
<pre>
    <code>
  app-service-mqtt:
       image: ${APP_SVC_REPOSITORY}/docker-app-service-configurable${ARCH}:${APP_SERVICE_VERSION}
       ports:
         - "127.0.0.1:48101:48101"
       container_name: edgex-app-service-configurable-mqtt
       env_file:
         - common.env
       hostname: edgex-app-service-configurable-mqtt
       networks:
         edgex-network:
           aliases:
             - edgex-app-service-configurable-mqtt
       depends_on:
         - consul
         - data
       read_only: true
       security_opt:
         - no-new-privileges:true
       environment:
         EDGEX_SECURITY_SECRET_STORE: "false"
         Registry_Host: edgex-core-consul
         Clients_CoreData_Host: edgex-core-data
         Clients_Data_Host: edgex-core-data # For device Services
         Clients_Notifications_Host: edgex-support-notifications
         Clients_Metadata_Host: edgex-core-metadata
         Clients_Command_Host: edgex-core-command
         Clients_Scheduler_Host: edgex-support-scheduler
         Clients_RulesEngine_Host: edgex-kuiper
         Databases_Primary_Host: edgex-redis
         # Required in case old configuration from previous release used.
         # Change to "true" if re-enabling logging service for remote logging
         Logging_EnableRemote: "false"
         #  Clients_Logging_Host: edgex-support-logging # un-comment if re-enabling logging service for remote logging
         edgex_profile: mqtt-export
         Service_Host: edgex-app-service-configurable-mqtt
         Service_Port: 48101
         MessageBus_SubscribeHost_Host: edgex-core-data
         Binding_PublishTopic: events
         # Added for MQTT export using app service
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_ADDRESSABLE_ADDRESS: ${MQTT_IP_ADDRESS}
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_ADDRESSABLE_PORT: ${MQTT_PORT}
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_ADDRESSABLE_PROTOCOL: tcp
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_ADDRESSABLE_TOPIC: ${MQTT_TOPIC}
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_PARAMETERS_AUTORECONNECT: "true"
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_PARAMETERS_RETAIN: "true"
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_PARAMETERS_PERSISTONERROR: "false"
#         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_ADDRESSABLE_PUBLISHER:
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_ADDRESSABLE_USER: ""
         WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_ADDRESSABLE_PASSWORD: ""
         # WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_PARAMETERS_QOS: ["your quality or service"]
         # WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_PARAMETERS_KEY: [your Key]  
         # WRITABLE_PIPELINE_FUNCTIONS_MQTTSEND_PARAMETERS_CERT: [your Certificate]
    </code>
</pre>

4. On EdgeLake, configure `run msg client` to accept data from EdgeX
<pre>
    <code>
&lt;run msg client where broker=local and port=!anylog_broker_port and log=false and topic=(
   name=anylogedgex-demo and 
   dbms=test and 
   table="bring [readings][][name]" and 
   column.edgex_id=(type=str and value="bring [readings][][id]") and 
   column.timestamp.timestamp=now and 
   column.device=(type=str and value="bring [readings][][device]") and
   column.value=(type=str and value="bring [readings][][value]")
)&gt;
    </code>
</pre>

EdgeLake deployment comes with a sample connection to EdgeX that accepts data IoTech System _lighting_ and _retail-device1_.