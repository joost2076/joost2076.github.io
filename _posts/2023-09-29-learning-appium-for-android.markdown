---
layout: post
title:  "Beginners Guide: Appium, js & mobile emulators"
date:   2023-10-31 12:19:54 +0200
categories: Testing
---

<style>

  @media(max-width:1000px){
    #toc{
      display:none;
    }
  }

  @media(min-width:1000px){
    .wrapper{
      max-width:calc(90% - 268px);
      margin-left: 270px;
    }

  }

  video{
    display:block; 
    max-width:360px;
    height:auto;
    margin-bottom:48px;
  }


  #toc{
      position:fixed;
      top:72px;
      left:0px;
      max-width:268px;
      font-size:14px;
      max-height:100%;
      overflow: scroll;
  }

</style>

<div  id="toc" markdown="1">
  * seed for toc
{:toc}
</div>

work in progress 

### End-to-end testing apps on emulated android devices with appium and javascript

Looking to get started with end to end testing android apps with appium and javascript?  Read on to get to grips with the basics of appium and everything you will need around it. This article offers an explanation of tooling and terms you will encountered while learning to test android apps with appium. 

Appium can be used in many languages, from java to javascript. My language of choice is *javascript*, which means we will also add *webdriverio* (an appium client in javascript). Be warned, this is my journey in getting familiar with appium and mobile devices, it is my first attempt at using appium, and also my first experience with the android ecosystem. 


## What is an e2e test ?

First up, lets define *e2e tests*: an e2e test is  a scenario / test that interacts with an app the way a user would. Goal is to test the final integration of all components, packages and services: Are the most important flows of the app working correctly? 

Zooming in on the important components a bit more:
- *with an app* means we will test a complete compiled app (not a component of it), running against a live 'production like' background. 
- *interacts like a user* means the interactions with which we will test scenarios are similar to the way a user would do them: with button touches, swiped initiated on the device itself (somewhat). 

### The good and the bad of e2e
E2E tests are *great* because they offer the best integration testing of all components at once and provide confidence in the app and the services it runs against. Writing and understanding e2e can also be a lot simpler then integration or unit tests, because there is less need to mock or setup, since the the complete app is used in a complete test env.   

*Downside* of e2e test are that they are generally more flaky and slow to run compared to other tests, fiddly to write (finding the right element at the right moment to click is sometimes harder then it seems) and more importantly, can practically only run pretty late in the dev life cycle. It is practically impossible to run e2e for every small change in some ui component or api service, because you would need to compile the whole app and make sure that everything is compatible at every moment (api versions, features etc). So generally, one would run e2e tests in a stage close to deployment. 

## What is appium?

