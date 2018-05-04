# k8s-ipam-haci

IP address managenent interface using HaCi (https://sourceforge.net/projects/haci/).

## Deploy

Apply the custom resource:

```bash
kubectl apply -f deploy/crd.yaml
```

Edit the configmap in `deploy/configmap.yaml`, for example to configure the network you want to use. Then:

```bash
kubectl apply -f deploy/configmap.yaml
```

Create a secret for HaCi username and password:

```bash
kubectl create secret generic k8s-ipam-haci \
--from-literal=HACI_USERNAME=... \
--from-literal=HACI_PASSWORD=... \
-n kube-system
```

Then, apply the RBAC configuration and the controller itself:

```bash
kubectl apply -f deploy/rbac.yaml
kubectl apply -f deploy/deployment.yaml
```

The controller can be configured using the following configuration values:

- *HACI_URL* URL to HaCi.
- *HACI_USERNAME* HaCi username.
- *HACI_PASSWORD* HaCi password
- *HACI_ROOT* HaCi root
- *HACI_NETWORK* HaCi network
- *CONTROLLER_TAG* (default: `kubernetes`) this controller's name. Used to generate the default name in HaCi. Used as a tag in HaCi to mark this controller as the "owner". Set to a unique name if you have multiple ipam controllers using the same network in HaCi.
- *LOG_LEVEL* (default: info) Log level (debug, info, ...)
- *NAME_TEMPLATE* (default `{{.Tag}}.{{.Namespace}}.{{.Name}}`) How the address reservations are stored internally.

## How to use

To create an IP address reservation manually, create an IP address request:

```yaml
apiVersion: ipam.nexinto.com/v1
kind: IpAddress
metadata:
  name: myip
spec:
  description: My great service runs here.
```

Check using describe if the address was successfully assigned:

```bash
kubectl describe ipaddress myip
```

The Status fields should contain your address:

```yaml
...
Status:
  Address:   10.10.0.0
  Name:      kubernetes.default.myip
  Provider:  HaCi
```

If something went wrong, an Event is created to explain why.

The IP addresses are namespaced.

The Spec supports the following fields:

- *description* (optional) description for this address reservation
- *name* (optional) name how the IP address management internally stores this address. The default is `$TAG.$NAMESPACE.$NAME`.
- *ref* (optional) do not create a new address; instead reuse an existing entry. Use the IPAM name (like `$TAG.$NAMESPACE.$NAME`), not the Kubernetes object name.

To list all addresses:

```bash
kubectl get ipaddress -o go-template='{{range .items}}{{.metadata.namespace}}-{{.metadata.name}} = {{.status.address}}{{"\n"}}{{end}}'
```