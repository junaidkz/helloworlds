## **Jenkins Freestyle Jobs: .NET MSBuild, Multi-Environment & Artifactory Push Control (Windows)**
This guide details how to create and configure Jenkins Freestyle jobs to build your .NET Framework 4.6.1 application using MSBuild on a Windows agent and conditionally push packages to JFrog Artifactory. The jobs will be parameterized for UI control.

### **Prerequisites**
1. **Jenkins Instance:** A running Jenkins master.
2. **Windows Jenkins Agent(s):** Configured Windows agents with:
 * .NET Framework 4.6.1 (or the version targeted by your project and compatible with MSBuild).
 * MSBuild: Typically available with Visual Studio or .NET Framework SDKs. Ensure MSBuild.exe is in the system's PATH or you know its full path.
 * NuGet CLI: nuget.exe in the PATH, for restoring packages.
 * JFrog CLI: Installed and in the system's PATH.
3. **JFrog Artifactory:** An accessible Artifactory instance.
4. **Jenkins Plugins:**
 * **Git Plugin:** (Usually installed by default) For SCM.
 * **Credentials Plugin:** (Usually installed by default) For managing secrets.
 * **Credentials Binding Plugin:** To bind credentials to environment variables.
 * *(Optional but Recommended)* **Build Name and Number Setter Plugin:** For more flexible versioning.
 * *(Optional)* **MSBuild Plugin:** Can simplify MSBuild execution but "Execute Windows batch command" offers more control. This guide will use batch commands.
5. **Artifactory Credentials in Jenkins:** Your Artifactory username and password (or API Key) stored in Jenkins.

### **Step 1: Create a New Freestyle Project**
1. From the Jenkins dashboard, click on **"New Item"**.
2. Enter an item name (e.g., my-dotnet-app-build-deploy).
3. Select **"Freestyle project"**.
4. Click **"OK"**.

### **Step 2: Configure General Settings**
1. **Description:**
Builds [Your .NET Application Name] using MSBuild for .NET Framework 4.6.1.
Parameters allow selecting the target environment (DEV, QA, PROD)
and choosing whether to push the built package to JFrog Artifactory.
Requires MSBuild, NuGet CLI, and JFrog CLI on the Windows agent.

### **Step 3: Parameterize the Job**
1. Check: **"This project is parameterized"**.
2. Click **"Add Parameter"** for each:
 * **Choice Parameter (for Environment Selection)**
   * **Name:** TARGET_ENVIRONMENT
   * **Choices:** DEV\nQA\nPROD
   * **Description:** Select the target environment.
 * **String Parameter (for Package Name)**
   * **Name:** PACKAGE_NAME
   * **Default Value:** my-dotnet-app
   * **Description:** Base name for the package/application.
 * **String Parameter (for Package Version)**
   * **Name:** PACKAGE_VERSION
   * **Default Value:** 1.0.0-${BUILD_NUMBER}
   * **Description:** Package version.
 * **String Parameter (for Solution File Path)**
   * **Name:** SOLUTION_FILE_PATH
   * **Default Value:** YourSolutionName.sln (e.g., src/MyApplication.sln)
   * **Description:** Relative path within the workspace to your solution (.sln) file.
 * **String Parameter (for Build Configuration)**
   * **Name:** BUILD_CONFIGURATION
   * **Default Value:** Release
   * **Description:** Build configuration (e.g., Debug, Release).
 * **String Parameter (for Artifact Path within Workspace)**
   * **Name:** BUILD_ARTIFACT_PATH
   * **Default Value:** YourProjectName\bin\${BUILD_CONFIGURATION}\YourProjectOutput.dll (Adjust this. Examples: WebApp\bin\Release\WebApp.zip, MyService\bin\Release\MyService.exe, Project\Output\Package.nupkg)
   * **Description:** Relative path within the workspace to the artifact to be pushed (e.g., DLL, EXE, ZIP, NuGet package). Use ${BUILD_CONFIGURATION} if path depends on it.
 * **Boolean Parameter (to Control Artifactory Push)**
   * **Name:** PUSH_TO_ARTIFACTORY
   * **Default Value:** (Checked or unchecked)
   * **Description:** Check to push the package to JFrog Artifactory.
 * **String Parameter (for Artifactory URL or Server ID)**
   * **Name:** ARTIFACTORY_SERVER_ID_OR_URL
   * **Default Value:** https://your-artifactory.example.com/artifactory
   * **Description:** Artifactory URL or pre-configured JFrog CLI Server ID.
 * **Credentials Parameter (for Artifactory Authentication)**
   * **Type:** Credentials
   * **Name:** ARTIFACTORY_CREDENTIALS_ID
   * **Credentials type:** Username with password
   * **Default Value:** (Select your Artifactory credential ID)
   * **Description:** Jenkins credentials for Artifactory.
 * **String Parameter (for DEV Artifactory Repository)**
   * **Name:** ARTIFACTORY_REPO_KEY_DEV
   * **Default Value:** dotnet-dev-local
   * **Description:** Artifactory repository key for DEV.
 * **String Parameter (for QA Artifactory Repository)**
   * **Name:** ARTIFACTORY_REPO_KEY_QA
   * **Default Value:** dotnet-qa-local
   * **Description:** Artifactory repository key for QA.
 * **String Parameter (for PROD Artifactory Repository)**
   * **Name:** ARTIFACTORY_REPO_KEY_PROD
   * **Default Value:** dotnet-prod-local
   * **Description:** Artifactory repository key for PROD.

