<properties
   pageTitle="Overview of the Azure Service Fabric Reliable Services configuration | Microsoft Azure"
   description="Learn about configuring stateful Reliable Services in Azure Service Fabric."
   services="Service-Fabric"
   documentationCenter=".net"
   authors="sumukhs"
   manager="timlt"
   editor=""/>

<tags
   ms.service="Service-Fabric"
   ms.devlang="dotnet"
   ms.topic="article"
   ms.tgt_pltfrm="NA"
   ms.workload="NA"
   ms.date="01/26/2016"
   ms.author="sumukhs"/>

# Configuring stateful Reliable Services
You can modify stateful Reliable Services' default configurations by using the configuration package (Config) or the service implementation (code).

+ **Config** - Configuration via the config package is accomplished by changing the Settings.xml file that is generated in the Microsoft Visual Studio package root under the Config folder for each service in the application.
+ **Code**   - Configuration via code is accomplished by overriding StatefulService.CreateReliableStateManager and creating a ReliableStateManager by using a  ReliableStateManagerConfiguration object with the appropriate options set.

By default, the Azure Service Fabric runtime looks for predefined section names in the Settings.xml file and consumes the configuration values while creating the underlying runtime components.

>[AZURE.NOTE] Do **not** delete the section names of the following configurations in the Settings.xml file that is generated in the Visual Studio solution unless you plan to configure your service via code.
Renaming the config package or section names will require a code change when configuring the ReliableStateManager.


## Replicator security configuration
Replicator security configurations are used to secure the communication channel that is used during replication. This means that services will not be able to see each other's replication traffic, ensuring that the data that is made highly available is also secure. By default, an empty security configuration section prevents replication security.

### Default section name
ReplicatorSecurityConfig

>[AZURE.NOTE] To change this section name, override the replicatorSecuritySectionName parameter to the ReliableStateManagerConfiguration constructor when creating the ReliableStateManager for this service.


## Replicator configuration
Replicator configurations configure the replicator that is responsible for making the stateful Reliable Service's state highly reliable by replicating and persisting the state locally.
The default configuration is generated by the Visual Studio template and should suffice. This section talks about additional configurations that are available to tune the replicator.

### Default section name
ReplicatorConfig

>[AZURE.NOTE] To change this section name, override the replicatorSettingsSectionName parameter to the ReliableStateManagerConfiguration constructor when creating the ReliableStateManager for this service.


### Configuration names
|Name|Unit|Default value|Remarks|
|----|----|-------------|-------|
|BatchAcknowledgementInterval|Seconds|0.05|Time period for which the replicator at the secondary waits after receiving an operation before sending back an acknowledgement to the primary. Any other acknowledgements to be sent for operations processed within this interval are sent as one response.|
|ReplicatorEndpoint|N/A|No default--required parameter|IP address and port that the primary/secondary replicator will use to communicate with other replicators in the replica set. This should reference a TCP resource endpoint in the service manifest. Refer to [Service manifest resources](service-fabric-service-manifest-resources.md) to read more about defining endpoint resources in a service manifest. |
|MaxPrimaryReplicationQueueSize|Number of operations|8192|Maximum number of operations in the primary queue. An operation is freed up after the primary replicator receives an acknowledgement from all the secondary replicators. This value must be greater than 64 and a power of 2.|
|MaxSecondaryReplicationQueueSize|Number of operations|16384|Maximum number of operations in the secondary queue. An operation is freed up after making its state highly available through persistence. This value must be greater than 64 and a power of 2.|
|CheckpointThresholdInMB|MB|200|Amount of log file space after which the state is checkpointed.|
|MaxRecordSizeInKB|KB|1024|Largest record size that the replicator may write in the log. This value must be a multiple of 4 and greater than 16.|
|OptimizeLogForLowerDiskUsage|Boolean|true|When true, the log is configured so that the replica's dedicated log file is created by using an NTFS sparse file. This lowers the actual disk space usage for the file. When false, the file is created with fixed allocations, which provide the best write performance.|
|MaxRecordSizeInKB|KB|1024|Largest record size that the replicator may write in the log. This value must be a multiple of 4 and greater than 16.|
|SharedLogId|guid|""|Specifies a unique guid to use for identifying the shared log file used with this replica. Typically, services should not use this setting. However, if SharedLogId is specified, then SharedLogPath must also be specified.|
|SharedLogPath|Fully qualified path name|""|Specifies the fully qualified path where the shared log file for this replica will be created. Typically, services should not use this setting. However, if SharedLogPath is specified, then SharedLogId must also be specified.|


