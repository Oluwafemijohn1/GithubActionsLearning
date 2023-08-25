# GithubActionsLearning
Deploying Android Apps Using GitHub Actions

This is a sample application to test continuous delivery.

- A workflow is a sequence of jobs that can run either in series or in parallel 
- A job usually is a container for more than one step, where each step is a self-contained function.

**To run the unit tests for the debug variant of your app, run the following command:**

```bash
./gradlew testDebugUnitTest
```

**To run the instrumented tests, you need to first have a device or emulator connected.Then, run the following command to execute the tests for the debug variant of your app:**

  ```bash
      ./gradlew connectedDebugAndroidTest
   ```
## 04. Generate a Signed Build
- Add the following code inside check_and_deploy.yml:

```yaml
name: Test and Deploy
jobs:
  unit_tests:
    runs-on: [ubuntu-latest]
    steps:
      - uses: actions/checkout@v3
      - name: set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'zulu'
          java-version: 11
      - name: Unit tests
        run: ./gradlew testDebugUnitTest
  android_tests:
    runs-on: [ macos-12 ]
    steps:
      - uses: actions/checkout@v3
      - name: set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'zulu'
          java-version: 11
      - name: Instrumented Tests
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 29
          script: ./gradlew connectedDebugAndroidTest
```
This code above does a few things:

Creates two parallel jobs named unit_tests and android_tests
**The unit_tests job runs on an Ubuntu runner, which checks out the code and runs the unit tests. ubuntu-latest refers to the latest available Ubuntu version on Github runners.**
**The android_tests job runs on a macOS 12 runner. If you want to use a specific version of an OS, you can mention the version instead of using latest. This job also checks out the code but runs the instrumentation tests instead. To do this, it uses the reactivecircus/android-emulator-runner action. The emulator can use hardware acceleration only on the macOS emulator. Therefore, this job needs to run on a macOS runner while others can run on Ubuntu runners.**

key-store:
quotes-signing-key
key-store-password: quotes-signing-key
key-alias: quotes
key-password: quotes-signing-key

In Terminal, run the following command and press enter to generate a keystore file:

```bash
 keytool -genkey -v -keystore quotes-signing-key.keystore -alias quotes -keypass quotes-signing-key -storepass quotes-signing-key -keyalg RSA -sigalg SHA256withRSA -keysize 2048 -validity 7500
```

- add the ALIAS, KEY_PASSWORD and KEY_STORE_PASSWORD in github environment variables
- To generate the base 64-encoded key, run the following command in the Terminal and copy the output string:

```bash
openssl base64 < path_to_signing_key | tr -d '\n' | tee some_signing_key.jks.base64.txt
```
e.g 
```bash
openssl base64 < quotes-signing-key.keystore | tr -d '\n' | tee quotes-signing-key.jks.base64.txt
```
- Go back to GitHub and paste the copied string as the value for SIGNING_KEY
- Add a new job named build to the workflow:

    ```yaml
    build:
    needs: [ unit_tests, android_tests ]
    runs-on: [ubuntu-latest]
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'zulu'
          java-version: 11
      - name: Generate Release APK
        run: ./gradlew assembleRelease
      - name: Sign APK
        uses: ilharp/sign-android-release@v1
        # ID used to access action output
        id: sign_app
        with:
          releaseDir: app/build/outputs/apk/release
          signingKey: ${{ secrets.SIGNING_KEY }}
          keykeyAlias: ${{ secrets.ALIAS }}
          keyStorePassword: ${{ secrets.KEY_STORE_PASSWORD }}
          keyPassword: ${{ secrets.KEY_PASSWORD }}
          buildToolsVersion: 33.0.0
      - uses: actions/upload-artifact@v3
        with:
          name: release.apk
          path: ${{steps.sign_app.outputs.signedFile}}
      - uses: actions/upload-artifact@v3
        with:
          name: mapping.txt
          path: app/build/outputs/mapping/release/mapping.txt
    ```

In this code, the build job performs multiple steps:

1. Checks out the code.
2. Generates a release APK using the assembleRelease Gradle task.
3. Signs the APK using the ilharp/sign-android-release action, which is a third-party action. This step uses the four secrets you added in the previous section. It also has an ID: sign_app.
4. Uploads the signed APK as an artifact to GitHub. This step uses the ID from the previous step to access its output, named signedFile.
5. Uploads the mapping file as an artifact. You’ll use this in a later step when you upload to the Play Store.

Commit the file to your project and push it to GitHub.

## 05 Trigger a Workflow
- You want to release a build in the following scenarios:

1. You push a version tag to the repository.
2. You create a pull request targeting the master branch.
   
Conditions to trigger a workflow are added under the on attribute.


```yaml
on:
  push:
    tags:
      - 'v*.*.*'
  pull_request:
    branches:
      - main
```

- To create a tag, run the following command in the Terminal:
```bash
git tag -a v0.1 -a -m "Release v0.1"
```
Push the tag to GitHub:
```bash
 git push --follow-tags   
```
Or 
```bash
git push origin master --follow-tags
```
Check the Actions tab on GitHub. You should see a new workflow run.

- To check that an empty commit can not trigger a workflow, run the following command in the Terminal:
```bash
git commit --allow-empty -m "Empty commit"
```
Push the commit to GitHub and check the Actions tab. You should not see a new workflow run.


