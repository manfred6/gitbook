# Provisioning Kubernetes

Now that we have created out three kubernetes virtual machines on proxomx (using terraform, see previous page), we can start provisioning them as kubernetes nodes.
For this, I decided on using k3s as its lightweight, easy to install and easy to manage. For our purposes here, this is perfect, as were not aiming to make a kubernetes lab but a detection / purple team lab, and thereby dont want to spend all out time managing kubernetes but actually doing some detection engineering.

K3S will be managed end-to-end using [`ansible`](https://docs.ansible.com/projects/ansible/latest/index.html).

The final design is a three-node K3s cluster running on Proxmox, provisioned with Terraform and configured with Ansible.
K3S uses embedded etcd for a highly available control plane, Flannel for pod networking, MetalLB for bare-metal `LoadBalancer` services, Traefik as the ingress controller, cert-manager for certificates issued from a private lab PKI, and Elastic Cloud on Kubernetes (ECK) to deploy and manage Elasticsearch, Kibana, Fleet Server, and Elastic Agents.

## Initializing the Environment

The previous terraform setup also generated the prerequisite `ansible.cfg` and `hosts.ini` files. In my configuration, my `hosts.ini` looks as follows:

```ini
[k3s_servers]
k3s-node-01 ansible_host=192.168.30.51 ansible_user=ubuntu
k3s-node-02 ansible_host=192.168.30.52 ansible_user=ubuntu
k3s-node-03 ansible_host=192.168.30.53 ansible_user=ubuntu
```

Before we do anything, we need to set up a virtual environment for ansible, and install the prerequisite python packages and ansible modules into it. This is important, as we run GOAD on the same host, and we dont want to mix up ansible / provider versions. Ive provided a convenience script [here]():

```bash
#!/bin/bash

set -euo pipefail

python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install -r requirements.txt

ansible-galaxy collection install -r requirements.yml
```

This can be executed as follows:

```bash
bash scripts/venv.sh
```

We can now test if ansible is wired up correctly using my `ping.yml` test as follows:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/ping.yml
```

Should we be successfull, this outputs the following:

```text
PLAY [Ping K3s hosts] *****************************************************************************************

TASK [Ping] ***************************************************************************************************
ok: [k3s-node-01]
ok: [k3s-node-03]
ok: [k3s-node-02]

PLAY RECAP ****************************************************************************************************
k3s-node-01                : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
k3s-node-02                : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
k3s-node-03                : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Deploying K3S

This section will go over the architecture and deployment of the K3S cluster, including the cluster itself, as well as essential services (exclusing ECK).

### Infrastructure Layout


#### K3S

The Kubernetes cluster consists of three VMs:

| Node | Role | CPU | RAM |
|---|---|---:|---:|
| `k3s-node-01` | K3s server + embedded etcd | 2 vCPU | 8 GiB |
| `k3s-node-02` | K3s server + embedded etcd | 2 vCPU | 8 GiB |
| `k3s-node-03` | K3s server + embedded etcd | 2 vCPU | 8 GiB |

All three nodes are control-plane/server nodes. This gives a real etcd quorum and lets the cluster tolerate the loss of one server while keeping control-plane quorum.
> Note that it is necessary to scale to three nodes right away, as an odd number of control nodes is required to achieve quorum.

K3S itself uses embedded etcd, and each node is both a master and worker (hence `combined`). For networking, I chose to go with `Flannel VXLAN` rather than `Cilium`, as this is natively supported by K3S.
Additionally, I will use the embedded `Traefik` ingress controller as it ships natively with K3S and is easy to configure. This counts also for the embedded `Helm` controller. 
For storage, I opted to go with `local-path`. This due to a couple of reasons. First, it is by far the simplest storage option, and doesnt require complex networking setups, which would add another layer of complexity which could break.
Second, I plan to run only one Elasticsearch instance. Should more be required, I can ensure data-availability and integrity by using the elastic-native shard replication mechanisms.
Other services I might instantiate later (Gitea, Git-Runners, etc.) Also dont need complex networked storage solutions.
I instantiate a `MetalLB` deployment to leverage it for Load Balancing, allowing me to have a single "external" IP to serve as ingress for the cluster, wich will be leveraged by `Traefik`.

#### PKI / DNS

In terms of PKI and DNS, I explicitely want to keep things simple and "hands off", while following best-practices as closely as possible.
I use the existing `OPNSense` DNS-Server together with `CoreDNS` for DNS. In `OPNSense` I create a wildcard entry for the MetalLB IP, such that all future services resolve automatically and I dont need to go provision new entries every time I add a service to the cluster.
For PKI, I set up an "offline" `Root CA`, which lives on the provisioning host. This is used to sign an `Intermediate Issuing CA`, which is imported into `cert-manager`. This allows `cert-manager` to issue certificates for services I deploy on my cluster. I can then simply import this trust chain into my kali browser, and have a clean, hand-off PKI setup.

### Infrastructure Details

This section will go over the details of the DNS, PKI and Ingress setup used.

#### Ingress

Because this is a Proxmox/VM environment rather than a managed cloud, Kubernetes cannot ask a cloud provider for a load balancer.

MetalLB provides that functionality.

Configured range:

```text
192.168.30.200-192.168.30.250
```

Traefik receives:

```text
192.168.30.200
```
> Note this can be configured to any address out of the `MetalLB` pool defined above. I just chose this one.

MetalLB runs in L2 mode. A node advertises ownership of the selected address with ARP. The IP can move to another node if necessary.

Most workloads do not need their own MetalLB IP because Traefik multiplexes many hostnames over the same address.


Traefik is the cluster's front door. The `websecure` entrypoint exposes HTTPS on port 443.
Conceptually, this looks as follows:

```text
client -> https://kibana.ares.internal -> 192.168.30.200 (traefik external) -> kibana-kb-http:5601
```

Application services stay on their native internal ports, and are exposed by traefik using `ClusterIP` services.


#### DNS

For DNS, I wanted to keep things simple. I didnt want to set up a dedicated DNS-Server, which id then need to manage. Therefore, I leveraged the built-in DNS-Server present in `OPNSense`.
There, I set up a wildcard record for my chosen domain, `*.ares.internal`. This Resolves to my MetalLB address, `192.168.30.200`.
This allows me to simply add new services on K3S through traefik, which reuse the same ingress address managed by `MetalLB`, and resolve automatically, withoug having to do anything else other than to create the traefik ingress resource.

The K3S nodes themselves also use OPNsense as their upstream resolver, employing `ares.internal` as a search domain.
On Ubuntu/systemd-resolved, Ansible thereby configures:

```ini
[Resolve]
DNS=192.168.30.1
Domains=ares.internal ~.
```

K3S is pointed at:

```text
/run/systemd/resolve/resolv.conf
```

instead of relying on the local `127.0.0.53` stub. The resolution path is therefore

```text
K3S pod -> CoreDNS -> OPNsense -> *.ares.internal
```

Effectively, this means that future services such as:

```text
kibana.ares.internal
es.ares.internal
fleet.ares.internal
grafana.ares.internal
gitea.ares.internal
```

all resolve to:

```text
192.168.30.200
```

#### PKI

The lab uses a two-tier private PKI. `Ares Lab Root CA` signs `Ares Lab Issuing CA`, which is used by `cert-manager` to sign the service certificates exposed by `Traefik`.
The root CA exists only on the provisioning host. As is standard in this project, this is generated and stored by Ansible in the .gitignored `.secrets` directory.
The root private key never enters Kubernetes, and is the trust anchor installed on lab clients.

The root signs the issuing CA.
The issuing CA private key is allowed into Kubernetes because cert-manager needs it to issue certificates.

##### cert-manager

cert-manager owns normal certificate issuance and renewal for all services running on K3S (that require ingress).
Ansible creates a secret containing the issuing CA keypair and chain, then creates a `ClusterIssuer`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ares-lab-ca
spec:
  ca:
    secretName: ares-lab-issuing-ca
```

cert-manager then issues a wildcard ingress certificate.

```text
*.ares.internal
```

Conceptually:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: lab-wildcard
  namespace: kube-system
spec:
  secretName: ares-lab-wildcard-tls
  dnsNames:
    - "*.ares.internal"
    - "ares.internal"
  issuerRef:
    name: ares-lab-ca
    kind: ClusterIssuer
```
These could also be individual certificates scoped per service, but im lazy :)

