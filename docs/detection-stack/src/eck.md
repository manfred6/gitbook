# Elastic Cloud on Kubernetes

This section details the configuration and deployment of the Elastic Cloud on Kubernetes (ECK) setup for my lab.

## ECK Configuration

As mentioned previously, all deployments on my ECK cluster are managed through Ansible. 
This includes ECK, which is configued and deployed using [this]() Ansible role.

Within [tasks/main.yml](), the Elasticsearch deployment is defined as follows:

```yaml
- name: Deploy single-node Elasticsearch
  kubernetes.core.k8s:
    kubeconfig: "{{ elastic_kubeconfig }}"
    state: present
    definition:
      apiVersion: elasticsearch.k8s.elastic.co/v1
      kind: Elasticsearch
      metadata:
        name: "{{ elasticsearch_name }}"
        namespace: "{{ elastic_namespace }}"
      spec:
        version: "{{ elastic_stack_version }}"
        monitoring:
          metrics:
            elasticsearchRefs:
              - name: "{{ elasticsearch_name }}"
          logs:
            elasticsearchRefs:
              - name: "{{ elasticsearch_name }}"
        nodeSets:
          - name: default
            count: 1
            podTemplate:
              spec:
                containers:
                  - name: elasticsearch
                    resources:
                      requests:
                        memory: "{{ elasticsearch_memory_request }}"
                        cpu: "{{ elasticsearch_cpu_request }}"
                      limits:
                        memory: "{{ elasticsearch_memory_limit }}"
                        cpu: "{{ elasticsearch_cpu_limit }}"
            volumeClaimTemplates:
              - metadata:
                  name: elasticsearch-data
                spec:
                  accessModes:
                    - ReadWriteOnce
                  storageClassName: "{{ elasticsearch_storage_class }}"
                  resources:
                    requests:
                      storage: "{{ elasticsearch_storage_size }}"
```

This is a single node deployment using `local-path` storage provider. Since im using a single elasticsearch pod, it is not necessary to introcude affinity rules. This would be a necessity if using multiple elasticsearch pods with sharding as data redundancy/performance measures, to prevent multiple pods landing on the same nodes (which would make the data irecoverable of say two out of three shards landed on the same k3s worker, and that died).
All settings are defined in [defaults/main.yml](), and can be adjusted there.

Kibana and Fleet are defined in the same fashion:


## Deployment

First, we must activate the venv. This can be done using [this helper script]() as follows:

```bash
bash sripts/venv.sh
```

This role can be deployed as such:

```bash
ansible-playbook -i inventory/hosts.ini playbooks/eck.yml
```

We can check the deployment status of the pods as follows:

```bash
export KUBECONFIG=/root/ares-detection/ansible/artifacts/kubeconfig.yaml
kubectl get pods -n kube-system
```

This should produce somethine like the following:

```text
```
