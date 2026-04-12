## **Hadoop**
![Steps](architecture.svg)

Hadoop is an open-source software framework for storing and processing big data in a distributed fashion on large clusters of commodity hardware. Essentially, it accomplishes two tasks: massive data storage and faster processing.
It was developed by the Apache Software Foundation and is based on two main components:

- Hadoop Distributed File System (HDFS): This is the storage component of Hadoop, designed to hold large amounts of data, potentially in the range of petabytes or even exabytes. The data is distributed across multiple nodes in the cluster, providing high availability and fault tolerance.

- Map-Reduce: This is the processing component of Hadoop, which provides a software framework for writing applications that process large amounts of data in parallel. MapReduce operations are divided into two stages: the Map stage, which sorts and filters the data, and the Reduce stage, which summarizes the data.

- Yet Another Resource Negotiator (YARN): This is the resource management layer in Hadoop. Introduced in Hadoop 2.0, YARN decouples the programming model from the resource management infrastructure, and it oversees and manages the compute resources in the clusters.

## **Properties of Hadoop**

- Scalability: Can store and distribute large data sets across many servers.

- Cost-effectiveness: Designed to run on inexpensive, commodity hardware.

- Flexibility: Can handle any type of data, structured or unstructured.

- Fault Tolerance: Data is automatically replicated to other nodes in the cluster.

- Data Locality: Processes data on or near the node where it's stored, reducing network I/O.

- Simplicity: Provides a simple programming model (MapReduce) for processing data.

- Open-source: Freely available to use and modify with a large community of contributors.

## **HDFS Architecture and Core Concepts**

HDFS (Hadoop Distributed File System) is a distributed file system. It is designed using a master/slave architecture.

A Hadoop cluster consists of multiple computers networked together.

A rack is a physical enclosure where multiple computers are fixed.Each rack typically has its individual power supply and a dedicated network switch. The importance of racks lies in the possibility of an entire rack failing if its switch or power supply goes out of network, affecting all computers within it. Multiple racks are connected, with their switches linked to a core switch, forming the Hadoop cluster.

> -- **Master/Slave Architecture: NameNode and DataNode**

In HDFS, there is one master and multiple slaves.

- NameNode (Master Node):
The Hadoop master is called the NameNode.
It is called NameNode because it stores and manages the names of directories and files within the HDFS namespace.

    Responsibilities:

     1. Manages the file system namespace. 
     2. Regulates access to files by clients (e.g., checking access permissions, user quotas). 
     3. Maintains an image of the entire HDFS namespace in memory, known as in-memory FS image (File System Image). This allows it to perform checks quickly. 
     4. Does not store actual file data. 
     5. Assigns DataNodes for block storage based on free disk space information from DataNodes. 
     6. Maintains the mapping of blocks to files, their order, and all other metadata.

- DataNode (Slave Node):
The Hadoop slaves are called DataNodes.
They are called DataNodes because they store and manage the actual data of the files.

    Responsibilities:
   
     1. Stores file data in the form of blocks. 
     2. Periodically sends a heartbeat to the NameNode to signal that it is alive. This heartbeat also includes resource capacity information that helps the NameNode in making decisions. 
     3. Sends a block report to the NameNode, which is health information about all the blocks maintained by that DataNode.

> --- **Key terminologies and Components of HDFS**


- Block

    ![Steps](block.svg)

    Block is nothing but the smallest unit of storage on a computer system. It is the smallest contiguous storage allocated to a file. In Hadoop, we have a default block size of 128MB or 256MB.

    !!! Note "Note"
        If you have a file of 50 MB and the HDFS block size is set to 128 MB, the file will only use 50 MB of one block. The remaining 78 MB in that block will remain unused, as HDFS blocks are allocated on a per-file basis.
        It's important to note that this is one of the reasons why HDFS is not well-suited to handling a large number of small files. Since each file is allocated its own blocks, if you have a lot of files that are much smaller than the block size, then a lot of space can be wasted.
        This is also why block size in HDFS is considerably larger than it is in other file systems
        (default of 128 MB, as opposed to a few KBs or MBs in other systems).

    Larger block sizes mean fewer blocks for the same amount of data, leading to less metadata to manage, less communication between the NameNode and DataNodes, and better performance for large, streaming reads of data.

- Replication Management

    To provide fault tolerance HDFS uses a replication technique. In that, it makes copies of the blocks and stores in on different DataNodes. Replication factor decides how many copies of the blocks get stored. It is 3 by default but we can configure to any value.

- Rack Awareness

    A rack contains many DataNode machines and there are several such racks in the production. HDFS follows a rack awareness algorithm to place the replicas of the blocks in a distributed fashion. This rack awareness algorithm provides for low latency and fault tolerance. Suppose the replication factor configured is 3. Now rack awareness algorithm will place the first block on a local rack. It will keep the other two blocks on a different rack. It does not store more than two blocks in the same rack if possible.

- Secondary Namenode

    The Secondary NameNode in Hadoop HDFS is a specially dedicated node in the Hadoop cluster that serves as a helper to the primary NameNode, but not as a standby NameNode. Its main roles are to take checkpoints of the filesystem metadata and help in keeping the filesystem metadata size within a reasonable limit.

    Here is what it does:

    1. Checkpointing: The Secondary NameNode periodically creates checkpoints of the namespace by merging the fsimage file and the edits log file from the NameNode. The new fsimage file is then transferred back to the NameNode. These checkpoints help reduce startup time of the NameNode

    2. Size management: The Secondary NameNode helps in reducing the size of the edits log file on the NameNode. By creating regular checkpoints, the edits log file can be purged occasionally, ensuring it does not grow too large.

    A common misconception is that the Secondary NameNode is a failover option for the primary NameNode. However, this is not the case; the Secondary NameNode cannot substitute for the primary NameNode in the event of a failure. For that, Hadoop 2 introduces the concept of Standby NameNode.

