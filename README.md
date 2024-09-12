# Ampernetacle

This is a Terraform configuration to deploy a Kubernetes cluster on
[Oracle Cloud Infrastructure][oci]. It creates a few virtual machines
and uses [kubeadm] to install a Kubernetes control plane on the first
machine, and join the other machines as worker nodes.

By default, it deploys a 4-node cluster using ARM machines. Each machine
has 1 OCPU and 6 GB of RAM, which means that the cluster fits within
Oracle's (pretty generous if you ask me) [free tier][freetier].

**It is not meant to run production workloads,**
but it's great if you want to learn Kubernetes with a "real" cluster
(i.e. a cluster with multiple nodes) without breaking the bank, *and*
if you want to develop or test applications on ARM.

**https://medium.com/codex/run-your-docker-containers-for-free-in-the-cloud-and-for-unlimited-time-361515cb0876**
## Getting started

1. Create an Oracle Cloud Infrastructure account (just follow [this link][createaccount]).
2. Have installed or [install kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl).
3. Have installed or [install terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli?in=terraform/oci-get-started).
4. Have installed or [install OCI CLI ](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm).
5. Configure [OCI credentials](https://learn.hashicorp.com/tutorials/terraform/oci-build?in=terraform/oci-get-started).
   If you obtain a session token (with `oci session authenticate`), make sure to put the correct region, and when prompted for the profile name, enter `DEFAULT` so that Terraform finds the session token automatically.
6. Download this project and enter its folder.
7. `terraform init`
8. `terraform apply`

That's it!

At the end of the `terraform apply`, a `kubeconfig` file is generated
in this directory. To use your new cluster, you can do:

Linux
```bash
export KUBECONFIG=$PWD/kubeconfig
kubectl get nodes
```

Windows
```powershell
$env:KUBECONFIG="$pwd\kubeconfig"
kubectl get nodes
```

The command above should show you 4 nodes, named `node1` to `node4`.

You can also log into the VMs. At the end of the Terraform output
you should see a command that you can use to SSH into the first VM
(just copy-paste the command).

## Windows

It works with Windows 10/Powershell 5.1.

It may be necesssary to change the execution policy to unrestricted.

[PowerShell ExecutionPolicy](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.security/set-executionpolicy?view=powershell-5.1)

## Customization

Check `variables.tf` to see tweakable parameters. You can change the number
of nodes, the size of the nodes, or switch to Intel/AMD instances if you'd
like. Keep in mind that if you switch to Intel/AMD instances, you won't get
advantage of the free tier.

## Stopping the cluster

`terraform destroy`

## Implementation details

This Terraform configuration:

- generates an OpenSSH keypair and a kubeadm token
- deploys 4 VMs using Ubuntu 20.04
- uses cloud-init to install and configure everything
- installs Docker and Kubernetes packages
- runs `kubeadm init` on the first VM
- runs `kubeadm join` on the other VMs
- installs the Weave CNI plugin
- transfers the `kubeconfig` file generated by `kubeadm`
- patches that file to use the public IP address of the machine

## Caveats

This doesn't install the [OCI cloud controller manager][ccm],
which means that you cannot
create services with `type: LoadBalancer`; or rather, if you create
such services, their `EXTERNAL-IP` will remain `<pending>`.

To expose services, use `NodePort`.

Likewise, there is no ingress controller and no storage class.

These might be added in a later iteration of this project.
Meanwhile, if you want to install it manually, you can check
the [OCI cloud controller manager github repository][ccm].

## Remarks

Oracle Cloud also has a managed Kubernetes service called
[Container Engine for Kubernetes (or OKE)][oke]. That service
doesn't have the caveats mentioned above; however, it's not part
of the free tier.

## What does "Ampernetacle" mean?

It's a *porte-manteau* between Ampere, Kubernetes, and Oracle.
It's probably not the best name in the world but it's the one
we have! If you have an idea for a better name let us know. 😊

## Possible errors and how to address them

### Authentication problem

If you configured OCI authentication using a session token
(with `oci session authenticate`), please note that this token
is valid 1 hour by default. If you authenticate, then wait more
than 1 hour, then try to `terraform apply`, you will get
authentication errors.

#### Symptom

The following message:

```
 Error: 401-NotAuthenticated
│ Service: Identity Compartment
│ Error Message: The required information to complete authentication was not provided or was incorrect.
│ OPC request ID: [...]
│ Suggestion: Please retry or contact support for help with service: Identity Compartment
```

#### Solution

Authenticate or re-authenticate, for instance with
`oci session authenticate`.

If prompted for the profile name, make sure to enter `DEFAULT`
so that Terraform automatically uses the session token.

If you previously used `oci session authenticate`, you
should be able to refresh the session with
`oci session refresh --profile DEFAULT`.

### Capacity issue

#### Symptom

If you get a message like the following one:
```
Error: 500-InternalError
│ ...
│ Service: Core Instance
│ Error Message: Out of host capacity.
```

It means that there isn't enough servers available at the moment
on OCI to create the cluster.

#### Solution

One solution is to switch to a different *availability domain*.
This can be done by changing the `availability_domain` input variable. (Thanks @uknbr for the contribution!)

Note 1: some regions have only one availability domain. In that
case you cannot change the availability domain.

Note 2: OCI accounts (especially free accounts) are tied to a
single region, so if you get that problem and cannot change the
availability domain, you can [create another account][createaccount].

### Using the wrong region

#### Symptom

When doing `terraform apply`, you get this message:

```
oci_identity_compartment._: Creating...
╷
│ Error: 404-NotAuthorizedOrNotFound
│ Service: Identity Compartment
│ Error Message: Authorization failed or requested resource not found
│ OPC request ID: [...]
│ Suggestion: Either the resource has been deleted or service Identity Compartment need policy to access this resource. Policy reference: https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/policyreference.htm
│
│
│   with oci_identity_compartment._,
│   on main.tf line 1, in resource "oci_identity_compartment" "_":
│    1: resource "oci_identity_compartment" "_" {
│
╵
```

#### Solution

Edit `~/.oci/config` and change the `region=` line to put the correct region.

To know what's the correct region, you can try to log in to
https://cloud.oracle.com/ with your account; after logging in,
you should be redirected to an URL that looks like
https://cloud.oracle.com/?region=us-ashburn-1 and in that
example the region is `us-ashburn-1`.

### Troubleshooting cluster creation

After the VMs are created, you can log into the VMs with the
`ubuntu` user and the SSH key contained in the `id_rsa` file
that was created by Terraform.

Then you can check the cloud init output file, e.g. like this:
```
tail -n 100 -f /var/log/cloud-init-output.log
```


[ccm]: https://github.com/oracle/oci-cloud-controller-manager
[createaccount]: https://bit.ly/free-oci-dat-k8s-on-arm
[freetier]: https://www.oracle.com/cloud/free/
[kubeadm]: https://kubernetes.io/docs/reference/setup-tools/kubeadm/
[oci]: https://www.oracle.com/cloud/compute/
[oke]: https://www.oracle.com/cloud-native/container-engine-kubernetes/
