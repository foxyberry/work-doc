
# Multi stage build 적용 방법 

### multi stage build 적용 개선 방안

1) 이유 
 - 도커 빌드 시간 감소: 기존 방식에서는 소소한 변경에도 전체 소스 코드를 다시 복사하고 빌드하는 과정을 거쳐야 했습니다. 이는 비효율적이며 시간 소모가 큰 작업이었습니다.

2) 장점 
- 이미지 크기 감소 및 배포 속도 향상: 불필요한 빌드 도구나 중간 생성물을 최종 이미지에서 제거함으로써, 이미지 크기가 줄어들고, 이는 배포 속도의 개선으로 이어집니다.
- 의존성 관리의 용이성: 빌드 단계에서만 필요한 의존성을 관리하고, 최종 이미지에는 필수적인 요소만 포함시키므로 의존성 관리가 간편해집니다.
- 가독성 및 유지보수성 향상: 소스 코드 복사와 빌드 과정을 분리하여 관리함으로써, Dockerfile의 구조가 명확해지고 유지보수가 용이해집니다.

3) 단점 
- 전체 빌드 프로세스 시간 증가 가능성: 각 단계마다 필요한 작업을 수행함으로써 단일 스테이지 빌드보다 시간이 길어질 수 있습니다. 이는 초기 설정과 캐시 최적화를 통해 다소 완화될 수 있습니다.
- 캐싱 관리의 중요성: 각 빌드 스테이지에서 적절한 캐싱 전략을 사용하지 않으면, 예상치 못한 빌드 시간 증가나 빌드 실패로 이어질 수 있습니다.



### 프로젝트 구조
```
├── application // Web Server
├── batch
│    ├── etl  // Data ETL
├── common
│    ├──  data // ES Data
│    ├──  util // Common Utils
├── build.gradle.kts // root build file 
├── docker-compose-*.yml // docker file for Dev Env
├── Dockerfile // docker build file for Service 
├── settings.gradle.kts // setting for module 
```

## 기존 방식
#### Dockerfile
```
FROM openjdk:17-jdk-slim

WORKDIR /app
COPY . .

RUN ./gradlew :application:build
ARG JAR_FILE=/app/application/build/libs/application-0.0.1-SNAPSHOT.jar
RUN cp ${JAR_FILE} app.jar

EXPOSE 8080

CMD ["java", "-jar", "app.jar"]
```



## 개선된 방식 

빌드 스테이지 개선
- 소스 코드 복사 최적화: COPY . . 대신에 필요한 파일만 선별적으로 복사함으로써, 소스 코드 변경 시 캐시 활용도를 극대화하고 빌드 시간을 단축시킵니다.
- 의존성 다운로드 최적화: --no-daemon --refresh-dependencies 옵션을 사용하여 필요한 의존성만 정확하게 다운로드하고, 불필요한 데몬 프로세스 실행을 방지합니다.

실행 스테이지 개선
- 최종 이미지 최적화: 빌드 스테이지에서 생성된 실행 가능한 JAR 파일만 최종 이미지로 복사하여, 런타임에 필요한 최소한의 의존성과 파일만 포함시키는 전략을 사용합니다. 이를 통해 실행 이미지의 크기를 최소화하고, 보안 및 성능을 개선합니다.

#### Dockerfile
```
# 빌드 스테이지
FROM gradle:7.6-jdk AS build
WORKDIR /app

# 필요한 빌드 파일 복사
COPY settings.gradle.kts .
COPY common/data/build.gradle.kts common/data/
COPY common/util/build.gradle.kts common/util/
COPY application/build.gradle.kts application/

# 의존성 다운로드
RUN gradle --no-daemon --refresh-dependencies

# 소스 코드 복사
COPY common/data/src common/data/src
COPY common/util/src common/util/src
COPY application/src application/src

# 어플리케이션 빌드
RUN gradle :application:build -PskipTests --no-daemon

# 실행 스테이지
FROM openjdk:17-jdk-slim
WORKDIR /app

COPY --from=build /app/application/build/libs/application-*.jar app.jar

EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```