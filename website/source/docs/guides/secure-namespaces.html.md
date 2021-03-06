---
name: '[Enterprise] Setup Secure Namespaces' 
content_length: 14
id: secure-namespaces
products_used:
  - Consul
description: >-
  In this guide you setup secure namespaces with ACLs.
level: Implementation
---

!> Warning: This guide is a draft and has not been fully tested.

!> Warning: Consul 1.7 is currently a beta release.

Namespaces can provide separation for teams within a single organization so that they can share access to one or more Consul datacenters without conflict. This allows teams to deploy services without name conflicts and create more granular access to the cluster with namespaced ACLs. 

Additionally, namespaced ACLs will allow you to delegate access control to specific resources within the datacenter including services, Connect proxies, key/value pairs, and sessions.  

This guide has two main sections, configuring namespaces and creating ACL tokens within a namespace. You must configure namespaces before creating namespaced tokens.

## Prerequisites 

To execute the example commands in this guide, you will need a Consul 1.7.x  Enterprise datacenter with [ACLs enabled](/consul/security-networking/production-acls). If you do not have an existing datacenter, you can use a single [local agent](/consul/getting-started/agent) or [Consul in containers](/consul/day-0/containers-guide). 

You will also need an ACL token with `operator=write` and `acl=write` privileges or you can use a token with the [built-in global management policy](https://www.consul.io/docs/acl/acl-system.html#builtin-policies). 

### [Optional] Configure Consul CLI 

If you are using Docker or other non-local deployment, you can configure a local Consul binary to interact with the deployment. Set the `CONSUL_HTTP_ADDR` [variable](https://www.consul.io/docs/commands/index.html#consul_http_addr) on your local machine or jumphost to the IP address of a client. 

```shell
$ export CONSUL_HTTP_ADDR=192.17.23.4
```
Note, this jumphost will need to use an ACL token to access the datacenter. The token's necessary privileges will depend on the operations, learn more about privileges by reviewing the ACL [rule documentation](https://www.consul.io/docs/acl/acl-rules.html). You can export the token with the `CONSUL_HTTP_TOKEN` [variable](https://www.consul.io/docs/commands/index.html#consul_http_token). Additionally, if you have [TLS encryption configured](/consul/security-networking/certificates) you will need to use valid certificates. 

## Configure namespaces 

In this section, you will create two namespaces that allow you to separate the datacenter for two teams. Each namespace will have an operator responsible for managing data and access within their namespace. The namespaced-operator will only have access to view and update data and access in their namespace. This allows for complete isolation of data between teams. 

To configure and manage namespaces, you will need one super-operator who has visibility into the entire datacenter. It will be their responsibility to set up namespaces. For this guide, you should be the super-operator. 

### Create namespace definitions 

First, create files to contain the namespace definitions for the `app-team` and `db-team` respectively. The definitions can be JSON or HCL. Save the following configurations, which specify the name and description of each namespace, to the files.

```json
{
   "name": "app-team",
   "description": "Namespace for app-team managing the production dashboard application"
}
```

```json
{
   "name": "db-team",
   "description": "Namespace for db-team managing the production counting application"
}
```

These namespace definitions are for minimal configuration, learn more about namespace options in the [documentation]().

### Initialize the namespaces

Next, use the Consul CLI to create each namespace by providing Consul with the namespace definition files. You will need `operator=write` privileges.

```shell
$ consul namespace write app-team.json
$ consul namespace write db-team.json
```

Finally, ensure both namespaces were created by viewing all namespaces. You will need `operator=read` privileges, which are included with the `operator=write` privileges, a requirement from the prerequisites.

```shell
$ consul namespace list 
app-team:
   Description:
      Namespace for app-team managing the production dashboard application
db-team:
   Description:
      Namespace for db-team managing the production counting application
default:
   Description:
      Builtin Default Namespace
```

Alternatively, you can view each namespace with `consul namespace read`. After you create a namespace, you can [update]() or [delete]() it.

## Delegate token management with namespaces

In this section, you will delegate token management to multiple operators. One of the key benefits of namespaces is the ability to delegate responsibilities of token management to more operators. This allows you to provide unrestricted access to portions of the datacenter, ideally to one or a few operators per namespace. 

The namespaced-operators are then responsible for managing access to services, Consul KV, and other resources within their namespaces. Additionally, the namespaced-operator should further delegate service-access privileges to developers or end-users. This is consistent with the current ACL management workflow. Before namespaces, only one or a few operators managed tokens for an entire datacenter. 

Note that namespaces control access to Consul data and services. They don't have any impact on compute or other node resources, and nodes themselves are not namespaced.

Namespaced-operators will only be aware of data within their namespaces. Without global privileges, they will not be able to see other namespaces. 

Nodes are not namespaced, so namespace-operators will be able to see all the nodes in the datacenter.

### Create namespace management tokens

First, the super-operator should use the [built-in namespace-policy](https://www.consul.io/docs/acl/acl-system.html#builtin-policies) to create a token for the namespaced-operators. Note, the namespace-management policy ultimately grants them unrestricted privileges for their namespace. You will need `acl=write` privileges to create namespaced-tokens.

```shell
$ consul acl token create \
      -namespace app-team \
      -description "App Team Administrator" \
      -policy-name "namespace-management"
<output>
```
```shell
$ consul acl token create \
      -namespace db-team \
      -description "DB Team Administrator" \
      -policy-name "namespace-management"
<output>
```

These policies grant privileges to create tokens, which enable the token holders to grant themselves any additional privileges needed for any operation within their namespace. The namespace-policy privileges have the following rules.

```hcl
{
   acl = "write"
   kv_prefix "" = "write"
   service_prefix "" = "write"
   session_prefix "" = "write"
}
```

The namespaced-operators can now create tokens for developers or end-users that allow access to see or modify namespaced data, including service registrations, key/value pairs, and sessions.

## Summary

In this guide, you learned how to create namespaces and how to secure the resources within a namespace. 

Note, the super-operator can also create policies that can be shared by all namespaces. Shared policies are universal and should be created in the `default` namespace. 