- Standby Namenode 

    In Hadoop, the Standby NameNode is part of the High Availability (HA) feature of HDFS that was introduced with Hadoop 2.x. This feature addresses one of the main drawbacks of the earlier versions of Hadoop: the single point of failure in the system, which was the NameNode.

    1. The Standby NameNode is essentially a hot backup for the Active NameNode. The Standby NameNode and Active NameNode are in constant synchronization with each other. When the Active NameNode updates its state, it records the changes to the edit log, and the Standby NameNode applies these changes to its own state, keeping both NameNodes in sync.

    2. The Standby NameNode maintains a copy of the namespace image in memory, just like the Active NameNode. This means it can quickly take over the duties of the Active NameNode in case of a failure, providing minimal downtime and disruption.

    3. Unlike the Secondary NameNode, the Standby NameNode is capable of taking over the role of the Active NameNode immediately without any data loss, thus ensuring the High Availability of the HDFS system.

    ![Steps](HadoopHAsvg.svg)

    Hadoop incorporates robust features for fault tolerance and high availability to ensure the reliability and continuous operation of the cluster. These two concepts, while related to system resilience, address different aspects of failure within the Hadoop ecosystem.

## **Hadoop Fault Tolerance**

Fault tolerance in Hadoop primarily addresses what happens when a data node fails. If a data file is broken into blocks and stored across various data nodes, the failure of one such node could lead to the loss of a part of the file, making it unreadable. Hadoop's solution to this fundamental problem is replication. It involves creating backup copies of each data block and storing them on different data nodes. This mechanism ensures that if one copy becomes unavailable, the data can still be read from another copy.

The number of copies made for each block is determined by the replication factor, which can be configured on a file-by-file basis and even modified after a file has been created in HDFS. For instance, if a file's replication factor is set to two, HDFS automatically creates two copies of each block for that file, ensuring they are placed on different machines. Typically, the replication factor is set to three, which is considered a reasonably good level of protection, though it can be increased for files deemed super critical.

Beyond individual node failures, Hadoop also provides protection against entire rack failures through rack awareness. Without rack awareness, if all three copies of a file's block were on nodes within the same rack, an entire rack failure would lead to the loss of all copies. By configuring Hadoop for rack awareness, it ensures that at least one copy of a block is placed in a different rack, thereby protecting against such widespread failures.

The Name Node plays a crucial role in maintaining the desired replication factor. Each data node sends periodic heartbeats to the Name Node. If a data node stops sending heartbeats, the Name Node identifies it as failed. In response, the Name Node initiates the re-replication of affected blocks to restore the number of replicas to the configured factor, for example, back to three. The Name Node constantly monitors and tracks the replication factor of every block and triggers replication whenever necessary. Reasons for re-replication can include a data node becoming unavailable, a replica getting corrupted, a hard disk failing on a data node, or even a user increasing the replication factor of a file.

While replication offers robust protection against failures, it comes with a cost: increased storage consumption. Making three copies of a file effectively reduces the cluster's usable storage capacity to one-third of its raw capacity, leading to higher costs. To mitigate this, Hadoop 2.x introduced storage policies, and Hadoop 3.x offers Erasure Coding as an alternative to traditional replication. Despite these alternatives, replication remains the conventional method for fault avoidance, and its costs are generally manageable because disks are relatively inexpensive.

## **Hadoop High Availability**

High availability refers to the uptime of a system, representing the percentage of time a service is operational. Enterprises typically aim for extremely high uptime, such as 99.999%, for their critical systems. It's important to distinguish high availability from fault tolerance: while data node, disk, or even rack failures, as discussed in fault tolerance, do not typically bring down the *entire* Hadoop cluster, high availability specifically addresses faults that would render the entire system unusable. The cluster, as a whole, usually remains available during data-related faults, with replication handling the underlying data protection.

The primary single point of failure in a Hadoop cluster is the Name Node. The Name Node is responsible for maintaining the file system namespace, including the list of directories and files, and managing the mapping of files to their blocks. Every client interaction with the Hadoop cluster begins with the Name Node. Consequently, if the Name Node fails, the entire Hadoop cluster becomes unusable, preventing any read or write operations. Therefore, protecting against Name Node failures is essential to achieve high availability for a Hadoop cluster.

The fundamental solution to protect against any failure is a backup. For the Name Node, this involves two key aspects: backing up all the HDFS namespace information that the Name Node maintains and having a standby Name Node machine readily available to take over its role quickly.

The Name Node maintains the complete file system in its memory as an in-memory FS image. Additionally, it keeps an edit log on its local disk, which records every change made to the file system like a journal. Because the in-memory FS image can be reconstructed from the edit log, backing up the Name Node's edit log is crucial. The recommended solution for backing up the edit log in Hadoop 2.x is the Quorum Journal Manager (QJM). The QJM consists of at least three machines, each running a lightweight Journal Node daemon. Instead of writing edit log entries to its local disk, the Name Node is configured to write them to the QJM. Utilizing three Journal Nodes provides double protection for the critical edit log, and for even higher protection, a QJM can consist of five or seven nodes.

A separate machine is added to the cluster and configured as a Standby Name Node. This Standby Name Node is set up to continuously read the edit log from the QJM, ensuring it stays updated with the latest file system changes. This configuration enables the Standby Name Node to assume the active Name Node role within a few seconds. Furthermore, all data nodes are configured to send their block reports (health information for blocks) to both the Active and Standby Name Nodes.

The mechanism by which the Standby Name Node determines that the Active Name Node has failed and should take over is managed by Zookeeper and Failover Controllers. A Failover Controller runs on each Name Node. The Failover Controller on the active Name Node maintains a lock in Zookeeper, while the Standby Name Node's Failover Controller continuously attempts to acquire this lock. If the Active Name Node fails or crashes, the lock it held in Zookeeper expires. As soon as the Standby Name Node successfully acquires the lock, it recognizes that the active Name Node has failed and proceeds to transition from its standby state to the active role.

