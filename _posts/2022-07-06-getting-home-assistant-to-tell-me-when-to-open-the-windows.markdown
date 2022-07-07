---
layout: post
title:  Getting Home Assistant to Tell Me When to Open the Windows
date:   2022-07-06
tags: homeassistant automations blueprint
description: "My house doesn't have A/C, so I use Home Assistant plus some indoor and outdoor temperature sensors to have an automation notify me when I should open the windows to cool the house down."
---

# Goals

My house doesn't have A/C so I'm primarily dependent on opening the windows to cool down my house. However, I don't want to open the windows if it's too hot outside. I also don't want to have to constantly check the temperature--instead I want Home Assistant to notify me when it's time to open the windows, or if it's time to close the windows because it got warmer outside. I call this a "temperature inversion" to sound fancy, so you'll see that term (or TID for temperature inversion detection) peppered throughout the guide.

To accomplish this, I have four(ish) sensors: an outdoor temperature sensor, an indoor temperature sensor, a window sensor to tell if the window is open or closed, and I also have a sensor that tells me today's high (because if the high isn't very hot, I don't care about the windows being open or not).

The goal is to set the desired state of the windows as follows:
* If it's ***cooler*** outside than inside (by some adjustable margin), **and** it's warmer inside than some threshold, the windows should be open.
* If it's ***warmer*** outside than inside (by some adjustable margin), **and** today's high is higher than some threshold, the windows should be closed.
* Otherwise, the windows can be in any state.

Whenever the windows don't match their desired state, I get a notification.

# Setup

