# Maven Central 라이브러리 배포 가이드

이 가이드는 Gradle 프로젝트를 Maven Central에 배포하는 전체 과정을 다룹니다.

## 목차

1. [사전 준비](#1-사전-준비)
2. [Sonatype Central 계정 설정](#2-sonatype-central-계정-설정)
3. [GPG 키 생성 및 설정](#3-gpg-키-생성-및-설정)
4. [Gradle 설정](#4-gradle-설정)
5. [GitHub Actions CI/CD 설정](#5-github-actions-cicd-설정)
6. [배포 및 확인](#6-배포-및-확인)
7. [다른 프로젝트에서 사용하기](#7-다른-프로젝트에서-사용하기)
8. [트러블슈팅](#8-트러블슈팅)

---

## 1. 사전 준비

### 필요한 것들

- GitHub 계정
- Sonatype Central 계정
- GPG (Windows: Gpg4win)
- Gradle 프로젝트

### GPG 설치 (Windows)

1. [Gpg4win](https://www.gpg4win.org/download.html) 다운로드 (무료)
2. 설치 후 PowerShell 재시작
3. 설치 확인:
   ```powershell
   gpg --version
   ```

---

## 2. Sonatype Central 계정 설정

### 2.1 계정 생성

1. [https://central.sonatype.com/](https://central.sonatype.com/) 접속
2. GitHub 계정으로 로그인

### 2.2 Namespace 등록

GitHub 계정으로 로그인하면 `io.github.{username}` 형식의 namespace를 사용할 수 있습니다.

1. **Namespaces** 메뉴 클릭
2. **Add Namespace** 클릭
3. `io.github.{your-github-username}` 또는 `io.github.{your-organization}` 입력

### 2.3 Namespace 인증 (Organization의 경우)

Organization namespace 인증이 필요한 경우:

1. Sonatype에서 제공하는 **Verification Key** 확인 (예: `ckt815j1im`)
2. GitHub Organization에 해당 이름의 **Public 저장소** 생성
    - 예: `https://github.com/jun-bank/ckt815j1im`
3. Sonatype에서 `⋯` → **Verify** 클릭
4. 인증 완료 후 저장소 삭제 가능

### 2.4 User Token 생성

1. Sonatype Central → 우측 상단 계정 → **View Account**
2. **Generate User Token** 클릭
3. Username과 Password 복사해두기 (나중에 사용)

---

## 3. GPG 키 생성 및 설정

### 3.1 GPG 키 생성

```powershell
gpg --full-generate-key
```

선택 옵션:
- Kind of key: **1 (RSA and RSA)**
- Keysize: **4096**
- Valid for: **0** (만료 없음)
- Real name: 본인 이름
- Email: GitHub 이메일
- Passphrase: 비밀번호 설정 **(꼭 기억해두세요!)**

### 3.2 키 ID 확인

```powershell
gpg --list-keys --keyid-format SHORT
```

출력 예시:
```
pub   rsa4096/C4061326 2025-11-30 [SC]
      8798249839785E6F47E6CBD2D19D591CC4061326
uid         [ultimate] jun <pickjog@gmail.com>
sub   rsa4096 2025-11-30 [E]
```

여기서 `C4061326`이 **키 ID**입니다 (마지막 8자리).

### 3.3 키서버에 공개키 업로드

```powershell
gpg --keyserver keyserver.ubuntu.com --send-keys {키ID}
```

예시:
```powershell
gpg --keyserver keyserver.ubuntu.com --send-keys C4061326
```

### 3.4 비밀키 내보내기 (CI/CD용)

```powershell
gpg --export-secret-keys --armor {키ID} > gpg-key.asc
```

이 파일 내용은 나중에 GitHub Secret에 등록합니다.

### 3.5 로컬 Gradle 설정 (선택사항)

로컬에서 배포하려면 `~/.gradle/gradle.properties` 파일 생성:

```properties
ossrhUsername=Sonatype_토큰_Username
ossrhPassword=Sonatype_토큰_Password

signing.keyId=C4061326
signing.password=GPG_비밀번호
signing.secretKeyRingFile=C:/Users/{username}/AppData/Roaming/gnupg/secring.gpg
```

secring.gpg 파일 생성 (GPG 2.1+ 버전):
```powershell
gpg --export-secret-keys -o C:\Users\{username}\AppData\Roaming\gnupg\secring.gpg
```

---

## 4. Gradle 설정

### 4.1 build.gradle 전체 예시

```groovy
plugins {
    id 'java-library'
    id 'org.springframework.boot' version '4.0.0'
    id 'io.spring.dependency-management' version '1.1.7'
    id 'maven-publish'
    id 'signing'
    id 'tech.yanand.maven-central-publish' version '1.3.0'
}

group = 'io.github.jun-bank'  // 본인의 namespace로 변경
version = '0.0.1'
description = 'Your Library Description'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
    withSourcesJar()   // 소스 JAR 필수
    withJavadocJar()   // Javadoc JAR 필수
}

repositories {
    mavenCentral()
}

dependencies {
    // 의존성 정의
}

// 라이브러리인 경우 bootJar 비활성화
bootJar {
    enabled = false
}

jar {
    enabled = true
}

// ========================================
// Maven Central 배포 설정
// ========================================
publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java

            versionMapping {
                usage('java-api') {
                    fromResolutionOf('runtimeClasspath')
                }
                usage('java-runtime') {
                    fromResolutionResult()
                }
            }

            pom {
                name = 'Your Library Name'
                description = 'Your library description'
                url = 'https://github.com/your-org/your-repo'

                licenses {
                    license {
                        name = 'The Apache License, Version 2.0'
                        url = 'https://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }

                developers {
                    developer {
                        id = 'your-github-id'
                        name = 'Your Name'
                        email = 'your-email@example.com'
                    }
                }

                scm {
                    connection = 'scm:git:git://github.com/your-org/your-repo.git'
                    developerConnection = 'scm:git:ssh://github.com:your-org/your-repo.git'
                    url = 'https://github.com/your-org/your-repo'
                }
            }
        }
    }
}

mavenCentral {
    authToken = System.getenv("MAVEN_CENTRAL_TOKEN")
    publishingType = "USER_MANAGED"  // 수동 승인, 자동은 "AUTOMATIC"
    maxWait = 120
}

signing {
    def signingKeyId = System.getenv("SIGNING_KEY_ID")
    def signingPassword = System.getenv("SIGNING_PASSWORD")
    def signingKey = System.getenv("GPG_SIGNING_KEY")

    if (signingKey != null && !signingKey.isEmpty()) {
        useInMemoryPgpKeys(signingKeyId, signingKey, signingPassword)
    }

    sign publishing.publications.mavenJava
}

tasks.withType(Javadoc) {
    failOnError = false
    options.addStringOption('Xdoclint:none', '-quiet')
    options.encoding = 'UTF-8'
}
```

### 4.2 주요 설정 설명

| 설정 | 설명 |
|------|------|
| `withSourcesJar()` | 소스 코드 JAR 생성 (Maven Central 필수) |
| `withJavadocJar()` | Javadoc JAR 생성 (Maven Central 필수) |
| `versionMapping` | BOM에서 관리하는 의존성 버전 해결 |
| `pom` | Maven Central 필수 메타데이터 |
| `publishingType` | USER_MANAGED(수동) 또는 AUTOMATIC(자동) |
| `useInMemoryPgpKeys` | CI/CD 환경에서 메모리 기반 GPG 키 사용 |

---

## 5. GitHub Actions CI/CD 설정

### 5.1 GitHub Secrets 등록

**Organization Settings** → **Secrets and variables** → **Actions**에 추가:

| Secret 이름 | 값 | 설명 |
|-------------|-----|------|
| `OSSRH_USERNAME` | Sonatype 토큰 Username | Token 생성 시 발급 |
| `OSSRH_PASSWORD` | Sonatype 토큰 Password | Token 생성 시 발급 |
| `GPG_KEY_ID` | `C4061326` | GPG 키 ID (8자리) |
| `GPG_PASSPHRASE` | GPG 비밀번호 | 키 생성 시 설정한 비밀번호 |
| `GPG_PRIVATE_KEY` | GPG 비밀키 내용 | `gpg-key.asc` 파일 내용 |
| `MAVEN_CENTRAL_TOKEN` | Base64 인코딩된 인증 정보 | 아래 참고 |

### 5.2 MAVEN_CENTRAL_TOKEN 생성

PowerShell에서 Base64 인코딩:

```powershell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("OSSRH_USERNAME값:OSSRH_PASSWORD값"))
```

출력된 값을 `MAVEN_CENTRAL_TOKEN` Secret으로 저장합니다.

### 5.3 워크플로우 파일

`.github/workflows/publish.yml`:

```yaml
name: Publish to Maven Central

on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
      - 'build.gradle'
      - 'gradle.properties'
  release:
    types: [created]
  workflow_dispatch:  # 수동 실행

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      # 1. 소스 체크아웃
      - name: Checkout repository
        uses: actions/checkout@v4

      # 2. Java 설정
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      # 3. GPG 키 가져오기
      - name: Import GPG key
        run: |
          echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --batch --import
          gpg --list-keys

      # 4. Gradle 설정
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      # 5. 빌드 권한
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      # 6. 빌드 및 테스트
      - name: Build with Gradle
        run: ./gradlew build

      # 7. Maven Central에 배포
      - name: Publish to Maven Central
        run: ./gradlew publishToMavenCentralPortal
        env:
          MAVEN_CENTRAL_TOKEN: ${{ secrets.MAVEN_CENTRAL_TOKEN }}
          SIGNING_KEY_ID: ${{ secrets.GPG_KEY_ID }}
          SIGNING_PASSWORD: ${{ secrets.GPG_PASSPHRASE }}
          GPG_SIGNING_KEY: ${{ secrets.GPG_PRIVATE_KEY }}

      # 8. 배포 결과 요약
      - name: Publish Summary
        run: |
          echo "## 📦 Published to Maven Central" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "- **Group:** io.github.jun-bank" >> $GITHUB_STEP_SUMMARY
          echo "- **Artifact:** common-lib" >> $GITHUB_STEP_SUMMARY
          echo "- **Version:** 0.0.1" >> $GITHUB_STEP_SUMMARY
```

---

## 6. 배포 및 확인

### 6.1 배포 실행

**방법 1: GitHub Actions (자동)**
- main 브랜치에 push하면 자동 실행
- 또는 Actions 탭에서 수동 실행 (workflow_dispatch)

**방법 2: 로컬 (수동)**
```powershell
./gradlew publishToMavenCentralPortal
```

### 6.2 배포 상태 확인

1. [https://central.sonatype.com/](https://central.sonatype.com/) 접속
2. **Deployments** 탭 클릭
3. 상태 확인:
    - **PENDING**: 검증 중
    - **VALIDATED**: 검증 완료, Publish 가능
    - **PUBLISHED**: 배포 완료
    - **FAILED**: 오류 발생

### 6.3 수동 Publish (USER_MANAGED인 경우)

`publishingType = "USER_MANAGED"` 설정 시:

1. Deployments에서 **VALIDATED** 상태 확인
2. **Publish** 버튼 클릭
3. **PUBLISHED** 상태가 되면 완료

### 6.4 Maven Central에서 검색

배포 완료 후 (몇 분 ~ 30분 소요):

- 검색: [https://central.sonatype.com/search](https://central.sonatype.com/search)
- 직접 URL: `https://central.sonatype.com/artifact/{group}/{artifact}/{version}`

---

## 7. 다른 프로젝트에서 사용하기

### 7.1 build.gradle

```groovy
repositories {
    mavenCentral()
}

dependencies {
    implementation 'io.github.jun-bank:common-lib:0.0.1'
}
```

### 7.2 pom.xml (Maven)

```xml
<dependency>
    <groupId>io.github.jun-bank</groupId>
    <artifactId>common-lib</artifactId>
    <version>0.0.1</version>
</dependency>
```

---

## 8. 트러블슈팅

### Q1: `401 Unauthorized` 오류

**원인**: 인증 정보가 잘못됨

**해결**:
- Sonatype Token이 만료되었는지 확인
- `MAVEN_CENTRAL_TOKEN`이 올바르게 Base64 인코딩되었는지 확인
- 형식: `Base64(username:password)`

### Q2: `Cannot perform signing task` 오류

**원인**: GPG 서명 설정 문제

**해결**:
- `GPG_PRIVATE_KEY` Secret에 `gpg-key.asc` 파일 내용이 올바르게 들어갔는지 확인
- `GPG_KEY_ID`가 8자리 키 ID인지 확인
- `GPG_PASSPHRASE`가 정확한지 확인

### Q3: `Invalid publication - dependencies without versions` 오류

**원인**: BOM에서 관리하는 의존성에 버전이 명시되지 않음

**해결**: `versionMapping` 블록 추가
```groovy
versionMapping {
    usage('java-api') {
        fromResolutionOf('runtimeClasspath')
    }
    usage('java-runtime') {
        fromResolutionResult()
    }
}
```

### Q4: `repoDir has no value available` 오류

**원인**: 플러그인 버전 문제

**해결**: 플러그인 버전을 `1.3.0`으로 업그레이드
```groovy
id 'tech.yanand.maven-central-publish' version '1.3.0'
```

### Q5: Namespace 인증 실패

**원인**: GitHub 저장소가 없거나 Private임

**해결**:
- Verification Key와 동일한 이름의 **Public** 저장소 생성
- 저장소 생성 후 Sonatype에서 다시 Verify

---

## 참고 자료

- [Sonatype Central Portal](https://central.sonatype.com/)
- [Maven Central 배포 요구사항](https://central.sonatype.org/publish/requirements/)
- [GPG 서명 가이드](https://central.sonatype.org/publish/requirements/gpg/)
- [tech.yanand.maven-central-publish 플러그인](https://github.com/yananhub/flying-gradle-plugin)
- [Gradle Maven Publish Plugin](https://docs.gradle.org/current/userguide/publishing_maven.html)

---

## 버전 히스토리

| 버전 | 날짜 | 변경 내용 |
|------|------|----------|
| 0.0.1 | 2025-11-30 | 최초 배포 |