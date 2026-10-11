Amazon EKS Production: Zero-Downtime Kubernetes Rolling Updates
Overview

This repository demonstrates production-oriented rolling deployments on Amazon Elastic Kubernetes Service (EKS), designed to minimize application downtime during releases.

The deployment strategy combines:

Kubernetes RollingUpdate with maxUnavailable: 0
Multiple application replicas
Readiness and startup probes
AWS Load Balancer Controller Pod Readiness Gates
PodDisruptionBudget (PDB)
Graceful application shutdown
Pod distribution across Availability Zones
Deployment monitoring and rollback procedures

Goal: Keep the existing application serving traffic until replacement Pods are healthy and ready to receive traffic.

Zero downtime cannot be guaranteed by a Deployment manifest alone. Application behavior, load-balancer health, cluster capacity, networking, and dependency availability must also be validated.
Architecture
                    Internet
                       |
                  Route 53 / DNS
                       |
                 AWS ALB / NLB
                       |
                Target Group
                       |
                Kubernetes Service
                       |
             +---------+---------+
             |         |         |
          Pod v1    Pod v1     Pod v1
             |         |         |
             +---- Rolling Update
                       |
                New Pods v2
                       |
              Readiness Checks
                       |
              Target Health Check
                       |
             Old Pods terminate

Repository Structure
eks-zero-downtime-deployment/
├── README.md
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── pdb.yaml
└── .gitignore


The ALB Ingress or other load-balancer resources may already be managed by your platform repository or Helm chart.

Prerequisites
An existing Amazon EKS cluster.
kubectl configured for the correct cluster and AWS account.
An application image stored in Amazon ECR or another accessible registry.
A minimum of three application replicas for this example.
Sufficient worker-node capacity for surge Pods.
AWS Load Balancer Controller installed and configured if using its ALB/NLB integration.
An existing Ingress or equivalent load-balancer configuration when external traffic is required.
A working application readiness endpoint.

Use a staging environment to validate the manifests before production.

1. Create the Namespace

Create namespace.yaml:

apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    elbv2.k8s.aws/pod-readiness-gate-inject: enabled


Apply it:

kubectl apply -f namespace.yaml


The namespace label enables automatic injection of AWS Load Balancer Controller Pod Readiness Gates for eligible new Pods.

Important: For readiness gates to be injected, the matching Service and the relevant TargetGroupBinding must exist before creating the Pods. The target group must use IP targets for this mechanism.

If your existing production namespace already exists, label it instead:

kubectl label namespace production \
  elbv2.k8s.aws/pod-readiness-gate-inject=enabled \
  --overwrite


Verify:

kubectl get namespace production --show-labels


For an existing application, plan the change carefully: adding the label does not inject readiness gates into already-created Pods.

2. Create the Kubernetes Service

Create service.yaml:

apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: http


Apply it:

kubectl apply -f service.yaml


Ensure the existing Ingress or AWS Load Balancer Controller configuration references this Service and that the corresponding target group uses IP targets.

3. Configure a Production Deployment

Create deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3

  revisionHistoryLimit: 10
  minReadySeconds: 10
  progressDeadlineSeconds: 600

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp

    spec:
      terminationGracePeriodSeconds: 60

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: myapp

      containers:
        - name: myapp
          image: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/myapp:VERSION

          ports:
            - name: http
              containerPort: 8080

          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            periodSeconds: 5
            failureThreshold: 30

          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]

          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              memory: "512Mi"


Replace the image, AWS account ID, region, and health endpoints with values for your application.

Production considerations:

maxUnavailable: 0 prevents the Deployment from intentionally reducing available replicas below the desired count during a rolling update.
maxSurge: 1 permits one additional Pod, subject to available cluster resources.
startupProbe accommodates slower application startup.
readinessProbe prevents Kubernetes Services from routing traffic to unready Pods.
livenessProbe detects application processes that are running but unhealthy.
preStop provides a short shutdown delay; your application must also stop accepting new work and gracefully drain in-flight requests.
topologySpreadConstraints encourages distribution across Availability Zones but does not guarantee it under all scheduling conditions.

The application must implement the configured health endpoints. Tune resource requests, limits, and probe thresholds based on real production measurements.

4. Configure a PodDisruptionBudget

Create pdb.yaml:

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp


Apply it:

kubectl apply -f pdb.yaml


With three replicas, this PDB requires at least two available Pods during supported voluntary disruptions, such as node drains.

A PDB is not a replacement for Deployment rolling-update settings, and it does not protect against involuntary failures.

5. Verify AWS Load Balancer Integration

Check the AWS Load Balancer Controller:

kubectl get deployment -A | grep aws-load-balancer-controller


Check the application Service and related resources:

kubectl get service -n production
kubectl get ingress -n production
kubectl get targetgroupbindings -n production


Inspect Pod readiness gates:

kubectl get pods -n production
kubectl describe pod -n production <POD_NAME>


