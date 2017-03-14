# Weather Bot Demo

This tutorial creates a bot for Slack that can help notify users with weather forecasts. Users can ask the bot for the forecast for a specific location. The bot can also be configured to send the forecast for a location at regular intervals, e.g. everyday at 8am.

This bot needs to perform a few functions, e.g. convert addresses to locations, retrieve weather forecasts for locations and integrate with Slack. Rather than implementing the bot as a monolithic application, containing logic for all these features, we want to deploy it as separate services.

Later on, we'll look at using a neat feature of the platform (sequences) to bind them together to create our bot without writing any code.

*There's video recordings for these demonstrations available [here](https://youtu.be/GrtWmGPmWsM) and [here](https://youtu.be/WcsQXohs_hE).*

# Getting Started with OpenWhisk

## Setting up the CLI

1. Navigate to OpenWhisk from your Bluemix Dashboard or go to https://console.ng.bluemix.net/openwhisk
2. Click on `Download OpenWhisk CLI`
3. Follow the 3-step instructions and make sure that the `wsk` command is working in your terminal

## Working with the OpenWhisk UI

1. Switch back to your browser and click `Back to Getting Started` or go to https://console.ng.bluemix.net/openwhisk
2. Click `Develop in your Browser`

We can now see the OpenWhisk user interface. There should be two actions included in your space by default.

Click on the `Hello World` action and then click `Run this Action`

The action executes and you should see the response:
```
{
  "message": "hello world"
}
```

Now click on the `Hello World with Params` action and run that too. On the *Invoking an Action* page you'll see there is a box for you to provide JSON Input.

Change this value to:
```
{
    "message": "hello from OpenWhisk"
}
```
And run the action. You should get the response:
```
{
  "message": "you sent me hello from OpenWhisk"
}
```

# Creating the Functions for our Bot

Now that we're familiar with how to run actions from the UI let's upload the functions we need for the bot.

Download the following two functions:
https://raw.githubusercontent.com/edshee/owtutorial/master/forecast_from_latlong.js
https://raw.githubusercontent.com/edshee/owtutorial/master/location_to_latlong.js

*You may need to right click on the links above and "save file as"*

Open a terminal window and navigate to the folder with your newly downloaded JS functions:
```
$ cd /path/to/functions/
```

## Address To Locations Service

This service handles retrieving latitude and longitude coordinates for addresses using an external Geocoding API.

```
$ cat location_to_latlong.js
var request = require('request');

function main(params) {
  var options = {
    url: "https://maps.googleapis.com/maps/api/geocode/json",
    qs: {address: params.address},
    json: true
  };

  return new Promise(function (resolve, reject) {
    request(options, function (err, resp) {
      if (err) {
       	console.log(err);
        return reject({err: err})
      }
      if (resp.body.status !== "OK") {
       	console.log(resp.body.status);
        return reject({err: resp.body.status})
      }

      resolve(resp.body.results[0].geometry.location);
    });
  });
};
```

This microservice implements the application logic within a single function (_main_) which is the interface used by OpenWhisk. Parameters are passed in as an object argument (_params_) to the function call. This service makes an API call to the Google Geocoding API, returning the results from API response for the first address match.

Returning a Promise from the function means we can return the service response asynchronously.

Let's deploy this service and invoke it to verify it's working…

```
$ wsk action create location_to_latlong location_to_latlong.js
ok: created action location_to_latlong
$ wsk action invoke location_to_latlong -b -r -p address "London"
{
    "lat": 51.5073509,
    "lng": -0.1277583
}
```

*The -r flag tells the command-line utility to only return the service result rather than the entire invocation JSON response. The -b flag blocks waiting for the service to finish rather than returning after the invocation starts.*

Great, that's working. We've deployed our first microservice without a server anywhere…

Let's try the next one for finding forecasts for locations.

## Forecast From Location Service

This service uses an external API to retrieve weather forecasts for locations, returning the text description for weather in the next twenty fours hours.

