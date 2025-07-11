# Firebase Cloud Messaging Delivery Method

Send Device Notifications using the Google Firebase Cloud Messaging service and the `googleauth` gem. FCM supports Android, iOS, and web clients.

```bash
bundle add "googleauth"
```

## Google Firebase Cloud Messaging Notification Service

To generate your Firebase Cloud Messaging credentials, you'll need to create your project if you have not already. See https://console.firebase.google.com/u/1/
Once you have created your project, visit the project dashboard and click the settings cog in the top of the left sidebar menu, then click project settings.

![Firebase Console](../images/fcm-project-settings.png)

In the project settings screen click on the Service accounts tab in the top navigation menu, then click the Generate new private key button.

![Service accounts](../images/fcm-credentials-json.png)

This json file will contain the necessary credentials in order to send notifications via Google Firebase Cloud Messaging.
See the below instructions on where to store this information within your application.

## Usage

```ruby
class CommentNotification
  deliver_by :fcm do |config|
    config.credentials = Rails.root.join("config/certs/fcm.json")
    config.device_tokens = -> { recipient.notification_tokens.where(platform: "fcm").pluck(:token) }
    config.json = ->(device_token) {
      {
        message: {
          token: device_token,
          notification: {
            title: "Test Title",
            body: "Test body"
          }
        }
      }
    }
    config.if = -> { recipient.android_notifications? }
  end
end
```

## Options

### `json`
Customize the Firebase Cloud Messaging notification object. This can be a Lambda or Symbol of a method name on the notifier.

The callable object will be given the device token as an argument.

There are lots of options of how to structure a FCM notification message. See https://firebase.google.com/docs/cloud-messaging/concept-options for more details.

### `invalid_token` - *Optional*
A lambda to allow your app to clean up invalid tokens.

### `error_handler` - *Optional*
A lambda to allow your app to handle other errors with FCM, such as incorrect configuration.
Can be useful to squash intermittent errors you don't want polluting your error collection service
or perform other clean up actions.

### `credentials`
The location of your Firebase Cloud Messaging credentials.

#### When a String Object

Internally, this string is passed to `Rails.root.join()` as an argument so there is no need to do this beforehand.

```ruby
deliver_by :fcm do |config|
  config.credentials = "config/credentials/fcm.json"
end
```

#### When a Pathname object

The Pathname object can point to any location where you are storing your credentials.

```ruby
deliver_by :fcm do |config|
  config.credentials = Rails.root.join("config/credentials/fcm.json")
end
```

#### When a Hash object

A Hash which contains your credentials

```ruby
deliver_by :fcm do |config|
  config.credentials = credentials_hash
end

credentials_hash = {
  "type": "service_account",
  "project_id": "test-project-1234",
  "private_key_id": ...,
  etc.....
}
```

#### When a Symbol

Points to a method which can return a Hash of your credentials, Pathname, or String to your credentials like the examples above.

We pass the notification object as an argument to the method. If you don't need to use it you can use the splat operator `(*)` to ignore it.

```ruby
deliver_by :fcm do |config|
  config.credentials = :fcm_credentials
  config.json = :format_notification
end

def fcm_credentials(*)
  Rails.root.join("config/certs/fcm.json")
end
```

#### Otherwise

If the credentials option is left out, it will look for your credentials in: `Rails.application.credentials.fcm`

## Gathering Notification Tokens

A recipient can have multiple tokens (i.e. multiple Android devices), so make sure to return them all.

Here, the recipient `has_many :notification_tokens` with columns `platform` and `token`.

```ruby
def fcm_device_tokens(recipient)
  recipient.notification_tokens.where(platform: "fcm").pluck(:token)
end
```

## Handling Failures

Firebase Cloud Messaging Notifications may fail delivery if the user has removed the app from their device.
In this case, FCM will return a [400 or 404 HTTP Status Code](https://firebase.google.com/docs/reference/fcm/rest/v1/ErrorCode)
and the delivery method will call the `invalid_token` handler, if you configure one.

```ruby
class CommentNotification
  deliver_by :fcm do |config|
    config.invalid_token = ->(device_token) { NotificationToken.find_by(token: device_token).destroy }
  end
end
```

In case of other errors, the delivery method call the `error_handler` handler, if you configure one,
or otherwise simply raise `Noticed::ResponseUnsuccessful` and the job can be retried.

```ruby
class CommentNotification
  deliver_by :fcm do |config|
    config.error_handler = ->(response) { Rails.logger.error(response.inspect) }
  end
end
```
