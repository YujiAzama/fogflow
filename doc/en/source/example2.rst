*****************************************
Define and trigger a service topology
*****************************************

In FogFlow a service topology is defined as a graph of several operators. 
Each operator in the service topology is annotated with its inputs and outputs, 
which indicate their dependency to the other tasks in the same topology. 
**Different from fog functions, a service topology is triggerred on demand by a customized "requirement" object.**


With a simple example we explain how developers can define and test a service topology in the following section. 


Use case on anomaly detection
---------------------------------------

This use case study is for retail stores to detect abnormal energy consumption in real-time.
As illustrated in the following picture, a retail company has a large number of shops distributed in different locations. 
For each shop, a Raspberry Pi device (edge node) is deployed to monitor the power consumption from all PowerPanels 
in the shop. Once an abnormal power usage is detected on the edge, 
the alarm mechanism in the shop is triggered to inform the shop owner. 
Moreover, the detected event is reported to the cloud for information aggregation. 
The aggregated information is then presented to the system operator via a dashboard service. 
In addition, the system operator can dynamically update the rule for anomaly detection.


.. figure:: figures/retails.png
   :width: 100 %


* Anomaly Detector: this operator is to detect anomaly events based on the collected data from power panels in a retail store. It has two types of inputs:    

	* detection rules, which are provided and updated by the operator; The detection rules input stream type is associated with ``broadcast``, meaning that the rules are needed by all task instances of this operator. The granularity of this operator is based on ``shopID``, meaning that a dedicated task instance will be created and configured for each shop
   	* sensor data from power panel

* Counter: this operator is to count the total number of anomaly events for all shops in each city. 
	Therefore, its task granularity is by ``city``. Its input stream type is the output stream type of the previous operator (Anomaly Detector). 

There are two types of result consumers: (1) a dashboard service in the cloud, which subscribes to the final aggregation results generated by the counter operator for the global scope; (2) the alarm in each shop, which subscribes to the anomaly events generated by the Anomaly Detector task on the local edge node in the retail store. 

.. figure:: figures/retail-flow.png
   :width: 70 %

Implement your operator functions required in your service topology
-----------------------------------------------------------------------

Before you can define the designed service topology, 
all operators used in your service topology must be provided by you or the other provider in the FogFlow system.
For  this specific use case, we need to implement two operators: anomaly_detector and counter. 
Please refer to the examples provided in our code repository. 

* `anomaly_detector`_ 

* `counter`_ 


.. _`anomaly_detector`: https://github.com/smartfog/fogflow/tree/master/application/operator/anomaly
.. _`counter`: https://github.com/smartfog/fogflow/tree/master/application/operator/counter


Register your task operators
--------------------------------------------------------

For each operator, once we create its docker image and push it to the FogFlow docker registry, 
we must register the operator in FogFlow. 
This can be done in one of the following two ways. 


.. note:: Please notice that each operator must have a unique name but the same operator can be associated with multiple docker images, 
            each of which is for one specific hardware or operating system but for implementing the same data processing logic. 
            During the runtime, FogFlow will select a proper docker image to run a scheduled task on an edge node, 
            based on the execution environment of the edge node. 


Register it via FogFlow Task Designer
==========================================================

The following picture shows the list of all registered operator docker images and the key information of each image. 

.. figure:: figures/operator-registry-list.png
   :scale: 100 %
   :alt: map to buried treasure


After clicking the "register" button, you can see a form as below. 
Please fill out the required information and click the "register" button to finish the registration. 
The form is explained as the following. 


* Image: the name of your operator docker image
* Tag: the tag you used to publish your operator docker image; by default it is "latest"
* Hardware Type: the hardware type that your docker image supports, including X86 or ARM (e.g. Raspberry Pi)
* OS Type: the operating system type that your docker image supports; currently this is only limited to Linux
* Operator: the operator name, which must be unique and will be used when defining a service topology
* Prefetched: if this is checked, that means all edge nodes will start to fetch this docker image in advance; otherwise, the operator docker image is fetched on demand, only when edge nodes need to run a scheduled task associated with this operator. 

.. important::
    
    Please notice that the name of your docker image must be consistent with the one you publish to `Docker Hub`_.
    By default, FogFlow will fetch the required docker images from Docker Hub using the name you register here for your operator. 


.. _`Docker Hub`: https://github.com/smartfog/fogflow/tree/master/application/operator/anomaly

.. figure:: figures/operator-register.png
   :scale: 100 %
   :alt: map to buried treasure


Register it programmatically by sending a NGSI update 
==========================================================

