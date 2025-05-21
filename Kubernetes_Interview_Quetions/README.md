# Kubernetes Interview Questions


### _Questions_
1. What is a Pod in Kubernetes, and why is it considered the smallest deployable unit?
   * In Kubernetes, a Pod is the smallest deployable unit that you can create and manage. It represents a single instance of a running process in your cluster.
   * Why is a Pod the smallest deployable unit?
        * A Pod can contain one or more containers, which are tightly coupled and share the same network namespace. This means they can communicate with each other using `localhost` and share storage volumes.
        * Pods are designed to run a single instance of a service or application, making them the smallest deployable unit in Kubernetes.
2. Can a Pod contain more than one container? When would you use this?
   * Yes, a Pod can contain more than one container, and this is called a multi-container Pod.
   * When to Use Multiple Containers in a Pod
        * You use multiple containers in a Pod only when they are tightly coupled and need to:
            * Share the same network and volumes. 
            * Work together closely to fulfill a single responsibility. 
            * Be co-scheduled and always run together.
        * Common Use Cases
            * Sidecar Pattern: A sidecar container can provide additional functionality to the main container, such as logging, monitoring, or proxying.
            * Proxy Pattern: A helper container acts as a proxy or handles authentication, security, or network routing.
3. What happens when a Pod crashes?
    * When a Pod crashes, Kubernetes will attempt to restart it based on the restart policy defined in the Pod specification.
    * The restart policy can be one of the following:
          * Always: The Pod will always be restarted regardless of the exit status.
          * OnFailure: The Pod will be restarted only if it exits with a non-zero status.
          * Never: The Pod will not be restarted.
    * If the Pod continues to crash, Kubernetes may mark it as "CrashLoopBackOff," indicating that it is repeatedly failing to start.
4. How does Kubernetes know when to restart a Pod?
    * Kubernetes uses a combination of health checks (liveness and readiness probes) and the restart policy defined in the Pod specification to determine when to restart a Pod.
    * Liveness Probe: A liveness probe checks if the container is still running. If it fails, Kubernetes will restart the container.
    * Readiness Probe: A readiness probe checks if the container is ready to accept traffic. If it fails, Kubernetes will stop sending traffic to that Pod until it passes again.
5. What is a Deployment in Kubernetes?
    * A Deployment in Kubernetes is a higher-level abstraction that manages the lifecycle of Pods. It provides declarative updates to applications and ensures that the desired state of the application is maintained.
    * Key Features of Deployments:
          * Declarative Updates: You can define the desired state of your application, and Kubernetes will automatically manage the changes.
          * Rollbacks: If a new version of an application fails, you can easily roll back to a previous version.
          * Scaling: You can scale the number of replicas (Pods) up or down as needed.
6. What is the difference between a ReplicaSet and a Deployment?
    * A ReplicaSet is a low-level controller that ensures a specified number of Pod replicas are running at any given time. It is responsible for maintaining the desired state of the Pods.
    * A Deployment is a higher-level abstraction that manages ReplicaSets and provides additional features such as rolling updates, rollbacks, and scaling.
    * In summary, a Deployment uses ReplicaSets to manage the Pods, while a ReplicaSet is responsible for ensuring the desired number of Pods are running.
7. How does a Service work in Kubernetes?
    * A Service in Kubernetes is an abstraction that defines a logical set of Pods and a policy to access them. It provides a stable endpoint (IP address and DNS name) for accessing the Pods, regardless of their individual IP addresses.
    * Key Features of Services:
          * Load Balancing: Services can distribute traffic across multiple Pods, providing load balancing.
          * Service Discovery: Services can be discovered using DNS names, making it easy for other components to find and communicate with them.
          * Types of Services: There are several types of Services, including ClusterIP (default), NodePort, LoadBalancer, and ExternalName.
8. What is the default type of Kubernetes Service?
    * The default type of Kubernetes Service is ClusterIP. This type of Service provides a stable IP address that is only accessible within the cluster. It is used for internal communication between Pods.
9. Explain the difference between kubectl apply and kubectl create?
    * kubectl create: This command is used to create a new resource in Kubernetes. If the resource already exists, it will return an error.
    * kubectl apply: This command is used to create or update a resource in Kubernetes. If the resource already exists, it will update it to match the desired state defined in the configuration file. If it does not exist, it will create it.
    * When to Use What?
        * Use kubectl apply when:
            * You're managing resources declaratively (with YAML). 
            * You're using version control, GitOps, or CI/CD pipelines. 
            * You want to update existing resources safely.
        * Use kubectl create when:
            * You’re creating a resource for the first time. 
            * You don’t plan to update it using YAML afterward.
