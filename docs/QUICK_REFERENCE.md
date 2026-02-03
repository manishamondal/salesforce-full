# 🚀 Quick Reference Card - Salesforce CI/CD Pipeline

> **Print this page for quick desk reference**

---

## 📞 Emergency Contacts
| Role | Name | Contact | Escalation Time |
|------|------|---------|-----------------|
| DevOps Lead | __________ | __________ | Critical: 15 min |
| Team Lead | __________ | __________ | High: 1 hour |
| Salesforce Admin | __________ | __________ | High: 1 hour |
| On-call Engineer | __________ | __________ | Critical: 15 min |

---

## 🔗 Important URLs
```
Jenkins Dashboard:    http://localhost:8080
Build Console:        ${BUILD_URL}/console
Build Artifacts:      ${BUILD_URL}/artifact/
SCA Reports:          ${BUILD_URL}/artifact/sca-reports/sca-report.html
GitHub Repository:    https://github.com/manishamondal/salesforce-full
```

---

## ⚡ Quick Commands

### Check Pipeline Health
```bash
# Test SF CLI
sf --version

# Test Org Connection
sf org display --target-org CICD_QAHub

# View Plugins
sf plugins
```

### Run Local Code Scan
```bash
sf scanner run --target force-app --format table
```

### Manual Deployment
```bash
# Full deployment
sf project deploy start --source-dir force-app --target-org CICD_QAHub

# Specific classes
sf project deploy start --source-dir force-app/main/default/classes/AccountService.cls --target-org CICD_QAHub
```

### Run Tests
```bash
sf apex run test --test-level RunLocalTests --target-org CICD_QAHub --wait 10
```

### Check Recent Deployments
```bash
sf org list metadata-types --target-org CICD_QAHub
```

### Git Rollback
```bash
# Revert last commit
git revert HEAD
git push origin main

# Then re-run Jenkins pipeline
```

---

## 🎮 Build Parameters - Quick Guide

| Scenario | DEPLOY_FORMAT | DEPLOY_SCOPE | TEST_LEVEL | APEX_CLASSES |
|----------|---------------|--------------|------------|--------------|
| **Full Deploy with Tests** | SOURCE | FULL | RunLocalTests | (empty) |
| **Quick Full Deploy** | SOURCE | FULL | NoTestRun | (empty) |
| **Delta Deploy** | SOURCE | DELTA | RunLocalTests | (empty) |
| **Hotfix (2 classes)** | SOURCE | FULL | NoTestRun | Class1,Class2 |
| **Package Deploy** | MDAPI | FULL | RunLocalTests | (empty) |

---

## 🚨 Common Issues - Fast Solutions

### Issue 1: Build Fails at SCA Stage ❌
**Symptom**: "Critical/High issues found"  
**Solution**:
1. Open: `${BUILD_URL}/artifact/sca-reports/sca-report.html`
2. Fix code violations
3. Commit and re-run

### Issue 2: Authentication Failed 🔐
**Symptom**: "JWT authentication failed"  
**Solution**:
1. Check JWT certificate expiry
2. Verify Connected App in Salesforce
3. Test manually: `sf org login jwt --help`
4. Update credentials in Jenkins if needed

### Issue 3: Test Failures ❌
**Symptom**: "Test failures detected"  
**Solution**:
1. Review console logs for failing test names
2. Run locally: `sf apex run test --test-level RunLocalTests`
3. Fix tests or code
4. Re-run pipeline

### Issue 4: Delta Baseline Not Found 🔍
**Symptom**: "Git ref not found"  
**Solution**:
1. Check branches: `git branch -a`
2. Use commit SHA instead of branch name
3. Or update DELTA_BASELINE parameter

### Issue 5: Jenkins Hung ⏸️
**Symptom**: Build stuck at approval
**Solution**:
1. Check approval timeout (30 min)
2. Approve or abort in Jenkins UI
3. If truly hung, restart Jenkins

### Issue 6: Deployment Conflicts ⚠️
**Symptom**: "Component already exists"  
**Solution**:
1. Review conflict in logs
2. Use FULL deployment instead of DELTA
3. Or manually resolve in Salesforce first

