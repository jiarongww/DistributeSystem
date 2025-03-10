
# Quick Start
To deploy a 3-instance cluster on a local machine, execute the following script：<br>
cd distribute-java-cluster && sh deploy.sh <br>
This script will deploy three instances—example1, example2, and example3—in the distribute-java-cluster/env directory.
It will also create a client directory for testing the read and write functionality of the Raft cluster.<br>
After a successful deployment, test the write operation using the following script：
cd env/client <br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world <br>
Test the read operation with the following command：<br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello

# Usage

## Defining Data Write and Read Interfaces
```protobuf
message SetRequest {
    string key = 1;
    string value = 2;
}
message SetResponse {
    bool success = 1;
}
message GetRequest {
    string key = 1;
}
message GetResponse {
    string value = 1;
}
```
```java
public interface ExampleService {
    Example.SetResponse set(Example.SetRequest request);
    Example.GetResponse get(Example.GetRequest request);
}
```

## Server Usage
1. Implement the StateMachine interface
```java
// This interface contains three main methods used internally by Raft
public interface StateMachine {
    /**
     * Creates a snapshot of the data in the state machine, called periodically on each node
     * @param snapshotDir Directory where snapshot data is stored
     */
    void writeSnap(String snapshotDir);
    
    /**
     * Reads snapshot data into the state machine, called during node startup
     * @param snapshotDir Directory containing snapshot data
     */
    void readSnap(String snapshotDir);
    
    /**
     * Applies data to the state machine
     * @param dataBytes Binary data
     */
    void applyData(byte[] dataBytes);
}
```

2. Implement data write and read interfaces
```
// The ExampleService implementation should include the following members
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
```
```
// Data write logic
byte[] data = request.toByteArray();
// Synchronously write data to the Raft cluster
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();
```
```
// Data read logic, implemented by the specific application state machine
Example.GetResponse response = stateMachine.get(request);
```

3. Server startup logic
```
// Initialize RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());
// Application state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();
// Set Raft options, for example:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;
// Initialize RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);
// Register Raft consensus service for inter-node communication
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);
// Register Raft service for client calls
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);
// Register application-specific service
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);
// Start RPCServer and initialize Raft node
server.start();
raftNode.init();
```
# DistributeSystem
