# Lab 1 - Hello World

This is a guided tour to walk you through getting started with Relay. It takes about 15 minutes to complete. By the end of the guide, you'll be able to run hello world and get some output.

## Step 1: Log into your account

Log into your account and make sure you're in the SE account. 

## Step 2: Create a simple workflow

Relay is based around the idea of workflows, which combine useful activities together to accomplish a specific task. Workflows are written in YAML and stored on the service so they can be triggered manually or via incoming events. Let's start by creating a simple "Hello World" workflow that just executes an `echo` command with any argument you give it.

From your newly-created account, navigate to the **Workflows** tab. Add a new workflow from the center of the screen or use the **New workflow** button in the top right corner. In the dialog box, give it a name of `hello-world-{your name}` and a description "Welcome to Relay". 

![New workflow](/images/new-workflow.png)

> **TIP:** Annotating workflows with your name will help you tell them apart in the shared TSE account

You'll get dropped into an editor window, where you can copy-and-paste the following workflow, then click the **Save changes** button on the top right of the editor window.

```yaml
parameters:
  message:
    description: "The message to output from the step"
steps:
- name: hello-world
  image: relaysh/core
  spec:
    message: !Parameter message
  input:
  - echo "Hello world. Your message was $(ni get -p {.message})"
```

This workflow starts by defining a parameter named `message`, whose value you'll need to supply before the workflow starts. Then it defines a step named `hello-world`, using a container image stored on Docker Hub at [`relaysh/core`](https://hub.docker.com/r/relaysh/core), which is an Alpine Linux-based image curated by Puppet.

The `spec` map defines keys and values that will be available inside the step. In this case, we're using the Relay `!Parameter` type to look up the value of the global `message` parameter and make it available to the container.

**TIP:** In addition to `!Parameter`, Relay has additional custom YAML types that you can use in your workflows. For example, you can use `!Secret` to access encrypted values. You also have access to several utility functions like `!Fn.merge` and `!Fn.concat` that allow you to manipulate data in your workflows.

Relay executes the container with the value of the `input` step as its entrypoint. The "Hello world" message uses a command-line tool built for Relay called `ni` to inject the value of the message parameter into its output.

## Step 3: Run it via the GUI

Once you save the workflow, use the **Run** button in the top right to execute it. You'll get a dialog box prompting you to fill in a value for the **message** parameter, so enter something witty (if you need a suggestion, try "relay.sh r00lz"), then click **Run workflow**.

Relay will queue your workflow run and then execute it. You'll see some statistics and a graph visualization of the step sequence. Because it only has one step, this workflow's graph has one node represented as a box; click it to see the step's details in the sidebar.

Once the step completes, you can select the **Code** tab to show the workflow code we just wrote. Select the **Logs** tab to see the earth-shattering results of your workflow run.

## Step 4: Install the CLI

The web app is great for visualization, log history, and troubleshooting. For workflow editing and authoring, however, you'll almost certainly want to use the command-line interface, `relay`.

