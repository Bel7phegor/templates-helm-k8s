# Helm Chart: (template for k8s)

This Helm chart provides a standardized, reusable blueprint for packaging, configuring, and deploying the Shopnow Frontend application onto Kubernetes clusters, specifically optimized for AWS Elastic Kubernetes Service (EKS).

The architecture isolates application logic from infrastructure configurations, allowing runtime components (Deployment, Service, Ingress, HPA, ConfigMap, ServiceAccount, and Secrets) to be dynamically managed and toggled via `values.yaml`.

---

## Directory Structure

```text
.
├── .helmignore
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── NOTES.txt
    ├── _helpers.tpl
    ├── configmap.yaml
    ├── deployment.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── secrets.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    └── tests/
        └── test-connection.yaml
```

---

## Resource Architecture and Components

### 1. Deployment (`deployment.yaml`)
* **Lifecycle Management:** Manages the pod specifications and desired replica counts for the application container.
* **Update Strategy:** Employs a `RollingUpdate` mechanism utilizing `maxSurge` and `maxUnavailable` parameters from configurations to guarantee zero-downtime rollouts.
* **Volume Mounts:** Natively mounts application runtime properties from the generated ConfigMap directly into `/app/src/main/resources/application.properties` utilizing a secure `subPath` isolation layer.
* **Secret Injection:** Automatically maps and injects sensitive environment variables into the container via `envFrom` when the secret component flag is enabled.

### 2. Service (`service.yaml`)
* **Internal Networking:** Exposes the underlying application pods internally within the cluster via a stable virtual IP layer.
* **Traffic Routing:** Maps incoming cluster traffic from a designated service port to the explicit `targetPort` of the containers using the `app: <global.appName>` label selector.

### 3. Ingress (`ingress.yaml`)
* **External Edge Routing:** Manages external ingress traffic routing configurations, integrating with the AWS Load Balancer Controller (ALB).
* **AWS ALB Integration Annotations:**
  * `alb.ingress.kubernetes.io/scheme: internet-facing`: Provisions a public-facing Application Load Balancer.
  * `alb.ingress.kubernetes.io/target-type: ip`: Routes traffic directly from the ALB to individual pod IP addresses, bypassing NodePort latencies (optimized for Amazon VPC CNI).
  * Automatically provisions AWS Certificate Manager (ACM) SSL certificates, configures listener ports for HTTP/HTTPS, and enforces SSL redirection rules when `ingress.acmArn` is provided.

### 4. Horizontal Pod Autoscaler (`hpa.yaml`)
* **Workload Elasticity:** Implements automatic horizontal scaling mechanisms utilizing the Kubernetes `autoscaling/v2` API.
* **Target Metric Evaluation:** Dynamically adjusts running replica sets between minimum and maximum boundaries based on defined target utilization percentages for both CPU and Memory resources.

### 5. ConfigMap and Secrets (`configmap.yaml` / `secrets.yaml`)
* **Decoupled Architecture:** Segregates application environments, credentials, and static configurations away from the core container image layers.
* **Data Binding:** ConfigMap parses configuration entries from `configmap.data` directly into configuration files. The Secrets manifest securely parses key-value parameters via `stringData` into designated cluster components.

### 6. ServiceAccount (`serviceaccount.yaml`)
* **Access Control Identity:** Assigns specific API access credentials to running pods, facilitating secure integrations with external AWS resources via IAM Roles for Service Accounts (IRSA) when custom annotations are applied.

---

## Configuration Reference (`values.yaml`)

The following matrix documents the configurable parameters of the Shopnow Frontend chart and their default values:

