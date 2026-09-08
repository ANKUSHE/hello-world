# Application Deployment & Validations

**Reference Architecture:** Modern Applications CI/CD Stack on VKS  
**Document Version:** 1.0  
**Package:** Application Deployment, Validations  
**Platform:** vSphere Kubernetes Service (VKS)  
**CD Model:** GitOps / ArgoCD  
**Registry:** Harbor  
**Gateway:** Istio / Gateway API  
**TLS:** cert-manager  
**DNS:** external-dns  

---

## Architecture Overview

```text
Developer / Source Repository
            |
            v
      Tekton CI
   Buildpacks / BuildKit
            |
            v
       OCI Image
            |
            v
    Harbor Registry
 Vulnerability Scan / Signing
            |
            v
      GitOps Repository
            |
            v
     ArgoCD Reconciliation
            |
            v
       VKS Runtime Cluster
            |
            v
       Deployment / Pods
            |
            v
          Service
            |
            v
    Istio Gateway / Route
            |
            +---- cert-manager
            |
            +---- external-dns
            |
            v
   Secure Application Endpoint
```

---

## Document Control

| Field | Value |
|---|---|
| Document Title | Application Deployment & Validations |
| Version | 1.0 |
| Reference | Modern Applications CI/CD Stack on VKS |
| Package |  Application Deployment, Validations |
| Status | Draft / Implementation |
| Platform | vSphere Kubernetes Service (VKS) |
| CD Model | GitOps / ArgoCD |
| Registry | Harbor |
| Gateway | Istio / Gateway API |
| TLS | cert-manager |
| DNS | external-dns |

---

## 1. Purpose

This document defines the implementation and validation procedure for deploying modern microservice applications on vSphere Kubernetes Service (VKS) using the GitOps deployment model described in the reference architecture.

The deployment path covered by this package is:

```text
Approved OCI Image
        |
        v
GitOps Repository
        |
        v
ArgoCD Reconciliation
        |
        v
VKS Workload
        |
        v
Service
        |
        v
Istio Gateway
        |
        v
cert-manager / external-dns
        |
        v
Application Endpoint
```

---

## 2. Scope

### Included

- Namespace preparation
- Kubernetes Deployment and Service
- Application configuration
- Resource configuration
- Health probes
- GitOps deployment
- ArgoCD validation
- Pod/Service validation
- Gateway validation
- TLS validation
- DNS validation
- Functional validation
- Rollback
- Troubleshooting
- Acceptance criteria
- Sign-off

### Out of Scope

The following are separate packages in the overall work allocation:

- Tekton implementation details
- Buildpacks/BuildKit implementation details
- ArgoCD platform installation
- Istio platform installation
- cert-manager installation
- external-dns installation
- Harbor implementation

---

## 3. Reference Architecture

The reference architecture establishes:

| Component | Role |
|---|---|
| Tekton | Kubernetes-native CI engine |
| ArgoCD | GitOps CD engine |
| Buildpacks/BuildKit | Artifact builders |
| Harbor | Container/OCI image registry |
| Istio Gateway | Secure gateway/routing layer |
| cert-manager | TLS certificate management |
| external-dns | DNS management |

ArgoCD uses the **GitOps pull model**: it monitors Git, compares desired and actual state, and reconciles the target VKS cluster.

The architecture recommends:

- **Staging:** automated sync/prune
- **Production:** manual approval gates

---

## 4. Prerequisites

Before deployment, confirm:

- [ ] Target VKS runtime cluster is available and accessible.
- [ ] Target namespace exists or the operator has permission to create it.
- [ ] Approved application image is available in Harbor.
- [ ] GitOps repository and target environment path are available.
- [ ] ArgoCD Application is configured for the target VKS cluster.
- [ ] Istio Gateway, cert-manager, and external-dns are available when external secure exposure is required.
- [ ] Required Kubernetes permissions are available.

---

## 5. Deployment Inputs

Confirm the following before deployment:

- Application name
- Namespace
- Environment
- Harbor registry/repository
- Image tag
- Container port
- Service port
- Replica count
- CPU/memory requests and limits
- Application health endpoint
- Hostname
- TLS requirements

> **Note:** Example values are illustrative only. The reference architecture does not prescribe application-specific ports, replica counts, resource values, or health endpoints.

---

## 6. Procedure 1 – Prepare Namespace

Verify the namespace:

```bash
kubectl get namespace <namespace>
```

If required, create it:

```bash
kubectl create namespace <namespace>
```

**Expected result:** Namespace status is `Active`.

---

## 7. Procedure 2 – Prepare Application Configuration

