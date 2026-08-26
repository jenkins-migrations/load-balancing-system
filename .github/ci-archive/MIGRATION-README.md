# Jenkins to GitHub Actions migration report

## Summary

Issue #3 migrates the repository's Jenkins declarative pipeline to
`.github/workflows/maven-deploy.yml`. The original `Jenkinsfile` is preserved
unchanged at `.github/ci-archive/Jenkinsfile` and removed from the repository
root. The repository contains no shared libraries.

## Repository evidence

- The archived Jenkins declarative pipeline uses `agent any` and four ordered
  stages: unit test, standalone deployment, Anypoint ARM deployment, and
  CloudHub deployment.
- The Jenkins credential binding is `credentials('anypoint.credentials')`.
  Its username is referenced as `ANYPOINT_CREDENTIALS_USR`; both password
  command fragments are redacted as `-Danypoint.******`.
- The ARM target is `local-4.0.0-ee`, and the CloudHub Mule version is `4.0.0`.
- The source defines no Jenkins trigger. `README.md` only identifies this as a
  professional DevOps and CI/CD repository and supplies no additional build,
  runtime, trigger, or deployment requirements.

## Trigger and permissions

- `workflow_dispatch` is the only trigger. This preserves the source's lack of
  an automatic trigger while providing an explicit deployment entry point.
- Workflow token access is restricted to `contents: read`.
- A per-ref concurrency group prevents overlapping deployments without
  cancelling an in-progress deployment.
- The job has a 60-minute timeout.

## Pipeline mapping

| Jenkins source | GitHub Actions equivalent |
| --- | --- |
| `agent any` | `ubuntu-latest` with Temurin Java 8 |
| `mvn clean test` | `Unit test` step |
| `mvn deploy -P standalone` | `Deploy standalone` step |
| ARM profile and `local-4.0.0-ee` target | `Deploy to Anypoint ARM` step |
| CloudHub profile and Mule `4.0.0` | `Deploy to CloudHub` step |
| Jenkins credential binding | Step-scoped GitHub Actions secrets |
| Ordered Jenkins stages | Ordered steps in one job; failure stops later steps |
| `post { success/failure }` | Conditional success and failure reporting steps |

`actions/checkout` v7.0.1 and `actions/setup-java` v6.0.0 are maintained by
GitHub and pinned to full commit SHAs. Checkout credential persistence is
disabled because later steps do not need Git credentials.

## Required secrets

Configure these repository or environment secrets before dispatching the
workflow:

| GitHub secret | Source mapping | Usage |
| --- | --- | --- |
| `ANYPOINT_USERNAME` | Jenkins `ANYPOINT_CREDENTIALS_USR` | Maven `anypoint.username` for ARM and CloudHub |
| `ANYPOINT_PASSWORD` | Redacted Jenkins password credential | Maven `anypoint.password` for ARM and CloudHub |

The exact source password property is unavailable because both Jenkins command
fragments are redacted. The migration uses the standard
`-Danypoint.password` Maven property. Secrets are injected only into the two
steps that need them and are not stored in the workflow.

## Limitations and operator checks

- No `pom.xml`, Maven wrapper, application source, or deployment configuration
  is present in this repository, so Maven execution cannot be validated here.
- Jenkins did not specify an operating system or Java version. Ubuntu and Java
  8 are explicit migration choices compatible with the source's Mule 4.0.0
  deployment intent; confirm them against the actual Mule project.
- Confirm that the real ARM and CloudHub Maven profiles accept
  `anypoint.username` and `anypoint.password`, particularly because the
  password arguments were redacted.
- Configure any Maven repository credentials or deployment environment
  protections required outside the provided source.

## Validation

- Workflow YAML was syntax-parsed locally.
- Action references were checked for full 40-character commit SHA pins.
- Stage order, ARM target, Mule version, permissions, and required secret names
  were checked against the archived Jenkinsfile.
- Secret scanning found no committed credentials.
- `actionlint` was not installed in the migration environment.
- The GitHub Advisory Database check for the pinned actions could not run
  because the validation environment had no GitHub token.
- Maven was not executed because the repository has no `pom.xml`.

## Rollback and archive

To roll back, move `.github/ci-archive/Jenkinsfile` back to `/Jenkinsfile` and
remove `.github/workflows/maven-deploy.yml`. The archive is intentionally kept
in version control for auditability; no Jenkins source remains active at the
repository root.