## Sample configuration via code
```csharp
protected override IReliableStateManager CreateReliableStateManager()
{
    return new ReliableStateManager(
        new ReliableStateManagerConfiguration(
            new ReliableStateManagerReplicatorSettings
            {
                RetryInterval = TimeSpan.FromSeconds(3)
            }));
}
```


## Sample configuration file
```xml
<?xml version="1.0" encoding="utf-8"?>
<Settings xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.microsoft.com/2011/01/fabric">
   <Section Name="ReplicatorConfig">
      <Parameter Name="ReplicatorEndpoint" Value="ReplicatorEndpoint" />
      <Parameter Name="BatchAcknowledgementInterval" Value="0.05"/>
      <Parameter Name="CheckpointThresholdInMB" Value="512" />
   </Section>
   <Section Name="ReplicatorSecurityConfig">
      <Parameter Name="CredentialType" Value="X509" />
      <Parameter Name="FindType" Value="FindByThumbprint" />
      <Parameter Name="FindValue" Value="9d c9 06 b1 69 dc 4f af fd 16 97 ac 78 1e 80 67 90 74 9d 2f" />
      <Parameter Name="StoreLocation" Value="LocalMachine" />
      <Parameter Name="StoreName" Value="My" />
      <Parameter Name="ProtectionLevel" Value="EncryptAndSign" />
      <Parameter Name="AllowedCommonNames" Value="My-Test-SAN1-Alice,My-Test-SAN1-Bob" />
   </Section>
</Settings>
```


## Remarks
BatchAcknowledgementInterval controls replication latency. A value of '0' results in the lowest possible latency, at the cost of throughput (as more acknowledgement messages must be sent and processed, each containing fewer acknowledgements).
The larger the value for BatchAcknowledgementInterval, the higher the overall replication throughput, at the cost of higher operation latency. This directly translates to the latency of transaction commits.

The value for CheckpointThresholdInMB controls the amount of disk space that the replicator can use to store state information in the replica's dedicated log file. Increasing this to a higher value than the default could result in faster reconfiguration times when a new replica is added to the set. This is due to the partial state transfer that takes place due to the availability of more history of operations in the log. This can potentially increase the recovery time of a replica after a crash.

If you set OptimizeForLowerDiskUsage to true, log file space will be over-provisioned so that active replicas can store more state information in their log files, while inactive replicas will use less disk space. This makes it possible to host more replicas on a node. If you set OptimizeForLowerDiskUsage to false, the state information is written to the log files more quickly.

The MaxRecordSizeInKB setting defines the maximum size of a record that can be written by the replicator into the log file. In most cases, the default 1024-KB record size is optimal. However, if the service is causing larger data items to be part of the state information, then this value might need to be increased. There is little benefit in making MaxRecordSizeInKB smaller than 1024, as smaller records use only the space needed for the smaller record. We expect that this value would need to be changed in only rare cases.

The SharedLogId and SharedLogPath settings are always used together to make a service use a separate shared log from the default shared log for the node. For best efficiency, as many services as
possible should specify the same shared log. Shared log files should be placed on disks that are used solely for the shared log file to reduce head movement contention. We expect that this value would need to be changed in only rare cases.
