---
title: "Reading flags from a file"
excerpt: ""
---
[block:api-header]
{
  "title": "Overview"
}
[/block]
This topic explains how to run feature flags from a file when you're using a server-side SDK.

If you're performing automated tests or prototyping, you might want to run application code that uses feature flags without connecting to LaunchDarkly. LaunchDarkly SDKs in offline mode return the default value for each flag evaluation. `Default` is not an actual property of the flag. It is the value that you specified as a fallback in your program code when you evaluated the flag.

However, in the server-side SDKs, you can also use files to configure the feature flag state you desire. 

This feature is currently available in the following server-side SDKs: 
* Go (version 4.4.0 and up), 
* Java (4.5.0 and up)
* .NET (5.5.0 and up)
* Ruby (5.4.0 and up)
* Python (6.6.0 and up)
* Node.js (5.7.0 and up)
* PHP (5.3.0 and up)
[block:callout]
{
  "type": "warning",
  "body": "Always configure production environments to receive flag updates from LaunchDarkly. Only use file-based flags in testing and pre-production environments.",
  "title": "Do not use file-based flag values in production environments"
}
[/block]

[block:api-header]
{
  "title": "Creating a flag data file"
}
[/block]
Flag data files can be either JSON or YAML. 

They contain up to three properties: 
* `flags`: Feature flag definitions. These can contain all the same kinds of rules and targets that you can define in a LaunchDarkly feature flag, which allows the flag to produce different values for different users.
* `flagValues`: Simplified feature flags that specify only a value, and produce the same value for all users.
* `segments`: User segment definitions. You will only use this property if you have feature flags that use segments.
[block:callout]
{
  "type": "warning",
  "body": "In some of the SDKs, YAML support requires an additional dependency. YAML is not available in PHP.",
  "title": "YAML files have limitations"
}
[/block]
The format of the data in `flags` and `segments` is defined by the LaunchDarkly application and is subject to change. Rather than trying to construct these objects yourself, it's simpler to request existing flags directly from the LaunchDarkly server in JSON format and use this output as the starting point for your file. 

Get the flags from `https://app.launchdarkly.com/sdk/latest-all`. Pass your SDK key in the `Authorization` header. 

For instance, in Linux, you could use this command:
[block:code]
{
  "codes": [
    {
      "code": "curl -H \"Authorization: EXAMPLE_SDK_KEY\" https://app.launchdarkly.com/sdk/latest-all >flagdata.json\n\n",
      "language": "shell"
    }
  ]
}
[/block]
The output will look something like this, with many more properties:
[block:code]
{
  "codes": [
    {
      "code": "{\n  \"flags\": {\n    \"flag-key-1\": {\n      \"key\": \"flag-key-1\",\n        \"on\": true,\n        \"variations\": [ \"a\", \"b\" ]\n      }\n  },\n  \"segments\": {\n    \"segment-key-1\": {\n      \"key\": \"segment-key-1\",\n      \"includes\": [ \"user-key-1\" ]\n    }\n  }\n}\n",
      "language": "json",
      "name": "JSON Output"
    }
  ]
}
[/block]
Data in this format allows the SDK to exactly duplicate all the kinds of flag behavior LaunchDarkly supports. However, in many cases you do not need this complexity. You may want to simply set specific flag keys to specific values. 

For that, you can use a much simpler format:
[block:code]
{
  "codes": [
    {
      "code": "{\n  \"flagValues\": {\n    \"my-string-flag-key\": \"value-1\",\n    \"my-boolean-flag-key\": true,\n    \"my-integer-flag-key\": 3\n  }\n}\n",
      "language": "json"
    }
  ]
}
[/block]
Or, in YAML:
[block:code]
{
  "codes": [
    {
      "code": "flagValues:\n  my-string-flag-key: \"value-1\"\n  my-boolean-flag-key: true\n  my-integer-flag-key: 3",
      "language": "yaml"
    }
  ]
}
[/block]
It is also possible to specify both `flags` and `flagValues`, if you want some flags to have simple values and others to have complex behavior. However, it is an error to use the same flag key or segment key more than once, either in a single file or across multiple files.
[block:api-header]
{
  "title": "Configuring the client to use a file"
}
[/block]
In all of the SDKs that have this feature, you can specify either a single file or multiple files. In all of them except PHP, you can also specify whether the SDK should reload the file data if it detects that you have modified a file. For instance, you could verify that your application behaves correctly when a flag changes.

