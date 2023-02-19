# Setup Gitlab CI For Android

[Source](https://www.howtodoandroid.com/setup-gitlab-ci-android/ "Permalink to Setup Gitlab CI For Android [Step By Step] - Howtodoandroid")

being an android developer, we don’t get a chance to work on Continuous integration most of the time. So, I created this tutorial to learn the GitLab CI with simple steps. GitLab is a complete DevOps platform, delivered as a single application.

Continuous integration is a way to avoid build failure and test failure before code changes are merged in.

## Step to setup Gitlab CI

1. Create Gitlab project and setup repository for the android project.
2. Setup Continous Integration using the gitlab default CI android template.
3. Automatically build android application and generate APK when pushing the code to the particular branch.
4. Run the Link check and Test Cases
5. Deploy the android APK to Firebase App Distrubution.

## Create GitLab project and setup repository

You create a project in Gitlab, first, we need to sign up an account with GitLab. visit https://gitlab.com/ to sign up for the account.

Once sign-in, the next step is to create a new project and repository.

![create new project in gitlab](images/create-new-project-1024x650.png)create a new project in GitLab

Enter the Project Name and the description to create a new project. Also, I have initialized the readme file for the project details.

### Setup GitLab project in Android Studio

Before setup, the GitLab project in android studio, you first, need to install the git in your local machine. you can download git in [Git – Downloads (git-scm.com)](https://git-scm.com/downloads) link. Now, you can create a new project in android studio and set up the GitLab project using the git commands.

![Create a new android studio project to setup gitlab](images/create-new-android-project-1024x736.png)create a new android studio project

Once the project got created, open the terminal in android studio and enter the git commands to add the project into the GitLab repository.

    git init
    git remote add origin git@gitlab.com:GITLABUSERNAME/YOURGITPROJECTNAME.git
    git add .
    git commit -m "Initial commit"
    git push -u origin master
    Code language: JavaScript (javascript)

**Note:**

![pushing code to gitlab repository](images/git-push-1024x124.png) git push command

 When you are entering **git push** it will ask for the username and password for your GitLab account. 

After the git push, you can able to see the initial android project code pushed to the git master branch of the GitLab project.

## Setup Continuous Integration on GitLab

Already we have completed the GitLab repository setup, Let’s create a GitLab pipeline to automate test and deployment. To get started we need to create** .gitlab-ci.yml** in the root direct of the project.

![Create gitlab-ci.yml file in root directory of the project](images/create-new-gitlab-ci-yml-file.png)

 Once created the .gitlab-ci.yml file, paste the below code in that file. I will explain the code in detail below,

```yaml
# To contribute improvements to CI/CD templates, please follow the Development guide at:
# https://docs.gitlab.com/ee/development/cicd/templates.html
# This specific template is located at:
# https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Android.gitlab-ci.yml

# Read more about this script on this blog post https://about.gitlab.com/2018/10/24/setting-up-gitlab-ci-for-android-projects/, by Jason Lenny
# If you are interested in using Android with FastLane for publishing take a look at the Android-Fastlane template.

image: openjdk:11-jdk

variables:

  # ANDROID_COMPILE_SDK is the version of Android you're compiling with.
  # It should match compileSdkVersion.
  ANDROID_COMPILE_SDK: "30"

  # ANDROID_BUILD_TOOLS is the version of the Android build tools you are using.
  # It should match buildToolsVersion.
  ANDROID_BUILD_TOOLS: "30.0.3"

  # It's what version of the command line tools we're going to download from the official site.
  # Official Site-> https://developer.android.com/studio/index.html
  # There, look down below at the cli tools only, sdk tools package is of format:
  #        commandlinetools-os_type-ANDROID_SDK_TOOLS_latest.zip
  # when the script was last modified for latest compileSdkVersion, it was which is written down below
  ANDROID_SDK_TOOLS: "7583922"

# Packages installation before running script
before_script:
  - apt-get --quiet update --yes
  - apt-get --quiet install --yes wget tar unzip lib32stdc++6 lib32z1

  # Setup path as ANDROID_SDK_ROOT for moving/exporting the downloaded sdk into it
  - export ANDROID_SDK_ROOT="${PWD}/android-home"
  # Create a new directory at specified location
  - install -d $ANDROID_SDK_ROOT
  # Here we are installing androidSDK tools from official source,
  # (the key thing here is the url from where you are downloading these sdk tool for command line, so please do note this url pattern there and here as well)
  # after that unzipping those tools and
  # then running a series of SDK manager commands to install necessary android SDK packages that'll allow the app to build
  - wget --output-document=$ANDROID_SDK_ROOT/cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-${ANDROID_SDK_TOOLS}_latest.zip
  # move to the archive at ANDROID_SDK_ROOT
  - pushd $ANDROID_SDK_ROOT
  - unzip -d cmdline-tools cmdline-tools.zip
  - pushd cmdline-tools
  # since commandline tools version 7583922 the root folder is named "cmdline-tools" so we rename it if necessary
  - mv cmdline-tools tools || true
  - popd
  - popd
  - export PATH=$PATH:${ANDROID_SDK_ROOT}/cmdline-tools/tools/bin/

  # Nothing fancy here, just checking sdkManager version
  - sdkmanager --version

  # use yes to accept all licenses
  - yes | sdkmanager --licenses || true
  - sdkmanager "platforms;android-${ANDROID_COMPILE_SDK}"
  - sdkmanager "platform-tools"
  - sdkmanager "build-tools;${ANDROID_BUILD_TOOLS}"

  # Not necessary, but just for surity
  - chmod +x ./gradlew

# Basic android and gradle stuff
# Check linting
lintDebug:
  interruptible: true
  stage: build
  script:
    - ./gradlew -Pci --console=plain :app:lintDebug -PbuildDir=lint

# Make Project
assembleDebug:
  interruptible: true
  stage: build
  script:
    - ./gradlew assembleDebug
  artifacts:
    paths:
      - app/build/outputs/

# Run all tests, if any fails, interrupt the pipeline(fail it)
debugTests:
  interruptible: true
  stage: test
  script:
    - ./gradlew -Pci --console=plain :app:testDebug
```

### **Understanding .gitlab-ci.yml**

first, we will see the important key terms in yml file.

**Image: **This tag is used to specify a Docker image to be used for execution. Gitlab runners will use this Docker image to run the pipeline.

**Variables:** This variable tag is used to set the variables used in the script execution.

**Before Script:** This tag is used to execute the scripts before the main script execution. Mainly used to set up the environment.

**Script:** We need to use a script tag to execute the main functions like building the APK or running the test cases.

**Stages:** This tag is used to specify the globally defined level of the pipeline run. This stage will execute on the given order. Each stage can contain different jobs. Every stage has the following tags.

* **Stage:** used to define the which stage this running job belongs to.
* **only: **This only tag is used to define on which condition this particular stage should execute. we can use this tag to execute the stage only on master branch changes or etc.
* **artifacts:** Jobs can output an archive of files and directories. This output is known as a job artifact. You can download job artifacts by using the GitLab UI or the API.

Now we know most of the tags used in the pipeline. below I have explained the pipeline yml file in detail.

**Defining the Docker Image**

```dockerfile
image: openjdk:11-jdk
Code language: CSS (css)
```

Docker containers are very fast to create and destroy instances, also a good choice for setting up a temporary environment for building and testing.

In this project, we are using [**Openjdk:**](https://hub.docker.com/_/openjdk)**[1](https://hub.docker.com/_/openjdk)**[**1-jdk**](https://hub.docker.com/_/openjdk) docker image.

**Defining the variables**

lets set up the variables needed to build the android applications.

```properties
ANDROID_COMPILE_SDK: "30"
ANDROID_SDK_TOOLS: "7583922"
ANDROID_BUILD_TOOLS: "30.0.3"
Code language: JavaScript (javascript)
```

**Setup android environment**

on the **before\_script**, we need to set up the android environment by setting up ANDROID\_SDK\_ROOT and installing the android SDK manager.

```shell
- apt-get --quiet update --yes
  - apt-get --quiet install --yes wget tar unzip lib32stdc++6 lib32z1

  # Setup path as ANDROID_SDK_ROOT for moving/exporting the downloaded sdk into it
  - export ANDROID_SDK_ROOT="${PWD}/android-home"
  # Create a new directory at specified location
  - install -d $ANDROID_SDK_ROOT
  # Here we are installing androidSDK tools from official source,
  # (the key thing here is the url from where you are downloading these sdk tool for command line, so please do note this url pattern there and here as well)
  # after that unzipping those tools and
  # then running a series of SDK manager commands to install necessary android SDK packages that'll allow the app to build
  - wget --output-document=$ANDROID_SDK_ROOT/cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-${ANDROID_SDK_TOOLS}_latest.zip
  # move to the archive at ANDROID_SDK_ROOT
  - pushd $ANDROID_SDK_ROOT
  - unzip -d cmdline-tools cmdline-tools.zip
  - pushd cmdline-tools
  # since commandline tools version 7583922 the root folder is named "cmdline-tools" so we rename it if necessary
  - mv cmdline-tools tools || true
  - popd
  - popd
  - export PATH=$PATH:${ANDROID_SDK_ROOT}/cmdline-tools/tools/bin/

  # Nothing fancy here, just checking sdkManager version
  - sdkmanager --version

  # use yes to accept all licenses
  - yes | sdkmanager --licenses || true
  - sdkmanager "platforms;android-${ANDROID_COMPILE_SDK}"
  - sdkmanager "platform-tools"
  - sdkmanager "build-tools;${ANDROID_BUILD_TOOLS}"

  # Not necessary, but just for surity
  - chmod +x ./gradlew
```

**Building the app**

```yaml
assembleDebug:
  interruptible: true
  stage: build
  script:
    - ./gradlew assembleDebug
  artifacts:
    paths:
      - app/build/outputs/
```

To start the build job, we need to define the job name and the **stage**. in the **script **tag we need to mention the Gradle script to build the android app. Also, In the **artifacts **tag, we need to define the output apk file location. So, that we can download the output of the build.

**Running tests**

```yaml
debugTests:
  interruptible: true
  stage: test
  script:
    - ./gradlew -Pci --console=plain :app:testDebug
Code language: JavaScript (javascript)
```

same as building the app, first need to define the job name and the stage. Then, the script tag needs to provide the Gradle script to execute the test cases.

## How to run the gitlab CI setup 

After you’ve added your new **.gitlab-ci.yml** file to the root of your directory, just push your changes to the repository. Then the build will be triggered automatically, you can see the process in the Pipelines tab of your project. Also, you have an option to run the pipeline manually in the pipeline tab.

![](images/run-pipeline-1024x382.png)

After your build is done, you can retrieve your build artifacts.

![](images/download-artifacts-1024x192.png)

### Gitlab CI/CD Editor

GitLab provides the Editor for the **.gitlab-ci.yml **file. You can directly edit and commit the yml file in the web app itself. Also, GitLab providing the default template for the different environments.

Check the gitlab ci templetes **[here](https://gitlab.com/gitlab-org/gitlab-foss/tree/master/lib/gitlab/ci/templates)**.

![](images/gitlab-editor-1024x383.png)

**Conclusion**

I hope now you learned about how to set up GitLab CI and pipeline configurations. In my next post i will explain about the GitLab CD.

Thanks for reading. Please let me know your feedback in the comments.