You can also register an operator docker image by sending a constructed NGSI update message to the IoT Broker deployed in the cloud. 

Here are the Curl and the Javascript-based code examples to register an operator docker image. 

.. note:: In the Javascript code example, we use the Javascript-based library to interact with FogFlow IoT Broker. You can find out the library from the github code repository (designer/public/lib/ngsi). You must include ngsiclient.js into your web page. 

.. note:: The Curl case assumes that the cloud IoT Broker is running on localhost on port 8070.

.. tabs::

   .. group-tab:: curl

        .. code-block:: console 

		curl -iX POST \
		  'http://localhost:8070/ngsi10/updateContext' \
	  	-H 'Content-Type: application/json' \
	  	-d '		
	     	{
			"contextElements": [
	            		{
	                	"entityId": {
	                    		"id": "counter:latest",
	                    		"type": "DockerImage",
	                    		"isPattern": false
	                	},
	                	"attributes": [
	               		{
	                  		"name": "image",
	                  		"type": "string",
	                  		"contextValue": "counter"
				},
	               		{
	                  		"name": "tag",
	                  		"type": "string",
	                  		"contextValue": "latest"
				},
	               		{
	                  		"name": "hwType",
	                  		"type": "string",
	                  		"contextValue": "X86"
				},
	               		{
	                  		"name": "osType",
	                  		"type": "string",
	                  		"contextValue": "Linux"
				},
	               		{
	                  		"name": "operatorName",
	                  		"type": "string",
	                  		"contextValue": "counter"
				},
	               		{
	                  		"name": "prefetched",
	                  		"type": "string",
	                  		"contextValue": false
				}],		
	                	"domainMetadata": [
	                	{
	                    		"name": "operator",
	                    		"type": "string",
			                "value": "counter"
	                	}
	                	]
	            	} ],
	        	"updateAction": "UPDATE"
		}			

   .. code-tab:: javascript

    	var image = {
        	name: "counter",
	        tag: "latest",
	        hwType: "X86",
	        osType: "Linux",
	        operatorName: "counter",
	        prefetched: false
    	};

    	//register a new docker image
    	var newImageObject = {};
	
    	newImageObject.entityId = {
	        id : image.name + ':' + image.tag, 
	        type: 'DockerImage',
	        isPattern: false
    	};
	
    	newImageObject.attributes = {};   
    	newImageObject.attributes.image = {type: 'string', value: image.name};        
    	newImageObject.attributes.tag = {type: 'string', value: image.tag};    
    	newImageObject.attributes.hwType = {type: 'string', value: image.hwType};      
    	newImageObject.attributes.osType = {type: 'string', value: image.osType};          
    	newImageObject.attributes.operator = {type: 'string', value: image.operatorName};      
    	newImageObject.attributes.prefetched = {type: 'boolean', value: image.prefetched};                      
	    
    	newImageObject.metadata = {};    
    	newImageObject.metadata.operator = {
	        type: 'string',
	        value: image.operatorName
    	};               
    
	    // assume the config.brokerURL is the IP of cloud IoT Broker
    	var client = new NGSI10Client(config.brokerURL);    
    	client.updateContext(newImageObject).then( function(data) {
	        console.log(data);
    	}).catch( function(error) {
	        console.log('failed to register the new device object');
    	});        
	


Specify a service topology
-----------------------------------
Assume that the tasks to be used in your service topology have been implemented and registered,
you can have two ways to specify your service topology. 


using FogFlow Topology Editor
=======================================

The first way is to use the FogFlow editor to specify a service topology.  

.. figure:: figures/retail-topology.png
   :width: 100 %

As seen in the picture, the following important information must be provided. 

#. define topology profile, including
    * topology name: the unique name of your topology
    * service description: some text to describe what this service is about
    * priority: define the priority level of all tasks in your topology, which will be utilized by edge nodes to decide how resource should be assigned to tasks 
    * resource usage: define if the tasks in this topology can use the resources on edge nodes in an exclusive way, meaning that not sharing resources with any task from the other topologies

#. draw the graph of data processing flows within the service topology
    With a right click at some place of the design board, you will see a memu pops up 
    and then you can start to choose either task or input streams 
    to define your data processing flows according to the design you had in mind. 

