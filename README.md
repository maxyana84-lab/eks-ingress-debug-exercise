# EKS Ingress Debug Exercise — AWS Load Balancer Controller

Hands-on debugging exercise: deploy intentionally broken Kubernetes manifests (Ingress + Services + Deployments) behind the AWS Load Balancer Controller on EKS, diagnose every failure using only `kubectl`/`aws cli`, fix them, and verify end-to-end traffic over HTTP and HTTPS.

## Architecture

- EKS cluster `alb-lab` (eu-north-1), created via `eksctl` from `cluster.yaml`
- 2x `t3.small` managed nodes (account restricted to Free Tier-eligible instance types — see Bug 0 below)
- AWS Load Balancer Controller (Helm) provisioning an internet-facing ALB
- `store` namespace: `store-api` (http-echo) + `store-frontend` (nginx), each with a Deployment + Service, routed by one Ingress

## Bug 0 — infrastructure (not in the original exercise)

`t3.medium` nodes failed to launch: `InvalidParameterCombination: The specified instance type is not eligible for Free Tier`. Diagnosed via `aws autoscaling describe-scaling-activities` on the node group's ASG. Fixed by switching to `t3.small` (also Free Tier-eligible on this account, with enough headroom for the control-plane add-ons).

## The 8 bugs

| # | Bug | Symptom | Diagnosis command | Fix |
|---|-----|---------|--------------------|-----|
| 1 | Legacy `kubernetes.io/ingress.class: nginx` annotation alongside ALB annotations | Ambiguous ingress class ownership | `kubectl get ingressclass`, `kubectl get ingress -o yaml` | Removed the annotation entirely |
| 2 | `alb.ingress.kubernetes.io/scheme: public` (invalid value) | Controller would reject/ignore the value | `kubectl describe ingress` events | `internet-facing` |
| 3 | `listen-ports` annotation value not quoted as a YAML string | `kubectl apply` fails client-side: `json: cannot unmarshal ... into ... annotations of type string` | `kubectl apply -f broken-ingress.yaml` (fails before reaching the API) | Quoted: `'[{"HTTP": 80}, {"HTTPS": 443}]'` |
| 4 | `certificate-arn: pending` (not a real ARN) | Controller event: `FailedBuildModel: no certificate found for host` | `kubectl describe ingress` → Events | Generated a self-signed cert, imported into ACM (`aws acm import-certificate`), used the real ARN |
| 5 | `ssl-redirect: 443` not quoted | Same client-side decode error as Bug 3 | `kubectl apply` error | Quoted: `'443'` |
| 6 | `ingressClassName: aws` (not a real IngressClass) | Controller never adopts the Ingress | `kubectl get ingressclass` | `alb` |
| 7 | Service `store-api` selector `environment: staging` vs. pod label `environment: production` | `kubectl get endpoints` shows `store-api` with `<none>` despite 2/2 healthy pods | `kubectl get pods --show-labels`, `kubectl get svc -o jsonpath='{.spec.selector}'` | Patched selector to `environment: production` |
| 8 | Service `store-api` `type: NodePort` while Ingress uses `target-type: ip` | Inconsistent traffic model (ALB-to-pod-IP bypasses NodePort) | Annotation review | `type: ClusterIP` |

## Verification

```
$ curl -v http://<alb-dns>/ -H "Host: store.example.com"
< HTTP/1.1 301 Moved Permanently
< Location: https://store.example.com:443/

$ curl -vk https://<alb-dns>/ -H "Host: store.example.com"
< HTTP/2 200
<title>Welcome to nginx!</title>
```

HTTP correctly redirects to HTTPS (ssl-redirect working as designed), and HTTPS serves the frontend through a verified TLS handshake against the ACM-imported certificate.

## Files

- `cluster.yaml` — eksctl cluster config
- `broken-ingress.yaml`, `broken-services.yaml`, `broken-deployments.yaml` — manifests as originally given, bugs intact
- `fixed-ingress.yaml` — corrected Ingress
- Service/Deployment fixes applied via `kubectl patch` (selector + type), documented above

## Incident report

Root cause: a batch of Ingress/Service manifests contained 8 defects spanning three categories — invalid/unquoted ALB annotation values that broke the controller's model-building step entirely; an incorrect `ingressClassName` that would have prevented the controller from ever adopting the resource; and a Service/Deployment label mismatch that silently zeroed out endpoints independent of the ALB issues.

Recommended safeguards:
1. CI validation (`kubeconform`/OPA) rejecting unquoted ALB annotations and unknown `ingressClassName` values before merge.
2. A pre-merge check diffing Service `selector` against the matching Deployment's pod labels.
3. Second-reviewer requirement for Ingress/Service changes in production namespaces.
4. A staging smoke test (`curl` + TLS handshake check) gated in the deploy pipeline.
