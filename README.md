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

## 06. Authenticate Firebase Project
<details>
<summary>06. Authenticate Firebase Project steps</summary>
<br>
Once the unit and instrumentation tests pass, you want to send the build to your QA team. A structured way of doing this is to use Firebase App Distribution. It lets you keep track of the uploaded builds and notify teams when new builds become available. You can even create groups and distribute a build only to specific groups.

First, you need to create a new project on Firebase. Open your web browser and go to the following URL: https://console.firebase.google.com/

Click Create a project.

Next, choose a project name. For this tutorial, name your project Quotes, then click Continue.

You don’t need Google Analytics for this project, so disable it. Click Create project.

Once the project is ready, click Continue.

Next, you need to add an Android app to the project. To do this, click the Android icon.

You’ll get a short form asking for the package name. The package name of the app is com.yourcompany.android.quotes.

Enter the package name and click Register app.

Because you’re only using App Distribution, skip the next two sections of the form by clicking Next.

Finally, click Continue to console to return to the home dashboard.

From the left navigation bar, select App Distribution in the Release & Monitor section.

Accept the terms and conditions and click Get started.

You’ll see the App Distribution dashboard, which contains three tabs: Releases, Invite links and Testers and Groups.

Go to the Testers and Groups tab.

In the Add testers input field, enter your email address. Firebase will then send you an invitation to become a tester for the app. Similarly, add the email addresses of the rest of your development and QA teams.

Next, you’ll create a group. A group is a collection of testers you want to send a specific build to. To create a group, click Add group and enter a name for your group — in this case, QA. Click Save

Now, you can choose the email addresses of all members of your QA team and add them here.

To give GitHub Actions access to your Firebase account, you’ll need two things:

1. App ID: An ID specific to your project.
2. Service Account: An authentication mechanism to access Firebase.
Click the gear icon at the top of the left navigation bar.

Select Project Settings from the menu.

Scroll down until you see the package name of the app.

Next to it, you’ll see a section named App ID with a long alphanumeric string. Copy this string and add it as a Secret on the GitHub repository named FIREBASE_APP_ID.

Switch back to the Firebase Project Settings page.

Select the Service accounts tab. Click Create service account and wait for it to be created.

Once done, click Generate new private key. A popup appears informing you to keep the private key safe.

Click Generate key and a JSON file will be downloaded to your computer. Open this file in any text editor of your choice.

Copy the contents of the file and store it as a Github token named SERVICE_ACCOUNT.

Switch back to the Firebase page.

Click on the link in the All service accounts section.

Here, you can view the service account that Firebase created for you. On the first account, click on the 3-dot menu and select Manage permissions.

And that’s it. You have complete the authentication needed to upload builds from GitHub Actions.
</details>

## 07. Deploy to Firebase App Distribution
- Add the following job to the workflow:

```yaml
 deploy-firebase:
    needs: [ build ]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: release.apk
      - name: unzip downloaded APK
        run: unzip app-release-unsigned-signed.apk
      - name: upload artifact to Firebase App Distribution
        uses: wzieba/Firebase-Distribution-Github-Action@v1
        with:
          appId: ${{secrets.FIREBASE_APP_ID}}
          serviceCredentialsFileContent: ${{secrets.SERVICE_ACCOUNT}}
          groups: QA
          file: app-release-unsigned-signed.apk
```
This code uses:

1. needs to specify that the job can only run if the build job has been completed successfully.
2. The actions/download-artifact action to download the artifact of the release APK. It uploads the downloaded artifact to Firebase App Distribution and makes it available to the QA group you created earlier. It also uses the two secrets named FIREBASE_APP_ID and SERVICE_ACCOUNT, which you added in the previous section.

Commit and push the changes to GitHub.
Next, create a new tag and push it.
    
```bash
git tag v0.2 -a -m "Release v0.2"
git push origin master --follow-tags
```

## 08. Authenticate with Google Play Console
<details>
<summary>Authenticate with Google Play Console steps</summary>
<br>
The action needed to deploy a build to the Play Store is similar to the one you used for app distribution but the setup is a bit more complex. It requires you to use a service account. Don’t worry if you haven’t heard of the term, you’ll create one shortly.

This video assumes that you have a Google Play account listing set up for your app.

Open your web browser and head to https://play.google.com/console. Go to Setup ▸ API access. Once there, click the Create a new Google Cloud project radio button and select Save Project in the bottom right corner of the page.

Once the Google Cloud project has been created, click View in Google Cloud Platform.

On the Google Cloud Console page, click the hamburger menu on the top-left and select APIs & Services ▸ Library.

Search for the Google Play Android Developer API. On the API details page, ensure the that API is enabled. In this case, the API is already enabled.

Next, open the hamburger menu again and select IAM & Admin. Select Service Accounts from the left menu.

Click Learn how to create service account.

On the Service accounts page, click CREATE SERVICE ACCOUNT .

Enter a name for the account and click CREATE AND CONTINUE.

Provide the account with the Editor and click CONTNUE.

Finally, click DONE to return to the Service Accounts page.

Click the three-dot menu next to the service account you just created.

Select Manage Keys to open the KEYS tab.

Click ADD KEY and select Create new key.

Select JSON and click CREATE.

This step will generate the key and then download it to your computer.

Go back to the Play Console tab on your browser and click Refresh on the pop-up.

The Service accounts section will refresh and the account you just created will appear.

You need to grant access to the service account to manage releases.

Click Manage Google Play permissions. This will open the Account permissions tab.

In the Releases section, select Release to production, exclude devices, and use Play App Signing and Release apps to testing tracks.

Finally, click Invite User.

On the confirmation popup, click Send invite.

On the next page, click Manage. This will open the the App permissions tab.

Click on Add app and choose your app. Click Apply.

Click Apply.

Finally, click Save changes. Click Yes.

Open the service account key file you downloaded and add its contents as a GitHub Secret named PLAY_SERVICE_ACCOUNT_JSON.

The service is now ready and you can automate Play Store uploads.
</details>

## 09. Deploy to Google Play Store
Create a new workflow file named play_store_workflow.yml in the .github/workflows directory.

- Add the following job to the workflow:

```yaml
name: Deploy to Play Store

on:
  push:
    branches:
      - master
jobs:
  build:
    runs-on: [ubuntu-latest]
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'zulu'
          java-version: 11
      - name: Generate Release AAB
        run: ./gradlew bundleRelease
      - name: Sign AAB
        uses: ilharp/sign-android-release@v1
        # ID used to access action output
        id: sign_app
        with:
          releaseDir: app/build/outputs/bundle/release
          signingKey: ${{ secrets.SIGNING_KEY }}
          keyAlias: ${{ secrets.ALIAS }}
          keyStorePassword: ${{ secrets.KEY_STORE_PASSWORD }}
          keyPassword: ${{ secrets.KEY_PASSWORD }}
          buildToolsVersion: 33.0.0
      - uses: actions/upload-artifact@v3
        with:
          name: release.aab
          path:  ${{steps.sign_app.outputs.signedFile}}
      - uses: actions/upload-artifact@v3
        with:
          name: mapping.txt
          path: app/build/outputs/mapping/release/mapping.txt

  deploy-play-store:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: release.aab
      - uses: actions/download-artifact@v3
        with:
          name: mapping.txt
      - name: Publish to Play Store internal test track
        uses: r0adkll/upload-google-play@v1.1.1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_SERVICE_ACCOUNT_JSON }}
          packageName: com.yourcompany.android.quotes
          releaseFiles: app-release-signed.aab
          track: internal
          changesNotSentForReview: true
          mappingFile: mapping.txt
```