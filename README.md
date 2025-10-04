# Security Analysis for OWASP Juice Shop OWASP Juice Shop

This guide provides step-by-step instructions for performing Static Application Security Testing (SAST) and Software Composition Analysis (SCA) on the OWASP Juice Shop application using **SonarQube** and **Snyk**.

## Project Overview

This project demonstrates the implementation of **Static Application Security Testing (SAST)** and **Software Composition Analysis (SCA)** using OWASP Juice Shop as a vulnerable web application. The goal is to understand the "shift-left" security approach by identifying and fixing security vulnerabilities early in the development cycle.

### Learning Objectives

- Understand the "shift-left" approach to security
- Apply Static Application Security Testing (SAST) tools to identify vulnerabilities
- Use Software Composition Analysis (SCA) tools to detect flaws in dependencies
- Learn to interpret scan reports and fix common vulnerabilities

## Prerequisites

- Node.js 20+
- npm
- Git
- Docker and Docker Compose (for SonarQube)

## Project Structure

```text
juice-shop-sast-and-sca-fix/
├── routes/                     # Express.js route handlers
├── models/                     # Database models (Sequelize)
├── lib/                        # Utility libraries
├── frontend/                   # Angular frontend application
├── data/                       # Static data and configurations
├── test/                       # Test suites (API and E2E)
├── docker-compose.yml          # Docker configuration
├── package.json                # Node.js dependencies
├── sonar-project.properties    # SonarQube configuration
└── READM.md # This file
```

## Setup and Installation

### 1. Clone and Setup

```bash
git clone https://github.com/RaedAddala/juice-shop-sast-and-sca-fix
cd juice-shop-sast-and-sca-fix
npm install
```

### 2. Build Docker Image

```bash
docker build -t juice-shop-security .
```

### 3. Run the Application

```bash
# Development mode
npm run serve:dev

# Production mode
docker run -p 3000:3000 juice-shop-security
```

## SAST Analysis with SonarQube

### 1. Start SonarQube via Docker Compose

In the root of your Juice Shop project (where the `docker-compose.yml` is), run:

```bash
docker-compose up -d sonarqube
```

Wait 1-2 minutes for it to fully start (check logs with `docker-compose logs sonarqube`). Access <http://localhost:9000>, log in with admin/admin, and change the password on first login.

Generate a user token for authentication: Go to **My Account > Security**, create a token (e.g., name it "juice-scan"), and copy it.

### 2. Configure SonarQube

The `sonar-project.properties` file is already configured in the project root. Update the `sonar.login` line with your generated token:

```properties
sonar.login=<your_generated_token_here>
```

### 3. Run the SAST Scan

Execute the scan using npx:

```bash
npx sonarqube-scanner
```

This will analyze the codebase (takes 5-10 minutes, depending on your machine). It uploads results to your local SonarQube instance.

Once complete, refresh <http://localhost:9000>, go to **Projects**, and select "OWASP Juice Shop Security Analysis" to view the dashboard.

## Identified SAST Vulnerabilities

### 1. SQL Injection (Critical)

**Location**: `routes/login.ts:36`
**Issue**: Direct string concatenation in SQL query

```typescript
// VULNERABLE CODE
models.sequelize.query(`SELECT * FROM Users WHERE email = '${req.body.email || ''}' AND password = '${security.hash(req.body.password || '')}' AND deletedAt IS NULL`, { model: UserModel, plain: true })
```

**Fix**: Use parameterized queries

```typescript
// FIXED CODE
models.sequelize.query(`SELECT * FROM Users WHERE email = $mail AND password = $pass AND deletedAt IS NULL`,
  { bind: { mail: req.body.email, pass: security.hash(req.body.password) }, model: UserModel, plain: true })
```

### 2. Cross-Site Scripting (XSS) (input Sanitization) (High)

**Location**: `models/feedback.ts:41`
**Issue**: Insufficient input sanitization

```typescript
// VULNERABLE CODE
sanitizedComment = security.sanitizeHtml(comment)
```

**Fix**: Use secure sanitization