You can grab the latest release from the [puppetlabs/relay GitHub site](https://github.com/puppetlabs/relay/releases); it's a single binary download so download the latest release for your operating system, install it into your `$PATH`, and make it executable.

For easy installation and updating on a Mac, you can use [Homebrew](https://brew.sh):

```shell
brew install puppetlabs/puppet/relay
```
We also have packaging scripts for [Arch Linux](https://github.com/puppetlabs/relay/blob/master/build/package/archlinux/PKGBUILD) and welcome additional packaging contributions!

## Step 5: Download your workflow

With the CLI installed, you can use it to authenticate to your account and operate on workflows. Running `relay` by itself will show the list of available subcommands, and you can run any subcommand with the `--help` option to get detailed usage information. Use the following commands to log in, get a list of your available workflows, and download the `hello-world` workflow we created earlier to a local file:

```
relay auth login
relay workflow list
relay workflow download hello-world > hello-world.yaml
```


## Step 6: Modify the workflow to pass data between steps

Relay executes each step in a workflow as a separate container. This allows you to link together smaller, built-to-purpose containers that each accomplish a concrete task rather than putting everything into one big script. Modular components increase reusability, but they do present more of a challenge to pass state information between steps.

Now that we have a single-step workflow running, let's add a step that generates additional input for the "hello world" message. This will illustrate how to use the output from one step as the input to another.

Replace the contents of the workflow file with the following code:

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
    message: !Parameter message
    dynamic: !Output [generated-output,dynamic]
  input:
  - echo "Hello world. Your message was $(ni get -p {.message}), and the generated output was $(ni get -p {.dynamic})"
```

We've added a `generated-output` step which uses the `ni` utility to set the value of a key named `dynamic` with the output of the `/bin/date` program. The spec for the `hello-world` step uses the `!Output` type to look up the value for `dynamic`, and make it available inside the step for `ni` to read.

## Step 7: Update and run the workflow via the CLI

Don't forget to save your work! You've got it stored locally, but you'll need to update the version that lives on the Relay service. Relay keeps your workflows in cloud storage and always runs the latest version you've uploaded. It's a good idea to use git to keep your workflows under revision control, but for now, we can just replace the previous version with our updates using the CLI:

```
relay workflow replace hello-world -f hello-world.yaml
```

Now, run the workflow, supplying the `message` parameter using the `-p` flag. When you have a workflow that takes multiple parameters, you can use several `-p` (or `--parameter`, if you really like typing!) flags in a row.

```
relay workflow run hello-world -p message="Run from CLI"
```

The output of this command will include a URL that you can open in your web browser to watch the progress of the run. Under the hood, Relay figured out that, because of the `!Output` tag, the `hello-world` step now has a _dependency_ on the `generated-output` step. So the graph now represents that ordering. You can also use an explicit `dependsOn: step-name` key in a step's definition to force ordering. Without ordering information, Relay will run the steps in parallel to speed up the workflow execution.

## Step 8: Modify workflow to add a secret

Secret management is one of the most significant advantages to using Relay over running scripts from your laptop. Relay keeps sensitive data like passwords and credentials securely stored in Hashicorp Vault and decrypts them for the duration of a workflow run. Eliminating password sprawl and hardcoded API keys reduces risk and increases the reusability of workflows.

Let's modify our `hello-world` workflow one last time to add a placeholder secret and supply that value. In the web app, click on the **Code** tab and edit your `hello-world` step to look like the following:

```yaml
- name: hello-world
  image: relaysh/core
  spec:
    message: !Parameter message
    dynamic: !Output [generated-output,dynamic]
    supersecret: !Secret mysecret
  input:
  - echo "Hello world. Your message was $(ni get -p {.message}), and the generated output was $(ni get -p {.dynamic})."
  - echo "Normally I would never say this, but your secret was $(ni get -p {.supersecret})"
```
The `ni` utility can access secrets just like regular parameters, but you wouldn't generally want to echo them to the logs as in this example!
When you click **Save changes**, you'll see a warning dialog that you're missing a required secret, and a prompt to fill it in. Follow the link in the prompt to open the **Setting** sidebar. Click the **+** icon and enter a secret value. Secrets can be deleted and re-added, but never viewed.

## Step 9: Extending the workflow 
One of Relay's great strengths is the ability to extend the workflow using either the GUI or code editor. We're going to extend this workflow by sending the message to Slack. 

* Navigate back to the GUI and select the **(+)** sign immediately below the trigger to add another step to execute in parallel with the `echo` step. 
* Select the "Send a message to a Slack channel" step
* Configure the step:
    * **connection** - Use the `puppet-workspace` connection to configure the step. 
    * **channel** - Use `#prog-relay-testing` to send to 
    * **username** - Use `Relay SE Training` 
    * **message** - Select `Parameter` from the dropdown and select `message` as the parameter. 

## Step 10: Adding a Push trigger 
Relay supports multiple ways to trigger the workflow: 
- Manual trigger (using the Run Workflow button)
- Scheduled trigger (using cron syntax)
- Webhook trigger (using one of the integration triggers)
- Push trigger (over https)

For more information, check out the [documentation](https://relay.sh/docs/using-workflows/using-triggers/).

We're going to set up a Push trigger:
- Navigate back to the GUI and click the "Add a trigger" button
- Find and select the "Push trigger" and click "Add trigger"
- Click on the Trigger and copy the bearer token.

From your terminal, run the following command: 
```
export TOKEN=... # get this from the web app
curl -X POST -H "Authorization: Bearer $TOKEN" \
   -d '{"data": {"message": "This is a push event"}}' \
   https://api.relay.sh/api/events
```

Congratulations! You now have a running example that exercises many of Relay's useful features:

* The Relay web app shows all the workflows in your account and keeps detailed logs of their execution history.
* You can use the `relay` cli to develop workflows locally, trigger parameterized runs, and add, delete or replace the workflows stored on the Relay service. 
* You can securely store sensitive data like API keys and passwords using the `!Secret` key and secret management sidebar.