##### Traefik

The cert-manager-issued wildcard certificate is configured as the default TLS certificate in `Traefik`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: TLSStore
metadata:
  name: default
  namespace: kube-system
spec:
  defaultCertificate:
    secretName: ares-lab-wildcard-tls
```

Ingress routes can then simply use:

```yaml
tls: {}
```

and receive the wildcard certificate automatically.

This is what makes future service onboarding easy, and easy is good, because im lazy and want to hack and do detection engineering, not micro-manage PKI.

### Infrastructure Deployment

Okay now that ive explained the whole setup, lets go and deploy it.

#### K3S

To deploy K3S, I created a dedicated ansible role [`k3s`]().
Settings can be adjusted in [`defaults/main.yml]().

We can deploy it using the following command:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/k3s.yml
```

Should everything have gone well, we now have a generated `kubeconfig.yaml` in `artifacts/`.
We can check if the cluster is created successfully by fetching the kubeconfig:

```bash
ansible-playbook -i inventory/hosts/ini playbooks/fetch-kubeconfig.yml
```

and running:

```bash
export KUBECONFIG=/root/ares-detection/ansible/artifacts/kubeconfig.yaml
kubectl get pods -n kube-system
```

This should output something like the following, with default services in a `Running` or `Completed` state:

```text
NAME                                      READY   STATUS      RESTARTS      AGE
coredns-7c99d4bb54-rp4hv                  1/1     Running     1 (38h ago)   38h
helm-install-cert-manager-psrj8           0/1     Completed   0             38h
helm-install-metallb-g72dn                0/1     Completed   2 (38h ago)   38h
local-path-provisioner-77b9867795-k8ppv   1/1     Running     1 (38h ago)   2d16h
metrics-server-6dc596dfb8-7hkrc           1/1     Running     3 (38h ago)   2d16h
```

We can now move to setting up `cert-manager` and our PKI infra.

#### cert-manager

The generation of our PKI is managed fully by Ansible. Therefore, we can generate the PKI as well as deploy `cert-manager` in one command:
Like for the K3S cluster itself, the PKI and `cert-manager` settings are defined in [defaults/main.yml]().

```bash
ansible-playbook -i inventory/hosts.ini playbooks/cert-manager.yml
```

This should generate the following artifacts:

```text
.secrets/
`-- pki
    |-- issuing
    |   |-- issuing-ca.crt
    |   |-- issuing-ca.csr
    |   `-- issuing-ca.key
    `-- root
        |-- root-ca.crt
        |-- root-ca.csr
        `-- root-ca.key