<table>
  <thead>
    <tr>
      <th width="25%">Parameter</th>
      <th width="10%">Data Type</th>
      <th width="10%">Default Value</th>
      <th width="55%">Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>global.<wbr>appName</code></td>
      <td>String</td>
      <td><code>"shopnow-frontend"</code></td>
      <td>Standard unique string identifier used as a prefix name for chart resources.</td>
    </tr>
    <tr>
      <td><code>global.<wbr>namespace</code></td>
      <td>String</td>
      <td><code>"shopnow"</code></td>
      <td>Target Kubernetes namespace where resources will be allocated.</td>
    </tr>
    <tr>
      <td><code>global.<wbr>enabledComponents.<wbr>secrets</code></td>
      <td>Boolean</td>
      <td><code>false</code></td>
      <td>Toggles the creation and reference binding of the Secrets resource.</td>
    </tr>
    <tr>
      <td><code>global.<wbr>enabledComponents.<wbr>configmap</code></td>
      <td>Boolean</td>
      <td><code>true</code></td>
      <td>Toggles the creation and volume mounting of the ConfigMap resource.</td>
    </tr>
    <tr>
      <td><code>global.<wbr>enabledComponents.<wbr>service</code></td>
      <td>Boolean</td>
      <td><code>true</code></td>
      <td>Toggles the creation of the internal-facing Service resource.</td>
    </tr>
    <tr>
      <td><code>global.<wbr>enabledComponents.<wbr>ingress</code></td>
      <td>Boolean</td>
      <td><code>true</code></td>
      <td>Toggles the generation of Ingress routing definitions (requires Service enablement).</td>
    </tr>
    <tr>
      <td><code>nameOverride</code></td>
      <td>String</td>
      <td><code>""</code></td>
      <td>Intercepts and overrides chart name string evaluations inside helper templates.</td>
    </tr>
    <tr>
      <td><code>fullnameOverride</code></td>
      <td>String</td>
      <td><code>""</code></td>
      <td>Completely overrides the generated naming prefixes of system resources.</td>
    </tr>
    <tr>
      <td><code>resources.<wbr>requests.<wbr>cpu</code></td>
      <td>String</td>
      <td><code>"100m"</code></td>
      <td>Guaranteed baseline compute capacity allocated to each running application pod.</td>
    </tr>
    <tr>
      <td><code>resources.<wbr>requests.<wbr>memory</code></td>
      <td>String</td>
      <td><code>"128Mi"</code></td>
      <td>Guaranteed baseline memory space allocated to each running application pod.</td>
    </tr>
    <tr>
      <td><code>resources.<wbr>limits.<wbr>cpu</code></td>
      <td>String</td>
      <td><code>"500m"</code></td>
      <td>Maximum burst compute boundary permitted for an individual running pod.</td>
    </tr>
    <tr>
      <td><code>resources.<wbr>limits.<wbr>memory</code></td>
      <td>String</td>
      <td><code>"512Mi"</code></td>
      <td>Maximum burst memory allocation limit permitted for an individual running pod.</td>
    </tr>
    <tr>
      <td><code>image.<wbr>repository</code></td>
      <td>String</td>
      <td><code>"AWS_ACCOUNT_ID.../<wbr>shopnow-fe"</code></td>
      <td>Complete registry directory target path hosting the container image.</td>
    </tr>
    <tr>
      <td><code>image.<wbr>tag</code></td>
      <td>String</td>
      <td><code>"latest"</code></td>
      <td>Target tag or version string identifier of the execution image.</td>
    </tr>
    <tr>
      <td><code>image.<wbr>pullPolicy</code></td>
      <td>String</td>
      <td><code>"IfNotPresent"</code></td>
      <td>Verification requirements for downloading images (<code>Always</code>, <code>IfNotPresent</code>, <code>Never</code>).</td>
    </tr>
    <tr>
      <td><code>image.<wbr>pullSecret</code></td>
      <td>String</td>
      <td><code>""</code></td>
      <td>Name reference for image pull secrets when pulling from restricted registries.</td>
    </tr>
    <tr>
      <td><code>autoscaling.<wbr>enabled</code></td>
      <td>Boolean</td>
      <td><code>true</code></td>
      <td>Toggles the deployment of the Horizontal Pod Autoscaler manifest.</td>
    </tr>
    <tr>
      <td><code>autoscaling.<wbr>minReplicas</code></td>
      <td>Integer</td>
      <td><code>1</code></td>
      <td>Lowest allowable cluster execution pod count threshold.</td>
    </tr>
    <tr>
      <td><code>autoscaling.<wbr>maxReplicas</code></td>
      <td>Integer</td>
      <td><code>5</code></td>
      <td>Upper limit scaling ceiling for application pods under system load.</td>
    </tr>
    <tr>
      <td><code>autoscaling.<wbr>targetCPUUtilization<wbr>Percentage</code></td>
      <td>Integer</td>
      <td><code>80</code></td>
      <td>Average CPU load metric target used to trigger horizontal scaling events.</td>
    </tr>
    <tr>
      <td><code>autoscaling.<wbr>targetMemoryUtilization<wbr>Percentage</code></td>
      <td>Integer</td>
      <td><code>80</code></td>
      <td>Average Memory usage metric target used to trigger horizontal scaling events.</td>
    </tr>
    <tr>
      <td><code>replicaCount</code></td>
      <td>Integer</td>
      <td><code>1</code></td>
      <td>Total static pod execution target if the Horizontal Pod Autoscaler is disabled.</td>
    </tr>
    <tr>
      <td><code>rollingUpdate.<wbr>maxSurge</code></td>
      <td>String</td>
      <td><code>"25%"</code></td>
      <td>Maximum allowable percentage or count of temporary pods created over capacity during updates.</td>
    </tr>
    <tr>
      <td><code>rollingUpdate.<wbr>maxUnavailable</code></td>
      <td>String</td>
      <td><code>"25%"</code></td>
      <td>Maximum allowable percentage or count of unserviceable pods permitted during updates.</td>
    </tr>
    <tr>
      <td><code>service.<wbr>type</code></td>
      <td>String</td>
      <td><code>"ClusterIP"</code></td>
      <td>Kubernetes service networking access layer architecture type (<code>ClusterIP</code>, <code>NodePort</code>, <code>LoadBalancer</code>).</td>
    </tr>
    <tr>
      <td><code>service.<wbr>port</code></td>
      <td>Integer</td>
      <td><code>80</code></td>
      <td>Network port entry point exposed externally by the cluster Service.</td>
    </tr>
    <tr>
      <td><code>service.<wbr>targetPort</code></td>
      <td>Integer</td>
      <td><code>80</code></td>
      <td>Internal execution port where the container application processes traffic.</td>
    </tr>
    <tr>
      <td><code>ingress.<wbr>className</code></td>
      <td>String</td>
      <td><code>"alb"</code></td>
      <td>Explicitly binds the handling ingress controller instance provider (e.g., AWS ALB).</td>
    </tr>
    <tr>
      <td><code>ingress.<wbr>host</code></td>
      <td>String</td>
      <td><code>"shopnow.com"</code></td>
      <td>Target domain address routing key used to match incoming host header paths.</td>
    </tr>
    <tr>
      <td><code>ingress.<wbr>acmArn</code></td>
      <td>String</td>
      <td><code>""</code></td>
      <td>Valid AWS Certificate Manager ARN used to bind SSL certificates to the load balancer.</td>
    </tr>
    <tr>
      <td><code>ingress.<wbr>path</code></td>
      <td>String</td>
      <td><code>"/"</code></td>
      <td>Access path matching route template.</td>
    </tr>
    <tr>
      <td><code>ingress.<wbr>pathType</code></td>
      <td>String</td>
      <td><code>"Prefix"</code></td>
      <td>Verification parsing mode applied against path parameters (<code>Prefix</code>, <code>Exact</code>).</td>
    </tr>
    <tr>
      <td><code>configmap.<wbr>name</code></td>
      <td>String</td>
      <td><code>"application-properties"</code></td>
      <td>Context name suffix string used to isolate the mounted ConfigMap.</td>
    </tr>
    <tr>
      <td><code>configmap.<wbr>data</code></td>
      <td>Object / Map</td>
      <td><code>{}</code></td>
      <td>Key-value dictionary block containing structured config parameters.</td>
    </tr>
    <tr>
      <td><code>secrets.<wbr>type</code></td>
      <td>String</td>
      <td><code>"Opaque"</code></td>
      <td>Standard classification profile describing the nature of the secret contents.</td>
    </tr>
    <tr>
      <td><code>secrets.<wbr>stringData</code></td>
      <td>Object / Map</td>
      <td><code>{}</code></td>
      <td>Unencoded key-value data mappings transformed natively into encoded secrets keys.</td>
    </tr>
    <tr>
      <td><code>serviceAccount.<wbr>create</code></td>
      <td>Boolean</td>
      <td><code>false</code></td>
      <td>Toggles whether a custom application ServiceAccount manifest should be created.</td>
    </tr>
    <tr>
      <td><code>serviceAccount.<wbr>annotations</code></td>
      <td>Object / Map</td>
      <td><code>{}</code></td>
      <td>Custom metadata keys passed to the ServiceAccount (useful for IRSA IAM bindings).</td>
    </tr>
    <tr>
      <td><code>serviceAccount.<wbr>automount</code></td>
      <td>Boolean</td>
      <td><code>true</code></td>
      <td>Enables or disables the automatic mounting of API access tokens to target pods.</td>
    </tr>
    <tr>
      <td><code>serviceAccount.<wbr>name</code></td>
      <td>String</td>
      <td><code>""</code></td>
      <td>Explicit alternative naming definition assigned to the ServiceAccount resource.</td>
    </tr>
  </tbody>
