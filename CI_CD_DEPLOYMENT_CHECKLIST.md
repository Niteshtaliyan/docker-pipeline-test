# CI/CD Setup Checklist & Deployment Steps

## Pre-Deployment Checklist

### Repository Setup
- [ ] Repository created on GitHub
- [ ] Repository cloned locally
- [ ] Main branch protection rules configured
- [ ] Develop branch created
- [ ] All team members have access

### GitHub Secrets Configuration
- [ ] Navigate to: Settings → Secrets and variables → Actions
- [ ] Add `DOCKER_HUB_USERNAME` secret with your Docker Hub username
- [ ] Add `DOCKER_HUB_PASSWORD` secret with Docker Hub Personal Access Token
  - **Note:** Use PAT token, NOT your Docker password
  - Create token at: https://hub.docker.com/settings/security
- [ ] Test secret access by viewing workflow logs (not the value itself)

### File Verification
- [ ] `.github/workflows/ci-cd.yml` exists and is valid
- [ ] `.github/workflows/pr-check.yml` exists and is valid
- [ ] `docker-compose.staging.yml` created for CI/CD tests
- [ ] All Dockerfiles present (api, web, tests)
- [ ] `README.md` updated with CI/CD info
- [ ] `GITHUB_ACTIONS_GUIDE.md` created
- [ ] `PIPELINE_DIAGRAM.md` created
- [ ] `METRICS.md` created
- [ ] `.env.example` file present

### Dockerfile Verification
- [ ] API Dockerfile uses `node:18-alpine`
- [ ] Web Dockerfile uses `nginx:alpine`
- [ ] Tests Dockerfile uses `mcr.microsoft.com/playwright:v1.34.3`
- [ ] All Dockerfiles pass Hadolint checks
- [ ] No hardcoded secrets in Dockerfiles
- [ ] Health checks defined in docker-compose files

### Database & Services
- [ ] `init.sql` database initialization script created
- [ ] PostgreSQL credentials configured in `.env`
- [ ] Redis service configured
- [ ] API endpoints documented
- [ ] Test suite configured in tests/

## Deployment Steps

### Step 1: Push Initial Commit

```bash
# Clone and setup
git clone https://github.com/<your-org>/week4.git
cd week4

# Create initial commit
git add .
git commit -m "ci: Add GitHub Actions CI/CD pipeline"
git push origin main
```

### Step 2: Monitor First Pipeline Run

```bash
# View workflow in GitHub Actions tab
# Expected duration: 7-9 minutes

# Check workflow status
gh run list --workflow ci-cd.yml -L 5

# View detailed logs
gh run view <run-id> --log

# Download artifacts
gh run download <run-id>
```

### Step 3: Verify Pipeline Success

Check that all stages passed:
- [ ] Build & Test stage completed
- [ ] Run Tests stage completed
- [ ] Security & Quality stage completed
- [ ] Publish stage completed (if on main)
- [ ] Notify stage posted summary

### Step 4: Review Security Results

```bash
# Download and review artifacts
gh run download <run-id> -D artifacts/

# Check SBOM files
cat artifacts/sbom-results/sbom-api.json

# Review Trivy reports (in GitHub Security tab)
```

### Step 5: Configure Additional Branch Protection

```bash
# Via GitHub CLI
gh repo edit --enable-auto-merge

# Via GitHub Web UI
Settings → Branches → Add rule
- Pattern: main
- Require status checks to pass:
  - build-and-test
  - run-tests
  - security-quality
```

### Step 6: Test PR Workflow

```bash
# Create test PR
git checkout -b test/pr-workflow
echo "# Test" >> README.md
git add .
git commit -m "test: PR workflow validation"
git push origin test/pr-workflow

# Create PR via GitHub
gh pr create --base main --head test/pr-workflow --title "Test PR workflow"

# Monitor PR checks
# Should see all pr-check.yml jobs running
```

### Step 7: Manual Deployment Test

```bash
# Trigger manual workflow
gh workflow run ci-cd.yml --ref main --input deploy=true

# Monitor deployment
gh run list --workflow ci-cd.yml -L 3
```

## Post-Deployment Verification

### 1. Image Publishing Success

```bash
# Verify images on Docker Hub
docker pull docker.io/<username>/api:latest
docker pull docker.io/<username>/web:latest
docker pull docker.io/<username>/tests:latest
```

### 2. Test Execution

```bash
# Pull test image
docker pull docker.io/<username>/tests:latest

# Run tests locally
docker run --rm docker.io/<username>/tests:latest npm test
```

### 3. Security Scan Results

- [ ] GitHub Security tab shows Trivy scans
- [ ] No critical or high vulnerabilities
- [ ] SBOMs generated and available
- [ ] Hadolint passed all checks

### 4. Performance Metrics

- [ ] Build time under 10 minutes (target 8 min)
- [ ] Test pass rate 100%
- [ ] Cache hit rate > 70%
- [ ] All jobs completed successfully

## Troubleshooting Guide

### Issue: Docker Hub Login Fails

**Symptoms:** `Error: Invalid credentials`

**Solution:**
1. Verify username in `DOCKER_HUB_USERNAME` secret
2. Ensure using Personal Access Token (not password)
3. Check token hasn't expired
4. Regenerate token: https://hub.docker.com/settings/security

