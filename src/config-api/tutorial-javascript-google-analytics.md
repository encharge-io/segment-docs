---
title: 'Creating a Javascript web source and Google Analytics destination'
sidebar: 'Tutorial: Google Analytics Destination'
---

Segment makes it easy to collect data from cloud services (i.e. advertising, helpdesk, and marketing) and send it to many destination services (i.e. CRM services and data warehouses).

A common use case is to send data from your website into Google Analytics to understand you website users.

This tutorial uses the Config API to:

* Create a personal access token to access the api
* Create a JavaScript source
* Create Google Analytics destination
* Update Google Analytics settings

Finally it uses the Tracking API to:

* Send sample events

## Prerequisites

### Google Analytics Website Tracking ID

The Google Analytics integration requires a Google account and website tracking ID. See the Google [Set Up Analytics Tracking doc](https://support.google.com/analytics/answer/1008080?hl=en) for instructions to create an ID.

### Personal Access Token

Programmatic access to the Config API for resources that you own happens through personal access tokens. Personal access tokens belong to Segment users, and can be used to access resources owned by the user who created them.

To set up Segment Protocols through the API you first need to create a personal access token with **full access** to your workspace through the `workspace` scope. Use your Segment account email and password:

```shell
$ USER=me@example.com
$ PASS=<your Segment password>
$ WORKSPACE=<your Segment workspace>

$ curl \
  -X POST \
  -d "{'access_token': {'description': 'my access token', 'scopes': 'workspace', 'workspace_names': ['workspaces/$WORKSPACE']}}" \
  -u "$USER:$PASS" \
  https://platform.segmentapis.com/v1beta/access-tokens
```

Example response:

```json
{
  "name": "access-tokens/5",
  "description": "my access token",
  "scopes": "workspace",
  "create_time": "2018-10-12T22:36:39Z",
  "token": "qiTgISif4zprgBb_5j4hXfp3qhDbxrntWwwOaHgAMr8.gg9ok4Bk7sWlP67rFyXeH3ABBsXyWqNuoXbXZPv1y2g",
  "workspace_names": [
    "workspaces/myworkspace"
  ]
}
```

> Warning: Secret Token
>
> Note that you can not retrive the plain-text `token` later, so you should save it in a secret manager.

Now that you have a personal access token, you can use this token to access the Config API by setting it in the `Authorization` header of your requests, for example:

```shell
$ ACCESS_TOKEN=qiTgISif4zprgBb_5j4hXfp3qhDbxrntWwwOaHgAMr8.gg9ok4Bk7sWlP67rFyXeH3ABBsXyWqNuoXbXZPv1y2g

$ curl \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  https://platform.segmentapis.com/v1beta/workspaces
```

Example response:
```json
{
 "workspaces": [
  {
   "name": "myworkspace",
   "display_name": "My Space",
   "id": "bb296fce9c",
   "create_time": "2012-08-12T15:40:04.406Z"
  }
 ]
}
```

## Create a Javascript Source

Data collection on Segment happens through "event sources". Event sources are created with a type that describes the environment you are sending events from, e.g. `javascript` for a website, `ios` for a mobile app, and `go` for a server. Segment gives each event source a unique "write key" that you configure the Segment SDK with.

Let's create a Javascript event source:

```shell
$ ACCESS_TOKEN=1fUBblCni_qlYlUdBkv16tVrTtxJjv4uWLB2y9NYsUo.INcqwxms0p4OI_4ZeUyUiGFBwXJ7VyHYwtQNLQ3nu-g

$ curl \
  -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -d "{'source': {'name': 'workspaces/$WORKSPACE/sources/js', 'catalog_name': 'catalog/sources/javascript'}}" \
  https://platform.segmentapis.com/v1beta/workspaces/$WORKSPACE/sources
```

<span class="example">Example response:</span>

```json
{
 "slug": "js",
 "name": "workspaces/myworkspace/sources/js",
 "parent": "workspaces/myworkspace",
 "catalog_name": "catalog/sources/javascript",
 "write_keys": [
  "LxKXARX9IuqR9v0gn7y2Fw1iHi0ephbF"
 ],
 "create_time": "2018-09-12T21:15:54.169Z",
 "library_source_config": {}
}
```

> Analytics.js
>
> The Javascript source requires that you add analytics.js to your website to collect data. See the [Quickstart: Analytics.js](https://segment.com/docs/sources/website/analytics.js/quickstart/) guide for full details.

## Create a Google Analytics Destination

On Segment, events are sent in real-time to "streaming destination" services. Destinations are created with the name of the service, e.g. `google-analytics`, and configuration for the services such as the website tracking ID.

Let's create a Google Analytics destination. Note that we are setting the `CLOUD` connection mode that sends data to Google Analytics via Segment's servers. Also note that we are setting the website tracking ID from our Google Analytics account.

```shell
$ curl \
  -X POST \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -d "{
    'destination': {
      'name': 'workspaces/$WORKSPACE/sources/js/destinations/google-analytics',
      'connection_mode': 'CLOUD',
      'config': [
        {
          'value': 'UA-970334309-1',
          'type': 'string',
          'name': 'workspaces/$WORKSPACE/sources/js/destinations/google-analytics/config/trackingId'
        }
      ]
    }
  }" \
  https://platform.segmentapis.com/v1beta/workspaces/$WORKSPACE/sources/js/destinations
```

<span class="example">Example response:</span>

```json
{
 "slug": "google-analytics",
 "name": "workspaces/myworkspace/sources/js/destinations/google-analytics",
 "parent": "workspaces/myworkspace/sources/js",
 "display_name": "Google Analytics",
 "create_time": "2018-09-24T17:08:04.138Z",
 "update_time": "2018-09-24T17:08:04.138Z",
 "enabled": false,
 "connection_mode": "CLOUD",
 "config": [
  {
   "name": "workspaces/myworkspace/sources/js/destinations/google-analytics/config/trackingId",
   "display_name": "Website Tracking ID",
   "value": "UA-970334309-1",
   "type": "string"
  },
  ...
 ]
}
```

## Update Google Analytics Settings

Over time we may want to modify the Google Analytics destination settings. First, the destination was created as disabled by default, so we can use the API to enable it. We can also toggle other settings like "[server side identify](https://segment.com/docs/destinations/google-analytics/#server-side-identify)". Note the required `update_mask` field that explicitly specifies the fields you want to update.

```shell
$ curl -i \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -X PATCH \
  -d "{
    'destination': {
      'enabled': 'true',
      'config': [
        {
          'name': 'workspaces/$WORKSPACE/sources/js/destinations/google-analytics/config/enableServerIdentify',
          'type': 'boolean',
          'value': true
        }
      ]
    },
    'update_mask': {
      'paths': ['destination.enabled', 'destination.config']
    }
  }" \
  https://platform.segmentapis.com/v1beta/workspaces/$WORKSPACE/sources/js/destinations/google-analytics
```

<span class="example">Example response:</span>

```json
{
 "slug": "google-analytics",
 "name": "workspaces/myworkspace/sources/js/destinations/google-analytics",
 "parent": "workspaces/myworkspace/sources/js",
 "display_name": "Google Analytics",
 "create_time": "2018-09-24T17:08:04.138Z",
 "update_time": "2018-09-24T17:16:42.499Z",
 "enabled": true,
 "connection_mode": "CLOUD",
 "config": [
  {
   "name": "workspaces/myworkspace/sources/js/destinations/google-analytics/config/enableServerIdentify",
   "display_name": "Enable Server Side Identify",
   "value": true,
   "type": "boolean"
  },
  {
   "name": "workspaces/myworkspace/sources/js/destinations/google-analytics/config/trackingId",
   "display_name": "Website Tracking ID",
   "value": "UA-970334309-1",
   "type": "string"
  },
  ...
 ]
}
```

See the [Google Analytics Destination](https://segment.com/docs/destinations/google-analytics/) reference for all the settings.

## Send Sample Events

We can also test our source and destination by interacting with the Tracking API from the command line.

Let's send an events to our new source. Note that `userID` is required and the `name` and `url` properties are expected by Google Analytics and other website tools. See the [Page API spec](https://segment.com/docs/sources/server/http/#page) for more details.

```shell
$ WRITE_KEY=LxKXARX9IuqR9v0gn7y2Fw1iHi0ephbF

$ curl https://api.segment.io/v1/page \
  -u $WRITE_KEY: \
  -H 'Content-type: application/json' \
  -d '{
    "userId": "78e56a08-ad10-42b1-88d2-b823623ac875",
    "name": "index",
    "properties": {
      "url": "https://example.com"
    },
    "context": {
      "library": "curl"
    }
  }'
```

To verify your events were sent to the source, open the event debugger in your browser, e.g. https://app.segment.com/myworkspace/sources/js/debugger.