First, I create a bunch of helpers which will serve as knobs for me to configure things:
* An `input_number` with entity ID `input_number.temperature_inversion_difference`, which allows me to adjust how much warmer or cooler it must be outside to get a notification.
* An `input_number` with entity ID `input_number.temperature_inversion_indoor_threshold`, which lets me set the threshold for the indoor temperature above which I want to be told to open the windows. (I.e. if it's already cool inside, I don't need a notification to tell me to open the windows.)
* An `input_number` with entity ID `input_number.temperature_inversion_outdoor_max_high`, which lets me set the outdoor high above which I want to be told to close the windows. (I.e. if the high is going to be a comfortable 74°F, then I don't need a notification telling me to open the windows even if it's warmer outside than inside, because I assume my house will still cool off in the evening.)
* An `input_number` with entity ID `input_number.temperature_inversion_required_duration`, which represents how long in minutes it should be cooler outside before I get a notification. (I.e. if the temperature is bouncing up and down, I don't need a notification yet.)
* An `input_select` with entity ID `input_select.tid_desired_window_state`, which will store the desired window state for other use in Home Assistant.
* An `input_text` with entity ID `input_text.tid_status`, which will store a short summary of the desired action (and be an empty string otherwise).

Next, I ensure that the indoor temperature sensor I'm using is assigned to an Area, so the automation can tell me which room's windows I need to open or close later.

Next, to get today's high, I setup the [AccuWeather integration](https://www.home-assistant.io/integrations/accuweather). It creates a ton of sensors, but the only one I care about is today's high: `sensor.home_realfeel_temperature_max_0d`. I disabled all the others.

Next, in my `configuration.yaml` I create a group to represent all the window sensors in one room, like so:
```yaml
binary_sensor:
  - platform: group
    name: Windows Upstairs Front
    device_class: window
    unique_id: windows_upstairs_front
    entities:
      - binary_sensor.window_sensor_upstairs_den_1_any
      - binary_sensor.window_sensor_upstairs_den_2_any
      - binary_sensor.window_sensor_upstairs_den_4_any
```
This allows me to treat them all as one sensor. Note that this group will be "off" ("closed") only if all the windows in the group are closed. That's OK--I only really care if at least one of the windows is open or not.

Next, also in `configuration.yaml`, I create a couple of [template sensors](https://www.home-assistant.io/integrations/template/):
```yaml
{% raw %}
template:
  - binary_sensor:
      - unique_id: temperature_cooler_outside
        state: >
          {{ states("sensor.indoor_temperature_sensor")|float >= states("input_number.temperature_inversion_difference")|float + states("sensor.outdoor_temperature_sensor")|float }}
        delay_on: '00:{{ states("input_number.temperature_inversion_required_duration")|int(1) }}:00'

      - unique_id: temperature_above_threshold
        state: >
          {{ states("sensor.indoor_temperature_sensor") > states("input_number.temperature_inversion_indoor_threshold") }}
        delay_on: '00:{{ states("input_number.temperature_inversion_required_duration")|int(1) }}:00'

      - unique_id: tid_high_greater_than_threshold
        state: >
          {{ states("sensor.home_realfeel_temperature_max_0d") >= states("input_number.temperature_inversion_outdoor_max_high") }}
{% endraw %}
```
The first indicates whether or not the indoor temperature is greater than the outdoor temperature by some margin (thus it's cooler outside).

The second indicates if the indoor temperature is above the threshold I configure (i.e. it's warm enough inside that I want to know when I can cool things down).

The third just indicates if today's high is greater than the threshold I configure.

# The automation blueprint

With all that set up, I can now just plug it all into the following automation blueprint:

```yaml
{% raw %}
blueprint:
  name: Temperature Inversion Automation
  description: Run specific actions when it's cooler outside than inside, and vice versa.
  domain: automation
  input:
    indoor_temperature_sensor:
      name: Indoor Temperature Sensor
      selector:
        entity:
          domain: sensor
          device_class: temperature
    outdoor_temperature_sensor:
      name: Outdoor Temperature Sensor
      selector:
        entity:
          domain: sensor
          device_class: temperature
    window_binary_sensor:
      name: Window binary sensor
      selector:
        entity:
          domain: binary_sensor
          device_class: window
    cooler_outside_binary_sensor:
      name: Cooler Outside Binary Sensor
      selector:
        entity:
          domain: binary_sensor
    indoors_above_threshold_binary_sensor:
      name: Indoors Above Threshold Binary Sensor
      selector:
        entity:
          domain: binary_sensor
    outdoor_high_greater_than_threshold_binary_sensor:
      name: Outdoor High Greater Than Threshold Binary Sensor
      selector:
        entity:
          domain: binary_sensor
    desired_window_state_input_select:
      name: Desired Window State Input Select
      selector:
        entity:
          domain: input_select
    status_title_text:
      name: Status Title Text
      selector:
        entity:
          domain: input_text
    notification_event:
      name: Notification Event
      selector:
        text:
trigger:
  - platform: state
    entity_id: !input cooler_outside_binary_sensor
  - platform: state
    entity_id: !input indoors_above_threshold_binary_sensor
  - platform: state
    entity_id: !input window_binary_sensor
  - platform: state
    entity_id: !input indoor_temperature_sensor
  - platform: state
    entity_id: !input outdoor_temperature_sensor
  - platform: state
    entity_id: !input outdoor_high_greater_than_threshold_binary_sensor
action:
  - variables:
      indoor_temperature_sensor: !input indoor_temperature_sensor
      outdoor_temperature_sensor: !input outdoor_temperature_sensor
      window_binary_sensor: !input window_binary_sensor
  - service: input_select.set_options
    target:
      entity_id: !input desired_window_state_input_select
    data:
      options: ["on", "off", "any"]
  - choose:
      - conditions:
          - condition: state
            entity_id: !input cooler_outside_binary_sensor
            state: "on"
          - condition: state
            entity_id: !input indoors_above_threshold_binary_sensor
            state: "on"
        sequence:
          - service: input_select.select_option
            data:
              option: "on"
            target:
              entity_id: !input desired_window_state_input_select
          - event: !input notification_event
            event_data:
              title: >-
                {% if states(window_binary_sensor) == "off" %}
                Open the {{ area_name(indoor_temperature_sensor) }} windows
                {% endif %}
              message: >-
                {% if states(window_binary_sensor) == "off" %}
                It is currently {{  states(indoor_temperature_sensor) }}{{ state_attr(indoor_temperature_sensor, "unit_of_measurement") }} in the {{ area_name(indoor_temperature_sensor) }} and {{ states(outdoor_temperature_sensor) }}{{ state_attr(outdoor_temperature_sensor, "unit_of_measurement") }} outside.
                {% else %}
                clear_notification
                {% endif %}
          - service: input_text.set_value
            data:
              value: >-
                {% if states(window_binary_sensor) == "off" %}
                Open the {{ area_name(indoor_temperature_sensor) }} windows
                {% endif %}
            target:
              entity_id: !input status_title_text
      - conditions:
          - condition: state
            entity_id: !input outdoor_high_greater_than_threshold_binary_sensor
            state: "on"
          - condition: state
            entity_id: !input cooler_outside_binary_sensor
            state: "off"
        sequence:
          - service: input_select.select_option
            data:
              option: "off"
            target:
              entity_id: !input desired_window_state_input_select
          - event: !input notification_event
            event_data:
              title: >-
                {% if states(window_binary_sensor) == "on" %}
                Close the {{ area_name(indoor_temperature_sensor) }} windows
                {% endif %}
              message: >-
                {% if states(window_binary_sensor) == "on" %}
                The high today is {{ states("sensor.home_realfeel_temperature_max_0d") }}{{ state_attr("sensor.home_realfeel_temperature_max_0d", "unit_of_measurement") }}, and it is currently {{ states(indoor_temperature_sensor) }}{{ state_attr(indoor_temperature_sensor, "unit_of_measurement") }} in the {{ area_name(indoor_temperature_sensor) }} and {{ states(outdoor_temperature_sensor) }}{{ state_attr(outdoor_temperature_sensor, "unit_of_measurement") }} outside.
                {% else %}
                clear_notification
                {% endif %}
          - service: input_text.set_value
            data:
              value: >-
                {% if states(window_binary_sensor) == "on" %}
                Close the {{ area_name(indoor_temperature_sensor) }} windows
                {% endif %}
            target:
              entity_id: !input status_title_text
    default:
      - service: input_select.select_option
        data:
          option: any
        target:
          entity_id: !input desired_window_state_input_select
      - event: !input notification_event
        event_data:
          message: clear_notification
      - service: input_text.set_value
        data:
          value: ""
        target:
          entity_id: !input status_title_text
mode: restart
{% endraw %}
```
Let's see how it works.

# Breaking down the automation

## The inputs

The inputs are pretty self-explanatory--just match each up to the actual sensor or template sensor from the earlier sections.

The only one we haven't covered is the "Notification Event". This is the name of a custom event that will fire to trigger the notification. I use a custom event since I want to use my [custom AppDaemon app that only updates uncleared notifications](/2022/07/updating-uncleared-home-assistant-android-notifications-with-appdaemon/). For reference, here's the YAML I use to configure that app:
```yaml
update_tid_notifications_upstairs_front:
  module: update_uncleared_android_notifications
  class: UpdateUnclearedAndroidNotifications
  notification_targets:
    - <redacted device ID 1>
    - <redacted device ID 2>
  notification_data:
    alert_once: true
    sticky: true
    clickAction: "/lovelace-default/cooling"
```
As you can see, it will send notifications to two Android devices, and clicking on the notification will open up the Home Assistant app and take you to the cooling dashboard.

## The triggers

The triggers are also self-explanatory--if any of the sensors changes, we should run the automation.

## The actions

Here's where it gets good. First, we create some variables based on the inputs so we can use them in templates later on.

Second, we initialize that `input_select` that will contain the desired window state.

We then use one giant [choose statement](https://www.home-assistant.io/docs/scripts#choose-a-group-of-actions) to test the two conditions we described at the top of the guide, or run a default sequence if neither of those conditions is true.

For the first condition, the `cooler_outside_binary_sensor` and the `indoors_above_threshold_binary_sensor` must both be on. If so, then we set the desired window state to "on" (meaning "open"). We then fire off our notification event with a message about what the current temperature is, but note that we use templates so that the fields only have the message if the desired window state ("on") doesn't match the current window state ("off"). If it does match, then we clear the notification because we don't need to take any action.

Note that the template for the message does some fancy stuff: it figures out what room's windows to open by using the [`area_name`](https://www.home-assistant.io/docs/configuration/templating/#areas) function, and it gets the units of the sensor by using the [`state_attr`](https://www.home-assistant.io/docs/configuration/templating/#states) function to get the `unit_of_measurement` attribute.

After sending the message, we also set the `status_title_text` to a short description of what we need to do, for use in the Lovelace dashboard.

The second condition proceeds similarly. Finally, the default action (if neither set of conditions is true) just sets the desired window state to `any`, and clears the notification and our status text.

# It's reusable!

Since it's a template, it's easy to re-use this for multiple rooms. (This is useful for me because the back of my house is often cooler than the front of my house.) All I have to do is create new template sensors, and then re-fill in the blueprint.

# Bonus: a dynamic Lovelace card

![Dynamic Lovelace Card]({{ '/assets/images/tid_lovelace_card.png' | relative_url }})

To make a nice card in Lovelace that shows me what the current status is in the front and back of my house (and outside on either side), I installed the [Vertical Stack in Card](https://github.com/ofekashery/vertical-stack-in-card) and [card-mod](https://github.com/thomasloven/lovelace-card-mod) HACS Frontend components.

I only want the card to appear when any of the windows needs to be either open or closed. However, the Lovelace [conditional card](https://www.home-assistant.io/dashboards/conditional/) doesn't support templates, so I first had to drop the following template sensor into `configuration.yaml`:
```yaml
{% raw %}
template:
  - binary_sensor:
        - unique_id: tid_any_upstairs_window_should_change
        state: >
          {{ "on" if ((states("input_select.tid_upstairs_front_desired_window_state") != "any") or
                      (states("input_select.tid_desired_window_state_upstairs_back") != "any"))
          else "off" }}
{% endraw %}
```
As you can see, it's "on" if any of the windows are in a state other than "any."

I also added a group containing all the status messages, so I could iterate over them in the card:
```yaml
group:
  tid_status_titles:
    name: "TID Status Titles"
    entities:
      - input_text.tid_upstairs_back_status_title
      - input_text.tid_upstairs_front_status_title
```

With that setup, here's the YAML for the resulting card:
```yaml
{% raw %}
type: conditional
conditions:
  - entity: binary_sensor.template_tid_any_upstairs_window_should_change
    state: 'on'
card:
  type: custom:vertical-stack-in-card
  title: Upstairs Cooling Status
  card_mod:
    style: |
      .card-header {
        padding-bottom: 0px;
      }
  cards:
    - type: markdown
      content: >-
        {% for status_title in expand("group.tid_status_titles") %}
          {% if states(status_title.entity_id) != "" %}
        ### <ha-icon icon="mdi:alert"></ha-icon>
        *{{states(status_title.entity_id)}}*
          {% endif %}
        {% endfor %}
      card_mod:
        style: |
          ha-markdown.no-header {
            padding-top: 0px !important;
            padding-bottom: 0px !important;
          }
    - type: glance
      entities:
        - entity: sensor.outdoor_front_temperature_sensor_mysensors_0
          name: Front
        - entity: binary_sensor.windows_upstairs_front
          name: Den
          card_mod:
            style: |
              :host {
                --paper-item-icon-color: {{ "initial" if is_state("input_select.tid_upstairs_front_desired_window_state", "any") else ("green" if is_state("binary_sensor.windows_upstairs_front", states("input_select.tid_upstairs_front_desired_window_state")) else "red") }}
              }
        - entity: sensor.aerq_temperature_and_humidity_sensor_v2_0_air_temperature_2
          name: Den
        - entity: sensor.aerq_temperature_and_humidity_sensor_v2_0_air_temperature
          name: Master Bedroom
        - entity: binary_sensor.windows_upstairs_back
          name: Master Bedroom
          card_mod:
            style: |
              :host {
                --paper-item-icon-color: {{ "initial" if is_state("input_select.tid_desired_window_state_upstairs_back", "any") else ("green" if is_state("binary_sensor.windows_upstairs_back", states("input_select.tid_desired_window_state_upstairs_back")) else "red") }}
              }
        - entity: sensor.temp_sensor_outdoor_back_mysensors_0
          name: Back
      state_color: false
      show_name: true
      show_state: true
      show_icon: true
      columns: 6
{% endraw %}
```

There's a lot going on here, so lets break it down.

The first parts are about creating the condition card, which shows a `custom:vertical-stack-in-card` if the condition is true.

The vertical stack has two cards: the first is a markdown card which iterates over the status in the group, and if it's not empty, prints it in bold next to a nice little [alert icon](https://www.home-assistant.io/dashboards/markdown/#icons).

The second card is a [glance card](https://www.home-assistant.io/dashboards/glance/) which is fairly straightforward. The only neat part here is that I use card-mod to color the icon of the windows red if they aren't in their desired state, green if they are in their desired state, and I leave them alone if their desired state is "any." (Apparently the CSS property `--paper-item-icon-color` controls the color.)

There's some other minor CSS modification using card-mod throughout, but that's mostly cosmetic.

# Conclusion

And that's it! I now get notifications when I need to open or close my windows, and my house's temperature is far more comfortable.