```bash
# Test credentials locally
docker login -u <username>
# Enter PAT token when prompted
```

### Issue: Tests Fail in CI

**Symptoms:** Tests pass locally but fail in workflow

**Steps to Debug:**
1. Download test logs artifact: `gh run download <run-id>`
2. Check `test-logs.txt` for error messages
3. Verify `docker-compose.staging.yml` matches local setup
4. Check API URL environment variable

**Common Causes:**
- Service startup timeout (increase wait time)
- API URL incorrect in tests
- Database not initialized
- Health checks failing

### Issue: Build Timeout

**Symptoms:** Pipeline exceeds 10 minutes

**Solution:**
1. Check Docker build logs for slow steps
2. Verify cache is being utilized
3. Consider splitting tests into parallel jobs
4. Optimize Dockerfile layer ordering

```bash
# Clear GitHub Actions cache
gh actions-cache delete --all
```

### Issue: Security Vulnerabilities Found

**Symptoms:** Trivy reports high/critical issues

**Steps:**
1. Review SBOM in artifacts
2. Identify vulnerable package/dependency
3. Update base image: `docker build --build-arg FROM_IMAGE=node:18-alpine --no-cache`
4. Research CVE details
5. Document acceptable risks if necessary

## Monitoring & Maintenance

### Weekly Tasks
- [ ] Review workflow run history
- [ ] Check for failed runs
- [ ] Monitor security scan results
- [ ] Update dependencies

### Monthly Tasks
- [ ] Review performance metrics
- [ ] Clean up old artifacts
- [ ] Update base images
- [ ] Audit GitHub Actions permissions
- [ ] Review access logs

### Quarterly Tasks
- [ ] Full security audit
- [ ] Performance optimization review
- [ ] Team training on CI/CD
- [ ] Backup workflow configurations

## Configuration Files Summary

| File | Purpose | Status |
|------|---------|--------|
| `.github/workflows/ci-cd.yml` | Main CI/CD pipeline | ✅ Created |
| `.github/workflows/pr-check.yml` | PR validation | ✅ Created |
| `docker-compose.staging.yml` | CI/CD test environment | ✅ Created |
| `GITHUB_ACTIONS_GUIDE.md` | Setup documentation | ✅ Created |
| `PIPELINE_DIAGRAM.md` | Architecture diagrams | ✅ Created |
| `METRICS.md` | Performance analysis | ✅ Created |
| `README.md` | Updated with CI/CD | ✅ Updated |
| `.env.example` | Environment template | ✅ Verified |
| `init.sql` | Database setup | ✅ Verified |

## Expected Workflow Duration

```
Stage 1 - Build & Test:    3 min (matrix: 3 parallel)
  ├─ API build            1:30 min
  ├─ Web build            1:20 min
  ├─ Tests build          0:50 min
  ├─ Hadolint scan        0:15 min
  └─ Trivy scan           1:00 min

Stage 2 - Run Tests:        2:30 min
  ├─ Load images          0:30 min
  ├─ Start services       0:45 min
  ├─ Wait health checks   0:30 min
  └─ Run tests            0:45 min

Stage 3 - Security/Qual:    2:00 min
  ├─ Trivy full scan      1:15 min
  └─ Generate SBOM        0:45 min

Stage 4 - Publish:          2:30 min (main only)
  ├─ API push             0:45 min
  ├─ Web push             0:45 min
  └─ Tests push           0:45 min

Stage 5 - Notify:           0:30 min

TOTAL (main branch):        ~10:30 min  (9-10 min with caching)
TOTAL (PR/develop):         ~7:30 min   (no publish stage)
```

## Success Criteria - Final Checklist

✅ **Build & Test**
- [ ] All images build successfully
- [ ] Hadolint linting passes
- [ ] Trivy security scan passes
- [ ] Build cache hit rate > 70%

✅ **Testing**
- [ ] All tests pass in CI
- [ ] 100% test success rate
- [ ] Tests run in < 3 minutes
- [ ] Test results archived

✅ **Security**
- [ ] Zero critical vulnerabilities
- [ ] Zero high vulnerabilities
- [ ] SBOM generated
- [ ] Scan results available in GitHub Security tab

✅ **Publishing**
- [ ] Images push to Docker Hub successfully
- [ ] Tags applied correctly
- [ ] Latest tag on main
- [ ] SHA tags on all branches

✅ **Performance**
- [ ] Total pipeline < 10 min
- [ ] No timeouts or throttling
- [ ] Notifications sent
- [ ] Artifacts archived

✅ **Documentation**
- [ ] README updated
- [ ] Setup guide complete
- [ ] Pipeline diagrams clear
- [ ] Metrics documented

## Support Resources

- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [Trivy Scanner](https://github.com/aquasecurity/trivy-action)
- [Hadolint](https://github.com/hadolint/hadolint-action)
- [GitHub CLI](https://cli.github.com)

## Next Steps

1. ✅ Push all workflow files to GitHub
2. ✅ Monitor first pipeline run (7-9 minutes)
3. ✅ Verify all stages pass
4. ✅ Create test PR to validate pr-check workflow
5. ✅ Review security scan results
6. ✅ Share pipeline status with team
7. ✅ Set up alerting for failures
8. ✅ Schedule regular reviews

---

**Document Version:** 1.0  
**Last Updated:** March 7, 2024  
**Status:** Ready for Deployment