#. define Task Profile for each task in the data flow, including
    As shown in the following picture, you can start to specify the profile of each task in the data processing flow
    by clicking the configuration button. The following information is required to specify a task profile. 
	
    * name: the name of the task 
    * operator: the name of the operator that implements the data processing logic of this task; please register your operator beforehand so that it can be shown from the list
    * groupby: to determine how many instances of this task should be created on the fly; currently including the following cases
	
        *  if ther is only one instance to be created for this task, please use "groupby" = "all"
        *  if you need to create one instance for each entity ID of the input streams, please user "groupby" = "entityID"
        *  if you need to create one instance for each unique value of some specific context metadata, please use the name of this registered context metadata    
		
    * shuffling of input streams: to indicate how the input stream should be assigned to the instance(s) of this task during the runtime, including the following two cases
	
        *  "shuffling" = "broadcast": the selected input streams should be repeatedly assigned to every task instance of this operator
        *  "shuffling" = "unicast": each of the selected input streams should be assigned to a specific task instance only once        
		
    * entity type of output streams: to specify the entity type of the produced output stream


using NGSI Update to create it
=======================================


Another way is to register a service topology by sending a constructed NGSI update message to the IoT Broker deployed in the cloud. 

Here are the Curl and the Javascript-based code examples to register an operator docker image. 

.. note:: In the Javascript code example, we use the Javascript-based library to interact with FogFlow IoT Broker. You can find out the library from the github code repository (designer/public/lib/ngsi). You must include ngsiclient.js into your web page. 

.. note:: The Curl case assumes that the cloud IoT Broker is running on localhost on port 8070.

.. tabs::

   .. group-tab:: curl

        .. code-block:: console 

		curl -iX POST \
		  'http://localhost:8070/ngsi10/updateContext' \
	  	-H 'Content-Type: application/json' \
	  	-d '		
	     	{
			"contextElements": [
	            		{
	                	"entityId": {
	                    		"id": "Topology.anomaly-detection",
	                    		"type": "anomaly-detection",
	                    		"isPattern": false
	                	},
	                	"attributes": [
	               		{
	                  		"name": "status",
	                  		"type": "string",
	                  		"contextValue": "enabled"
				}, {
	                  		"name": "template",
	                  		"type": "object",
	                  		"contextValue": { 
				       		"description":"detect anomaly events from time series data points",
				       		"name":"anomaly-detection",
				       		"priority": {
				            		"exclusive": false,
	            			       		"trigger": "on-demand",   
       							"tasks":[  
				        			{  
             							"name":"AnomalyDetector",
             							"operator":"anomaly",
             							"groupBy":"shop",
             							"input_streams":[  
                							{  
                     								"type": "PowerPanel",
                    								"shuffling": "unicast",
                      								"scoped": true
                							},
                							{  
                      								"type": "Rule",
                    								"shuffling": "broadcast",
                      								"scoped": false               
                							}                       
             							],
             							"output_streams":[  
                							{  
                   								"type":"Anomaly"
                							}
	             						]
        			  				},
			          				{  
             							"name":"Counter",
             							"operator":"counter",
             							"groupBy":"*",
             							"input_streams":[  
                							{  
                   								"type":"Anomaly",
                   								"shuffling": "unicast",
                   								"scoped": true               
                							}           
             							],
             							"output_streams":[  
                							{  
                   								"type":"Stat"
                							}
             							]
          							}          
       							]
    						}
					}
				}]
			}]
		}
		

   .. code-tab:: javascript

	// the json object that represent the structure of your service topology
    	// when using the FogFlow topology editor, this is generated by the editor
	var topology = {  
       		"description":"detect anomaly events from time series data points",
       		"name":"anomaly-detection",
       		"priority": {
            		"exclusive": false,
            		"level": 100
       		},
       		"trigger": "on-demand",   
       		"tasks":[  
          			{  
             				"name":"AnomalyDetector",
             				"operator":"anomaly",
             				"groupBy":"shop",
             				"input_streams":[  
                				{  
                     					"type": "PowerPanel",
                    					"shuffling": "unicast",
                      					"scoped": true
                				},
                				{  
                      					"type": "Rule",
                    					"shuffling": "broadcast",
                      					"scoped": false               
                				}                       
             				],
             				"output_streams":[  
                				{  
                   					"type":"Anomaly"
                				}
	             				]
        			  },
			          {  
             				"name":"Counter",
             				"operator":"counter",
             				"groupBy":"*",
             				"input_streams":[  
                				{  
                   					"type":"Anomaly",
                   					"shuffling": "unicast",
                   					"scoped": true               
                				}           
             				],
             				"output_streams":[  
                				{  
                   					"type":"Stat"
                				}
             				]
          			}
       		]
    	}
    
	//submit it to FogFlow via NGSI Update
    	var topologyCtxObj = {};
    
    	topologyCtxObj.entityId = {
        	id : 'Topology.' + topology.name, 
        	type: topology.name,
        	isPattern: false
    	};
	    
    	topologyCtxObj.attributes = {};   
    	topologyCtxObj.attributes.status = {type: 'string', value: 'enabled'};
    	topologyCtxObj.attributes.template = {type: 'object', value: topology};    
    
    	// assume the config.brokerURL is the IP of cloud IoT Broker
    	var client = new NGSI10Client(config.brokerURL);    

    	// send NGSI10 update    
    	client.updateContext(topologyCtxObj).then( function(data) {
	        console.log(data);                
    	}).catch( function(error) {
	        console.log('failed to submit the topology');
    	});    
	

