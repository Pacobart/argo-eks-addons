# argo-eks-addons
Deploy miscellaneous addons to EKS using ArgoCD

## Setup

1. Clone repo
2. Change `repoUrl` in `chart/values.yaml` to your repo

## useful commands

Log into argocd from the cli:
```
export ARGOCD_SERVER=`kubectl get svc argo-cd-argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'`
export ARGOCD_PWD=$(aws secretsmanager get-secret-value --secret-id devops-eks-argocd | jq -r '.SecretString')
argocd login $ARGOCD_SERVER --username admin --password $ARGOCD_PWD --insecure
```

List argocd applications:
```
argocd app list
```

Sync argocd applications:
```
argocd app sync add-ons
```


## Sample application definition

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: add-ons
  namespace: argocd
spec:
  project: default
  syncPolicy:
    automated:
      prune: true
  source:
    repoURL: https://github.com/Pacobart/argo-eks-addons.git
    targetRevision: HEAD
    path: chart
    helm:
      releaseName: add-ons
      values: |
        region: us-west-2
        account: XXXXXXXXXXXX
        clusterName: devops-eks
        environment: dev
        domain_name_suffix: mydomain.com
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
```

## How to setup new addon

1. create new directory in `addons`
2. add chart config
3. create new template in `chart\templates`
4. add variables to `chart/values.yaml`

## How to apply different values to environments (See exzternal-dns addon)

1. Add a valueFiles section into the chart/templates/<addon>.yaml file. Ex: https://github.com/Pacobart/argo-eks-addons/blob/main/chart/templates/ingress-nginx.yaml#L16-L18
2. Create `values-<environemnt>.yaml` file in add-ons/<addon>
3. Add environment specific values here. We reference the base `values.yaml` as well for global changes

## How to have different chart versions per environment:

1. restructure addon directory in `add-ons/<addon>` like the following
```
ingress-nginx
-- global-values.yaml
-- dev
---- Chart.yaml
---- values.yaml
-- uat
---- Chart.yaml
---- values.yaml
-- prod
---- Chart.yaml
---- values.yaml
```
2. Modify `chart/templates/<addon>.yaml` file:
```
spec.project.source.path: add-ons/ingress-nginx/{{ .Values.environment}}
```