### **Step 4: Configure Source Code Management**
1. Select **"Git"**.
2. **Repository URL:** Your Git repo URL.
3. **Credentials:** If private.
4. **Branch Specifier:** e.g., */main or ${GIT_BRANCH}.

### **Step 5: Configure Build Triggers (Optional)**
As needed.

### **Step 6: Configure Build Environment**
1. Check: **"Use secret text(s) or file(s)"**.
2. **Add** -> **"Username and password (separated)"**.
 * **Variable:** ARTIFACTORY_CREDENTIALS_ID
 * **Username Variable:** ARTIFACTORY_USER
 * **Password Variable:** ARTIFACTORY_PASS

### **Step 7: Add Build Steps**
1. **Restore NuGet Packages:**
 * Click **"Add build step"** -> **"Execute Windows batch command"**.
 * **Command:**
echo --- Restoring NuGet Packages ---
REM Ensure nuget.exe is in your PATH or provide the full path
nuget restore "%SOLUTION_FILE_PATH%"
echo --- NuGet Restore Complete ---
2. **Build .NET Application with MSBuild:**
 * Click **"Add build step"** -> **"Execute Windows batch command"**.
 * **Command:**
echo --- Building .NET Application ---
REM Ensure MSBuild.exe is in your PATH or use its full path.
REM Common paths for MSBuild 14.0 (VS 2015, for .NET 4.6.1):
REM "C:\Program Files (x86)\MSBuild\14.0\Bin\MSBuild.exe"
REM Adjust MSBuild path if necessary.
SET MSBUILD_PATH="C:\Program Files (x86)\MSBuild\14.0\Bin\MSBuild.exe"
IF NOT EXIST %MSBUILD_PATH% (
  echo WARNING: MSBuild not found at %MSBUILD_PATH%. Assuming it's in PATH.
  SET MSBUILD_PATH=MSBuild.exe
)

