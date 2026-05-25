## GitHub Actions를 사용하여 자동으로 Docker 이미지를 빌드하고 푸시하기
오늘은 GitHub Actions와 Jib 플러그인을 활용하여 자바 애플리케이션의 Docker 이미지를 자동으로 빌드하고 Amazon ECR(Elastic Container Registry)에 푸시하는 과정을 살펴보겠습니다. 이 방법은 CI/CD 파이프라인에서 매우 유용하게 사용될 수 있습니다.


### Gradle 설정 업데이트
먼저, build.gradle.kts 파일에 필요한 Jib 플러그인과 설정을 추가합니다. Jib은 Google에서 제공하는 도구로, Docker 데몬 없이 자바 애플리케이션을 컨테이너 이미지로 빌드할 수 있게 해줍니다.

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

### GitHub Actions 설정
GitHub Actions 워크플로우를 설정하여, main 브랜치에 태그가 생성될 때나 develop 브랜치에 Pull Request가 머지될 때 이미지 빌드 및 푸시가 자동으로 실행되도록 합니다.

.github/workflows/build.yml 파일을 생성하고 다음과 같이 작성합니다.


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

위 설정에 따라, 태그가 v* 형식으로 main 브랜치에 푸시되면 지정된 태그와 함께 이미지가 빌드되어 ECR로 푸시됩니다. develop 브랜치로의 Pull Request가 머지되는 경우에는 최신 이미지가 빌드되지만 태그 없이 푸시됩니다.


### Conclusion
GitHub Actions와 Jib을 사용하면, 복잡한 Docker 명령어 없이도 간단하게 Java 애플리케이션의 Docker 이미지를 빌드하고 관리할 수 있습니다. 이 방법을 통해 개발 팀은 보다 효율적으로 CI/CD 파이프라인을 구축하고 유지할 수 있습니다.
