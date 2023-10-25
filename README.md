# Takebishi "Device Gateway" on Red Hat Device Edge

This demo introduces how to deploy Takebishi Device Gateway and RHAMQ to Red Hat Device Edge.

# Background
In Japan, there are many factories built about more than 30 years ago.
Therefore, there are still many legacy manufacturing equipments.
Also, in many cases, that equipment is not from a specific single vendor, but is multi-vendor.
For companies promoting smart factories, it is easy to start collecting data if the factory is new, while for older equipment, it is necessary to collect data using vendor-specific protocols.

Takebishi Device Gateway is a gateway solution that aggregates vendor-specific protocols into a single gateway function and enables data collection.

By combining Device Gateway with Red Hat open source technology, it is possible to flexibly implement data integration applications that target data from legacy PLCs as well.

The simplest configuration for an IoT Gateway pattern using Device Gateway is as follows: 

![Simple Architecture](./doc/architecture.png)

You can consider an edge box solution where PLC data collected by Device Gateway is sent to MQTT Broker (Red Hat AMQ) and applications such as inference and visualization, etc can be flexibly added onto the edge box via MQTT Broker.

# Environment
- HW: Intel NUC NUC12WSHi7
- OS: Red Hat Enterprise Linux 9.2

# Getting Started
Technical aspects in this demo are very simple.
In collaboration with Takebishi, you can run Takebishi's solution, Device Gateway, which can easily connect to old PLCs and other equipment, as a container on MicroShift.

## Create a namespace

```
oc create -f manifest/namespace.yaml
```

## Deploy Device Gateway

```bash
oc apply -f manifest/dgw
```

Check the pod status:

```bash
oc get po -n dgw
```

```
NAME                   READY   STATUS    RESTARTS   AGE
dgw-7cc9645bd5-zmpb7   1/1     Running   0          7m52s
```

> Note. Takebishi Device Gateway container images are provided as Red Hat certified container :
> https://catalog.redhat.com/software/container-stacks/detail/64b78a22c9f380494e648f67

You can access Device Gateway's web console via Route:

```bash
oc get route -n dgw -ojsonpath='{.items[*].status.ingress[*].host}'
```

```
dgw-dgw.apps.edge-jp.demo.local
```

![Device Gateway Console](./images/console.png)  

Default account is as follows:

```
username: administrator
password: admin
```

### Deploy Red Hat AMQ

The data collected by Device gateway can then be linked to IT applications via Red Hat AMQ, or more specifically, ActiveMQ.

Deploy AMQ Broker Operator:

```bash
oc apply -f manifest/amq
```

Check the pod status:

```bash
oc get po,svc -n dgw
```

```
NAME              READY   STATUS    RESTARTS   AGE
pod/broker-ss-0   1/1     Running   0          11m

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)              AGE
service/broker-all-0-svc   ClusterIP   172.30.133.141   <none>        61616/TCP            11m
service/broker-hdls-svc    ClusterIP   None             <none>        8161/TCP,61616/TCP   19h
service/broker-ping-svc    ClusterIP   None             <none>        8888/TCP             19h
```

# Configuration of Device Gateway
All Device Gateway settings are done through the web console. Here, as a simple test, we will add a setting to send Device Gateway status to MQTT broker in 1 second cycles.

## IoT Interface Settings
**1) Open the "IoT Interface" tab and click the "+" button in the "MQTT" section.**

![IoT Interface menu](./doc/IoT-Interface-menu.png)

**2) In the Host column, specify the MQTT endpoint of Red Hat AMQ that you have just deployed:**

![IoT Interface Setting](./doc/IoT-Interface-Setting.png)

Click the check mark in the upper right corner of the screen to update the settings, then press the "Sava" button in the menu at the top of the screen.

## Event Settings

Event settings are usually some action on a data source, such as a PLC. In this example, you configure Red Hat AMQ to send data in JSON format as follows:

**1) Open the "Event" tab and click the "+" button in the "Event" section.**
![Event menu](./doc/Event-menu.png)

**2)Select "Cycle" in the Trigger menu**
![Event trigger menu](./doc/Event-trigger.png)

**3)Select "Send MQTT" in the Action menu.**
![Event action menu](./doc/Event-action.png)

**4)Configuration of sending to MQTT Topic**

![Event action setting detailed](./doc/Event-mqtt-setting.png)

The format of the Message to be sent is, for example, as follows:

```json
{
   "time":  <%=@nowUtc("yyyyMMdd HH:mm:ss.ms")%>,
   "status": <%=@p("Managemnt.Information.Summary.ledStatus")%>,
   "send": <%=@p("Managemnt.Lan.Summary.sendBytes0")%>,
   "recieve": <%=@p("Managemnt.Lan.Summary.receiveBytes0")%>
}
```

Click the check mark in the upper right corner of the screen to update the settings, then press the "Sava" button in the menu at the top of the screen.

## Check the data sent to MQTT topic
Install mosquitto's CLI in an environment that can connect to OpenShift, and execute the following command, and you can check the status of data sent to MQTT Topic.

```bash
$ mosquitto_sub -h YOUR_DEVICE_IP -p 32000 -t test
```

```JSON
{
   "time":  20230718 01:59:37.606,
   "status": Green Right (No Error),
   "send": 2897082,
   "recieve": 928442
}
```