- Store non-sensitive configuration in ConfigMaps or the project's approved configuration mechanism.
- Store sensitive values using the approved Secret/external-secret mechanism.
- Verify that configuration keys required by the application are present before rollout.
- Do not embed environment-specific secrets directly in the container image.

---

## 8. Procedure 3 – Deploy Application Workload

Define an `apps/v1` Deployment with:

- Approved image
- Desired replicas
- Labels/selectors
- Container port
- Resource requests/limits
- Appropriate readiness/liveness probes

Example validation commands:

```bash
kubectl apply -f deployment.yaml -n <namespace>

kubectl rollout status deployment/<deployment-name> -n <namespace>

kubectl get deployment -n <namespace>
```

---

## 9. Procedure 4 – Deploy Service

Create a `ClusterIP` Service unless another exposure model is explicitly required.

Ensure the Service selector exactly matches the application Pod labels.

Validate:

```bash
kubectl get svc -n <namespace>

kubectl get endpoints -n <namespace>

kubectl get endpointslice -n <namespace>
```

**Expected result:** Service exists and expected endpoints are populated.

---

## 10. Procedure 5 – GitOps Deployment

1. Update the target GitOps manifest to reference the approved image tag.
2. Commit and push the change to the appropriate environment branch/path.
3. Confirm ArgoCD detects the new Git revision.
4. For staging, use the configured automated sync/prune policy.
5. For production, follow the configured review/approval gate before synchronization.

---

## 11. Procedure 6 – Kubernetes Validation

Validate:

- Namespace
- Deployment
- ReplicaSet
- Pods
- Events
- Image
- Resources
- Rollout status

Core commands:

```bash
kubectl get deployment -n <namespace>

kubectl get pods -n <namespace> -o wide

kubectl describe pod <pod-name> -n <namespace>

kubectl get events -n <namespace> --sort-by='.lastTimestamp'

kubectl rollout status deployment/<deployment-name> -n <namespace>
```

Verify that:

- Desired, updated, and available replicas are consistent.
- There are no unexpected restarts.
- There are no critical events.

---

## 12. Procedure 7 – Application Health Validation

Confirm readiness and liveness probes are passing.

Check application logs:

```bash
kubectl logs <pod-name> -n <namespace>
```

For a multi-container Pod:

```bash
kubectl logs <pod-name> -n <namespace> -c <container-name>
```

Validate the application health endpoint from inside the cluster using an appropriate test client.

> The exact health endpoint is application-specific.

---

## 13. Procedure 8 – Service and Internal Connectivity Validation

Confirm:

- Service exists.
- Service has endpoints.
- In-cluster request reaches the Service DNS name and expected port.

**Expected result:** Application returns the expected health or API response.

---

## 14. Procedure 9 – ArgoCD Validation

Confirm the ArgoCD Application points to:

- Intended Git revision
- Target VKS cluster
- Target namespace

**Expected state:**

```text
Sync Status: Synced
Health:      Healthy
```

Investigate any of the following before accepting the deployment:

- `OutOfSync`
- `Degraded`
- `Missing`
- Failed resource state

---

## 15. Procedure 10 – Gateway and Route Validation

When external exposure is required, verify Gateway and route resources.

Validate:

- Hostname
- Listener
- TLS configuration
- Backend Service reference
- Backend port
- Route status

Commands may include:

```bash
kubectl get gateway -A

kubectl get httproute -A

kubectl describe gateway <gateway-name> -n <namespace>

kubectl describe httproute <route-name> -n <namespace>
```

---

## 16. Procedure 11 – TLS Validation

When cert-manager is used, verify the Certificate resource and related certificate request/secret resources.

Commands:

```bash
kubectl get certificate -A

kubectl describe certificate <certificate-name> -n <namespace>

kubectl get certificaterequest -A
```

**Expected result:**

```text
Certificate Ready=True
```

The Gateway must be able to use the intended certificate.

---

## 17. Procedure 12 – DNS Validation

When external-dns is used, verify that the application hostname resolves to the intended Gateway/load-balancer endpoint.

Use `nslookup` or `dig` from an appropriate client.

**Expected result:**

- Hostname resolves correctly.
- External traffic reaches the intended Gateway.

---

## 18. Procedure 13 – External HTTPS Validation

Test the application endpoint using HTTPS:

```bash
curl -I https://<application-domain>
```

Validate:

- HTTP status
- TLS certificate
- Hostname
- Routing
- Application response

> **Production requirement:** Do not use insecure certificate bypasses for final production acceptance.

---

## 19. Procedure 14 – Functional Validation

Execute application-specific smoke tests after infrastructure validation.

Validate:

- Application startup
- Health endpoint
- Critical API path(s)
- Required dependencies
- Authentication/authorization behavior, where applicable
- Absence of critical application errors

