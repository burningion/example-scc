# Example SCC / Agent YAML for OpenShift Origin

In order to reproduce this environment, you'll need to download OpenShift, and run:

```
$ ./oc cluster up
```

You can then log into the admin console at localhost:8443, using the supplied default username and password.

Once there, click the top right icon that says create project. Create one named `django-example`. Click into it, and click import yaml / json.

Paste the OpenShift YAML template from [here](https://raw.githubusercontent.com/burningion/django-ex/master/openshift/templates/django.json), and naming the project `django-example`. Keep the namespace for the imagestream set to the default, `openshift`.

You can also use the [same repo](https://github.com/burningion/django-ex) as the url for your Git repo.

With this, everything will spin up, and your project will be created.

To add the Datadog Agent, you'll need to run:

```
$ ./oc project django-example
$ ./oc apply -f create-roles.yaml
```

In the terminal. This will create the roles necessary, for running the container.

You can then add your Agent API key to the `create-agent.yaml` file, followed by:

```
$ ./oc apply -f create-agent.yaml
```

Which will create the agent daemonset for Datadog. With this, you should see your agent added to the cluster.

## Issues

In my case, when I spin up my agent, I get a bunch of errors at the beginning of the Docker container's life:

```
[s6-init] making user provided files available at /var/run/s6/etc...exited 0.
s6-chown: fatal: unable to chown /var/run/s6/etc/cont-init.d/89-copy-customfiles.sh: Operation not permitted
s6-chown: fatal: unable to chown /var/run/s6/etc/cont-init.d/50-mesos.sh: Operation not permitted
s6-chown: fatal: unable to chown /var/run/s6/etc/cont-init.d/59-defaults.sh: Operation not permitted
s6-chown: fatal: unable to chown /var/run/s6/etc/cont-init.d/01-check-apikey.sh: Operation not permitted
s6-chown: fatal: unable to chown /var/run/s6/etc/cont-init.d/60-network-check.sh: Operation not permitted
s6-chown: fatal: unable to chown /var/run/s6/etc/cont-init.d/50-ecs.sh: Operation not permitted
s6-chmod: fatal: unable to change mode of /var/run/s6/etc/cont-init.d/89-copy-customfiles.sh: Operation not permitted
s6-chown: fatal: unable to chown /var/run/s6/etc/cont-init.d/10-dogstatsd-socket.sh: Operation not permitted
s6-chmod: fatal: unable to change mode of /var/run/s6/etc/cont-init.d/59-defaults.sh: Operation not permitted

```

... and so on.

It appears this is related to [line 90](https://github.com/DataDog/datadog-agent/blob/master/Dockerfiles/agent/Dockerfile#L90) of the Agent Docker image, where we try to create a user in the `root` group.

Looking at the [OpenShift Origin Specific Guidelines](https://docs.openshift.org/latest/creating_images/guidelines.html#openshift-specific-guidelines), it seems we wouldn't have the permissions to do this in the first place, and instead we'll need to follow the instructions for random UIDs there.

## Workaround

I got the Agent to still work and get all access by changing the `SecurityContextConstraints` part of my YAML to run as `root`.

You can do this too by changing the lines for `runAsUser type: MustRunAsRange` to `runAsUser type: RunAsAny`.

You may need to delete and recreate the daemonset to see your changes applied:

```
$./oc apply -f create-roles.yaml
$./oc delete daemonset datadog-agent
$./oc apply -f create-agent.yaml
```

With this, you can then login to your pod and see that you are no longer `I have no username!` and are now `root`:

```
$./oc get pods
$./oc exec <AGENTNAME> -it bash
```