```typescript
// FIXED CODE
sanitizedComment = security.sanitizeSecure(comment)
```

### 3. Path Traversal (Medium)

**Location**: `routes/fileServer.ts:25`
**Issue**: Inadequate path validation

```typescript
// VULNERABLE CODE
if (!file.includes('/')) {
  verify(file, res, next)
}
```

**Fix**: Proper path sanitization

```typescript
// FIXED CODE
const sanitizedFile = path.basename(file)
if (sanitizedFile === file && !file.includes('..')) {
  verify(sanitizedFile, res, next)
}
```

### 4. Cross-Site Scripting (XSS) (Sanitizer Disabled) (High)

**Locations**:

  - `frontend/src/app/track-result/track-result.component.ts:45`
  - `frontend/src/app/administration/administration.component.ts:57`
  - `frontend/src/app/administration/administration.component.ts:75`
  - `frontend/src/app/data-export/data-export.component.ts:55`
  - `frontend/src/app/last-login-ip/last-login-ip.component.ts:38`
  - `frontend/src/app/score-board/score-board.component.ts:86`
  - `frontend/src/app/search-result/search-result.component.ts:135`
  - `frontend/src/app/search-result/search-result.component.ts:161`
  - `frontend/src/app/about/about.component.ts:121`
  
**Issue**: Disabling Angular built-in sanitization

```typescript
// VULNERABLE CODE
this.results.orderNo = this.sanitizer.bypassSecurityTrustHtml(`<code>${results.data[0].orderId}</code>`)
```

**Fix**: Use parameterized queries

In the template:

```typescript
// FIXED CODE
<code>{{ results.orderNo }}</code>
```

and in the typescript code:

```typescript
// FIXED CODE
this.results.orderNo = results.data[0].orderId;
```

## SCA Analysis with Snyk

### Setup Snyk

```bash
# Install Snyk CLI
npm install -g snyk

# Authenticate (requires Snyk account)
snyk auth

# Test for vulnerabilities
snyk test
```

### Run SCA Scan

```bash
# Scan for vulnerabilities
snyk test --json > snyk-report.json

# Monitor project
snyk monitor
```

## Identified SCA Vulnerabilities

### 1. Prototype Pollution in lodash (High)

**Package**: `lodash@4.17.20`
**CVE**: CVE-2020-8203
**Fix**: Update to `lodash@4.17.21`

```bash
npm update lodash
```

### 2. Regular Expression DoS in semver (Medium)

**Package**: `semver@6.3.0`
**CVE**: CVE-2022-25883
**Fix**: Update to `semver@7.3.8`

```bash
npm install semver@^7.3.8
```

### 3. Cross-Site Scripting in sanitize-html (Medium)

**Package**: `sanitize-html@1.27.5`
**CVE**: CVE-2021-26539
**Fix**: Update to `sanitize-html@2.7.3`

```bash
npm install sanitize-html@^2.7.3
```

## Testing and Validation

### Verify SAST Fixes

```bash
# Re-run SonarQube scan
sonarqube-scanner

# Check that security hotspots are resolved
# Access SonarQube dashboard at http://localhost:9000
```

### Verify SCA Fixes

```bash
# Re-run Snyk test
snyk test

# Should show fewer or no vulnerabilities
npm audit
```

## Key Learnings

1. **Shift-Left Security**: Early detection saves time and resources
2. **Automated Scanning**: Essential for continuous security monitoring
3. **Dependency Management**: Regular updates prevent security debt
4. **False Positives**: Critical thinking required for scan results
5. **Security Culture**: Tools are enablers, not replacements for secure coding

## Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [SonarQube Documentation](https://docs.sonarqube.org/)
- [Snyk Documentation](https://docs.snyk.io/)
- [OWASP Juice Shop](https://owasp-juice.shop/)
- [Shift-Left Security](https://www.synopsys.com/glossary/what-is-shift-left-security.html)

## Contributing

1. Fork the repository
2. Create a feature branch
3. Run security scans
4. Submit pull request with scan results

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
