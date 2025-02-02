# Port Agent

## [Documentation](https://docs.getport.io/create-self-service-experiences/setup-backend/port-execution-agent/)

## Control the payload of your self-service actions

Some of the 3rd party applications that you may want to integrate with may not accept the raw payload incoming from Port's
self-service actions. The Port agent allows you to control the payload that is sent to every 3rd party application.

You can alter the requests sent to your third-party application by providing a payload mapping config file when deploying the 
Port-agent container.

### Control the payload mapping

The payload mapping file is a JSON file that specifies how to transform the request sent to the Port agent to the
request that is sent to the third-party application.

The payload mapping file is mounted to the Port agent as a volume. The path to the payload mapping file is set in the
`CONTROL_THE_PAYLOAD_CONFIG_PATH` environment variable. By default, the Port agent will look for the payload mapping
file
at `~/control_the_payload_config.json`.

The payload mapping file is a json file that contains a list of mappings. Each mapping contains the request fields that
will be overridden and sent to the 3rd party application.

You can see examples showing how to deploy the Port agent with different mapping configurations for various common use cases below.

Each of the mapping fields can be constructed by JQ expressions. The JQ expression will be evaluated against the
original payload that is sent to the port agent from Port and the result will be sent to the 3rd party application.

Here is the mapping file schema:

```
[ # Can have multiple mappings. Will use the first one it will find with enabled = True (Allows you to apply mapping over multiple actions at once)
  {
      "enabled": bool || JQ,
      "url": JQ, # Optional. default is the incoming url from port
      "method": JQ, # Optional. default is POST. Should return one of the following string values POST / PUT / DELETE / GET 
      "headers": dict[str, JQ], # Optional. default is {}
      "body": ".body", # Optional. default is the whole payload incoming from Port.
      "query": dict[str, JQ] # Optional. default is {},
      "report" { # Optional. Used to report the run status back to Port right after the request is sent to the 3rd party application
        "status": JQ, # Optional. Should return the wanted runs status
        "link": JQ, # Optional. Should return the wanted link or a list of links
        "summary": JQ, # Optional. Should return the wanted summary
        "externalRunId": JQ # Optional. Should return the wanted external run id
      }
  }
]
```

**The body can be partially constructed by json as follows:**

```json
{
  "body": {
    "key": 2,
    "key2": {
      "key3": ".im.a.jq.expression",
      "key4": "\"im a string\""
    }
  }
}
```

### The incoming message to base your mapping on

<details>
<summary>Example for incoming event</summary>

```json
{
  "action": "action_identifier",
  "resourceType": "run",
  "status": "TRIGGERED",
  "trigger": {
    "by": {
      "orgId": "org_XXX",
      "userId": "auth0|XXXXX",
      "user": {
        "email": "executor@mail.com",
        "firstName": "user",
        "lastName": "userLastName",
        "phoneNumber": "0909090909090909",
        "picture": "https://s.gravatar.com/avatar/dd1cf547c8b950ce6966c050234ac997?s=480&r=pg&d=https%3A%2F%2Fcdn.auth0.com%2Favatars%2Fga.png",
        "providers": [
          "port"
        ],
        "status": "ACTIVE",
        "id": "auth0|XXXXX",
        "createdAt": "2022-12-08T16:34:20.735Z",
        "updatedAt": "2023-11-13T15:11:38.243Z"
      }
    },
    "origin": "UI",
    "at": "2023-11-13T15:20:16.641Z"
  },
  "context": {
    "entity": "e_iQfaF14FJln6GVBn",
    "blueprint": "kubecostCloudAllocation",
    "runId": "r_HardNzG6kzc9vWOQ"
  },
  "payload": {
    "entity": {
      "identifier": "e_iQfaF14FJln6GVBn",
      "title": "myEntity",
      "icon": "Port",
      "blueprint": "myBlueprint",
      "team": [],
      "properties": {
      },
      "relations": {},
      "createdAt": "2023-11-13T15:24:46.880Z",
      "createdBy": "auth0|XXXXX",
      "updatedAt": "2023-11-13T15:24:46.880Z",
      "updatedBy": "auth0|XXXXX"
    },
    "action": {
      "invocationMethod": {
        "type": "WEBHOOK",
        "agent": true,
        "synchronized": false,
        "method": "POST",
        "url": "https://myGitlabHost.com"
      },
      "trigger": "DAY-2"
    },
    "properties": {
    },
    "censoredProperties": []
  }
}
```

