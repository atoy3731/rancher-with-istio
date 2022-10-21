## Rancher with Istio

**This is an active WIP!**

This provides some examples on enabling Istio injection with Rancher.

### Istio Resources

* example-istio-operator.yaml
    * Provides configuration on disabling the default Ingress Gateway (IGW) and configuring a specific Rancher IGW in the cattle-system namespace.
    * Also configures some lower resource requirements. These might need adjusting depending on your environment.
* example-gateway.yaml
    * Provides an Istio Gateway that will leverage the above Rancher IGW.
    * This is configured to use the existing secret (`tls-rancher-ingress`) that the Rancher chart will create.
    * You'll need to modify the domains/hosts depending on your use-case and certificate.
* example-virtual-service.yaml
    * Provides an Istio VirtualService to manage the routing from your IGW pods to your Rancher pods.
    * References the above gateway.
    * You'll need to modify the domain/host for your use-case.

### Creating the Namespace

You need to create/label the cattle-system namespace for istio-injection.

If the namespace already exists:
```bash
kubectl label namespace cattle-system istio-injection=enabled
```

If it doesn't exist yet:
```bash
kubectl create namespace cattle-system
kubectl label namespace cattle-system istio-injection=enabled
```

If there are workloads/pods running in your cattle-system namespace already, you'll need to cycle them. Most likely, you'll need:
```bash
kubectl rollout restart -n cattle-system deploy/rancher
kubectl rollout restrt -n cattle-system deploy/rancher-webhook
```

Confirm that worked by verifying pods in cattle-system (with the exception of the IGW pods) are now 2/2:
```
[09:44:51] ~ $ kubectl get pods -n cattle-system
NAME                                      READY   STATUS    RESTARTS   AGE
rancher-56dcbcc897-fc6bz                  2/2     Running   0          29m
rancher-56dcbcc897-qclzm                  2/2     Running   0          29m
rancher-56dcbcc897-xtp68                  2/2     Running   0          29m
rancher-ingressgateway-85f99f7ff4-kpgzg   1/1     Running   0          28m
rancher-ingressgateway-85f99f7ff4-p5rdf   1/1     Running   0          28m
rancher-ingressgateway-85f99f7ff4-pjzrp   1/1     Running   0          28m
rancher-webhook-6b895884cf-c8lmv          2/2     Running   0          28m
```

### Updating/Installing Rancher

* example-rancher-values.yaml
    * These are example values to use for Rancher when installing via Helm.
    * There are 2 options for certificate management, depending on if you want to level Rancher to create the `Certificate` resource or not.
    * If you do not want to use Rancher, refer to `example-cert-manager-certificate.yaml` to create your own. This will require that you have cert-manager installed/configured beforehand.

### Find IP of Rancher

* Once you have Istio ingress/injection enabled, you can get the IP of your IGW by doing:
```
kubectl get svc -n cattle-system rancher-ingressgateway
```

Note the external IP (this requires you to have a cloud provider (AWS, Azure, Google, etc.) or tool (MetalLB) to provision LoadBalancer), and configure your DNS with your hostname to access that IP.

### Adding Security Resources

Once Istio is installed and ingress is working, you can layer on security resources. Check out the `istio-security/` directory for examples. The order of rolling these out is very important, so the order is below with notes:

1. *1-istio-ingress-authpolicy.yaml*: Allows traffic from the outside work into your IGW pods.
2. *2-rancher-ingress-authpolicy.yaml*: Allow traffic from the IGW pods to Rancher and opens comms for websocket traffic.
3. *3-rancher-webhook-ingress-authpolicy.yaml*: Allows traffic into the Rancher webhook to manage operator/controller connections.
4. *4-rancher-peer-authentication.yaml*: Enforces mTLS for connections to Rancher (excludes 443 due to websockets, but Rancher itself enforces encryption).
4. *5-rancher-destination-rule.yaml*: Enables TLS from the sidecar container to the Rancher HTTPS endpoint.
5. *zz-deny-authpolicy.yaml*: Default-deny authpolicy in `cattle-system` namespace.

### Caveats

* Istio-Injecting `fleet-default`
    * Right now, you cannot istio-inject fleet-default due to jobs requiring extra hand-holding when it comes to Istio. This will most likely be resolved when Istio moves forward with Ambient Mesh and moves away from sidecars.