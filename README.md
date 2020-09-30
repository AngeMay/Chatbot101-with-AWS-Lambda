# Chatbot 101 with AWS Lambda

[![published](https://static.production.devnetcloud.com/codeexchange/assets/images/devnet-published.svg)](https://developer.cisco.com/codeexchange/github/repo/Cpanchal1/Chatbot101-with-AWS-Lambda)

In this lab, we will introduce how we can leverage serverless computing through an example of creating an interactive Webex Teams chatbot. Our chatbot will wait for a user to message it in Webex Teams and will respond by posting a joke. For the purposes of this guide we'll host the code as a serverless function.

Serverless (or Function-as-a-Service) environments enable developers to upload their code as a series of functions on a cloud platform, such as AWS. Despite the name, serverless functions do actually run on servers, but resources for the application are only provided as needed. Once code is uploaded, you can allow it to be triggered by certain events, such as a HTTP/REST API call.  

## Overall Architecture

![](./images/architecture.png)

In this lab you will:

1. Create a Chatbot *(Webex Developer Portal)*
2. Code the Chatbot's actions *(Python)*
3. Use AWS Lambda to host and trigger the Chatbot's actions *(AWS Lambda platform)*

You will need:

1. A free Webex Teams account
2. A Webex Teams Desktop or Browser Client
3. A free-tier Amazon AWS account
4. Python environment and a text editor on your laptop


## The admin stuff...

### Create a free-tier Amazon Web Services account
You can create an AWS account [here](https://portal.aws.amazon.com/billing/signup#/start). You may be asked to enter your credit card details when registering for an account, but all AWS activity in lab will stay within the limits of the free tier (so you will not incur any cost from the activities from this lab).

### Registering for Cisco Webex Teams

During this lab we will use the Cisco Webex Teams API to both send and receive messages for the ChatBot we are going to build. To do that the first thing we do is [register](https://www.webex.com/team-collaboration.html) for a free Webex Teams account. Click **Try Teams free** on the top left corner and complete the sign up process.

![](./images/webex-register.gif)

### Install the Webex Teams client

Once you've registered you may want to test out Webex Teams by sending and receiving a few messages, as during the lab we'll need to send some messages to the bot we're going to create. To do this we need to have a client. To download the client, follow to the link: https://www.webex.com/downloads.html and click the download button for the **Webex Teams** client.

Alternatively you can use the web browser client which can be found at: https://teams.webex.com

### Creating our Bot in Webex Teams

Now we have our Teams account we have access to the developer site for Webex Teams: https://developer.webex.com. This portal has guides to the Teams API, SDK's available and example projects to help you on your developer journey. We won't go into the details of this site in this lab but we do need to create our bot and get an access token.

Your bot's access token is required to use any of the Webex APIs. In order to create rooms, post a message or retreive any details using Webex Teams APIs, your bot will use this token to authenticate itself. 

To create the bot and obtain its access token, navigate to the **My Apps** page of your account https://developer.webex.com/my-apps select **Create New App** and when prompted choose the Bot type then complete the form which asks for Username / Bot Name, Bot Icon and a short description. Once submitted, you will then be given the access token. ***Copy and paste this to another document as we will use it later.***

![](./images/create-bot.gif)

### Creating a Webex Teams Room

In this section you will create a new Webex Teams space and add your bot to the space.

1. Open Webex Teams and create a new teams space by pressing the "+" sign on the client

3. Give your new space a name 

4. If you have not already added your bot do it now by clicking the dotted button top right and selecting people then use the option to "add people"

5. Type the name of the Bot you created.

![](./images/create-room.gif)

### Obtaining the Room ID 

You will need the Room ID of the Teams Room you just created with the bot for when it comes to programming the bot's actions. 

1. Go to www.developer.webex.com. Click on **Documentation** -> **API Reference** -> **Rooms** -> **List Rooms**. The API form is populated for you, just press **Run**.

2. Find the room ID associated with the space you created earlier. It's normally the first room. Copy the "id" from the response and paste it somewhere for later use. Remember to remove the quotes.

![](./images/get-token.gif)

COOL! Now you have a teams space and a bot, but it's not very interesting yet. We now need to teach it how to talk and do stuff.


## Coding the Chatbot


Open your preferred text editor...time to write some code! We'll be using Python because it is a very popular language for scenarios such as this. There are many other options for runtimes to build your functions on including Go, NodeJS, Java, Ruby and .NET. If you prefer to skip ahead to the AWS stuff, you can clone the 'chatbot.py' file in this repo, replace the ROOMID and TOKEN variables to match the roomId and Bot Token you obtained earlier and move straight on to the packaging part. 

The code is made up of four components:

1. The tokens we'll be needing later in the code. \
The variables `ROOMID` and `TOKEN` will be used later so you don't have to copy and paste the actual tokens repetitively throughout the code. It minimizes error and is far more efficient if you ever want to change these values in the future. These variables are capitalised as this denotes that the contents of these variables shouldn't be changed throughout the code (not an official python rule, but followed by developer community). \
Replace the `insert-roomid-here` with the ROOMID you obtained earlier - ensure you keep the double quotes. Replace the `insert-bot-token-here` with your bot's access token - ensure you keep the double quotes and the word `Bearer`. 


```python
ROOMID = "insert-roomid-here"
TOKEN = "Bearer insert-bot-token-here"
```

![](./images/add-creds.gif)

2. The '`getJoke`' method which retrieves a joke from the icanhazdadjoke.com database. 
In this method, we make one API call. The `url` variable holds the location of the API resource. The `headers` variable describes what data structure to expect, in this case, it is JSON. You can find these details, plus anything else you'd need to include in your API call, by looking through the [API Documentation](https://icanhazdadjoke.com/api). \
In this method, we make one API call. The `url` variable holds the location of the API resource. The `headers` variable describes what data structure to expect, in this case, it is JSON. You can find these details, plus anything else you'd need to include in your API call, by looking through the [API Documentation](https://icanhazdadjoke.com/api). 

``` python
def getJoke():

    url = "https://icanhazdadjoke.com"
    headers = {
        "Accept":"application/json"
    }
    
    response = requests.request("GET", url=url, headers=headers)
    return response.json()["joke"]
```


3. The '`postJoke`' method which posts this joke to Webex Teams. \
In this method, we are using the url which points to the Webex Teams (formerly called Cisco Spark) messages resource. Using the '**POST**' method, we are able to push a message on Webex Teams. We specify three things in this API call:
- from which account the message should be sent. This is done by providing an authorization token, which in this case belongs to the bot we created earlier. 
- which Teams room we'd like to post the message into. We use the RoomID we obtained earlier.
- what message we would like to send. In this case, it is the joke we retrieved from the `getJoke` method. 


```python
url = "https://api.ciscospark.com/v1/messages"
headers = {
    "Authorization": TOKEN,  # Bot's access token
    "Content-Type": "application/json"
}
payload = {
   "roomId": ROOMID,
    "text": joke
}

response = requests.request("POST", url, data=json.dumps(payload), headers=headers)
```

4. The '`main`' method which triggers the two methods above.
This is the connection between retrieving the joke and posting it on to Webex Teams. First, it calls the `getJoke()` method and stores the response in a variable called `joke`. Then, it feeds this joke as a parameter when it calls the second method, `postJoke(...)`. This is also the function that will be triggered when someone tries to interact with our bot. 


## Packing and uploading


Now that the code is complete, let's get it ready for AWS Lambda. There is quite a specific way that Lambda likes to have the code packaged. All the dependencies that the python code requires needs to be packaged up in a separate folder. In our python code, the `requests` library is not included in the Lambda python interpreter, so we must package this up before the upload. 

### **Step 1** - Package the dependencies our code (including the dependencies).

Using Terminal on a Mac or Command Prompt on Windows, navigate to the same directory as your 'chatbot.py' file. If you've cloned this repo, it'll be in the same folder as this README. Type the following commands in order to place all dependencies into a directory called 'package':

```
pip install --target ./package requests
cd package
```

Now we will zip the dependencies and the python code.

**On Mac:**
```
zip -r9 ${OLDPWD}/function.zip .
```
![](./images/package-code-1.gif)

Next, we need to add the chatbot.py file into the zip folder. Do this by typing in the following:
```
cd $OLDPWD
zip -g function.zip chatbot.py
```

![](./images/package-code-2.gif)

**On Windows:**
Use your mouse (not Command Prompt) to navigate to the 'Chatbot101-with-AWS-Lambda' directory. Create a new folder called 'function'. Drag both 'chatbot.py' and the 'package' folder into it. Right-click the 'function' folder -> **Send To** -> **Compressed (zipped) folder**. 

Now our bot code is all ready to be uploaded!
### **Step 2** - Upload to Lambda function


Log into your AWS account, select **Services**. Under the 'Compute' section, select **Lambda**. Select **Create Function** - we will be creating from scratch. 

Give your function a name and select Python3.8 as the runtime environment. Again, select **Create Function** to complete the process. 

![](./images/create-function.gif)

Now that your function has been created, you need to upload your function.zip file. You can do this by scrolling down to the 'Function Code' section and selecting **Actions** -> **Upload a .zip file**. 

![](./images/upload-function.gif)

Finally, let's change some settings to help our chatbot run smoothly. Scroll down to 'Basic Settings' and select **edit**. Set the handler to be chatbot.main as we want to trigger the 'main' method in the 'chatbot.py' script. Finally, change the timeout to be 1 minute so the script has enough time to fully execute. 


## Triggering the Chatbot to react - APIs and Webhooks


This is what will tie all your hard work together! We've got a chatbot. We've got instructions on what the chatbot should do. We've got our serverless function. So what are we still missing? The trigger. Your chatbot should only retrieve and post a joke on Webex Teams when you interact with it (it'd be quite annoying if it worked any other way).

So we need two things here. The first is an API, the second is a [webhook](https://developer.webex.com/docs/api/guides/webhooks). 

The API we create will trigger the main function in chatbot.py (remember the handler we changed in the step above). 
A webhook listens out for a specific event. It subscribes to alerts and notifies us each time a certain event occurs. We want to be alerted each time someone messages our bot, so that our flow can be triggered. How will it notify us? It will call the API we've created using the **POST** method, which will in turn trigger the chatbot.py script.

### **Step 1** - Creating the API:

![](./images/create-api.gif)

a. On AWS, select **Services**. Under the 'Networking & Content Delivery' section, select **API Gateway** -> **Create API**. We will build a new **REST API**. Select **Build**. Give your API a name. 

b. Once finished, you'll see a mostly-empty page with an **Actions** button. Use this to **Create Resource**. Give your resource a name, e.g. 'messages' and select **Create Resource**. 

![](./images/create-method.gif)

c. This resource now needs REST methods assigned to it. Select **Actions** -> **Create Method**. Using the dropdown list, select **POST** and click on the tick button. Check the 'Use Lambda Proxy integration' box and type in the name of the function you created earlier in the 'Lambda Function' textbox. Save and finally use the **Actions** -> **Deploy API** tab to finish making your API. You will then see the API url which you'll need for the next step. It will end with a /resource-name where 'resource-name' is the name of the resource you created in step 1b.

![](./images/deploy-api.gif)

Congrats on creating your first AWS API!

### **Step 2** - Creating the webhook:

Go to www.developer.webex.com. Select **Documentation** -> **API Reference** -> **Webhooks** -> **Create a Webhook**.

Here we specify what event we're listening out for:

* **Authorization** Use the bot's bearer token which you copied earlier

* **name:** Create any name for your webhook

* **targetUrl:** The url you copied at the end of Step 1c 

* **resource:** The resource we're listening out for relates to 'messages'

* **event:** When a new message is 'created'

* **filter:** We don't want the Bot to reply to every single message on Webex Teams (that would be annoying!); we only want the bot to respond with a joke when it has been specifically mentioned (e.g. @bot), so we use a filter called 'mentionedPeople'. We set 'mentionedPeople=me', where 'me' refers to the Bot in this case (because we used the Bot's authorization token). 

![](./images/create-webhook.gif)

Click Run to create the webhook. 

You can test that it works by opening the Webex Teams room you created earlier and sending a message to the bot. Don't forget to mention the bot in your message using the '@' symbol! And voila...

![](./images/test.gif)


## ....Congratulations - You've used a serverless function to create an interactive chatbot!
