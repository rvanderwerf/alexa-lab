# alexa-lab

Lab 1:  Create your Amazon Developer account (5 minutes)
============
1)  Go to <a href="https://developer.amazon.com">https://developer.amazon.com</a>
2)  Use the link in the upper right hand corner to sign in to the Amazon Developer Portal or create your account.

Lab 2:  Install alexa skills plugin (10 minutes)
============
1)  Install sdkman
    Follow instructions at <a href="http://sdkman.io/install.html">http://sdkman.io/install.html</a>
2)  Download and install gradle
3)  Download and install grails


add the following to the dependencies {} block to your build.gradle:
```
dependencies {
     compile "org.grails.plugins:alexa-skills:0.1.1"
}
```

also if you have issues resolving the plugin, try adding my bintray repo (NOT to buildscript block but the lower one):

```

repositories {
    maven { url  "http://dl.bintray.com/rvanderwerf/alexa-skills" }
}

```

How it works
============
run 'grails create-speechlet <classPrefix>'
This will create a speechlet class file located in grails-app/speechlets. Do not use the suffix 'Speechlet' because that will be automatically appended to the name.


Configuration
=============

Here's the config values you can set in application.yml or application.groovy:


**alexaSkills.supportedApplicationIds (String)**
 
 - this is a comma delimited list of application Ids you speechlet will recognize. 
 - you can get ID this under Skills->create on developer.amazon.com
 
 
**alexaSkills.disableVerificationCheck (boolean)**

 - if you are debugging and sending in canned request (see 'serializeRequests') via CURL turn this on
 - be very careful not to disable this in a public service as it turns off all checking the request is really coming from Amazon.
 
**alexaSkills.serializeRequests (boolean)**

 - if you are having problems wondering what is going on with your skill or want to capture a request and replay it in CURL it will write them to disk

**alexaSkills.serializeRequestsOutputPath (boolean)** 

 - if you turn on serialization requests this is the path the files will be saved to. The files will be names in the format:
 - files will be saved in the format speechlet-${System.currentTimeMillis()}.out
 
## Intents

Intents are sort of like simplified Intents on android if you have every done Android coding. They signal an intention for a command to run to do something.

Let's look at an example one to get an idea:
```
{
  "intents": [
    {
      "intent": "ResponseIntent",
      "slots": [
        {
          "name": "Answer",
          "type": "AMAZON.LITERAL"
        }
      ]
    },
    {
      "intent": "QuestionCountIntent",
      "slots": [
        {
          "name": "Count",
          "type": "AMAZON.NUMBER"
        }
      ]
    },
    {
      "intent": "AMAZON.HelpIntent"
    }
  ]
}
```

