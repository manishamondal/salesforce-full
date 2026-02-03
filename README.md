# Salesforce CI/CD Pipeline - README

## Overview
Enterprise-grade Salesforce CI/CD pipeline demonstrating responsible end-to-end ownership with Jenkins, GitHub, and Salesforce DX.

## 🎯 Business Objectives

### 1. Jenkins Governance & Pipeline Stability
- **Target**: <15% avoidable pipeline failures
- **Approach**: Proactive monitoring, structured problem handling, comprehensive error logging
- **Deliverable**: 3+ recurring issues with RCA and permanent fixes

### 2. Security & Quality Awareness
- **Target**: 0 Critical/High issues for 2 consecutive deployment cycles
- **Approach**: Embedded Salesforce Code Analyzer with enforced thresholds
- **Deliverable**: Static scan reports in every Jenkins build

## 🏗️ Architecture

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   GitHub    │─────▶│   Jenkins   │─────▶│  Salesforce │
│ Repository  │      │   Pipeline  │      │     Org     │
└─────────────┘      └─────────────┘      └─────────────┘
      │                      │                     │
      │                      ▼                     │
      │              ┌──────────────┐              │
      │              │ Static Code  │              │
      │              │   Analysis   │              │
      │              └──────────────┘              │
      │                      │                     │
      └──────────────────────┴─────────────────────┘
                    Feedback Loop
```

## ✨ Key Features

### 🔐 Security & Quality Controls
- ✅ **Universal Static Code Analysis**: Runs for ALL deployment types (FULL, DELTA, APEX_CLASSES)
- ✅ **Enforced Thresholds**: Automatic build failure on Critical/High severity issues
- ✅ **Multi-format Reports**: HTML, JSON, CSV reports archived in Jenkins
- ✅ **Secure Authentication**: JWT-based authentication with credential management

### 🚀 Deployment Capabilities
- ✅ **Multiple Formats**: SOURCE and MDAPI deployments
- ✅ **Flexible Scope**: FULL or DELTA (git-based) deployments
- ✅ **Targeted Deployments**: Deploy specific Apex classes only
- ✅ **Test Levels**: NoTestRun or RunLocalTests
- ✅ **Validation (Dry-run)**: Check deployments before actual execution

### 📊 Monitoring & Governance
- ✅ **Comprehensive Logging**: Stage-wise metrics and timing
- ✅ **Artifact Archiving**: All reports, logs, and metrics preserved
- ✅ **Failure Categorization**: CODE_QUALITY, VALIDATION, TEST_FAILURE, etc.
- ✅ **Build Metrics**: Duration tracking, success rate, trends

### 🔄 Rollback Readiness
- ✅ **Deployment Tracking**: Capture deployment IDs for rollback
- ✅ **Failure Handling**: Detailed error logs with recommended actions
- ✅ **Git-based Rollback**: Revert commits and redeploy
- ✅ **Emergency Procedures**: Documented rollback strategies

### ✅ Production Approval Workflow
- ✅ **Manual Gate**: 30-minute approval window after validation
- ✅ **Deployment Summary**: Clear information before approval
- ✅ **Abort Safety**: Can cancel deployment during approval phase

## 📁 Project Structure

```
salesforce-full/
├── Jenkinsfile                      # Main pipeline definition
├── README.md                        # This file
├── sfdx-project.json               # Salesforce DX project config
├── force-app/                      # Salesforce metadata
│   └── main/default/classes/       # Apex classes
│       ├── AccountService.cls
│       ├── ContactService.cls
│       ├── LeadService.cls
│       ├── OpportunityService.cls
│       └── UtilityService.cls
├── manifest/                       # Package.xml files
│   ├── package-all.xml
│   ├── package-core.xml
│   ├── package-sales.xml
│   └── package-utility.xml
└── docs/                          # Comprehensive documentation
    ├── SETUP_GUIDE.md            # Complete setup instructions
    ├── TROUBLESHOOTING_GUIDE.md  # Common issues and solutions
    ├── RCA_TEMPLATE.md           # Root Cause Analysis template
    └── PIPELINE_METRICS.md       # Metrics tracking and governance
