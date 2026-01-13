# Exercise 4: Advanced Pipeline Features

Welcome to the advanced OpenShift Pipelines exercise! This exercise builds upon the concepts from exercises 1-3 and introduces advanced Tekton features including parallel task execution, conditional logic, multiple workspaces, and finally tasks.

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Concepts Covered](#concepts-covered)
4. [Architecture](#architecture)
5. [Step-by-Step Instructions](#step-by-step-instructions)
6. [Understanding the Pipeline](#understanding-the-pipeline)
7. [Running Different Scenarios](#running-different-scenarios)
8. [Advanced Topics](#advanced-topics)
9. [Troubleshooting](#troubleshooting)
10. [Next Steps](#next-steps)

## Overview

This exercise demonstrates a production-ready CI/CD pipeline that includes:

- **Parallel Task Execution**: Run code quality checks, security scans, and tests simultaneously
- **Conditional Logic**: Control task execution based on parameters using `when` expressions
- **Multiple Workspaces**: Separate workspaces for source code, caching, and test results
- **Finally Tasks**: Notifications that run regardless of pipeline success or failure
- **Advanced Parameters**: Fine-grained control over pipeline behavior

### What You'll Build

A comprehensive pipeline that:
1. Fetches source code from Git
2. Runs parallel validation (code quality, security, tests)
3. Builds a container image
4. Conditionally deploys to different environments
5. Sends notifications about execution status

## Prerequisites

Before starting this exercise, ensure you have completed:

- **Exercise 1**: Basic pipeline and tasks
- **Exercise 2**: PipelineRuns
- **Exercise 3**: Triggers (optional but recommended)

You should have:
- OpenShift Pipelines Operator installed
- Access to an OpenShift cluster
- `oc` CLI tool installed
- `tkn` CLI tool installed
- A project/namespace to work in

### Task Dependencies

This pipeline depends on tasks from Exercise 1 (`01_pipeline` directory):
- `apply-manifests` - Used for deploying Kubernetes manifests
- `update-deployment` - Used for updating deployment images

These tasks must be installed in your namespace before running the pipeline. See [Step 3](#step-3-install-tasks-from-previous-exercises) for installation instructions.

## Concepts Covered

### 1. Parallel Task Execution

Tasks can run simultaneously when they don't depend on each other, reducing overall pipeline execution time.

```yaml
# All three tasks run in parallel after fetch-repository
- name: code-quality-check
  runAfter:
  - fetch-repository

- name: security-scan
  runAfter:
  - fetch-repository

- name: run-tests
  runAfter:
  - fetch-repository
```

### 2. Conditional Task Execution

Use `when` expressions to control whether a task runs based on parameter values.

```yaml
- name: security-scan
  when:
  - input: "$(params.enable-security-scan)"
    operator: in
    values: ["true", "True", "TRUE"]
```

### 3. Multiple Workspaces

Different workspaces serve different purposes:
- **shared-workspace**: Source code and build artifacts
- **cache-workspace**: Dependency caching for faster builds
- **test-results-workspace**: Test reports and coverage data

### 4. Finally Tasks

Tasks in the `finally` section run regardless of pipeline success or failure, perfect for cleanup and notifications.

```yaml
finally:
- name: send-notification
  taskRef:
    name: send-notification
  params:
  - name: pipeline-status
    value: $(tasks.status)
```

### 5. Task Results

Tasks can output results that other tasks can consume:

```yaml
results:
- name: quality-score
  description: Code quality score (0-100)
```

## Architecture

### Pipeline Flow Diagram

```
┌─────────────────────┐
│ fetch-repository    │
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    │  Parallel   │
    │  Execution  │
    └──────┬──────┘
           │
    ┌──────┴──────────────────────────┐
    │                                  │
┌───▼────────────┐  ┌────────────┐  ┌▼──────────┐
│ code-quality   │  │ security-  │  │ run-tests │
│ check          │  │ scan       │  │           │
└───┬────────────┘  └─────┬──────┘  └┬──────────┘
    │                     │            │
    └──────────┬──────────┴────────────┘
               │
        ┌──────▼──────┐
        │ build-image │
        └──────┬──────┘
               │
        ┌──────▼──────────┐
        │ Conditional     │
        │ Deployment      │
        └──────┬──────────┘
               │
    ┌──────────┴──────────┐
    │                     │
┌───▼──────────┐  ┌───────▼────────┐
│ apply-       │  │ update-        │
│ manifests    │  │ deployment     │
└───┬──────────┘  └───────┬────────┘
    │                     │
    └──────────┬──────────┘
               │
        ┌──────▼──────────┐
        │ conditional-    │
        │ deploy          │
        └─────────────────┘
               │
        ┌──────▼──────────┐
        │ Finally:        │
        │ send-           │
        │ notification    │
        └─────────────────┘
```

## Step-by-Step Instructions

### Step 1: Create Your Project

```bash
# Create a new project for this exercise
oc new-project advanced-pipelines-demo

# Or switch to an existing project
oc project your-project-name
```

### Step 2: Install the Custom Tasks

Install all the custom tasks required for this pipeline:

```bash
# Navigate to the exercise directory
cd 04_advanced_pipeline

# Create the code quality check task
oc create -f 01_code_quality_task.yaml

# Create the security scan task
oc create -f 02_security_scan_task.yaml

# Create the run tests task
oc create -f 03_run_tests_task.yaml

# Create the conditional deploy task
oc create -f 04_conditional_deploy_task.yaml

# Create the notification task
oc create -f 05_notification_task.yaml
```

Verify the tasks were created:

```bash
tkn task list
```

You should see:
```
NAME                    DESCRIPTION                                              AGE
code-quality-check      This task performs code quality checks...                1 minute ago
security-scan           This task performs security vulnerability scanning...    1 minute ago
run-tests               This task runs unit tests and integration tests...       1 minute ago
conditional-deploy      This task performs conditional deployment...             1 minute ago
send-notification       This task sends notifications about pipeline...          1 minute ago
```

### Step 3: Install Tasks from Previous Exercises

This pipeline also uses tasks from Exercise 1:

```bash
# If not already installed, create these tasks
oc create -f ../01_pipeline/01_apply_manifest_task.yaml
oc create -f ../01_pipeline/02_update_deployment_task.yaml
```

### Step 4: Create the Advanced Pipeline

```bash
oc create -f 06_advanced_pipeline.yaml
```

Verify the pipeline was created:

```bash
tkn pipeline describe advanced-build-and-deploy
```

### Step 5: Run Your First Pipeline

Let's start with a development build (fastest feedback):

```bash
oc create -f 08_pipelinerun_development.yaml
```

Watch the pipeline execution:

```bash
# Using tkn CLI
tkn pipelinerun logs advanced-build-deploy-development -f

# Or watch in the OpenShift Console
# Navigate to: Pipelines → PipelineRuns → advanced-build-deploy-development
```

## Understanding the Pipeline

### Pipeline Parameters

The pipeline accepts numerous parameters for fine-grained control:

#### Git Parameters
- `git-url`: Repository URL
- `git-revision`: Branch, tag, or commit SHA (default: "main")

#### Build Parameters
- `IMAGE`: Container image reference
- `deployment-name`: Name of the Kubernetes deployment

#### Quality & Security Parameters
- `language`: Programming language (nodejs, python, java, go)
- `enable-security-scan`: Enable/disable security scanning (default: "true")
- `run-tests`: Enable/disable test execution (default: "true")
- `test-coverage-threshold`: Minimum coverage percentage (default: "80")

#### Deployment Parameters
- `deploy-to-production`: Whether to deploy (default: "false")
- `environment`: Target environment (dev, staging, production)
- `require-approval`: Require manual approval (default: "false")

#### Notification Parameters
- `notification-webhook`: Webhook URL for notifications

### Task Descriptions

#### 1. code-quality-check
Performs linting and code style validation. Supports multiple languages and can be configured to fail on warnings.

**Results:**
- `quality-score`: Score from 0-100
- `issues-found`: Number of issues detected

#### 2. security-scan
Scans for security vulnerabilities in dependencies and code. Uses severity thresholds to determine failures.

**Results:**
- `vulnerabilities-found`: Total vulnerabilities
- `critical-count`: Critical vulnerabilities
- `scan-status`: passed/failed

#### 3. run-tests
Executes unit and integration tests with coverage reporting.

**Results:**
- `test-status`: passed/failed
- `tests-passed`: Number of passing tests
- `tests-failed`: Number of failing tests
- `coverage-percentage`: Test coverage percentage

#### 4. conditional-deploy
Advanced deployment with environment validation and approval workflows.

**Results:**
- `deployment-status`: deployed/skipped/failed
- `deployed-to`: Target environment

#### 5. send-notification
Sends notifications via console, Slack, email, or webhook. Runs in the `finally` section.

**Results:**
- `notification-sent`: true/false

## Running Different Scenarios

### Scenario 1: Production Deployment

Full validation with all checks enabled and deployment to production:

```bash
oc create -f 07_pipelinerun_production.yaml
```

**Features:**
- All security scans enabled
- High test coverage threshold (80%)
- Deployment to production environment
- Approval required
- All workspaces allocated

**Use Case:** Release to production

### Scenario 2: Development Build

Fast feedback loop for development:

```bash
oc create -f 08_pipelinerun_development.yaml
```

**Features:**
- Security scan disabled for speed
- Lower coverage threshold (70%)
- Deploy to dev environment
- No approval required
- Minimal workspace allocation

**Use Case:** Rapid development iteration

### Scenario 3: Pull Request Validation

Comprehensive validation without deployment:

```bash
oc create -f 09_pipelinerun_pr_validation.yaml
```

**Features:**
- All checks enabled
- No deployment
- Test results saved
- Suitable for CI validation

**Use Case:** PR/MR validation before merge

### Scenario 4: Custom Configuration

Create your own PipelineRun with custom parameters:

```bash
cat <<EOF | oc create -f -
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: my-custom-run
spec:
  pipelineRef:
    name: advanced-build-and-deploy
  params:
  - name: git-url
    value: https://github.com/YOUR-ORG/YOUR-REPO.git
  - name: git-revision
    value: feature-branch
  - name: deployment-name
    value: my-app
  - name: IMAGE
    value: image-registry.openshift-image-registry.svc:5000/$(context.pipelineRun.namespace)/my-app
  - name: language
    value: python
  - name: enable-security-scan
    value: "true"
  - name: deploy-to-production
    value: "false"
  workspaces:
  - name: shared-workspace
    volumeClaimTemplate:
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 500Mi
EOF
```

## Advanced Topics

### When Expressions

When expressions provide powerful conditional logic:

```yaml
when:
- input: "$(params.enable-security-scan)"
  operator: in
  values: ["true", "True", "TRUE"]
```

**Operators:**
- `in`: Input matches one of the values
- `notin`: Input doesn't match any values

**Use Cases:**
- Skip expensive tasks in development
- Deploy only from specific branches
- Run security scans only for production

### Workspace Management

**Best Practices:**
- Use `volumeClaimTemplate` for dynamic provisioning
- Mark optional workspaces with `optional: true`
- Size workspaces appropriately for your needs
- Consider using cache workspaces for dependencies

**Example:**
```yaml
workspaces:
- name: cache-workspace
  optional: true
  volumeClaimTemplate:
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 500Mi
```

### Task Results and Parameters

Tasks can pass data to subsequent tasks:

```yaml
# In task definition
results:
- name: quality-score
  description: Code quality score

# In pipeline
- name: next-task
  params:
  - name: previous-score
    value: $(tasks.code-quality-check.results.quality-score)
```

### Finally Tasks

Finally tasks always run, making them perfect for:
- Sending notifications
- Cleaning up resources
- Collecting metrics
- Updating external systems

**Special Variables:**
- `$(tasks.status)`: Overall pipeline status
- `$(tasks.TASKNAME.status)`: Individual task status

## Troubleshooting

### Common Issues

#### Issue 1: Task Not Found

**Error:**
```
error: couldn't get task "code-quality-check": tasks.tekton.dev "code-quality-check" not found
```

**Solution:**
```bash
# Verify tasks are created
tkn task list

# Create missing tasks
oc create -f 01_code_quality_task.yaml
```

#### Issue 2: Workspace Volume Claim Fails

**Error:**
```
Failed to create PVC: exceeded quota
```

**Solution:**
- Reduce workspace size in PipelineRun
- Use existing PVC instead of volumeClaimTemplate
- Check namespace resource quotas

```bash
# Check quotas
oc describe quota

# Use smaller storage
# Edit PipelineRun and change storage: 1Gi to storage: 500Mi
```

#### Issue 3: Parallel Tasks Not Running

**Symptom:** Tasks run sequentially instead of in parallel

**Cause:** Tasks have unnecessary `runAfter` dependencies

**Solution:** Ensure parallel tasks only depend on common prerequisites:

```yaml
# Correct - runs in parallel
- name: task-a
  runAfter: [fetch-repository]
- name: task-b
  runAfter: [fetch-repository]

# Incorrect - runs sequentially
- name: task-a
  runAfter: [fetch-repository]
- name: task-b
  runAfter: [task-a]  # Creates unnecessary dependency
```

#### Issue 4: When Expression Not Working

**Symptom:** Task runs when it shouldn't (or vice versa)

**Common Mistakes:**
- Case sensitivity: "true" vs "True" vs "TRUE"
- String vs boolean: Use string "true" not boolean true
- Operator confusion: `in` vs `notin`

**Solution:**
```yaml
# Handle multiple cases
when:
- input: "$(params.enable-feature)"
  operator: in
  values: ["true", "True", "TRUE", "yes", "Yes"]
```

#### Issue 5: Finally Task Not Receiving Status

**Error:**
```
$(tasks.status) is empty
```

**Solution:** Ensure you're using the correct variable syntax:

```yaml
finally:
- name: notify
  params:
  - name: status
    value: $(tasks.status)  # Correct
    # NOT: $(params.status) or $(results.status)
```

### Debugging Tips

1. **Check PipelineRun Status:**
```bash
tkn pipelinerun describe <pipelinerun-name>
```

2. **View Task Logs:**
```bash
tkn pipelinerun logs <pipelinerun-name> -t <task-name>
```

3. **List All Resources:**
```bash
tkn pipelinerun list
tkn taskrun list
```

4. **Check Events:**
```bash
oc get events --sort-by='.lastTimestamp'
```

5. **Inspect PVC:**
```bash
oc get pvc
oc describe pvc <pvc-name>
```

## Next Steps

### Extend the Pipeline

1. **Add More Validation Tasks:**
   - License compliance checking
   - Container image scanning
   - Performance testing
   - Accessibility testing

2. **Implement Approval Gates:**
   - Manual approval tasks
   - Integration with approval systems
   - Slack-based approvals

3. **Add Monitoring:**
   - Prometheus metrics
   - Custom dashboards
   - Alert integration

4. **Enhance Notifications:**
   - Rich Slack messages with buttons
   - Email with HTML formatting
   - Integration with incident management

### Integration Ideas

1. **GitOps Integration:**
   - Update GitOps repository after build
   - Trigger ArgoCD sync
   - Create pull requests for manifest updates

2. **Security Integration:**
   - Integrate with Snyk, Aqua, or Twistlock
   - Policy enforcement with OPA
   - Vulnerability database updates

3. **Testing Integration:**
   - SonarQube for code quality
   - JMeter for performance testing
   - Selenium for UI testing

4. **Artifact Management:**
   - Push to Artifactory or Nexus
   - Generate SBOMs
   - Sign container images

### Learning Resources

- [Tekton Documentation](https://tekton.dev/docs/)
- [OpenShift Pipelines Documentation](https://docs.openshift.com/container-platform/latest/cicd/pipelines/understanding-openshift-pipelines.html)
- [Tekton Catalog](https://hub.tekton.dev/)
- [Tekton Best Practices](https://tekton.dev/docs/pipelines/pipelines/#best-practices)

### Practice Exercises

1. **Modify the pipeline** to add a new validation task
2. **Create a staging environment** deployment scenario
3. **Implement a rollback mechanism** using finally tasks
4. **Add metrics collection** to track pipeline performance
5. **Create a custom task** for your specific use case

## Conclusion

Congratulations! You've completed the advanced pipeline exercise. You now understand:

✅ Parallel task execution for efficiency  
✅ Conditional logic with when expressions  
✅ Multiple workspace management  
✅ Finally tasks for guaranteed execution  
✅ Advanced parameter usage  
✅ Real-world CI/CD patterns  

These concepts form the foundation of production-ready CI/CD pipelines in OpenShift.

---

**Need Help?**
- Check the [Troubleshooting](#troubleshooting) section
- Review the [Tekton Documentation](https://tekton.dev/docs/)
- Ask in the OpenShift community forums

**Feedback:**
If you have suggestions for improving this exercise, please open an issue or pull request!