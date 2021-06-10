# Integrating with Relay

There's a growing [ecosystem of integrations](https://relay.sh/integrations/) that connect Relay with other tools and services, but it's a big world out there. There are always useful integrations that don't yet exist, but should. This guide will help you build an integration between Relay and an external service, so you can automate away your pain.

## Decide on a goal

This may seem self-evident, but it's important to determine what your goal is _before_ you begin coding. Relay has several useful points of extensibility, and understanding the purpose of each will help you find a starting point. Integrations consist of containers that are used in different parts of Relay, which have two flavors: **steps** and **triggers**.

* **Steps** - Relay runs steps, passing in parameters and secrets, as part of an automation workflow. Most of the work in Relay is done by steps, which use an SDK to retrieve and modify a workflow's state as it executes.
* **Triggers** - Relay supports several [types of triggers](../reference/relay-workflows.md#Triggers), which are event handlers that initiate workflow runs. The Relay service handles `push` triggers natively, but `webhook` triggers work by executing a user-defined container to handle the payload of the webhook's incoming HTTP request. Therefore, integrations that connect to Relay using webhooks need to provide a trigger container.

Steps and triggers come together in workflows, YAML code written to accomplish the task you're faced with. Workflow authoring is covered in the [Using workflows](../using-workflows.md) documentation, so here we'll focus on creating and using steps and triggers.

## Creating containers for Relay

In its simplest form, a Relay container image runs, does some work, and then exits. It's possible to use any OCI-compliant container, as long as its entrypoint terminates with a zero exit code upon success and a non-zero code upon failure. However, beyond "hello world" examples, you'll likely want an image with more capabilities than just using `alpine:latest`. There are two possible paths here, depending on your use case.

1. If the work you're trying to do can be written in a shell or Python script, use the `relaysh/core` image. This is an Alpine-based image that the Relay team maintains, which comes pre-loaded with the Relay SDK. There are `relaysh/core:latest` and `relaysh/core:latest-python` flavors available. You can use these either as a base image in your own Dockerfile or in workflows directly, by specifying an `inputFile` tag whose value is a URL to your script.
2. If there's an existing image that's made for your integration target, you can use it as the starting point for a custom container. Adding the Relay tools in at build time will allow you to use the metadata service via either the command-line interface or code SDK.

### Using relaysh/core

Using the `relaysh/core` image is the easiest option because it avoids having to build and push a custom container image, but it restricts you to providing your script in a single file. To use it, make your script accessible via HTTPS — GitHub repositories or gists both work fine — and provide the URL to it in the `inputFile` attribute on a `step` or `trigger` definition. For very simple scripts, you can use the [`input` attribute](../reference/relay-workflows.md#input) and provide your commands as an array of shell commands.

There are two variants of `relaysh/core`, indicated by their tag: `relaysh/core:latest` accepts a Bash shell as input and `relaysh/core:latest-python` expects a Python script. Specify which you want to use in the `image` attribute of your definition.

#### relaysh/core shell example

Because webhook trigger containers need to handle a web request, the shell image isn't suitable for triggers, only steps.

```yaml
steps:
  - name: run-my-script
    image: relaysh/core:latest
    inputFile: https://git.io/JfiwE
```

#### relaysh/core Python example

The Python image is usable for both steps and triggers. The following workflow receives a pushed tag from Docker Hub and sends a notification with the name of the tag.

```yaml
parameters:
  dockerTagName:
    description: The tag of the container which was pushed
triggers:
  - name: my-python-trigger
    source:
      type: webhook
      image: relaysh/core:latest-python
      inputFile: https://git.io/JfiwD
    binding:
      parameters:
        dockerTagName: ${event.tag}
steps:
  - name: my-python-notification
    image: relaysh/core:latest-python
    inputFile: https://TODO/
    spec:
      tag: ${parameters.dockerTagName}
```

### Using an upstream image

While it's possible to use an upstream image without modification, customizing it can greatly increase its usefulness. Adding the Relay SDK or `ni` command-line utility to the container enables access to the Relay service APIs, which allow you to retrieve and set parameters, access secrets, and use Relay's persistent logging framework. Additionally, writing a custom entrypoint allows you to control the container's behavior to ensure it runs correctly in Relay.

For trigger containers, we recommend using the `relaysh/core:latest-python` image as your base image and adding your own code as the entrypoint. See the [Writing entrypoint code](#writing-entrypoint-code) section for details.

This section assumes you're familiar with Dockerfiles and the container build/push process.

#### Adding the CLI

The [`ni` CLI utility](../cli/ni.md) is a small Go program meant to be run inside Relay containers. The most common use for it is to get and set variables from the Relay service API from entrypoint shell scripts. To add it to your container, use a snippet like this in your Dockerfile:

```shell
RUN set -eux ; \
    mkdir -p /tmp/ni && \
    cd /tmp/ni && \
    wget https://packages.nebula.puppet.net/sdk/ni/v1/ni-v1-linux-amd64.tar.xz && \
    wget https://packages.nebula.puppet.net/sdk/ni/v1/ni-v1-linux-amd64.tar.xz.sha256 && \
    echo "$( cat ni-v1-linux-amd64.tar.xz.sha256 )  ni-v1-linux-amd64.tar.xz" | sha256sum -c - && \
    tar -xvJf ni-v1-linux-amd64.tar.xz && \
    mv ni-v1*-linux-amd64 /usr/local/bin/ni && \
    cd - && \
    rm -fr /tmp/ni
```

This will download, verify, and install the latest version of the CLI. To use it in your entrypoint, the `ni get` subcommand will (with no arguments) print a JSON string containing all the parameters listed in the step's `spec` section. Given a parameter to retrieve, it'll print that in string form (without an enclosing JSON object).

The `ni output set` subcommand will write values back to the metadata service for use by later steps, and `ni log` will write log messages which can be helpful when debugging.

```yaml
parameters:
  message:
    description: "The message to output from the final step"
steps:
- name: generated-output
  image: relaysh/core
  input:
  - ni output set --key dynamic --value "$(/bin/date)"
- name: hello-world
  image: relaysh/core
  spec:
    message: ${parameters.message}
    dynamic: ${outputs.'generated-output'.dynamic}
    supersecret: ${secrets.mysecret}
  input:
  - echo "Hello world. Your message was $(ni get -p {.message}), and the generated output was $(ni get -p {.dynamic})."
  - echo "Everything I know about your run is contained in $(ni get)"
  - ni log info "This worked out wonderfully."
```

#### Adding the SDK

If you're using Python in your container, the latest version of the Relay SDK can be installed via `pip` in your Dockerfile:

```shell
RUN pip --no-cache-dir install relay-sdk
```

The SDK itself has a top-level README and inline API documentation; check out the [latest docs at readthedocs.io](https://relay-sdk-python.readthedocs.io/en/latest/).

### Writing entrypoint code

Whichever route you've chosen, at this point you've got the setup for a Relay-friendly container. The last part is probably the most interesting, because it's where you need to write enough code to make the container do your bidding. We'll handle the examples for steps and triggers slightly differently, since there are different APIs you'll be working with.

#### Trigger entrypoints and webhook API

Under the hood, when a workflow references a webhook trigger, the Relay app will prompt the user to register the webhook on the remote service. Once it's set up, Relay associates all incoming requests to their associated trigger containers. When a request comes in, Knative launches the appropriate container and connects the incoming payload to the entrypoint, an HTTP handler.

The container does the following:
* Starts a webserver on the port specified by the environment variable `$PORT` (defaults to `8080`).
* Handles a POST request and decode the incoming payload.
* Decides whether to run the rest of the workflow.
* Maps values from the request payload onto event data.
* Sends this mapping back into the Relay service as an event.

The Relay Python SDK is by far the easiest way to do this. The [Integration template repository](https://github.com/relay-integrations/template/) has a simple starting point, and there are full-featured examples for [Dockerhub push events](https://github.com/relay-integrations/relay-dockerhub/blob/master/triggers/dockerhub-trigger-image-pushed/handler.py) and [new Pagerduty incidents](https://github.com/relay-integrations/relay-pagerduty/blob/master/triggers/incident-triggered/handler.py). Check out the [Relay integrations on Github](https://github.com/relay-integrations/) for more ideas.

#### Step entrypoints

Steps have a relatively easy lot in life. They run, do some work to advance the state of the workflow, and exit. In many cases, a step can use existing scripts which you modify only enough to take advantage of the Relay service API. The [Getting started](../getting-started.md) shows an example workflow that uses the `input` map to access this API for retrieving and setting keys and values to pass data through the workflow. For more advanced use cases, you'll probably need a script that uses the [ni utility](../cli/ni.md) or its [python SDK equivalent](https://relay-sdk-python.readthedocs.io/en/latest/#accessing-data-from-the-step-spec) to get and set data.

Formally, a step container running in Relay can expect the following:
* It will be executed without persistent storage.
* Only keys declared in the step's `spec` map in the workflow will be available in the metadata service.
* It can set output values through the metadata service which will be available to later steps.
* Exiting with a zero exit code will cause the workflow run to continue.
* Exiting with a non-zero exit code will cause the workflow run to be terminated and no dependent steps will run.

For examples of using the `ni` utility in shell commands, see the [kustomize integration](https://github.com/relay-integrations/relay-kustomize/tree/master/steps/apply); for examples of using the Python SDK, the [AWS EC2 integration](https://github.com/relay-integrations/relay-aws-ec2/tree/master/steps) has several steps of varying complexity.

### Building and publishing containers

Once you've got your base image and custom code together, you can proceed with building and publishing the containers. Relay has conventions for container and integration metadata which aren't _required_ but will increase consistency and help with future compatibility. These conventions are encoded in the [template integration repo](https://github.com/relay-integrations/template), specifically, Relay containers should be built with `LABEL` directives that indicate their title and description.

The following Dockerfile shows this in action:

```shell
FROM relaysh:core-python
COPY "./handler.py" "/step-github-trigger-pull-request-merged.py"
ENTRYPOINT []
CMD ["python3", "/step-github-trigger-pull-request-merged.py"]

LABEL "org.opencontainers.image.title"="GitHub pull request merged"
LABEL "org.opencontainers.image.description"="This trigger fires when a GitHub pull request is merged."
```

You can publish images to any publicly-accessible container registry. The service defaults to Docker Hub for workflow `image` attributes that are not fully-qualified with a hostname, so these are equivalent:

```yaml
steps:
  - name: default-registry
    image: relaysh/core
  - name: explicit-registry
    image: index.docker.io/relaysh/core
```

## Designing a complete integration

While Relay supports running any Linux container, it is most powerful when metadata accompanies the container. The metadata helps the service describe and validate the image's requirements. We call the combination of metadata and image definitions a Relay integration.

You may have already taken a look at the integrations we publish in [our GitHub organization](https://github.com/relay-integrations). Each of these is based on an initial [template](https://github.com/relay-integrations/template) that provides a starting point for the required parts of an integration:

* A file called `integration.yaml` at the root of the integration that defines the integration name and version
* For each step image, a file called `step.yaml` in the respective directory (for a step named `foo`, the path would be `steps/foo/step.yaml`)
* For each trigger image, a file called `trigger.yaml` in the respective directory (for a trigger named `bar`, the path would be `triggers/bar/trigger.yaml`)

For a deeper look at Relay integrations, take a look at the [integration layout RFC](https://github.com/puppetlabs/relay-rfcs/blob/master/content/0006-integration-layout/rfc.md).

### Image metadata

In addition to providing a name and version, the information in a `step.yaml` or `trigger.yaml` enables Relay to give users more context about the image when they use it.

Specifying the following fields will help users get the best experience when they use your images:

* `summary`: A human-readable title for the image, like "List EC2 instances"
* `description`: A paragraph or two describing the purpose of the image and how to work with it in Markdown format
* `examples`: A list of examples or templates that a user can start from
* `schemas`: JSON Schema validation schemas for your `spec`s, outputs, and events

### Working with examples

Any step or trigger can provide zero or more usage examples, either in the form of a single step or trigger or an entire workflow. You must indicate whether an example is for a step, trigger, or workflow by using an API version and kind. For example,

```yaml
examples:
- summary: Create a new GitHub issue
  content:
    apiVersion: v1
    kind: Step
    name: create-issue-on-github
    image: relaysh/github-step-issue-create
    spec:
      title: My first issue
- summary: Use a user-specified parameter to create a new GitHub issue
  content:
    apiVersion: v1
    kind: Workflow
    parameters:
      title:
        description: The GitHub issue title
    steps:
    - name: create-issue-on-github
      image: relaysh/github-step-issue-create
      spec:
        title: ${parameters.title}
```

### Working with schemas

Relay generates documentation and performs validation when you provide schemas for your steps and triggers. In step metadata, schemas look like this:

```yaml
schemas:
  spec:
    $schema: 'http://json-schema.org/draft-07/schema#'
    type: object
    properties:
      title:
        type: string
    required: [title]
  outputs:
    $schema: 'http://json-schema.org/draft-07/schema#'
    type: object
    properties:
      issueNumber:
        type: integer
    required: [issueNumber]
```

In trigger metadata, the schemas are:

```yaml
schemas:
  spec:
    $schema: 'http://json-schema.org/draft-07/schema#'
    type: object
    properties: {} # ...
  event:
    $schema: 'http://json-schema.org/draft-07/schema#'
    type: object
    properties: {} # ...
```

For a particular schema, you can also write it in a separate file and then reference it. This is useful if your editor provides [JSON Schema meta-validation](https://json-schema.org/specification.html#meta-schemas).

```yaml
schemas:
  spec:
    source: file
    file: spec.schema.json
```

Our JSON Schema documentation renderer supports one extension, `x-relay-connectionType`, that you can add to an object schema to indicate that the value must come from a [Relay connection](../using-workflows/managing-connections.md). For example, to indicate that a step requires an AWS connection, you might write:

```yaml
# ...
connection:
  type: object
  x-relay-connectionType: aws
```

You can also use the `writeOnly` schema property to hint to a user that a value typically comes from a secret:

```yaml
# ...
password:
  type: string
  writeOnly: true
```
