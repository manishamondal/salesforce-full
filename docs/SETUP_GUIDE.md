# Salesforce CI/CD Pipeline - Complete Setup Guide

## Prerequisites

### 1. Software Requirements
- **Jenkins**: Version 2.400+ 
- **Git**: Version 2.30+
- **Salesforce CLI (sf)**: Latest version
- **Java**: JDK 11 or 17 (for Jenkins)
- **Operating System**: Windows/Linux/Mac

### 2. Salesforce Requirements
- Salesforce Developer/Sandbox/Production org
- System Administrator access
- API enabled
- Connected App with JWT authentication

---

## Part 1: Jenkins Installation (Windows)

### Step 1: Install Java
```powershell
# Download and install OpenJDK 17
# From: https://adoptium.net/

# Verify installation
java -version
```

### Step 2: Install Jenkins
```powershell
# Download Jenkins Windows installer
# From: https://www.jenkins.io/download/

# Run installer (jenkins.msi)
# Default port: 8080

# Access Jenkins
# Open browser: http://localhost:8080

# Get initial admin password
Get-Content "C:\Program Files\Jenkins\secrets\initialAdminPassword"
```

### Step 3: Install Suggested Plugins
- Navigate to: Manage Jenkins > Manage Plugins
- Install:
  - Git plugin
  - Pipeline plugin
  - Credentials Binding plugin
  - SSH Agent plugin

---

## Part 2: Salesforce CLI Installation

### Windows
```powershell
# Download and install SF CLI
# From: https://developer.salesforce.com/tools/salesforcecli

# Or use npm
npm install -g @salesforce/cli

# Verify installation
sf --version

# Install required plugins
echo y | sf plugins install sfdx-git-delta
sf plugins install @salesforce/sfdx-scanner

# Verify plugins
sf plugins
```

### Configure Git (if not already done)
```powershell
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

---

## Part 3: Salesforce Connected App Setup

### Step 1: Create Connected App
1. Log in to Salesforce org
2. Setup > App Manager > New Connected App
3. Configure:
   - **Connected App Name**: Jenkins CI/CD
   - **API Name**: Jenkins_CICD
   - **Contact Email**: your.email@example.com
   - **Enable OAuth Settings**: ✓
   - **Callback URL**: http://localhost:1717/OauthRedirect
   - **Use digital signatures**: ✓ (upload certificate - see below)
   - **Selected OAuth Scopes**:
     - Full access (full)
     - Perform requests at any time (refresh_token, offline_access)
   - **Require Secret for Web Server Flow**: ✓

### Step 2: Generate SSL Certificate
```powershell
# Generate private key and certificate
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
openssl x509 -req -in server.csr -signkey server.key -out server.crt -days 3650

# Create combined key file for JWT
openssl pkcs8 -topk8 -inform PEM -outform PEM -in server.key -out server.key.pem -nocrypt

# Save server.crt (upload to Connected App)
# Save server.key.pem (use in Jenkins)
```

### Step 3: Configure Connected App Policies
1. After creating app, click "Manage"
2. Click "Edit Policies"
3. **OAuth Policies**:
   - Permitted Users: Admin approved users are pre-authorized
   - IP Relaxation: Relax IP restrictions
4. Save

### Step 4: Assign Permission Set (if using restricted users)
1. Create Permission Set with API Enabled
2. Assign to integration user
3. Manage Connected Apps > Jenkins CI/CD > Manage Permission Sets

---

## Part 4: GitHub Repository Setup

### Step 1: Create GitHub Repository
```powershell
# Create repository on GitHub
# Name: salesforce-full

# Clone locally
git clone https://github.com/YOUR_USERNAME/salesforce-full.git
cd salesforce-full

# Copy project files
# (Copy Jenkinsfile and force-app directory)

# Initialize SFDX project (if not done)
sf project generate --name salesforce-full

