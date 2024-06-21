# 📱 iOS Project with GitHub Actions

This repository contains an iOS project that demonstrates the use of GitHub Actions for continuous integration (CI).

## ✨ Features

- **iOS Application**: This repository includes a sample iOS application.
- **GitHub Actions**: Configuration for automating CI/CD workflows.

## 🚀 Getting Started

### Prerequisites

- Xcode 15.1 or later
- GitHub account

## 🔄 GitHub Actions Workflow

This project utilizes GitHub Actions to automate building and testing the iOS application. The workflow is defined in the `.github/workflows/build-ios-app.yml` file.

### 🛠️ Workflow Configuration

The workflow triggers on every push to the `main` branch, pull requests targeting the `main` branch, and can be manually triggered.

## 📚 Tutorial Followed

For detailed steps and explanations, please refer to Andrew Hoog's tutorial:

[How to Build an iOS App with GitHub Actions](https://www.andrewhoog.com/post/how-to-build-an-ios-app-with-github-actions-2023/)

## 📝 To-Do for Continuous Deployment (CD)

- **Deploy to TestFlight**: Future plans include adding steps to deploy the iOS application to TestFlight as part of the CI/CD pipeline.

#### `build-ios-app.yml`

```yaml
name: "Build iOS app"

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build_with_signing:
    runs-on: macos-latest
    steps:
      - name: Check Xcode version
        run: /usr/bin/xcodebuild -version

      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Install Apple certificate and provisioning profile
        env:
          BUILD_CERTIFICATE_BASE64: ${{ secrets.BUILD_CERTIFICATE_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          BUILD_PROVISION_PROFILE_BASE64: ${{ secrets.BUILD_PROVISION_PROFILE_BASE64 }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          # Create variables
          CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12
          PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db

          # Import certificate and provisioning profile from secrets
          echo -n "$BUILD_CERTIFICATE_BASE64" | base64 --decode -o $CERTIFICATE_PATH
          echo -n "$BUILD_PROVISION_PROFILE_BASE64" | base64 --decode -o $PP_PATH

          # Create temporary keychain
          security create-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH

          # Import certificate to keychain
          security import $CERTIFICATE_PATH -P "$P12_PASSWORD" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security list-keychain -d user -s $KEYCHAIN_PATH

          # Apply provisioning profile
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles

      - name: Build archive
        run: |
          xcodebuild -scheme "ExampleOfGitHubActions" \
          -archivePath $RUNNER_TEMP/ExampleOfGitHubActions.xcarchive \
          -sdk iphoneos \
          -configuration Debug \
          -destination generic/platform=iOS \
          -allowProvisioningUpdates \
          clean archive

      - name: Export IPA
        env:
          EXPORT_OPTIONS_PLIST: ${{ secrets.EXPORT_OPTIONS_PLIST }}
        run: |
          EXPORT_OPTS_PATH=$RUNNER_TEMP/ExportOptions.plist
          echo -n "$EXPORT_OPTIONS_PLIST" | base64 --decode -o $EXPORT_OPTS_PATH
          xcodebuild -exportArchive -archivePath $RUNNER_TEMP/ExampleOfGitHubActions.xcarchive -exportOptionsPlist $EXPORT_OPTS_PATH -exportPath $RUNNER_TEMP/build

      - name: Upload application
        uses: actions/upload-artifact@v3
        with:
          name: App
          path: ${{ runner.temp }}/build
          retention-days: 3
