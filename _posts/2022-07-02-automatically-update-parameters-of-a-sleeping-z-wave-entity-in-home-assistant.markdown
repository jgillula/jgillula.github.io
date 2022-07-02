---
layout: post
title:  Automatically Update Parameters of a Sleeping Z-Wave Device in Home Assistant
date:   2022-07-02
tags: homeassistant zwave automations scripts
description: "Sleeping Z-Wave devices don't check for parameter updates, so we use a bunch of scripts and automations to make them."
---


# Goals

I have a couple of Z-Wave temperature sensors in my house that are battery operated and so spend most of their time sleeping. When they're sleeping they will occasionally send new readings to Home Assistant, but won't check to see if their configuration parameters have been changed. However, I want to change their parameters over the course of the day: I want them to be more sensitive to temperature changes during the day, and less sensitive at night (in order to save battery power and get readings when I really care about them).

# Approach

The general idea is to:

1. Use the awesome [custom scheduler component](https://github.com/nielsfaber/scheduler-component) and [custom scheduler card](https://github.com/nielsfaber/scheduler-card) for Home Assistant to set a schedule that will call a script with certain parameters at certain times.
2. Use that script to iterate over our chosen Z-Wave devices, setting their parameters and adding them to a group to indicate that they need to be updated.
3. Use an automation that triggers when the Z-Wave devices report a measurement to ping them, thereby causing them to check in and retrieve their new parameters.

Let's see how this works in action. We'll work backwards to chain the components together.

# Step 0. Setup

In order to know what Z-Wave devices we'll want to update, we'll create two [old-style groups](https://www.home-assistant.io/integrations/group#old-style-groups) in Home Assistant's `configuration.yaml`. The first indicates the target Z-Wave devices we'll want to update, and the second will list which target devices currently need to actually be updated.

```yaml
group:
  aerq_temperature_sensors:
    name: "Target Aerq Temperature Sensors"
    entities:
      - sensor.aerq_temperature_and_humidity_sensor_v2_0_air_temperature
      - sensor.aerq_temperature_and_humidity_sensor_v2_0_air_temperature_2
  aerq_sensors_needing_update:
    name: "Target Aerq Sensors Needing Update"
    entities: []
```

Note that the group needing update is empty. That's because we'll populate it automatically as part of the schedule, so that devices are only in that group when they do indeed need to be updated.

# Step 1. The Automation

Let's build up the automation. The goal is to have it trigger when a target device that needs to be updated reports a new sensor reading, so we can ping it and make it read its new parameters from Home Assistant.

## The Trigger

```yaml
{% raw %}
trigger:
  - platform: zwave_js.event
    entity_id:
      - sensor.aerq_temperature_and_humidity_sensor_v2_0_air_temperature
      - sensor.aerq_temperature_and_humidity_sensor_v2_0_air_temperature_2
    event_source: node
    event: metadata updated
{% endraw %}
```
The trigger is a [`zwave_js.event`](https://www.home-assistant.io/integrations/zwave_js#zwave_jsevent) trigger. It requires an `event_source` and an `event` to describe what sort of Z-Wave JS events we want to trigger on. We'll use the [`metadata updated`](https://zwave-js.github.io/node-zwave-js/#/api/node?id=quotmetadata-updatedquot) `event` type, since that seems to fire any time one of the sensors sends in a new reading. Its `event_source` type is `node`.

We also only want to trigger when one of our target Z-Wave devices reports a new sensor reading (and not every single Z-Wave device), so we list them in the `entity_id` field. Unfortunately Z-Wave JS doesn't support a group here, so we have to manually list all the target Z-Wave sensors here (duplicating what we listed for the target group in step 0).

## The Condition

```yaml
{% raw %}
condition:
  - condition: template
    value_template: >-
      {% set ns = namespace(update_needed=false) %}
      {% for entity_id in state_attr('group.aerq_sensors_needing_update', 'entity_id') %}
        {% set ns.update_needed = ns.update_needed or (entity_id in device_entities(trigger.device_id)) %}
      {% endfor %}
      {{ ns.update_needed }}
{% endraw %}
```

The trigger will trigger any time a target device fires a `metadata updated` event--not just those that need to be updated. Thus we need to use a condition to ensure that the triggering device (whose `device_id` is automatically populated into `trigger.device_id` by Home Assistant) does indeed need to be updated.

To do so, we create a variable called `update_needed`, initially `false`, which we'll set to true if we detect that the triggering device needs an update. (We have to use a `namespace` because Jinja doesn't persist variables into or out of for loops, unless they're part of a namespace. Don't ask me why.)

We then loop through all the `entity_id`s in the group that lists the sensors that need to be updated. If the `entity_id` is in the list of entities for the triggering device (given by the `device_entities()` function), `ns.update_needed` will be set to `true`.

Finally, we just print the value of `ns.update_needed` to evaluate whether the condition is true or false.

## The Actions

### Updating the group of sensors to update

```yaml
{% raw %}
  - service: group.set
    data:
      object_id: aerq_sensors_needing_update
      entities: >
        {% set ns = namespace(item_to_update="") %}
        
        {% for entity_id in state_attr('group.aerq_sensors_needing_update', 'entity_id') %}
          {% if entity_id in device_entities(trigger.device_id) %}
            {% set ns.item_to_update = entity_id %}
          {% endif %}
        {% endfor %}

        {{ state_attr('group.aerq_sensors_needing_update', 'entity_id') | reject("eq", ns.item_to_update) | list }}
{% endraw %}
```
The first action is to update the group of sensors needing an update, removing the sensor we're updating, so we don't bother to update it again. We do so by calling the [`group.set`](https://www.home-assistant.io/integrations/group#services) service, which allows us to set the members of a group. To figure out what members we want, we first have to identify the entity that we're currently updating.

We do that by looping through the entities in the group of sensors needing an update, and if we find an `entity_id` that matches one associated with the triggering device, we store it in `ns.item_to_update`.

We then use `state_attr` to get the list of entities currently in the group, apply the [`reject`](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.reject) filter to reject the item we're currently updating, and use the [`list`](https://jinja.palletsprojects.com/en/3.1.x/templates/#jinja-filters.list) filter to turn the remaining items into a list.

### Tell the device when to check in next
```yaml
{% raw %}
  - service: zwave_js.set_config_parameter
    target:
      device_id: '{{ trigger.device_id }}'
    data:
      parameter: 4
      value: >
        {% if (states.switch | selectattr("attributes.entities", "eq", ["script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule"]) | list | length) >= 1 and
              (states.switch | selectattr("attributes.entities", "eq", ["script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule"]) | map(attribute='state') | list | first) == "on" %}
          {{ [65535, ([0, ((as_timestamp((states.switch | selectattr("attributes.entities", "eq",["script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule"]) | map(attribute='attributes.next_trigger') | first)) - as_timestamp(now())) | round(0, 'floor') | int)+60] | max)] | min }} 
        {% else %}
          {{ states("input_number.default_aerq_periodic_report_interval") | int }}
        {% endif %}
{% endraw %}
```

The sensors I'm working with report at configurable intervals if their values have changed, but you can also set them to [report no matter what at configurable intervals](https://aeotec.freshdesk.com/support/solutions/articles/6000227918#:~:text=Parameter%204%3A%20Periodic%20Reports.), too. This is convenient, because based on the schedule I can calculate how long it should be until the next time I'll want to change their parameters, and make sure they check in then (even if their readings haven't changed between now and then). This action does that calculation and sets the parameter. Note that this calculation has to be performed here in the automation because the duration isn't an absolute time--it's how long to wait until the next forced check-in. Of course, we can't know that duration ahead of time: it depends on when the sensor actually checks in.

We use the [`zwave_js.set_config_parameter`](https://www.home-assistant.io/integrations/zwave_js#service-zwave_jsset_config_parameter) service to set parameter 4 (which happens to be the number of the parameter we want).

To get the duration, we first use an if statement to see if there's a [custom schedule entity](https://github.com/nielsfaber/scheduler-component#scheduler-entities) whose target is the script we use to mark which devices need to be updated. Schedule entities are stored as switches, so we filter on all switches and select one whose `entities` attribute is that script. We then ensure it's actually enabled by making sure it's state is "on".

Then, assuming we found the schedule entity, we extract its next trigger time, subtract `now()` from it to get the number of seconds until then, round it to the nearest second, convert it to an int, at 60 seconds to give us a buffer to ensure the Home Assistant script will be run before the device checks in, and then ensure the value is between 0 and 65535.

Otherwise, if we didn't find the entity, we just use a default reporting interval that's set by an `input_number` helper.

### Pinging the device

```yaml
{% raw %}
  - service: zwave_js.ping
    target:
      device_id: '{{ trigger.device_id }}'
{% endraw %}
```
Finally, we ping the device. This causes the device to check in and get its new parameters from Home Assistant.

## The full automation

Putting it all together, the automation is as follows:
```yaml
{% raw %}
alias: 'Aerq Sensors: Update Config Parameters Based on Schedule'
description: ''
trigger:
  - platform: zwave_js.event
    entity_id:
      - sensor.aerq_temperature_and_humidity_sensor_v2_0_air_temperature
      - sensor.aerq_temperature_and_humidity_sensor_v2_0_air_temperature_2
    event_source: node
    event: metadata updated
condition:
  - condition: template
    value_template: >-
      {% set ns = namespace(update_needed=false) %} {% for entity_id in state_attr('group.aerq_sensors_needing_update', 'entity_id') %}
        {% set ns.update_needed = ns.update_needed or (entity_id in device_entities(trigger.device_id)) %}
      {% endfor %}  {{ ns.update_needed }}
action:
  - service: group.set
    data:
      object_id: aerq_sensors_needing_update
      entities: >
        {% set ns = namespace(item_to_update="") %}

        {% for entity in state_attr('group.aerq_sensors_needing_update', 'entity_id') %}
          {% if entity in device_entities(trigger.device_id) %}
            {% set ns.item_to_update = entity %}
          {% endif %}
        {% endfor %}

        {{ state_attr('group.aerq_sensors_needing_update', 'entity_id') | reject("eq", ns.item_to_update) | list }}
  - service: zwave_js.set_config_parameter
    target:
      device_id: '{{ trigger.device_id }}'
    data:
      parameter: 4
      value: >
        {% if (states.switch | selectattr("attributes.entities", "eq", ["script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule"]) | list | length) >= 1 and
              (states.switch | selectattr("attributes.entities", "eq", ["script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule"]) | map(attribute='state') | list | first) == "on" %}
          {{ [65535, ([0, ((as_timestamp((states.switch | selectattr("attributes.entities", "eq",["script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule"]) | map(attribute='attributes.next_trigger') | first)) - as_timestamp(now())) | round(0, 'floor') | int)+60] | max)] | min }} 
        {% else %}
          {{ states("input_number.default_aerq_periodic_report_interval") | int }}
        {% endif %}
  - service: zwave_js.ping
    target:
      device_id: '{{ trigger.device_id }}'
trace:
  stored_traces: 20
mode: single
{% endraw %}
```
Note that we've set `stored_traces` to 20 to facilitate debugging (so we can see lots of traces, since many of them will not actually go through the full script because they'll be stopped early by the test condition).

# 2. The Script

```yaml
{% raw %}
alias: Temperature Sensors (aerq) Set All Config Parameters Based on Schedule
sequence:
  - repeat:
      count: '{{ expand("group.aerq_temperature_sensors") | length() }}'
      sequence:
        - variables:
            entity_id: >-
              {{ expand("group.aerq_temperature_sensors")[repeat.index -
              1].entity_id }}
        - service: zwave_js.set_config_parameter
          target:
            entity_id: '{{ entity_id }}'
          data:
            parameter: 1
            value: '{{ temperature_change_threshold_degrees*10 | int }}'
        - service: zwave_js.set_config_parameter
          target:
            entity_id: '{{ entity_id }}'
          data:
            parameter: 3
            value: '{{ check_interval_minutes | int }}'
        - service: group.set
          data:
            object_id: aerq_sensors_needing_update
            add_entities:
              - '{{ entity_id }}'
mode: single
fields:
  temperature_change_threshold_degrees:
    name: Temperature Change Report Threshold
    description: (degrees)
    required: true
    default: 0.1
    selector:
      number:
        min: 0.1
        max: 10.0
        step: 0.1
        mode: box
  check_interval_minutes:
    name: Threshold Check Interval
    description: (minutes)
    required: true
    default: 2
    selector:
      number:
        min: 1
        max: 255
        step: 1
        mode: box
{% endraw %}
```

The script is fairly straightforward. First, we assume that `temperature_change_threshold_degrees` and `check_interval_minutes` have been passed as parameters to the script. These are the parameters we want to update for our Z-Wave devices, and more information about them can be found [here](https://aeotec.freshdesk.com/support/solutions/articles/6000227918#:~:text=Configuration%20Parameters.).

Then we iterate over each of the `entity_id`s in `group.aerq_temperature_sensors`, and for each of them do the following.

1. Get the `entity_id` and store it as a variable we can use in templates for the rest of the sequence.
2. Use the [`zwave_js.set_config_parameter`](https://www.home-assistant.io/integrations/zwave_js#service-zwave_jsset_config_parameter) service to set parameter 1, which is the minimum change in temperature we want the sensor to report. We use the value passed in by `temperature_change_threshold_degrees` multiplied by 10 since `temperature_change_threshold_degrees` is in degrees, and the parameter the device expects should be in 10ths of a degree. (E.g. a value or 3.8 degrees should correspond to 38 10ths of a degree.)
3. Use the [`zwave_js.set_config_parameter`](https://www.home-assistant.io/integrations/zwave_js#service-zwave_jsset_config_parameter) service to set parameter 3, which is the interval in minutes at which the sensor should check for a temperature change. We use the value passed in by `check_interval_minutes`.
4. Use the [`group.set`](https://www.home-assistant.io/integrations/group#services) service to add the given `entity_id` to the group of target devices that need to be updated.

Thus the script will set the parameters we want (as passed into it) for the devices we want to update, and add them to the group to indicate they need to be updated.

# 3. The Scheduler

Last but not least we need to add the [customer scheduler card](https://github.com/nielsfaber/scheduler-card). The scheduler should call the script, passing in the parameters we want.

We assume that the components are already installed. If not, you'll have to [install HACS](https://hacs.xyz/docs/setup/download) and then install the custom scheduler [component](https://github.com/nielsfaber/scheduler-component#installation)/[card](https://github.com/nielsfaber/scheduler-card#installation).

We simply change Lovelace into editing mode, click the "Add Card" button, choose the "Custom: Scheduler Card" (all the way at the bottom), click "Show Code Editor" to switch to code editing mode, and drop in the following YAML:
```yaml
{% raw %}
type: custom:scheduler-card
include:
  - script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule
discover_existing: false
customize:
  script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule:
    actions:
      - service: script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule
        name: Change Parameters
        icon: mdi:cog
        variables:
          temperature_change_threshold_degrees:
            name: Temperature Change Report Threshold
            min: 0.1
            max: 2
            step: 0.1
            unit: °F
          check_interval_minutes:
            name: Threshold Check Interval
            min: 1
            max: 60
            step: 1
            unit: ' minutes'
time_step: 5
title: false
display_options:
  primary_info: default
  secondary_info:
    - relative-time
    - time
    - additional-tasks
  icon: action
{% endraw %}
```
The `type` is self-explanatory, and the `include` just indicates the script we want to call as part of our schedule. `discover_existing`, `title`, and `display_options` are all configuration options for the UX; more info can be found [here](https://github.com/nielsfaber/scheduler-card#options).

The `customize` section is used to tell the scheduler card what service to call (in this case `script.temperature_sensors_aerq_set_all_config_parameters_based_on_schedule`), and what parameters to send it. Specifying those parameters in the `variables` section tells the custom scheduler card how to display them and let me select them from the user interface.

# Conclusion

And that's it! We now have a scheduler that periodically runs a script which primes an automation that will trigger and cause our target Z-Wave devices to check in with Home Assistant and read their new configuration parameters!