```

## 🚀 Quick Start

### Prerequisites
- Jenkins 2.400+
- Salesforce CLI (sf)
- Git 2.30+
- Java JDK 11 or 17
- Salesforce org with API access
- GitHub repository

### Setup (5 Steps)
1. **Install Tools**: Jenkins, SF CLI, Git
2. **Configure Salesforce**: Create Connected App with JWT
3. **Configure Jenkins**: Add credentials (JWT key, Client ID)
4. **Create Pipeline**: Import Jenkinsfile from GitHub
5. **Run First Build**: Test with DEPLOY_SCOPE=FULL, TEST_LEVEL=NoTestRun

📖 **Detailed Instructions**: See [docs/SETUP_GUIDE.md](docs/SETUP_GUIDE.md)

## 🎮 Usage

### Build Parameters

| Parameter | Options | Description |
|-----------|---------|-------------|
| `DEPLOY_FORMAT` | SOURCE, MDAPI | Deployment format |
| `DEPLOY_SCOPE` | FULL, DELTA | Deploy all or changed files only |
| `TEST_LEVEL` | NoTestRun, RunLocalTests | Salesforce test execution |
| `DELTA_BASELINE` | Git ref | Baseline for delta (e.g., origin/main) |
| `PACKAGE_XML_PATH` | Path | package.xml for MDAPI deployments |
| `APEX_CLASSES` | Class names | Deploy specific classes (comma-separated) |

### Common Scenarios

#### Deploy All Changes
```
DEPLOY_FORMAT: SOURCE
DEPLOY_SCOPE: FULL
TEST_LEVEL: RunLocalTests
```

#### Deploy Only Changes Since Last Release
```
DEPLOY_FORMAT: SOURCE
DEPLOY_SCOPE: DELTA
DELTA_BASELINE: origin/main
TEST_LEVEL: RunLocalTests
```

#### Quick Hotfix (Specific Classes)
```
APEX_CLASSES: AccountService,ContactService
TEST_LEVEL: NoTestRun
```

#### Package-based Deployment
```
DEPLOY_FORMAT: MDAPI
PACKAGE_XML_PATH: manifest/package-core.xml
TEST_LEVEL: RunLocalTests
```

## 📊 Pipeline Stages

```
1. ✓ Checkout                    - Clone Git repository
2. ✓ Verify Git                  - Verify branch and commits
3. ✓ Authenticate to Salesforce  - JWT-based org login
4. ✓ Install Plugins             - sfdx-git-delta, sfdx-scanner
5. ✓ Validate Delta Baseline     - Ensure baseline exists (DELTA only)
6. ✓ Generate Delta              - Create delta package (DELTA only)
7. 🔒 Static Code Analysis       - Scan with enforced thresholds
8. ✓ Validate Deployment         - Dry-run (check-only)
9. ⏸️ Approve Deployment          - Manual approval gate
10. ✓ Deploy                     - Actual deployment
11. ✓ Post-Deployment Validation - Verify deployment success
```

## 🔍 Static Code Analysis

### Configuration
- **Threshold**: 0 Critical, 0 High severity issues (enforced)
- **Execution**: ALL deployment types (not just DELTA)
- **Output Formats**: HTML, JSON, CSV
- **Action on Violation**: Build fails, deployment blocked

### Report Locations
```
${BUILD_URL}/artifact/sca-reports/sca-report.html  # Visual report
${BUILD_URL}/artifact/sca-reports/sca-report.json  # Programmatic access
${BUILD_URL}/artifact/sca-reports/sca-report.csv   # Data analysis
${BUILD_URL}/artifact/sca-reports/summary.txt      # Quick summary
```

### Sample Summary
```
=== Static Code Analysis Summary ===
Build: 42
Date: 2026-02-04 10:30:00
Scope: FULL
Directory Scanned: force-app

Critical Issues: 0
High Issues: 0

