# Mock Exam 1

  - Take me to the [Mock Exam 1](https://kodekloud.com/topic/mock-exam-1-6/)

Solutions for lab - Mock Exam 1:

With questions where you need to modify API server, you can use [this resource](https://github.com/kodekloudhub/community-faq/blob/main/docs/diagnose-crashed-apiserver.md) to diagnose a failure of the API server to restart.


- 1
  <details>

  AppArmor Profile: First load the AppArmor module to the Kernel

  ```bash
  apparmor_parser -q /etc/apparmor.d/frontend
  ```

  Service Account: The pod should use the service account called `frontend-default` as it has the least privileges of all the service accounts in the `omni` namespace (excluding default)

  The other service accounts, `fe` and `frontend` have additional permissions (check the roles and rolebindings associated with these accounts)
  Use the below YAML File to re-create the pod.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    annotations:
      container.apparmor.security.beta.kubernetes.io/nginx: localhost/restricted-frontend # Apply profile 'restricted-fronend' on 'nginx' container
    labels:
      run: nginx
    name: frontend-site
    namespace: omni
  spec:
    serviceAccount: frontend-default # Use the service account with least privileges
    containers:
    - image: nginx:alpine
      name: nginx
      volumeMounts:
      - mountPath: /usr/share/nginx/html
        name: test-volume
    volumes:
    - name: test-volume
      hostPath:
        path: /data/pages
        type: Directory
  ```

  Delete the unused service accounts in the `omni` namespace.

  ```bash
  kubectl -n omni delete sa frontend
  kubectl -n omni delete sa fe
  ```
  </details>


- 2

  <details>

  To extract the secret, run:

  ```bash
  mkdir -p /root/CKS/secrets/
  kubectl -n orion get secrets a-safe-secret -o jsonpath='{.data.CONNECTOR_PASSWORD}' | base64 --decode > /root/CKS/secrets/CONNECTOR_PASSWORD
  ```

  One way that is more secure to distribute secrets is to mount it as a read-only volume.

  Create pod using:

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      name: app-xyz
    name: app-xyz
    namespace: orion
  spec:
    containers:
    - image: nginx
      name: app-xyz
      ports:
      - containerPort: 3306
      volumeMounts:
      - name: secret-volume
        mountPath: /mnt/connector/password
        readOnly: true
    volumes:
    - name: secret-volume
      secret:
        secretName: a-safe-secret
  ```
  </details>


- 3

  <details>

  Get all the images of pods running in the `delta` namespace:

  ```bash
  kubectl -n delta get pods -o json | jq -r '.items[].spec.containers[].image'
  ```

  Scan each image using `trivy image scan` . Example:

  ```bash
  trivy image --severity CRITICAL kodekloud/webapp-delayed-start | grep Total
  ```

  Or, do the above two steps in a single line

  ```bash
  for i in $(kubectl -n delta get pods -o json | jq -r '.items[].spec.containers[].image') ; do echo $i ; trivy image --severity CRITICAL $i 2>&1 | grep Total ; done
  ```

  If the image has HIGH or CRITICAL vulnerabilities, delete the associated pod.

  For example, if 'kodekloud/webapp-delayed-start', 'httpd' and 'nginx:1.16' have these vulnerabilities:

  ```bash
  kubectl -n delta delete pod simple-webapp-1
  kubectl -n delta delete pod simple-webapp-3
  kubectl -n delta delete pod simple-webapp-4
  ```
  </details>


- 4

  <details>

  Copy the `audit.json` seccomp profile to `/var/lib/kubelet/seccomp/profiles`:

  ```bash
  cp /root/CKS/audit.json /var/lib/kubelet/seccomp/profiles
  ```

  Create the pod using the below YAML File

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      run: nginx
    name: audit-nginx
    namespace : default
  spec:
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/audit.json
    containers:
    - image: nginx
      name: nginx
  ```
  </details>


- 5

   <details>

   The fixes are mentioned in the same report. We are asked to fix the `FAIL` for Controller Manager and Scheduler

   * Update the `kube-controller-manager` and `kube-scheduler` static pod manifests under `/etc/kubernetes/manifests` as per the recommendations.
   * Make sure that `--profiling=false`
   * Make sure both pods restart

   </details>


- 6
   <details>

  1. Create `/opt/security_incidents`

      ```bash
      mkdir -p /opt/security_incidents
      ```

  1. Enable file_output in `/etc/falco/falco.yaml`

      ```yaml
      file_output:
        enabled: true
        keep_alive: false
        filename: /opt/security_incidents/alerts.log
      ```

  1. Add the updated rule under the `/etc/falco/falco_rules.local.yaml`:

      ```yaml
      - rule: Write below binary dir
        desc: an attempt to write to any file below a set of binary directories
        condition: >
          bin_dir and evt.dir = < and open_write
          and not package_mgmt_procs
          and not exe_running_docker_save
          and not python_running_get_pip
          and not python_running_ms_oms
          and not user_known_write_below_binary_dir_activities
        output: >
          File below a known binary directory opened for writing (user=%user.name file_updated=%fd.name command=%proc.cmdline)
        priority: CRITICAL
        tags: [filesystem, mitre_persistence]
      ```

  1. To perform hot-reload falco use `kill -1` (SIGHUP) on controlplane node:

        ```bash
        kill -1 $(pidof falco)
        ```

  1.  Verify falco is running, i.e. you didn't make some syntax error that crashed it

      ```bash
      systemctl status falco
      ```

  1.  Check the new log file. It may take up to a minute for events to be logged.

      ```bash
      cat /opt/security_incidents/alerts.log
      ```

  </details>


- 7

  <details>

  Recreate the pod using the YAML file as below:

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    labels:
      run: busy-rx100
    name: busy-rx100
    namespace: production
  spec:
    runtimeClassName: gvisor
    containers:
    - image: nginx
      name: busy-rx100
   ```
  </details>


- 8
  <details>

  1. Create the below admission-configuration inside `/root/CKS/ImagePolicy` directory

      use this YAML file:

      ```yaml
      apiVersion: apiserver.config.k8s.io/v1
      kind: AdmissionConfiguration
      plugins:
      - name: ImagePolicyWebhook
        configuration:
          imagePolicy:
            kubeConfigFile: /etc/admission-controllers/admission-kubeconfig.yaml
            allowTTL: 50
            denyTTL: 50
            retryBackoff: 500
            defaultAllow: false
      ```
  1. The `/root/CKS/ImagePolicy` is mounted at the path /etc/admission-controllers directory in the kube-apiserver. So, you can directly place the files under `/root/CKS/ImagePolicy`.<br/>Snippet of the volume and volumeMounts (Note these are already present in apiserver manifest, so you do not need to add them)

      ```yaml
      containers:
      - # other stuff omitted for brevity
        volumeMounts:
        - mountPath: /etc/admission-controllers
            name: admission-controllers
            readOnly: true
      volumes:
      - hostPath:
          path: /root/CKS/ImagePolicy/
          type: DirectoryOrCreate
        name: admission-controllers
      ```

  1. Update the kube-apiserver command flags and add `ImagePolicyWebhook` to the `enable-admission-plugins` flag

      ```
      - --admission-control-config-file=/etc/admission-controllers/admission-configuration.yaml
      - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
      ```

  1. Wait for the API server to restart. May take up to a minute.

      You can use the folloowing command to monitor the containers

      ```bash
      watch crictl ps
      ```

      `CTRL + C` exits the watch.

  1. Finally, update the pod with the correct image

      ```bash
      kubectl set image -n magnum pods/app-0403 app-0403=gcr.io/google-containers/busybox:1.27
      ```

  </details>
