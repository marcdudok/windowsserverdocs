---
title: Develop a solution extension
description: Develop a solution extension Windows Admin Center SDK (Project Honolulu)
ms.technology: manage
ms.topic: article
author: nwashburn-ms
ms.author: niwashbu
ms.date: 06/18/2018
ms.localizationpriority: medium
ms.prod: windows-server-threshold
---

# Develop a solution extension

>Applies To: Windows Admin Center, Windows Admin Center Preview

Solution extensions primarily define a type of connection. Windows Admin Center includes four types of solution extensions by default, one each for Windows Server connections, Windows PC connections, failover cluster connections and hyper-converged cluster connections. When you click on a connection in the Windows Admin Center home screen, the solution extension for the connection's type is loaded and will attempt to connect to the target connection. If the connection is successful, the solution extension's UI will load and display the tool extensions for the solution in the left navigation pane.

If you plan on building management GUI for a new type of connection such as network switches, Linux servers, or even services such as Remote Desktop Services, you'll need to create a new solution extension.

> [!NOTE]
> Not familiar with the different extension types? Learn more about the [extensibility architecture and extension types](understand-extensions.md).

Follow these steps to create a solution extension:

- [Prepare](prepare-development-environment.md) your development environment
    - [Create](prepare-development-environment.md#create-new-extension-with-the-windows-admin-center-cli) an extension with the Windows Admin Center CLI
- Modify the project into a solution extension
- Create a connection provider
- Add your content to the project

## Prepare your development environment

Follow these step-by-step directions to [prepare](prepare-development-environment.md) your development environment and [create](prepare-development-environment.md#create-new-extension-with-the-windows-admin-center-cli) an extension with the Windows Admin Center CLI.
 
## Modify the project into a solution extension

All of the changes required to configure your extension as a Solution are confined to the manifest.json file in {extension root}\src.

Open up the manifest file and find the "entryPoints" array, the first (currently only) entry has an "entryPointType" of "tool". This designates the extension as a tool that will be loaded in the tools menu on a solution. In order to make a solution, we need to change this value to "solution". You will also need to update the "name" and "urlName" properties.

The next property that needs to be changed is "rootNavigationBehavior". This property has two valid values: "connections" or "path". The "connections" behavior is detailed later in the documentation.

### Setting ```path``` as a ```rootNavigationBehavior```

Set the value to "path", and then delete the "requirements" property, and leave the "path" property as an empty string. You have have completed the minimal required configuration to build a solution extension. Save the file, and gulp build -> gulp serve as you would a tool, and then side load the extension into your local Windows Admin Center extension.

A valid manifest entryPoints array looks like this:
```
    "entryPoints": [
        {
          "entryPointType": "solution",
          "name": "main",
          "urlName": "testsln",
          "displayName": "resources:strings:displayName",
          "description": "resources:strings:description",
          "icon": "sme-icon:icon-win-powerShell",
          "path": "",
          "rootNavigationBehavior": "path"
        }
    ],
```

Tools built with this kind of structure will not required connections to load, but won't have node connectivity functionality either.

### Setting ```connections``` as a ```rootNavigationBehavior```

When you set the rootNavigationBehavior property to "connections", you are telling the Windows Admin Center Shell that there will be a connected node (always a server of some sorts) that it should connect to, and verify connection status. With this, there are 2 steps in verifying connection. 1) Windows Admin Center will attempt to make an attempt to log into the node with your credentials (for establishing the remote PowerShell session), and 2) it will execute the PowerShell script you provide to identify if the node is in a connectable state.

A valid solution definition with connections will look like this:

``` json
        {
          "entryPointType": "solution",
          "name": "example",
          "urlName": "solutionexample",
          "displayName": "resources:strings:displayName",
          "description": "resources:strings:description",
          "icon": "sme-icon:icon-win-powerShell",
          "rootNavigationBehavior": "connections",
          "connections": {
            "header": "resources:strings:connectionsListHeader",
            "connectionTypes": [
                "msft.sme.connection-type.example"
                ]
            },
            "tools": {
                "enabled": false,
                "defaultTool": "solution"
            }
        },
```