</details>


### The report mapping

After the request is sent to the 3rd party application, the Port agent can report the run status back to Port.
The report mapping is used to construct the report that will be sent to Port.

The report mapping can use the following fields:

`.body` - The incoming message as mentioned [Above](#the-incoming-message-to-base-your-mapping-on)
`.request` - The request that was calculated using the control the payload mapping and sent to the 3rd party application 
`.response` - The response that was received from the 3rd party application


### Examples

#### Terraform Cloud

Create the following blueprint, action and mapping to trigger a Terraform Cloud run.

<details>
<summary>Blueprint</summary>

```json
{
  "identifier": "terraform_cloud_workspace",
  "title": "Terraform Cloud Workspace",
  "icon": "Terraform",
  "schema": {
    "properties": {
      "workspace_id": {
        "title": "Workspace Id",
        "type": "string"
      },
      "organization_name": {
        "title": "Organization Name",
        "type": "string"
      },
      "workspace_name": {
        "title": "Workspace Name",
        "type": "string"
      }
    },
    "required": [
      "workspace_id",
      "organization_name",
      "workspace_name"
    ]
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "relations": {}
}
```
</details>

<details>
<summary>Action</summary>

```json
[
  {
    "identifier": "trigger_tf_run",
    "title": "Trigger TF Cloud run",
    "icon": "Terraform",
    "userInputs": {
      "properties": {},
      "required": [],
      "order": []
    },
    "invocationMethod": {
      "type": "WEBHOOK",
      "agent": true,
      "synchronized": false,
      "method": "POST",
      "url": "https://app.terraform.io/api/v2/runs/"
    },
    "trigger": "DAY-2",
    "requiredApproval": false
  }
]
```
</details>

<details>
<summary>Mapping - (Should be saved as `invocations.json`)</summary>

```json
[
  {
    "enabled": ".action == \"trigger_tf_run\"",
    "headers": {
      "Authorization": "\"Bearer \" + env.TF_TOKEN",
      "Content-Type": "\"application/vnd.api+json\""
    },
    "body": {
      "data": {
        "attributes": {
          "is-destroy": false,
          "message": "\"Triggered via Port\"",
          "variables": ".payload.properties | to_entries | map({key: .key, value: .value})"
        },
        "type": "\"runs\"",
        "relationships": {
          "workspace": {
            "data": {
              "type": "\"workspaces\"",
              "id": ".payload.entity.properties.workspace_id"
            }
          }
        }
      }
    },
    "report": {
      "status": "if .response.statusCode == 201 then \"SUCCESS\" else \"FAILURE\" end",
      "link": "\"https://app.terraform.io/app/\" + .body.payload.entity.properties.organization_name + \"/workspaces/\" + .body.payload.entity.properties.workspace_name + \"/runs/\" + .response.json.data.id",
      "externalRunId": ".response.json.data.id"
    }
  }
]
```
</details>

**Port agent installation for Terraform cloud example**:

```sh
helm repo add port-labs https://port-labs.github.io/helm-charts

helm repo update

helm install my-port-agent port-labs/port-agent \
    --create-namespace --namespace port-agent \
    --set env.secret.PORT_CLIENT_ID=YOUR_PORT_CLIENT_ID \
    --set env.secret.PORT_CLIENT_SECRET=YOUR_PORT_CLIENT_SECRET \
    --set env.normal.PORT_ORG_ID=YOUR_ORG_ID \
    --set env.normal.KAFKA_CONSUMER_GROUP_ID=YOUR_KAFKA_CONSUMER_GROUP \
    --set env.normal.KAFKA_CONSUMER_BROKERS=PORT_KAFKA_BROKERS \
    --set env.normal.STREAMER_NAME=KAFKA \
    --set env.normal.KAFKA_CONSUMER_AUTHENTICATION_MECHANISM=SCRAM-SHA-512 \
    --set env.normal.KAFKA_CONSUMER_AUTO_OFFSET_RESET=earliest \
    --set env.normal.KAFKA_CONSUMER_SECURITY_PROTOCOL=SASL_SSL \
    --set env.secret.TF_TOKEN=YOU_TERRAFORM_CLOUD_TOKEN \
    --set-file controlThePayloadConfig=./invocations.json
```



#### CircleCI

Create the following blueprint, action and mapping to trigger a CircleCI pipeline.

<details>
<summary>Blueprint</summary>

```json
{
  "identifier": "circle_ci_project",
  "title": "CircleCI Project",
  "icon": "CircleCI",
  "schema": {
    "properties": {
      "project_slug": {
        "title": "Slug",
        "type": "string"
      }
    },
    "required": [
      "project_slug"
    ]
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "relations": {}
}
```
</details>

<details>
<summary>Action</summary>

```json
[
  {
    "identifier": "trigger_circle_ci_pipeline",
    "title": "Trigger CircleCI pipeline",
    "icon": "CircleCI",
    "userInputs": {
      "properties": {},
      "required": [],
      "order": []
    },
    "invocationMethod": {
      "type": "WEBHOOK",
      "agent": true,
      "synchronized": false,
      "method": "POST",
      "url": "https://circleci.com"
    },
    "trigger": "DAY-2",
    "requiredApproval": false
  }
]
```
</details>

<details>
<summary>Mapping - (Should be saved as `invocations.json`)</summary>

```json
[{
    "enabled": ".action == \"trigger_circle_ci_pipeline\"",
    "url": "(env.CIRCLE_CI_URL // \"https://circleci.com\") as $baseUrl | .payload.entity.properties.project_slug | @uri as $path | $baseUrl + \"/api/v2/project/\" + $path + \"/pipeline\"",
    "headers": {
      "Circle-Token": "env.CIRCLE_CI_TOKEN"
    },
    "body": {
      "branch": ".payload.properties.branch // \"main\"",
      "parameters": ".payload.action.invocationMethod as $invocationMethod | .payload.properties | to_entries | map({(.key): (.value | tostring)}) | add | if $invocationMethod.omitUserInputs then {} else . end"
    }
  }]
```
</details>

**Port agent installation for CircleCI example**:

```sh
helm repo add port-labs https://port-labs.github.io/helm-charts

helm repo update

helm install my-port-agent port-labs/port-agent \
    --create-namespace --namespace port-agent \
    --set env.secret.PORT_CLIENT_ID=YOUR_PORT_CLIENT_ID \
    --set env.secret.PORT_CLIENT_SECRET=YOUR_PORT_CLIENT_SECRET \
    --set env.normal.PORT_ORG_ID=YOUR_ORG_ID \
    --set env.normal.KAFKA_CONSUMER_GROUP_ID=YOUR_KAFKA_CONSUMER_GROUP \
    --set env.normal.KAFKA_CONSUMER_BROKERS=PORT_KAFKA_BROKERS \
    --set env.normal.STREAMER_NAME=KAFKA \
    --set env.normal.KAFKA_CONSUMER_AUTHENTICATION_MECHANISM=SCRAM-SHA-512 \
    --set env.normal.KAFKA_CONSUMER_AUTO_OFFSET_RESET=earliest \
    --set env.normal.KAFKA_CONSUMER_SECURITY_PROTOCOL=SASL_SSL \
    --set env.secret.CIRCLE_CI_TOKEN=YOUR_CIRCLE_CI_PERSONAL_TOKEN \
    --set-file controlThePayloadConfig=./invocations.json
```

#### Windmill

This example helps internal developer teams to trigger [Windmill](https://www.windmill.dev) job using Port's self service actions. In particular, you will create a blueprint for `windmillJob` that will be connected to a backend action. You will then add some configuration files (`invocations.json`) to control the payload and trigger your Windmill job directly from Port.


Create the following blueprint, action and mapping to trigger a Windmill job.

<details>
<summary>Blueprint</summary>

```json
{
  "identifier": "windmillJob",
  "description": "This blueprint represents a windmill job in our software catalog",
  "title": "Windmill",
  "icon": "DefaultProperty",
  "schema": {
    "properties": {
      "workspace": {
        "type": "string",
        "title": "Workspace"
      },
      "path": {
        "type": "string",
        "title": "File Path"
      },
      "trigerredBy": {
        "type": "string",
        "title": "Triggered By",
        "format": "user"
      },
      "createdAt": {
        "type": "string",
        "format": "date-time",
        "title": "Created At"
      }
    },
    "required": []
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "relations": {}
}
```
</details>

<details>
<summary>Action</summary>

```json
[
   {
      "identifier":"trigger_windmill_pipeline",
      "title":"Trigger Windmill Pipeline",
      "icon":"DefaultProperty",
      "userInputs":{
         "properties":{
            "workspace":{
               "title":"Workspace",
               "description":"The Workspace identifier",
               "type":"string"
            },
            "file_path":{
               "title":"File Path",
               "description":"The path of the job script in the workspace, including the /u and /f prefix",
               "type":"string"
            },
            "job_data":{
               "title":"Job Data",
               "description":"The data to be passed to the job in order to execute successfully",
               "type":"object"
            }
         },
         "required":[
            "workspace",
            "file_path",
            "job_data"
         ],
         "order":[
            "workspace",
            "file_path",
            "job_data"
         ]
      },
      "invocationMethod":{
         "type":"WEBHOOK",
         "agent":true,
         "synchronized":false,
         "method":"POST",
         "url":"https://app.windmill.dev/api"
      },
      "trigger":"CREATE",
      "requiredApproval":false
   }
]
```
</details>

<details>
<summary>Mapping - (Should be saved as `invocations.json`)</summary>

```json
[
  {
    "enabled": ".action == \"trigger_windmill_pipeline\"",
    "url": "\"https://app.windmill.dev\" as $baseUrl | .payload.properties.workspace as $workspace | .payload.properties.file_path as $path | $baseUrl + \"/api/w/\" + $workspace + \"/jobs/run_wait_result/f/\" + $path",
    "headers": {
      "Authorization": "\"Bearer \" + env.WINDMILL_TOKEN",
      "Content-Type": "\"application/json\""
    },
    "body": ".payload.properties.job_data"
  }
]
```
</details>

**Port agent installation for Windmill example**:

```sh
helm repo add port-labs https://port-labs.github.io/helm-charts

helm repo update

helm install my-port-agent port-labs/port-agent \
    --create-namespace --namespace port-agent \
    --set env.normal.PORT_ORG_ID=YOUR_ORG_ID \
    --set env.normal.KAFKA_CONSUMER_GROUP_ID=YOUR_KAFKA_CONSUMER_GROUP \
    --set env.secret.KAFKA_CONSUMER_USERNAME=YOUR_KAFKA_USERNAME \
    --set env.secret.KAFKA_CONSUMER_PASSWORD=YOUR_KAFKA_PASSWORD
    --set env.normal.KAFKA_CONSUMER_BROKERS=PORT_KAFKA_BROKERS \
    --set env.normal.STREAMER_NAME=KAFKA \
    --set env.normal.KAFKA_CONSUMER_AUTHENTICATION_MECHANISM=SCRAM-SHA-512 \
    --set env.normal.KAFKA_CONSUMER_AUTO_OFFSET_RESET=earliest \
    --set env.normal.KAFKA_CONSUMER_SECURITY_PROTOCOL=SASL_SSL \
    --set env.secret.WINDMILL_TOKEN=YOUR_WINDMILL_TOKEN \
    --set-file controlThePayloadConfig=./invocations.json
```
#### Run action
Run this action with some input

```json showLineNumbers
{
    "workspace": "demo",
    "file_path": "f/examples/ban_user_example",
    "job_data": {
        "value": "batman",
        "reason": "Gotham city in need of superhero",
        "database": "$res:f/examples/demo_windmillshowcases",
        "username": "Jack",
        "slack_channel": "bans"
    }
}
```