## **Secondary Name Node**

It is common to confuse the Secondary Name Node with the Standby Name Node, but they serve distinct purposes. As explained, a Standby Name Node acts as a direct backup for the Name Node in case of failure. The Secondary Name Node, however, addresses a different operational concern.

When the Name Node restarts, for example, due to maintenance, it loses its in-memory FS image. It then reconstructs this image by reading the edit log. The challenge arises because the edit log grows continuously, and its size directly impacts the Name Node's restart time. A very large edit log could cause the Name Node to take an hour or more to start, which is undesirable.

The Secondary Name Node solves this problem by performing a checkpoint activity periodically, typically every hour. During a checkpoint, the Secondary Name Node performs the following steps:

1.   It reads the current edit log.

2.  It then creates the latest file system state, which is an exact copy of the in-memory FS image.

3.   This state is then saved to disk as an on-disk FS image.

4.   Once the new on-disk FS image is created, the Secondary Name Node truncates (clears) the edit log, as all the changes have been applied to the new on-disk image.

5.   For subsequent checkpoints, the Secondary Name Node reads the *previous* on-disk FS image and applies only the new changes accumulated in the edit log during the last hour. It then replaces the old on-disk FS image with the new one and truncates the edit log again.

This checkpointing process is essentially a merging of an on-disk FS image and the edit log. It is a quick process because it only deals with a limited amount of new edit logs (e.g., from the last hour), and the FS image itself is smaller compared to the cumulative edit log. The primary benefit is that when the Name Node eventually restarts, it also performs this quick checkpoint activity, reading a much smaller edit log (only from the last checkpoint) and applying it to the latest on-disk FS image, thus minimizing the restart time.

It is important to note that when a Hadoop High Availability configuration is implemented with a Standby Name Node, the Standby Name Node also performs this checkpoint activity. Consequently, in a high availability setup, a separate Secondary Name Node service is no longer required.

## **Write Operation in HDFS**

![Steps](write.svg)

When a client intends to write a file into HDFS, the very first step involves the client interacting with the NameNode. The NameNode is considered the "centerpiece" of the HDFS cluster because it stores all the metadata and possesses complete information about the entire cluster's data and its slave nodes (DataNodes). Therefore, any write operation must begin with a create request sent from the client to the File System API, which then forwards it to the NameNode. The NameNode's initial role is to check for access rights to ensure that the specific user or users have permission to write to the requested path.

Once access rights are verified, the NameNode's crucial function is to provide the address of the slave (DataNode) where the client should begin writing the data directly. It's a significant point to understand that the client sends only one copy of the data. This is a key design choice because if the client were to send multiple copies (e.g., three copies for a replication factor of three), it would create significant network overhead. For instance, writing 10 TB of data would necessitate sending 30 TB over the network, which is inefficient.

After the client starts writing the data directly to the initial DataNode via an FS Data Output Stream, the replication process begins among the DataNodes themselves, not initiated by the client. This process follows a data write pipeline: once the first DataNode has received and started writing a block, it immediately starts copying that block to another DataNode. This second DataNode, upon receiving the block, in turn starts copying it to a third DataNode, and so on, until the required replication level is achieved. For example, if the replication factor is three, the block will be written to DataNode 1, then DataNode 1 will copy to DataNode 3, and DataNode 3 will copy to DataNode 7. All decisions regarding which DataNodes handle the replication are taken care of by the NameNode (the master). DataNodes are in constant communication with the NameNode and report block information.

Once the required replicas are created, an acknowledgement process takes place in reverse order. The last DataNode in the pipeline sends an acknowledgement to the second-to-last DataNode, which then sends it to the first DataNode. Finally, the first DataNode sends the ultimate acknowledgement back to the client.

An important aspect of HDFS write operations is that they occur in parallel. It's not a serialized process where block one is fully written before block two begins. While write operations are generally costly, HDFS's parallel writing prevents them from being prohibitively slow, ensuring that writing terabytes of data doesn't take days. Furthermore, HDFS is designed to automatically handle failures during writing. If any DataNode goes down while data is being written, the NameNode will immediately provide the address of another DataNode where the data can be copied, ensuring data integrity and availability.

A common question is who divides the large file into smaller blocks. The responsibility for dividing the file into smaller blocks falls to the HDFS client itself (also referred to as the Hadoop client). The user or their specific machine does not need to manually divide or specify any logic for splitting the file. The Hadoop setup itself acts as the Hadoop client, containing all the necessary APIs to perform this division automatically. You, as the user, are the client, but the actual HDFS setup on your machine acts as the 'Hadoop client' that performs the block division.

## **Read Operation in HDFS**

![Steps](read.svg)


When a client wishes to read a file from HDFS, the initial and crucial step involves the client interacting with the NameNode. The NameNode is recognized as the "centerpiece" of the HDFS cluster because it is responsible for storing all the metadata related to files and their block locations across the various DataNodes. Before providing any information, the NameNode first checks for access rights to ensure that the particular user or client is authorized to read the requested file.

Once access rights are verified, the NameNode's primary role shifts to providing the specific locations of the data blocks. It will furnish the client with the addresses of the DataNodes where each block of the file is actually stored. For example, it might instruct the client to "go on slave two to read block one," "go on slave ten to read block two," and so on. A very important distinction in HDFS is that the client then directly interacts with these respective DataNodes to read the file, and the NameNode is not involved in the actual data transfer during the read process.

The reason for this direct client-DataNode interaction, bypassing the NameNode for data flow, is critical: the NameNode would become a severe bottleneck if all data had to pass through it. Considering that HDFS deals with data in the range of petabytes and operates across thousands of nodes, routing all read operations through a single NameNode would make it incredibly inefficient and slow. Therefore, the read operation itself is distributed and performed in parallel directly from the DataNodes, which significantly enhances efficiency. For instance, one client might be reading a block from DataNode One, while another client simultaneously reads a different block from DataNode Three, or even the same client reads different blocks from different DataNodes in parallel.

