---
title: "Automating Releases with GitHub Actions"
date: "2025-11-08"
draft: false
hideToc: true
tags: ["Gradle", "Maven Central", "Publishing", "CI", "Automation"]
series: "Tv Maniac Journey"
---

In my [previous post](https://thomaskioko.me/posts/publishing_gradle_plugins/), I walked through publishing Gradle plugins to Maven Central. While that worked, the manual process was tedious. Every release meant updating versions, creating tags, running publish commands, and writing release notes. Today, we're eliminating all that toil by automating the entire release pipeline with GitHub Actions.

## What is Continuous Delivery(CI/CD)?

Continuous Delivery (CD) is the practice of automating software releases so that code can be deployed to production at any time. Unlike Continuous Deployment (which automatically deploys every change), CD ensures your code is always in a releasable state but gives you control over when to release.

For Gradle plugins, this means automating the build, test, and publish pipeline. Instead of manually running commands locally, we use GitHub Actions to handle the entire process - from code validation to Maven Central publication. The result? Faster releases, fewer errors, and the confidence that every release follows the same proven process.

## The End Goal

The ideal workflow should be dead simple:

1. Update `CHANGELOG.md` with release notes
2. Create and push a git tag
3. Watch as GitHub Actions builds, publishes to Maven Central, and creates a GitHub Release

That's it. No manual builds, no credential juggling, no copy-pasting release notes.

![Automated Release](https://github.com/user-attachments/assets/ba727f61-96f3-4111-a85e-53eb5702b5e3)

## How It All Fits Together

Our automated release pipeline consists of three main components working together:

```
            ┌─────────────────────────────────────────────────┐
            │         Developer pushes tag v1.0.0             │
            └───────────────────┬─────────────────────────────┘
                                │
                                ▼
            ┌─────────────────────────────────────────────────┐
            │ GitHub Actions: Release Workflow Triggers       │
            └───────────────────┬─────────────────────────────┘
                                │
                                ├──► Build Job
                                │    ├─ Check formatting
                                │    ├─ Build & test
                                │    └─ Validate plugins
                                │
                                └──► Publish Job (After build)
                                    ├─ Extract version from tag
                                    ├─ Parse CHANGELOG.md
                                    ├─ Publish to Maven Central
                                    └─ Create GitHub Release
```

When you push a tag, the release workflow triggers. It first calls the build workflow to ensure everything compiles and tests pass. Only after successful validation does it proceed to publish. This prevents broken releases and ensures consistency.

## Building the Foundation

Before automating releases, we need a solid build workflow. This runs on every push and PR to catch issues early.

### Composite Actions for Reusability

Instead of repeating JDK and Gradle setup across workflows, we create a composite action at `.github/actions/gradle-setup/action.yml`. This encapsulates the setup logic with configurable inputs like Java version and wrapper validation.

### The Build Workflow

Our build workflow splits work into three parallel jobs:

- **Formatting** - Quick spotless check
- **Build** - Compile, test, and upload artifacts
- **Validate** - Check plugin descriptors using artifacts from build

The workflow also exposes `workflow_call`, allowing our release workflow to reuse it:

## Automating Releases

The release workflow triggers on any git tag and handles everything from building to publishing. Here's the structure:

```yaml
on:
  push:
    tags:
      - '**'

jobs:
  build:
    uses: ./.github/workflows/build.yml  # Reuse our build workflow

  publish:
    needs: build  # Only publish if build passes
    permissions:
      contents: write  # Required for creating releases
    steps:
      # Extract version, parse changelog, publish, create release
```

The workflow has two jobs: `build` reuses our existing build workflow, and `publish` handles the release process. Let's look at the key pieces:

### Tag-Based Versioning

We extract the version number from git tags. The workflow removes the `v` prefix (`v1.0.0` → `1.0.0`) and passes it to Gradle via environment variable:

```bash
VERSION=${GITHUB_REF_NAME#v}
echo "VERSION_NAME=$VERSION" >> $GITHUB_ENV
```

Gradle automatically picks up `ORG_GRADLE_PROJECT_VERSION_NAME` and overrides the version. No code changes needed!

### Extracting Release Notes

We parse `CHANGELOG.md` using awk to extract release notes automatically. The script finds the section matching the version and grabs everything until the next version header:

```markdown
## 1.0.0
- Added Android Multiplatform plugin
- Improved build performance
```

This gets extracted and used as the GitHub Release body.

### Publishing to Maven Central

The workflow uses `publishAndReleaseToMavenCentral` task with credentials from GitHub Secrets. We use in-memory GPG signing instead of file-based signing for better security in CI.

## Securing Credentials

You need four GitHub Secrets for publishing:

1. `CENTRAL_PORTAL_USERNAME` - Maven Central username
2. `CENTRAL_PORTAL_PASSWORD` - Maven Central user token
3. `MAVEN_SIGNING_PRIVATE_KEY` - GPG key in ASCII armor format
4. `MAVEN_SIGNING_PASSWORD` - GPG key passphrase


Once you have all these, add them to you Project Repository Secrets page.

## Releasing

With everything configured, releasing is simple:

**1. Update CHANGELOG.md**
```markdown
## 1.0.0
- Added Android Multiplatform plugin
- Improved build performance
```

**2. Commit and push**
```bash
git add CHANGELOG.md
git commit -m "Prepare v1.0.0 release"
git push
```

**3. Create and push tag**
```bash
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

That's it! GitHub Actions handles the rest - building, testing, publishing to Maven Central, and creating a GitHub Release.

![Maven Central Repository](https://github.com/user-attachments/assets/a9967ad9-7699-4b4b-bab3-ebed8276fc8f)

## Wrapping Up

Now releasing a new version means updating CHANGELOG.md and pushing a tag. GitHub Actions handles everything else. Until next time, happy coding! ✌️

**Related:**
- [Publishing Gradle Plugins to Maven Central](https://thomaskioko.me/posts/publishing_gradle_plugins/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Vanniktech Maven Publish Plugin](https://github.com/vanniktech/gradle-maven-publish-plugin)
