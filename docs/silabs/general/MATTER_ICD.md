# Matter Intermittently Connected Devices (ICD)

Matter introduces the concept of Intermittently Connected Devices (ICD) in the SDK and in the specification.
An Intermittently Connected Device is the Matter representation of a device that is not always reachable.
This covers battery-powered devices that disable their underlying hardware when in a low-power mode or devices that can be disconnected from the network, like a phone app.

This page focuses on features designed to improve the performance and reliability of battery-powered devices and their different configuration options.

## ICD Configurations

The ICD feature-set offers two types of configurations : cluster configurations and subscription configurations.
The cluster configurations are exposed through the ICD Manager Cluster interface.
The subscription configurations are exposed through build arguments and public APIs of the Matter SDK.

### ICD Management Cluster

The ICD Management Cluster enables configuration of the ICD’s behavior.
It is required for an ICD to have this cluster enabled on endpoint 0 to be certifiable.

#### Configuration Attributes

The ICD Management Cluster exposes three configuration attributes.
These configurations are independent from the underlying transport configurations.

| Attribute | Type | Constraints | Description |
|-|-|-|-|
| IdleModeInterval      | uint32 | 1 to 64800   | Maximum interval in seconds or milliseconds the server can stay in idle mode |
| ActiveModeInterval    | uint32 | all          | minimum interval in milliseconds the server will stay in active mode |
| ActiveModeThreshold   | uint16 | desc         | minimum amount of time in milliseconds the server typically will stay active after network activity when in active mode |

These configurations can be changed two different ways.

To change them within a build command
```shell
./scripts/examples/gn_silabs_example.sh ./examples/light-switch-app/silabs ./out/light-switch-app_ICD BRD4187C --icd sl_idle_mode_interval_ms=5000 sl_active_mode_interval_ms=0 sl_active_mode_threshold_ms=500
```
These options can also be change by setting them to a default value in the projects openthread.gni file. See examples/lock-app/silabs/openthread.gni for an example on how they can be configured.
```cpp
// ICD Matter Configuration flags
sl_idle_mode_interval_s = 600       //10min Idle Mode Interval
sl_active_mode_interval_ms = 10000  //10s Active Mode Interval
sl_active_mode_threshold_ms = 1000  //1s Active Mode Threshold
```
The second way of changing the configuration is to set these defines in the projects ChipProjectConfig.h.
```cpp
/**
 * @def CHIP_CONFIG_ICD_IDLE_MODE_INTERVAL_SEC
 *
 * @brief Default value for the ICD Management cluster IdleModeInterval attribute, in seconds
 */
#define CHIP_CONFIG_ICD_IDLE_MODE_INTERVAL_SEC 5


/**
 * @def CHIP_CONFIG_ICD_ACTIVE_MODE_INTERVAL_MS
 *
 * @brief Default value for the ICD Management cluster ActiveModeInterval attribute, in milliseconds
 */
#define CHIP_CONFIG_ICD_ACTIVE_MODE_INTERVAL_MS 0

/**
 * @def CHIP_CONFIG_ICD_ACTIVE_MODE_THRESHOLD_MS
 *
 * @brief Default value for the ICD Management cluster ActiveModeThreshold attribute, in milliseconds
 */
#define CHIP_CONFIG_ICD_ACTIVE_MODE_THRESHOLD_MS 500
```
Using the build arguments either in the build command or in the .gni file is the preferred method.

#### ICD Check-In Protocol Use-Case

The ICD Check-In Protocol use case is used by ICDs to maintain a known relationship in case subscriptions with clients are lost.
This includes how a client shares a Check-In token (symmetric key) with the ICD, when Check-In messages are sent and how the Check-In Protocol requirements are respected.

The Check-In Protocol is a fail-safe mechanism which allows an ICD to notify a registered client that it is available for communication when all subscriptions between the client and ICD are lost.
A subscription can be lost for several reasons, such as:

* The ICD might not have full RAM retention when it is in an idle state.
* When the ICD is powered off to change the battery.
* Power or network outage causing the connection between the client and the ICD to be interrupted.
* The client is unavailable for any reason (e.g. during a software update or hosted on a mobile device that is sometimes out-of-home).

The Check-In message is sessionless and relies on a shared secret that has been given to the ICD during the registration of the client using the ICD Management cluster.
For more information on the ICD Check-In Protocol use-case, see the associated specification section.

#### User Active Mode Trigger

Since ICDs are not immediately responsive, they require a means to render them available for communication within user initiated use cases.
Some of the user initiated use cases are:

* Opening a new commissioning window to add another administrator.
* Reconfiguration of an existing fabric (e.g. IPKs, NOC rotation, ACL changes).
* Reconfiguration of cluster functionality (e.g. ICD Management, Bindings, Groups, Scenes).
* Removal of a device from a fabric.
* Changes to the device's settings.

To enable these user initiated use cases, ICDs need to provide a way for a user to put them in active mode and render them responsive.
The User Active Mode Trigger feature in the ICD Management cluster indicates whether a particular device implements an active mode trigger.

### Subscription Configurations

#### Subscription Maximum Interval Negotiation

The subscription mechanism is used by ecosystems and controllers to receive attribute change updates and liveness checks.
The maximum interval of a subscription request is what defines the frequency at which a device will send a liveness check if there are no attribute changes.

Within the subscription request / response model, a device has the opportunity to decide the maximum interval at which it will send its liveness check (Empty Report Update). The device can set a maximum interval within this range if and only if it is an ICD:

```shell
MinIntervalRequested ≤ MaxInterval ≤ MAX(IdleModeInterval, MaxIntervalRequested)
```

The following table shows the subscribe response fields.
| Action Field | Type | Description |
|-|-|-|
| SubscriptionId | uint32 | identifies the subscription |
| MaxInterval | uint16 | the final maximum interval for the subscription in seconds |

The Matter SDK provides a default implementation that allows an ICD to negotiate its MaxInterval.
The goal of the algorithm is to set the MaxInterval to the IdleModeInterval.

```cpp
#if CHIP_CONFIG_ENABLE_ICD_SERVER

    // Default behavior for ICDs where the wanted MaxInterval for a subscription is the IdleModeInterval
    // defined in the ICD Management Cluster.
    // Behavior can be changed with the OnSubscriptionRequested function defined in the application callbacks

    // Default Behavior Steps :
    // If MinInterval > IdleModeInterval, try to set the MaxInterval to the first interval of IdleModeIntervals above the
    // MinInterval.
    // If the next interval is greater than the MaxIntervalCeiling, use the MaxIntervalCeiling.
    // Otherwise, use IdleModeInterval as MaxInterval

    // GetPublisherSelectedIntervalLimit() returns the IdleModeInterval if the device is an ICD
    uint32_t decidedMaxInterval = GetPublisherSelectedIntervalLimit();

    // Check if the PublisherSelectedIntervalLimit is 0. If so, set decidedMaxInterval to MaxIntervalCeiling
    if (decidedMaxInterval == 0)
    {
        decidedMaxInterval = mMaxInterval;
    }

    // If requestedMinInterval is greater than the IdleTimeInterval, select next active up time as max interval
    if (mMinIntervalFloorSeconds > decidedMaxInterval)
    {
        uint16_t ratio = mMinIntervalFloorSeconds / static_cast<uint16_t>(decidedMaxInterval);
        if (mMinIntervalFloorSeconds % decidedMaxInterval)
        {
            ratio++;
        }

        decidedMaxInterval *= ratio;
    }

    // Verify that decidedMaxInterval is an acceptable value (overflow)
    if (decidedMaxInterval > System::Clock::Seconds16::max().count())
    {
        decidedMaxInterval = System::Clock::Seconds16::max().count();
    }

    // Verify that the decidedMaxInterval respects MAX(GetPublisherSelectedIntervalLimit(), MaxIntervalCeiling)
    uint16_t maximumMaxInterval = std::max(GetPublisherSelectedIntervalLimit(), mMaxInterval);
    if (decidedMaxInterval > maximumMaxInterval)
    {
        decidedMaxInterval = maximumMaxInterval;
    }

    // Set max interval of the subscription
    mMaxInterval = static_cast<uint16_t>(decidedMaxInterval);

#endif // CHIP_CONFIG_ENABLE_ICD_SERVER
```

