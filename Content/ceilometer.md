###Ceilometer Metering Service
OpenStack Ceilometer services provides metering data related to OpenStack services. It collects events and metering data by monitoring notifications from the other service services and publishes collected data to various targets including data stores and message queues.

Ceilometer main components are:

1.  The **ceilometer-compute**, an agent running on each compute node polling the local libvirt daemon to acquire performance data for the local instances, messages and emits these data via RabbitMQ.
2.  The **ceilometer-central**, an agent running on the other nodes, polling the other openstack services for statistics not related to the compute resources.
3.  The **ceilometer-notification**, a service running on the controller, responsible for consuming messages from the RabbitMQ and building events and metering data.
4.  The **ceilometer-collector**, a service running on the controller, dispatching collected telemetry data to the metering data store or other external message queues.
5.  The **ceilometer-alarm-evaluator**, running on the controller, responsible for evaluating when fire an alarm due to the associated statistic trend crossing a threshold.
6.  The **ceilometer-alarm-notifier**, running on the controller, responsible for notify alarms and trigger actions.
7.  The **ceilometer-api**, running on the controller providing access to the users.

Only the collector and the API services have access to the data store. Ceilometer service uses a NoSQL database as datastore.

####Implementing Ceilometer
On the Controller node, setup the Ceilometer service as described in OpenStack official documentation, [here](http://docs.openstack.org/liberty/install-guide-rdo/ceilometer-install.html).

To keep things simple, we are going to install only the compute agent on any of the Compute node. This agent will be able to acquire performance and metering data only from the virtual machines running on that compute node. In this sectionwe are not going to monitor other resources as Images, Storage and Networking. On each Compute node we want metwer, install the component as reported [here](http://docs.openstack.org/liberty/install-guide-rdo/ceilometer-nova.html).

####Working with the metering service
In this section we are going to work with few basic concepts of the metering service:

* [Meters](./ceilometer.md#meters)
* [Samples](./ceilometer.md#samples)
* [Statistics](./ceilometer.md#statistics)
* [Alarms](./ceilometer.md#alarms)
* [Pipelines](./ceilometer.md#pipelines)

##### Meters
A meters measures a particular aspect of resource usage as for example, the current CPU utilization % for an instance. The lifecycle of meters is decoupled from the existence of the related resources, in the sense that the meter continues to exist after the resource has been terminated. All meters have a string name, an unit of measurement, and a type indicating whether values are monotonically increasing (i.e. *cumulative*), interpreted as a change from the previous value (i.e. *delta*), or a standalone value relating only to the current duration (i.e. *gauge*).

The following returns all meters available on the system

    # ceilometer meter-list
    +---------------------------------+------------+-----------+------------------------------------------
    | Name                            | Type       | Unit      | Resource ID
    +---------------------------------+------------+-----------+------------------------------------------
    | cpu                             | cumulative | ns        | 9d29e3bd-caa1-4c04-b570-9dec555c8adc
    | cpu.delta                       | delta      | ns        | 9d29e3bd-caa1-4c04-b570-9dec555c8adc
    | cpu_util                        | gauge      | %         | 9d29e3bd-caa1-4c04-b570-9dec555c8adc
    | disk.allocation                 | gauge      | B         | 9d29e3bd-caa1-4c04-b570-9dec555c8adc
    | disk.capacity                   | gauge      | B         | 9d29e3bd-caa1-4c04-b570-9dec555c8adc 
    ...

To query for a specific resource or project or user

    # ceilometer meter-list --query resource=<resource_id>
    # ceilometer meter-list --query project=<project_id>
    # ceilometer meter-list --query user=<user_id>

##### Samples
A sample is the individual datapoints associated with a particular meter. As such, all samples encompass the same attributes as the meter itself, but with the addition of a timestamp and and a value, also called the sample volume.

Here to show last 10 samples associated with a given meter

    # ceilometer sample-list --meter cpu_util --limit 10
    +--------------------------------------+----------+-------+---------------+------+----------------------------+
    | Resource ID                          | Name     | Type  | Volume        | Unit | Timestamp                  |
    +--------------------------------------+----------+-------+---------------+------+----------------------------+
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.5923854989 | %    | 2016-08-23T17:57:27.681000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6333676252 | %    | 2016-08-23T17:56:27.922000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6422183317 | %    | 2016-08-23T17:55:27.687000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6170192908 | %    | 2016-08-23T17:54:27.682000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.5867382656 | %    | 2016-08-23T17:53:27.683000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6299362877 | %    | 2016-08-23T17:52:27.920000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6039615059 | %    | 2016-08-23T17:51:27.822000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6117520497 | %    | 2016-08-23T17:50:27.674000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.6477669865 | %    | 2016-08-23T17:49:27.671000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu_util | gauge | 93.9214347448 | %    | 2016-08-23T17:48:27.659000 |
    +--------------------------------------+----------+-------+---------------+------+----------------------------+

We can rely on the query option to constrain the query, for example by timestamp:

    # ceilometer sample-list --meter cpu -q 'timestamp>2016-08-23T18:40:00;timestamp<=2016-08-23T18:45:00'
    +--------------------------------------+------+------------+-------------+------+----------------------------+
    | Resource ID                          | Name | Type       | Volume      | Unit | Timestamp                  |
    +--------------------------------------+------+------------+-------------+------+----------------------------+
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu  | cumulative | 8.37122e+12 | ns   | 2016-08-23T18:44:27.816000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu  | cumulative | 8.31506e+12 | ns   | 2016-08-23T18:43:27.823000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu  | cumulative | 8.25889e+12 | ns   | 2016-08-23T18:42:27.814000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu  | cumulative | 8.20286e+12 | ns   | 2016-08-23T18:41:27.925000 |
    | 377b756d-ffdc-4881-a5da-d027f4c53877 | cpu  | cumulative | 8.14657e+12 | ns   | 2016-08-23T18:40:27.808000 |
    +--------------------------------------+------+------------+-------------+------+----------------------------+

##### Statistics
A statistic is set of samples aggregates over a given time window called period. Ceilometer currently employs 5 different aggregation functions:

1. **count**: the number of samples in each period
2. **max**: the maximum of the sample volumes in each period
3. **min**: the minimum of the sample volumes in each period
4. **avg**: the average of the sample volumes over each period
5. **sum**: the sum of the sample volumes over each period

For example, if we are interested in average CPU utilization samples aggregated by period of 5 minutes, we can query for statistics as following:

        # ceilometer statistics --period 300 --meter cpu_util --aggregate avg
        +--------+----------------------------+----------------------------+---------------+----------+----------------------------+----------------------------+
        | Period | Period Start               | Period End                 | Avg           | Duration | Duration Start             | Duration End               |
        +--------+----------------------------+----------------------------+---------------+----------+----------------------------+----------------------------+
        | 300    | 2016-08-22T15:15:35.874000 | 2016-08-22T15:20:35.874000 | 1.02116748717 | 0.0      | 2016-08-22T15:16:53.842000 | 2016-08-22T15:16:53.842000 |
        | 300    | 2016-08-22T15:25:35.874000 | 2016-08-22T15:30:35.874000 | 1.08469925267 | 0.0      | 2016-08-22T15:26:54.008000 | 2016-08-22T15:26:54.008000 |
        | 300    | 2016-08-22T16:15:35.874000 | 2016-08-22T16:20:35.874000 | 10.7021343894 | 0.001    | 2016-08-22T16:16:53.855000 | 2016-08-22T16:16:53.856000 |
        | 300    | 2016-08-22T16:25:35.874000 | 2016-08-22T16:30:35.874000 | 46.3678514945 | 0.002    | 2016-08-22T16:26:54.033000 | 2016-08-22T16:26:54.035000 |
        | 300    | 2016-08-22T16:35:35.874000 | 2016-08-22T16:40:35.874000 | 38.2137658871 | 188.705  | 2016-08-22T16:36:53.882000 | 2016-08-22T16:40:02.587000 |
        | 300    | 2016-08-22T16:40:35.874000 | 2016-08-22T16:45:35.874000 | 43.1203780243 | 0.002    | 2016-08-22T16:43:00.210000 | 2016-08-22T16:43:00.212000 |
        | 300    | 2016-08-22T16:45:35.874000 | 2016-08-22T16:50:35.874000 | 44.839546195  | 179.438  | 2016-08-22T16:47:00.975000 | 2016-08-22T16:50:00.413000 |
        | 300    | 2016-08-22T16:50:35.874000 | 2016-08-22T16:55:35.874000 | 92.7558238772 | 240.019  | 2016-08-22T16:51:00.224000 | 2016-08-22T16:55:00.243000 |
        | 300    | 2016-08-22T16:55:35.874000 | 2016-08-22T17:00:35.874000 | 92.8823971135 | 120.131  | 2016-08-22T16:56:00.243000 | 2016-08-22T16:58:00.374000 |








##### Alarms
##### Pipelines





    