%MSBUILD_PATH% "%SOLUTION_FILE_PATH%" /p:Configuration="%BUILD_CONFIGURATION%" /p:DeployOnBuild=true /p:PublishProfile="%BUILD_CONFIGURATION%" /p:VisualStudioVersion=14.0 /t:Rebuild
REM Add other MSBuild properties as needed, e.g., /p:OutDir=YourCustomOutput\
REM If you are creating a deployment package (e.g., Web Deploy zip), ensure your .csproj files are configured for it.
REM The BUILD_ARTIFACT_PATH parameter should point to the output of this step.
echo --- Build Complete ---
   * **Note on /p:VisualStudioVersion=14.0**: This is typical for .NET 4.6.1 projects built with VS2015 tools. Adjust if you use a different toolset version that's compatible.
   * **Note on DeployOnBuild=true /p:PublishProfile="%BUILD_CONFIGURATION%"**: These are common for web applications to generate deployment packages. For other types of projects (libraries, console apps), you might not need them, or you might have custom MSBuild targets.
3. **Conditionally Push to Artifactory (using Execute Windows batch command and JFrog CLI):**
 * Click **"Add build step"** -> **"Execute Windows batch command"**.
 * **Command:**
@echo off
echo --- Job Parameters ---
echo Target Environment: %TARGET_ENVIRONMENT%
echo Package Name: %PACKAGE_NAME%
echo Package Version: %PACKAGE_VERSION%
echo Build Artifact Path in Workspace: %BUILD_ARTIFACT_PATH%
echo Push to Artifactory: %PUSH_TO_ARTIFACTORY%
echo Artifactory Server/URL: %ARTIFACTORY_SERVER_ID_OR_URL%
REM ARTIFACTORY_USER and ARTIFACTORY_PASS are injected

REM Verify the artifact exists
IF NOT EXIST "%WORKSPACE%\%BUILD_ARTIFACT_PATH%" (
  echo Error: Build artifact not found at %WORKSPACE%\%BUILD_ARTIFACT_PATH%
  echo Please check the BUILD_ARTIFACT_PATH parameter and your build process.
  exit /b 1
)

IF /I "%PUSH_TO_ARTIFACTORY%" NEQ "true" (
  echo --- Skipping Artifactory Push (PUSH_TO_ARTIFACTORY parameter is not true) ---
  exit /b 0
)

echo --- Preparing to Push to Artifactory ---

SET TARGET_ACTUAL_REPO_KEY=
IF /I "%TARGET_ENVIRONMENT%" == "DEV" SET TARGET_ACTUAL_REPO_KEY=%ARTIFACTORY_REPO_KEY_DEV%
IF /I "%TARGET_ENVIRONMENT%" == "QA" SET TARGET_ACTUAL_REPO_KEY=%ARTIFACTORY_REPO_KEY_QA%
IF /I "%TARGET_ENVIRONMENT%" == "PROD" SET TARGET_ACTUAL_REPO_KEY=%ARTIFACTORY_REPO_KEY_PROD%

IF "%TARGET_ACTUAL_REPO_KEY%" == "" (
  echo Error: Unknown TARGET_ENVIRONMENT: '%TARGET_ENVIRONMENT%'. Must be DEV, QA, or PROD, or repo key not set.
  echo Please check ARTIFACTORY_REPO_KEY_DEV/QA/PROD parameters.
  exit /b 1
)
echo Selected Artifactory Repository Key: %TARGET_ACTUAL_REPO_KEY%

REM Ensure JFrog CLI is installed
jfrog -v > nul 2>&1
IF ERRORLEVEL 1 (
    echo Error: JFrog CLI (jfrog) not found or not in PATH. Please install it on the Windows agent.
    exit /b 1
)

REM Ensure credentials are set
IF "%ARTIFACTORY_USER%" == "" (
    echo Error: ARTIFACTORY_USER environment variable is not set.
    echo Please configure 'Use secret text(s) or file(s)' in Build Environment.
    exit /b 1
)
IF "%ARTIFACTORY_PASS%" == "" (
    echo Error: ARTIFACTORY_PASS environment variable is not set.
    echo Please configure 'Use secret text(s) or file(s)' in Build Environment.
    exit /b 1
)