To ensure security during this direct interaction, the NameNode doesn't just give out DataNode addresses freely. It also provides a token to the client. This token acts like an "ID card"; the client must show this token to the DataNode for authentication before the DataNode grants access to the data. This mechanism prevents unauthorized access to the data stored on the DataNodes.

At a deeper, API level, when an end-user initiates a read operation on their client machine, a Java Virtual Machine (JVM) starts. The very first API class to come into play is the `HDFS client class`. This class sends an "open request" to the `Distributed File System API`. This `File System API` then interacts directly with the NameNode to request the block locations. After the NameNode performs its authorization and authentication checks, it provides the necessary block locations. Following this, the client uses an `FS Data Input Stream` – a standard Java API specifically adapted for Hadoop – to start directly reading the data from the DataNodes.

Finally, HDFS is designed with robust fault tolerance during read operations. If, at any point while reading data, a DataNode goes down (e.g., due to a server failure or power loss), there are no issues. The client will immediately complain to the NameNode about the problem. The NameNode, which constantly monitors its DataNodes through heartbeats, will detect that the particular slave has stopped responding. In such a scenario, the NameNode will then provide the location of another DataNode where the same block of data is available. This ensures that the client can continue reading the file seamlessly without data loss or interruption.

## **IQ**

!!!- info "What is the difference between Nodes in HDFS?"
    The differences between NameNode, BackupNode and Checkpoint NameNode are as follows:

    NameNode: NameNode is at the heart of the HDFS file system that manages the metadata i.e. the data of the files is not stored on the NameNode but rather it has the directory tree of all the files present in the HDFS file system on a Hadoop cluster. NameNode uses two files for the namespace:
    fsimage file: This file keeps track of the latest checkpoint of the namespace. edits file: This is a log of changes made to the namespace since checkpoint.

    Checkpoint Node:    Checkpoint Node keeps track of the latest checkpoint in a directory that has the same structure as that of NameNode’s directory.
    Checkpoint node creates checkpoints for the namespace at regular intervals by downloading the edits and fsimage file from the NameNode and merging it locally. The new image is then again updated back to the active NameNode.

    BackupNode: This node also provides check pointing functionality like that of the Checkpoint node but it also maintains its up-to-date in-memory copy of the file system namespace that is in sync with the active NameNode.

!!!- info "How will you disable a Block Scanner on HDFS DataNode?"
    In HDFS, there is a configuration dfs.datanode.scan.period.hours in hdfs-site.xml to set the number of hours interval at which Block Scanner should run.
    We can set dfs.datanode.scan.period.hours=0 to disable the Block Scanner. It means it will not run on HDFS DataNode.

!!!- info "How will you get the distance between two nodes in Apache Hadoop?"
    In Apache Hadoop we can use the NetworkTopology.getDistance() method to get the distance between two nodes.
    Distance from a node to its parent is considered as 1.

!!!- info "How does inter cluster data copying work in Hadoop?"
    In Hadoop, there is a utility called DistCP (Distributed Copy) to perform large
    inter/intra-cluster copying of data. This utility is also based on MapReduce. It creates Map tasks for files given as input.
    After every copy using DistCP, it is recommended to run cross checks to confirm that there is no data corruption and the copy is complete.

!!!- info "What is the Replication factor in HDFS?"
    Replication factor in HDFS is the number of copies of a file in a file system. A Hadoop application can specify the number of replicas of a file it wants HDFS to maintain. This information is stored in NameNode. We can set the replication factor in following ways:

    We can use Hadoop fs shell, to specify the replication factor for a file. Command as follows:
    $hadoop fs –setrep –w 5 /file_name
    In the above command, the replication factor of file_name file is set as 5.

    We can also use Hadoop fs shell, to specify the replication factor of all the files in a directory.
    $hadoop fs –setrep –w 2 /dir_name
    In the above command, the replication factor of all the files under directory dir_name is set as 2.

!!!- info "What are the two messages that NameNode receives from DataNode?"
    NameNode receives following two messages from every DataNode:

        Heartbeat: This message signals that DataNode is still alive. Periodic receipt of Heartbeat is very important for NameNode to decide whether to use a DataNode or not.

        Block Report: This is a list of all the data blocks hosted on a DataNode. This report is also very useful for the functioning of NameNode. With this report, NameNode gets information about what data is stored on a specific DataNode.

!!!- info "How does indexing work in Hadoop?"
    Indexing in Hadoop has two different levels.
    Index based on File URI: In this case data is indexed based on different files. When we search for data, the index will return the files that contain the data.
    Index based on InputSplit: In this case, data is indexed based on locations where input split is located.

!!!- info "What data is stored in a HDFS NameNode?"
    NameNode is the central node of an HDFS system. It does not store any actual data on which MapReduce operations have to be done. But it has all the metadata about the data stored in HDFS DataNodes.
    NameNode has the directory tree of all the files in the HDFS filesystem. Using this metadata it manages all the data stored in different DataNodes.

!!!- info "What would happen if NameNode crashes in a HDFS cluster?"
    There is only one NameNode in a HDFS cluster. This node maintains metadata about DataNodes. Since there is only one NameNode, it is the single point of failure in a HDFS cluster. When NameNode crashes, the system may become unavailable.
    We can specify a secondary NameNode in the HDFS cluster. The secondary NameNode takes the periodic checkpoints of the file system in HDFS. But it is not the
    backup of NameNode. We can use it to recreate the NameNode and restart it in case of a crash.


!!!- info "What are the main functions of Secondary NameNode?"
    Main functions of Secondary NameNode are as follows:

    FsImage: It stores a copy of the FsImage file and EditLog.

    NameNode crash: In case NameNode crashes, we can use Secondary NameNode's FsImage to recreate the NameNode.

    Checkpoint: Secondary NameNode runs Checkpoint to confirm that data is not corrupt in HDFS.

    Update: It periodically applies the updates from EditLog to the FsImage file. In this way the FsImage file on Secondary NameNode is kept up to date. This helps in saving time during NameNode restart.