✅ PASSED: No Critical/High issues above threshold
```

## 📈 Metrics & Governance

### Tracked Metrics
- **Pipeline Failure Rate**: Target <15% avoidable failures
- **Build Duration**: FULL <60min, DELTA <30min
- **Code Quality**: 0 Critical/High issues for 2+ consecutive builds
- **MTTR**: Mean Time To Recovery by failure category

### Failure Categories
1. **CODE_QUALITY** (15%): SCA violations
2. **VALIDATION** (25%): Metadata errors
3. **TEST_FAILURE** (30%): Apex test failures
4. **AUTHENTICATION** (10%): Connection issues
5. **CONFIGURATION** (20%): Parameter/setup issues

### Reporting
- Weekly metrics review
- Monthly trend analysis
- Quarterly governance assessment

📖 **Details**: See [docs/PIPELINE_METRICS.md](docs/PIPELINE_METRICS.md)

## 🔧 Troubleshooting

### Quick Diagnostics
```bash
# Check SF CLI
sf --version
sf plugins

# Test Org Connection
sf org display --target-org CICD_QAHub

# View Recent Deployments
sf org list metadata-types --target-org CICD_QAHub

# Run Local Code Scan
sf scanner run --target force-app --format table
```

### Common Issues
1. **SCA Violations**: Review HTML report, fix code, rerun
2. **Auth Failure**: Check JWT certificate and Connected App
3. **Test Failures**: Review test results, fix tests, redeploy
4. **Delta Errors**: Verify baseline exists, use commit SHA

📖 **Complete Guide**: See [docs/TROUBLESHOOTING_GUIDE.md](docs/TROUBLESHOOTING_GUIDE.md)

## 🔄 Rollback Procedures

### Scenario 1: Validation Failed
- No action needed (nothing deployed)
- Fix issues and rerun

### Scenario 2: Deployment Failed
- Review failure logs in logs/deployment-failure.log
- Revert Git commit: `git revert HEAD`
- Redeploy previous version

### Scenario 3: Post-Deployment Issues
- Emergency hotfix: Deploy specific fixed classes
- Full rollback: Revert commit and deploy
- Use destructive changes if needed

📖 **Detailed Procedures**: See [docs/TROUBLESHOOTING_GUIDE.md](docs/TROUBLESHOOTING_GUIDE.md)

## 📋 RCA Documentation

### Process
1. Use template: [docs/RCA_TEMPLATE.md](docs/RCA_TEMPLATE.md)
2. Document within 24 hours of incident
3. Include: symptoms, root cause, resolution, prevention
4. Target: 3+ RCAs with permanent fixes

## 🛡️ Security Best Practices

### Implemented
- ✅ JWT-based authentication (no password storage)
- ✅ Credentials managed via Jenkins Credential Manager
- ✅ Static code analysis with security rules
- ✅ No hardcoded credentials (detected by scanner)
- ✅ SOQL injection prevention
- ✅ Secure API communication (HTTPS)

## 📚 Documentation

### Available Guides
1. [SETUP_GUIDE.md](docs/SETUP_GUIDE.md) - Complete installation and configuration
2. [TROUBLESHOOTING_GUIDE.md](docs/TROUBLESHOOTING_GUIDE.md) - Common issues and solutions
3. [RCA_TEMPLATE.md](docs/RCA_TEMPLATE.md) - Root Cause Analysis template
4. [PIPELINE_METRICS.md](docs/PIPELINE_METRICS.md) - Metrics tracking and reporting

### Quick Links
- Jenkins Console: http://localhost:8080
- Build Artifacts: ${BUILD_URL}/artifact/
- GitHub Repo: https://github.com/manishamondal/salesforce-full

## 🎯 Success Criteria

### ✅ Objective 1: Pipeline Stability
- [x] Maintain <15% avoidable failure rate
- [x] Track and categorize all failures
- [x] Document 3+ recurring issues with RCA
- [x] Implement preventive measures

### ✅ Objective 2: Security & Quality
- [x] Integrate Salesforce Code Analyzer
- [x] Enforce 0 Critical/High threshold
- [x] Achieve 2 consecutive clean builds
- [x] Archive reports in Jenkins

### ✅ Objective 3: Production Readiness
- [x] Build and validation stages
- [x] Deployment to target orgs
- [x] Approval workflow (30-min timeout)
- [x] Rollback readiness and failure handling
- [x] Comprehensive monitoring and logging

## Key folders
force-app/  -> Source format
manifest/   -> MDAPI package.xml strategies
mdapi/      -> MDAPI output for delta/full
scripts/    -> Jenkins / shell scripts

Use Git tags as delta baselines.