For an eligible Pod, inspect the Conditions section for a condition with a name similar to:

target-health.elbv2.k8s.aws/...


The condition should become true only when the associated load-balancer target is healthy.

If no readiness gate is present, verify namespace labeling, Service selectors, TargetGroupBinding creation, target type, and whether the Pods were created after the prerequisites existed.

6. Deploy a New Application Version

Use an immutable image tag or digest for the release.

Update the image:

kubectl set image deployment/myapp \
  myapp=ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/myapp:NEW_VERSION \
  -n production


Monitor rollout progress:

kubectl rollout status deployment/myapp \
  -n production \
  --timeout=10m


Watch Pods:

kubectl get pods -n production -w


Check the Deployment:

kubectl get deployment myapp -n production
kubectl get replicasets -n production

Expected rollout sequence
Kubernetes creates a new Pod.
The application starts and passes its startup and readiness checks.
The AWS Load Balancer Controller readiness gate waits for the eligible load-balancer target to become healthy.
The new Pod becomes available.
Kubernetes can terminate an old Pod and proceed with the next replacement.
The Deployment finishes when the desired replicas are updated and available.

If the new Pod cannot become healthy, the rollout should stall rather than deliberately removing an available replica to make progress. Investigate the failure instead of forcing the rollout.

7. Verify Production Traffic

For ALB-based applications, check the target group's health in the AWS console or through the AWS CLI.

Example:

aws elbv2 describe-target-health \
  --target-group-arn "$TARGET_GROUP_ARN" \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
  --output table


Run continuous HTTP checks from a suitable monitoring host while deploying:

while true; do
  curl -sS -o /dev/null \
    -w '%{http_code} %{time_total}\n' \
    https://YOUR_APPLICATION_DOMAIN/health/ready \
    || echo "Request failed"

  sleep 1
done


Use an application endpoint that reflects meaningful service availability, not just process liveness.

Monitor the following throughout the release:

HTTP 5xx error rate.
Request latency and timeouts.
ALB target health and unhealthy target counts.
Pod readiness and restart counts.
CPU, memory, and node capacity.
Application and dependency errors.

A successful rollout command is not sufficient evidence of zero downtime. Confirm the external traffic path and service-level indicators.

8. Roll Back a Failed Release

Inspect rollout history:

kubectl rollout history deployment/myapp -n production


Roll back to the previous revision:

kubectl rollout undo deployment/myapp -n production


Verify the rollback:

kubectl rollout status deployment/myapp \
  -n production \
  --timeout=10m

kubectl get pods -n production


Confirm that the restored version is healthy and that traffic has recovered.

For GitOps-managed deployments, revert the source-of-truth change and let the GitOps controller reconcile the desired state.

9. Production Deployment Checklist
At least three replicas, sized for expected production traffic.
Sufficient node capacity for surge Pods and rescheduling.
Readiness and startup probes tested against real application behavior.
AWS Load Balancer Controller integration verified, where applicable.
Target health and Pod readiness gates verified for new Pods.
Graceful shutdown and in-flight request draining implemented.
PDB configured for voluntary disruptions.
Pods distributed across Availability Zones where possible.
Metrics, logs, alerts, and continuous traffic checks enabled.
Rollback procedure tested.
Database migrations and API changes are backward compatible.
Release tested in staging before production rollout.
10. Troubleshooting
Rollout is stuck
kubectl describe deployment myapp -n production
kubectl get events -n production \
  --sort-by=.metadata.creationTimestamp
kubectl describe pods -n production


Check readiness failures, image pulls, scheduling constraints, quota, and available resources.

Pods remain unready because of target health

Check the AWS target group, health-check path, target port, security groups, network connectivity, and TargetGroupBinding status.

Do not remove readiness gates merely to force the rollout to finish.

Insufficient cluster capacity

maxSurge: 1 requires enough capacity to schedule an extra Pod. Check node capacity, pod IP availability, subnet capacity, quotas, and autoscaler behavior.

Traffic drops despite healthy Pods

Kubernetes readiness and external load-balancer health are separate signals. Verify readiness-gate injection, target registration, connection draining, DNS, and application-level errors.

Rollback does not restore service

A rollback cannot automatically reverse incompatible database migrations, broken shared dependencies, or external infrastructure changes. Keep application releases backward compatible and maintain a tested recovery plan.

References
Amazon EKS Best Practices: Load Balancing
Amazon EKS Best Practices: Highly Available Applications
AWS Load Balancer Controller: Pod Readiness Gates
Kubernetes: Deployments
Kubernetes: Pod Disruption Budgets
Conclusion

A production EKS rolling update requires coordination between Kubernetes, the application, and the AWS load-balancing layer.

The combination of maxUnavailable: 0, adequate surge capacity, realistic readiness checks, AWS Pod Readiness Gates where applicable, graceful termination, and monitored rollback significantly reduces the risk of release-related downtime. Validate the full production traffic path before treating a deployment as zero-downtime capable.