If the default implementation does not fit within the use-case,
an implementation can override the default implementation.
The first step is to implement the `ApplicationCallback` class from the `ReadHandler.h` header.

```cpp
/*
 * A callback used to interact with the application.
 */
class ApplicationCallback
{
public:
    virtual ~ApplicationCallback() = default;
    /*
     * Called right after a SubscribeRequest has been parsed and processed. This notifies an interested application
     * of a subscription that is about to be established. It also provides an avenue for altering the parameters of the
     * subscription (specifically, the min/max negotiated intervals) or even outright rejecting the subscription for
     * application-specific reasons.
     *
     * TODO: Need a new IM status code to convey application-rejected subscribes. Currently, a Failure IM status code is sent
     * back to the subscriber, which isn't sufficient.
     *
     * To reject the subscription, a CHIP_ERROR code that is not equivalent to CHIP_NO_ERROR should be returned.
     *
     * More information about the set of paths associated with this subscription can be retrieved by calling the appropriate
     * Get* methods below.
     *
     * aReadHandler:            Reference to the ReadHandler associated with the subscription.
     * aSecureSession:          A reference to the underlying secure session associated with the subscription.
     *
     */
    virtual CHIP_ERROR OnSubscriptionRequested(ReadHandler & aReadHandler, Transport::SecureSession & aSecureSession)
    {
        return CHIP_NO_ERROR;
    }
    /*
     * Called after a subscription has been fully established.
     */
    virtual void OnSubscriptionEstablished(ReadHandler & aReadHandler){};
    /*
     * Called right before a subscription is about to get terminated. This is only called on subscriptions that were terminated
     * after they had been fully established (and therefore had called OnSubscriptionEstablished).
     * OnSubscriptionEstablishment().
     */
    virtual void OnSubscriptionTerminated(ReadHandler & aReadHandler){};
 };
```

The second step is registering the callback object to the Interaction Model Engine.

```cpp
// Register ICD subscription callback to match subscription max intervals to its idle time interval
chip::app::InteractionModelEngine::GetInstance()->RegisterReadHandlerAppCallback(&mICDSubscriptionHandler);
```

#### Persistent Subscriptions

Persistent subscriptions were added to Matter as a means to ensure that an ICD can re-establish its subscription and by extension its secure session to a subscriber in the event of a power cycle.
When a device accepts a subscription request, it will persist the subscription.
When the device reboots, it will try to re-establish its subscription with the subscriber.
If the subscription is torn down during normal operations or if the re-establishment fails,
the subscription will be deleted.

Persistent subscriptions are enabled by default on all Silicon Labs sample applications.

#### Subscription Timeout Resumption

Matter also provides a retry mechanism for devices to try to re-establish a lost subscription with a client. This feature should not be used on an ICD since it can significantly reduce battery life. This functionality can be disabled by adding

`chip_subscription_timeout_resumption = false`

#### Subscription Synchronization

To avoid forcing an ICD to become active multiple times, the Matter SDK allows an ICD to synchronize its subscription reporting and send all the reports at the same time. The mecansim syncrhonizes the maximum interval of the all subscription to only require the ICD to become active one. This functionality can be enabled by adding

`sl_use_subscription_synching = true`

For further details on Matter ICD's operating on OpenThread, visit [Matter Intermittently Connected Devices over OpenThread](../thread/OT_SLEEPY_END_DEVICE.md).
And for Matter ICD's operating via WiFi, visit [Matter Intermittently Connected Devices over WiFi](../wifi/WIFI_SLEEPY_END_DEVICE.md).