# Commit and push
git add .
git commit -m "Initial commit - Salesforce CI/CD project"
git push origin main
```

### Step 2: Configure Git Authentication in Jenkins
1. Generate GitHub Personal Access Token
   - GitHub > Settings > Developer settings > Personal access tokens
   - Generate new token with `repo` scope
2. Add to Jenkins:
   - Manage Jenkins > Manage Credentials
   - Add Credentials > Username with password
   - ID: `github-credentials`

---

## Part 5: Jenkins Configuration

### Step 1: Add Salesforce Credentials
1. **Add JWT Key File**:
   ```
   Manage Jenkins > Manage Credentials > System > Global credentials
   Kind: Secret file
   File: server.key.pem (your private key)
   ID: sfdx_jwt_key
   Description: Salesforce JWT Key
   ```

2. **Add Client ID**:
   ```
   Manage Jenkins > Manage Credentials > System > Global credentials
   Kind: Secret text
   Secret: [Consumer Key from Connected App]
   ID: sfdx_client_id
   Description: Salesforce Client ID
   ```

### Step 2: Configure Git in Jenkins
```
Manage Jenkins > Global Tool Configuration
Git:
- Name: Default
- Path to Git executable: C:\Program Files\Git\bin\git.exe (Windows)
                          or /usr/bin/git (Linux)
```

### Step 3: Create Jenkins Pipeline Job
1. New Item > Pipeline
2. Name: `Salesforce-CICD-Pipeline`
3. Configure:
   - **Pipeline section**:
     - Definition: Pipeline script from SCM
     - SCM: Git
     - Repository URL: https://github.com/YOUR_USERNAME/salesforce-full.git
     - Credentials: [Select github-credentials]
     - Branch: */main
     - Script Path: Jenkinsfile
   - **Build Triggers** (optional):
     - Poll SCM: H/5 * * * * (every 5 minutes)
     - Or use GitHub webhooks

---

## Part 6: First Build Test

### Step 1: Test Salesforce Authentication
```powershell
# On Jenkins server, test manually first
cd "C:\Program Files\sf"
.\bin\sf org login jwt \
  --client-id "YOUR_CLIENT_ID" \
  --jwt-key-file "C:\path\to\server.key.pem" \
  --username "your.email@example.com" \
  --instance-url "https://login.salesforce.com" \
  --alias "TestOrg"

# Verify
.\bin\sf org display --target-org TestOrg
```

### Step 2: Run First Pipeline Build
1. Open Jenkins Dashboard
2. Click on `Salesforce-CICD-Pipeline`
3. Click "Build with Parameters"
4. Set parameters:
   - DEPLOY_FORMAT: SOURCE
   - DEPLOY_SCOPE: FULL
   - TEST_LEVEL: NoTestRun
   - DELTA_BASELINE: origin/main
   - PACKAGE_XML_PATH: manifest/package.xml
   - APEX_CLASSES: (leave empty)
5. Click "Build"

### Step 3: Monitor Build
- Click on build number (e.g., #1)
- View "Console Output"
- Troubleshoot any errors

---

## Part 7: Verification & Testing

### Verify Static Code Analysis
1. After successful build, check artifacts:
   - Build > Artifacts > sca-reports/sca-report.html
2. Verify summary shows:
   ```
   ✅ PASSED: No Critical/High issues above threshold
   ```

### Verify Deployment Logs
1. Check artifacts:
   - logs/deployment-summary.txt
   - logs/build-metrics-summary.txt
   - logs/pipeline-summary.txt

### Test Different Scenarios

#### Scenario 1: DELTA Deployment
```
Parameters:
- DEPLOY_SCOPE: DELTA
- DELTA_BASELINE: HEAD~1 (or previous commit SHA)
```

#### Scenario 2: MDAPI Deployment
```
Parameters:
- DEPLOY_FORMAT: MDAPI
- PACKAGE_XML_PATH: manifest/package-core.xml
```

#### Scenario 3: Specific Apex Classes
```
Parameters:
- APEX_CLASSES: AccountService,ContactService
```

#### Scenario 4: With Tests
```
Parameters:
- TEST_LEVEL: RunLocalTests
```

---

## Part 8: Rollback Testing

### Test Rollback Scenario
1. Make a code change and deploy
2. Intentionally break something
3. Revert Git commit:
   ```powershell
   git revert HEAD
   git push origin main
   ```
4. Re-run pipeline
5. Verify rollback successful

---

## Part 9: Documentation & RCA Setup

### Create RCA Document for Test Issue
1. Use template: `docs/RCA_TEMPLATE.md`
2. Document a test failure scenario
3. Include:
   - Root cause
   - Resolution steps
   - Preventive measures
4. Save as: `docs/RCA-2026-02-04-001.md`

### Track Metrics
1. Create metrics file: `docs/metrics-tracker.xlsx` (or use PIPELINE_METRICS.md)
2. Record each build:
   - Build number
   - Status
   - Duration
   - SCA results
3. Calculate failure rate weekly

---

## Part 10: Advanced Configuration

### Enable Email Notifications
```groovy
// Add to Jenkinsfile post section
post {
    always {
        emailext (
            subject: "Salesforce Build ${currentBuild.result}: ${BUILD_NUMBER}",
            body: """
            Build: ${BUILD_URL}
            Status: ${currentBuild.result}
            Duration: ${currentBuild.durationString}
            """,
            to: "team@example.com"
        )
    }
}
```

### Enable Slack Notifications
```groovy
// Install Slack Notification plugin
// Add to Jenkinsfile
post {
    success {
        slackSend(
            color: 'good',
            message: "✅ Salesforce deployment successful: ${BUILD_URL}"
        )
    }
    failure {
        slackSend(
            color: 'danger',
            message: "❌ Salesforce deployment failed: ${BUILD_URL}"
        )
    }
}
```

### Schedule Regular Builds
```
Build Triggers > Build periodically
H 2 * * * (daily at 2 AM)
H 2 * * 1 (weekly on Monday at 2 AM)
```

---

## Troubleshooting Common Setup Issues

### Issue: SF CLI Not Found
```powershell
# Add to Jenkins environment
Manage Jenkins > Configure System > Global properties
Environment variables:
- Name: PATH
- Value: C:\Program Files\sf\bin;${env.PATH}
```

### Issue: Git Not Found
```powershell
# Install Git
# Add to system PATH
# Restart Jenkins service
```

### Issue: Authentication Fails
- Verify Connected App is active
- Check certificate is valid
- Verify username is correct
- Check IP restrictions in Salesforce

### Issue: Plugins Not Installing
```powershell
# Update SF CLI first
sf update