Here this application can trigger 3 intents: ResponseIntent, QuestionCountIntent, and HelpIntent. You use JSON to tell it what these are.
You can define 'slots' which is the expected responses which are comprised of a name and Type. There are pre-defined data types you can use
to help amazon parse what is said (Number for example knows the user is going to say a number. Literal is a simple String. Some intents don't need anything
as they are 'built in' like the help intent (no inputs).
 
A little bit about slots - they are entirely optional. You can do things like define a list of allowed responses (custom slots) that are valid things for the user to say.
In this case the ResponseIntent is free form, and we sort out what we want to do by parsing the string we get from Amazon (translated via TTS).

## Sample Utterances

Let's look at another example:


    ResponseIntent {test|Answer}
    ResponseIntent {last player|Answer}
    ResponseIntent {test test|Answer}
    ResponseIntent {test test test|Answer}
    ResponseIntent {test test test test|Answer}
    QuestionCountIntent {Count} questions


Here we see the intent each utterances (what the person says) and map it to an intent. Amazon will generate common words like conjunctions and ignored things on their
end so you don't have to mess with handling things like 'and', 'the', 'or' etc. Variables are surrounded by {}. You can use the | operator to specify alternate options.
If you want to say something to your skill, it must match the pattern here. For this case, we want to parse multi-world answers to a question.
Basically here we support one, two, three of four word responses - anything else will be cut off/ignored. The last item allows the user to answer how many questions they want to be asked.

## Requirements

  - A publicly available container(Self running Grails jar, Tomcat, Elastic Beanstalk, etc) to launch your app with HTTPS on port 443 (no way around this). Can be self-signed for DEV mode only. 
  - If you want to use audio snippets via SSML they must be hosted as above but with a recognized CA (no self-signed). S3 Static HTTPS buckets are easiest.
  - SSL Certificate common name MUST match hostname (even for self-signed DEV certs)
  

## Let's make a Grails app using the plugin and I will explain more as we go

The plugin is for Grails 3.x only. Let's create a new app first:


    grails create-app skillsTest


Now lets open build.gradle and add the plugin into the dependencies {} closure:


    compile "org.grails.plugins:alexa-skills:0.1.1"


If have an issue resolving the plugin, add my repository on bintray to your app:
 
```
    repositories {
        maven { url  "http://dl.bintray.com/rvanderwerf/alexa-skills" } 
    
    }
```    
 
Now let's create a skill from the command line:


    grails create-speechlet SkillsTest
    | Rendered template Speechlet.groovy to destination grails-app/speechlets/skillstest/SkillsTestSpeechlet.groovy


 
Also the plugin will generate a Controller class embedded in your Speechlet file. If you are using Spring Security, you will want to make sure that uri is
 assessable to the outside to the Alexa service can contact it (there are some requirements I'll fill you in on later): 

```

/**
 * this controller handles incoming requests - be sure to white list it with SpringSecurity
 * or whatever you are using
 */
@Controller
class SkillsTestController {

    def speechletService
    def skillsTestSpeechlet


    def index() {
        speechletService.doSpeechlet(request,response, skillsTestSpeechlet)
    }

}

```

The speechlet artefact will be registered as a Spring bean so it's automatically injected. There is also a service the plugin provides to handle the boring
stuff like verifying the request, checking the app ID (more on that later) and the plumbing that calls your skill.


## Built-in Events

Looking at the code example above, you can see it made several methods for you. These are part of the skill(speechlet) lifecycle.
The first one is:

### onSessionStarted

This allows you to store variables for the duration of the session (the interactions as a whole of the app for that time). You can do 
some setup here and store variables you can use later. This is technically optional for you to implement.
 
### onLaunch

This is called when you invoke the skill. When you say 'Alexa open skillTest' etc this is you chance to say an opening message about your app, what it does, or what they will
need to do. 

### onIntent

This is the meat and is required to be implemented. When your sample utterances maps to an Intent when the user says something this  is invoked.
Here you should generate a card that will appear in the Alexa app on your phone  (you can also get to this on your local network via browser by going to 'echo.amazon.com'. 
Cards are similar to Android cards (more popular in Android Wear) that simply show a message to the user they can see what is going on. You can make several kinds
of cards which are  Simple, Standard, and LinkAccount. Here we have a switch statement to figure out what Intent to process and call the code for the appropriate intent.

### onSessionEnded

This is the last one, optional, where you can clean up and session resources you might have created like database records for the run.


I've added a few other helper methods to see how to render a help response, link an account, and get some setup details from the Grails config. As a first pass, just try to get the default template working, then start to change things like changing the text, add an Intent, etc.


 
## Set up your app on Amazon Developer Portal

This is separate from AWS. I am not aware of any APIs that will create all of this for you so you have to sign up for an account and do this by hand for each skill you want to run.

<img src="http://grailsblog.ociweb.com/images/alexa/developerportal-1.png"/>

* Pull down the 'twitterAuth' app <a href="https://github.com/rvanderwerf/twitterAuth">here</a> to get some Intents/Sample utterances to try. They are located in src/main/resources.

* Sign up for the Amazon developer program <a href="https://developer.amazon.com">here</a> if you haven't already

* Click on Apps and Services -> Alexa

* Click on Alexa Skill Kit / Get Started -> Add New Skill

<img src="http://grailsblog.ociweb.com/images/alexa/developerportal2.png"/>

* Pick any name and any invocation name you want to start the app on your Echo / Alexa Device


<img src="http://grailsblog.ociweb.com/images/alexa/developerportal3.png"/>

* Copy the contents of src/main/resources/IntentSchema.json into Intent Schema.

* Don't fill in anything for slots

* Under Sample Utterances, copy the contents of the file src/main/resources/SampleUtterances.txt

<img src="http://grailsblog.ociweb.com/images/alexa/developerportal4.png"/>

* Under configuration Copy the url for /twitterAuth/twitter/index for the endpoint for your server (Choose amazon https not ARN). Click next

* Leave 'enable account linking' turned off.

* For domain list, enter a domain that matches your SSL cert the oauth tokens will be valid for. You may use a self-signed cert for development mode, but if you want to
publish your skill, your server will need to be running a real recognized certificate (a cheap option is RapidSSL).

* Enter the url for the privacy policy on your server. It can be any valid web page, a link will show during account linking in the alexa app

* Hit Save

* Click on SSL Certificate. If you have a self-signed cert (will only work for DEV mode) paste it here under 'I will upload a self-signed certificate in X.509 format.'

* Hit Save and go to Test page and hit Save

<img src="http://grailsblog.ociweb.com/images/alexa/developerportal5.png"/>

* Go to Privacy and Compliance, and fill out the info there (It's required)

 
Now note the application ID it gives you. You will need to add this to the application.groovy/yml file so the application will know the app ID and accept it.

Copy the application ID on the first tab 'SKILL INFORMATION', and paste that into application.groovy


    alexaSkills.supportedApplicationIds="amzn1.echo-api.request.8bd8f02f-5b71-4404-a121-1b0385e56123,amzn1.echo-sdk-ams.app.84d004e5-e084-4087-a8c3-a80b12fd2009,amzn1.echo-sdk-ams.app.dc1dea0e-ab91-446d-a1c7-460df5e83489"
    alexaSkills.disableVerificationCheck = true // helpful for debugging or replay a command via curl
    alexaSkills.serializeRequests = true // this logs the requests to disk to help you debug
    alexaSkills.serializeRequestsOutputPath = "/tmp/"



Build and deploy your war file to your server (btw it must be port 443 HTTPS, no exceptions)


## Test your app

Now try it on your Echo/Alexa device. Say either 'start' or 'open' and the invocation name you gave the app and follow the prompts! You can also use the test function on the portal itself.

## Debugging

Debugging can be very frustrating. Make sure to turn your log level to debug. In the Grails config, the there is an option called 'serializeRequests' and a output path for them.
This allows you to capture the request that came from Amazon. If you are trying to test a fix for a bug, you can replay this via CURL. The files will look like this:

    -rw-r--r-- 1 tomcat tomcat      598 Jun  6 22:45 speechlet-1465249551692.out
    -rw-r--r-- 1 tomcat tomcat      636 Jun  6 22:45 speechlet-1465249557538.out
    -rw-r--r-- 1 tomcat tomcat      598 Jun  6 22:46 speechlet-1465249601675.out
    -rw-r--r-- 1 tomcat tomcat      636 Jun  6 22:46 speechlet-1465249607216.out

The build-in security provided by the plugin and underlying library will now allow you to reuse a request because the hash ahd timestamps are too old (to prevent this type of attack called a 'replay attack'). You can disable this check for dev purposes with the 'disableVerificationCheck' Grails config value.
Now you can replay a file via CURL back to your server to avoid the whole voice interaction to test that one case (and test it locally!):

    curl -vX POST http://localhost:8080/test/index -d @speechlet-14641464378122470.out --header "Content-Type: application/json"

If you save enough requests of a normal interaction, you could write some functional tests that replay this as part of a test suite or just be able to dev against them locally a bit.    

## Advanced stuff - full UI with Spring Security, OAuth

If your want a UI for the user and use account linking, an easy path is to add Spring Security, Spring Security UI, Spring Security OAuth grails plugins.
You can see an example of how to do this <a href="https://github.com/rvanderwerf/twitterAuth">here</a>.

## Advanced stuff - say sample audio clips in your skill

There is a supported markup called SSML which allows up to 90 seconds of low quality mp3 sound clips to play. Their settings/requirements are quick picky to work:

* Must be hosted on a recognized CA HTTPS port 443 server (I use S3 static website for this)
* Must be 16khz, 48 kb/s MP3 files
* Use the open source tool 'ffmpeg' to encode them - these settings work: 'ffmpeg -y -i -ar 16000 -ab 48k -codec:a libmp3lame -ac 1 file.mp3'
* See the <a href="https://github.com/rvanderwerf/heroQuiz">heroQuiz</a> app for an example of playback sounds being used.

### Example SSML:

        <speak>
          <audio src="\"https://s3.amazonaws.com/vanderfox-sounds/groovybaby1.mp3\"/"> ${speechText}
          </audio>
        </speak>
     
     

## Useful links
* Emulator: <https://echosim.io>
* Account Linking: <https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/linking-an-alexa-user-with-a-user-in-your-system>
* Getting Started Guide: <https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/getting-started-guide>
* Custom Skill Overview: <https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/overviews/understanding-custom-skills>
* Defining the Voice Interface: <https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/defining-the-voice-interface>
* Voice Design Handbook: <https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/alexa-skills-kit-voice-design-handbook>
* Custom Skill JSON reference: <https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/alexa-skills-kit-interface-reference>
* SSML Reference: <https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/speech-synthesis-markup-language-ssml-reference> 

---------------------

## Conclusion
  
The Alexa service is a great invention that is catching on. Google and Apple are dipping their toes into the market. I see the star trek experience in the home
being a reality for everyone in a few years. Already my wife uses it (who swore she never would), and my 4 year old asks it to tell her jokes all the time. I use it to control my lights very often. The sky is the limit, and these tools can be useful in the workplace too. Get out there and build some neat stuff for Alexa with Grails!

