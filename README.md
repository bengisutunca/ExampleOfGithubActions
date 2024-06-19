# iOS Project with GitHub Actions

This repository contains an iOS project that demonstrates the use of GitHub Actions for continuous integration (CI).

## Features

- **iOS Application**: A sample iOS application.
- **GitHub Actions**: Workflow configuration for CI/CD.

## Getting Started

### Prerequisites

- Xcode 15.1 or later
- A GitHub account

## GitHub Actions Workflow

This project uses GitHub Actions to automatically build and test the iOS application. The workflow is defined in the `.github/workflows/build-ios-app.yml` file.

### Workflow Configuration

The workflow is triggered on every push to the `main` branch, on pull requests targeting the `main` branch, and can also be manually triggered.

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
      - name: check Xcode version
        run: /usr/bin/xcodebuild -version

      - name: checkout repository
        uses: actions/checkout@v3

      - name: Install the Apple certificate and provisioning profile
        env:
          BUILD_CERTIFICATE_BASE64: ${{ secrets.BUILD_CERTIFICATE_BASE64 }}
          P12_PASSWORD: ${{ secrets.P12_PASSWORD }}
          BUILD_PROVISION_PROFILE_BASE64: ${{ secrets.BUILD_PROVISION_PROFILE_BASE64 }}
          KEYCHAIN_PASSWORD: ${{ secrets.KEYCHAIN_PASSWORD }}
        run: |
          # create variables
          CERTIFICATE_PATH=$RUNNER_TEMP/build_certificate.p12
          PP_PATH=$RUNNER_TEMP/build_pp.mobileprovision
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db

          # import certificate and provisioning profile from secrets
          echo -n "$BUILD_CERTIFICATE_BASE64" | base64 --decode -o $CERTIFICATE_PATH
          echo -n "$BUILD_PROVISION_PROFILE_BASE64" | base64 --decode -o $PP_PATH

          # create temporary keychain
          security create-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "$KEYCHAIN_PASSWORD" $KEYCHAIN_PATH

          # import certificate to keychain
          security import $CERTIFICATE_PATH -P "$P12_PASSWORD" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security list-keychain -d user -s $KEYCHAIN_PATH

          # apply provisioning profile
          mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
          cp $PP_PATH ~/Library/MobileDevice/Provisioning\ Profiles

      - name: build archive
        run: |
          xcodebuild -scheme "ExampleOfGithubActions" \
          -archivePath $RUNNER_TEMP/ExampleOfGithubActions.xcarchive \
          -sdk iphoneos \
          -configuration Debug \
          -destination generic/platform=iOS \
          -allowProvisioningUpdates \
          clean archive

      - name: export ipa
        env:
          EXPORT_OPTIONS_PLIST: ${{ secrets.EXPORT_OPTIONS_PLIST }}
        run: |
          EXPORT_OPTS_PATH=$RUNNER_TEMP/ExportOptions.plist
          echo -n "$EXPORT_OPTIONS_PLIST" | base64 --decode -o $EXPORT_OPTS_PATH
          xcodebuild -exportArchive -archivePath $RUNNER_TEMP/ExampleOfGithubActions.xcarchive -exportOptionsPlist $EXPORT_OPTS_PATH -exportPath $RUNNER_TEMP/build

      - name: Upload application
        uses: actions/upload-artifact@v3
        with:
          name: app
          path:  ${{ runner.temp }}/build
          retention-days: 3
