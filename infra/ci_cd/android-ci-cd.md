# 안드로이드 어플리케이션 CI/CD 파이프라인 구축
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
      $ANDROID_HOME/tools/bin/sdkmanager --uninstall 'platforms;android-31' 'platform-tools'
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