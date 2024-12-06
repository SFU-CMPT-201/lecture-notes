# RPC

## Socket API vs. RPC

* Socket API
    * Fine-grained control
    * Low-level read/write
    * Communication oriented
    * But not programmer friendly
* Another Abstraction: RPC (Remote Procedure Call)
    * Goal: it should appear as if the programmer were calling a local function
    * A mechanism to enable function calls between different processes
    * First proposed in the 80's
    * Examples
        * Sun RPC
        * Java RMI
        * CORBA
    * Other examples that borrow the idea
        * XML-RPC
        * Android Bound Services with AIDL
        * Google Protocol Buffers
* RPC

    ```c
    // Client process
    int main (…)

    {
        ...
        rpc_call(...);
        ...
    }

    // Server process
    ...
    void rpc_call(…) {
        ...
    }
    ...
    ```

## Local Calls

* E.g., `x = local_call("str");`
* The compiler generates code to handle the call. This includes, transferring arguments to the
  function `local_call()`, jumping to the code for `local_call()` and executing it, transferring
  a return value to the caller, and returning back to the caller and continuing the execution.
    * Push the parameters to the stack.
    * Call `local_call()`.
    * Assigns registers.
    * Adjust stack pointers.
    * Saves the return value.
    * Calls the return instruction.
* Remote procedure calls provide an illusion of performing a local call.

## How RPC Works

* An RPC system uses an *Interface Definition Language (IDL)* to define an RPC call and the data
  structures for parameters and return values.
* For example, [gRPC](https://grpc.io/) is a popular RPC system, and it has [this
  example](https://grpc.io/docs/languages/cpp/quickstart/#update-the-grpc-service):

  ```c
  // The greeting service definition.
  service Greeter {
    // Sends a greeting
    rpc SayHello (HelloRequest) returns (HelloReply) {}
  }

  // The request message containing the user's name.
  message HelloRequest {
    string name = 1;
  }

  // The response message containing the greetings
  message HelloReply {
    string message = 1;
  }
  ```

  Suppose we have one server and one client. The above example uses gRPC's IDL to define a function
  called `SayHello()` that takes `HelloRequest` as the only parameter and `HelloReply` as the return
  value. The server implements this `SayHello()` function. The client calls the function. gRPC
  internally takes care of of the communication between the server and the client.
* An RPC system uses an IDL definition to generate *two* components called *client stub* and *server
  stub*.
    * Client stub: This is a *client-side library* that contains a function to be called. An RPC
      system automatically generates the code for this library. In the above example, gRPC
      automatically generates the client stub, which contains a function named `SayHello()`. The
      client calls this function to communicate with the server. Underneath, gRPC generates the
      implementation for `SayHello()`. It uses the socket API to communicate with the server, i.e.,
      sending the input parameters, letting the server know that it wants to execute `SayHello()`,
      receiving the return value, etc.
    * Server stub: This is a *server-side library* that communicates with a client using the socket
      API. It also defines a function to be executed. In the above example, gRPC automatically
      generates the server stub that calls `SayHello()`. Server-side developers need to implement
      `SayHello()`. When a new client requests comes in through the socket API, the server stub
      makes a call to `SayHello()` to execute it.
* Marshalling/Unmarshalling
    * When sending input parameters and receiving return values, an RPC system uses the socket API
      to transfer the data. Thus, we need to transform input data or return values into a byte
      stream that we can transfer over the network.
    * Turning data into a byte stream is called *marshalling*.
    * Turning a byte stream into data is called *unmarshalling*.
    * This is also called *serialization/deserialization*.
    * There are many libraries out there.
* Overall Operation

  ```bash
  Client Process                Server Process
  +-----------------+           +-----------------+
  | Client Function |           | Server Function |
  +-----------------+           +-----------------+
           |                             |
  +-----------------+           +-----------------+
  | Client Stub     |           | Server Stub     |
  +-----------------+           +-----------------+
           |  Marshalling/Unmarshalling  |
  +-----------------+           +-----------------+
  | Socket API      | --------- | Socket API      |
  +-----------------+           +-----------------+
  ```

## Invocation Semantics

* One significant difference is what is called *invocation semantics*.
* Unlike local calls, remote calls are subject to failures.
    * Lost request: A client request can be lost before reaching the server.
    * Lost reply: A client reply can be lost before reaching the client.
    * Crash before execution: The server can crash before executing the request.
    * Crash before reply: The server executes the request but can crash before sending a reply.
    * Client crash: The client crashes before receiving a reply.
* Local procedure calls provide *exactly-once* invocation semantics.
* Remote procedure calls depend on how failures are handled.
    * Client-side: Request retransmission
    * Server-side: Two possibilities regarding how to handle client-side retransmissions.
        * Re-execution: Easy to implement.
        * Duplicate filtering & retransmission of the previous reply: Extra implementation required.
* Invocation semantics: Depending on which server-side mechanism is used.
    * Maybe: An RPC call may be invoked at the server side. This is when no fault-tolerance
      mechanisms are used.
    * At-most-once: An RPC call will be invoked at most once. This is when (client-side) request
      retransmission and (server-side) duplicate filtering are combined.
    * Zero-or-more: An RPC call will be invoked zero or more times. This is when (client-side)
      request retransmission and (server-side) re-execution are combined.
* RPC is ideal for *idempotent functions*.
    * An idempotent function is a function that (i) produces the same output even if it is called
      multiple times, given the input is the same, and (ii) doesn't alter the system state after the
      first execution.
    * For RPC, an idempotent function is safe even with re-executions and there are no side effects.