When the rootNavigationBehavior is set to "connections" you are required to build out the connections definition in the  manifest. This inclued the "header" property (will be used to display in your solution header when a user selects it from the menu), a connectionTypes array (this will specify which connectionTypes are used in the solution. More on that in the connectionProvider documentation.).

### Enabling and disabling the tools menu ###

The final property to talk about in the solution definition is the "tools" property. This will dictate if the tools menu is displayed, as well as the tool that will be loaded. When enabled, Windows Admin Center will render the left hand tools menu. With defaultTool, it is required that you add a tool entry point to the manifest in order to load the appropriate resources. The value of "defaultTool" needs to be the "name" property of the tool as it is defined in the manifest.

## Create a connection provider

Connection Providers play an important role in how Windows Admin Center defines and communicates with connectable objects, or targets. Primarily, a Connection Provider performs actions while a connection is being made, such as ensuring that the target is online and available, and also ensuring that the connecting user has permission to access the target.

By default, Windows Admin Center ships with the following Connection Providers:

* Server
* Windows Client
* Failover Cluster
* HCI Cluster

To create your own custom Connection Provider, follow these steps:

* Add Connection Provider details to ```manifest.json```
* Define Connection Status Provider
* Implement Connection Provider in application layer

### Add Connection Provider details to ```manifest.json```

Now we’ll walk through what you need to know to define a Connection Provider in your project’s manifest.json file.

#### Create entry in ```manifest.json```

The manifest.json file is located in the \src folder and contains, among other things, definitions of entry points into your project. Types of entry points include Tools, Solutions, and Connection Providers. We’ll be defining a Connection Provider.

A sample of a Connection Provider entry in manifest.json is below:

``` json
    {
      "entryPointType": "connectionProvider",
      "name": "addServer",
      "path": "/add",
      "displayName": "resources:strings:addServer_displayName",
      "icon": "sme-icon:icon-win-server",
      "description": "resources:strings:description",
      "connectionType": "msft.sme.connection-type.server",
      "connectionTypeName": "resources:strings:addServer_connectionTypeName",
      "connectionTypeUrlName": "server",
      "connectionTypeDefaultSolution": "msft.sme.server-manager!servers",
      "connectionTypeDefaultTool": "msft.sme.server-manager!overview",
      "connectionStatusProvider": {
        "powerShell": {
          "script": "## Get-My-Status ##\nfunction Get-Status()\n{\n# A function like this would be where logic would exist to identify if a node is connectable.\n$status = @{label = $null; type = 0; details = $null; }\n$caption = \"MyConstCaption\"\n$productType = \"MyProductType\"\n# A result object needs to conform to the following object structure to be interpreted properly by the Windows Admin Center shell.\n$result = @{ status = $status; caption = $caption; productType = $productType; version = $version }\n# DO FANCY LOGIC #\n# Once the logic is complete, the following fields need to be populated:\n$status.label = \"Display Thing\"\n$status.type = 0 # This value needs to conform to the LiveConnectionStatusType enum. >= 3 represents a failure.\n$status.details = \"success stuff\"\nreturn $result}\nGet-Status"
        },
        "displayValueMap": {
          "wmfMissing-label": "resources:strings:addServer_status_wmfMissing_label",
          "wmfMissing-details": "resources:strings:addServer_status_wmfMissing_details",
          "unsupported-label": "resources:strings:addServer_status_unsupported_label",
          "unsupported-details": "resources:strings:addServer_status_unsupported_details"
        }
      }
    },
```

An entry point of type "connnectionProvider" indicates to the Windows Admin Center shell that the item being configured is a provider that will be used by a Solution to validate a connection state. Connection Provider entry points contains a number of important properties, defined below:

| Property | Description |
| -------- | ----------- |
| entryPointType | This is a required property. There are three valid values: "tool", "solution", and "connectionProvider". | 
| name | Identifies the Connection Provider within the scope of a Solution. This value must be unique inside a full Windows Admin Center instance (not just a Solution). |
| path | Represents the URL path for the "Add Connection" UI, if it will be configured by the Solution. This value must map to a route that is configured in app-routing.module.ts file. When the Solution entry point is configured to use the connections rootNavigationBehavior, this route will load the module that is used by the Shell to display the Add Connection UI. More information available in the section on rootNavigationBehavior. |
| displayName | The value entered here is displayed on the right hand side of the shell, below the black Windows Admin Center bar when a user loads a Solution’s connections page. |
| icon | Represents the icon used in the Solutions drop down menu to represent the Solution. |
| description | Enter a short description of the entry point. |
| connectionType | Represents the connection type that the provider will load. The value entered here will also be used in the Solution entry point to specify that the Solution can load those connections. The value entered here will also be used in Tool entry point(s) to indicate that the Tool is compatible with this type. This value entered here will also be used in the connection object that is submitted to the RPC call on the "Add window", in the application layer implementation step. |
| connectionTypeName | Used in the connections table to represent a connection that uses your Connection Provider. This is expected to be the plural name of the type. |
| connectionTypeUrlName | Used in creating the URL to represent the loaded Solution, after Windows Admin Center has connected to an instance. This entry is used after connections, and before the target. In this example, "connectionexample" is where this value appears in the URL: http://localhost:6516/solutionexample/connections/connectionexample/con-fake1.corp.contoso.com |
| connectionTypeDefaultSolution | Represents the default component that should be loaded by the Connection Provider. This value is a combination of: [a] The name of the extension package defined at the top of the manifest; [b] Exclamation point (!); [c] The Solution entry point name.	For a project with name "msft.sme.mySample-extension", and a Solution entry point with name "example", this value would be "msft.sme.solutionExample-extension!example". |
| connectionTypeDefaultTool | Represents the default Tool that should be loaded on a successful connection. This property value is made up of two parts, similar to the connectionTypeDefaultSolution. This value is a combination of: [a] The name of the extension package defined at the top of the manifest; [b] Exclamation point (!); [c] The Tool entry point name for the Tool that should be loaded initially. For a project with name "msft.sme.solutionExample-extension", and a Solution entry point with name "example", this value would be "msft.sme.solutionExample-extension!example". |
| connectionStatusProvider | Please see section "Define Connection Status Provider" |

### Define Connection Status Provider

Connection Status Provider is the mechanism by which a target is validated to be online and available, also ensuring that the connecting user has permission to access the target. There are currently two types of Connection Status Providers:  PowerShell, and RelativeGatewayUrl.

*	PowerShell Connection Status Provider
    * 	Determines if a target is online and accessible with a PowerShell script. The result must be returned in an object with a single property "status", defined below.
*	RelativeGatewayUrl Connection Status Provider
    *	Determines if a target is online and accessible with a rest call. The result must be returned in an object with a single property "status", defined below.

#### Define  ```status```

Connection Status Providers are required to return an object with a single property "status" that conforms to the following format:

``` json
{
    status: {
        label: string;
        type: int;
        details: string;
    }
}
```

Status properties:

* Label
    * A label describing the status return type. Note, values for label can be mapped in runtime. See entry below for mapping values in runtime.

* Type
    * The status return type. Type has the following enumeration values. For any value 2 or above, the platform will not navigate to the connected object, and an error will be displayed in the UI.

Types:

| Value | Description |
| ----- | ----------- |
| 0 | Online |
| 1 | Warning |
| 2 | Unauthorized |
| 3 | Error |
| 4 | Fatal |
| 5 | Unknown |

* Details
    * Additional details describing the status return type.

#### PowerShell Connection Status Provider script

The Connection Status Provider PowerShell script determines if a target is online and accessible with a PowerShell script. The result must be returned in an object with a single property "status". An example script is shown below.

