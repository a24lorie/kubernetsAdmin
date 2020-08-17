### Introduction

Nodes can fail in different ways, some may be recoverable while others are not. This Lab Step focuses on first diagnosing node failures, and then recovering a node that entered into a failure mode but can still be connected to by SSH.

In the case of hardware failures or other unrecoverable errors in a cloud environment, the appropriate remedy is to add a new node and delete the old one. The method for adding a new node varies depending on how the cluster was originally created. If  [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/) is used, then you can use the  `join`  command with the appropriate token to join the cluster. With the CloudFormation template that this Lab uses, you can terminate the failed instance, and a new one will be created automatically when health checks fail. Nodes can be removed from the cluster by using  `kubectl delete node <name_of_node>`.

### Instructions

1. List all of the nodes in the cluster using  `wide`  output to include additional columns in the output:
```
kubectl get nodes -o wide
```
All of the nodes have a  **STATUS** of **Ready**. The **Ready** status means a node is healthy and ready to accept pods. You will force a worker node into a different status to experience a type of failure. Before moving on, notice the **OS-IMAGE**  of the nodes is **Ubuntu 16**. This is the same OS used in Kubernetes certification exams.

2. Connect to one of the worker nodes using SSH, and enter  _yes_  when prompted about the host's authenticity:
```
worker_dns=$(kubectl get nodes | grep \<none\> | cut -d' ' -f1 | head -1)  
ssh $worker_dns
```
3. Check the status of the kubelet running on the worker node:
```
sudo systemctl status kubelet
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid0-462905a1-2543-4f6f-b56f-ef5ab7041622.png)

Recall that the kubelet is the primary agent running on nodes that watches for pod specs. Ubuntu 16 is a systemd-based Linux operating system. The tool used to manage services in systemd-based systems is  `systemctl`. Common systemctl commands include:

-   `status`: Show a status summary of the service including its current state (**active (running****)**), the location of the service file (**/lib/systemd/system/kubelet.service**), any drop-in files that help to configure the service (**/etc/systemd/system/kubelet.service.d/10-hostname.conf**  and **/etc/systemd/system/kubelet.service.d/10-kubeadm.conf**), the command executed to start the service (**/user/bin/kubelet --bootstrap-kubeconfig**...), and the ten most recent log messages at the bottom of the output. These bits of information can be very helpful when debugging a failed node.
-   `start`: Starts a service
-   `stop`: Stops a service
-   `enable`: Enables a service so that it is automatically started on boot. Note that enable does not automatically start a service until the system is restarted. Therefore, this command is usually followed by  `start` to start the service without rebooting.

4. Press _q_  to quit the status output view.

5. Enter the following to view all of the log messages associated with kubelet service
```
journalctl -u kubelet
```
A service is a type of unit in systemd. The  `-u`  shows messages for a specific unit, in this case the kubelet service. You can search these messages for any errors that occur and possibly put the worker node into a failed state. There are no failures in this case though, so you will simulate one.

6. Stop the kubelet running on the worker node:
```
sudo systemctl stop kubelet
```
You can confirm the service stopped by viewing the  `status`  output again. Another service you could stop to create a failure would be  `docker`, which is the container runtime used by the nodes in this Lab. When diagnosing an outage, you should check the status of both the kubelet and the container runtime.

7. Close the ssh connection to the worker to return to the bastion host's shell:
```
exit
```
8. List the status of all the nodes in the cluster:
```
kubectl get nodes
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid1-2590f862-fcda-450b-ba14-20ebecf51a37.png)

Notice that the first worker node has a **STATUS** of **NotReady**. Because you created the failure, you know exactly what is wrong. But if it was an unexpected failure, you would issue the  `systemctl status`  and  `journalctl`  commands to diagnose the problem.

9. Start the kubelet service on the failed worker node:
```
ssh $worker_dns  
sudo systemctl start kubelet  
exit
```
10. List the status of all the nodes in the cluster:
```
kubectl get nodes
```
All of the nodes should be **Ready** within 40 seconds of the kubelet starting. Nodes can also fail to accept pod requests due to resource pressure. For example, if a node is out of disk space or running low on memory. You will simulate this situation in the following instructions.

11. Connect to the worker node and restart its kubelet with an additional option to make it detect memory pressure:
```
ssh $worker_dns  
sudo sed -i 's%\(/usr/bin/kubelet\) %\1 --eviction-hard=memory.available<0.95Gi %' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf  
sudo systemctl daemon-reload  # reload the modified configuration file  
sudo systemctl restart kubelet  
exit
```
The commands add the `--eviction-hard` option to the command that starts the kubelet in `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`. The  `--eviction-hard` sets the threshold for when Kubernetes starts to detect memory pressure and is forced to start evicting pods to try to reclaim memory. Its value is 100 mebibytes (`Mi`). The node only has 1Gi of memory in total. Setting the threshold to 0.95Gi means Kubernetes will always detect memory pressure.

12. List the status of all the nodes in the cluster:
```
kubectl get nodes
```
All of the nodes still have a **STATUS**  of **Ready**. However, if you were scheduling pods, you may notice that one worker is not accepting any new pods and is evicting existing pods. This requires more details to diagnose than the  `get`  command provides.

13. Describe the worker node with the increased hard eviction threshold:
```
kubectl describe node $worker_dns | more
```
Focus on the **Conditions**  section near the top of the output:

![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid1-0031d622-2730-4e8f-a80f-fa7a78daba2f.png)

You will notice that the node is reporting memory pressure (**MemoryPressure**). The node is still **Ready**  because it is listening for pod requests, although it typically would not accept any new requests because it does not have enough memory available. If the memory shortage was due to pods running on the node, Kubernetes could remedy the situation itself by evicting the pods. However, if the node had processes running outside of Kubernetes that were consuming the memory, you would need to connect to the node and stop processes to free up memory for Kubernetes. You would take similar action if the node was reporting it was out of disk space (**OutOfDisk**), disk space is running low (**DiskPressure**), or there are too many processes running on the node (**PIDPressure**). You may want to  `drain`  or  `cordon`  the node to prevent pods from being scheduled on the node until you are sure the issue is resolved. When you are ready, you would  `uncordon`  the node to allow pods to be scheduled on the node again.

14. Press _spacebar_  until you have paged through all of the  `describe`  output, and the shell prompt is returned to you.

At the end of the output in the **Events**  section, which includes the most recent events at the end, you will see the following two events related to the memory pressure:

![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid0-23fd5d99-a727-4c57-b148-a688aef14875.png)

15. Restart the worker node's kubelet without the hard eviction limit option to repair the node:
```
ssh $worker_dns  
sudo sed -i 's%--eviction-hard=memory.available<0.95Gi %%' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf  
sudo systemctl daemon-reload  
sudo systemctl restart kubelet  
exit
```
16. Confirm that the node no longer reports memory pressure:
```
kubectl describe node $worker_dns
```
![alt](https://assets.cloudacademy.com/bakery/media/uploads/blobid0-c2951048-f739-4745-8e71-a7f7b3efcd7a.png)

### Summary

In this Lab Step, you learned how to detect and resolve several node-level issues that can cause failures or have an undesirable impact on the performance of your Kubernetes cluster.