Trigger the service topology by sending a customized requirement
------------------------------------------------------------------------------

Once developers submit a specified service topology and the implemented operators, 
the service data processing logic can be triggered on demand by a high level processing requirement. 
The processing requirement is sent as NGSI10 update, with the following properties: 

* topology: which topology to trigger
* expected output: the output stream type expected by external subscribers
* scope: a defined geoscope for the area where input streams should be selected
* scheduler: which type of scheduling method should be chosen by Topology Master for task assignment

.. important::
    
    Please notice that the input data for the leaf nodes of your service topology must be already reported by your IoT devices. 
    It works like this in the current version, but this requirement will be no longer needed in the new version. 
    
    Also, the entity ID of your input data must follow a pattern: "Stream.[Type].[xxx]". 
    For example, for a temperature sensor with "Temperature" as the entity type, 
    the entity ID of this temperature entity requires to be "Stream.Temperature.[xxx]",
    for example, "Stream.Temperature.001". Please look at the example in the section |connect_device_to_fogflow|
    

.. |connect_device_to_fogflow| raw:: html

    <a href="./example3.html" target="_blank">Connect an IoT device to FogFlow</a>


Here are the Curl and the Javascript-based code examples to trigger a service topology by sending a customized requirement entity to FogFlow. 
 
.. note:: The Curl case assumes that the cloud IoT Broker is running on localhost on port 8070.

.. tabs::

   .. group-tab:: curl

        .. code-block:: console 
	
		curl -iX POST \
		  'http://localhost:8070/ngsi10/updateContext' \
	  	-H 'Content-Type: application/json' \
	  	-d '		
	     	{
			"contextElements": [
	            	{
	                	"entityId": {
	                    		"id": "Requirement.163ed933-828c-4c20-ab2a-59f73e8682cf",
	                    		"type": "Requirement",
	                    		"isPattern": false
	                	},
	                	"attributes": [
	                	{
	                  		"name": "output",
	                  		"type": "string",
	                  		"contextValue": "Stat"
	                	}, {
	                  		"name": "scheduler",
	                  		"type": "string",
	                  		"contextValue": "closest_first"
				}, {
	                  		"name": "restriction",
	                  		"type": "object",
	                  		"contextValue": { 
			  			"scopes": [{ 
				   			"scopeType": "circle", 
				   			"scopeValue": {
							   	"centerLatitude": 49.406393,
								"centerLongitude": 8.684208,
								"radius": 10.0
							   }
			        		}]
			  		}
				}
	                	],
	                	"domainMetadata": [
	                	{
	                    		"name": "topology",
	                    		"type": "string",
			                "value": "Topology.anomaly-detection"
	                	}
	                	]
	            	}
	        	],
	        	"updateAction": "UPDATE"
		     }	

   .. code-tab:: javascript

    	var rid = 'Requirement.' + uuid();    
   
   	var requirementCtxObj = {};    
    	requirementCtxObj.entityId = {
     		id : rid, 
        	type: 'Requirement',
        	isPattern: false
    	};
    
    	var restriction = { scopes:[{scopeType: geoscope.type, scopeValue: geoscope.value}]};
               
    	requirementCtxObj.attributes = {};   
    	requirementCtxObj.attributes.output = {type: 'string', value: 'Stat'};
    	requirementCtxObj.attributes.scheduler = {type: 'string', value: 'closest_first'};    
    	requirementCtxObj.attributes.restriction = {type: 'object', value: restriction};    
                       
    	requirementCtxObj.metadata = {};               
    	requirementCtxObj.metadata.topology = {type: 'string', value: curTopology.entityId.id};
		    
    	console.log(requirementCtxObj);
	            
    	// assume the config.brokerURL is the IP of cloud IoT Broker
    	var client = new NGSI10Client(config.brokerURL);                
    	client.updateContext(requirementCtxObj).then( function(data) {
        	console.log(data);
    	}).catch( function(error) {
        	console.log('failed to send a requirement');
    	});    


