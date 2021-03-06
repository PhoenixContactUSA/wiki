##### Table of Contents  
[Node Structure](#nodeStructure)  
 

<a name="nodeStructure"/>
# Node Structure

| **Key** | **Value** |
| --- | --- |
| $is | "node" |
| $mixin | "readonlyMixin&#124;/users/rick/myMixIn" |
| @city | "San Francisco" |
| point1 | ```{"$is":"point/temperature"}``` |
| point2 | ```{"$is":"point/numeric"}``` |
| addNode | ```{"$is":"action/addNode"}``` |

Configurations
---
Config's names must start with "$". Configuration values affect how systems can work. Any data consumer with the "Write" permission can change configuration settings.
  
- $is can never be changed
  - "$is":"node" reference to the node in /defs/profile/node
* $mixin has multiple ref nodes separated by "|" for predefined configs or attributes
  - Mixins that start with "/" reference the node's path to the current broker
  - Mixin paths like "readonlyMixin" reference "/defs/mixin/readonlyMixin"
  - All config/attribute points from ref node or mixin node can be overwritten, but device or virtual driver doesn't have to react to the change

Attributes
---
Attribute's names must start with "@". Any data consumer with the "Write" permission can change attributes. Empty names or any name with the following characters are prohibited: / \ ? % \* : | " < > .

# Root Nodes of a Broker

1. defs: definition of common structures, things here should be same on all broker defined on [www.iot-dsa.org](http://www.iot-dsa.org), except things in local node

- profile: definitions of node type, contains predefined configurations, attributes, and nodes
  - Location: local, dslink
  - System defined
  - Can implement interfaces that provide more configurations, attributes, and nodes

- interface: Profiles can implement these
  - Location: local, dslink
  - System defined

- mixin: definition of node mixin, contains predefined configurations and attributes
  - Location: local
  - User defined

- type - TODO: elaborate on this
  - Location: local

- error - TODO: elaborate on this
  - Location: local

- settings: broker settings - TODO: elaborate on what settings are available
  - Location: local

- users - TODO: elaborate on user management and device
  - Location: local
  - user structures, can also contain device if it belongs to one user

- connections
  - Location: local
  - DSLink connections

# Every response is a stream
Sample request:
```json
{
  "reqId":2,
  "method":"list",
  "path":"/connections/dslink1"
}
```
- reqId can be a positive number or a unique string, corresponding response must returned with same value
- method can be "set", "remove", "invoke", "list", "alias" , "subscribe", "unsubscribe" and "close"
  - DSA methods are defined in "/defs/method/" i.e. definition node of "list" method: /defs/method/list 

Response:
```json
{
  "reqId":2,
  "update":[
    [
      "$is",
      "node"
    ],
    [
      "$permission",
      "write"
    ],
    [
      "@city",
      "San Francisco"
    ],
    [
      "point1",
      {
        "$is":"temperaturePoint",
        "@name":"Custom Name for Point1"
      }
    ],
    [
      "point2",
      {
        "$is":"numericPoint"
      }
    ]
  ]
}
```
- Every response is a stream with table structure List of Rows
- The structure of columns are defined in /defs/method/{methodName}, except the "invoke" method's column structure are defined in action node, or in the $is or $mixin of the action node.
- Each row can be one of these 2 format
  - a row/list with same number of items as columns structure
  - a map with key:value pairs
    - key can be column name or a rowMeta value
    - rowMeta is always optional
    - when required column is omitted, used the default value defined in column otherwise use null

- reqId must be same value as request
- stream can be omitted, or be the following values
  - close means the stream will no longer have any update, data consumer can close the stream session
  - initialize means the server hasn't finished sending existing/cached data, the values sent thus far are incomplete
  - if stream is omitted, it means cached value are finished, and update can still happen in the future

- update stream data. if stream=close, then update is optional
- if there is no data in cache and no new changes, the stream still need to send a blank update list to tell data consumer the initialize phase is finished 

# List Method

## Request

```json
{
  "reqId":2,
  "method":"list",
  "path":"/connections/dslink1"
}
```
- path is the path to the node, it's a parameter defined in /defs/method/list
definition node data of /defs/method/list :

| Key | Value |
| --- | --- |
| $require | "Read" |
| $params | ```[{"name":"path","type":"path"}]``` |
| $columns | ```[{"name":"name","type":"name"}, {"name":"value","type":"object"}]``` |
| $rowMeta | ```[{"name":"change","type":"enums/listMethodChange","default":"update"}]``` |

## Response
this is a sample response of above requestFrame 1:

```json
{
  "reqId":2,
  "update":[
    [
      "$is",
      "node"
    ],
    [
      "$permission",
      "write"
    ],
    [
      "@city",
      "San Francisco"
    ],
    [
      "point1",
      {
        "$is":"temperaturePoint",
        "@name":"Custom Name for Point1"
      }
    ],
    [
      "point2",
      {
        "$is":"numericPoint"
      }
    ]
  ]
}
```

Frame 2:
```json
{
  "reqId":2,
  "stream":"close",
  "update":[
    [
      "point3",
      {
        "$is":"temperaturePoint"
      }
    ],
    {
      "name":"point2",
      "change":"remove"
    }
  ]
}
```
- during initialize phase, configs are sent first, then attributes, then nodes
  - if exists, $is must be the first row of a node, and $mixin must be the second, properties in $is and $mixin will be merged into the node structure and can be overwritten by other results in the response.

# Invoke Method
sample node data of /connections/dsLink/addMath :

| Key | Value |
| --- | --- |
| $is | "action" |
| $params | ```[{"name":"v1","type":"number"}, {"name":"v2","type":"number"}]``` |
| $columns | ```[{"name":"result","type":"number"}]``` |

## Request

```json
{
  "reqId":3,
  "method":"invoke",
  "path":"/connections/dslink/add",
  "params":{
    "v1":2,
    "v2":1
  }
}
```

## Response

```json
{
  "reqId":3,
  "stream":"closed",
  "update":[
    [
      3
    ]
  ]
}
```

# Subscribe Method

## Request

subscribe and unsubscribe has same request and response structure except the method name.
```json
{
  "reqId":5,
  "method":"subscribe",
  "paths":[
    "/node1/point1",
    "/node1/point2",
    "/node1/point3",
    "/node2/point1",
    "/node2/point2"
  ]
}
```

## Response

subscribe response is always a blank stream, it's just a placeholder for errors.

```json
{
  "reqId":5,
  "stream":"close"
}
```

## Update

```json
{
  "reqId":0,
  "updates":[
    [
      "/node1/point1",
      12,
      "2014-11-27T09:11.000-08:00"
    ],
    {
      "path":"/node1/point2",
      "status":"error",
      "msg":"my error message",
    },
    {
      "path":"/node1/point2",
      "status":"disconnected"
    },
    {
      "path":"/node1/point3",
      "value":10,
      "ts":"2014-11-27T09:11.000-08:00",
      "count":5,
      "sum":75,
      "min":10,
      "max":20
    }
  ]
}
```
status can be either "error" or "disconnected", when it's error, the msg is a optional field for error detail

definition node data of /defs/method/subscribe:

| Key | Value |
| --- | --- |
| $require | "Read" |
| $params | ```[{"name":"paths","type":"list/path"}]``` |
| $columns | ```[{"name":"path","type":"path"}, {"name":"value","type":"object"}, {"name":"ts","type":"time"}]``` |
| $rowMeta | ```[{"name":"status","type":"enums/valueStatus","default":"ok"}, {"name":"count","type":"int","default":1}, {"name":"sum","type":"number"}, {"name":"min","type":"number"}, {"name":"max","type":"number"}]``` |

# Set Method

## Request

set method can be used to set any config, attribute or node value when data consumer has the permission.

```json
{
  "reqId":3,
  "method":"set",
  "path":"/connections/dslink/@myattribute",
  "value":"hello dsa"
}
```

## Response

```json
{
  "reqId":3,
  "stream":"close"
}
```

# Close Method
close a stream, close request has no response

## Request

```json
{
  "reqId":3,
  "method":"close"
}
```

# Requests and Responses
requests are merged in a wrapper JSON object

```json
{
  "sessionId":"9D8F1ACCB",
  "requests":[
    {
      "reqId":2,
      "method":"list",
      "path":"/connections/dslink1"
    },
    {
      "reqId":3,
      "method":"list",
      "path":"/connections/dslink2"
    }
  ]
}
```

- sessionId optional. Unique session id, if omitted, server should automatically create one based on the connection
- requests a list of request methods
responses
```json
{
  "sessionId":"9D8F1ACCB",
  "responses":[
    {
      "reqId":2,
      "update":[
        [
          "$is",
          "node"
        ],
        [
          "$permission",
          "Write"
        ],
        [
          "@city",
          "San Francisco"
        ],
        [
          "point2",
          {
            "$is":"numericPoint"
          }
        ]
      ]
    }
  ]
}
```
- sessionId optional. Unique session id, only needed when request contains sessionId
- responses a list of response objects

# Errors
every stream response can contain error.
```json
{
  "reqId":1,
  "stream":"close",
  "error":{
    "type":"PermissionDenied",
    "phase":"request",
    "path":"/connection/dslink1",
    "msg":"permission denied",
    "detail":"user Steve is not allowed to access data in '/connection/dslink1'"
  }
}
```
- error error object
  - msg required, a short description of the error
  - type optional, a standard error code if the error type is known
    - predefined error structure are in /defs/error

  - phase optional, indicate whether the error happen on request or response, if omitted it means "request" phase
  - path optional, on which path this error happened, can be omitted if it's same path as the path in request.
  - detail a detail message of the error, can be the stack trace or other message
    - when /settings/$errorDetail = false, error won't contain detail message

  - anything else to describe the error