This is a sample application to test continuous delivery.

- A workflow is a sequence of jobs that can run either in series or in parallel 
- A job usually is a container for more than one step, where each step is a self-contained function.

## To run the unit tests for the debug variant of your app, run the following command:

```bash
./gradlew testDebugUnitTest
```

## To run the instrumented tests, you need to first have a device or emulator connected.

## Then, run the following command to execute the tests for the debug variant of your app:

  ```bash
      ./gradlew connectedDebugAndroidTest
   ```


  