# Then install plugins
echo y | sf plugins install sfdx-git-delta
sf plugins install @salesforce/sfdx-scanner
```

---

## Security Best Practices

1. **Credential Management**:
   - Never commit credentials to Git
   - Use Jenkins Credential Manager
   - Rotate JWT certificates annually

2. **Access Control**:
   - Limit Jenkins access to authorized users
   - Use role-based access control (RBAC)
   - Enable audit logging

3. **Network Security**:
   - Use HTTPS for Jenkins
   - Configure firewall rules
   - Use VPN for remote access

4. **Code Security**:
   - Enable SCA thresholds (already configured)
   - Review code before deployment
   - Use branch protection rules

---

## Maintenance Checklist

### Daily
- [ ] Monitor build success rate
- [ ] Review failed builds
- [ ] Check artifact archiving

### Weekly
- [ ] Review SCA reports
- [ ] Update metrics tracker
- [ ] Check for SF CLI updates

### Monthly
- [ ] Review and update documentation
- [ ] Analyze failure trends
- [ ] Performance optimization review
- [ ] Security audit

### Quarterly
- [ ] Update Jenkins plugins
- [ ] Review Connected App settings
- [ ] Certificate expiry check
- [ ] Team training session

---

## Success Criteria Validation

### ✅ Objective 1: Pipeline Stability
- [ ] Pipeline runs successfully
- [ ] Failure rate tracked
- [ ] 3+ RCAs documented

### ✅ Objective 2: Security & Quality
- [ ] SCA runs for all deployments
- [ ] 0 Critical/High issues enforced
- [ ] Reports archived in Jenkins
- [ ] 2 consecutive clean builds achieved

### ✅ Objective 3: Production Readiness
- [ ] Approval workflow functional
- [ ] Rollback tested
- [ ] Error handling verified
- [ ] Documentation complete

---

## Next Steps

1. **Week 1-2**: Setup and basic validation
   - Complete all installation steps
   - Run first successful build
   - Document first RCA

2. **Week 3-4**: Optimize and stabilize
   - Fine-tune SCA thresholds
   - Optimize build times
   - Document 2nd and 3rd RCAs

3. **Week 5-6**: Advanced features
   - Add notifications
   - Implement advanced rollback
   - Create custom dashboards

4. **Ongoing**: Maintain and improve
   - Weekly metrics review
   - Monthly optimization
   - Continuous learning

---

## Support & Resources

### Documentation
- Salesforce DX: https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/
- Jenkins: https://www.jenkins.io/doc/
- Git: https://git-scm.com/doc

### Community
- Salesforce Stack Exchange: https://salesforce.stackexchange.com/
- Jenkins Community: https://community.jenkins.io/

### Training
- Trailhead: https://trailhead.salesforce.com/
- Jenkins Training: https://www.jenkins.io/doc/tutorials/

---

**Document Version**: 1.0  
**Last Updated**: February 4, 2026  
**Author**: DevOps Team  
**Feedback**: devops@example.com