10. What is a YAML manifest in Kubernetes?
    * A YAML manifest in Kubernetes is a configuration file that defines the desired state of a Kubernetes resource. It is written in YAML format and contains information such as the resource type, metadata, specifications, and desired state.
    * YAML manifests are used with kubectl commands to create, update, or delete resources in the Kubernetes cluster.

### questions on Kubernetes (use cases)

1. Your team is running multiple instances of a REST API. How do you expose them with load balancing?
   * To expose multiple instances of a REST API with load balancing in Kubernetes, you can use a combination of: 
       * Pods (running your API instances) 
       * Service of type LoadBalancer or ClusterIP.
       * ingress for HTTP routing and domain-based routing.
   * How Load Balancing Works
       * The Service acts like a virtual IP with load balancing. 
       * It routes each request to one of the Pods using kube-proxy, usually via round-robin. 
       * If a Pod crashes or is removed, the Service automatically updates its routing.
     
2. You have a model training job and a logging agent that should run in sync. How do you architect this using Pods?
    * To run a model training job and a logging agent in sync, the ideal architecture is to use a multi-container Pod in Kubernetes. This allows both containers to share the same network namespace and storage volumes, ensuring they can communicate and share data easily.

3. What’s the best way to roll out a new version of your app without downtime?
    * The best way to roll out a new version of your app without downtime is to use a rolling update strategy with a Deployment in Kubernetes. This allows you to gradually replace old Pods with new ones while ensuring that a minimum number of Pods are always available to serve traffic.
    * Steps for Rolling Update:
        1. Update the Deployment manifest with the new image version.
        2. Use kubectl apply to apply the changes.
        3. Kubernetes will create new Pods with the updated image and gradually terminate the old Pods.
        4. Monitor the rollout status using kubectl rollout status deployment/<deployment-name>.
        5. If any issues arise, you can easily roll back to the previous version using kubectl rollout undo deployment/<deployment-name>.

4. A new app version is causing failures. How do you roll back using Kubernetes?
   * To roll back to a previous version of an application in Kubernetes, you can use the kubectl rollout undo command. This command allows you to revert to the last successful deployment or a specific revision.
   
5. How would you scale your app during a peak load?
   * To scale your app during a peak load in Kubernetes, you can use a combination of horizontal scaling, resource limits, and autoscaling mechanisms to dynamically adjust to demand without manual intervention or downtime.
   * Horizontal Pod Autoscaler (HPA): he Horizontal Pod Autoscaler (HPA) automatically adjusts the number of Pods in a Deployment, ReplicaSet, or StatefulSet based on CPU, memory, or custom metrics.
   * If average CPU > 70%, HPA adds Pods (up to 10).
   * If a load drops, it scales down (but not below 3).
     ```yaml
       apiVersion: autoscaling/v2
       kind: HorizontalPodAutoscaler
       metadata:
         name: my-app-hpa
       spec:
         scaleTargetRef:
           apiVersion: apps/v1
           kind: Deployment
           name: my-app
       minReplicas: 3
       maxReplicas: 10
       metrics:
          - type: Resource
            resource:
              name: cpu
              target:
                 type: Utilization
                 averageUtilization: 70
   ```
6. Your Pod isn’t being recreated after failure. What might be wrong?
    * If your Pod isn't being recreated after a failure, there are several potential reasons:
         * Check the Pod's restart policy. If it's set to "Never," the Pod won't be restarted after failure.
         * Check if the Pod is part of a ReplicaSet or Deployment. If not, it won't be automatically recreated.
         * Check if the node where the Pod was running is down. If the node is unreachable, Kubernetes may not be able to recreate the Pod.
         * Check for resource limits. If the Pod exceeds its resource limits (CPU/memory), it may be terminated and not restarted.
         * Check for any errors in the event logs using kubectl describe pod <pod-name> to see if there are any specific issues causing the failure.
      
7. A single Pod works, but when you scale, traffic doesn’t route correctly. What’s your first debug step?
    * If a single Pod works but traffic doesn't route correctly when scaled, the first debug step is to check the Service configuration. Ensure that the Service is correctly set up to route traffic to all Pods in the ReplicaSet or Deployment.
    * Summary of Debug Flow
        * Check Pod Readiness: kubectl get pods, kubectl describe pod 
        * Check Service Endpoints: kubectl get endpoints <svc>
        * Check Service Config: kubectl describe service <svc>
        * Test Internal Connectivity (e.g., with busybox pod) 

