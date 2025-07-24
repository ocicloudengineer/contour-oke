# Demo: Contour on OKE with an Internal Load Balancer using kubectl

This guide provides a complete, step-by-step demo for installing Contour in OKE with an internal load balancer, setting it as the default `IngressClass`, and deploying a sample blue/green Nginx application, all using `kubectl`.

***
### ## Prerequisites

* An active OKE cluster.
* `kubectl` configured to communicate with your cluster.
* The **OCID of the private subnet** where the internal load balancer will be created.

***
### ## Step 1: Prepare the Contour Installation Manifest

First, download the official manifest, then modify it to include the OCI annotations for an internal load balancer.

1.  Download the manifest file:
    ```bash
    curl -o contour-oke.yaml [https://projectcontour.io/quickstart/contour.yaml](https://projectcontour.io/quickstart/contour.yaml)
    ```

2.  Open the downloaded `contour-oke.yaml` file in a text editor. Find the `Service` resource named `envoy` and add the `annotations` block to its metadata. The final resource should look like this:

    ```yaml
    # Find this Service definition in your contour-oke.yaml file
    apiVersion: v1
    kind: Service
    metadata:
      name: envoy
      namespace: projectcontour
      annotations:
        # --- ADD THIS ANNOTATIONS BLOCK ---
        oci.oraclecloud.com/load-balancer-type: "lb"
        service.beta.kubernetes.io/oci-load-balancer-internal: "true"
        service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
        service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "100"
        service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "400"
        # Replace with the OCID of your private subnet
        service.beta.kubernetes.io/oci-load-balancer-subnet1: "ocid1.subnet.oc1.sa-bogota-1.aaaaaaaa3bxqw5xjoxhuixkdj5swfl3ysmre2aiqiwt4bajwxsclts64eveq"
        # --- END OF ADDED BLOCK ---
    spec:
      selector:
        app: envoy
      ports:
      - name: http
        port: 80
        protocol: TCP
        targetPort: 8080
      type: LoadBalancer

  For this example I removed the 443 port.

    ```

***
### ## Step 2: Install Contour and Verify

Apply the manifest to your cluster and verify that the resources are created correctly.

1.  Install Contour using your modified file:
    ```bash
    kubectl apply -f contour-oke.yaml
    ```
    *Expected Result:*
    ```
    namespace/projectcontour created
    customresourcedefinition.apiextensions.k8s.io/contourconfigurations.projectcontour.io created
    customresourcedefinition.apiextensions.k8s.io/contourdeployments.projectcontour.io created
    customresourcedefinition.apiextensions.k8s.io/extensionservices.projectcontour.io created
    customresourcedefinition.apiextensions.k8s.io/httpproxies.projectcontour.io created
    ...and several other resources
    ```

2.  Verify the pods are running:
    ```bash
    kubectl get pods -n projectcontour
    ```
    *Expected Result:*
    ```
    NAME                       READY   STATUS    RESTARTS   AGE
    contour-7f99c65b89-abcde   1/1     Running   0          2m
    contour-7f99c65b89-fghij   1/1     Running   0          2m
    envoy-klmno                2/2     Running   0          2m
    ```

3.  Verify the service receives an internal IP address (this may take a minute or two):
    ```bash
    kubectl get service -n projectcontour contour-envoy
    ```
    *Expected Result:*
    ```
    NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    contour-envoy   LoadBalancer   10.96.123.45    10.0.20.155   80:31283/TCP,443:32135/TCP   2m
    ```

***
### ## Step 3: Create and Configure the Default `IngressClass`

This resource registers Contour as the default Ingress controller for the cluster.

