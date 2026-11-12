# Detection-as-Code CI/CD Pipeline
 
[![Lint & Convert](https://github.com/RedX300/detection-as-code/actions/workflows/lint-and-convert.yml/badge.svg)](https://github.com/RedX300/detection-as-code/actions/workflows/lint-and-convert.yml)
[![Integration Test](https://github.com/RedX300/detection-as-code/actions/workflows/integration-test.yml/badge.svg)](https://github.com/RedX300/detection-as-code/actions/workflows/integration-test.yml)
 
A fully automated detection engineering pipeline that demonstrates the complete
detection lifecycle: authoring Sigma rules, validating via CI/CD, testing against
real attack telemetry with Splunk Attack Range, and deploying to Splunk.
 
## Architecture
Author (Sigma) -> Validate (CI) -> Test (Attack Range) -> Deploy (Splunk)
 
## Quick Start
1. Clone this repo
2. Install sigma-cli: `pip install sigma-cli pySigma-backend-splunk`
3. Run lint: `sigma check rules/`
4. Convert: `sigma convert -t splunk -p splunk_windows -p pipelines/splunk_pipeline.yml rules/`

See **[docs/README.md](docs/README.md)** for the full rule conversion guide: adding rules
from SigmaHQ, the custom pipeline, output formats, and troubleshooting.
 
## Detection Library
| # | Detection | ATT&CK | Status |
|---|-----------|--------|--------|
| 1 | LSASS Dump via Procdump | T1003.001 | Production |
 
## Project Structure
(paste directory tree here)
# detection-as-code
