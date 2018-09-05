# GoogleIoTCore #

This library allows your agent code to work with [Google IoT Core](https://cloud.google.com/iot-core/).

This version of the library supports the following functionality:
- [Registering a device](https://cloud.google.com/iot/docs/how-tos/getting-started#device_registration) in Google IoT Core (optional feature).
- [Connecting and disconnecting](https://cloud.google.com/iot/docs/how-tos/mqtt-bridge) to/from Google IoT Core.
- [Publishing telemetry events](https://cloud.google.com/iot/docs/how-tos/mqtt-bridge#publishing_telemetry_events) to Google IoT Core.
- [Receiving configurations](https://cloud.google.com/iot/docs/how-tos/config/configuring-devices) from Google IoT Core.
- [Reporting a device state](https://cloud.google.com/iot/docs/how-tos/config/getting-state#reporting_device_state) to Google IoT Core.

The library provides an opportunity to work via [different transports](https://cloud.google.com/iot/docs/concepts/protocols), but currently supports [MQTT transport](https://cloud.google.com/iot/docs/how-tos/mqtt-bridge) only.

**To add this library to your project, add** `#require "GoogleIoTCore.agent.lib.nut:1.0.0"` **to the top of your agent code**.

## Library Usage ##

The library API specification is described [here](#api-specification).

The [working examples](./examples) are provided together with the library and described [here](./examples/README.md).

Below sections explain the main ideas and design of the library and provide the usage recommendations.

### General Concepts ??? ###

TODO - some general explanation / how to (maybe part from Production Flow)

### Production Flow ###

TODO - update

Each Google IoT Core device should have its own public and private RSA keys (Elliptic Curve is also supported by Google IoT Core, but is not supported by imp).

Public key is saved inside the Google IoT Core platform, so it is used only when registering a device in Google IoT Core. Private key is used on the client's side when connecting to the Google IoT Core servers.

Currently EI-agent can't generate a pair of RSA keys, so these keys should be generated by some provisioner and delivered to the agent.
Then you have two options:
1. This library can register a device in Google IoT Core and set a public key for it. Both public and private keys should be delivered to the agent. Also, the agent should have credentials for Google IoT Core device registry.
1. Right after generation of keys the provisioner can register a device in Google IoT Core and set a public key for it. Only private key should be delivered to the agent.

Google [recommends](https://cloud.google.com/iot/docs/concepts/device-security#provisioning_credentials) the second way for production purposes.

### Automatic JWT Token Refreshing ###

[JWT token](https://cloud.google.com/iot/docs/how-tos/credentials/jwts) has an expiration time, so it should be updated before expiration to prevent the client from unexpected disconnection. This feature is enabled by default and can be disabled by setting the `tokenAutoRefresh` [client's option](#options-1) to `False`.

For MQTT Transport the client implements the following token updating algorithm:
1. Using a timer wakes up when the current JWT token is near to expiration.
1. Waits for all current MQTT operations to be finished.
1. Calculates a new JWT token.
1. Disconnects from the MQTT broker.
1. Connects to the MQTT broker again using the new JWT token as an MQTT client's password.
1. Subscribes to the topics which were subscribed to before the reconnection.
1. Sets the timer for the new JWT token expiration.

The library does these operations automatically and invisibly. All the API calls, made at the time of updating, are scheduled in a queue and processed right after the token updating algorithm is successfuly finished. If the token update fails, the onDisconnected() callback is called (if the callback has been set).

### Pending Requests ###

All requests to a remote server are made asynchronously, so several operations can be processed concurrently. But only limited number of pending operations of the same type is allowed. This number can be changed in [the client's options](#options-1). If you exceed this number, the `GOOGLE_IOT_CORE_ERROR_OP_NOT_ALLOWED_NOW` error will be returned in response to your call.

### Errors Processing ###

TODO

## API Specification ##

### GoogleIoTCore.MqttTransport Class ###

#### Constructor: GoogleIoTCore.MqttTransport(*[options]*) ####

This method returns a new GoogleIoTCore.MqttTransport instance.

| Parameter | Data Type | Required? | Description |
| --- | --- | --- | --- |
| [*options*](#options) | Table | Optional | Key-value table with the transport's settings. |

##### Options #####

These settings affect the transport's behavior and the operations. Every setting is optional and has a default.

| Key (String) | Value Type | Default | Description |
| --- | --- | --- | --- |
| "url" | String | `ssl://mqtt.googleapis.com:8883` | MQTT broker URL formatted as `ssl://<hostname>:<port>`. |
| "qos" | Integer | 0 | MQTT QoS. [Google IoT Core supports QoS 0 and 1 only](https://cloud.google.com/iot/docs/how-tos/mqtt-bridge#quality_of_service_qos). |
| "keepAlive" | Integer | 60 | Keep-alive MQTT parameter, in seconds. For more information, see [here](https://cloud.google.com/iot/docs/how-tos/mqtt-bridge#keep-alive). |

Note, Google IoT Core does not support the `retain` MQTT flag.

### GoogleIoTCore.Client Class ###

#### Constructor: GoogleIoTCore.Client(*projectId, cloudRegion, registryId, deviceId, privateKey[, onConnected[, onDisconnected[, options]]]*) ####

This method returns a new GoogleIoTCore.Client instance.

| Parameter | Data Type | Required? | Description |
| --- | --- | --- | --- |
| *projectId* | String | Yes | [Project ID](https://cloud.google.com/iot/docs/requirements#permitted_characters_and_size_requirements). |
| *cloudRegion* | String | Yes | [Cloud region](https://cloud.google.com/iot/docs/requirements#cloud_regions). |
| *registryId* | String | Yes | [Registry ID](https://cloud.google.com/iot/docs/requirements#permitted_characters_and_size_requirements). |
| *deviceId* | String | Yes | [Device ID](https://cloud.google.com/iot/docs/requirements#permitted_characters_and_size_requirements). |
| *privateKey* | String | Yes | [Private key](https://cloud.google.com/iot/docs/how-tos/credentials/keys). |
| [*onConnected*](#callback-onconnectederror) | Function | Optional | Callback called every time the client is connected. |
| [*onDisconnected*](#callback-ondisconnectederror) | Function | Optional | Callback called every time the client is disconnected. |
| [*options*](#options-1) | Table | Optional | Key-value table with additional settings. |

##### Callback: onConnected(*error*) #####

This callback is called every time the client is connected.

This is a good place to call the [enableCfgReceiving()](#enablecfgreceivingonreceive-ondone) method, if this functionality is needed.

| Parameter | Data Type | Description |
| --- | --- | --- |
| *error* | Integer | `0` if the connection is successful, an [error code](#error-codes) otherwise. |

##### Callback: onDisconnected(*error*) #####

This callback is called every time the client is disconnected.

| Parameter | Data Type | Description |
| --- | --- | --- |
| *error* | Integer | `0` if the disconnection was caused by the disconnect() method, an [error code](#error-codes) which explains a reason of the disconnection otherwise. |

##### Options #####

These additional settings affect the client's behavior and the operations. Every setting is optional and has a default.

| Key (String) | Value Type | Default | Description |
| --- | --- | --- | --- |
| "maxPendingSetStateRequests" | Integer | 3 | Maximum amount of pending [Set State operations](#reportstatestate-onreported). |
| "maxPendingPublishTelemetryRequests" | Integer | 3 | Maximum amount of pending [Publish Telemetry operations](#publishdata-subfolder-onPublished). |
| "tokenTTL" | Integer | 3600 | [JWT token's time to live](https://cloud.google.com/iot/docs/how-tos/credentials/jwts#required_claims), in seconds. |
| "tokenAutoRefresh" | Boolean | True | Enable [Automatic JWT Token Refreshing](#automatic-jwt-token-refreshing). |

```squirrel
#require "GoogleIoTCore.agent.lib.nut:1.0.0"
```

#### setOnConnected(*callback*) ####

This method sets [*onConnected*](#callback-onconnectederror) callback. The method returns nothing.

#### setOnDisconnected(*callback*) ####

This method sets [*onDisconnected*](#callback-ondisconnectederror) callback. The method returns nothing.

#### setPrivateKey(privateKey) ####

This method sets [Private key](https://cloud.google.com/iot/docs/how-tos/credentials/keys). The method returns nothing.

#### register(*iss, secret, publicKey[, onRegistered[, name[, keyFormat]]]*) ####

This complementary method registers the device in Google IoT Core.

It makes the minimal required registration - only one private-public key pair, without expiration setting, is registered.

First, the method attempts to find already existing device with the device ID specified in the client’s constructor and compare that device’s public key with the key passed in. And then:
- If no device found, the method tries to register the new one.
- Else if a device is found and the keys are identical, the method succeeds, assuming the device is already registered.
- Otherwise, the method returns the `GOOGLE_IOT_CORE_ERROR_ALREADY_REGISTERED` error. (TODO - Not confusing?)

**If you are going to use this method, add** `#require "OAuth2.agent.lib.nut:2.0.0"` **to the top of your agent code**.

The method returns nothing. A result of the operation may be obtained via the [*onRegistered*](#callback-onregisterederror) callback if specified in this method.

| Parameter | Data Type | Required? | Description |
| --- | --- | --- | --- |
| *iss* | String  | Yes | [JWT issuer](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#jwt-auth). TODO - haven't found a better link |
| *secret* | String  | Yes | [JWT sign secret key](https://developers.google.com/identity/protocols/OAuth2ServiceAccount#jwt-auth). TODO - haven't found a better link |
| *publicKey* | String  | Yes | [Public key](https://cloud.google.com/iot/docs/how-tos/credentials/keys) for the device. It must correspond to the private key set for the client. |
| *[onRegistered](#callback-onregisterederror)* | Function  | Optional | Callback called when the operation is completed or an error occurs. |
| *name* | String | Optional | [Device name](https://cloud.google.com/iot/docs/reference/cloudiot/rest/v1/projects.locations.registries.devices#resource-device). |
| *keyFormat* | String | Optional | [Public key format](https://cloud.google.com/iot/docs/reference/cloudiot/rest/v1/projects.locations.registries.devices#publickeyformat). If not specified or `null`, `RSA_X509_PEM` is applied. |

##### Callback: onRegistered(*error*) ######

This callback is called when the device is registered.

| Parameter | Data Type | Description |
| --- | --- | --- |
| *[error](#error-codes)* | Integer | `0` if the operation is completed successfully, an [error code](#error-codes) otherwise. |

```squirrel
```

#### connect(*[transport]*) ####

This method opens a connection to Google IoT Core.

If already connected, the [*onConnected*](#callback-onconnectederror) callback will be called with the `GOOGLE_IOT_CORE_ERROR_ALREADY_CONNECTED` error.

Default transport is [GoogleIoTCore.MqttTransport](#googleiotcoremqtttransport-class) with default configuration.

Google IoT Core supports only one connection per device.

The method returns nothing. A result of the operation may be obtained via the [*onConnected*](#callback-onconnectederror) callback specified in the client's constructor or set by calling [setOnConnected()](#setonconnectedcallback) method.

| Parameter | Data Type | Required? | Description |
| --- | --- | --- | --- |
| *transport* | GoogleIoTCore.\*Transport  | Optional | Instance of GoogleIoTCore.\*Transport class. |

```squirrel
```

#### disconnect() ####

This method closes the connection to Google IoT Core. Does nothing if the connection is already closed.

The method returns nothing. When the disconnection is completed the [*onDisconnected*](#callback-ondisconnectederror) callback is called, if specified in the client's constructor or set by calling [setOnDisconnected()](#setondisconnectedcallback) method.

#### isConnected() ####

This method checks if the client is connected to Google IoT Core.

The method returns Boolean: `true` if the client is connected, `false` otherwise.

#### publish(*data[, subfolder[, onPublished]]*) ####

This method [publishes a telemetry event to Google IoT Core](https://cloud.google.com/iot/docs/how-tos/mqtt-bridge#publishing_telemetry_events).

The method returns nothing. A result of the operation may be obtained via the [*onPublished*](#callback-onpublisheddata-error) callback if specified in this method.

| Parameter | Data Type | Required? | Description |
| --- | --- | --- | --- |
| *data* | String or Blob  | Yes | Application specific data. Application can use the [Serializer](https://developer.electricimp.com/libraries/utilities/serializer) library to convert Squirrel objects to Blobs. |
| *subfolder* | String  | Optional | The subfolder can be used as an event category or classification. For more information, see [here](https://cloud.google.com/iot/docs/how-tos/mqtt-bridge#publishing_telemetry_events_to_separate_pubsub_topics). |
| *[onPublished](#callback-onpublisheddata-error)* | Function  | Optional | Callback called when the operation is completed or an error occurs. |

##### Callback: onPublished(*data, error*) ######

This callback is called when the data is considered as published or an error occurs.

| Parameter | Data Type | Description |
| --- | --- | --- |
| *data* | String or Blob | The original *data* passed in to the [publish()](#publishdata-subfolder-onpublished) method. |
| *[error](#error-codes)* | Integer | `0` if the operation is completed successfully, an [error code](#error-codes) otherwise. |

TODO - data is really String or Blob? or Blob always?

```squirrel
```

#### enableCfgReceiving(*onReceive[, onDone]*) ####

This method enables/disables [configuration receiving from Google IoT Core](https://cloud.google.com/iot/docs/how-tos/config/configuring-devices).

Disabled by default and after every successful [connect()](#connecttransport) method call. 

To enable the feature, specify the [*onReceive*](#callback-onreceiveconfiguration) callback. To disable the feature, specify `null` as that callback.

The method returns nothing. A result of the operation may be obtained via the [*onDone*](#callback-ondoneerror) callback if specified in this method.

| Parameter | Data Type | Required? | Description |
| --- | --- | --- | --- |
| *onReceive* | Function  | Yes | [Callback](#callback-onreceiveconfiguration) called every time a configuration is received from Google IoT Core. `null` disables the feature. |
| *[onDone](#callback-ondoneerror)* | Function  | Optional | [Callback](#callback-ondoneerror) called when the operation is completed or an error occurs. |

##### Callback: onReceive(*configuration*) #####

This callback is called every time [a configuration](https://cloud.google.com/iot/docs/concepts/devices#device_configuration) is received.

| Parameter | Data Type | Description |
| --- | --- | --- |
| *configuration* | Blob(TODO: check!) | [Configuration](https://cloud.google.com/iot/docs/concepts/devices#device_configuration). An arbitrary user-defined blob. Application can use the [Serializer](https://developer.electricimp.com/libraries/utilities/serializer) library to convert Blobs to Squirrel objects. |

##### Callback: onDone(*error*) #####

This callback is called when the method is completed.

| Parameter | Data Type | Description |
| --- | --- | --- |
| *[error](#error-codes)* | Integer | `0` if the operation is completed successfully, an [error code](#error-codes) otherwise. |


```squirrel
```

#### reportState(*state[, onReported]*) ####

This method [reports a device state to Google IoT Core](https://cloud.google.com/iot/docs/how-tos/config/getting-state#reporting_device_state).

The method returns nothing. A result of the operation may be obtained via the [*onReported*](#callback-onreportedstate-error) callback if specified in this method.

| Parameter | Data Type | Required? | Description |
| --- | --- | --- | --- |
| *state* | String or Blob  | Yes | [Device state](https://cloud.google.com/iot/docs/concepts/devices#device_state). Application specific data. Application can use the [Serializer](https://developer.electricimp.com/libraries/utilities/serializer) library to convert Squirrel objects to Blobs. |
| *[onReported](#callback-onreportedstate-error)* | Function  | Optional | Callback called when the operation is completed or an error occurs. |

##### Callback: onReported(*state, error*) #####

This callback is called when the state is considered as reported or an error occurs.

| Parameter | Data Type | Description |
| --- | --- | --- |
| *state* | String or Blob | The original *state* passed in to the [reportState()](#reportstatestate-onreported) method. |
| *[error](#error-codes)* | Integer | `0` if the operation is completed successfully, an [error code](#error-codes) otherwise. |

TODO - state is really String or Blob? or Blob always?

```squirrel
```

#### setDebug(*value*) ####

This method enables (*value* is `true`) or disables (*value* is `false`) the client debug output (including error logging). It is disabled by default. The method returns nothing.

### Error Codes ###

An *Integer* error code which specifies a concrete error (if any) happened during an operation.

| Error Code | Error Name | Description |
| --- | --- | --- |
| -99..0 and 128 | - | [MQTT-specific](https://developer.electricimp.com/api/mqtt) errors. |
| 1000 | GOOGLE_IOT_CORE_ERROR_NOT_CONNECTED | The client is not connected. |
| 1001 | GOOGLE_IOT_CORE_ERROR_ALREADY_CONNECTED | The client is already connected. |
| 1002 | GOOGLE_IOT_CORE_ERROR_OP_NOT_ALLOWED_NOW | The operation is not allowed now. E.g. the same operation is already in process. |
| 1003 | GOOGLE_IOT_CORE_ERROR_TOKEN_REFRESHING | An error occured while [refreshing the token](#automatic-jwt-token-refreshing). This error code can only be passed in to the [*onDisconnected*](#callback-ondisconnectederror) callback. |
| 1004 | GOOGLE_IOT_CORE_ERROR_ALREADY_REGISTERED | Another device is already registered with the same Device ID. |
| 1010 | GOOGLE_IOT_CORE_ERROR_GENERAL | General error. |

## Examples ##

Working examples are provided in the [examples](./examples) directory and described [here](./examples/README.md).

## Testing ##

Tests for the library are provided in the [tests](./tests) directory and described [here](./tests/README.md).

## License ##

The GoogleIoTCore library is licensed under the [MIT License](./LICENSE).