4 directories, 6 files
```
> note this gets generated in the project root, not in the `ansible/` directory.

Additionally, we can check that cert-manager has been deployed correctly:

```bash
kubectl get pods -n cert-manager
```

This should output something like the following, with pods in `Running` state:

```text
NAME                                       READY   STATUS    RESTARTS      AGE
cert-manager-689c4c5575-4gx62              1/1     Running   3 (38h ago)   2d14h
cert-manager-cainjector-6fbb9c8cd6-ng6fs   1/1     Running   3 (38h ago)   2d14h
cert-manager-webhook-646c95c5ff-9l5z8      1/1     Running   1 (38h ago)   2d14h
```

We can also check that the `Ares Lab Issuing CA` has been imported correctly as follows:

```bash
kubectl get secret -n cert-manager
```

which should output something like the following:

```text
NAME                                 TYPE                 DATA   AGE
cert-manager-webhook-ca              Opaque               3      2d14h
lab-issuing-ca                       kubernetes.io/tls    2      2d14h
sh.helm.release.v1.cert-manager.v1   helm.sh/release.v1   1      2d14h
sh.helm.release.v1.cert-manager.v2   helm.sh/release.v1   1      38h
```

check some certificate metadata:

```bash
kubectl get secret -n cert-manager lab-issuing-ca -o json | \
    jq -r '.data["tls.crt"]' | \
    base64 -d | \
    openssl x509 -noout -subject -issuer -dates
```

in my case:

```text
subject=CN = Ares Lab Issuing CA
issuer=CN = Ares Lab Root CA
notBefore=Sep  9 19:15:15 2026 GMT
notAfter=Sep  9 19:15:15 2031 GMT
```

#### Traefik

The `Traefik` role bundles `MetalLB` as well. Its settings can be configured in [defaults/main.yml]().
Trafik can, like the other services, be set up using the following command:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/traefik.yml
```

if all went well, we can check the following:

```bash
kubectl get pods -n kube-system | grep traefik
```

which should provide something like the following output:

```text
helm-install-traefik-crd-8hrl2            0/1     Completed   0             38h
helm-install-traefik-pxg7g                0/1     Completed   2 (38h ago)   38h
traefik-59b7647586-6fpdp                  1/1     Running     1 (38h ago)   2d17h
```

Now, we should be all set for deploying ECK. Its deployment and configuration shall be discussed in the next section.
