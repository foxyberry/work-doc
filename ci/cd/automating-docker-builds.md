## Automating Docker Builds with GitHub Actions and Jib
This guide simplifies the process of setting up automated Docker image builds for a Java application using GitHub Actions and the Jib plugin. This method integrates seamlessly into CI/CD pipelines, enhancing productivity and reducing manual deployment efforts.


### Update Gradle Configuration
First, add the necessary Jib plugin to your build.gradle.kts. Jib, developed by Google, enables building container images from Java applications without a Docker daemon.


```
// Import Jib plugins
import com.google.cloud.tools.jib.gradle.JibExtension
import com.google.cloud.tools.jib.gradle.JibPlugin

// Apply the Jib plugin
plugins {
  id("com.google.cloud.tools.jib") version "3.2.0"
}

// Configure Jib
plugins.withType<JibPlugin> {
    configure<JibExtension> {
        from {
            image = "openjdk:17-jdk-slim"
        }
        to {
            val tag = project.properties.getOrDefault("tag", "alpha")
            image = "296090417872.dkr.ecr.ap-southeast-1.amazonaws.com/dutti-english-teacher:$tag"
        }
    }
}

```

### Set Up GitHub Actions
Create a .github/workflows/build.yml to automate image builds and pushes triggered by new tags on main branch or merges into develop branch.


```
name: Build and Push Image

on:
  push:
    tags:
      - 'v*' # Trigger on tag push to main branch
  pull_request:
    branches:
      - develop # Trigger on PR merge to develop branch

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2

      - name: Set up JDK 17
        uses: actions/setup-java@v2
        with:
          java-version: 17

      - name: Build and Push Docker Image
        run: |
          if [ "${{ github.ref_type }}" = "tag" ]; then
            ./gradlew jib --tag="${{ github.ref }}"
          elif [ "${{ github.event_name }}" = "pull_request" ]; then
            ./gradlew jib
          fi

```


### Conclusion
Using GitHub Actions and Jib for Docker image builds streamlines the deployment process for Java applications, eliminating the need for direct Docker command manipulation. This setup facilitates a more efficient and robust CI/CD pipeline, ideal for development teams aiming to automate and simplify their workflow.