Record:

| Field | Details |
|---|---|
| Test Case | Test identifier/name |
| Expected Result | Expected behavior |
| Actual Result | Observed behavior |
| Evidence | Logs/screenshots/output |
| Tester | Person executing validation |

---

## 20. Procedure 15 – Rollback Validation

If the new release fails validation, revert the GitOps change so Git remains the source of truth.

For an immediate Kubernetes rollback where operationally appropriate:

```bash
kubectl rollout undo deployment/<deployment-name> -n <namespace>
```

After rollback, repeat:

- Pod validation
- Service validation
- ArgoCD validation
- Gateway/TLS/DNS validation
- Functional validation

---

## 21. Troubleshooting

| Issue | Troubleshooting |
|---|---|
| Pending Pod | Inspect `kubectl describe pod` and events; check node capacity, requests, taints/tolerations, affinity, and storage requirements. |
| ImagePullBackOff / ErrImagePull | Verify Harbor image name/tag, registry connectivity, and image-pull credentials if required. |
| CrashLoopBackOff | Inspect current/previous logs, configuration, dependencies, command/arguments, and application startup errors. |
| Readiness failure | Verify health path, port, protocol, startup timing, and application readiness. |
| Service has no endpoints | Compare Service selectors with Pod labels and verify Pods are Ready. |
| Internal connectivity works but external access fails | Troubleshoot in order: Pod → Service → Route → Gateway → TLS → DNS → external client. |

---

## 22. Deployment Acceptance Criteria

The deployment is accepted when all applicable criteria are satisfied:

- [ ] Namespace is Active.
- [ ] Deployment is available.
- [ ] Expected Pods are Running/Ready.
- [ ] Correct image/version is deployed.
- [ ] No unexpected restarts.
- [ ] No critical events.
- [ ] Service is present with endpoints.
- [ ] ArgoCD is Synced and Healthy.
- [ ] Gateway/route is valid where applicable.
- [ ] Certificate is Ready where applicable.
- [ ] DNS resolves where applicable.
- [ ] HTTPS succeeds.
- [ ] Application smoke/functional tests pass.

---

## 23. Evidence and Records

Capture the following evidence:

- Git commit/revision
- Image reference
- ArgoCD revision and status
- Deployment/Pod status
- Events
- Relevant logs
- Service/endpoints
- Gateway/route status
- Certificate status
- DNS result
- HTTPS result
- Functional-test evidence

---

## 24. Final End-to-End Checklist

| # | Validation Item | Status |
|---:|---|:---:|
| 1 | Target VKS cluster verified | ☐ |
| 2 | Namespace verified | ☐ |
| 3 | Approved Harbor image verified | ☐ |
| 4 | GitOps manifest verified | ☐ |
| 5 | ArgoCD revision verified | ☐ |
| 6 | Deployment rollout verified | ☐ |
| 7 | Pods and readiness verified | ☐ |
| 8 | Events/logs checked | ☐ |
| 9 | Service and endpoints verified | ☐ |
| 10 | Internal connectivity verified | ☐ |
| 11 | Gateway/route verified | ☐ |
| 12 | TLS verified | ☐ |
| 13 | DNS verified | ☐ |
| 14 | HTTPS verified | ☐ |
| 15 | Functional validation passed | ☐ |
| 16 | Evidence recorded | ☐ |
| 17 | Deployment accepted or rolled back | ☐ |

---

## Validation Sign-Off

| # | Validation Item | Expected Result | Status | Evidence / Remarks |
|---:|---|---|:---:|---|
| 1 | Namespace | Active | ☐ | |
| 2 | Deployment | Available / desired replicas met | ☐ | |
| 3 | Pods | Running and Ready | ☐ | |
| 4 | Pod Events | No critical deployment failures | ☐ | |
| 5 | Image | Approved image/tag deployed | ☐ | |
| 6 | Resources | Requests/limits as approved | ☐ | |
| 7 | Service | Service exists | ☐ | |
| 8 | Endpoints | Expected endpoints populated | ☐ | |
| 9 | ArgoCD | Synced / Healthy | ☐ | |
| 10 | Gateway/Route | Valid and routed | ☐ | |
| 11 | Certificate | Ready=True, if applicable | ☐ | |
| 12 | DNS | Hostname resolves correctly, if applicable | ☐ | |
| 13 | HTTPS | Expected response | ☐ | |
| 14 | Functional Test | Smoke/functional test passed | ☐ | |
| 15 | Rollback | Rollback path verified/available | ☐ | |

---

## Reference

This README is converted from the **Application Deployment & Validations Implementation Document**, Package 6 of the **Modern Applications CI/CD Stack on VKS** reference architecture.




