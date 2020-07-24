# VEP-3 Vitess Upgrade Order

```
VEP: 3
Title: Vitess Upgrade Order
Author: Sugu Sougoumarane
Reviewers: Deepthi Sigireddi, Rafael Chacon Vivas
Status: Approved
Created: 2020-07-24
```

## Abstract

Vitess is predominantly a two layer system. At the top layer, we have vtgate, vtctld and vtworker. At the next layer, we have vttablet which connects to mysql.
In Kubernetes, we also use mysqlctld as a sidecar to manage the life cycle of the underlying mysql. This would belong to the vttablet layer.

Vitess also relies on an externally configured "topo server", which can be configured and upgraded independently.

In terms of interaction, the top layer sends requests to the vttablets. The vttablets send messages to each other. VTTablets also send messages to mysqlctld.
There is a currently unused use case where a vttablet can send a message to a vtgate requesting it to resolve a hung two-pc transaction. This use case will
not be taken into consideration in this document.

The previous upgrade order for vitess has used the "bottom to top" approach. This would imply the following order: mysqlctld, vttablet, vtgate, vtctld.
VTGate and vtctld can be upgraded in any order because they don't depend on each other.

The advantage of a bottom to top upgrade order is that a new API can be implemented in the bottom layer, and that API can be immediately used by the top layer.
While deploying the top layer, the bottom layers would already be upgraded. So, the new binaries will work. The old binaries will also continue to work because
of backward compatibility.

However, this approach poses a problem: one must upgrade all vttablets before upgrading the top layer. This is not very practical because it's not possible
to canary the vtgates: Deploying a new vtgate before all vttablet will make it connect to the older vttablets and use the newer API, and this will end up
failing. The same problem exists for vtctld.

In order to support the above use case, we will remove the upgrade order requirements starting from version 8.0: Components can be upgraded in any order.

This means that a new version of a server must work with an older version as well as the current version.

The does not change the one-version backward compatibility guarantee: For example, to upgrade from version 4.0 to 6.0, one must upgrade to version 5.0 first.