## ICD Device Types

Matter introduces two types of ICDs.

- Short Idle Time ICDs
- Long Idle Time ICDs

### Short Idle Time ICDs

Short Idle Time ICDs are battery powered devices that can always be reached by clients.
This means that their polling intervals are small enough to guarantee that a message sent from a client will be able to reach the ICD without any synchronization.
A door lock, for example, is typicaly a short idle time ICD because it needs to be able to receive commands from clients at any given time.
These devices are usually not the initiators in the communication flow.

#### Requirements

This section lists the requirements that Short Idle Time ICDs must respect to be certifiable.

1. The ICD Management Cluster must be present on the Root Endpoint (0) with mandatory attributes.
2. The transport slow poll configuration must be smaller or equal to 15s.
This requirement is not enforced in Matter 1.3 since LIT ICD are not certifiable.
Once LIT ICD officially launch, this will be a mandatory requirement.

Support of the ICD Check-In Protocol use-case and the user active mode trigger is optional for SIT ICDs.

#### Configurations

These are recommended configurations based on the state of the current implementation of SIT ICDs.
The recommended configurations are likely to change with the Matter 1.4 release.

```cpp
// ICD Default configurations
chip_enable_icd_server = true
chip_subscription_timeout_resumption = false
sl_use_subscription_synching = true

// ICD Matter Configuration flags
sl_idle_mode_duration_s = 600  // 10min Idle Mode Duration
sl_active_mode_duration_ms = 10000  // 10s Active Mode Duration
sl_active_mode_threshold_ms = 1000  // 1s Active Mode Threshold

// Openthread Configuration flags
sl_ot_idle_interval_ms = 5000  // 5s Idle Intervals
sl_ot_active_interval_ms = 500  // 500ms Active Intervals
```
> **Note**: Wi-Fi polling configuration are dictated by the Access Point and cannot be changed at the Matter level.

### Long Idle ICDs

Long Idle Time ICDs are battery powered devices that require synchronization between the client and the ICD for communication to succeed.
A sensor device is an example of a device that are typically a long idle time ICD.

Long Idle Time ICDs are ready for integration in the Matter 1.3 release. The core feature-set for ICDs has been implemented through the `ICDManager`.
LIT ICDs should be certifiable with the Matter 1.4 release. 
Splitting the two milestones in different releases is to allow more in depth interoperability testing to validate the proposed feature-set achieves it's power consumption and usability goals.

#### Requirements

This section lists the requirements that Long Idle Time ICDs must respect to be certifiable.

1. The ICD Management Cluster must be present on the Root Endpoint (0) with mandatory attributes.
2. The `LITS` (Long Idle Time Support) feature map must be set to 1.
All required features, attributes and commands required by this feature map must also be present.
2. The `CIP` (Check-In Protocol support) feature map must be set to 1.
All required attributes and commands required by this feature map must also be present.
3. The `UAT` (User Active Mode Trigger support) feature map must be set to 1.
All required attributes and commands required by this feature map must also be present.
4. The `ActiveModeThreshold` cannot be lower than 5 seconds.


#### Configurations

These are recommended configurations based on the state of the current implementation of LIT ICDs.
The recommended configurations are likely to change with the Matter 1.4 release.

```cpp
// ICD Default configurations
chip_enable_icd_server = true
chip_subscription_timeout_resumption = false
sl_use_subscription_synching = true
icd_enforce_sit_slow_poll_limit = true
chip_icd_report_on_active_mode = true
chip_enable_icd_lit = true

# Openthread Configuration flags
sl_ot_idle_interval_ms = 3600000  // 60mins Idle Polling Interval
sl_ot_active_interval_ms = 1000  // 1s Active Polling Interval

# ICD Matter Configuration flags
sl_idle_mode_duration_s = 3600  // 60min Idle Mode Duration
sl_active_mode_duration_ms = 0  // 0 Active Mode Duration
sl_active_mode_threshold_ms = 5000  // 5s Active Mode Threshold
```