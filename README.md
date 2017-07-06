## Overview
Simple package to help with Cloudformation deployments

### Goals
* Make it easy to deploy multi-stack, multi-account, multi-environment, and multi-region Cloudformation templates
* Promote use of 'native' Cloudformation templates (i.e., with minimal pre-processing)

### Installation
Requires:
* nodejs v8.0.0 or later
* npm

Add the github link as a npm dependency to your local NPM repo.
```
npm install --save 'https://github.com/jaymell/nodeCf.git#v.9.7.1'
```

### Usage
```
node_modules/.bin/nodeCf <ENVIRONMENT> [ ACTION ] -s,--stacks <STACK NAMES> [ -r <REGION> ] [ -p <PROFILE> ] [ -e, --extra-vars <EXTRA VARS> ]
```

Run deployment against specified ENVIRONMENT.

* ACTION defaults to 'deploy': choices are 'deploy', 'delete', and 'validate'
* REGION specifies the desired AWS Region. Currently defaults to 'us-east-1'
* PROFILE specifies an optional name for an AWS profile to assume when running the job
* STACK NAME corresponds to the name of your Cloudformation templates, separated by commas if multiple
* EXTRA VARS indicate extra variables for deployment; useful for any variables that are only known at runtime; in the form "KEY=VALUE" -- additional variables should be separated by spaces

### Template Files
Cloudformation templates -- by default are stored in `./templates` in either json or yaml format.

By default, the template name should match the unqualified name of the stack -- e.g., a stack named 'service' should have a corresponding file named `service`, `service.yml`,  `service.yaml`, or `service.json` in the templates folder. You can override this, however, by passing a `templateName` property on the stack object.

Example:
```
- name: network
  templateName: mynetwork
  parameters:
    VpcIPRange: "{{VpcIPRange}}"
```

### Config Files
Config files must be written in yaml and by default are looked for in `./config`
* Environment Config File (Required): Stores environment-specific variables  -- e.g., `./config/dev.yml`
* Global config file (Required -- but maybe should be optional): Stores application- (but not environment-) specific variables -- e,g, `./config/global.yml`
* Stack configuration (Required) -- Defines parameters and tags to pass to Cloudformation, as well as pre-and post-tasks (see below for more info)
* NodeCf configuration (Optional) -- This feature doesn't actually exist yet, but should allow for overriding variables that get set in the `config` module

### Required variables:
* environment -- this must be passed on command line as first argument to command; its name must match the name of your environment variables file
* region -- this must be passed on command line; currently defaults to us-east-1 if not specified
* account -- your AWS account number
* application -- this can be anything, but the name of your repository is a good default; it is used for naming and uniquely identifying resources
* infraBucket -- Cloudformation stacks over a certain size must first be uploaded to s3; as a result, nodeCf requires the name of a bucket to use for deployments; the scripts will handle creating it for you (assuming its name has not already been taken by some other random AWS user).

Example stacks.yml:
```
---
stacks:
- name: network
  parameters:
    VpcIPRange: "{{VpcIPRange}}"
    PrivateSubnet0: "{{PrivateSubnet0}}"
    PrivateSubnet1: "{{PrivateSubnet1}}"
- name: rds
  parameters:
    NetworkStack: "{{environment}}-{{application}}-network"
    PrivateSubnet0: "{{PrivateSubnet0}}"
    PrivateSubnet1: "{{PrivateSubnet1}}"
```

Example env.yml:
```
---
account: "{{accounts.production}}"
infraBucket: myUniqueBucketname # required -- nodeCf will attempt to create it if it doesn't exist
VpcIPRange: 10.0.0.0/8
PrivateSubnet0Cidr: 10.0.0.0/24
PrivateSubnet1Cidr: 10.0.1.0/24
```

Example global.yml:
```
---
application: MyApplication
accounts:
  production: <MY AWS Account number -- e.g., 123456789012>
```

### Filters
You can also use your own filters, which are custom node functions that allow you to modify variables or perform arbitrary actions; by default, place them in `./config/filters.js`. Export them as you would in any other nodejs module, e.g.:
```
module.exports = {
  sync: {
    mySyncFunction: mySyncFunction,
  },
  async: {
    myAsyncFunction: myAsyncFunction
  }
};
```

You could then pass your variable through a filter like this:
```
{{ myVariable | filterName }}
```

If the filter function is asynchronous, you must indicate so by wrapping it in an `async` key in modules.exports. You can read more about filters in the [Nunjucks documentation](https://mozilla.github.io/nunjucks/templating.html#filters).

### Creation Tasks, Pre-Tasks, Post-Tasks, and Lambda Artifacts
There are a few additional properties you can add to the individual stack definitions to help with deployments.

#### Pre-Tasks, Post-Tasks and Creation Tasks
If you consistently need to run an arbitrary shell script or command immediately prior to or after deploying a CF template, you can add it under `preTasks` or `postTasks`, which consists of an array of shell-interpreted strings. If you only need to run a script when a stack is first created, you can call it under `creationTasks`. For example:

```
stacks:
- name: network
  parameters:
    VpcIPRange: "{{VpcIPRange}}"
    PrivateSubnet0: "{{PrivateSubnet0}}"
    PrivateSubnet1: "{{PrivateSubnet1}}"
  creationTasks:
  - "./scripts/doThisOnlyWhenStackFirstCreated.js"
  preTasks:
  - "./scripts/preTask1.sh"
  - "./scripts/preTask2.sh"
  postTasks:
  - "./scripts/peerToSharedVpc.sh"
```

Note that you're not limited to shell scripts -- these can be any scripts in any language, provided the system deploying the stacks has the proper tools installed.

#### Lambda Artifacts
Deploying Lambda functions via Cloudformation can be a pain. The code must be built and packaged (which is _not_ handled here), uploaded to s3 with a unique name (if the name of the artifact doesn't change with subsequent deployments, your code won't be updated), then the location in s3 must be passed to the actual CF template in which the Lambda function is defined. There are a few helpers to make this a bit easier.

Assuming you've built and packaged your lambda function (e.g., in a zip file), you can specify the path to it under a `lambdaArtifact` property, then reference its location with `{{lambda.STACK NAME.bucket}}` and `{{lambda.STACK NAME.key}}`. NodeCf will handle uploading it to s3 with a unique name.

For example:
```
stacks:
- name: myLambdaStack
  lambdaArtifact: ./lambda/dist/myLambda.zip
  parameters:
    LambdaBucket: "{{lambda.myLambdaStack.bucket}}"
    LambdaKey: "{{lambda.myLambdaStack.key}}"
```

### TO DO
* (Optionally) delete templates from s3 after deployment
* Use change sets
* Make it easy to set stack update policies
* Add example project