The appium docs are worth a read, but summarised it is a library 
>"designed to facilitate UI automation of many app platforms" [link](https://appium.io/docs/en/2.0/).

App platforms supported by appium  are among others roku (tvs), android, ios and most desktop browsers. Appium facilitates the automation, meaning it *does **not** provide the api to write tests in nor does it do the actual automation*: It facilitates the automation by *providing an api* that sits in between the test you write and the actual automation of actions in those tests. That api is the [webdriver protocol](https://www.w3.org/TR/webdriver2/). 

To work with appium, you will also need a *client and a driver*: the  client library implements an api describing user actions that you can use in a test, in a programming language you like into a request conforming to the webdriver protocol. The 'driver' library handles the actual automation, that is transforms the requests into an action performed on the device. 

Zooming in a bit more on each component:
- **The Client**  implements a test api for you to use when writing tests, For instance functions to *findElement, touchAction, startRecordingScreen* are implemented (For the js client see [here](https://webdriver.io/docs/api)). That api is then transformed by the client into a http request that conforms to the webdriver protocol (for some example, check [here](https://www.w3.org/TR/webdriver2/#endpoints)) and is send to the appium server by the client. 

  There are multiple clients, and the difference between them is the language they are written in.  (java, python, c#, ruby and javascript), thought the api they implement will sometimes use different function names or might not have missing functionality (some webdriver api's are without any client implementation) etc. The client is what you will use most extensively and explicitly.  

- **The appium server** listens for http requests (in our case coming from the client), passes it to the driver, and responds after the driver has finised with it. It is actually not much more then a webdriver api compliant server, there is little other logic. It is there so there is no need to define the server parts again and again by each driver author, since it can simply be re-used. Other then starting it up, installing a driver and maybe tweak some settings, you will only use this implicitly.  

- **The driver** runs on the appium server and handles the actual automation and transforms the result into a the suitable webdriver response to send back to the client. The driver is specific to the object you are trying to automate. There are drivers for android, ios, roku, chrome etc. Other then installing it (and maybe again tweak some settings), you will need not worry much about it. It does its job in the background. The uiautomator2 driver for android you find [here](https://github.com/appium/appium-uiautomator2-driver).

So from this architecture is becomes clear that apart from appium  we will also need a client (*webdriverio*, because i know javascript) and a driver (*uiautomator2* because of android).


## Getting a grip on the android emulated device

Now that there is some basic understanding about appium and e2e tests, we can zoom in a bit more on understanding and running the device we want to test the app on: the android emulated devices. 

An *emulated device* is a software simulation of a real device (like a pixel phone or some samsung tablet) that behaves the same as the real device it is trying to emulate. It is a way to test a device without having one handy, that makes it easier to play with different os and api versions installed on the device, and cheaper because you do not have to own the device.  

There is a lot to get familiar with when setting up android and emulated devices. Easiest way to get started is to download *android studio*. With that you can setup the emulators using an the studio's ui and help. But that will not be an option when automating tests in a pipeline. So what follows is a description of setting up the necessary tooling without android studio. 

The most important of what we will need is the following:
1. **Java** (installation will not be covered here)
2. **Avd manager** (android virtual device manager): a tool to create and delete emulated (=virtual) devices. [link](https://developer.android.com/tools/avdmanager)
3. **Sdk manager**: a tool to download packages needed for setting up an avd(android virtual device) and more android related tooling. [link](https://developer.android.com/tools/sdkmanager)
4. **Adb** (android debug bridge): Tool to manage and inspect a running emulated device [link](https://developer.android.com/tools/adb)
5. **Emulator tool**: a tool that starts up the virtual device with settings of your choice [link](https://developer.android.com/studio/run/emulator-commandline)


### Installing the tooling

Android has made installing pretty straigh forward with the [command line tools package](https://developer.android.com/tools). Follow the instructions described [here](https://developer.android.com/tools/sdkmanager), or read on for my description for linux. After downloading, unzipping and setting up some paths about what tooling is where, you should now be ready to run the avdmanager and the sdkmanager. Everything else you need can be downloaded with the sdkmanager. 

On unbuntu running the following in bash downloads the software and packages needed to setup and run the emulated devices
(assuming Java is already setup correctly):
```
#https://developer.android.com/tools/sdkmanager
ANDROID_SDK_ROOT=~/Android/sdk #can be any dir you like
TEMP_DIR=~/Downloads #also, anywhere you like

ANDROID_TOOLS_ZIP="commandlinetools-linux-10406996_latest.zip"
BUILD_TOOLS_VERSION="33.0.2"
ANDROID_VERSION="34"

wget https://dl.google.com/android/repository/${ANDROID_TOOLS_ZIP} -P $TEMP_DIR 
unzip $TEMP_DIR/$ANDROID_TOOLS_ZIP -d $ANDROID_SDK_ROOT  
cd $ANDROID_SDK_ROOT
mkdir ./cmdline-tools/latest 
cd cmdline-tools
mv NOTICE.txt source.properties bin lib latest/

cd ./cmdline-tools/latest/bin
sdkmanager "platform-tools" "emulator" "platforms;android-$ANDROID_VERSION"  "build-tools;$BUILD_TOOLS_VERSION"

```

You do need to add some directories to your path, so you can run the tools without navigating to the directory the tool is in. You need to log in or open a new bash window for it to take effect
```
export ANDROID_SDK_ROOT=~/Android/sdk
export PATH="$PATH:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/emulator:$ANDROID_SDK_ROOT/platform-tools:$ANDROID_SDK_ROOT/build-tools/${BUILD_TOOLS_VERSION}"

```

Once you have the tooling, you can make and run emulated devices. 


### Setting up and running the emulator

To make an emulated device, we need to provide the device (for instance *pixel 6*) and what is called a system image (for instance: *system-images;android-34;google_apis;x86_64*). 

The system image is a combination of an android api level, a google api variant (basically with or without playstore support) and an ['cpu instruction set'](https://en.wikipedia.org/wiki/Instruction_set_architecture). You can check out which combo's are available with the command `sdkmanager --list`. Since not all of the system image parts might already be on your computer or container, you probably need to download the system image parts first. 

To see which devices are available, use `avdmanager list`. If you list the available devices, you will see most of them are googles or generic. 

The code to setup an emulator you find here
```
API_VERSION="android-34" 
API_VARIANT="google_apis"
ARCH="x86_64" 
SYSTEM_IMAGE="system-images;${API_VERSION};${API_VARIANT};${ARCH}" #eg system-images;android-34;google_apis;x86_64

sdkmanager --install "${SYSTEM_IMAGE}"
EMULATOR_DEVICE="pixel_6"
EMULATOR_NAME="${EMULATOR_DEVICE}_${API_VERSION}"
avdmanager --verbose create avd --force --name "${EMULATOR_NAME}" --device "${EMULATOR_DEVICE}" --package "${SYSTEM_IMAGE}"

```
if all is well, `avdmanager list` output should show your newly created emulated device. Now we can run the emulated device with the [emulator tool](https://developer.android.com/studio/run/emulator-commandline). There are plenty of options to play around with when starting the emulator [link](https://developer.android.com/studio/run/emulator-commandline#common). These range from settings about the network speed, the screen size, memory, cameras, to specific flags not render on screen or play audio.
```

emulator @pixel_6_android-34
#or 
emulator --name pixel_6_android-34

```
run emulator -list-avds to get names of installed emulators


### Limitations of emulators

**Some functionality not supported by emiulators**
For a list of things that are simply not possible with emulators. [link](https://developer.android.com/studio/run/advanced-emulator-usage#limitations)

**Not all phone types testable by emulator**
You might know that most mobile phone manufacturers do not use stock android, but build there own ui on top of android (like oneUI for samsung). Sadly, it does not seem possible to emulate these other android flavours in a simple way (at least I have not found anu emulator system images for samsung or any other android phones). To make an emulator look more like different devices, you can use skins. But as far as I understand they are just cosmetic: they do nothing more but make the running emulator look like the real device on screen.   


## Tools to control and inspect the device

There are 2 tools in the android ecosystem you can use to inspect, debug and control running emulated devices. the *emulator console* and *adb*. You do not always not need to know much about these tools, since appium will set it up and use it for you. But when using appium for android in an automated environment, you will run into some problems (ports and address bindings) that can only be solved with some understanding about the setup of both emulator console and adb. Likewise you can do without understanding the commands available with telnet and adb (since webdriver / appium have you covered), but is good to know yourself, since they might come in handy when inspecting and debugging. 

### Android Debug Bridge (adb)

To control the device, we can use [adb](https://developer.android.com/tools/adb). Adb can be used to take screenshots or recordings, install apps, push and pull files from and to the device. It also provides shell access to the device (with lots of your favorite linux commands (see ls /system/bin), provides sqlite3 to view databases and more.(examples later on)  With adb you can also check which devices are up and running (`adb devices`), output logs (`adb logcat`). To allow root access (if possible), use `adb root`.

### Emulator Console 

Some things you will only be able to do with adb if you have a root shell. And a root shell is only possible on a non-production build and with some specific google api (not playstore). The [emulator console](https://developer.android.com/studio/run/emulator-console) can help out then. It allows you to play with stuff like battery, phone calls, rotation and many others. To use the console,  Typing in "help" will give you a list of possible commands. 

For a comparison of what each tool can do, check out this [page](https://developer.android.com/studio/run/emulator-comparison)

Both ADB and the emulator console need to be reachable (by a client) when running tests with appium, the clients need to be able to reach the emulated device. 

#### on the device
Adb and the console will be started automatically on an emulator when the emulator is fired up. By convention the port used on the emulator to connect to the the console is 5554 and the adb (daemon) is 5555 (and it increases if you start multiple emulators: eg 5556 and 5557). Though you can also specify the ports when starting an emulator. 

#### from the client
Adb itself consists of  a client (that can run anywhere), and a daemon (running on the emulated device or its host?, listening by default on port 5555). The client of adb will mostly be started up when you type in an adb command (like `adb devices`) and will scan a port range to find daemons. The client normally listens on port 5037 for commands coming from for instance appium. 

Wen using console you must connect to it with telnet `telnet ip port`. (with 5554 being the default). The console is running on the emulated device.  how to get the ip ?. Telnet is not specific to android and is probably already installed on your computer. When running `adb devices`, the telnet port is used in the name (for instance *emulator-5554*)


## Setting up the appium server and the uitautomator driver
Now that we have an emulator up and running, we need something to communicate with it. For the e2e tests, that will be the appium server. Remember that the appium server just listens for webdriver protocol requests (coming from the webdriverio client, which is explained later on), and passed those along to a driver. The driver and the appium server need to be on the same machine. 
 
todo (deno?)
The Appium server can be installed simply with [yarn or npm](https://appium.io/docs/en/2.1/quickstart/install/). 
Once you have appium, you can install any driver using appium. Since we are automating android, the [uiautomator2](https://github.com/appium/appium-uiautomator2-driver) is needed: `appium driver install uiautomator2`.   
Documentation [at](https://github.com/appium/appium-uiautomator2-driver/tree/master/docs)

The uiautomator driver is dependend on java and some of the android tooling (mostly the platform tools). So for the driver to run we need also to install these. But if you are running the server and the emulator on the same machine, there are no extra steps. 

The appium server can be started with the command `appium`. There are multiple flags, described [here](https://appium.readthedocs.io/en/stable/en/writing-running-appium/server-args/). One flag to take not of is `--relaxed-security`. Using this flag on startup, will allow you to use adb in the test yourself. This gives more flexibilty from time to time (but do read [this](https://appium.io/docs/en/2.1/guides/security/) for an explanation of the risks)


### What does the real automation: a bit about the uiautomator2
As said before, you need not know a lot about the uiautomator2 driver since it will be used for you by appium. But you might be curious about how the uiautomator2 driver does it magic, that is how does it automate swipes, screen captures, calls etc ? 

Well, you can read more about it [here](https://github.com/appium/appium-uiautomator2-driver), but in a nutshell...., it uses other libraries, with even more client and servers. The uitautomator2 driver itself is a client, that has a server/daemon counterpart. This server sits on the same computer as the emulated device (well, it gets installed on when running test), and it executes tests using adb, the emulator console and a different library called [ui-automator](https://developer.android.com/training/testing/other-components/ui-automator) (not the predecessor of uiautomater2). The uiautomator  is describes as  "UI testing framework suitable for cross-app functional UI testing across system and installed apps" . You can use it directly without appium to write tests in java/kotlin. 

So the whole chain of automation is from webdriverio client, to appium server, uitautomater2 driver (itself a client), to the uitautomater2 server, wich makes use of the uiautomator library, adb and telnet.



## Questions to ask yourself before getting started writing tets
Before writing the e2e test, it is good to ask some questions about how to arrange your code. These come up for pretty much any e2e framework

### How to find elements 
Since the e2e test are going to have to swipe or touch elements, we need to know how to find them. With android, there are several ways, but each has its upside's and downside with regards to stability, maintenance (and speed). Any way you choose, you will probably always need  some maintenance / work procedures if changes are mode to the code/ui. 

- You can always find an element by the type /attributes and or position in the whole hierarchy of elements. You can either use xpath or the [uiselector](https://developer.android.com/reference/androidx/test/uiautomator/UiSelector)
By prefacing the selector with android= ('android=new UiSelector().text("I agree")'), you can tell the webdriverio client you want to use the uiselector api. An xpath selector does not require any preface, just the xpath expression:  `//android.widget.Button[@text="I agree"]` or `//[@text="I agree"]`.


- By id: you can add ids to componenents that are needed in the tests. then you should be able to select them with the id= selector
This is a popular approach since it is a lot easier. 

- By accessability: you can use the tilde '~' before a text to use an accessibility label. 
Best way for e2e tests in my hubmle opionion is to use accessability. Anything you swipe or press should also have a clear text or label that is used for accessibility and can be used for tests. 

But see [here](#Issues-with-selectors) for some problems with both ids and accessability labels. 


#### How to find the right selector ?
If you have android studio, you can use the [layout inspector](https://developer.android.com/studio/debug/layout-inspector). If that is not an options, you might also use the [appium inspector](https://github.com/appium/appium-inspector). Both allow you to check a view of your app for so you can construct your selector. While slow, I found the appium inspector very helpfull. 
(you might also use chrome devtools to debug, but this requires some code changes in the app: see https://developer.chrome.com/docs/devtools/remote-debugging/webviews/ and not sure how it is supposed to work)


### How to await and timeout 
When writing tests, especially asynchronous, things might not be ready when you think they are. 
Firstly, an element you expect to be there might not be rendered for a second, eventhough your test is already looking for it. Luckily, webdriverio automatically retries finding your element.  (appium can also do reties, but webdriverio recommends not using that to prevent strange behavior https://webdriver.io/docs/autowait/)

### How to setup / deal with app state ?
Some scenarios you want to test will depend on some other state being present. 
For instance, a test to delete something should fail if there is nothing to delete. In some cases it might make sense to first add this in a test and in a follow up test delete it. But not everyone thinks this is a good idea, because tests should fail as much is possible because there is an issue with deletion, not with some other test that also happens to setup the data needed for a later test. So what are your options ?

#### Remote Data
If it is stored in a database and fetched in the app, you can obviously seed the database with testdata before running the test. 
Unfortunately, there is no easy way to stubbing specific requests (though some might say you should not do that in an e2e): Appium is not sitting inbetween the device and the internet, so it has really no way to do that. If you want that, you would need to setup someting inbetween the device and its services. (something like https://github.com/mock-server/mockserver)

#### Local Data
But if the app state is local? There are some techniques available, best described here in this blog by [headspin](https://www.headspin.io/blog/making-your-appium-tests-fast-and-reliable-part-5-setting-up-app-state). Another way would be to create or overwrite the data stored on the app. For that you would need to know where the data is stored, how you can access it so you can either update, delete or overwrite it. 

Local app data is stored in the `/data/data/[app-name]/` directory. Here you should be able to find a databases or xmls related to the local storage used by your app. What the name will be, is different on the programming language / library / framework you are using. For instance,the react-native asyncstorage library will store the data in a  database  named RKstorage. For apps in java the preferences are stored in an [xml](https://developer.android.com/training/data-storage/shared-preferences). 

Access to the data is a bit more complicated. You need root access (elevated privileges). The command `adb root` helps you get root for *debug builds* of your app running on an emulator *without the google play api*  (https://developer.android.com/studio/run/managing-avds#system-image).
One workaround is to use the ['run-as' command](https://gist.github.com/jevakallio/452c54ef613792f25e45663ab2db117b) (not did not try it out).
And remember, if you want to use root in your tests with appium, you need to run appium server with the --relaxed-security flag. 
 

## Into the test code. 

### the webdriverio client and deno as test runner
Assuming we have installed and setup everything we need (android, emulators, appium, driver), we just need to install one last thing: the appium client. Webdriverio made a javascript client for appium you can use. Webdriverio also offers a cli you can use to run tests. 

However, I choose to run these examples in [Deno](https://docs.deno.com/runtime/manual). Deno is a javascript runtime that comes with build in support for a lot of javascript features (await, arrow syntax ) and typescript, both of which require some more setup with any nodejs (based) runner. Deno also provides some basic support for [testing](https://docs.deno.com/runtime/manual/basics/testing/). So apart from installing the webdriverio and some deno test extensions and setting up your editor to recognize deno ([vscode instructions](https://docs.deno.com/runtime/manual/references/vscode_deno/) there is no further setup required. 


in Deno you export the modules you need in a depts.ts file. With running `deno cache deps.ts` will install them.
```
//deps.ts
export { remote } from 'npm:webdriverio';
export type { RemoteOptions } from 'npm:webdriverio';

export {
  assertEquals,
} from "https://deno.land/std@0.202.0/assert/mod.ts";

export {
  afterAll,
  afterEach,
  beforeAll,
  beforeEach,
  describe,
  it,
} from "https://deno.land/std@0.202.0/testing/bdd.ts";
```

### Setting up and Connecting to the appium server
In the setup for the test we want to get the driver object (with the api's to write our tests in) and start a video recording. In the teardown we want to save and stop the video, and delete the driver session. 

```

const wdOpts: RemoteOptions = {
  hostname: Deno.env.get("APPIUM_HOST") || 'localhost',
  port: 4723,
  logLevel: 'silent',
  capabilities, #3
};

describe("some test", ()=>{

let driver: WebdriverIO.Browser

  beforeEach(async function() {
    console.log("getting driver")
    driver = await remote(wdOpts) # 1
    await driver.startRecordingScreen()
  })

  afterEach(async function(){
    await driver.saveRecordingScreen("./vid2.mp4")
    await driver.stopRecordingScreen()  
    await driver.deleteSession();
  })
  ...

})
```

1. We can get a *webdriverio object* with all the api functions webdriverio provides to write our tests in with the *remote* function. This function takes in a *RemoteOptions* object that contains  urls and ports used to connect to appium, what level of logging we want. The webdriverio object contains the functions to find elements, touch, click swipe, find attributes, start screenrecordings etc. 
2. On of the functions provided by the driver. We use it to start a recording in the beforeEach hook and stop in the the afterEach hook.
3. keep reading on to learn about the capabilities


### Capabilities and settings

We also need to specify "Capabilities":  [`the name given to the set of parameters used to start an Appium session.`](https://appium.io/docs/en/2.1/guides/caps/). This is where you specify what emulator (device) you want to use, what app to install and start up by default, set some caps related to saving state. Some of the caps are specific to a device (Android vs ios, app vs browser). See [here](https://appium.readthedocs.io/en/stable/en/writing-running-appium/caps/) for a description of appium supported capabilities, and [here](https://github.com/appium/appium-uiautomator2-driver) for the capabilities specific to the uiautomator2 driver.


This is an example of the capabilities
```
  const capabilities = {
    # related to the device and its driver
    "platformName": "Android",
    "appium:deviceName": "Android",             
    "appium:automationName": "UiAutomator2", #-> the name of the driver doing the automation
    "appium:avd":  "Pixel_6_API_34",         #-> name of the emulator that is going to be used, it should exist but it will be started if not already running    

    # related to the app to test
    "appium:appPackage": "org.wikipedia.dev",#->the application id of your app. You can find it in the build.gradle?? file. 
    "appium:app": PACKAGE_PATH,              #->the path to where the apk of the build you want to test is located. Will be installed.   
    "appium:appActivity": "org.wikipedia.main.MainActivity", #-> what screen of your app you want to open
  }

```


#### Activities
In the capabilities, you can also specify an activity. 
An activity can be described as a specific view where a user can interact with the app. Normally each functionality or behavior of the app might have an activity. For instance activities might be the onboarding of an app, the main landing page, rating some article etc.  You can read more about activities in this [intro](https://developer.android.com/guide/components/activities/intro-activities).

The following 3 things are important to know for testing with regards to activities:
- The *appActivity* capability will help you go to a specific activity directly when starting the test. 
- You can find the activities that are defined for an app in its [manifest file](https://developer.android.com/guide/topics/manifest/manifest-intro).  
- If you have a running emulator, you can find the current activity of the app with the following adb command: `adb shell dumpsys window displays | grep -E 'mCurrentFocus|mFocusedApp'`


#### saving app state in between test
Some noteworthy settings of appium are the noreset and fullreset options. This controls whether the state of the app is saved after a test or is reset (noreset, default = true), and whether the app and its data is removed completely (fullreset). [See](https://appium.readthedocs.io/en/stable/en/writing-running-appium/other/reset-strategies/). With a fullreset set to true, you wipe the app and any data by starting a new session (getting a new driver). 
(Similarly, if you play around on the emulator (without appium) and do not want to save the state you left it in (or do), you can use the snapshots>Settings>autosave to quickboot option.) 


### Writing a Test:  Api basics

#### Example: basic user interactions
The following examples show the use of the most basic actions, like tapping, swiping typing etc, and the api's you can use for that, while implementing a basic test for the onboarding of the android wikipedia app (Todo link). 

During the onboarding of the app, some info is presented, you can add languages, and also opt out of sending data. 
The test simply continues on the first 3 onboarding steps, then rejects the step about data collecting and then checks if the setting  is set to "no". 

The first part is straightforward shows some ways to navigate forward through the steps. 

```
  it("opting out of event logging when onboarding", async ()=>{
    
    // skip the first step by clicking continue (see 1)
    const continueButton = await driver.$(`//android.widget.Button[@text="Continue"]`)
    await continueButton.click()
    await sleep(2000)

    //skip the second and third step by swiping from right to left (2a)
    await driver.executeScript("mobile: swipeGesture", [{
      "left": 30, "top": 900, "width": 800, "height": 20,
      "direction": "left",
      "percent": 1
    }]);
    await sleep(2000)

    //the second swipe is executed with a different api (2b)
    await driver.performActions([{"type":"pointer","id":"finger1","parameters":{"pointerType":"touch"},
      "actions":[{"type":"pointerMove","duration":0,"x":900,"y":937},{"type":"pointerDown","button":0},
      {"type":"pointerMove","duration":250,"origin":"viewport","x":100,"y":937},
      {"type":"pointerUp","button":0}]}])
  

    const selector= `//android.widget.Button[@text="Reject"]`
    const button = await driver.$(selector)
    await button.touchAction('tap')    

    await driver.$('//android.widget.TextView[@text="Featured article"]').waitForDisplayed()
  })

```
We want to skip the first three steps of the onboarding (the data collection is the 4th step). We use 1 button click to continue, and two navigation swipes.

1. The most basic function is the findElement(s) function, which has a shorthand form $ ($$ for findElements). It takes in one of the selectors referred to [here](#How-to-find-elements). We use the click function to click it.  (Noteworthy here is that there is an function touchAction that does not seem to work for buttons located near the buttom of the screen. This is described [here](#issues-with-selectors))

2.  There are 2 api's to do gestured.  I find the [mobile-gestures api](https://github.com/appium/appium-uiautomator2-driver/blob/master/docs/android-mobile-gestures.md) easier to understand then the [actions ap](https://appium.readthedocs.io/en/latest/en/commands/interactions/actions/).
    Notice the use of the sleep function here. While webdriverio can retry clicks/findelement with selectors, that is not an option when swiping part of a screen.

3. Once we reach the data collection step, we tap reject with the *touchAction* function.  

4. After the onboarding is complete, we check if we end up on the view that shows a 'featured article'. (todo is waitfordisplayed necessary.??) 
 

In the next part of the test, lets make sure the privacy setting is saved in the preferences. 
It navigates to the settings by starting the settings activity, since the test is not about navigating to the settings, but just to check whether the 'send user data' / privacy setting is correct. 

```       
    //requires adb root 
    await driver.startActivity("org.wikipedia.dev", "org.wikipedia.settings.SettingsActivity"); #5

    # scroll the button in view (6)
    let sbutton = await driver.$("android=new UiScrollable(new UiSelector()).scrollIntoView(text(\"Privacy\"));")            
    
    const val = await sbutton.getAttribute("selected")
    assertEquals(val,"false")

```

Two things to note here: 
5. The `startActivity` requires adb to be running as root (`adb root`). If that is not an option, I think the only alternative is to navigate to the settings by clicking the relevant buttons
6. the privacy setting is not in view immediately. That means (unlike in html) you will not be able to find the settings with the normal selector, because only ui-element that are actually rendered on screen, will be found. Instead you need to use the uiselector scrollable object (todo link). 
7.  Finally, we check whether the selected state for this privacy setting is false by using the getAttribute function. 

it all looks like this:
<video  src="{{ site.baseurl }}/media/appium-js/vid1.mp4" controls preload > </video>

#### executescript 

In this example the executescript command is used with the 'mobile : swipeGesture' parameter.  The executescript command can do a lot [more](https://appium.readthedocs.io/en/latest/en/commands/mobile-command/). For instance, the executescript command can be used to power up a shell. This is usefull for updating settings and databases, as we will see in the next section. 
 

#### Example 2: local data setup and searching

In the next step, lets gets fancy with the executescript command to do some setup of local data. 
Since the appium driver will reset the app after every deletesession (TODO), for every fresh test we need to handle the onboarding. While we can just press the skip button, it is also possible to update the local preferences with the executescript command. (not saying that is the better solution here, just as an example for executescript and local preferences)

After some investigation, you will find the local preferences are saved in the "/data/data/org.wikipedia.dev/shared_prefs/org.wikipedia.dev_preferences.xml" file. (Remember, all local app data resides in the /data/data/package dir). This xml file contains an entry `<boolean name='initialOnboardingEnabled' value='false'/>` that is added after the onboarding to indicate the onboarding need not be started.  



```

    driver = await remote(wdOpts)
    await driver.activateApp("org.wikipedia.dev")
    const result = await driver.executeScript('mobile: shell',
        [{
          'command': 'sed' ,
          'args': [ "-i \"$ i <boolean name='initialOnboardingEnabled' value='false'/>\" /data/data/org.wikipedia.dev/shared_prefs/org.wikipedia.dev_preferences.xml"]
        }])
    console.log(result)
    await driver.closeApp()
    await driver.activateApp("org.wikipedia.dev")

```
8. So this code simply gets a new driver instance (which will do a fresh install of the app and wipes all its data). Then it activates the app to make sure the preferences file exists.


9. The executescript command then is used (with the mobile: shell as first parameter) to execute a script on the emulated device. The script used the `sed` linux tool to insert the relevant line in the preferences file. 
  Simply inserting / updating this entry and restarting the app should not show the onboarding. There is a bit of a problem in that the xml file is not yet created after just installing the app. For the file to be created, you need to open the app once. (so it all seems, did not check the code) 


10. Then the app needs to be restarted to make the app reload the settings file. 

Remember though that using the executescript with a shell requires appium to be started with the --relaxed-security flag.

Now that app is skipping the onboarding, we can search for some article on the wikipedia app. 



```
    // Search for software testing
    const button = await driver.$("~Search")
    // await button.touchAction(['longPress', 'release'])
    await button.click();
    await sleep(1000)
 
    
    //searches for software testing
    const textbox = await driver.$('id=org.wikipedia.dev:id/search_card')
    await textbox.touchAction('tap')

    const textInput = await driver.$('id=org.wikipedia.dev:id/search_toolbar')
    await textInput.sendKeys("software testing".split(""))

    //Clicks the first search result about software testing
    const firstResult = await driver.$("id=org.wikipedia.dev:id/page_list_item_title")
    const title = await firstResult.getAttribute("text")
    assertEquals(title, "Software testing")
    await firstResult.touchAction('tap')
    
    // then redirected to the page about software testing    
    await sleep(1000)
    let activity = await driver.getCurrentActivity()
    assertEquals(activity, "org.wikipedia.page.PageActivity")    
    await driver.$(`//android.widget.TextView[@text="Software testing"]`)
    
```

The rest of the test is pretty straightforward. 
- use of the sendkeys command to type. 
- use of getCurrentActivity to check if we are indeed redirected to a PageActivity. 
- instead of using the xpath selector, the id= shorthand is used to select widgets.

it all looks like this:
<video  src="{{ site.baseurl }}/media/appium-js/vid2.mp4"   preload controls > </video>


#### where to look when things go wrong
In the appium settings, in the examples the logging has been set to 'silent'. That is because appium server logs are more usefull when things go wrong. It will by default log for instance permission errors, and wrong usage of api's. So I would suggest keeping a close eye on those logs while writing tests.  Logcat,  which can be started with `adb logcat` also provides a lot of logging to study (https://developer.android.com/tools/logcat) 

### Issues and problems

#### Issues with selectors
Both the accessibility and the id strategies might have some problems. 
The accessibility seems to respond to the content-description attribute of elements. But for textelements, this attribute should not be used (source), eventhough an accessibilityselector seems not to respond to textelements. 
Also the id is not without problems. At least for react native, the Id must be set using the testID attribute, and by adding the full package name +id path before the actual name (https://github.com/facebook/react-native/issues/32237)

#### Tap, touch or click ?
One of the problems I encountered with playing with the wikipedia android app, is that it is not possible to use the touchAction tap for buttons located near the bottom of the screen. This had to do with some bug about the button not being inside a rectangle (even though it is clearly visibile and touchable on the emulator). Clicking does work luckily, and longpressing also works.  Though what does it mean to click something on a phone (see https://discuss.appium.io/t/difference-between-click-and-tap/295) ?  

#### API galore
There are so many api's and examples in different languages that I was confused from time to time. Was I not using it correct, or was it not usable for android, or not with this specific client ?

### Choices /TODOS

#### Test runners
Though Deno offers some testing support out of the box, it is not as feature rich as you would like your test runners to be in a testing pipeline. For instance there is as of yet no support for retries, or timeouts. Webdriverio cli might make the most sense to use. 

#### Android frameworks:
Can you test this way with all the app flavors (android, kotlin, reactnative, flutter)?
Flutter seems to have its own appium driver. 

#### What alternatives were available ?
**android ecosystem**
There is espresso, for e2e / ui testing of android apps. (scope is within the app) 
There is the uiautomator to write android tests. (scope is within the device)

**general**
There is detox for ios and android app
There is playwright for webview with android (experimental)

#### can I re-use the tests for an IOS app ?
It seems to run the default ios driver, you need macos (or run macos inside a vm). So i could not try. 


https://developer.android.com/studio/test
aapt dump badging pathwithapk



