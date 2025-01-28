## Extended Management Configuration

k0rdent is deployed with the following default configuration, which may vary
depending on the release version:

```yaml
apiVersion: k0rdent.mirantis.com/v1alpha1
kind: Management
metadata:
  name: kcm
spec:
  providers:
  - name: k0smotron
  - name: cluster-api-provider-aws
  - name: cluster-api-provider-azure
  - name: cluster-api-provider-vsphere
  - name: projectsveltos
release: kcm-0-0-7
```

As you can see, the `Management` object defines the providers that are available from wtihin k0rdent. Some of these are
providers directly used by the user, such as aws, azure, and so on, and others are used internally
by k0rdent, such as Sveltos.

To see what is included in a specific release, look at the `release.yaml` file in the tagged release.
For example, here is the [v0.0.7 release.yaml](https://github.com/k0rdent/kcm/releases/download/v0.0.7/release.yaml).

There are two options to override the default management configuration of k0rdent:

1. Update the `Management` object after the k0rdent installation using `kubectl`:

    `kubectl --kubeconfig <path-to-management-kubeconfig> edit management`

2. Deploy k0rdent skipping the default `Management` object creation, then provide your
   own `Management` configuration. For example:

	- Create `management.yaml` file and configure core components and providers. For example, this system will use only
    AWS as an end user provider:

    ```yaml
    apiVersion: k0rdent.mirantis.com/v1alpha1
    kind: Management
    metadata:
      name: kcm
    spec:
      providers:
      - name: k0smotron
      - name: cluster-api-provider-aws
      - name: projectsveltos
    release: kcm-0-0-7
    ```

	3. Install k0rdent as normal, but specify the `--create-management=false` controller argument to helm, as in:

    ```shell
		helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm --version 0.0.7 -n kcm-system --create-namespace --set="controller.createManagement=false"
    ```

	4. Apply the `kcm` `Management` object after k0rdent installation:

      ```bash
      kubectl --kubeconfig <path-to-management-kubeconfig> create -f management.yaml
      ```