</table>

---

## Operations Guide (CLI Usage)

All operational commands must be executed from the project root workspace directory containing the `./helm` chart infrastructure subdirectory.

### 1. Dry-Run Evaluation and Manifest Rendering
Process, interpolate, and stream the generated Kubernetes object definitions to standard output to validate compilation logic and indentation parameters before applying changes:
```bash
helm template shopnow-frontend ./helm --debug
```

### 2. Linting and Conformity Check
Verify structural configuration syntax rules and chart specification formatting parameters against community guidelines:
```bash
helm lint ./helm
```

### 3. Deploying the Application
Provisions the chart manifests into the designated target cluster environment under an isolated namespace structure:
```bash
helm install shopnow-frontend ./helm -n shopnow --create-namespace
```

### 4. Executing In-Place Upgrades
Safely push changes following revisions to configurations within `values.yaml` or when altering container image tags:
```bash
helm upgrade shopnow-frontend ./helm -n shopnow
```

### 5. Uninstalling the Workload
Completely tear down, unbind, and remove all operational resources, endpoints, and records associated with the release:
```bash
helm uninstall shopnow-frontend -n shopnow
```

## 6. Contact

**Author:** Nguyễn An Phúc (@Bel7phegor)
* **Profiles:** [LinkedIn: nguyen-an-phuc](https://www.linkedin.com/in/nguyen-an-phuc) | [GitHub: Bel7phegor](https://github.com/Bel7phegor) | [Portfolio: anphuc.site](https://anphuc.site)
* **Email:** [nguyenanphuc12032002@gmail.com](mailto:nguyenanphuc12032002@gmail.com)