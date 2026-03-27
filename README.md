# Kite Standard Library

Cloud-agnostic resource types for [Kite](https://github.com/kitecorp/kite). Write infrastructure once, provision to any cloud provider.

## Overview

The standard library defines abstract resource schemas that providers map to their concrete implementations. Instead of writing cloud-specific code, you write:

```kite
import Server from "std/compute"

resource Server backend {
    name = "api-server"
    cpu = 4
    memory = 16
    image = "ubuntu-22.04"
}
```

The active provider (AWS, Azure, GCP) handles the translation — `Server` becomes an EC2 Instance on AWS, a Virtual Machine on Azure, etc.

## Standard Types

| Domain | Type | Description | AWS | Azure |
|--------|------|-------------|-----|-------|
| compute | `Server` | Compute instance | Ec2Instance | VirtualMachine |
| networking | `Network` | Virtual network | Vpc | Vnet |
| networking | `Subnet` | Network subnet | Subnet | Subnet |
| storage | `Bucket` | Object storage | S3Bucket | StorageAccount |
| loadbalancing | `LoadBalancer` | Load balancer | LoadBalancer | LoadBalancer |
| dns | `DnsZone` | DNS hosted zone | HostedZone | DnsZone |
| dns | `DnsRecord` | DNS record | RecordSet | DnsRecordSet |

## Usage

### Import

```kite
import Server from "std/compute"
import Network, Subnet from "std/networking"
import Bucket from "std/storage"
import * from "std"
```

### Provider Selection

```kite
// Uses default provider from kitefile.yml
resource Server backend {
    name = "api"
    cpu = 4
    memory = 16
    image = "ubuntu-22.04"
}

// Explicit provider
@provider("aws")
resource Server backend {
    name = "api"
    cpu = 4
    memory = 16
    image = "ubuntu-22.04"
}
```

### Default Provider

Configure in `kitefile.yml`:

```yaml
defaultProvider: aws
dependencies:
  - name: aws
    version: 0.1.8
```

## How It Works

1. User writes `resource Server backend { ... }` with abstract properties
2. Engine resolves the target provider (`@provider` decorator or `defaultProvider`)
3. Provider's adapter maps abstract properties to concrete ones (e.g., `cpu: 4, memory: 16` becomes `instanceType: "t3.xlarge"`)
4. Concrete handler performs the actual cloud API call
5. Cloud response is mapped back to abstract properties (`publicIp`, `state`, etc.)

## For Provider Authors

Providers declare which standard types they support in `provider.json`:

```json
{
  "name": "aws",
  "version": "0.1.8",
  "standardTypes": {
    "Server": "Ec2Instance",
    "Network": "Vpc",
    "Subnet": "Subnet",
    "Bucket": "S3Bucket"
  }
}
```

For custom property mapping logic, implement `StandardTypeAdapter<C>` from `kite-provider-sdk`:

```java
public class ServerAdapter implements StandardTypeAdapter<Ec2Instance> {
    public String standardTypeName() { return "Server"; }
    public String concreteTypeName() { return "Ec2Instance"; }

    public Ec2Instance toConcreteProperties(Map<String, Object> props) {
        return Ec2Instance.builder()
            .instanceType(resolveInstanceType(props))
            .imageId(resolveAmi(props))
            .build();
    }

    public Map<String, Object> toAbstractProperties(Ec2Instance concrete) {
        return Map.of(
            "publicIp", concrete.getPublicIp(),
            "state", concrete.getState()
        );
    }

    public ResourceTypeHandler<Ec2Instance> getConcreteHandler() { return handler; }
}
```

## Module Structure

```
kite-stdlib/
├── build.gradle
├── README.md
└── src/main/resources/stdlib/
    ├── compute/
    │   └── Server.kite
    ├── networking/
    │   ├── Network.kite
    │   └── Subnet.kite
    ├── storage/
    │   └── Bucket.kite
    ├── loadbalancing/
    │   └── LoadBalancer.kite
    └── dns/
        ├── DnsZone.kite
        └── DnsRecord.kite
```