---

## 📊 Key Metrics to Track

### Daily
- [ ] Build success rate: ____%
- [ ] Failed builds today: ____
- [ ] SCA violations: ____

### Weekly
- [ ] Total builds: ____
- [ ] Avoidable failures: ____
- [ ] Failure rate: ____% (Target: <15%)
- [ ] Average build duration: ____ min

### Monthly
- [ ] RCAs completed: ____ (Target: 3+)
- [ ] Clean SCA builds: ____ consecutive (Target: 2+)
- [ ] MTTR improvement: ____

---

## 🔍 Diagnostic Checklist

When a build fails, check in order:

1. **Console Output** ✓
   - What stage failed?
   - What's the error message?

2. **SCA Report** ✓ (if failed at SCA)
   - Open HTML report
   - Check severity levels

3. **Validation Logs** ✓ (if failed at validation)
   - Review metadata errors
   - Check for missing dependencies

4. **Test Results** ✓ (if RunLocalTests)
   - Which tests failed?
   - Code coverage %?

5. **Authentication** ✓ (if connection issues)
   - JWT certificate valid?
   - Org accessible?

6. **Git/Delta** ✓ (if DELTA deployment)
   - Baseline exists?
   - Delta generated correctly?

---

## 📁 File Locations - Quick Access

### On Jenkins Server
```
SCA Reports:      ${JENKINS_HOME}/workspace/[JOB]/sca-reports/
Logs:             ${JENKINS_HOME}/workspace/[JOB]/logs/
Delta:            ${JENKINS_HOME}/workspace/[JOB]/delta/
Build Artifacts:  Via Jenkins UI: ${BUILD_URL}/artifact/
```

### In Git Repository
```
Pipeline:         Jenkinsfile
Source Code:      force-app/
Packages:         manifest/
Documentation:    docs/
```

---

## 🎯 Success Criteria - Quick Check

### Objective 1: Pipeline Stability ✅
- [ ] Failure rate <15%
- [ ] 3+ RCAs documented
- [ ] Metrics tracked weekly

### Objective 2: Security & Quality ✅
- [ ] SCA runs for all deployments
- [ ] 0 Critical/High enforced
- [ ] 2 consecutive clean builds
- [ ] Reports in Jenkins artifacts

### Objective 3: Production Ready ✅
- [ ] Validation before deploy
- [ ] Approval workflow active
- [ ] Rollback procedures ready
- [ ] Comprehensive logging

---

## 📚 Documentation Quick Links

| Document | Purpose | Lines |
|----------|---------|-------|
| [SETUP_GUIDE.md](SETUP_GUIDE.md) | Installation & Setup | 700+ |
| [TROUBLESHOOTING_GUIDE.md](TROUBLESHOOTING_GUIDE.md) | Issues & Solutions | 600+ |
| [RCA_TEMPLATE.md](RCA_TEMPLATE.md) | Root Cause Analysis | 200+ |
| [PIPELINE_METRICS.md](PIPELINE_METRICS.md) | Metrics & Tracking | 400+ |
| [VALIDATION_CHECKLIST.md](VALIDATION_CHECKLIST.md) | Verify Implementation | 350+ |
| [README.md](../README.md) | Project Overview | 400+ |

---

## 🛠️ Maintenance Schedule

### Daily (5 min)
- Check build dashboard
- Review any failures

### Weekly (30 min)
- Update metrics tracker
- Review SCA trends
- Check for SF CLI updates

### Monthly (2 hours)
- Generate failure analysis report
- Review and update documentation
- Team sync on improvements

### Quarterly (half day)
- Certificate expiry check
- Security audit
- Performance optimization
- Stakeholder presentation

---

## 🎓 Training Resources

### New Team Members
1. Read: [README.md](../README.md) (30 min)
2. Read: [SETUP_GUIDE.md](SETUP_GUIDE.md) (1 hour)
3. Run: First test build (1 hour)
4. Review: [TROUBLESHOOTING_GUIDE.md](TROUBLESHOOTING_GUIDE.md) (30 min)