The examples below show how to configure the client to use two data files called `file1.json` and `file2.json`. The client assumes these two files are in the current working directory, but you can specify any relative or absolute file path. It also enables automatic reloading, if supported.
[block:code]
{
  "codes": [
    {
      "code": "import (\n    ld \"gopkg.in/launchdarkly/go-client.v4\"\n    \"gopkg.in/launchdarkly/go-client.v4/ldfiledata\"\n    \"gopkg.in/launchdarkly/go-client.v4/ldfilewatch\"\n)\n\nfileSource, err := ldfiledata.NewFileDataSourceFactory(\n    ldfiledata.FilePaths(\"file1.json\", \"file2.json\"),\n    ldfiledata.UseReloader(ldfilewatch.WatchFiles))\n\nconfig := ld.DefaultConfig\nconfig.UpdateProcessorFactory = fileSource\nconfig.SendEvents = false\n\nclient := ld.MakeCustomClient(\"sdk key\", config, 5*time.Second)",
      "language": "go"
    },
    {
      "code": "import com.launchdarkly.client.*;\nimport com.launchdarkly.client.files.*;\n\nFileDataSourceFactory fileSource = FileComponents.fileDataSource()\n    .filePaths(\"file1.json\", \"file2.json\")\n    .autoUpdate(true);\n\nLDConfig config = new LDConfig.Builder()\n    .updateProcessorFactory(fileSource)\n    .sendEvents(false)\n    .build();\n\nLDClient client = new LDClient(\"sdk key\", config);",
      "language": "java"
    },
    {
      "code": "using LaunchDarkly.Client;\nusing LaunchDarkly.Client.Files;\n\nvar fileSource = FileComponents.FileDataSource()\n    .WithFilePaths(\"file1.json\", \"file2.json\")\n    .WithAutoUpdate(true);\n\nvar config = Configuration.Default(\"sdk key\")\n    .WithUpdateProcessorFactory(fileSource)\n    .WithEventProcessorFactory(Components.NullEventProcessor);\n\nvar client = new LDClient(config);\n\n// Note that in the .NET SDK, if you want to use YAML files instead of JSON\n// you must provide a YAML parser; see FileDataSourceFactory.WithParser.",
      "language": "csharp",
      "name": ".NET"
    },
    {
      "code": "require 'ldclient-rb'\n\nfactory = LaunchDarkly::FileDataSource.factory(\n  paths: [ \"file1.json\", \"file2.json\" ],\n  auto_update: true\n)\n\nconfig = LaunchDarkly::Config.new(\n  update_processor_factory: factory,\n  send_events: false\n)\n\nclient = LaunchDarkly::Client.new(\"sdk key\", config)\n\n# Note that in the Ruby SDK, automatic reloading uses a somewhat inefficient\n# file-polling mechanism unless you install the \"listen\" gem.",
      "language": "ruby"
    },
    {
      "code": "import ldclient\nfrom ldclient.config import Config\nfrom ldclient.file_data_source import FileDataSource\n\nfactory = FileDataSource.factory(paths=[\"file1.json\", \"file2.json\"],\n    auto_update=True)\n\nconfig = Config(update_processor_class=factory, send_events=False)\n\nldclient.set_config(config)\nldclient.set_sdk_key(\"sdk key\")\nclient = ldclient.get()\n\n# Note that in the Python SDK, if you want to use YAML files instead of JSON\n# you must install the \"pyyaml\" package. Also, automatic reloading uses a\n# somewhat inefficient file-polling mechanism unless you install the \"watchdog\"\n# package.",
      "language": "python"
    },
    {
      "code": "const LaunchDarkly = require('ldclient-node');\n\nconst dataSource = LaunchDarkly.FileDataSource({\n  paths: [ \"file1.json\", \"file2.json\" ]\n});\n\nconst config  = {\n  updateProcessor: dataSource\n};\n\nconst client = LaunchDarkly.init(sdkKey, config);\n",
      "language": "javascript",
      "name": "Node.js"
    },
    {
      "code": "// Note that automatic reloading is not supported in PHP, since normally in PHP\n// the entire in-memory application state is recreated for each request.\n\n$fr = LaunchDarkly\\Integrations\\Files::featureRequester([\n    'file1.json',\n    'file2.json'\n]);\n$client = new LaunchDarkly\\LDClient(\"your_sdk_key\", [\n    'feature_requester' => $fr,\n    'send_events' => false\n]);",
      "language": "php"
    }
  ]
}
[/block]
Note that if you do not want your code to connect to LaunchDarkly at all, you will normally want to prevent the SDK from trying to send analytics events, so the option for disabling events is also included in these examples. Also, since there will be no LaunchDarkly connection, you do not have to use a valid SDK key; the SDK key parameter is still required but can be any string.

If any of the specified files is missing or invalid, the SDK does not use any of the file data and logs an error message instead.