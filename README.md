# Borderlands Twitch Login

**Allows users to login to BL2 and TPS with their (or their bot's) Twitch account, and allows mod developers to make mods with Twitch features using their login.**

![Twitch Login](https://i.imgur.com/16ITbmE.png)

Borderlands Twitch Login makes it easy to create and use mods which provide integration with Twitch! Want a channel point redemption to make the streamer drop their gun? Spawn Badass enemy with the name of the last subscriber? Create channel point predictions about gun drops? Borderlands Twitch Login makes all of these things simple to implement and install. (Mods not included.)

## For Users: Installation

To be able to run mods featuring Twitch functionality using Twitch Login, users only need to install those mods, as well as Twitch Login. To install Twitch Login:

1. [Install UnrealEngine PythonSDK](https://github.com/bl-sdk/PythonSDK#installation) if you have not already.

2. Download the current version of [`Borderlands-Twitch-Login-main.zip`](https://github.com/mopioid/Borderlands-Twitch-Login/archive/refs/heads/main.zip).

3. Locate the SDK's `Mods` folder (located in the `Win32` folder of the `Binaries` folder of your BL2/TPS installation).

4. Copy the `TwitchLogin` folder from `Borderlands-Twitch-Login-main.zip` to the SDK's `Mods` folder.

5. Launch the game, select "Mods" from the main menu, and select Twitch Login.

6. The options to manage your Twitch account login are listed in the bottom right.

## For Developers: Modding API

Twitch Login handles all of the implementation details of interacting with the Twitch API for you. It obtains a Twitch OAuth token permitting any scopes required by mods, which it uses to send requests and listen to PubSub topics on mods' behalf.

If you are interested in making BL2 & TPS mods utilizing Twitch features, then first, here are some
resources you might find useful:

* [A simple guide to Python](https://learnxinyminutes.com/docs/python/)
* [Getting started writing BL2 & TPS mods](https://bl-sdk.github.io/faq/#developer-faq)
* [Things possible via requests to the Twitch API](https://dev.twitch.tv/docs/api/reference)
* [Things able to be received via the Twitch PubSub API](https://dev.twitch.tv/docs/pubsub)

Next, on to the TwitchLogin API. A very simple mod using Twitch Login looks like this:

```python
import unrealsdk
from Mods import ModMenu, TwitchLogin

def channel_points_redeemed(data):
    try: redemption = data["message"]["data"]["redemption"]["reward"]["title"]
    except KeyError: return
    if redemption == "Test Redemption":
        unrealsdk.Log("Redemption redeemed")

def setup_redemptions():
    def callback(status, data):
        if status == 200:
            unrealsdk.Log("Redemption created successfully")

    TwitchLogin.Requests.Post(
        "channel_points/custom_rewards",
        Params = { "broadcaster_id" : TwitchLogin.UserID },
        Data = { "title": "Test Redemption", "cost": 1000 },
        Callback = callback
    )

@TwitchLogin.RegisterWhileEnabled
class TwitchExample(ModMenu.SDKMod):
    Name = "Twitch Example"
    Version = "1.0"
    Author = "mopioid"

    TwitchScopes = [ "channel:read:redemptions", "channel:manage:redemptions" ]
    TwitchTopics = { "channel-points-channel-v1.{UserID}" : channel_points_redeemed }

    def TwitchLoginChanged(self, logged_in):
        if logged_in:
            setup_redemptions()

ModMenu.RegisterMod(TwitchExample())
```

This mod registers for two scopes: `channel:read:redemptions` and `channel:manage:redemptions`. These allow it to listen for messages on the PubSub topic `channel-points-channel-v1.{UserID}`, and send requests to `https://api.twitch.tv/helix/channel_points/custom_rewards`.

This example will be broken down in the full documentation to follow.

### Implementing an SDKMod Object

When designing your standard SDKMod object, there are a few additions you can make to it to implement features of the TwitchLogin API.

#### `TwitchScopes` attribute

```python
class TwitchExample(ModMenu.SDKMod):
    TwitchScopes = [ "channel:read:redemptions", "channel:manage:redemptions" ]
```

To register for scopes, a mod may optionally define a `TwitchScopes` attribute, containing a sequence of strings representing scopes in the Twitch API; e.g. "channel:read:redemptions". See: https://dev.twitch.tv/docs/authentication#scopes

TwitchLogin attempts to acquire authorization for each scope requested by mods in this way. The user is directed to the Twitch authorization webpage, in which they are presented with the complete list of scopes requested.

Upon authorization, Twitch grants permission for each valid scope. If any scopes requested by mods were not granted, i.e. they were invalid scopes, an error will be logged to the console and TwitchLogin's `logging.log` file.

#### `TwitchTopics` attribute

```python
def channel_points_redeemed(data):
    try: redemption = data["message"]["data"]["redemption"]["reward"]["title"]
    except KeyError: return
    if redemption == "Test Redemption":
        unrealsdk.Log("Redemption redeemed")

class TwitchExample(ModMenu.SDKMod):
    TwitchTopics = { "channel-points-channel-v1.{UserID}" : channel_points_redeemed }
```

A mod may optionally define a `TwitchTopics` attribute, containing a mapping of strings to callables. The strings serving as keys represent topics in the Twitch PubSub API. See: https://dev.twitch.tv/docs/pubsub#topics

Topic strings may optionally include the format token `{UserID}`. If it does, the token will be replaced with the user ID (A.K.A. channel ID) of the user's Twitch account when listening for the topic. For example, a string of `channel-points-channel-v1.{UserID}` will listen for the topic `channel-points-channel-v1.44322889`.

The callable each topic maps to is invoked each time a message is received for its respective topic. Callables should accept a single parameter; upon invocation of the callable, a dictionary containing the `data` field of the message will be passed to this parameter.

If Twitch returns any errors when attempting to listen for a topic, the error will be logged to the console and TwitchLogin's "logging.log" file.

#### `TwitchLoginChanged` method

```python
class TwitchExample(ModMenu.SDKMod):
    def TwitchLoginChanged(self, logged_in):
        if logged_in:
            setup_redemptions()

```

If the mod defines a `TwitchLoginChanged` method, it will be invoked each time the user's login status changes. This method should accept a boolean as the only parameter after `self`. If the user has successfully authenticated, a value of `True` will be provided. If they have logged out, a value of `False` will be instead.

### The `TwitchLogin` Module

```python
from Mods import TwitchLogin
```

Twitch Login itself serves as an importable module, accessible to other mods from `Mods.TwitchLogin`. It provides access to the interactive features of the API.

#### `TwitchLogin.RegisterMod` Method

```python
class TwitchExample(ModMenu.SDKMod):
    def Enable():
        super().Enable()
        TwitchLogin.RegisterMod(self)
```

Register the Twitch scopes and PubSub topics specified by the given mod's `TwitchScopes` and/or `TwitchTopics` attributes, and register the mod to have its `TwitchLoginChanged()` method called when the user's authentication changes.

#### `TwitchLogin.UnregisterMod` Method

```python
class TwitchExample(ModMenu.SDKMod):
    def Disable():
        TwitchLogin.UnregisterMod(self)
        super().Disable()
```

Remove the given mod's registration from TwitchLogin. It will no longer receive messages on its PubSub topic callbacks, and scopes it registered for will not be requested in future user authorizations. It will also not receive notifications of the user's login status changing on its `TwitchLoginChanged` method.

#### `TwitchLogin.RegisterWhileEnabled` Decorator

```python
@TwitchLogin.RegisterWhileEnabled
class TwitchExample(ModMenu.SDKMod):
    pass
```

A convenient decorator for SDKMod classes that configures them to be registered with TwitchLogin while enabled. SDKMod classes with this decorator do not need to invoke `TwitchLogin.RegisterMod()` and `TwitchLogin.UnregisterMod()` manually.

More specifically, `TwitchLogin.RegisterMod()` is called on instances of the mod class at the end of its `Enable()` method, and `TwitchLogin.UnregisterMod()` is called on it before its `Disable()` method is run.


#### `TwitchLogin.Token`

A string representing the OAuth token that provides the user's current authentication. If the user is not currently authenticated, this will be `None`.

#### `TwitchLogin.Scopes`

A sequence of strings representing the scopes for which API access is permitted via the user's current authentication.

#### `TwitchLogin.UserName`

A string representing the username associated with the user's currently authenticated account. If the user is not currently authenticated, this will be `None`.

#### `TwitchLogin.UserID`

A string representing the user ID associated with the user's currently authenticated account. If the user is not currently authenticated, this will be `None`.


### `TwitchLogin.Requests` Submodule

```python
def callback(status, data):
    if status == 200:
        unrealsdk.Log("Redemption created successfully")

TwitchLogin.Requests.Post(
    "channel_points/custom_rewards",
    Params = { "broadcaster_id" : TwitchLogin.UserID },
    Data = { "title": "Test Redemption", "cost": 1000 },
    Callback = callback
)
```

Requests to the Twitch API are provided via the `TwitchLogin.Requests` submodule. Requests are performed asynchronously; when making a request, a mod specifies a callback function to be invoked when the request completes. The results of the request are passed as parameters to the provided callback.

The scopes required for a given Twitch API request must have been registered for as per the [`TwitchScopes` attribute](#twitchscopes-attribute) and [`RegisterMod` method](#twitchlogin-registermod-method) described above. The user must have then logged in upon being notified of their requirement to do so. Without the authorized scope, the Twitch API will respond to the request with an error.

Twitch Login supports `GET`, `POST`, `PUT`, `PATCH`, and `DELETE` requests. These request types may be performed via convenient methods: `TwitchLogin.Requests.Get`, `TwitchLogin.Requests.Post`, `TwitchLogin.Requests.Put`, `TwitchLogin.Requests.Patch`, and `TwitchLogin.Requests.Delete`. Each of these methods accepts the following arguments:

| Argument | Description |
| --- | --- |
| `Path`     | The path in the Twitch API URL to send the request. This should be the entire path after "/helix/"; for example, specfying `channel_points/custom_rewards` causes the request to be sent to `https://api.twitch.tv/helix/channel_points/custom_rewards`. |
| `Params`   | Optional dictionary, list of tuples, or bytes, to append to the URL of the request as its query. If a dictionary or sequence of tuples `(key, value)` is provided, the contents will be form-encoded. See: https://docs.python-requests.org/en/latest/api |
| `Data`     | Optional object to send as the request of the body. A JSON-serializable dictionary is generally recommended for the Twitch API, but this may also be a sequence of tuples, bytes, or file-like object. See: https://docs.python-requests.org/en/latest/api |
| `Callback` | The function to be invoked upon completion of the request. This function must accept two arguments - An `int` representing the status code of the response, and a `dict` containing the content of the response, if any. |