!!!- info "What happens if an HDFS file is set with a replication factor of 1 and DataNode crashes?"
    Replication factor is the same as the number of copies of a file on HDFS. If we set the replication factor of 1, it means there is only 1 copy of the file.
    In case, DataNode that has this copy of file crashes, the data is lost. There is no way to recover it. It is essential to keep a replication factor of more than 1 for any business critical data.
 

!!!- info "What is the meaning of Rack Awareness in Hadoop?"
    In Hadoop, most of the components like NameNode, DataNode etc are rack- aware. It means they have the information about the rack on which they exist. The main use of rack awareness is in implementing fault-tolerance.
    Any communication between nodes on the same rack is much faster than the communication between nodes on two different racks.
    In Hadoop, NameNode maintains information about the rack of each DataNode. While reading/writing data, NameNode tries to choose the DataNodes that are closer to each other. Due to performance reasons, it is recommended to use close data nodes for any operation. So Rack Awareness is an important concept for high performance and fault- tolerance in Hadoop.
    If we set Replication factor 3 for a file, does it mean any computation will also take place 3 times?"
    No. Replication factor of 3 means that there are 3 copies of a file. But computation takes place only one copy of the file. If the node on which the first copy exists does not respond then computation will be done on the second copy.


!!!- info "How will you check if a file exists in HDFS?"
    In Hadoop, we can run hadoop fs command with option e to check the existence of a file in HDFS. This is generally used for testing purposes. Command will be as follows:
    + %>hadoop fs -test -ezd file_uri e is for checking the existence of file z is for checking non-zero size of
    File d is for checking if the path is directory


!!!- info "Why do we use the fsck command in HDFS?"
    fsck command is used for getting the details of files and directories in HDFS. Main uses of fsck command in HDFS are as follows:
    delete: We use this option to delete files in HDFS.

    move: This option is for moving corrupt files to lost/found.

    locations: This option prints all the locations of a block in HDFS.

    racks: This option gives the network topology of data-node locations.
    
    blocks: This option gives the report of blocks in HDFS.


!!!- info "How does partitioning work in Hadoop?"
    Partitioning is the phase between Map phase and Reduce phase in Hadoop workflow. Since the partitioner gives output to Reducer, the number of partitions is the same as the number of Reducers.
    Partitioner will partition the output from Map phase into distinct partitions by using a user-defined condition.
    Partitions can be like Hash based buckets.
    E.g. If we have to find the student with the maximum marks in each gender in each subject. We can first use the Map function to map the keys with each gender. Once mapping is done, the result is passed to the Partitioner. Partitioner will partition each row with gender on the basis of subject. For each subject there will be a different Reducer. Reducer will take input from each partition and find the student with the highest marks.


!!!- info "Why does HDFS store data in Block structure?"
    HDFS stores all the data in terms of Blocks. With Block structure there are some benefits that HDFS gets. Some of these are as follows:
    Fault Tolerance: With Block structure, HDFS implements replication. By replicating the same block in multiple locations, fault tolerance of the system increases. Even if some copy is not accessible, we can get the data from another copy.
    Large Files: We can store very large files that cannot be even stored on one disk, in HDFS by using Block structure. We just divide the data of the file in multiple Blocks. Each Block can be stored on the same or different machines.
    Storage management: With Block storage it is easier for Hadoop nodes to calculate the data storage as well as perform optimization in the algorithm to minimise data transfer across the network.


!!!- info "How will you create a custom Partitioner in a Hadoop job?"
    Partition phase runs between Map and Reduce phase. It is an optional phase. We can create a custom partitioner by extending the org.apache.hadoop.mapreduce.Partitio class in Hadoop. In this class, we have to override the getPartition(KEY key, VALUE value, int numPartitions) method.
    This method takes three inputs. In this method, numPartitions is the same as the number of reducers in our job. We pass key and value to get the partition number to which this key,value record will be assigned. There will be a reducer corresponding to that partition.
    The reducer will further handle summarizing of the data.Once the custom Partitioner class is ready, we have to set it in the Hadoop job. We can use following method to set it:
    job.setPartitionerClass(CustomPartitioner)


!!!- info "What is a Checkpoint node in HDFS?"
    A Checkpoint node in HDFS periodically fetches fsimage and edits from NameNode, and merges them. This merge result is called a Checkpoint. Once a Checkpoint is created, Checkpoint Node uploads the Checkpoint to NameNode. Secondary nodes also take a Checkpoint similar to Checkpoint Node. But it does not upload the Checkpoint to NameNode.
    Main benefit of Checkpoint Node is in case of any failure on NameNode. A NameNode does not merge its edits to fsimage automatically during the runtime. If we have a long running task, the edits will become huge. When we restart NameNode, it will take much longer time, because it will first merge the edits. In such a scenario, a Checkpoint node helps for a long running task.
    Checkpoint nodes performs the task of merging the edits with fsimage and then uploads these to NameNode. This saves time during the restart of NameNode.
 

!!!- info "What is a Backup Node in HDFS?"
    Backup Node in HDFS is similar to Checkpoint Node. It takes the stream of edits from NameNode. It keeps these edits in memory and also writes these to storage to create a new checkpoint. At any point of time, Backup Node is in sync with the Name Node.
    The difference between Checkpoint Node and Backup Node is that Backup Node does not upload any checkpoints to Name Node. Also Backup node takes a stream instead of periodic reading of edits from Name Node.


