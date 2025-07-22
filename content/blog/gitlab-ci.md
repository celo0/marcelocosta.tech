---
title: "Automating SAST, SCA, and DefectDojo Uploads with GitLab CI"
date: 2025-07-22
tags: ["gitlab-ci", "sast", "sca", "semgrep", "bearer", "trivy", "defectdojo", "devsecops", "security pipeline"]
---

Most projects barely scratch the surface when it comes to security in CI pipelines — if they implement it at all. I wanted more than just a checkbox. I wanted **source code analysis, dependency scanning, and centralized vulnerability tracking through DefectDojo**.

<!--more-->

So I integrated the following tools into my GitLab CI pipeline:

- 🔍 **Semgrep** for SAST
- 🕵️ **Bearer** to detect sensitive data usage
- 🧱 **Trivy** for SCA (filesystem dependency scanning)
- 📬 **Automatic uploads** of all reports to DefectDojo

---

### 🔧 The GitLab CI Setup

One important detail in this setup is that both **Semgrep** and **Trivy** are executed twice — each with a different purpose:

- ✅ **First run**: generates a full JSON report for DefectDojo
- ❌ **Second run**: enforces quality gates and fails the pipeline if high-severity issues are found

This way, even if the pipeline breaks due to critical findings, the results are still preserved and uploaded.

---

### 🔍 Semgrep (SAST)

```yaml
sast-semgrep:
  image: 
    name: returntocorp/semgrep
    entrypoint: [""]
    pull_policy: always
  stage: security
  tags:
    - docker-runner
  script:
    # Generate complete report for upload
    - semgrep scan . --config auto --json > semgrep-report-$CI_PIPELINE_ID.json

    # Fail pipeline if errors found
    - semgrep scan . --config auto --severity ERROR --error
  artifacts:
    paths:
      - semgrep-report-$CI_PIPELINE_ID.json
    when: always
  rules:
    - if: '$CI_COMMIT_BRANCH == "master" || $CI_COMMIT_BRANCH =~ /^release/'
```

---

### 🕵️ Bearer (Sensitive Data Detection)

```yaml
sast-bearer:
  image: 
    name: bearer/bearer
    entrypoint: [""]
    pull_policy: always
  tags:
    - docker-runner
  stage: security
  script:
    - bearer scan . --severity critical,high,medium,low,warning --fail-on-severity critical,high --exit-code 1 --format json --output bearer-report-$CI_PIPELINE_ID.json
  artifacts:
    paths:
      - bearer-report-$CI_PIPELINE_ID.json
    when: always
  rules:
    - if: '$CI_COMMIT_BRANCH == "master" || $CI_COMMIT_BRANCH =~ /^release/'
```

---

### 🧱 Trivy (SCA / Dependency Scanner)

```yaml
sca-trivy:
  image: 
    name: aquasec/trivy
    entrypoint: [""]
    pull_policy: always
  tags:
    - docker-runner
  stage: security
  script:
    # Full vulnerability report for DefectDojo
    - trivy fs . --scanners vuln --cache-dir /var/cache/trivy --severity CRITICAL,HIGH,MEDIUM,LOW --format json -o trivy-report-$CI_PIPELINE_ID.json

    # Fail pipeline on CRITICAL or HIGH vulns, running on the results of the previous scan
    - trivy convert --cache-dir /var/cache/trivy --scanners vuln --exit-code 1 --severity CRITICAL,HIGH trivy-report-$CI_PIPELINE_ID.json
  artifacts:
    paths:
      - trivy-report-$CI_PIPELINE_ID.json
    when: always
  rules:
    - if: '$CI_COMMIT_BRANCH == "master" || $CI_COMMIT_BRANCH =~ /^release/'
```

---

### 📤 Uploading to DefectDojo

Finally, a job handles automated upload of all reports to DefectDojo. It:

1. Creates a new **engagement**
2. Uploads reports from Semgrep, Trivy, and Bearer via API
3. Marks them as active and verified

```yaml
upload-defect-dojo:
  image: 
    name: alpine/curl:latest
    entrypoint: [""]
  stage: security-report
  tags:
    - docker-runner
  needs: 
    - job: sast-semgrep
    - job: sast-bearer
    - job: sca-trivy
      optional: true
  script:
    - |
      echo "== Creating DefectDojo engagement =="
      ENGAGEMENT_RESPONSE=$(curl -s -X POST "$DEFECTDOJO_URL/api/v2/engagements/" \
      -H "Authorization: Token $DEFECTDOJO_KEY" \
      -H "Content-Type: application/json" \
      -d '{
            "name": "GitLab Scan - '"$CI_COMMIT_REF_NAME"' - Pipeline '"$CI_PIPELINE_ID"'",
            "product": '"$DEFECTDOJO_PID"',
            "target_start": "'"$(date +%F)"'",
            "target_end": "'"$(date +%F)"'",
            "engagement_type": "CI/CD",
            "status": "In Progress"
          }')
      ENGAGEMENT_ID=$(echo "$ENGAGEMENT_RESPONSE" | grep -o '"id":[0-9]*' | grep -o '[0-9]*')

      echo "== Uploading Semgrep report =="
      curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
      -H "Authorization: Token $DEFECTDOJO_KEY" \
      -F "engagement=$ENGAGEMENT_ID" \
      -F "scan_type=Semgrep JSON Report" \
      -F "minimum_severity=Low" \
      -F "active=true" -F "verified=true" \
      -F "file=@semgrep-report-$CI_PIPELINE_ID.json"

      echo "== Uploading Trivy report =="
      curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
      -H "Authorization: Token $DEFECTDOJO_KEY" \
      -F "engagement=$ENGAGEMENT_ID" \
      -F "scan_type=Trivy Scan" \
      -F "minimum_severity=Low" \
      -F "active=true" -F "verified=true" \
      -F "file=@trivy-report-$CI_PIPELINE_ID.json"

      echo "== Uploading Bearer report =="
      curl -X POST "$DEFECTDOJO_URL/api/v2/import-scan/" \
      -H "Authorization: Token $DEFECTDOJO_KEY" \
      -F "engagement=$ENGAGEMENT_ID" \
      -F "scan_type=Bearer CLI" \
      -F "minimum_severity=Low" \
      -F "active=true" -F "verified=true" \
      -F "file=@bearer-report-$CI_PIPELINE_ID.json"
  rules:
    - if: '$CI_COMMIT_BRANCH == "master" || $CI_COMMIT_BRANCH =~ /^release/'
```

---

### ✅ Results

- 🔄 **Automated scans** for every `master` and `release/*` branch
- ☁️ **Centralized visibility** in DefectDojo
- 🚫 Pipeline **fails automatically** on critical findings, without losing traceability

This setup brings **real security enforcement** to your DevSecOps flow — with full visibility and minimal friction.

---

Let me know if you'd like a version in Portuguese or want to extend this to include **SBOM generation with CycloneDX**.