### Salesforce DX
- Official Docs: https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/
- Trailhead: https://trailhead.salesforce.com/

### Jenkins
- Pipeline Tutorial: https://www.jenkins.io/doc/book/pipeline/
- Best Practices: https://www.jenkins.io/doc/book/pipeline/pipeline-best-practices/

---

## 📋 Pre-Deployment Checklist

Before running a production deployment:

- [ ] Code reviewed and approved
- [ ] Local SCA scan passed
- [ ] Tests passing locally
- [ ] Change ticket created
- [ ] Rollback plan documented
- [ ] Stakeholders notified
- [ ] Backup taken (if critical)
- [ ] Off-hours deployment scheduled (if needed)

---

## 🎉 Pipeline Stages - Visual Flow

```
┌────────────────┐
│   Checkout     │ ← Clone from GitHub
└───────┬────────┘
        │
┌───────▼────────┐
│  Verify Git    │ ← Check branch/commits
└───────┬────────┘
        │
┌───────▼────────┐
│ Authenticate   │ ← JWT to Salesforce
└───────┬────────┘
        │
┌───────▼────────┐
│Install Plugins │ ← sfdx-git-delta, scanner
└───────┬────────┘
        │
┌───────▼────────┐
│Generate Delta  │ ← If DELTA scope
└───────┬────────┘
        │
┌───────▼────────┐
│      SCA       │ ← Static Code Analysis
│  🔒 GATE 1 🔒  │    (0 Critical/High)
└───────┬────────┘
        │ PASS
┌───────▼────────┐
│   Validate     │ ← Dry-run deployment
│  🔒 GATE 2 🔒  │    (Check metadata)
└───────┬────────┘
        │ PASS
┌───────▼────────┐
│    Approve     │ ← Manual approval
│  ⏸️ GATE 3 ⏸️  │    (30 min timeout)
└───────┬────────┘
        │ APPROVED
┌───────▼────────┐
│     Deploy     │ ← Actual deployment
└───────┬────────┘
        │
┌───────▼────────┐
│ Post-Validate  │ ← Verify success
└───────┬────────┘
        │
        ✅ SUCCESS
```

---

## 💡 Pro Tips

1. **Use Commit SHAs** for DELTA_BASELINE instead of branch names for stability
2. **Run SCA Locally** before pushing to catch issues early
3. **Test with NoTestRun** first for quick validation
4. **Archive SCA Reports** - they're automatically saved for every build
5. **Document Issues** - use RCA template even for small issues
6. **Monitor Trends** - weekly metrics help identify patterns
7. **Update Docs** - keep troubleshooting guide current with new issues

---

## 🔄 Rollback - Emergency Procedure

### If deployment succeeds but issues found:

```bash
# 1. Revert the commit
git revert HEAD
git push origin main

# 2. Run emergency deployment
# In Jenkins: Set TEST_LEVEL=NoTestRun for speed

# 3. Verify rollback
sf org display --target-org CICD_QAHub

# 4. Document incident
# Use RCA_TEMPLATE.md
```

### If deployment fails mid-way:
- Check logs for deployment ID
- Review what was deployed vs. what failed
- Fix issues and re-run full deployment
- Document as RCA

---

## 📞 Escalation - When to Escalate

| Severity | When | Who | SLA |
|----------|------|-----|-----|
| **Critical** | Production down, can't deploy | DevOps Lead + Manager | 15 min |
| **High** | Deployment blocked, recurring issues | Team Lead | 1 hour |
| **Medium** | Intermittent failures, performance issues | Assigned Engineer | 4 hours |
| **Low** | Documentation, enhancements | Backlog | 2 days |

---

**Version**: 1.0  
**Last Updated**: February 4, 2026  
**Print Date**: __________  
**Keep this card accessible during builds!**

---

## ✅ Today's Build Log

| Build # | Time | Status | Duration | Notes |
|---------|------|--------|----------|-------|
| | | | | |
| | | | | |
| | | | | |