8. What is a headless Service and when should you use one?
    * A headless Service in Kubernetes is a Service without a ClusterIP. It allows you to directly access the individual Pods without load balancing or a stable IP address.
    * Use Cases for Headless Services:
        * Stateful applications: When you need to maintain stable network identities for each Pod (e.g., databases).
        * Service discovery: When you want to use DNS to discover individual Pods instead of a single IP.
        * Custom load balancing: When you want to implement your own load balancing logic at the application level.
9. what is a StatefulSet and when would you use it?
    * A StatefulSet in Kubernetes is a controller that manages the deployment and scaling of a set of Pods, providing guarantees about the ordering and uniqueness of these Pods.
    * Use Cases for StatefulSets:
        * Stateful applications: When you need to maintain stable network identities and persistent storage for each Pod (e.g., databases, message queues).
        * Ordered deployment: When you need to ensure that Pods are started or terminated in a specific order.
        * Unique identities: When each Pod needs a unique identity (e.g., hostname) that persists across rescheduling.
10. Describe a use case for initContainers.
    * Init containers are specialized containers that run before the main application containers in a Pod. They are used to perform initialization tasks that must be completed before the main application starts.
    * Use Cases for Init Containers:
        * Preloading data: Downloading or preloading data required by the main application.
        * Configuration: Generating configuration files or secrets needed by the main application.
        * Dependency checks: Ensuring that certain conditions are met (e.g., waiting for a database to be ready) before starting the main application.

### Real-World Ops (Experienced) 
1. How do you handle zero-downtime deployments in production?
    * To handle zero-downtime deployments in production, you can use a combination of rolling updates, readiness probes, and traffic management techniques.
    * Steps for Zero-Downtime Deployment:
          1. Use a Deployment with a rolling update strategy to gradually replace old Pods with new ones.
          2. Configure readiness probes to ensure that new Pods are only added to the Service when they are ready to accept traffic.
          3. Use a Service mesh (e.g., Istio) or ingress controller to manage traffic routing and canary releases.
          4. Monitor the deployment process and rollback if any issues arise.
2. How do you monitor and alert on Pod health?
    * To monitor and alert on Pod health in Kubernetes, you can use a combination of built-in Kubernetes features and external monitoring tools.
    * Steps for Monitoring Pod Health:
          1. Use liveness and readiness probes to check the health of your Pods.
          2. Use Kubernetes events and logs to track Pod status changes and errors.
          3. Integrate with monitoring tools like Prometheus, Grafana, or ELK stack to collect metrics and logs.
          4. Set up alerts based on specific conditions (e.g., Pod restarts, high CPU/memory usage) using tools like Alertmanager or PagerDuty.
3. What are readiness and liveness probes, and how do they differ in effect?
    * Readiness and liveness probes in Kubernetes are health checks that help the platform manage the lifecycle and traffic routing of containers.
    * Readiness Probe:
        * Check if a container is ready to accept traffic.
        * If it fails, the Pod is removed from the Service endpoints, and traffic is not sent to it.
        * Used to ensure that only healthy Pods receive traffic.
    * Liveness Probe:
        * Check if a container is still running and healthy.
        * If it fails, Kubernetes will restart the container.
        * Used to ensure that failed or unresponsive containers are automatically restarted.
4. You applied a Deployment with 3 replicas, but only 1 Pod is running. Why?
    * If you applied a Deployment with 3 replicas but only 1 Pod is running, there could be several reasons:
        * Insufficient resources: The cluster may not have enough resources (CPU/memory) to schedule the additional Pods.
        * Scheduling issues: There may be node affinity or taints that prevent the Pods from being scheduled on available nodes.
        * CrashLoopBackOff: The other Pods may be crashing due to application errors or misconfigurations, causing Kubernetes to keep restarting them.
        * Check the events and logs using kubectl describe deployment <deployment-name> and kubectl get pods -o wide to diagnose the issue.
5. A Pod is stuck in CrashLoopBackOff. How do you troubleshoot it?
    * To troubleshoot a Pod stuck in CrashLoopBackOff, you can follow these steps:
        1. Check the Pod logs using kubectl logs <pod-name> to see if there are any error messages or stack traces.
        2. Describe the Pod using kubectl describe pod <pod-name> to check for events, status, and resource limits.
        3. Check the container's exit code using kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].state.terminated.exitCode}' to determine if it exited with an error.
        4. If the Pod is part of a Deployment or ReplicaSet, check the rollout status using kubectl rollout status deployment/<deployment-name>.
        5. If necessary, use kubectl exec to access the container and debug further.