# 안드로이드 어플리케이션 CI/CD 파이프라인 구축

[targetSdkVersion & compileSdkVersion 살펴보기](https://medium.com/google-developers/picking-your-compilesdkversion-minsdkversion-targetsdkversion-a098a0341ebd#.bpqsg4ili)

빌드와 배포용 도구로 Azure Pipelines를 사용합니다.
1. develop 브랜치로 코드가 푸시되면 트리거되도록 설정합니다.
```yml
trigger:
  branches:
    include:
      - develop
```
2. platform-tools 31.0.3에서 오류가 나서 cmd line에서 31.0.1로 변경했습니다.</br>
`$ANDROID_HOME`은 Azure에서 사전에 지정해 놓은 환경변수로 **/Users/runner/Library/Android/sdk/**의 경로가 지정되어 있습니다.
```yml
- script: |
      rm -rf $ANDROID_HOME/platform-tools/ && \
      cd $ANDROID_HOME                         && \
      curl -sS https://dl.google.com/android/repository/d027ce0f9f214a4bd575a73786b44d8ccf7e7516.platform-tools_r31.0.1-darwin.zip > platform-tools.zip && \
      unzip platform-tools.zip                                  && \
      rm platform-tools.zip
  displayName: Download platform-tools
```
3. Azure Pipelines Library에 업로드한 google-services.json 파일을 다운로드 합니다.</br>
CI 스크립트에서 불러올 google-services.json 파일을 Azure DevOps Library에 미리 업로드 하였습니다.

![Azure Pipelines Library](/infra/images/azure_pipelines_tab_library.png)
![Google Service Json](/infra/images/secret_google_json.png)
```yml
- task: DownloadSecureFile@1
  name: download_google_services_json
  displayName: Download google-services json file
  inputs:
    secureFile: google-services.json
    retryCount: 5
```
4. google-services.json을 작업 디렉토리로 이동합니다.
`$(Agent.TempDirectory)`는 Azure에서 지정한 default 임시 디렉토리이며, 실제 경로는 **/Users/runner/work/_temp/**입니다.
`$(Build.SourcesDirectory)`도 Azure에서 지정한 default 작업 디렉토리이며, 실제 경로는 **/Users/runner/work/1/s/**입니다.
```yml
- script: |
    mv $(Agent.TempDirectory)/google-services.json $(Build.SourcesDirectory)/app/
```
5. Azure DevOps Library에 미리 업로드한 xxx.jks 파일을 다운로드 합니다.(App Signing Key)

![Signing Key](/infra/images/secret_app_signing_key.png)
```yml
- task: DownloadSecureFile@1
  name: download_signing_file
  displayName: Download signing file
  inputs:
    secureFile: xxx.jks
    retryCount: 5
```
6. App bundle로 빌드합니다.
![앱 번들 정책](images/../../images/app-bundle-rule.png)
```yml
  - task: Gradle@2
    displayName: Build App
    inputs:
      gradleWrapperFile: gradlew
      workingDirectory: $(Build.SourcesDirectory)
      tasks: :app:bundleRelease
      publishJUnitResults: false
      javaHomeOption: JDKVersion
      jdkVersionOption: 11
      gradleOptions: -Xmx3072m
      sonarQubeRunAnalysis: false
```
7. App bundle에 Signing (google play store console에 등록한 jks 전용)
// 미완성
```yml
 - task: CmdLine@2
   displayName: Signing and aligning AAB file(s)
   inputs:
      script: jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 -keystore $(Agent.TempDirectory)/hayd_android.jks -storepass $(KEYSTORE-PASSWORD) -keypass $(KEY-PASSWORD) $(system.defaultworkingdirectory)/app/build/outputs/bundle/productionRelease/app-production-release.aab $(KEY-ALIAS)
```
8. Signing 완료된 App bundle 파일을 작업 디렉토리로 복사합니다.
```yml
 - task: CopyFiles@2
   displayName: Copy AAB file(s)
   inputs:
      SourceFolder: $(system.defaultworkingdirectory)/app/build/outputs/bundle/productionRelease/
      Contents: '**/*.aab'
      TargetFolder: $(Build.SourcesDirectory)/release/
```
9. App bundle publish
```yml
- task: PublishPipelineArtifact@1
  name: publish_pipeline_aab
  displayName: Publish pipeline AAB
  inputs:
    targetPath: $(Build.SourcesDirectory)/release/app-production-release.aab
    artifact: release-aab
    publishLocation: pipeline
```
10. 슬랙 APK 배포 채널에 APK 파일을 공유하기 위해 .abb -> .apk로 확장자를 변경해야 합니다. 이에 필요한 bundle tools 설치합니다.
```yml
- task: InstallBundletool@1
  displayName: Install bundle tool
  inputs:
    username: $(github-username)
    personalAccessToken: $(github-personal-access-token)
```
11. App bundle을 APK 확장자로 변환하는 작업 입니다.(여기서도 signing 작업을 위해 signing key를 파라미터로 전달합니다.)
```yml
 - task: AabConvertToUniversalApk@1
   displayName: Convert AAB to APK
   inputs:
      aabFilePath: $(Build.SourcesDirectory)/release/app-production-release.aab
      keystoreFilePath: $(Agent.TempDirectory)/hayd_android.jks
      keystorePassword: $(KEYSTORE-PASSWORD)
      keystoreAlias: $(KEY-ALIAS)
      keystoreAliasPassword: $(KEY-PASSWORD)
      outputFolder: $(Build.SourcesDirectory)/release/
```
12. APKs publish
```yml
- task: PublishPipelineArtifact@1
  name: publish_pipeline_apk
  displayName: Publish pipeline APK
  inputs:
    targetPath: $(Build.SourcesDirectory)/release/universal.apk
    artifact: release-apk
    publishLocation: pipeline
```

**최종 코드**
```yml
# Android
# Build your Android project with Gradle.
# Add steps that test, sign, and distribute the APK, save build artifacts, and more:
# https://docs.microsoft.com/azure/devops/pipelines/languages/android

trigger:
  branches:
    include:
      - develop

pool:
  vmImage: macOS-12

steps:
  - script: |
      rm -rf $ANDROID_HOME/platform-tools/ && \
      cd $ANDROID_HOME                         && \
      curl -sS https://dl.google.com/android/repository/d027ce0f9f214a4bd575a73786b44d8ccf7e7516.platform-tools_r31.0.1-darwin.zip > platform-tools.zip && \
      unzip platform-tools.zip                                  && \
      rm platform-tools.zip
    displayName: Download platform-tools

  - task: DownloadSecureFile@1
    name: download_google_services_json
    displayName: Download google-services json file
    inputs:
      secureFile: google-services.json
      retryCount: 5

  - script: |
      mv $(Agent.TempDirectory)/google-services.json $(Build.SourcesDirectory)/app/

  - task: DownloadSecureFile@1
    name: download_privates_gradle
    displayName: Download privates gradle file
    inputs:
      secureFile: privates-dev.gradle
      retryCount: 5

  - script: |
      mv $(Agent.TempDirectory)/privates-dev.gradle $(Build.SourcesDirectory)/
      mv $(Build.SourcesDirectory)/privates-dev.gradle $(Build.SourcesDirectory)/privates.gradle

  - task: DownloadSecureFile@1
    name: download_signing_file
    displayName: Download signing file
    inputs:
      secureFile: hayd_android.jks
      retryCount: 5

  - task: Gradle@2
    displayName: Build App
    inputs:
      gradleWrapperFile: gradlew
      workingDirectory: $(Build.SourcesDirectory)
      tasks: :app:bundleRelease
      publishJUnitResults: false
      javaHomeOption: JDKVersion
      jdkVersionOption: 11
      gradleOptions: -Xmx3072m
      sonarQubeRunAnalysis: false

  - task: CmdLine@2
    displayName: Signing and aligning AAB file(s)
    inputs:
      script: jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 -keystore $(Agent.TempDirectory)/hayd_android.jks -storepass $(KEYSTORE-PASSWORD) -keypass $(KEY-PASSWORD) $(system.defaultworkingdirectory)/app/build/outputs/bundle/productionRelease/app-production-release.aab $(KEY-ALIAS)

  - task: CopyFiles@2
    displayName: Copy AAB file(s)
    inputs:
      SourceFolder: $(system.defaultworkingdirectory)/app/build/outputs/bundle/productionRelease/
      Contents: '**/*.aab'
      TargetFolder: $(Build.SourcesDirectory)/release/

  - task: PublishPipelineArtifact@1
    name: publish_pipeline_aab
    displayName: Publish pipeline AAB
    inputs:
      targetPath: $(Build.SourcesDirectory)/release/app-production-release.aab
      artifact: release-aab
      publishLocation: pipeline

  - task: InstallBundletool@1
    displayName: Install bundle tool
    inputs:
      username: $(github-username)
      personalAccessToken: $(github-personal-access-token)

  - task: AabConvertToUniversalApk@1
    displayName: Convert AAB to APK
    inputs:
      aabFilePath: $(Build.SourcesDirectory)/release/app-production-release.aab
      keystoreFilePath: $(Agent.TempDirectory)/hayd_android.jks
      keystorePassword: $(KEYSTORE-PASSWORD)
      keystoreAlias: $(KEY-ALIAS)
      keystoreAliasPassword: $(KEY-PASSWORD)
      outputFolder: $(Build.SourcesDirectory)/release/

  - task: PublishPipelineArtifact@1
    name: publish_pipeline_apk
    displayName: Publish pipeline APK
    inputs:
      targetPath: $(Build.SourcesDirectory)/release/universal.apk
      artifact: release-apk
      publishLocation: pipeline
```

**Gradle 설정**
```gradle
Plugin.metaClass.isAndroidApp = {-> delegate.class.getCanonicalName() == "com.android.build.gradle.AppPlugin" }
Plugin.metaClass.isDynamicFeature = {-> delegate.class.getCanonicalName() == "com.android.build.gradle.DynamicFeaturePlugin" }
Plugin.metaClass.isAndroidLibrary = {-> delegate.class.getCanonicalName() == "com.android.build.gradle.LibraryPlugin" }

buildscript {
    apply from: 'versions.gradle'
    apply from: 'privates.gradle'
    repositories {
        google()
        mavenCentral()

        maven { url "https://www.jitpack.io" }
    }

    dependencies {
        classpath 'com.google.gms:google-services:4.3.10'
        classpath "com.android.tools.build:gradle:7.0.3"
        classpath deps.kotlin.plugin
        classpath deps.dagger.hilt_plugin
        classpath deps.navigation.safe_args_plugin
        classpath deps.firebase.remote_config
        classpath deps.firebase.perf_plugin  // Performance Monitoring plugin
        classpath deps.firebase.crashlytics_plugin
        classpath 'com.google.android.gms:oss-licenses-plugin:0.10.4'
    }

}

allprojects {
    repositories {
        google()
        mavenCentral()

        maven { url "https://jitpack.io" }
        maven { url 'https://maven.google.com/' }
        maven { url 'https://devrepo.kakao.com/nexus/content/groups/public/' }
    }

    plugins.whenPluginAdded {
        if (it.isAndroidApp() || it.isAndroidLibrary() || it.isDynamicFeature()) {
            android {
                flavorDimensions "flavors"
                productFlavors {
                    development {
                        dimension "flavors"
                    }
                    production {
                        dimension "flavors"
                    }
                }

                compileSdkVersion build_versions.compile_sdk
                buildToolsVersion build_versions.build_tools

                defaultConfig {
                    minSdkVersion build_versions.min_sdk
                    targetSdkVersion build_versions.target_sdk
                    versionCode build_versions.version_code
                    versionName build_versions.version_name

                    buildConfigField("String", "kakaoAppKey", "\"${privates.kakao_app_key}\"")
                    buildConfigField("String", "googleWebClientKey", "\"${privates.google_web_client_key}\"")
                    buildConfigField("String", "apiEndPoint", "\"${privates.api_end_point}\"")
                    buildConfigField("String", "clientSecretKey", "\"${privates.client_secret_key}\"")
                    buildConfigField("String", "iamportUserCode", "\"${privates.iamport_user_code}\"")

                    testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
                    multiDexEnabled true
                }

                buildFeatures {
                    dataBinding true
                    viewBinding true
//                    compose true
                }

                composeOptions {
//                    kotlinCompilerExtensionVersion compose_version
                }

                compileOptions {
                    sourceCompatibility JavaVersion.VERSION_11
                    targetCompatibility JavaVersion.VERSION_11
                }
            }

            dependencies {
//                implementation deps.compose.runtime
//                implementation deps.compose.ui
//                implementation deps.compose.foundation
//                implementation deps.compose.foundation_layout
//                implementation deps.compose.material
//                implementation deps.compose.livedata
//                implementation deps.compose.tooling
//                implementation deps.compose.theme_adapter
            }
        }
    }

    tasks.withType(org.jetbrains.kotlin.gradle.tasks.AbstractKotlinCompile).all {
        kotlinOptions.freeCompilerArgs += ["-Xuse-experimental=kotlinx.coroutines.ExperimentalCoroutinesApi"]
    }

    tasks.withType(org.jetbrains.kotlin.gradle.tasks.KotlinCompile).all {
        kotlinOptions {
            jvmTarget = JavaVersion.VERSION_11.toString()
//            useIR = true
        }
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```
**배포**