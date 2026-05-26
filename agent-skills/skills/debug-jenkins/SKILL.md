---
name: jenkins-debug-skill
description: Professional DevSecOps skill for debugging Jenkins CI/CD pipelines, Groovy DSL syntax errors, and containerized build environment failures.
version: 1.0.0
author: Senior DevSecOps Engineer
tags:
  - jenkins
  - devsecops
  - groovy
  - cicd
  - debugging
  - mcp
---

# Jenkins Pipeline Debugging Skill

Use this skill when a user asks to investigate, troubleshoot, or fix a failing Jenkins pipeline, analyze Jenkinsfile syntax, debug Groovy DSL errors, or isolate pipeline runtime environment bugs such as Kubernetes or Docker agent failures.

## Capability Integration (MCP Setup)

Before parsing logs manually, checking for available Model Context Protocol (MCP) servers is mandatory:
1. Jenkins MCP Server: If available, use tools like get_build_log, get_pipeline_node, or query_build_status to fetch the exact console output and pipeline structure.
2. File System MCP: If debugging locally or via an attached workspace, use read_file to fetch the exact Jenkinsfile or shared library code instead of asking the user to paste it.

---

## The Jenkins Debug Mantra

When investigating a failure, the agent must follow these strict analytical steps:

1. Isolate the Failure Layer: Determine if the crash is due to:
   - Syntax or Parsing Error: Groovy CPS constraints, missing closures, invalid DSL keywords.
   - Environment or Infrastructure Error: Jenkins Agent offline, Docker container pull failure, Kubernetes pod OOMKilled, missing tool definitions.
   - Application or Build Error: Unit test failures, compilation errors, static analysis blockages.
   - Security or Permission Error: Rejected sandbox signatures, missing credentials, Vault or Secrets fetch timeout.

2. Verify over Guessing: Never guess syntax or variables. Cross-reference with standard Jenkins step definitions and Groovy specification.

---

## Core Reference Rules and Syntaxes

### 1. Safe Execution and Sandbox Approvals
- Groovy in Jenkins runs inside a script security sandbox. Non-whitelisted methods (such as standard Java I/O or System.currentTimeMillis()) will throw a RejectedAccessException.
- Fix Pattern: Advise the user to approve the signature in Manage Jenkins, then In-process Script Approval, or rewrite using Jenkins Native DSL alternatives (for example, use readFile instead of new File().text).

### 2. Built-in Environment and Context Variables
Ensure all referenced global variables adhere to Jenkins rules:
- env.BUILD_NUMBER (String): Current build sequence number.
- currentBuild.currentResult: Valid values are SUCCESS, UNSTABLE, FAILURE, ABORTED.
- params.YOUR_PARAM_NAME: Safely accesses pipeline parameter values.

### 3. Declarative vs Scripted Context
- Do not mix Declarative structure (pipeline context) with raw Scripted code outside a script block.
- Environment block constraint: In Declarative Jenkins, variables assigned inside an environment block cannot dynamically evaluate shell script execution results unless using the sh(returnStdout: true) pattern correctly wrapped in a string interpolation.

---

## Actionable Debugging Workflows

### Phase 1: Context Gathering (Discovery and Diagnostics)
If the user provides an error, run these diagnostic validations:

```groovy
// 1. Check for the common "CPS transformation" issue when using non-serializable objects (e.g., Iterators, Regex Matchers)
@NonCPS
def parseDataWithRegex(String text) {
    // Non-serializable logic must be isolated inside a @NonCPS annotated method
    def matcher = text =~ /pattern/
    return matcher ? matcher[0] : null
}

// 2. Validate conditional blocks inside Declarative Pipeline
stage('Deploy') {
    when {
        expression { return params.DEPLOY_ENV == 'production' } // Ensure 'return' keyword is inside expression closure
    }
    steps {
        echo "Deploying..."
    }
}
```

### Phase 2: Root Cause Analysis (RCA) Guide

| Error Signature | Potential Root Cause | Verified Resolution Pattern |
| :--- | :--- | :--- |
| java.io.NotSerializableException | Using non-serializable Java objects within normal pipeline steps across serialization checkpoints. | Move the logic into a helper method marked with @NonCPS. |
| org.jenkinsci.plugins...NoSuchMethodError | Mismatch between pipeline syntax and installed plugin version, or invalid syntax inside declarative stage. | Change the step keyword to match the exact plugin documentation or wrap scripted segments inside a script block. |
| WorkflowScript: Method closures not supported | Declaring a dynamic closure or function inside a declarative steps block without a script wrapper. | Encapsulate the function in a Jenkins Shared Library or inside a script block. |
| Exit code 137 | Container agent or build step was killed by the OS or Kernel due to Out of Memory. | Increase the memory request or limit in the Kubernetes pod template or Docker resource allocation parameters. |

### Phase 3: Fix Verification Protocol
Before returning any fixed Jenkinsfile code snippet to the user, the agent must check:
- Is every opening brace balanced with a corresponding closing brace?
- Are pipeline parameters accessed via params.NAME instead of just NAME to avoid variable scoping conflicts?
- If the block is within a declarative pipeline, are shell executions placed exclusively inside a steps block, or wrapped inside script block if complex logic is required?
- Are string variables utilizing double quotes if Groovy string interpolation (${env.VARIABLE}) is needed? Single quotes must be used for literal strings only.