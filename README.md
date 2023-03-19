# ACM GitOps - Helm Version

ACM, PolicyGenerator Kustomize Pugin, Helm

# 🤠 For the impatient

- Install RHACM Operator 2.6/2.7+ into your OpenShift4 cluster
- Label you local cluster with `useglobal=true` and `teamargo=true`
- Boostrap global ArgoCD for policy

```bash
oc apply -f bootstrap-acm-global-gitops/setup.yaml
```

- Install Team based ArgoCD's

```bash
oc apply -f applicationsets/team-argo-appset.yaml
```

# WIP

 This uses the [PR to implement kustomizeOptions](https://github.com/open-cluster-management-io/policy-generator-plugin/pull/109)

```yaml
policies:
  - name: team-gitops
    manifests:
      - path: helm-input/
        kustomizeOptions:
          enableAlphaPlugins: true
          enableHelm: true
```

# ZTP SNO Instance

