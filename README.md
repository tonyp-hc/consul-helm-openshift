# consul-helm-openshift
This repository is based off [kawsark's](https://github.com/kawsark/consul-helm-openshift/) original work but updated for Consul 1.9.0 and OpenShift 4.x.

This has been tested with:
- Helm 3.4.0
- Kubernetes 1.19.0
- consul 1.9.0-beta1 and consul 1.9.0-ent-beta1
- consul-helm 0.25.0
- OpenShift 4.6.1

The following images are being used (consul:1.9.0-beta1 can be swapped in)
- consul:      hashicorp/consul-enterprise:1.9.0-ent-beta1
- consul-k8s:  hashicorp/consul-k8s:0.19.0
- envoy:      envoyproxy/envoy-alpine:v1.14.4

A valid license file is required to run Consul Enterprise.

## Considerations
- This deployment is for testing purposes and is definitely not suitable for production.
- Please review each YAML file carefully against your organization's security policies before applying them to your cluster.
- This deployment is using pre-generated templates built against consul-helm:0.25.0

## Special note about Persistent Volumes (PVs)
A traditional [consul-helm](https://learn.hashicorp.com/tutorials/consul/kubernetes-reference-architecture#infrastructure-requirements) installation will rely on [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) to avoid data loss if consul servers are lost. While PVs are typically available for cloud-based OpenShift installations, there may be additional restrictions or steps necessary for on-premise or self-managed OpenShift clusters.

Instructions for enabling PVs will be available further along in this README.

Persistent Volume Claims (PVCs) can also be [predefined](https://www.consul.io/docs/k8s/installation/platforms/self-hosted-kubernetes#predefined-persistent-volume-claims-pvcs) if it is not feasible to allow the helm chart to create them.

## Assumptions
1. An OpenShift cluster has already been installed. This guide has been tested with OpenShift 4.6.1 on Azure using the [RedHat Installer](https://cloud.redhat.com/openshift/install/azure/installer-provisioned).
2. The cluster should be able to retrieve the necessary images (consul, consul-k8s, envoy) and for the versions listed above. If the images need to be stored in a custom repository, they need to be imported into that mirror and the helm chart configured appropriately.

# Manual Installation

## Pre-setup
Ensure that KUBECONFIG has been set up and that the CLI utilities are functional
```bash
$ export KUBECONFIG=/path/to/kubeconfig:$KUBECONFIG

# test config
$ oc cluster-info # or kubectl clusterinfo
$ oc get pods # or kubectl get pods
```

Create a namespace to work within
```bash
$ oc create ns consul

# Switch to the new namespace
$ oc config set-context --current --namespace=consul
```

### Gossip Encryption
Consul will require an encryption key to [enable gossip encryption](https://learn.hashicorp.com/tutorials/consul/gossip-encryption-secure). This key is set with the `encrypt` parameter in the agent configuration file. `consul keygen` can be used to generate a cryptographically secure key.

These templates have a gossip encryption key hardcoded into them which should *NOT* be used. Please generate a new key and substitute your own. The example used in this repository is merely to provide context.

The gossip encryption key needs to be [stored as a Kubernetes secret](https://github.com/hashicorp/consul-helm/blob/master/values.yaml#L69-L86) like so:
```bash
$ kubectl create secret generic consul-gossip-encryption-key --from-literal=key=YZqGRaEajsh8M1w4e1z/Jg==
```

## Setup
### Consul Servers
```bash
$ cd manifests/consul/templates
$ oc apply -f server-serviceaccount.yaml
$ oc apply -f server-securitycontextconstraints.yaml
$ oc get scc consul-server
NAME            PRIV    CAPS         SELINUX    RUNASUSER        FSGROUP     SUPGROUP   PRIORITY     READONLYROOTFS   VOLUMES
consul-server   false   <no value>   RunAsAny   MustRunAsRange   MustRunAs   RunAsAny   <no value>   false            ["configMap","downwardAPI","emptyDir","persistentVolumeClaim","projected","secret"]


$ oc adm policy add-scc-to-user consul-server -z consul-server
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:consul-server added: "consul-server"
```

### Consul Clients (for OpenShift nodes)
```bash
$ oc apply -f client-serviceaccount.yaml
$ oc apply -f client-securitycontextconstraints.yaml
$ oc get scc consul-client
NAME            PRIV    CAPS         SELINUX     RUNASUSER        FSGROUP     SUPGROUP    PRIORITY     READONLYROOTFS   VOLUMES
consul-client   false   <no value>   MustRunAs   MustRunAsRange   MustRunAs   MustRunAs   <no value>   false            ["configMap","downwardAPI","emptyDir","persistentVolumeClaim","projected","secret"]

$ oc adm policy add-scc-to-user consul-client -z consul-client
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:consul-client added: "consul-client"
```


### Apply all the things! 
```bash
# from within manifests/consul/templates/
$ oc apply -f . 
```

### Enable Enterprise License (optional)
The Consul Enterprise license can be applied a few different ways. Most commonly, `server.enterpriseLicense` will be set in the [helm chart](https://github.com/hashicorp/consul-helm/blob/master/values.yaml#L260-L262) but there may be cases where that is not desired. It can also be set using the `consul license put` command like below from any server node in the cluster.
```bash
$ oc exec -it consul-server-0 -- consul license put <paste-license-file-contents>
License is valid
License ID: 148528f7-8d2b-31c4-421f-b4af8a5af74d
Customer ID: 24f4f838-62d0-39cc-ecbf-d62629719802
Expires At: 2020-11-30 00:00:00 +0000 UTC
Terminates At: 2020-11-30 00:00:00 +0000 UTC
Datacenter: *
Modules:
  Global Visibility, Routing and Scale
  Governance and Policy
Licensed Features:
  Automated Backups
  Automated Upgrades
  Enhanced Read Scalability
  Network Segments
  Redundancy Zone
  Advanced Network Federation
  Namespaces
  SSO
  Audit Logging
```


## Check Setup Status
```bash
$ oc get pods
NAME                                                          READY   STATUS    RESTARTS   AGE
consul-7fzt5                                                  1/1     Running   0          82s
consul-8pk7z                                                  1/1     Running   0          82s
consul-connect-injector-webhook-deployment-79954999fb-fqbx7   1/1     Running   0          52s
consul-controller-697c75b9b9-w2z9s                            1/1     Running   0          40s
consul-nzjst                                                  1/1     Running   0          82s
consul-server-0                                               1/1     Running   0          7m7s
consul-server-1                                               1/1     Running   0          7m7s
consul-server-2                                               1/1     Running   0          7m7s
consul-webhook-cert-manager-79f48486c-9l4qc                   1/1     Running   0          27s


$ oc exec -it consul-server-0 -- consul version
Consul v1.9.0+ent-beta1
Revision 97019dcbb
Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)


$ oc exec -it consul-server-0 -- consul members
Node                               Address            Status  Type    Build           Protocol  DC              Segment
consul-server-0                    10.129.3.185:8301  alive   server  1.9.0+entbeta1  2         oc-us-east-poc  <all>
consul-server-1                    10.128.2.30:8301   alive   server  1.9.0+entbeta1  2         oc-us-east-poc  <all>
consul-server-2                    10.131.0.36:8301   alive   server  1.9.0+entbeta1  2         oc-us-east-poc  <all>
ttp-oc-mvrvd-worker-eastus1-t84gj  10.131.0.37:8301   alive   client  1.9.0+entbeta1  2         oc-us-east-poc  <default>
ttp-oc-mvrvd-worker-eastus2-rmb7m  10.128.2.31:8301   alive   client  1.9.0+entbeta1  2         oc-us-east-poc  <default>
ttp-oc-mvrvd-worker-eastus3-mgths  10.129.3.186:8301  alive   client  1.9.0+entbeta1  2         oc-us-east-poc  <default>
```

## Deploy sample services
```bash
# $ cd $WORKINGDIR
$ oc apply -f services/api.yml
$ oc apply -f services/web.yml

$ oc describe pod api-deployment-v1-6bcbd7bd68-ht5kb | grep -i consul.hashicorp.com
Annotations:  consul.hashicorp.com/connect-inject: true
              consul.hashicorp.com/connect-inject-status: injected
              consul.hashicorp.com/connect-service: api
              consul.hashicorp.com/connect-service-port: 9090

$ oc get deployments web-deployment api-deployment-v1
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
web-deployment      1/1     1            1           15m
api-deployment-v1   1/1     1            1           15m
```


# Additional Info

## Generating consul-helm templates (optional)
This repo contains templates created from [consul-helm:0.25.0](https://github.com/hashicorp/consul-helm/blob/master/CHANGELOG.md#0250-oct-12-2020) in order to apply specific customizations, like disabling PVCs. As of [1.9.0](https://github.com/hashicorp/consul-helm/blob/master/CHANGELOG.md#0250-oct-12-2020), Consul natively supports traditional helm installs for OpenShift.

**Note:** If running this command, it must be done with v0.25.0 as of the current version of this guide. Later versions of consul-helm require upgrading consul-k8s and consul images.

```bash
$ git clone --depth 1 --branch v0.25.0 https://github.com/hashicorp/consul-helm.git
$ mkdir -p consul-helm/manifests
$ helm template consul-oc --output-dir consul-helm/manifests/ -f oc-values.yaml hashicorp/consul --version v0.25.0 -n consul

$ ls consul-helm/manifests/consul/templates/
client-config-configmap.yaml                  controller-deployment.yaml                    server-podsecuritypolicy.yaml
client-daemonset.yaml                         controller-leader-election-role.yaml          server-role.yaml
client-podsecuritypolicy.yaml                 controller-leader-election-rolebinding.yaml   server-rolebinding.yaml
client-role.yaml                              controller-mutatingwebhookconfiguration.yaml  server-service.yaml
client-rolebinding.yaml                       controller-podsecuritypolicy.yaml             server-serviceaccount.yaml
client-securitycontextconstraints.yaml        controller-serviceaccount.yaml                server-statefulset.yaml
client-serviceaccount.yaml                    controller-webhook-service.yaml               tests/
connect-inject-clusterrole.yaml               crd-proxydefaults.yaml                        ui-service.yaml
connect-inject-clusterrolebinding.yaml        crd-servicedefaults.yaml                      webhook-cert-manager-clusterrole.yaml
connect-inject-deployment.yaml                crd-serviceintentions.yaml                    webhook-cert-manager-clusterrolebinding.yaml
connect-inject-mutatingwebhook.yaml           crd-serviceresolvers.yaml                     webhook-cert-manager-configmap.yaml
connect-inject-podsecuritypolicy.yaml         crd-servicerouters.yaml                       webhook-cert-manager-deployment.yaml
connect-inject-service.yaml                   crd-servicesplitters.yaml                     webhook-cert-manager-podsecuritypolicy.yaml
connect-inject-serviceaccount.yaml            dns-service.yaml                              webhook-cert-manager-serviceaccount.yaml
controller-clusterrole.yaml                   server-config-configmap.yaml
controller-clusterrolebinding.yaml            server-disruptionbudget.yaml
```

*Important*: There are very likely changes/updates that need to be made to the generated YAML files. The following are the changes that were made to the manifest files provided in this repository:

## Updates to generated consul-helm templates
### Disabling Anti Affinity Rules
In case the deployment target has fewer than three worker nodes, the generated templates have [disabled antiaffinity rules](https://github.com/kawsark/consul-helm-openshift/blob/e031f13e9e8a81d8b42b7ed451fa70282fb8e0ec/oc.values.yaml#L148-L159) by setting `affinity: null` in `oc-values.yaml`

### Disabling PVCs
In the generated `server-statefulset.yaml` file, add a new Data volume:
```yaml
volumes:
  - name: data
    emptyDir: {}
```

Under volumeMounts, rename the /consul/data mount from "data-consul" to data:
If deploying to the default namespace, the mount name might originally be "data-default"
```yaml
volumeMounts:
  - name: data
    mountPath: /consul/data
```

Comment out the volumeClaimTemplates section from the bottom of server-statefulset.yaml
```yaml
# volumeClaimTemplates:
#   - metadata:
#       name: data-default
#     spec:
#       accessModes:
#         - ReadWriteOnce
#       resources:
#         requests:
#           storage: 10Gi
```

For an example of the differences between the original server-statefulset.yaml file and our customized one, check out `overrides/server-statefulset.yaml.orig` and `overrides/server-statefulset-nopvc.yaml`

```bash
$ diff overrides/server-statefulset.yaml.orig overrides/server-statefulset-nopvc.yaml
52a53,54
>         - name: data
>           emptydir: {}
86c88
<             - name: data-default
---
>             - name: data
134,142c136,144
<   volumeClaimTemplates:
<     - metadata:
<         name: data-default
<       spec:
<         accessModes:
<           - ReadWriteOnce
<         resources:
<           requests:
<             storage: 10Gi
---
> #  volumeClaimTemplates:
> #    - metadata:
> #        name: data-default
> #      spec:
> #        accessModes:
> #          - ReadWriteOnce
> #        resources:
> #          requests:
> #            storage: 10Gi
```

### Reorganizing templates
`$ cd consul-helm/manifests/consul/templates/`

Some of these directories may vary depending on your oc-values.yaml definition (like mesh-gateways)

`$ mkdir -pv server client mesh-gateway connect-inject sync-catalog controller crd dns ui webhook`

If some services were not enabled in your oc-values.yaml file, there will be some error output here but it can be ignored
```bash
$ for tpl in server client mesh-gateway connect-inject sync-catalog controller crd dns ui webhook; do mv ${tpl}-*.yaml ./${tpl}/; done
```


### Fix client-config-configmap.yaml
TODO: the gossip encryption key was not added into `data.extra-from-values.json` so I added it manually to the generated template.

`$ vim client/client-config-configmap.yaml`

#### original file snippet
```yaml
data:
  extra-from-values.json: |-
    {}
```

#### updated file snippet
```yaml
data:
  extra-from-values.json: |-
    {
      "primary_datacenter": "oc-us-east-poc",
      "log_level":"INFO",
      "encrypt":"YZqGRaEajsh8M1w4e1z/Jg=="  # Remember: update with your own gossip encryption key
    }
```

The gossip encryption key was, however, present in `server/server-config-configmap.yaml`

### Security Context Constraints (SCC)
Depending on your deployment requirements, SCCs will also need to be configured. Consul 1.9.0 generates `client-securityconstraints.yaml` but does not generate `server-securitycontextconstraints` or anything similar.

One has been provided in the `overrides` folder and so will be present in the templates in this repo. If you generate new templates, you will need to make sure this is ported back in (and updated, if necessary):

```bash
$ cp overrides/server-securitycontextconstraints.yaml consul-helm/manifests/consul/templates/server/
```


## Updating Container Images
If updating container images but not generating new templates, the following files will need updates:
```bash
$ grep -R -e "image:" -e "-image" ./manifests
./manifests/consul/templates/webhook/webhook-cert-manager-deployment.yaml:        image: hashicorp/consul-k8s:0.19.0
./manifests/consul/templates/connect-inject/connect-inject-deployment.yaml:          image: "hashicorp/consul-k8s:0.19.0"
./manifests/consul/templates/connect-inject/connect-inject-deployment.yaml:                -consul-image="hashicorp/consul-enterprise:1.9.0-ent-beta1" \
./manifests/consul/templates/connect-inject/connect-inject-deployment.yaml:                -envoy-image="envoyproxy/envoy-alpine:v1.14.4" \
./manifests/consul/templates/connect-inject/connect-inject-deployment.yaml:                -consul-k8s-image="hashicorp/consul-k8s:0.19.0" \
./manifests/consul/templates/tests/test-runner.yaml:      image: "hashicorp/consul-enterprise:1.9.0-ent-beta1"
./manifests/consul/templates/server/server-statefulset.yaml:          image: "hashicorp/consul-enterprise:1.9.0-ent-beta1"
./manifests/consul/templates/controller/controller-deployment.yaml:        image: hashicorp/consul-k8s:0.19.0
./manifests/consul/templates/client/client-daemonset.yaml:          image: "hashicorp/consul-enterprise:1.9.0-ent-beta1"
```


## Enabling Consul Enterprise Namespaces
[Consul Enterprise Namespaces](https://www.consul.io/docs/enterprise/namespaces) provide tenant isolation and delegation capabilities within your service catalog and service mesh. They are not enabled in a default helm installation but can be enabled even on an already running instance.

The manifest files included with this repo are built against consul-helm v0.25.0, consul-k8s v0.19.0, and consul v1.9.0. The following release of consul-k8s (v0.20.0) and consul-helm v0.26.0) includes support for additional features like Kubernetes health check synchronization with Consul for connect injected pods (`connectInject.healthChecks` [[GH-651](https://github.com/hashicorp/consul-helm/pull/651)].

To enable namespaces, you can run the `helm template` command like above but update your `oc-values.yaml` with the following:
- `global.enableConsulNamespaces: true`
- `connectInject.k8sAllowNamespaces: ["*"]`
- `connectInject.consulNamespaces.mirroringK8S: true`
- `connectInject.consulNamespaces.mirroringK8SPrefix: "oc-"` 

This will generate entirely new template files which can be deployed fresh. However, if you already followed the instructions in this repo to deploy Consul Enterprise on Openshift, you can also run the following using just the manifests provided in the `enable-namespaces` directory:

```bash
$ oc apply -f enable-namespaces/
clusterrole.rbac.authorization.k8s.io/consul-connect-injector-webhook configured
deployment.apps/consul-connect-injector-webhook-deployment configured
deployment.apps/consul-controller configured
customresourcedefinition.apiextensions.k8s.io/serviceintentions.consul.hashicorp.com configured
```

If all went well, we should be able to run the following command:

```bash
$ oc exec -it consul-server-0 -- consul namespace list
default:
   Description:
      Builtin Default Namespace
```