!!!- info "What is the meaning of the term Data Locality in Hadoop?"
    In a Big Data system, the size of data is huge. So it does not make sense to move data across the network. In such a scenario, Hadoop tries to move computation closer to data. So the Data remains local to the location wherever it was stored. But the computation tasks will be moved to data nodes that hold the data locally.
    Hadoop follows following rules for Data Locality optimization:
    Hadoop first tries to schedule the task on a node that has an HDFS file on a local disk. If it cannot be done, then Hadoop will try to schedule the task on a node on the same rack as the node that has data. If this also cannot be done, Hadoop will schedule the task on the node with the same data on a different rack. The above method works well, when we work with the default replication factor of 3 in Hadoop.


!!!- info "What is a Balancer in HDFS?"
    In HDFS, data is stored in blocks on a DataNode. There can be a situation when data is not uniformly spread into blocks on a DataNode. When we add a new DataNode to a cluster, we can face such a situation. In such a case, HDFS provides a useful tool Balancer to analyze the placement of blocks on a DataNode. Some people call it a Rebalancer also. This is an administrative tool used by admin staff. We can use this tool to spread the blocks in a uniform manner on a DataNode.


!!!- info "What are the important points a NameNode considers before selecting the DataNode for placing a data block?"
    Some of the important points for selecting a DataNode by NameNode are as follows:
    NameNode tries to keep at least one replica of a Block on the same node that is writing the block.
    It tries to spread the different replicas of the same block on different racks, so that in case of one rack failure, another rack has the data.
    One replica will be kept on a node on the same node as the one that is writing it. It is different from point 1. In Point 1, a block is written to the same node. At this point the block is written on a different node on the same rack. This is important for minimizing the network I/O. NameNode also tries to spread the blocks uniformly among all the DataNodes in a cluster.
 

!!!- info "What is Safemode in HDFS?"
    Safemode is considered as the read-only mode of NameNode in a cluster. During the startup of NameNode, it was in SafeMode. It does not allow writing to the file-system in Safemode. At this time, it collects data and statistics from all the DataNodes. Once it has all the data on blocks, it leaves Safemode.
    The main reason for Safemode is to avoid the situation when NameNode starts replicating data in DataNodes before collecting all the information from DataNodes. It may erroneously assume that a block is not replicated well enough, whereas, the issue is that NameNode does not know about the whereabouts of all the replicas of a block. Therefore, in Safemode, NameNode first collects the information about how many replicas exist in a cluster and then tries to create replicas wherever the number of replicas is less than the policy.


!!!- info "How will you replace HDFS data volume before shutting down a DataNode?"
    In HDFS, DataNode supports hot swappable drives. With a swappable drive we can add or replace HDFS data volumes while the DataNode is still running. The procedure for replacing a hot swappable drive is as follows:
    First we format and mount the new drive. We update the DataNode configuration dfs.datanode.data.dir to reflect the data volume directories. Run the "dfsadmin -reconfig datanode HOST:PORT start" command to start the reconfiguration process Once the reconfiguration is complete, we just unmount the old data volume After unmount we can physically remove the old disks.

!!!- info "Why do we need Serialization in Hadoop map reduce methods?"
    In Hadoop, there are multiple data nodes that hold data. During the processing of map and reduce methods data may transfer from one node to another node. Hadoop uses serialization to convert the data from Object structure to Binary format. With serialization, data can be converted to binary format and with de-serialization data can be converted back to Object format with reliability.


!!!- info "What is the use of Distributed Cache in Hadoop?"
    Hadoop provides a utility called Distributed Cache to improve the performance of jobs by caching the files used by applications. An application can specify which file it wants to cache by using JobConf configuration. Hadoop framework copies these files to the nodes one which a task has to be executed. This is done before the start of execution of a task.
    DistributedCache supports distribution of simple read only text files as well as complex files like jars, zips etc.





!!!- info "What are the important steps when you are partitioning a table?"
    Don’t over partition the data with too small partitions, it’s overhead to the namenode.
    if dynamic partition, at least one static partition should exist and set to strict mode by using given commands.
    SET hive.exec.dynamic.partition = true;
    SET hive.exec.dynamic.partition.mode = nonstrict;

    first load data into nonpartitioned table, then load such data into a partitioned table. It’s not possible to load data from local to partitioned tables.
    insert overwrite table table_name partition(year) select * from nonpartitiontable;

!!!- info "What is the difference between block And split?"
    Block: How much chunk data is stored in the memory called block.
    Split: how much data to process the data called split.


!!!- info "Why Hadoop Framework reads a file parallel, why not sequential?"
    To retrieve data faster, Hadoop reads data parallel, the main reason it can access data faster. While, writes in sequence, but not parallel, the main reason it might result is that one node can be overwritten by another and where the second node. Parallel processing is independent, so there is no relation between two nodes. If you write data in parallel, it’s not possible where the next chunk of data has. For example 100 MB data write parallel, 64 MB one block another block 36, if data writes parallel the first block doesn’t know where the remaining data is. So Hadoop reads parallel and writes sequentially.
 

!!!- info "If I change block size from 64 to 128?"
    Even if you have changed block size, it does not affect existing data. After changing the block size, every file chunked after 128 MB of block size. It means old data is in 64 MB chunks, but new data stored in 128 MB blocks.


!!!- info "How much Hadoop allows maximum block size and minimum block size?"
    Minimum: 512 bytes. It’s local OS file system block size. No one can decrease fewer than block size.
    Maximum: Depends on the environment. There is no upper bound.




!!!- info "What is speculative execution?"
    Hadoop runs the process in commodity hardware, so it’s possible to fail if the system also has low memory. So if system failed, process also failed, it’s not recommendable.Speculative execution is a process performance optimization technique.Computation/logic distribute to the multiple systems and execute which system execute quickly. By default this value is true. Now even if the system crashes, not a problem, the framework chooses logic from other systems.
    Eg: logic distributed on A, B, C, D systems, completed within a time.

    System A, System B, System C, System D systems executed 10 min, 8 mins, 9 mins 12 mins simultaneously. So consider system B and kill remaining system processes, framework take care to kill the other system process.
    



