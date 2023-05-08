# Engine Identity Validation

## Summary

Longhorn-manager communicates with longhorn-engine's gRPC ControllerService, ReplicaService, and SyncAgentService by sending requests to TCP/IP addresses kept up-to-date by its various controllers. Additionally, the longhorn-engine controller server sends requests to the longhorn-engine replica server's ReplicaService and SyncAgentService using TCP/IP addresses it keeps in memory. These addresses are relatively stable in normal operation. However, during periods of high process turnover (e.g. a node reboot or network event), it is possible for one longhorn-engine component to stop and another longhorn-engine component to start in its place using the same ports. If this happens quickly enough, other components with stale address lists attempting to execute requests against the old component may errantly execute requests against the new component. One harmful effect of this behavior that has been observed is the [expansion of an unintended longhorn-engine replica](https://github.com/longhorn/longhorn/issues/5709).

This proposal intends to ensure all gRPC requests to longhorn-engine components are actually served by the intended component.

### Related Issues

https://github.com/longhorn/longhorn/issues/5709

## Motivation

### Goals

- Eliminate the potential for negative effects caused by a Longhorn component communicating with an incorrect longhorn-engine component.
- Provide effective logging when incorrect communication occurs to aide in fixing TCP/IP address related race conditions.

### Non-goals [optional]

- Fix race conditions within the Longhorn control plane that lead to attempts to communicate with an incorrect longhorn-engine component.
- Refactor the in-memory data structures the longhorn-engine controller server uses to keep track of and initiate communication with replicas.

## Proposal

1. Add additional flags to the longhorn-engine CLI that inform controller and replica servers of thier associated volume and/or instance name.
1. Use [gRPC client interceptors](https://github.com/grpc/grpc-go/blob/master/examples/features/interceptor/README.md) to automatically inject [gRPC metadata](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md) (i.e. headers) containing volume and/or instance name information every time gRPC request is made.
1. Use [gRPC server interceptors](https://github.com/grpc/grpc-go/blob/master/examples/features/interceptor/README.md) to automatically validate the volume and/or instance name information in [gRPC metadata](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md) (i.e. headers) every time a gRPC request is received.
1. Reject any request (with an appropriate error code) if the provided information does not match the information a controller or replica server was launched with.
1. Log the rejection at the client and the server, making it easy to identify situations in which incorrect communication occurs.

Use gRPC [client and server interceptors](https://github.com/grpc/grpc-go/blob/master/examples/features/interceptor/README.md) to 

### User Stories

#### Story 1

Before this proposal:

As an administrator, after an intentional or unintentional node reboot, I notice one or more of my volumes is degraded and new or existing replicas aren't coming online. In some situations, the UI reports confusing information or one or more of my volumes might be unable to attach at all. Digging through logs, I see errors related to mismatched sizes, and at least one replica does appear to have a larger size reported in `volume.meta` than others. I don't know how to proceed.

After this proposal:

As an administrator, after an intentional or unintentional node reboot, my volumes work as expected. If I choose to dig through logs, I may see some messages about refused requests to incorrect components, but this doesn't seem to negatively affected anything.

#### Story 2

Before this proposal:

As a developer, I am aware that it is possible for one Longhorn component to communicate with another, incorrect component, and that this communication can lead to unexpected replica expansion. I want to work to fix this behavior. However, when I look at a support bundle, it is very hard to catch this communication occuring. I have to trace TCP/IP addresses through logs, and if no negative effects are caused, I may never notice it.

After this propsal:

Any time one Longhorn component attempts to communicate with another, incorrect component, it is clearly represented in the logs.

### User Experience In Detail

See the user stories above. This enhancement is intended to be largely transparent to the user. It should eliminate rare failures so that users can't run into them.

### API Changes

Increment the longhorn-engine CLIAPIVersion by one. Do not increment the longhorn-engine CLIAPIMinVersion. The changes in this LEP are backwards compatible. All gRPC metadata validation is by demand of the client. If a less sophisticated (not upgraded) client does not inject any metadata, the server performs no validation. If a less sophisticated (not upgraded) client only injects some metadata (e.g. `volume-name` but not `instance-name`), the server only validates the metadata provided.

Add an `instance-name` flag to the `longhorn controller <volume-name>` command (e.g. `longhorn controller <volume-name> --instance-name <instance-name>`). This command already accepts the volume name, so an additional flag would be redundant. The longhorn-engine controller server remembers its volume and instance name.

Add a `volume-name` flag and an `instance-name` flag to the `longhorn replica <directory>` command (e.g. `longhorn replica <directory> -volume-name <volume-name> -instance-name <instance-name>`). The longhorn-engine replica server remembers its volume and instance name.

Add a `volume-name` flag and an `instance-name` flag to the `longhorn replica <directory>` command (e.g. `longhorn replica <directory> -volume-name <volume-name> -instance-name <instance-name>`). The longhorn-engine sync-agent server remembers its volume and instance name.

Add a `volume-name` and `controller-instance-name` flag to every CLI command that launches an asynchronous task (e.g. `longhorn ls-replica -volume-name <volume-name> -controller-instance-name <controller-instance-name>`). All such commands create a controller client and these flags allow appropriate gRPC metadata to be injected into every client request. Requests that reach the wrong longhorn-engine controller server are rejected.

Add an additional `replica-instance-name` flag to CLI commands that launch asynchronous tasks that communicate directly with the longhorn-engine replica server (e.g. `longhorn add-replica <address> -size <size> -current-size <current-size> -volume-name <volume-name> -controller-instance-name <controller-instance-name> -replica-instance-name <replica-instance-name>`). All such commands create a replica client and these flags allow appropriate gRPC metadata to be injected into every client request. Requests that reach the wrong longhorn-engine replica server are rejected.

Return 14 NOT FOUND with an appropriate message when metadata validation fails. In this case, the NOT_FOUND refers to the longhorn-engine component the caller was actually attempting to communicate with. (The particular return code is definitely open to discussion.)

## Design

### Implementation Overview

#### Interceptors (longhorn-engine)

Add a gRPC server interceptor to all `grpc.NewServer` calls.

```
server := grpc.NewServer(withIdentityValidationInterceptor(volumeName, instanceName))
```

Implement the interceptor so that it validates metadata with best effort.

```
func withIdentityValidationInterceptor(volumeName, instanceName string) grpc.ServerOption {
	return grpc.UnaryInterceptor(identityValidationInterceptor(volumeName, instanceName))
}

func identityValidationInterceptor(volumeName, instanceName string) grpc.UnaryServerInterceptor {
	// Use a closure to remember the correct volumeName and/or instanceName.
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
		md, ok := metadata.FromIncomingContext(ctx)
		if ok {
			incomingVolumeName, ok := md["volume-name"]
			// Only refuse to serve if both client and server provide validation information.
			if ok && volumeName != "" && incomingVolumeName[0] != volumeName {
				return nil, status.Errorf(codes.InvalidArgument, "Incorrect volume name; check controller address")
			}
		}

		if ok {
			incomingInstanceName, ok := md["instance-name"]
			// Only refuse to serve if both client and server provide validation information.
			if ok && instanceName != "" && incomingInstanceName[0] != instanceName {
				return nil, status.Errorf(codes.InvalidArgument, "Incorrect instance name; check controller address")
			}
		}

		// Call the RPC's actual handler.
		return handler(ctx, req)
	}
}
```

Add a gRPC client interceptor to all `grpc.Dial` calls.

```
connection, err := grpc.Dial(serviceUrl, grpc.WithInsecure(), withIdentityValidationInterceptor(volumeName, instanceName))

```

Implement the interceptor so that it injects metadata with best effort.

```
func withIdentityValidationInterceptor(volumeName, instanceName string) grpc.DialOption {
	return grpc.WithUnaryInterceptor(identityValidationInterceptor(volumeName, instanceName))
}

func identityValidationInterceptor(volumeName, instanceName string) grpc.UnaryClientInterceptor {
	// Use a closure to remember the correct volumeName and/or instanceName.
	return func(ctx context.Context, method string, req any, reply any, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		if volumeName != "" {
			ctx = metadata.AppendToOutgoingContext(ctx, "volume-name", volumeName)
		}
		if instanceName != "" {
			ctx = metadata.AppendToOutgoingContext(ctx, "instance-name", instanceName)
		}
		return invoker(ctx, method, req, reply, cc, opts...)
	}
```

Modify all client constructors to include this additional information. Wherever these client packages are consumed (e.g. the replica client is consumed by the controller, both the replica and the controller clients are consumed by longhorn-manager), callers can inject this additional information into the constructor and get validation for free.

```
func NewControllerClient(address, volumeName, instanceName string) (*ControllerClient, error) {
    // Implementation.
}
```

#### CLI Commands (longhorn-engine)

Add additional flags to all longhorn-engine CLI commands depending on their function.

E.g. command that launches a server:

```
func ReplicaCmd() cli.Command {
	return cli.Command{
		Name:      "replica",
		UsageText: "longhorn controller DIRECTORY SIZE",
		Flags: []cli.Flag{
			// Other flags.
            cli.StringFlag{
				Name:  "volume-name",
				Value: "",
				Usage: "Name of the volume (for validation purposes)",
			},
			cli.StringFlag{
				Name:  "instance-name",
				Value: "",
				Usage: "Name of the instance (for validation purposes)",
			},
		},
        // Rest of implementation.
	}
}
```

E.g. command that directly communicates with both a controller and replica server.

```
func AddReplicaCmd() cli.Command {
	return cli.Command{
		Name:      "add-replica",
		ShortName: "add",
		Flags: []cli.Flag{
            // Other flags.
			cli.StringFlag{
				Name:     "volume-name",
				Required: false,
				Usage:    "Name of the volume (for validation purposes)",
			},
			cli.StringFlag{
				Name:     "controller-instance-name",
				Required: false,
				Usage:    "Name of the controller instance (for validation purposes)",
			},
			cli.StringFlag{
				Name:     "replica-instance-name",
				Required: false,
				Usage:    "Name of the replica instance (for validation purposes)",
			},
		},
		// Rest of implementation.
	}
}
```

#### Instance-Manager Integration

// TODO

#### Longhorn-Manager Integration

// TODO

### Test plan

Integration test plan.

For engine enhancement, also requires engine integration test plan.

### Upgrade strategy

Anything that requires if the user wants to upgrade to this enhancement.

## Note [optional]

Additional notes.
