### Introduction

In this Lab Step, you will troubleshoot issues that relate to communicating with and accessing a Kubernetes cluster. The bastion host has  `kubectl`  installed,but has not been configured correctly to talk to the cluster. You will learn how to diagnose and resolve this problem and more related to accessing a cluster.

### Instructions

1. For convenience, enable  `kubectl`  autocompletion:
```
echo "source <(kubectl completion bash)" >> ~/.bashrc  
source ~/.bashrc
```
The first command adds the  `completion`  command to your `.bashrc`  file so it is executed on every login. The second command runs the commands in the  `.bashrc`  file so that completions are enabled for the current shell.

2. Attempt to list the nodes in the cluster:
```
kubectl get nodes
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid0-ac1fda23-3c5f-4fbb-8ed1-e6dc6b9641ce.png)

The result is a  **connection refused**  message. By default,  `kubectl`  will attempt to connect to the Kubernetes API server on port 8080 of the local host. The bastion host does not have an API server so there is nothing to connect to. You need to configure kubectl to connect to the API server running on the master node.  `kubectl` is configured to connect to clusters by using kubeconfig files. By default,  `kubectl`  looks for a kubeconfig file  `~/.kube/config`. This file does not exist on the bastion host. You can copy the kubeconfig file on the master to get it working on the bastion host.

3. Enter the following commands to copy the master node's kubeconfig file, and enter _yes_  when prompted about the authenticity of the host:
```
mkdir .kube  # create the .kube directory  
master_ip=$(aws ec2 describe-instances --region us-west-2 \  
 --filters "Name=tag:Name,Values=k8s-master" \  
 --query "Reservations[*].Instances[*].PrivateIpAddress" \  
 --output text)  # get the master's IP address using the AWS CLI  
scp -o "ForwardAgent yes" $master_ip:.kube/config .kube/config  # secure copy (scp) the kubeconfig file
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid0-d4f5b6f5-60c8-477a-ab56-f6c722fda302.png)

The kubeconfig file that you copied from the master node includes:

1.  Information about the cluster, such as the server address
2.  Information about users to authenticate as including certificates that were generated when the cluster was created

4. View the contents of the kubeconfig file:
```
cat ~/.kube/config
```
Most of the contents are certificate and key data, but you can notice that there are several top-level keys including  **clusters**,  **contexts**,  **current-context**, and  **users**. The following image highlights the **contexts**  and **current-context**  keys:

![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid0-bac2320c-57ad-4e06-9ebe-e1d6a59aac76.png)

A context is a triple of a **cluster**  (**kubernetes**), a  **user**  (**kubernetes-admin**), and a namespace (if not specified, the default namespace is used). The context is also given a name for reference (**kubernetes-admin@kubernetes**). The  **current-context**  sets the context that will be used by  `kubectl`  by default. To manage the configuration of  `kubectl`, you can use its  `config`  command.

5. View the  `config`  commands provided by  `kubectl`:
```
kubectl config --help
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid1-62f39b9c-e9ff-4689-8e06-253bc1b730cc.png)

These commands can be used to safely write to kubeconfig files using the  **delete**, **set**, **unset**, and **use** commands. Assuming the configuration file has the correct cluster and user information, you may have issues connecting to a cluster because the current context is not configured at all; it is set to a different context than you expect.

6. Enter the following  `config`  command to get a summary of the contexts available in a kubeconfig file:
```
kubectl config get-contexts
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid2-ea5ec08d-254d-4d83-8820-7256505de8ad.png)

This view is useful for showing you all of the configured contexts and which is the **CURRENT**  context. If there is no current context set, there will be no *****  in the first column. In this Lab, there is only one context, but you could have several contexts in practice. There are also multiple contexts for different clusters in the Kuberenetes certification exams. If the current context is not what you want, you can use the  `use-context`  command to change it.

