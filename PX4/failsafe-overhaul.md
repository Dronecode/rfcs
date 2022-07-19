- Feature Name: failsafe_overhaul
- Start Date: 2022-07-19
- RFC PR: [dronecode/rfc#0000](https://github.com/dronecode/rfc/pull/0000)

# Summary
[summary]: #summary

The current implementation of failsafe actions in PX4 is very rigid and limited.
It is impossible to combine failsafe conditions, trigger multiple actions, or set progressively more aggressive actions (currently hardcoded for battery voltage monitoring).
It is also very difficult to extend the failsafe system to respond to user specific conditions because it is tightly tied into `commander`. Generalizing failsafe configuration to a standard framework with variable conditions and actions allows more flexibility. Additionally, more conditions and actions should be added.

There is a fixed set of failsafe “conditions” each with an associated action. This code is mostly handled in the commander module. This leaves limited room to implement more complex/realistic failsafe scenarios, including combining conditions, and progressively more aggressive actions.

# Motivation
[motivation]: #motivation

The current failsafe system lacks features that my team needed and I believe can be useful to other people, such as:
* Progressive failsafes. Battery monitoring has a warning, failsafe, and emergency level hardcoded. But this feature would be very useful for other situations, such as telemetry loss for 30 seconds vs 5 minutes.
* Combining failsafe actions. The action taken when something fails can depend on a combination of conditions. If RC is lost and telemetry still exists or vice versa, it might be safe to continue the mission - or RTL, but if RC is lost and telemetry also becomes lost, the drone is not within communication and it may be best to land. This is a simple example, but a feature like this allows an operator to define precisely the best course of action for any combination of failures.
* Multiple actions.
* Additional actions and triggers, including sending messages over MAVLink to the companion computer or over CAN bus. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Failsafe condition
A failsafe condition defines the parameters for considering a "failure". This includes an event, such as:
* telemetry loss
* geofence violation
* battery voltage
* peripheral failure notified over CAN
The condition usually includes an associated parameter, such as a battery voltage threshold or telemetry loss timeout.

## Failsafe Action
A failsafe action is something that is *done* when one or more failsafe conditions are triggered:
* switching flight mode
* notifying the user
* notifying companion computer over MAVLink  
* notifying rest of system over CAN

## Failsafe Profile
A failsafe profile associates one or more conditions with an action. When **all** associated conditions are met, the action is triggered.
Profiles have an associated priority so that stricter profiles have precedence when conditions overlap.

TODO: "profile" is probably not the best term for this. Ideas welcome.

## User Interface
Two ways of exposing configuration to the user are proposed:

1. Text file configuration
For example:
```
# 15% battery warning
p 10
c BATT 15%
a RTL

# 5% battery warning
p 10
c BATT 5%
a LAND

# if both telem and rc are lost, rtl
p 10
c TLM 30
c RC 3
a RTL

# but if only telem is lost, switch to hold mode
p 20 # lower priority - check the above one first
c TLM 30
a HOLD

# and if telem/RC lost for 10 minutes, terminate
p 1 # highest priority
c TLM 600
c RC 600
a TERM

# traffic avoidance
p 10
c ADSB
a COMPANION # ask the companion computer what to do about it
```

2. Parameter configuration
* FS_PROFILE_N
* FS_P{i}_COND_N
* FS_P{i}\_COND{j}_{TYPE, THRESHOLD}
* FS_P{i}_ACTION

The file method is easier to implement and less clunky looking, but is more static - requires file editing to change, no nice UI, no runtime modification.
The parameter method (if possible to implement) is more difficult to work with manually but easier to automate or create a UI for.

## Migration Guidance
Existing failsafe can be emulated with the new system by defining a set of default profiles, each with one condition and one action.

Battery failsafe would have three associated profiles, one for each of the tiers (warning, failsafe, emergency).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

* Migrate failsafe checking from `commander` into a `failsafe` workqueue task
* Maintain a small amount of state per profile, for instance for keeping track of data loss timeout
* Receive events over uORB - either a message that states something went wrong, or lack of messages = timeout
* Integrate the events interface for "warnings"
* Implement other actions, such as a MAVLink message to the companion computer.
* Unsure how to implement priorities best. Overlapping profiles should have a priority but this shouldn't prevent multiple profiles from being active? Perhaps same priority = both can be triggered.

# Drawbacks
[drawbacks]: #drawbacks

Increased complexity of failsafe configuration for the user. Implementation is also pending some refining, but that aside, I don't see any technical drawbacks.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

I believe this implementation is best because:
* It generalizes failsafes to whatever the user needs, rather than hardcoding specific use cases
* Standardizes how failsafes are considered, currently each failsafe triggered is implemented specifically and UX is inconsistent.

Having a priority based system with strict profiles removes the need for if-else logic or any other control flow, simplifying the implementation significantly.

# Prior art
[prior-art]: #prior-art

N/A.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Most importantly, what the configuration for the user will look like.
* How best to implement priorities and choose between overlapping profiles.

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, this allows us to do a lot more for failure detection and handling in an autonomous setting. Communicating with the companion computer or other CAN peripherals allows more complex logic when triggering failsafes and allows us to better respond to specific/complex failures gracefully instead of the current "all or nothing" approach. This improves safety and reliability in an autonomous setting where things will inevitably go wrong and we want to take the best course of action.
