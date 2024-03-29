# Mock Exam 2

  - Take me to the [Mock Exam 2](https://kodekloud.com/topic/mock-exam-2-6/)

Solutions for lab - Mock Exam 2:

With questions where you need to modify API server, you can use [this resource](https://github.com/kodekloudhub/community-faq/blob/main/docs/diagnose-crashed-apiserver.md) to diagnose a failure of the API server to restart.


- 1
  <details>

  Use this YAML file below:

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: allow-redis-access
    namespace: prod-x12cs
  spec:
    podSelector:
      matchLabels:
        run: redis-backend
    policyTypes:
    - Ingress
    ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            access: redis
      - podSelector:
          matchLabels:
            backend: prod-x12cs
      ports:
      - protocol: TCP
        port: 6379
  ```
  </details>


- 2
  <details>

  Use this YAML file below:

  ```yaml
  kind: NetworkPolicy
  apiVersion: networking.k8s.io/v1
  metadata:
    name: allow-app1-app2
    namespace: apps-xyz
  spec:
    podSelector:
      matchLabels:
        tier: backend
        role: db
    ingress:
    - from:
      - podSelector:
          matchLabels:
            name: app1
            tier: frontend
      - podSelector:
          matchLabels:
            name: app2
            tier: frontend
  ```
  </details>


- 3

  <details>

  Update the Pod to use the field `automountServiceAccountToken: false`

  Using this option makes sure that the service account token secret is not mounted in the pod at the location `/var/run/secrets/kubernetes.io/serviceaccount`, provided you have removed any explicit volumes and volumeMounts which will be present if you extracted the manifest from the running pod with `-o yaml`

  Note that this option merely tells the controller not to add a volume and mount if not already present. It does *not* remove any existing mount for the secret.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      run: apps-cluster-dash
    name: apps-cluster-dash
    namespace: gamma
  spec:
    containers:
    - image: nginx
      name: apps-cluster-dash
    serviceAccountName: cluster-view
    automountServiceAccountToken: false
    # Note that we have manually deleted volume/mount that previously existed for secret.
  ```

  </details>


- 4

  <details>

  Add the below rule to `/etc/falco/falco_rules.local.yaml` on controlplane and restart falco using `systemctl restart falco.service` to override the current rule

  ```yaml
  - rule: Terminal shell in container
    desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
    condition: >
      spawned_process and container
      and shell_procs and proc.tty != 0
      and container_entrypoint
      and not user_expected_terminal_shell_in_container_conditions
    output: >
      %evt.time.s,%user.uid,%container.id,%container.image.repository
    priority: ALERT
    tags: [container, shell, mitre_execution]
  ```
  </details>


- 5

  <details>

  The role called dev-user-access has been created for all three namespaces: dev-a. dev-b and dev-z. However, the role in the 'dev-z' namespace grants martin access to all operation on all pods. To fix this, delete and re-create the role as below:

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: dev-user-access
    namespace: dev-z
  rules:
  - apiGroups:
    - ""
    resources:
    - pods
    verbs:
    - get
    - list
  ```
  </details>


- 6

  <details>

  Check the process which is bound to port 8088 on this node using netstat"

  ```
  $ netstat -natulp | grep 8088
  ```

  This shows that the the process `openlitespeed` is the one which is using this port.

  Check if any service is running with the same name

  ```
  $ systemctl list-units  -t service --state active | grep -i openlitespeed
  ```
  >  `lshttpd.service     loaded active running OpenLiteSpeed HTTP Server`


  This shows that a service called `openlitespeed` is managed by `lshttpd.service` which is currently active.

  Stop the service and disable it

  ```
  $ systemctl stop lshttpd
  $ systemctl disable lshttpd
  ```

  Finally, check for the package by the same name

  ```
  $ apt list --installed | grep openlitespeed
  ```

  Uninstall the package

  ```
  $ apt remove openlitespeed -y
  ```
  </details>


- 7

  <details>


  The path to the seccomp profile is incorrectly specified for the omega-app pod.</br>
  As per the question, the profile is created at `/var/lib/kubelet/seccomp/custom-profiles.json`

  ```
  controlplane $ kubectl -n omega describe pod omega-app
  ```

  Output:

  ```
  Events:
    Type     Reason  Age              From             Message
    ----     ------  ----             ----             -------
    Normal   Pulled  5s (x3 over 7s)  kubelet, node01  Container image "hashicorp/http-echo:0.2.3" already present on machine
    Warning  Failed  5s (x3 over 7s)  kubelet, node01  Error: failed to generate security options for container "test-container": failed to generate seccomp security options for container: cannot load seccomp profile "/var/lib/kubelet/seccomp/profiles/custom-profile.json": open /var/lib/kubelet/seccomp/profiles/custom-profile.json: no such file or directory
  ```

  Fix the seccomp profile path in the POD Definition file `/root/CKS/omega-app.yaml`</br>


  ```yaml
  securityContext:
    seccompProfile:
      localhostProfile: custom-profile.json
      type: Localhost
  ```

  Next, update the `custom-profile.json` to allow `read` and `write` syscalls.</br>
  Once done, you should see an output similar to below:

  ```
  controlplane $ cat /var/lib/kubelet/seccomp/custom-profile.json | jq -r '.syscalls[].names[]' | grep -w write
  ```

  > write

  ```
  controlplane $ cat /var/lib/kubelet/seccomp/custom-profile.json | jq -r '.syscalls[].names[]' | grep -w read
  ```

  > read

  Finally, re-create the pod

  ```
  controlplane $ kubectl replace -f /root/CKS/omega-app.yaml --force
  ```

  > pod "omega-app" deleted
    pod/omega-app replaced

  The POD should now run successfully.

  NOTE:

  It may still run even if the above two syscalls are not added. However, adding the syscalls is required to successfully complete this question.

  </details>


- 8

  <details>

  Remove the `SYS_ADMIN` capability from the container for the simple-webapp-1 pod in the POD definition file and re-run the scan.

  ```
  controlplane $ kubesec scan /root/CKS/simple-pod.yaml > /root/CKS/kubesec-report.txt
  ```

  The fixed report should PASS with a message like this:

  ```json
  [
    {
      "object": "Pod/simple-webapp-1.default",
      "valid": true,
      "fileName": "API",
      "message": "Passed with a score of 3 points",
      "score": 3,
  ```
  </details>

- 9

  <details>

  Run trivy image scan on all of the images and check which one does not have HIGH or CRITICAL vulnerabilities.

  ```
  controlplane $ trivy image nginx:alpine
  ```

  ```
  2021-04-26T03:41:49.033Z        INFO    Detecting Alpine vulnerabilities...
  2021-04-26T03:41:49.041Z        INFO    Trivy skips scanning programming language libraries because no supported file was detected
  nginx:alpine (alpine 3.13.5)
  ============================
  Total: 0 (HIGH: 0, CRITICAL: 0)
  ```

  Next, use this image to create the pod

  ```
  controlplane $ kubectl -n seth run secure-nginx-pod --image nginx:alpine
  ```
  </details>