!!!- info "What are the setup and clean up methods?"
    If you don’t know the starting and ending points/lines, it’s much more difficult to solve those problems. Setup and clean up can resolve it. N number of blocks, by default 1 mapper called to each split. Each split has one start and clean up methods. N number of methods, number of lines. Setup is initialising job resources.
    The purpose of cleaning up is to close the job resources. The map processes the data.
    Once the last map is completed, cleanup is initialized. It Improves the data transfer performance. All these block size comparisons can be done in reducer as well. If you have any key and value, compare one key value to another key value. If you compare record levels, use these setup and cleanup. It opens once and processes many times and closes once. So it saves a lot of network wastage during the process.


!!!- info "How does a NameNode handle the failure of the Data Nodes?"
    HDFS has master/slave architecture. An HDFS cluster consists of a single
    NameNode, a master server that manages the file system namespace and regulates access to files by clients.
    In addition, there are a number of DataNodes, usually one per node in the cluster, which manage storage attached to the nodes that they run on. The NameNode and DataNode are pieces of software designed to run on commodity machines.
    NameNode periodically receives a Heartbeat and a Block report from each of the DataNodes in the cluster. Receipt of a Heartbeat implies that the DataNode is functioning properly. A Blockreport contains a list of all blocks on a DataNode. When NameNode notices that it has not received a heartbeat message from a data node after a certain amount of time, the data node is marked as dead. Since blocks will be under replication the system begins replicating the blocks that were stored on the dead DataNode. The NameNode Orchestrates the replication of data blocks from one DataNode to another. The replication data transfer happens directly between DataNode and the data never passes through the NameNode.
    



!!!- info "How is HDFS different from traditional File Systems?"
    HDFS, the Hadoop Distributed File System, is responsible for storing huge data on the cluster. This is a distributed file system designed to run on commodity hardware.
    It has many similarities with existing distributed file systems. However, the differences from other distributed file systems are significant.
    HDFS is highly fault-tolerant and is designed to be deployed on low-cost hardware.
    HDFS provides high throughput access to application data and is suitable for applications that have large data sets.
    HDFS is designed to support very large files. Applications that are compatible with HDFS are those that deal with large data sets. These applications write their data only once but they read it one or more times and re
    !!!- info "Quire these reads to be satisfied at streaming speeds. HDFS supports write-once-read-many semantics on files.
 

!!!- info "What is Hdfs block size and how is it different from Traditional File System block size?"
    In HDFS data is split into blocks and distributed across multiple nodes in the cluster. Each block is typically 64Mb or 128Mb in size. Each block is replicated multiple times. Default is to replicate each block three times. Replicas are stored on different nodes. HDFS utilizes the local file system to store each HDFS block as a separate file. HDFS Block size can not be compared with the traditional file system block size.


!!!- info "What is a NameNode and how many instances of NameNode run on a Hadoop Cluster?"
    The NameNode is the centrepiece of an HDFS file system. It keeps the directory tree of all files in the file system, and tracks where across the cluster the file data is kept. It does not store the data of these files itself.
    There is only One NameNode process run on any hadoop cluster. NameNode runs on its own JVM process. In a typical production cluster it runs on a separate machine.
    The NameNode is a Single Point of Failure for the HDFS Cluster. When the NameNode goes down, the file system goes offline.
    Client applications talk to the NameNode whenever they wish to locate a file, or when they want to add /copy /move /delete a file. The NameNode responds to successful requests by returning a list of relevant DataNode servers where the data lives.


!!!- info "How does the client communicate with Hdfs?"
    The Client communication to HDFS happens using Hadoop HDFS API. Client applications talk to the NameNode whenever they wish to locate a file, or when they want to add/copy/move/delete a file on HDFS. The NameNode responds the successful requests by returning a list of relevant DataNode servers where the data lives. Client applications can talk directly to a DataNode, once the NameNode has provided the location of the data.


!!!- info "How the Hdfs blocks are replicated?"
    HDFS is designed to reliably store very large files across machines in a large cluster. It stores each file as a sequence of blocks; all blocks in a file except the last block are the same size. The blocks of a file are replicated for fault tolerance. The block size and replication factor are configurable per file. An application can specify the number of replicas of a file. The replication factor can be specified at file creation time and can be changed later. Files in HDFS are write-once and have strictly one writer at any time.
    The NameNode makes all decisions regarding replication of blocks. HDFS uses a
    rack-aware replica placement policy. In default configuration there are a total 3 copies of a data block on HDFS, 2 copies are stored on datanodes on the same rack and 3rd copy on a different rack.


!!!- info "Can you give some examples of Big Data?"
    There are many real life examples of Big Data! Facebook is generating 500+ terabytes of data per day, NYSE (New York Stock Exchange) generates about 1 terabyte of new trade data per day, a jet airline collects 10 terabytes of censor data for every 30 minutes of flying time. All these are day to day examples of Big Data!
 


!!!- info "What is structured and unstructured Data?"
    Structured data is the data that is easily identifiable as it is organized in a structure. The most common form of structured data is a database where specific information is stored in tables, that is, rows and columns. Unstructured data refers to any data that cannot be identified easily. It could be in the form of images, videos, documents, email, logs and random text. It is not in the form of rows and columns.


!!!- info "Since the data is replicated thrice in Hdfs, does it mean that any calculation done on One Node will also be replicated on the other Two?"
    Since there are 3 nodes, when we send the MapReduce programs, calculations will be done only on the original data. The master node will know which node exactly has that particular data. In case, if one of the nodes is not responding, it is assumed to be failed. Only then, the re
    !!!- info "Quired calculation will be done on the second replica.


!!!- info "What is throughput and how does Hdfs get a good throughput?" 
    Throughput is the amount of work done in a unit time. It describes how fast the data is getting accessed from the system and it is usually used to measure performance of the system. In HDFS, when we want to perform a task or an action, then the work is divided and shared among different systems. So all the systems will be executing the tasks assigned to them independently and in parallel. So the work will be completed in a very short period of time. In this way, the HDFS gives good throughput. By reading data in parallel, we decrease the actual time to read data tremendously.