1.  Create a file named `contour-class.yaml`:
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: IngressClass
    metadata:
      name: contour
      annotations:
        ingressclass.kubernetes.io/is-default-class: "true"
    spec:
      controller: projectcontour.io/ingress-controller
    ```
2.  Apply it:
    ```bash
    kubectl apply -f contour-class.yaml
    ```
    *Expected Result:*
    ```
    ingressclass.networking.k8s.io/contour created
    ```

***
### ## Step 4: Deploy the Sample Blue/Green Applications

This manifest creates "blue" and "green" Nginx deployments and services, each with a custom welcome page.

1.  Create a file named `nginx-blue-green.yaml`:
    ```yaml
    # Blue Application ConfigMap and Deployment
    apiVersion: v1
    kind: ConfigMap
    metadata: { name: blue-html }
    data:
      index.html: '<html><body style="background-color:#337ab7;color:white;text-align:center;font-family:sans-serif"><h1>Welcome to the BLUE Nginx instance!</h1></body></html>'
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata: { name: nginx-blue-deployment }
    spec:
      replicas: 1
      selector: { matchLabels: { app: nginx-blue } }
      template:
        metadata: { labels: { app: nginx-blue } }
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports: [{ containerPort: 80 }]
            volumeMounts: [{ name: html, mountPath: /usr/share/nginx/html }]
          volumes: [{ name: html, configMap: { name: blue-html } }]
    ---
    apiVersion: v1
    kind: Service
    metadata: { name: nginx-blue-service }
    spec:
      ports: [{ port: 80 }]
      selector: { app: nginx-blue }
    ---
    # Green Application ConfigMap and Deployment
    apiVersion: v1
    kind: ConfigMap
    metadata: { name: green-html }
    data:
      index.html: '<html><body style="background-color:#5cb85c;color:white;text-align:center;font-family:sans-serif"><h1>Welcome to the GREEN Nginx instance!</h1></body></html>'
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata: { name: nginx-green-deployment }
    spec:
      replicas: 1
      selector: { matchLabels: { app: nginx-green } }
      template:
        metadata: { labels: { app: nginx-green } }
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports: [{ containerPort: 80 }]
            volumeMounts: [{ name: html, mountPath: /usr/share/nginx/html }]
          volumes: [{ name: html, configMap: { name: green-html } }]
    ---
    apiVersion: v1
    kind: Service
    metadata: { name: nginx-green-service }
    spec:
      ports: [{ port: 80 }]
      selector: { app: nginx-green }
    ```
2.  Apply it:
    ```bash
    kubectl apply -f nginx-blue-green.yaml
    ```
    *Expected Result:*
    ```
    configmap/blue-html created
    deployment.apps/nginx-blue-deployment created
    service/nginx-blue-service created
    configmap/green-html created
    deployment.apps/nginx-green-deployment created
    service/nginx-green-service created
    ```

***
### ## Step 5: Create the `Ingress` Resource

This `Ingress` manifest routes traffic to your blue and green services.

1.  Create a file named `blue-green-ingress.yaml`:
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: cas-ms-api-docs-ing
    spec:
      ingressClassName: contour
      rules:
      - http:
          paths:
          - path: /blue
            pathType: Prefix
            backend:
              service:
                name: nginx-blue-service
                port:
                  number: 80
          - path: /green
            pathType: Prefix
            backend:
              service:
                name: nginx-green-service
                port:
                  number: 80
    ```
2.  Apply it:
    ```bash
    kubectl apply -f blue-green-ingress.yaml
    ```
    *Expected Result:*
    ```
    ingress.networking.k8s.io/cas-ms-api-docs-ing created
    ```

***
### ## Step 6: Test the Blue/Green Routes ✅

Since the load balancer is internal, you must test from a machine **inside the same VCN**.

1.  Get the internal IP of your load balancer:
    ```bash
    kubectl get service -n projectcontour contour-envoy -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
    ```
    *Expected Result:*
    ```
    10.0.20.155
    ```
2.  Test the `/blue` and `/green` paths, using the IP from the previous command.
    ```bash
    curl [http://10.0.20.155/blue](http://10.0.20.155/blue)
    ```
    *Expected Result:*
    ```html
    <html><body style="background-color:#337ab7;color:white;text-align:center;font-family:sans-serif"><h1>Welcome to the BLUE Nginx instance!</h1></body></html>
    ```
    ```bash
    curl [http://10.0.20.155/green](http://10.0.20.155/green)
    ```
    *Expected Result:*
    ```html
    <html><body style="background-color:#5cb85c;color:white;text-align:center;font-family:sans-serif"><h1>Welcome to the GREEN Nginx instance!</h1></body></html>
    ```
