# CI/CD Cheatsheet

> Quick-reference concepts and config snippets for CI/CD pipelines.

## Core Commands / Concepts

### GitHub Actions — Key Concepts
```yaml
# Trigger on push to main
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test
```

### GitLab CI — Key Concepts
```yaml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - docker build -t myapp .

test:
  stage: test
  script:
    - npm test
```

### Jenkins — Key Concepts
```groovy
pipeline {
    agent any
    stages {
        stage('Build') { steps { sh 'npm install' } }
        stage('Test')  { steps { sh 'npm test' } }
        stage('Deploy') { steps { sh './deploy.sh' } }
    }
}
```

## Pipeline Stages

| Stage | Purpose |
|-------|---------|
| Checkout | Clone source code |
| Build | Compile / package application |
| Unit Test | Fast automated tests |
| SAST | Static code analysis |
| Integration Test | Test against dependencies |
| Build Image | Create Docker image |
| Push | Push to registry |
| Deploy | Deploy to environment |
| Smoke Test | Post-deploy sanity checks |

## Common Patterns

<!-- Add pipeline patterns: caching, matrix builds, conditional stages, reusable workflows here -->