7. Re-attempt to list the nodes in the cluster:
```
kubectl get nodes
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid4-f4a8875d-0ee7-4451-ae84-f31df23742ba.png)

With the kubeconfig file in the default location of ~/.kube/config and the correct current-context defined,  `kubectl`  is able to communicate with the cluster. If you received a connection refused or timeout error, it would most likely be due to firewall or cluster node and component issues that you will investigate in the following Lab Steps. If you received a permission denied error, you would need to consider using another user for the context or modify the role-based access controls (RBAC) for the user. It is instructive to see why the kubernetes-admin user has access to list the nodes.

8. Confirm the kubernetes-admin user is authorized to list nodes:
```
kubectl auth can-i list nodes
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid0-fe0da7a3-6ef1-42f7-bb4f-848b41de400e.png)

The  `can-i`  command will return a binary response for whether or not a user is authorized to perform a specified action. You can learn more about the actions from the command's help page (`kubectl auth can-i --help`). The kubernetes-admin user is authorized because it is bound to an RBAC role that grants permission. Kubernetes does not store any user resources. Instead, role binding resources store information about the names of users or groups that are assigned to a role. Roles can be for a specific namespace (Role) or for an entire cluster (ClusterRole). Similarly, role bindings can be for a specific namespace (RoleBinding) or for an entire cluster (ClusterRoleBinding).

9. List all of the cluster roles:
```
kubectl get clusterrolebindings
```
Kubernetes creates several cluster roles by default. The one that is germane to this exercise is the **cluster-admin**  role.

10. Show the cluster-admin cluster role resource YAML:
```
kubectl get clusterrole cluster-admin -o yaml
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid0-ae5ad3c1-b245-4278-9254-9a3908011f09.png)

The  **rules**  key allows all actions (**verbs**) on all  **resources**. You can get the same information from  `kubectl describe clsuterrole cluster-admin`, but it is useful to use to  `-o yaml`  option on  `get`  commands to quickly create templates from existing resources. This is very useful when you need to create new resources and cannot remember which  **apiVersion**  to use or what the names of certain keys are.

11. Describe the cluster-admin cluster role binding to understand how the cluster-admin role is bound to users:
```
kubectl describe clusterrolebinding cluster-admin
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid1-c0ffac06-ccf7-4490-90fd-707bb012791d.png)

The **Role**  map specifies the  **Name**  of the role that is being bound, and the **Subjects** map lists all the subjects (users, groups, or service accounts) that are bound to the role. In this case, the role is bound to a **Group** named **system:masters**. Because identities are managed outside of Kubernetes, you cannot use kubectl to show that the kubernetes-admin user is a member of the  **system:masters**  group. For completeness, the following instruction will show you how to verify the group membership.

12. Extract the kubernetes-admin certificate from the kubeconfig file, and use OpenSSL to show the certificate details:
```
grep "client-cert" ~/.kube/config | \  
  sed 's/\(.*client-certificate-data: \)\(.*\)/\2/' | \  
  base64 --decode \  
  > cert.pem  
openssl x509 -in cert.pem -text -noout
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid2-91ee4358-776e-4b50-80cf-397f3e41eb04.png)

The first **Subject**  shows the **system:masters**  organization (**O**) and the  **kubernetes-admin**  common name. Kubernetes maps the common name to users and the organization to groups. You now understand the full authorization chain:

1.  The context uses a kubernetes-admin client certificate
2.  The certificate declares the kubernetes admin user as a member of the system:masters organization, which is treated as a group in Kubernetes
3.  There is a cluster role binding between the system:masters group and the cluster-admin cluster role
4.  The cluster-admin cluster role grants all actions on all resources

### Summary

In this Lab Step, you learned how to diagnose and resolve issues related to communicating with the desired Kuberentes cluster using  `kubectl`. You understood the role of kubeconfig files, contexts, and  `config`  commands to ensure that  `kubectl`  is properly configured to communicate with the target cluster. You also reviewed roles and role bindings, and how they allow the admin user to perform any action on the cluster.
