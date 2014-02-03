# Storm HDFS

Storm components for interacting with HDFS file systems


## Usage
The following example will write pipe("|")-delimited files to the HDFS path hdfs://localhost:54310/foo. After every 1,000 tuples it will sync filesystem, making that data visible to other HDFS clients. It will rotate files when they reach 5 megabytes in size.

```java
// use "|" instead of "," for field delimiter
RecordFormat format = new DelimitedRecordFormat()
        .withFieldDelimiter("|");

// sync the filesystem after every 1k tuples
SyncPolicy syncPolicy = new CountSyncPolicy(1000);

// rotate files when they reach 5MB
FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);

FileNameFormat fileNameFormat = new DefaultFileNameFormat();

HdfsBolt bolt = new HdfsBolt()
        .withFsUrl("hdfs://localhost:54310")
        .withPath("/foo/")
        .withFileNameFormat(fileNameFormat)
        .withRecordFormat(format)
        .withRotationPolicy(rotationPolicy)
        .withSyncPolicy(syncPolicy);
```

## Customization


### Record Formats
Record format can be controlled by providing an implementation of the `org.apache.storm.hdfs.format.RecordFormat` interface:

```java
public interface RecordFormat extends Serializable {
    byte[] format(Tuple tuple);
}
```

The provided `org.apache.storm.hdfs.format.DelimitedRecordFormat` is capable of producing formats such as CSV and tab-delimited files.


### File Naming
File naming can be controlled by providing an implementation of the `org.apache.storm.hdfs.format.FileNameFormat` interface:

```java
public interface FileNameFormat extends Serializable {
    void prepare(Map conf, TopologyContext topologyContext);
    String getName(long rotation, long timeStamp);
}
```

The provided `org.apache.storm.hdfs.format.DefaultFileNameFormat`  will create file names with the following format:

     {prefix}{componentId}-{taskId}-{rotationNum}-{timestamp}{extension}

For example:

     MyBolt-5-7-1390579837830.txt

By default, prefix is empty and extenstion is ".txt".



### Sync Policies
Sync policies allow you to control when buffered data is flushed to the underlying filesystem (thus making it available to clients reading the data) by implementing the `org.apache.storm.hdfs.sync.SyncPolicy` interface:

```java
public interface SyncPolicy extends Serializable {
    boolean mark(Tuple tuple, long offset);
    void reset();
}
```
The `HdfsBolt` will call the `mark()` method for every tuple it processes. Returning `true` will trigger the `HdfsBolt` to perform a sync/flush, after which it will call the `reset()` method.

The `org.apache.storm.hdfs.sync.CountSyncPolicy` class simply triggers a sync after the specified number of tuples have been processed.




### File Rotation Policies
Similar to sync policies, file rotation policies allow you to control when data files are rotated by providing a `org.apache.storm.hdfs.rotation.FileRotation` interface:

```java
public interface FileRotationPolicy extends Serializable {
    boolean mark(Tuple tuple, long offset);
    void reset();
}
``` 

The `org.apache.storm.hdfs.rotation.FileSizeRotationPolicy` implementation allows you to trigger file rotation when data files reach a specific file size:

```java
FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);
```

## Support for HDFS Sequence Files

The `org.apache.storm.hdfs.bolt.SequenceFileBolt` class allows you to write storm data to HDFS sequence files:

```java
        // sync the filesystem after every 1k tuples
        SyncPolicy syncPolicy = new CountSyncPolicy(1000);

        // rotate files when they reach 5MB
        FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, Units.MB);

        FileNameFormat fileNameFormat = new DefaultFileNameFormat().withExtension(".seq");

        // create sequence format instance.
        DefaultSequenceFormat format = new DefaultSequenceFormat("timestamp", "sentence");

        SequenceFileBolt bolt = new SequenceFileBolt()
                .withFsUrl("hdfs://localhost:54310")
                .withPath("/data/")
                .withFileNameFormat(fileNameFormat)
                .withSequenceFormat(format)
                .withRotationPolicy(rotationPolicy)
                .withSyncPolicy(syncPolicy)
                .withCompressionType(SequenceFile.CompressionType.RECORD)
                .withCompressionCodec("deflate");
```

The `SequenceFileBolt` requires that you provide a `org.apache.storm.hdfs.bolt.format.SequenceFormat` that maps tuples to
key/value pairs:

```java
public interface SequenceFormat extends Serializable {
    Class keyClass();
    Class valueClass();

    Writable key(Tuple tuple);
    Writable value(Tuple tuple);
}
```

## Trident API
storm-hdfs also includes a Trident `state` implementation for writing data to HDFS, with an API that closely mirrors that of the bolts.

 ```java
         Fields hdfsFields = new Fields("field1", "field2");

         FileNameFormat fileNameFormat = new DefaultFileNameFormat()
                 .withPrefix("trident")
                 .withExtension(".txt");

         RecordFormat recordFormat = new DelimitedRecordFormat()
                 .withFields(hdfsFields);

         FileRotationPolicy rotationPolicy = new FileSizeRotationPolicy(5.0f, FileSizeRotationPolicy.Units.MB);

         HdfsState.Options options = new HdfsState.Options()
                 .withFileNameFormat(fileNameFormat)
                 .withRecordFormat(recordFormat)
                 .withRotationPolicy(rotationPolicy)
                 .withFsUrl("hdfs://localhost:54310")
                 .withPath("/trident");

         StateFactory factory = new HdfsStateFactory().withOptions(options);

         TridentState state = stream
                 .partitionPersist(factory, hdfsFields, new HdfsUpdater(), new Fields());
 ```


## License

Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