Example PowerShell script:

``` ts
## Get-My-Status ##

function Get-Status()
{
    # A function like this would be where logic would exist to identify if a node is connectable.
    $status = @{label = $null; type = 0; details = $null; }
    $caption = "MyConstCaption"
    $productType = "MyProductType"

    # A result object needs to conform to the following object structure to be interperated properly by the Windows Admin Center shell.
    $result = @{ status = $status; caption = $caption; productType = $productType; version = $version }

    # DO FANCY LOGIC #

    # Once the logic is complete, the following fields need to be populated:
    $status.label = "Display Thing"
    $status.type = 0 # This value needs to conform to the LiveConnectionStatusType enum. >= 3 represents a failure.
    $status.details = "success stuff"

    return $result
}

Get-Status
```

#### Define ```RelativeGatewayUrl``` Connection Status Provider method

The Connection Status Provider RelativeGatewayUrl method calls a rest API to determine if a target is online and accessible. The result must be returned in an object with a single property "status". An example Connection Provider entry in manifest.json of a RelativeGatewayUrl is shown below.

``` json
    {
      "entryPointType": "connectionProvider",
      "name": "addServer",
      "path": "/add/server",
      "displayName": "resources:strings:addServer_displayName",
      "icon": "sme-icon:icon-win-server",
      "description": "resources:strings:description",
      "connectionType": "msft.sme.connection-type.server",
      "connectionTypeName": "resources:strings:addServer_connectionTypeName",
      "connectionTypeUrlName": "server",
      "connectionTypeDefaultSolution": "msft.sme.server-manager!servers",
      "connectionTypeDefaultTool": "msft.sme.server-manager!overview",
      "connectionStatusProvider": {
        "relativeGatewayUrl": "<URL here post /api>",
        "displayValueMap": {
          "wmfMissing-label": "resources:strings:addServer_status_wmfMissing_label",
          "wmfMissing-details": "resources:strings:addServer_status_wmfMissing_details",
          "unsupported-label": "resources:strings:addServer_status_unsupported_label",
          "unsupported-details": "resources:strings:addServer_status_unsupported_details"
        }
      }
    },
```

Notes about using RelativeGatewayUrl:

* "relativeGatewayUrl" specifies where to get the connection status from a gateway URL. This URI is relative from /api. If $connectionName is found in the URL, it will be replaced with the name of the connection.
* All relativeGatewayUrl properties must be executed against the host gateway, which can be accomplished by creating a gateway extension

#### Map values in runtime

The label and details values in the status return object can be formatted at tune time by including keys and values in the "defaultValueMap" property of the provider.

For example, if you add the value below, any time that "defaultConnection_test" showed up as a value for either label or details, Windows Admin Center will automatically replace the key with the configured resource string value.

``` json
    "defaultConnection_test": "resources:strings:addServer_status_defaultConnection_label"
```

### Implement Connection Provider in application layer

Now we’re going to implement the Connection Provider in the application layer, by creating a TypeScript Class that implements OnInit. The class has the following functions:

| Function | Description |
| -------- | ----------- |
| constructor(private appContextService: AppContextService, private route: ActivatedRoute) |  |
| public ngOnInit() |  |
| public onSubmit() | Contains logic to update shell when an add connection attempt is made |
| public onCancel() | Contains logic to update shell when an add connection attempt is canceled |

#### Define ```onSubmit```

onSubmit issues an RPC call back to the app context to notify the shell of an "Add Connection". The basic call uses "updateData" like this:

``` ts
this.appContextService.rpc.updateData(
    EnvironmentModule.nameOfShell,
    '##',
    <RpcUpdateData>{
        results: {
            connections: connections,
            credentials: this.useCredentials ? this.creds : null
        }
    }
);
```

The result is a connection property, which is an array of objects that conform to the following structure:

``` ts

/**
 * The connection attributes class.
 */
export interface ConnectionAttribute {

    /**
     * The id string of this attribute
     */
    id: string;

    /**
     * The value of the attribute. used for attributes that can have variable values such as Operating System
     */
    value?: string | number;
}

/**
 * The connection class.
 */
export interface Connection {

    /**
     * The id of the connection, this is unique per connection
     */
    id: string;

    /**
     * The type of connection
     */
    type: string;

    /**
     * The name of the connection, this is unique per connection type
     */
    name: string;

    /**
     * The property bag of the connection
     */
    properties?: ConnectionProperties;

    /**
     * The ids of attributes identified for this connection
     */
    attributes?: ConnectionAttribute[];

    /**
     * The tags the user(s) have assigned to this connection
     */
    tags?: string[];
}

/**
 * Defines connection type strings known by core
 * Be careful that these strings match what is defined by the manifest of @msft-sme/server-manager
 */
export const connectionTypeConstants = {
    server: 'msft.sme.connection-type.server',
    cluster: 'msft.sme.connection-type.cluster',
    hyperConvergedCluster: 'msft.sme.connection-type.hyper-converged-cluster',
    windowsClient: 'msft.sme.connection-type.windows-client',
    clusterNodesProperty: 'nodes'
};
```

#### Define  ```onCancel```

onCancel cancels an "Add Connection" attempt by passing an empty connections array:

``` ts
this.appContextService.rpc.updateData(EnvironmentModule.nameOfShell, '##', <RpcUpdateData>{ results: { connections: [] } });
```

## Connection Provider example

The full TypeScript class for implementing a connection provider is below. Note that the "connectionType" string matches the "connectionType as defined in the connection provider in manifest.json.

``` ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { AppContextService } from '@msft-sme/shell/angular';
import { Connection, ConnectionUtility } from '@msft-sme/shell/core';
import { EnvironmentModule } from '@msft-sme/shell/dist/core/manifest/environment-modules';
import { RpcUpdateData } from '@msft-sme/shell/dist/core/rpc/rpc-base';
import { Strings } from '../../generated/strings';

@Component({
  selector: 'add-example',
  templateUrl: './add-example.component.html',
  styleUrls: ['./add-example.component.css']
})
export class AddExampleComponent implements OnInit {
  public newConnectionName: string;
  public strings = MsftSme.resourcesStrings<Strings>().SolutionExample;
  private connectionType = 'msft.sme.connection-type.example'; // This needs to match the connectionTypes value used in the manifest.json.
  
  constructor(private appContextService: AppContextService, private route: ActivatedRoute) {
    // TODO:
  }

  public ngOnInit() {
    // TODO
  }

  public onSubmit() {
    let connections: Connection[] = [];

    let connection = <Connection> {
      id: ConnectionUtility.createConnectionId(this.connectionType, this.newConnectionName),
      type: this.connectionType,
      name: this.newConnectionName
    };

    connections.push(connection);

    this.appContextService.rpc.updateData(
      EnvironmentModule.nameOfShell,
      '##',
      <RpcUpdateData> {
        results: {
          connections: connections,
          credentials: null
        }
      }
    );
  }

  public onCancel() {
    this.appContextService.rpc.updateData(
      EnvironmentModule.nameOfShell, '##', <RpcUpdateData>{ results: { connections: [] } });
  }
}

```

## Example projects

Check out these examples available on our [GitHub site](https://github.com/Microsoft/windows-admin-center-sdk/):

* [Developer Tools](https://github.com/Microsoft/windows-admin-center-sdk/tree/master/windows-admin-center-developer-tools) is a sample extension containing a rich collection of sample functionality and tool examples that you can browse and use in your own extension. Developer Tools is a fully functioning extension that can be side-loaded into Windows Admin Center in Developer Mode.

* [Solution Example](https://github.com/Microsoft/windows-admin-center-sdk/tree/master/solution-example) is a sample extension configured as a solution extension. Use this as a guide in building your own solution extension.
