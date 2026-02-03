# Salesforce CI/CD Pipeline Troubleshooting Guide

## Table of Contents
1. [Quick Reference](#quick-reference)
2. [Common Issues & Solutions](#common-issues--solutions)
3. [Failure Categories](#failure-categories)
4. [Diagnostic Commands](#diagnostic-commands)
5. [Rollback Procedures](#rollback-procedures)
6. [Monitoring & Metrics](#monitoring--metrics)

---

## Quick Reference

### Pipeline Health Checklist
- [ ] Jenkins server is running
- [ ] Git repository is accessible
- [ ] Salesforce org is reachable
- [ ] JWT authentication is valid
- [ ] Required plugins are installed (sfdx-git-delta, sfdx-scanner)
- [ ] Sufficient disk space available

### Key URLs
- Jenkins Dashboard: http://localhost:8080
- Build Logs: `${BUILD_URL}/console`
- SCA Reports: `${BUILD_URL}/artifact/sca-reports/`
- GitHub Repository: https://github.com/manishamondal/salesforce-full

### Emergency Contacts
- DevOps Lead: [Name/Email]
- Salesforce Admin: [Name/Email]
- On-call Engineer: [Name/Email]

---

## Common Issues & Solutions

### Issue 1: Static Code Analysis - Critical/High Violations

**Symptom**:
```
❌ FAILED: X Critical issues found (threshold: 0)
⚠️ WARNING: Critical security/quality issues detected!
```

**Root Causes**:
- New code introduced security vulnerabilities
- Code complexity exceeds thresholds
- Hardcoded credentials or sensitive data
- SOQL/DML in loops

**Resolution Steps**:
1. Review SCA report: `${BUILD_URL}/artifact/sca-reports/sca-report.html`
2. Identify specific violations:
   ```bash
   cat sca-reports/sca-report.csv | grep -i "critical\|high"
   ```
3. Fix code issues based on recommendations
4. Run local scan before committing:
   ```bash
   sf scanner run --target force-app --format table
   ```
5. Commit fixes and re-run pipeline

**Prevention**:
- Enable pre-commit hooks for local scanning
- Code review checklist includes security best practices
- Developer training on secure coding

**Related RCA**: RCA-2026-XX-XX-001

---

### Issue 2: Authentication Failure

**Symptom**:
```
ERROR: Authentication failed for user manisha.mondal@accenture.com
```

**Root Causes**:
- Expired JWT certificate
- Invalid Connected App configuration
- Network connectivity issues
- Incorrect credentials in Jenkins

**Resolution Steps**:
1. Verify JWT key file:
   ```bash
   # Check if JWT_KEY_FILE exists and is valid
   ls -la $JWT_KEY_FILE
   ```
2. Test authentication manually:
   ```bash
   sf org login jwt \
     --client-id "$CLIENT_ID" \
     --jwt-key-file "$JWT_KEY_FILE" \
     --username "manisha.mondal@accenture.com" \
     --instance-url "https://login.salesforce.com"
   ```
3. Verify Connected App settings in Salesforce
4. Regenerate JWT certificate if expired
5. Update Jenkins credentials

**Prevention**:
- Set calendar reminders for certificate expiry (typically 1-2 years)
- Monitor authentication failures
- Keep backup certificates

**Related RCA**: RCA-2026-XX-XX-002

---

### Issue 3: Test Failures (RunLocalTests)

**Symptom**:
```
ERROR: Deployment failed with test failures
Code coverage: XX% (Required: 75%)
```

**Root Causes**:
- New code not covered by tests
- Tests broken by recent changes
- Test data setup issues
- Governor limit exceptions in tests

**Resolution Steps**:
1. Identify failing tests from logs:
   ```bash
   grep "Test failure" logs/deployment-failure.log
   ```
2. Run tests locally:
   ```bash
   sf apex run test --test-level RunLocalTests --target-org CICD_QAHub --wait 10
   ```
3. Fix failing tests or add missing test coverage
4. Verify code coverage:
   ```bash
   sf apex get test --test-run-id XXXXXX --code-coverage
   ```
5. Re-run deployment

**Prevention**:
- Require 80%+ code coverage for new classes
- Run tests locally before pushing
- Maintain test data factory classes
- Include tests in code review

**Related RCA**: RCA-2026-XX-XX-003

---

### Issue 4: Delta Deployment - Invalid Baseline

**Symptom**:
```
ERROR: Git ref 'origin/main' not found
fatal: bad revision 'origin/main'
```

**Root Causes**:
- Baseline branch doesn't exist
- Git repository not fetched properly
- Incorrect DELTA_BASELINE parameter

**Resolution Steps**:
1. List available branches:
   ```bash
   git branch -a
   ```
2. Fetch latest from remote:
   ```bash
   git fetch origin
   ```
3. Verify baseline exists:
   ```bash
   git rev-parse --verify origin/main
   ```
4. Update DELTA_BASELINE parameter or use commit SHA
5. Re-run pipeline

**Prevention**:
- Use commit SHAs instead of branch names for stability
- Validate baseline in pipeline before delta generation
- Document branching strategy

**Related RCA**: RCA-2026-XX-XX-004

---

### Issue 5: Metadata API Conflicts

**Symptom**:
```
ERROR: Component already exists
ERROR: Cannot delete referenced component
```

**Root Causes**:
- Metadata conflicts between source and target org
- Destructive changes not handled properly
- Custom metadata type dependencies

**Resolution Steps**:
1. Review conflict details in logs
2. Use `--ignore-conflicts` flag (already in pipeline)
3. For complex conflicts, deploy in stages:
   - First: Remove dependencies
   - Second: Deploy main changes
   - Third: Recreate dependencies
4. Consider FULL deployment instead of DELTA

**Prevention**:
- Maintain org synchronization
- Use scratch orgs for development
- Document metadata dependencies
- Use package.xml carefully

**Related RCA**: RCA-2026-XX-XX-005

---

### Issue 6: Jenkins Plugin Issues

**Symptom**:
```
ERROR: sfdx-git-delta plugin not found
ERROR: Unknown command: scanner
```

**Root Causes**:
- Plugin not installed
- Plugin installation failed
- SF CLI version incompatibility

**Resolution Steps**:
1. Check installed plugins:
   ```bash
   sf plugins
   ```
2. Install missing plugins:
   ```bash
   echo y | sf plugins install sfdx-git-delta
   sf plugins install @salesforce/sfdx-scanner
   ```
3. Update SF CLI:
   ```bash
   sf update
   ```
4. Verify installation:
   ```bash
   sf plugins --core
   ```

**Prevention**:
- Pin plugin versions in Jenkins configuration
- Regular plugin updates
- Monitor plugin compatibility

**Related RCA**: RCA-2026-XX-XX-006

---

## Failure Categories

### Category: CODE_QUALITY (15% of failures)
- **Indicators**: SCA violations, PMD errors
- **Common Causes**: Hardcoded credentials, SOQL in loops, complexity
- **Avg Resolution Time**: 2-4 hours
- **Owner**: Development Team

### Category: VALIDATION (25% of failures)
- **Indicators**: Metadata validation errors
- **Common Causes**: Missing dependencies, API version mismatches
- **Avg Resolution Time**: 1-2 hours
- **Owner**: DevOps Engineer

### Category: TEST_FAILURE (30% of failures)
- **Indicators**: Test failures, low code coverage
- **Common Causes**: Broken tests, missing test data
- **Avg Resolution Time**: 3-6 hours
- **Owner**: Development Team

### Category: AUTHENTICATION (10% of failures)
- **Indicators**: JWT errors, connection timeouts
- **Common Causes**: Expired certificates, network issues
- **Avg Resolution Time**: 30 min - 1 hour
- **Owner**: DevOps/Admin

### Category: CONFIGURATION (20% of failures)
- **Indicators**: Parameter errors, path not found
- **Common Causes**: Incorrect parameters, missing files
- **Avg Resolution Time**: 30 min - 1 hour
- **Owner**: DevOps Engineer

---

## Diagnostic Commands

### Check SF CLI Installation
```bash
sf --version
sf plugins
```

### Test Org Connection
```bash
sf org display --target-org CICD_QAHub --json
sf org list --json
```

### List Recent Deployments
```bash
sf org list metadata-types --target-org CICD_QAHub
```

### View Deployment Status
```bash
sf deploy metadata report --job-id <DEPLOYMENT_ID> --target-org CICD_QAHub
```

### Run Code Analysis Locally
```bash
sf scanner run --target force-app --format html,csv --outfile local-sca-report
```

### Check Test Results
```bash
sf apex run test --test-level RunLocalTests --target-org CICD_QAHub --json
```

### Generate Delta Preview
```bash
sf sgd:source:delta --from origin/main --to HEAD --output delta --generate-delta
ls -la delta/
```

### View Jenkins Logs
```bash
# In Jenkins server
tail -f /var/log/jenkins/jenkins.log

# Or from build
curl -u user:token ${BUILD_URL}/consoleText
```

---

## Rollback Procedures

### Scenario 1: Failed Deployment (Not Applied)
**Status**: Deployment failed during validation or dry-run

**Action**: 
- Fix issues and re-run pipeline
- No rollback needed (nothing was deployed)

### Scenario 2: Partial Deployment Success
**Status**: Some components deployed, others failed

**Steps**:
1. Identify deployed components from logs
2. Use Quick Deploy if validation passed:
   ```bash
   sf deploy metadata quick --job-id <VALIDATION_ID> --target-org CICD_QAHub
   ```
3. Or revert Git changes:
   ```bash
   git revert HEAD
   git push origin main
   ```
4. Re-run full deployment

### Scenario 3: Successful Deployment, Post-Issues Discovered
**Status**: Deployment successful but issues found in production

**Steps**:
1. Emergency hotfix:
   - Create hotfix branch from previous good commit
   - Deploy hotfix via pipeline with APEX_CLASSES parameter
2. Full rollback:
   ```bash
   # Revert to previous commit
   git revert HEAD
   git push origin main
   
   # Deploy reverted code
   # In Jenkins: Set DEPLOY_SCOPE=FULL, TEST_LEVEL=NoTestRun (if emergency)
   ```
3. Validate rollback:
   ```bash
   sf org display --target-org CICD_QAHub
   ```

### Scenario 4: Database/Configuration Rollback
**Status**: Metadata deployed but need to revert data changes

**Steps**:
1. Use Salesforce Change Sets for manual rollback
2. Or deploy destructive changes:
   ```xml
   <!-- destructiveChanges.xml -->
   <?xml version="1.0" encoding="UTF-8"?>
   <Package xmlns="http://soap.sforce.com/2006/04/metadata">
       <types>
           <members>ComponentToDelete</members>
           <name>ApexClass</name>
       </types>
       <version>59.0</version>
   </Package>
   ```
3. Deploy with destructive changes:
   ```bash
   sf deploy metadata --manifest package.xml --pre-destructive-changes destructiveChanges.xml
   ```

---

## Monitoring & Metrics

### Key Performance Indicators (KPIs)

#### 1. Pipeline Failure Rate
**Target**: <15% avoidable failures

**Tracking**:
```bash
# Calculate from logs
total_builds=$(grep -c "Build Number:" logs/pipeline-summary.txt)
failed_builds=$(grep -c "Status: FAILURE" logs/pipeline-summary.txt)
failure_rate=$((failed_builds * 100 / total_builds))
echo "Failure Rate: ${failure_rate}%"
```

**Dashboard Location**: Jenkins Dashboard > Build Statistics

#### 2. Deployment Duration
**Target**: <30 minutes for DELTA, <60 minutes for FULL

**Tracking**: Available in `logs/build-metrics-summary.txt`

#### 3. Code Quality Trends
**Target**: 0 Critical/High issues for 2 consecutive cycles

**Tracking**: Review `sca-reports/summary.txt` from last 10 builds

#### 4. Test Coverage
**Target**: >75% org-wide, >80% new code

**Tracking**: From Salesforce org or test results

### Weekly Review Checklist
- [ ] Review failure logs and categorize
- [ ] Update RCA documents for recurring issues
- [ ] Check certificate expiry dates
- [ ] Verify plugin versions
- [ ] Review and optimize pipeline performance
- [ ] Update documentation based on lessons learned

### Monthly Tasks
- [ ] Generate failure trends report
- [ ] Review and update thresholds (SCA_THRESHOLD_HIGH/CRITICAL)
- [ ] Team training on common issues
- [ ] Pipeline optimization review
- [ ] Disaster recovery drill

---

## Escalation Matrix

| Severity | Response Time | Escalation Level | Contact |
|----------|---------------|------------------|---------|
| Critical (Prod Down) | 15 min | Manager + On-call | [Contact] |
| High (Deployment Blocked) | 1 hour | Team Lead | [Contact] |
| Medium (Intermittent Failures) | 4 hours | Assigned Engineer | [Contact] |
| Low (Documentation/Enhancement) | 2 days | Backlog | [Contact] |

---

## Additional Resources

### Documentation
- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/)
- [SF CLI Command Reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/)
- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)

### Tools
- Salesforce Code Analyzer: https://forcedotcom.github.io/sfdx-scanner/
- Git Delta Plugin: https://github.com/scolladon/sfdx-git-delta

### Training
- Salesforce Trailhead: CI/CD modules
- Jenkins certification courses
- Security best practices for Salesforce

---

**Document Version**: 1.0  
**Last Updated**: February 4, 2026  
**Owner**: DevOps Team  
**Review Frequency**: Monthly