set JFROG_CLI_OFFER_CONFIG=false
set JFROG_CLI_USER=%ARTIFACTORY_USER%
set JFROG_CLI_PASSWORD=%ARTIFACTORY_PASS%
REM If using an Access Token in the password field of Jenkins credential:
REM set JFROG_CLI_ACCESS_TOKEN=%ARTIFACTORY_PASS%
REM set JFROG_CLI_PASSWORD=

REM Extract filename from BUILD_ARTIFACT_PATH
FOR %%F IN ("%BUILD_ARTIFACT_PATH%") DO SET ARTIFACT_FILENAME=%%~nxF

REM Construct the target path in Artifactory.
SET JFROG_TARGET_PATH_IN_REPO=%TARGET_ACTUAL_REPO_KEY%/%PACKAGE_NAME%/%PACKAGE_VERSION%/

echo Attempting to upload %WORKSPACE%\%BUILD_ARTIFACT_PATH% to Artifactory path: %JFROG_TARGET_PATH_IN_REPO%
echo Using Artifactory URL: %ARTIFACTORY_SERVER_ID_OR_URL%

jfrog rt u ^
  "%WORKSPACE%\%BUILD_ARTIFACT_PATH%" ^
  "%JFROG_TARGET_PATH_IN_REPO%" ^
  --url="%ARTIFACTORY_SERVER_ID_OR_URL%" ^
  --build-name="%PACKAGE_NAME%" ^
  --build-number="%BUILD_NUMBER%" ^
  --fail-no-op=true
IF ERRORLEVEL 1 (
    echo Error: JFrog CLI upload failed.
    exit /b 1
)

REM Optional: Publish build info
REM jfrog rt build-publish "%PACKAGE_NAME%" "%BUILD_NUMBER%" --url="%ARTIFACTORY_SERVER_ID_OR_URL%"

echo --- Artifactory Push Successful for %TARGET_ENVIRONMENT% ---

### **Step 8: Post-build Actions (Optional)**
* **"E-mail Notification"**
* **"Archive the artifacts"**

### **Step 9: Save and Run the Job**
1. Click **"Save"**.
2. Click **"Build with Parameters"**.
3. Fill in parameters (especially SOLUTION_FILE_PATH, BUILD_CONFIGURATION, BUILD_ARTIFACT_PATH for your .NET project).
4. Click **"Build"**.

### **Step 10: Adapting for Different Environments / Multiple Jobs**
The strategies remain the same as in the original guide (single generic job vs. cloned jobs per environment). For .NET, ensure your solution/project files handle different configurations (Debug, Release, custom environment configs) correctly if needed.

### **Important Considerations for .NET/Windows**
* **MSBuild Path:** The script includes a common path for MSBuild 14.0. If your agent has a different setup, you might need to adjust SET MSBUILD_PATH= or ensure MSBuild is reliably in the system PATH.
* **NuGet Path:** Similarly, ensure nuget.exe is in the PATH.
* **Output Paths:** .NET project output paths can be complex (bin\Configuration\framework\output). Double-check your BUILD_ARTIFACT_PATH parameter to accurately point to the final deployable artifact (e.g., a ZIP from a web deploy package, a specific DLL, an EXE, or a NuGet package if you're building one).
* **Web Deploy Packages:** If you're building a web application and want to deploy a Web Deploy package (ZIP file), ensure your .csproj is configured for this (e.g., <DeployOnBuild>true</DeployOnBuild>, <WebPublishMethod>Package</WebPublishMethod>). The output ZIP path would then be your BUILD_ARTIFACT_PATH.
* **Batch Scripting:** The Artifactory push script is now a Windows batch script. Pay attention to syntax differences from bash (e.g., variable setting/expansion, IF conditions).
* **Workspace Paths:** Use %WORKSPACE% in batch scripts.
This adapted guide should help you set up your Jenkins Freestyle job for .NET Framework 4.6.1 applications on Windows agents.
