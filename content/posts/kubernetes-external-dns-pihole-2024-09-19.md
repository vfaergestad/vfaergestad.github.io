---
title: "Deploying External-DNS for Pi-hole on Kubernetes"
date: 2024-09-19T20:00:50+02:00
draft: false
---

## Why

External-DNS is a Kubernetes addon(?) that automatically creates DNS records for your services and ingresses in a DNS provider. You reading this probably means you know what it is, if not, check it [out](https://github.com/kubernetes-sigs/external-dns).

It fits great into an Infrastructure as Code and GitOps workflow, where you can define your DNS records (or use existing resources) in your Kubernetes manifests and let External-DNS do the rest.

This guide has a goal of being one step up from the [official docs](https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/pihole), with more explanation and a bit more reasoning behind the steps.
## What

- [Pi-hole](https://github.com/pi-hole/pi-hole): **The** network-wide ad blocker.
- [External-DNS](https://github.com/kubernetes-sigs/external-dns)
- [Kubernetes](https://kubernetes.io/) (I'm using [k3s](https://k3s.io/))
- [Kustomize](https://kustomize.io/) (optional)

## What it's not
- How to set up Pi-hole.
- How to set up Kubernetes.
- How to use it in a GitOps workflow.

## How

### Prerequisites

- Access to a Kubernetes cluster using `kubectl`.
- Pi-hole running somewhere your cluster can reach it.

### Deploy External-DNS

There are two different ways of deploying External-DNS according to the [docs](https://kubernetes-sigs.github.io/external-dns/latest/):
- Helm Chart
- YAML manifests

I usually prefer helm charts, for reasons I should write about some day, but the Helm Chart barely [does not support](#how-helm-almost-works-for-the-pihole-provider) the `pihole` provider yet. So we'll use the YAML manifests.

First, create a namespace for the External-DNS deployment:

```bash
kubectl create namespace external-dns
```

This could be anything, so if you change it, look for the `namespace` field in the manifests and commands and change them accordingly.


#### Configure Pi-hole Authentication (Optional)
If your Pi-hole instance doesn't require authentication or you prefer not to use Kubernetes secrets (you shouldn't, at least not directly ([1password-operator](https://developer.1password.com/docs/k8s/k8s-operator/?deployment-type=helm))), you can skip ahead to deploying the manifests.

However, if your Pi-hole's admin dashboard is password-protected, you'll need to deploy this secret somehow ([1password-operator](https://developer.1password.com/docs/k8s/k8s-operator/?deployment-type=helm)).

There is a possibility to use the `--pihole-password` flag, but please don't. You would want to check in your manifests to a Git repository, and you don't want to check in your password.

Create the secret by running: (please use your password instead of `passwordhere`)

```bash
Copy code
kubectl --namespace external-dns create secret generic pihole-password \
--from-literal=EXTERNAL_DNS_PIHOLE_PASSWORD=passwordhere
```

Both the secret name and the literal key are a subject of change, so if you change them, look for the `secretKeyRef` field in the manifests and change them accordingly.

#### Create the manifests

The following yaml's is taken directly from the [docs](https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/pihole).

Let's start with the fun part, the permission-stuff.

I will propose some filenames, but you can name them whatever you want. Just make sure you know what's in them.

A ServiceAccount: This will be the account External-DNS will use to interact with the Kubernetes API.

`external-dns-serviceaccount.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
```

A ClusterRole: This defines the permissions you want the External-DNS ServiceAccount to have:

`external-dns-clusterrole.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list","watch"]
```

A ClusterRoleBinding: This binds the ClusterRole to the ServiceAccount. Be sure to change the `namespace` field if you changed the namespace.

`external-dns-clusterrolebinding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: external-dns
```

Finally, the actual fun part, the Deployment. This is where you define the External-DNS container and its configuration.

`external-dns-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.14.2
        # If authentication is disabled and/or you didn't create
        # a secret, you can remove this block.
        envFrom:
        - secretRef:
            # Change this if you gave the secret a different name
            name: pihole-password
        args:
        - --source=service
        - --source=ingress
        # Pihole only supports A/AAAA/CNAME records so there is no mechanism to track ownership.
        # You don't need to set this flag, but if you leave it unset, you will receive warning
        # logs when ExternalDNS attempts to create TXT records.
        - --registry=noop
        # IMPORTANT: If you have records that you manage manually in Pi-hole, set
        # the policy to upsert-only so they do not get deleted.
        - --policy=upsert-only
        - --provider=pihole
        # Change this to the actual address of your Pi-hole web server
        - --pihole-server=http://pihole-web.pihole.svc.cluster.local
      securityContext:
        fsGroup: 65534 # For ExternalDNS to be able to read Kubernetes token files
```

Now take a minute and look through the spec, especially the `args` field. My way may not be your way.

### Apply the manifests

Now that you have 4 different manifests, lets apply them. This can be done in a boring way, or in a fun way. I'll show you the fun way (and then the boring way).

Kustomize is a tool that does so much more than what I'm going to show you, but for now, we'll use it to apply the manifests:

Create a `kustomization.yaml` file:

```yaml
resources:
- external-dns-serviceaccount.yaml
- external-dns-clusterrole.yaml
- external-dns-clusterrolebinding.yaml
- external-dns-deployment.yaml
```

If you want to take a look at the final manifests before applying them, you can run:

```bash
kustomize build .
```

And apply it:

```bash
kubectl apply -k .
```

Notice that we don't specify a file, but rather a directory. This is because Kustomize will look for a `kustomization.yaml` file in the directory and apply it.

If you don't want to use Kustomize, you can apply the manifests one by one:

```bash
kubectl apply -f external-dns-serviceaccount.yaml
kubectl apply -f external-dns-clusterrole.yaml
kubectl apply -f external-dns-clusterrolebinding.yaml
kubectl apply -f external-dns-deployment.yaml
```

Or all at once:

```bash
kubectl apply -f external-dns-serviceaccount.yaml -f external-dns-clusterrole.yaml -f external-dns-clusterrolebinding.yaml -f external-dns-deployment.yaml
```

You could also just smash them all together in a single file, but since you of course are checking in your manifests to a Git repository, and you want to keep their history clean, I would advise against it.

### Verify

There would be no point in deploying External-DNS if you didn't verify that it works.

Start by finding the External-DNS pods name:

```bash
kubectl get pods --namespace external-dns
```

Check its logs:

```bash
kubectl logs --namespace external-dns pods/<pod-name>
```

You should see something like:

```
time="2024-09-19T16:44:10Z" level=info msg="config: {APIServerURL: KubeConfig: ............"
time="2024-09-19T16:44:10Z" level=info msg="Instantiating new Kubernetes client"
time="2024-09-19T16:44:10Z" level=info msg="Using inCluster-config based on serviceaccount-token"
time="2024-09-19T16:44:10Z" level=info msg="Created Kubernetes client https://10.43.0.1:443"
time="2024-09-19T16:44:10Z" level=info msg="All records are already up to date"
```

If you see `All records are already up to date`, that's great! It means External-DNS is working and it doesn't need to create any records.

Now, let's create a test ingress to see if External-DNS creates the records for them.

Create a test ingress (Taken from the [docs](https://kubernetes-sigs.github.io/external-dns/latest/docs/tutorials/pihole)).
`test.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: foo
spec:
  ingressClassName: nginx
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: foo
            port:
              number: 80
```

Apply it:

```bash
kubectl apply -f test.yaml
```

Remember, that since we are not specifying a namespace, it will be created in the namespace that `kubectl` is currently using (often `default`).

Check the logs of the External-DNS pod again:

```bash
kubectl logs --namespace external-dns pods/<pod-name>
```

After a few seconds, you should see something like:

```
time="2024-09-19T16:50:12Z" level=info msg="add foo.bar.com IN A -> 172.20.10.151"
```

And if you check your Pi-hole dashboard, you should see a new record for `foo.bar.com`.

Remember to clean up after yourself (or not, if you want to keep the records):

```bash
kubectl delete -f test.yaml
```

### Conclusion

And that's it! You now have External-DNS running in your cluster, creating DNS records for your services and ingresses in your Pi-hole instance. My suggested next steps would be to:
- Use an external secret manager to store the Pi-hole password ([1password-operator](https://developer.1password.com/docs/k8s/k8s-operator/?deployment-type=helm)).
- Check in your manifests to a Git repository and use a GitOps workflow to deploy them. ([Flux](https://fluxcd.io/), [ArgoCD](https://argoproj.github.io/argo-cd/), etc.)

## Appendix

### How Helm *almost* works for the `pihole` provider

External-DNS does have a Helm Chart, but it does not officially support the `pihole` provider yet. However, I still tried.

Let's take a look at the manifest that we already created, but only look at the parts of it are Pi-Hole specific.

`external-dns-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  template:
    spec:
      containers:
      - name: external-dns
        envFrom:
        - secretRef:
            # Change this if you gave the secret a different name
            name: pihole-password
        args:
          - --source=service
          - --source=ingress
          - --registry=noop
          - --policy=upsert-only
          - --provider=pihole
          - --pihole-server=http://pihole-web.pihole.svc.cluster.local
```

First, the args. All of these, except the `--pihole-server` are templated in the Helm Chart, so you can just set them in the `values.yaml` file.

```yaml
sources:
  - service
  - ingress
registry: noop
policy: upsert-only
provider: pihole
```

And the `--pihole-server` is solved bu the `extraArgs` field in the Helm Chart:

```yaml
extraArgs:
  - --pihole-server=http://pihole-web.pihole.svc.cluster.local
```

We are really close to something here, and only the `envFrom` field is left. This is where the Helm Chart falls short. The `envFrom` field is not templated in the Helm Chart. We could of course hack our way around it, but that would be a hack.

So, until the `envFrom` field is templated in the Helm Chart, we'll have to use the YAML manifests, or a helm-kustomize workaround. Maybe I'll do a pull request to the Helm Chart someday.