---
layout: post
title:  Updating Uncleared Home Assistant Android Notifications with AppDaemon
date:   2022-07-04
tags: homeassistant appdaemon
description: "With a simple AppDaemon script, we can send notifications with a given tag to devices that haven't previously cleared them."
---


# Goals

For one of my Home Assistant automations, I want to send [notifications to a couple of Android devices](https://companion.home-assistant.io/docs/notifications/notifications-basic/). I also want to be able to update the contents of those notifications when I get new sensor readings, so [per the documentation I use a tag](https://companion.home-assistant.io/docs/notifications/notifications-basic/#replacing) to replace the existing notifications.

But there's one snag that Home Assistant doesn't support out of the box: if I dismiss (clear) a notification on one of the Android devices, then the next time I try to send an update it will pop up again.

Instead, I want notifications to exist in sort of a session. The session starts the first time the automation sends a notification, and stops when the automation sends a [`clear_notification`](https://companion.home-assistant.io/docs/notifications/notifications-basic/#clearing) message. During the session, new updates to the notification will only be sent to Android devices that haven't cleared the notification over the course of the session.

> ***Note: this only works for Android devices, since iOS doesn't report when notifications are cleared by the user.***

# Complications

To accomplish this I wrote an [AppDaemon](https://appdaemon.readthedocs.io/) script which will send notifications and listen for the [`mobile_app_notification_cleared`](https://companion.home-assistant.io/docs/notifications/notification-cleared) event to know when a notification has been cleared.

Unfortunately the [`mobile_app_notification_cleared`](https://companion.home-assistant.io/docs/notifications/notification-cleared) event doesn't contain useful information about what device cleared the notification--it contains a `DEVICE_ID` which is a hexadecimal string, but I can't find any way to easily map that `DEVICE_ID` back to the actual string representing the target (i.e. the `<your_device_id_here>` in the [`notify.mobile_app_<your_device_id_here>`](https://companion.home-assistant.io/docs/notifications/notifications-basic) service call).

However, when a notification is cleared the event does contain all the data of the notification--including its tag. Thus, by sending each notification with a unique tag that contains both the name of the AppDaemon app *and* the name of the specific target, we can distinguish between devices when we get the [`mobile_app_notification_cleared`](https://companion.home-assistant.io/docs/notifications/notification-cleared) event.

# High level overview

At a high level, the AppDaemon app works as follows:

0. Via the app's [YAML configuration](https://appdaemon.readthedocs.io/en/latest/APPGUIDE.html#configuration-of-apps) you specify the Android devices you want to target, as well as any default data you want to include in every notification. (I.e. the `data` object that's at the same level as the `title` or `message`.)
1. The app listens for a specific event, `notify_android_uncleared_<APP_NAME>`, where `<APP_NAME>` is the name of the app in the app's YAML configuration, and where the event data matches the data you'd send in a [`notify.mobile_app_<your_device_id_here>`](https://companion.home-assistant.io/docs/notifications/notifications-basic) service call.
2. When it first sees the `notify_android_uncleared_<APP_NAME>` event, it adds all the target devices to the list of active devices, and sends them all the notification. With every notification, it overwrites the `tag` field in the notification's `data`, to the form `<APP_NAME>_<your_device_id_here>`.
3. The app also listens for the [`mobile_app_notification_cleared`](https://companion.home-assistant.io/docs/notifications/notification-cleared) event. If it receives it, it checks to see if the `tag` in the event's `data` matches the form `<APP_NAME>_<your_device_id_here>`. If so, it removes the corresponding device from the list of active devices.
4. If the app hears the `notify_android_uncleared_<APP_NAME>` event again, it only sends the updated notification to targets that are still active.
5. If the app hears the `notify_android_uncleared_<APP_NAME>` event but the `message` is [`clear_notification`](https://companion.home-assistant.io/docs/notifications/notifications-basic/#clearing), then the app sends that message to all the remaining active targets, then sets a flag to indicate the session is over. The next time it hears the `notify_android_uncleared_<APP_NAME>` event it starts over at step (2).

# The code

Here's the AppDaemon code, hopefully commented in enough detail to make sense:

```python
import appdaemon.plugins.hass.hassapi as hass


class UpdateUnclearedAndroidNotifications(hass.Hass):
    def initialize(self):
        # self.notification_targets is a set (list) of all the targets we should send notifications to
        self.notification_targets = set(self.args.get("notification_targets"))
        # self.active_targets is the list of tags that haven't been cleared; or None if the tag is inactive
        self.active_targets = None
        # self.message_data is the default additional data we send in a notification
        self.notification_data = self.args.get("notification_data", dict())
        # we use a unique tag for every target to detect when a notification is cleared
        self.tag_prefix = self.name

        # Listen for the event that tells us to send a new message
        self.listen_event(self.notify_android_uncleared_callback, "notify_android_uncleared_" + self.tag_prefix)
        # Also listen for the event that tells us a notification was cleared        
        self.listen_event(self.notification_cleared_callback, "mobile_app_notification_cleared")


    # This method is called whenever we want to send or update a notification
    def notify_android_uncleared_callback(self, event_name, data, kwargs):
        self.log("event_name={}, data={}, kwargs={}".format(event_name, data, kwargs), level="DEBUG")
        
        # If the tag isn't active, initialize all the notification targets
        if self.active_targets is None:
            self.active_targets = set(self.notification_targets)

        # Then for all remaining active targets for the given tag:
        for target in self.active_targets:
            # Send the notification
            self.send_notification(target, data["message"], data.get("title", None), data.get("data", dict()))
        
        # If the message was the special message `clear_notification`:
        if data["message"] == "clear_notification":
            # Then deactivate the tag by removing it
            self.active_targets = None
            self.log("Clearing tag {}".format(self.tag_prefix))


    # This method actually sends the notification to the given target
    def send_notification(self, target, message, title, data):
        # First we make a copy of the default notification data
        notification_data = self.notification_data.copy()
        # Then we override it with whatever data was passed in by the event
        for key in data.keys():
            notification_data[key] = data[key]
        # And finally we override the tag so it's unique to each target
        notification_data["tag"] = self.tag_prefix + "_" + target
        self.log("Sending message={}, title={}, data={} to target={}".format(message, title, notification_data, target), level="DEBUG")
        # And then send the notification, with or without a title
        if title is not None:
            self.call_service("notify/"+target, title=title, message = message, data = notification_data)
        else:
            self.call_service("notify/"+target, message = message, data = notification_data)


    # This method is called when an Android device clears any notification
    def notification_cleared_callback(self, event_name, data, kwargs):
        # First check if the data contains a tag, and it starts with the tag prefix
        if "tag" in data and data["tag"].startswith(self.tag_prefix):
            # Then extract the target from the tag
            target = data["tag"][len(self.tag_prefix)+1:]
            self.log("target {} cleared notifcation with tag {}".format(target, self.tag_prefix), level="DEBUG")
            # And if the tag is active and the target is active, remove it
            if self.active_targets is not None and target in self.active_targets:
                self.log("removing target {}".format(target), level="DEBUG")
                self.active_targets.remove(target)
```

And here's some sample YAML to set up the app:

```yaml
my_notification_tag:
  module: update_uncleared_android_notifications
  class: UpdateUnclearedAndroidNotifications
  notification_targets:
    # These is the same as the <your_device_id_here> you would use in the notify.mobile_app_<your_device_id_here> service call
    - mobile_app_device_id_1
    - mobile_app_device_id_2
  # notification_data and all of its sub-fields are optional, and should match the data you'd provide to a notify.mobile_app_<your_device_id_here> service call
  # Also note that any data passed by the event will override this default data once, for that particular notification
  notification_data:    
    alert_once: true
    sticky: true
    clickAction: "/lovelace-default/some-url"
    tag: some-tag # Note that if you specify a tag, it will be overwritten    
```

Then to send the notification to all of the devices that haven't cleared it yet, you'd fire off an event like so:
```yaml
- event: notify_android_uncleared_my_notification_tag # Note that my_notification_tag is the name of the AppDaemon app instance
  event_data:
    title: Some title # title is optional
    message: Some message # message is required
    data: # data and all its subfields are optional
      color: "red"
      icon_url: "https://github.com/home-assistant/assets/blob/master/logo/logo-small.png?raw=true"
      tag: some-tag # Note that if you specify a tag, it will be overwritten
```

# Using the app

To use the app, follow the [instructions in the AppDaemon documentation](https://appdaemon.readthedocs.io/en/latest/APPGUIDE.html). You'll need to create a new python file named `update_uncleared_android_notifications.py`, and then drop the AppDaemon YAML in `apps.yaml` or another YAML config file (and configure it appropriately). Then just fire off the event from Home Assistant like shown, and you're all set!