# Detection-as-Code CI/CD Pipeline
 
[![Lint & Convert](https://github.com/YOUR_USER/detection-as-code/actions/workflows/lint-and-convert.yml/badge.svg)](link)
[![Integration Test](https://github.com/YOUR_USER/detection-as-code/actions/workflows/integration-test.yml/badge.svg)](link)
 
A fully automated detection engineering pipeline that demonstrates the complete
detection lifecycle: authoring Sigma rules, validating via CI/CD, testing against
real attack telemetry with Splunk Attack Range, and deploying to Splunk.
 
## Architecture
Author (Sigma) -> Validate (CI) -> Test (Attack Range) -> Deploy (Splunk)
 
## Quick Start
1. Clone this repo
2. Install sigma-cli: `pip install sigma-cli pySigma-backend-splunk`
3. Run lint: `sigma check rules/`
4. Convert: `sigma convert -t splunk -p splunk_windows rules/`
 
## Detection Library
| # | Detection | ATT&CK | Status |
|---|-----------|--------|--------|
| 1 | LSASS Dump via Procdump | T1003.001 | Production |
 
## Project Structure
(paste directory tree here)
# detection-as-code