!!!- info "What is streaming access?"
    As HDFS works on the principle of ‘Write Once, Read Many‘, the feature of streaming access is extremely important in HDFS. HDFS focuses not so much on storing the data but how to retrieve it at the fastest possible speed, especially while analyzing logs. In HDFS, reading the complete data is more important than the time taken to fetch a single record from the data.


!!!- info "How indexing is done in Hdfs?"
    Hadoop has its own way of indexing. Depending upon the block size, once the data is stored, HDFS will keep on storing the last part of the data which will say where the next part of the data will be. In fact, this is the base of HDFS.


!!!- info "What is a Rack?"
    Rack is a storage area with all the datanodes put together. These data nodes can be physically located at different places. Rack is a physical collection of datanodes which are stored at a single location. There can be multiple racks in a single location.


!!!- info "On what basis Data will be stored on a Rack?"
    When the client is ready to load a file into the cluster, the content of the file will be divided into blocks. Now the client consults the Namenode and gets 3 datanodes for every block of the file which indicates where the block should be stored. While placing the datanodes, the key rule followed is “for every block of data, two copies will exist in one rack, and a third copy in a different rack“. This rule is known as “Replica Placement Policy“.


!!!- info "Do we need to place 2nd and 3rd Data in Rack 2 only?"
    Yes, this is to avoid datanode failure.


!!!- info "What if Rack 2 and DataNode fails?"
    If both rack2 and datanode present in rack 1 fails then there is no chance of getting data from it. In order to avoid such situations, we need to replicate that data more number of times instead of replicating only thrice. This can be done by changing the value in the replication factor which is set to 3 by default.


!!!- info "What is the difference between Gen1 and Gen2 Hadoop with regards to the NameNode?"
    In Gen 1 Hadoop, Namenode is the single point of failure. In Gen 2 Hadoop, we have what is known as Active and Passive Namenodes kind of a structure. If the active Namenode fails, the passive Namenode takes over the charge.




 

!!!- info "Which are the two types of writes In Hdfs?"
    There are two types of writes in HDFS:
    Posted and non-posted write. Posted Write is when we write it and forget about it, without worrying about the acknowledgement.

    It is similar to our traditional Indian post.

    Non-posted Write, we wait for the acknowledgement. It is similar to today's courier services. Naturally, non-posted write is more expensive than the posted write. It is much more expensive, though both writes are asynchronous.


!!!- info "Why reading is done in parallel and writing is not in Hdfs?"
    Reading is done in parallel because by doing so we can access the data fast. But we do not perform the write operation in parallel. The reason is that if we perform the write operation in parallel, then it might result in data inconsistency. For example, you have a file and two nodes are trying to write data into the file in parallel, then the first node does not know what the second node has written and vice-versa. So, this makes it confusing which data to be stored and accessed.







!!!- info "What is the configuration of a typical Slave Node on a Hadoop Cluster and how many Jvms run on a Slave Node?"
    Single instance of a Task Tracker is run on each Slave node. Task tracker is run as a separate JVM process.
    Single instance of a DataNode daemon is run on each Slave node. DataNode daemon is run as a separate JVM process.
    One or Multiple instances of Task Instance is run on each slave node. Each task instance is run as a separate JVM process. The number of Task instances can be controlled by configuration. Typically a high end machine is configured to run more task instances.






!!!- info "Explain in brief the three Modes in which Hadoop can be run?"
    The three modes in which Hadoop can be run are:

    Standalone (local) mode - No Hadoop daemons running, everything runs on a single Java Virtual machine only.

    Pseudo-distributed mode - Daemons run on the local machine, thereby simulating a cluster on a smaller scale.

    Fully distributed mode - Runs on a cluster of machines.


!!!- info "Explain what are the features of Standalone local Mode?"
    In stand-alone or local mode there are no Hadoop daemons running, and everything runs on a single Java process. Hence, we don't get the benefit of distributing the code across a cluster of machines. Since it has no DFS, it utilizes the local file system. This mode is suitable only for running MapReduce programs by developers during various stages of development. It's the best environment for learning and good for debugging purposes.


!!!- info "What are the features of fully distributed mode?"
    In Fully Distributed mode, the clusters range from a few nodes to 'n' number of nodes. It is used in production environments, where we have thousands of machines in the Hadoop cluster. The daemons of Hadoop run on these clusters. We have to configure separate masters and separate slaves in this distribution, the implementation of which is quite complex. In this configuration, Namenode and Datanode run on different hosts and there are nodes on which the task tracker runs. The root of the distribution is referred to as HADOOP_HOME.


!!!- info "Explain what are the main features Of pseudo mode?"
    In Pseudo-distributed mode, each Hadoop daemon runs in a separate Java process, as such it simulates a cluster though on a small scale. This mode is used both for development and QA environments. Here, we need to do the configuration changes.



!!!- info "Which are the three main Hdfs site.xml properties?"
    The three main hdfs-site.xml properties are:

    Dfs.name.dir which gives you the location on which metadata will be stored and where DFS is located – on disk or onto the remote.

    Dfs.data.dir which gives you the location where the data is going to be stored.

    Fs.checkpoint.dir which is for secondary Namenode.



!!!- info "What does etc.init.d do?"
    /etc /init.d specifies where daemons (services) are placed or to see the status of these daemons. It is very LINUX specific, and has nothing to do with Hadoop.


!!!- info "What is the function Of Hadoop-env.sh and where is it present?"
    This file contains some environment variable settings used by Hadoop; it provides the environment for Hadoop to run. The path of JAVA_HOME is set here for it to run properly. Hadoop-env.sh file is present in the conf/hadoop-env.sh location. You can also create your own custom configuration file conf/hadoop-user-env.sh, which will allow you to override the default Hadoop settings.
 
