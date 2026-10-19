# Kubernetes Deployment Limitations - Q&A Quiz

## Question 1: What is the "all-or-nothing" limitation of a standard Kubernetes Deployment rolling update?

**Question:** What is meant by the "all-or-nothing" limitation of a standard Kubernetes Deployment rolling update?

**Answer:** The update proceeds from start to finish without the ability to automatically pause or abort based on complex success criteria such as error rates or latency, potentially rolling out a buggy version to 100% of users.

A standard rolling update replaces old pods with new pods according to the rollout settings. The rollout does not natively pause to analyze metrics such as error rates or wait for manual approval unless a basic health probe fails.

---

## Question 2: Why is precise traffic shifting difficult with standard Kubernetes Deployments?

**Question:** Why is "Traffic Shifting" or "Traffic Splitting" difficult to achieve precisely using only standard Kubernetes Deployments?

**Answer:** Traffic is distributed based on the number of pod replicas. For example, 1 new pod out of 10 total pods represents approximately 10% of the pods, making it computationally expensive or impractical to route very small percentages such as 1% without significant over-provisioning.

The training example explains that sending approximately 1% of traffic using replica counts could require 1 new pod and 99 old pods.

```text
Standard Deployment traffic split

1 new pod + 9 old pods
        |
        v
Approximately 10% new / 90% old

For approximately 1%:
1 new pod + 99 old pods
```

**Key point:** Traffic weight cannot be cleanly decoupled from the physical replica count using only the standard Deployment approach shown in the training material.

---

## Question 3: What happens if a Readiness Probe passes but the application is functionally broken?

**Question:** A Deployment uses a Readiness Probe that checks:

```text
GET /health
```

The endpoint returns:

```text
200 OK
```

However, the application cannot connect to the database, causing user transactions to fail. How does a standard Deployment update respond?

**Answer:** The rollout proceeds because the Readiness Probe is passing with HTTP `200`.

According to the training material, the Deployment controller relies on the Readiness Probe. Because `/health` returns success, Kubernetes treats the pods as ready and continues the rollout, potentially replacing the stable pods with the broken version that cannot access the database.

```text
/health -> 200 OK
      |
      v
Readiness Probe passes
      |
      v
Rollout continues
      |
      v
Old pods can be replaced by new pods
```

---

## Question 4: What is the technical reality of a Kubernetes Deployment rollback?

**Question:** What is the technical reality of performing a "Rollback" on a standard Kubernetes Deployment?

**Answer:** A rollback is essentially another rolling update in reverse. The controller incrementally replaces the current pods with pods defined by a previous revision configuration.

The rollback is not an instantaneous restoration. The controller starts another rolling update whose target state is the previous revision, and the replacement process takes time.

```text
Current revision
      |
      | rollback
      v
Previous revision becomes target state
      |
      v
Rolling replacement of pods
      |
      v
Previous revision restored incrementally
```

---

## Question 5: What safety capability is missing from standard Deployments but available in progressive delivery tools?

**Question:** Which critical safety feature is missing from standard Kubernetes Deployments but is provided by progressive delivery tools such as Argo Rollouts?

**Answer:** The ability to query external metrics, such as Prometheus success rates, during an update and automatically abort the rollout when business or operational KPIs degrade.

The training material explains that standard Deployments cannot query external systems such as Prometheus or Datadog to verify business metrics such as error rates during an update. Advanced progressive delivery tools can pause and query those metrics before proceeding.

```text
Standard Deployment
        |
        v
Relies on readiness information

Progressive Delivery
        |
        v
Can evaluate external metrics
        |
   +----+----+
   |         |
 Good      Bad KPI
   |         |
Continue    Abort
```

---

# Quick Answer Sheet

```text
Q1: All-or-nothing limitation?
    -> Rollout can proceed to completion without automatic decisions based on complex metrics.

Q2: Why is precise traffic splitting difficult?
    -> Traffic distribution is tied to replica counts.

Q3: /health returns 200 but database access is broken?
    -> Readiness passes and the rollout continues.

Q4: What is a Deployment rollback technically?
    -> Another rolling update targeting a previous revision.

Q5: What major progressive-delivery safety feature is missing?
    -> External-metric/KPI analysis with automatic rollout abort.
```

---

# Quick Memory Diagram

```text
Standard Kubernetes Deployment
            |
   +--------+--------+---------+---------+
   |                 |         |         |
   v                 v         v         v
Rolling update    Traffic   Readiness  Rollback
   |              tied to      Probe      |
   |              replicas      |         |
   v                 |          v         v
No advanced       Hard to     Passing   Reverse
KPI-based         express     probe can  rolling
abort logic       tiny %      miss app   update
                             failures

Progressive delivery adds advanced metric-based
checks and can stop a rollout when KPIs degrade.
```