_Provision a new instance for the [Weather Insights](https://new-console.ng.bluemix.net/catalog/services/weather-company-data/) service in IBM Bluemix to access these API credentials._

```
$ cat forecast_from_latlong.js
var request = require('request');

function main(params) {
  if (!params.lat) return Promise.reject("Missing latitude");
  if (!params.lng) return Promise.reject("Missing longitude");
  if (!params.username || !params.password) return Promise.reject("Missing credentials");

  var url = "https://twcservice.mybluemix.net/api/weather/v1/geocode/"+params.lat+"/"+params.lng+"/forecast/daily/3day.json";
  var options = {
    url: url,
    json: true,
    auth: {
      user: params.username,
      password: params.password
    }
  };

  return new Promise(function (resolve, reject) {
    request(options, function (err, resp) {
      if (err) {
        return reject({err: err})
      }

      resolve({text: resp.body.forecasts[0].narrative});
    });
  });
}
```

This service expects four parameters, latitude and longitude coordinates along with the API credentials. Passing in API credentials as parameters mean don't have to embed them within code and can change them dynamically at runtime.

Let's deploy this service and verify it's working…

```
$ wsk action create forecast_from_latlong forecast_from_latlong.js
ok: created action forecast_from_latlong
$ wsk action invoke forecast_from_latlong -p lat "51.50" -p lng "-0.12" -p username $WEATHER_USER -p password $WEATHER_PASS -b -r
{
    "text": "Partly cloudy. Lows overnight in the low 60s."
}
```

Yep, looks good.

We don't want to pass in the API credentials with every request, so let's bind them as default parameters to the action. This means we only need to invoke the service with the latitude and longitude parameters, which matches the output from the previous service.

```
$ wsk action update forecast_from_latlong -p username $WEATHER_USER -p password $WEATHER_PASS
ok: updated action forecast_from_latlong
$ wsk action invoke forecast_from_latlong -p lat "51.50" -p lng "-0.12"  -b -r
{
    "text": "Partly cloudy. Lows overnight in the low 60s."
}
```

Okay, great, that's the first two services working.

## Sending Messages To Slack

Slack is basically a messaging app on steroids.

It's meant for teams and workplaces, can be used across multiple devices and platforms, and is equipped with robust features that allow you to not only chat one-on-one with associates but also in groups. You're able to upload and share files with them too as well as integrate with other apps and services. You can granularly control almost every setting, including the ability to create custom emoji.

Go to https://slack.com/create and create a new team. When it gets to the `send invitations` screen just click on `skip for now`.

Once you are up and running with your team channel go to https://api.slack.com/apps/new and create a new App for your team.

*Your app can be called whatever you like but make sure you select the team you just created as the "Development Slack Team"*

Now click on `Incoming Webhooks` (on the left hand side) and enable the "Activate Incoming Webhooks" checkbox.

Click on `Add New Webhook to Team` at the bottom of the page and select the channel "#General" in the `Post to` dropdown.

Copy the Webhook URL that gets generated. We will need it for the next step.

In your browser, navigate back to your OpenWhisk dashboard and click on `Browse Public Packages`.

*Packages* are a way for OpenWhisk users to re-use actions that have been created by others. There are a lot of publicly available packages but you can also publish your own (Privately or Publicly).

Click on `Slack` in the package browser and click on `New Binding` in the bottom left.

Choose any name for your configuration and pick a username (e.g. OpenWhisk). Make sure that the `channel` is set to `General` and then paste your slack webhook url in to the `url` field.

Click on `Save Configuration`

Now click on `Run this Action`. In the JSON Input field change the value to:
```
{
    "text": "Hello from OpenWhisk!"
}
```
Click on `Run with this Value`

If all has gone successfully you should see a new message appear in your slack channel saying "Hello from OpenWhisk!".

## Creating Weather Bot Using Sequences

Right, we have the three microservices to handle the logic in our bot. How can we join them together to create our application?

OpenWhisk comes with a feature to help with this, called _Sequences_.

Sequences allow you to define Actions that are composed by executing other Actions in order, passing the output from one service as the input to the next. This is a bit like Unix pipes, where a single command executes multiple commands and pipes the output through the commands.

Let's define a new sequence for our bot to join these services together.

Click on the `location_to_latlong` action and then click on `Link into a Sequence` (Hint: bottom right!)

Click on `My Actions`, select the `forecast_from_latlong` action and click `Add to Sequence`

Our sequence now sends the result of `location_to_latlong` in to `forecast_from_latlong` meaning we can give it a location and it'll return the forecast. Yay!

We don't want our sequence to stop there though. We'd like it to post this result straight to our slack team.

Click on `Extend` from the sequence view and then select `Slack`.

Make sure the binding you created earlier is selected (on the left side) and then click `Add to Sequence`.

Now click on `This Looks Good`, give your sequence a name and save it.

Click on `Run this Sequence` and run it with the following input:
```
{
    "address": "London, UK"
}
```

Excellent! It worked. You're now able to change the address parameter and the weather forecast will appear in your slack team.

## Connecting To Triggers

Triggers are used to represent event streams from the external world into OpenWhisk. They can be invoked manually, through the REST API, or automatically, after connecting to trigger feeds.

Actions can be bound to Triggers using Rules. When a Trigger is fired, the Action is invoked with the request parameters. Multiple Actions can listen to the same Trigger.

![OpenWhisk architecture](https://raw.githubusercontent.com/openwhisk/openwhisk/master/docs/images/OpenWhisk.png)

Let's look at binding the bot service to a sample trigger.

Click on the sequence you created and click on `Automate`

*We can see a selection of feeds that could be used as triggers in OpenWhisk. It's possible to configure your own feeds so that actions can be automated off all sorts of events.*

Let's pick the `Periodic (Alarm-based Triggers)` option. Click `New Alarm`.

*We can create a scheduled trigger so that actions will run on a set time period. E.g. we may want to get a notification of the weather at 8am every morning.*

Select the Minutes tab at the top and then enter `1` in the `Invoke every N Minutes` field. For the payload let's pass in the following (feel free to specify your own):
```
{
    "address": "Bangalore, India"
}
```

Give your trigger a name and create it.

Now click on `Next` and then `This Looks Good`. Finally `Save Rule`.

*We've now created a new Rule (based on a new trigger) that will run our sequence every minute and alert us to the weather in Bangalore*

You're now fully equipped to go out and start building your own OpenWhisk actions. **Great Success!!!**
