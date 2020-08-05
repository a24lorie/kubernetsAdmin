
### Description

An important consideration for any platform used to deploy production applications is observability. This Lab essentially answers how Kubernetes handles and helps you with observing everything that happens in the platform. Logging and monitoring are two pillars of observability and you will learn how to master each in this Lab. You will use what is built into Kubernetes and  `kubectl` as well as how to extend the platform to use external logging and monitoring systems. More specifically, you will use the sidecar multi-container Pod pattern to stream Pod logs to S3 using Fluentd. You will also install Metrics Server as an example of a monitoring system. All of these combined give you powerful debugging skills to diagnose and resolve issues with applications running in Kubernetes.

This Lab is valuable to anyone working with Kubernetes, but the content has been prepared considering topics described in the [Certified Kubernetes Application Developer (CKAD) Exam Curriculum](https://github.com/cncf/curriculum). Completion of the Lab will help you get hands-on experience, which is essential for passing the CKAD exam.

### Learning Objectives

-   Understand liveness probes and readiness probes
-   Understand container logging including how to use logging agents and the sidecar pattern
-   Understand how to monitor applications in Kubernetes
-   Understand debugging in Kubernetes

### Intended Audience

-   Kubernetes admins and operators
-   Application developers and DevOps engineers deploying applications in containers and using or considering Kubernetes
-   This Lab is recommended for Certified Kubernetes Application Developer (CKAD) examinees

### Prerequisites

-   Knowledge of Kubernetes Pod Design (Pods, Deployments, Services, Jobs)
-   Experience with `kubectl`

You can complete the [Kubernetes Pod Design for Application Developers Lab](https://cloudacademy.com/lab/kubernetes-pod-design-application-developers/) to satisfy the prerequisites.

**Environment before**

![PREVIEW](https://assets.cloudacademy.com/bakery/media/uploads/laboratories/environment_start_images/k8s-cluster_4zOBkqG.png)

**Environment after**

![enter image description here](https://assets.cloudacademy.com/bakery/media/uploads/laboratories/environment_end_images/after_n63VXfh.png)
