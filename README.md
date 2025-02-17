# Kubernetes Startup Probe Lab

This lab demonstrates the use of Kubernetes `startupProbe` to manage the startup behavior of applications that take a long time to initialize.  We will deploy a Pod with a `startupProbe` that checks for the existence of a marker file, and a `livenessProbe` that checks the application's health endpoint.

## Prerequisites

*   A Kubernetes cluster (e.g., minikube, kind, or a cloud provider's Kubernetes service)
*   `kubectl` command-line tool configured to connect to your cluster

## Lab Steps

1.  **Create the `pod.yaml` file:**

    Create a file named `pod.yaml` with the following content:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-with-startup-check
    spec:
      containers:
      - image: quay.io/wildfly/wildfly
        name: wildfly
        startupProbe:
          exec:
            command: [ "stat", "/opt/jboss/wildfly/standalone/tmp/startup-marker" ]
          initialDelaySeconds: 60
          periodSeconds: 60
          failureThreshold: 15
        livenessProbe:
          httpGet:
            path: /health
            port: 9990
          periodSeconds: 10
          failureThreshold: 3
    ```

    **Explanation:**

    *   `apiVersion: v1`: Specifies the Kubernetes API version.
    *   `kind: Pod`: Defines a Pod resource.
    *   `metadata.name`: Sets the name of the Pod to `pod-with-startup-check`.
    *   `spec.containers`: Defines the container within the Pod.
        *   `image: quay.io/wildfly/wildfly`: Uses the `quay.io/wildfly/wildfly` Docker image, which is an application server that can take some time to start.
        *   `name: wildfly`: Sets the name of the container.
        *   `startupProbe`: Configures the `startupProbe`.
            *   `exec.command: [ "stat", "/opt/jboss/wildfly/standalone/tmp/startup-marker" ]`: Executes the `stat` command to check for the existence of the file `/opt/jboss/wildfly/standalone/tmp/startup-marker`. This file is expected to be created by the WildFly application server when it has successfully started.
            *   `initialDelaySeconds: 60`:  The probe will start after 60 seconds of container start.
            *   `periodSeconds: 60`: The probe will be executed every 60 seconds.
            *   `failureThreshold: 15`: If the probe fails 15 times, Kubernetes will restart the container.  This allows WildFly considerable time to start up completely.
        *   `livenessProbe`: Configures the `livenessProbe`.
            *   `httpGet.path: /health`:  Performs an HTTP GET request to the `/health` endpoint.
            *   `httpGet.port: 9990`: Specifies the port to use for the HTTP GET request.
            *   `periodSeconds: 10`: The probe will be executed every 10 seconds.
            *   `failureThreshold: 3`: If the probe fails 3 times, Kubernetes will restart the container.

2.  **Deploy the Pod:**

    Use `kubectl` to create the Pod from the `pod.yaml` file:

    ```bash
    kubectl apply -f pod.yaml
    ```

3.  **Verify the Pod Status:**

    Check the status of the Pod:

    ```bash
    kubectl get pod pod-with-startup-check
    ```

    Initially, the Pod will be in a state other than `Running` until the `startupProbe` reports success. After the startup probe determines the wildfly instance is running, the pod should transition to the `Running` state.

4.  **Inspect the Pod Details:**

    Use the `describe` command to view the startup and liveness probe configurations:

    ```bash
    kubectl describe pod pod-with-startup-check
    ```

    This will show the details of the probes, including the command, ports, paths, and timing parameters.  Pay attention to the `Liveness` and `Startup` sections.

## Understanding the Behavior

*   **Startup Probe First:** The `startupProbe` runs first, and the `livenessProbe` (and `readinessProbe` if configured) only start running after the `startupProbe` reports success.
*   **Long Startup Times:**  The `startupProbe` is designed for applications that have lengthy startup times. It prevents the `livenessProbe` from prematurely killing the container during initialization.
*   **Failure Threshold:**  The `failureThreshold` is crucial.  If the `startupProbe` fails more times than the `failureThreshold` allows, the container will be restarted.
*   **Probe Actions:**  The `startupProbe` can use any of the probe actions (Exec, HTTPGet, TCPSocket) that are available for liveness and readiness probes.  The most appropriate action depends on how you can determine if your application has successfully started.

## Key Takeaways

*   `startupProbe` is essential for applications that take a long time to start.
*   It prevents premature restarts by allowing the application time to initialize.
*   The `failureThreshold`, `initialDelaySeconds` and `periodSeconds` parameters should be carefully configured based on the application's startup characteristics.
*   After the startup probe is successful, liveness and readiness probes take over.
*   Startup probes are an important part of maintaining the availability and reliability of your Kubernetes deployments.