---
name: xerces-release
description: >-
  Releases martinhickson/xerces-j from the maintenance branch. Use when
  dispatching the Release workflow, bumping a bravura version, publishing
  to GitHub Packages or Maven Central, or changing the Xerces JDK,
  groupId, or artifactId. Covers the JDK 11 build, the xercesImpl HTTP 422
  failure, and the POM restore rules.
---

# Xerces-J release

Remote: `martinhickson/xerces-j`. Release from `maintenance`. The workflow file is `.github/workflows/release.yml` (`workflow_dispatch` only). CI is `.github/workflows/ci.yml`.

`pom.xml` is CRLF. A match on `<profiles>` must allow the carriage return.

## Coordinates

The committed POM stays `xerces:xercesImpl`. `maven.compiler.release` is 11, so the compiler source and target level are 11. Do not commit a rewritten groupId or artifactId.

| Destination | groupId | artifactId | When |
|---|---|---|---|
| Git, tag, GitHub Release assets | `xerces` | `xercesImpl` | Always |
| Maven Central | `io.github.martinhickson` | `xercesImpl` | `deploy_to_maven_central` is true. Rewrite for the deploy, then restore the POM |
| GitHub Packages | `xerces` | `xercesimpl` | `upload_to_github_packages` is true. Lowercase for the deploy, then restore the POM |

Central and GitHub Packages can both run. Central publishes first. GitHub Packages then deploys the restored `xerces` POM. Collected release assets are copied before the GitHub Packages rewrite, so the GitHub Release still attaches `xercesImpl-*.jar`.

Confirm a Central jar on repo1 (`io/github/martinhickson/xercesImpl/<version>/`). `search.maven.org` can lag behind a publish that already succeeded.

## JDK

The build is Maven. Do not run Ant, and do not add `maven-antrun-plugin`. CI and the release workflow compile with Temurin 11. `maven.compiler.release` is 11. Do not set `java-version` to 17 or 21, and do not lower the compiler level below 11. JDK 11 already owns `org.w3c.dom.html`, so `HTMLDOMImplementation.java` is excluded from the main compile and compiled with `--patch-module jdk.xml.dom`. The release job installs Maven 3.9.9 itself. The release version is `mvn versions:set` only.

This repo has no `formatter-maven-plugin`. `mvn formatter:format` fails with `NoPluginFoundForPrefixException`.

## Dispatch

If `gh` returns 401 while `gh auth status` shows a valid `hosts.yml` login, `GITHUB_TOKEN` in the environment is overriding it. Prefix GitHub CLI calls with `env -u GITHUB_TOKEN`.

Dispatch the workflow that is already on the remote ref. A local edit does nothing until it is pushed. Do not re-run a failed run id after changing the workflow; that run keeps the old file. Use `--ref maintenance`.

```bash
env -u GITHUB_TOKEN gh workflow run release.yml --ref maintenance \
  -f version=VERSION \
  -f release_branch=release/VERSION \
  -f source_branch=maintenance \
  -f upload_to_github_packages=true \
  -f deploy_to_maven_central=false \
  -f central_group_id=io.github.martinhickson
```

`deploy_to_maven_central` defaults to true. A GitHub Packages-only cut must pass `false`.

Read `git tag -l` before choosing a version. Do not reuse a tag. When asked for a bumped version, increment past the last attempt even if that attempt never tagged.

Version lines already used:

- Maven Central: `2.12.3-bravura-<n>` (tags `2.12.3-bravura-1`, `2.12.3-bravura-2`)
- GitHub Packages only: `2.12.2.<n>-bravura-<n>` (published `2.12.2.8-bravura-8`). Earlier `2.12.2.4`, `2.12.2.5`, and `2.12.2.7` attempts failed before tagging

The version commit, release branch `release/<version>`, and annotated tag are created by the workflow. `maintenance` is not version-bumped.

## GitHub Packages

Publishing `xercesImpl` returns HTTP 422. GitHub Packages rejects an artifactId that contains an uppercase letter. The repository is `xerces-j`, so the deployed POM also needs a project-level `distributionManagement` entry or GitHub looks for a repository named after the package.

The upload step must:

1. Run only after `target/` exists. When Central is off, `mvn -B package -DskipTests` runs first. Collect then copies `target/xercesImpl-*.jar` and `target/xercesImpl-*.pom`, excluding sources and javadoc jars.
2. Copy `pom.xml` aside.
3. Replace the first `<artifactId>xercesImpl</artifactId>` with `xercesimpl`.
4. Insert this block immediately before `  <profiles>` (CRLF-safe):

```xml
  <distributionManagement>
    <repository>
      <id>github</id>
      <name>GitHub Packages</name>
      <url>https://maven.pkg.github.com/martinhickson/xerces-j</url>
    </repository>
  </distributionManagement>
```

5. Deploy with `mvn -B deploy -DskipTests`. The Maven server id `github` uses `GITHUB_ACTOR` and `GITHUB_TOKEN`. The job needs `packages: write`. Do not pass `altDeploymentRepository`; the POM URL is the repository.
6. Restore the saved POM. Fail if `xercesimpl` or `maven.pkg.github.com` remains.

Upload runs before commit, push, and tag. A failed upload must not create a tag.

Collect must not pipe `find` under `set -o pipefail`. A missing `target/` used to exit 1 before the "no artifacts" warning, which skipped the upload. Keep the `find` in a process substitution, and skip it when `target/` is absent.

## Maven Central

Leave this path in place. It copies the POM, runs `.github/scripts/rewrite-group-id.sh xerces <central_group_id>`, deploys with `-Pcentral-release`, then restores the POM. Fail if the Central groupId is still in the POM. The `central-release` profile holds the Central `distributionManagement`, signing, and javadoc. Do not put the GitHub Packages URL in that profile.
