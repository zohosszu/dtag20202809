## Create an App from a Container Image

In thischapter we will learn how to create a new project on OpenShift and
how to create an application from an existing docker image using Web UI.

### Add a new project

> IMPORTANT: Please replace *Username* with your username

- login to web UI via console-openshift-console.apps.ocp4.hX.rhaw.io

- Use the same username and password that assigned to you

- On the left-hand side menu, select `Home` and then select `Project`

- Click `Create Project`

- Enter *web-terminal-username* as name of the project. The display name and description is optional.

- Click `Create`

### Deploy an Image

- Make sure on the top left side that you are in the 'Developer' view and not in the 'Administrator' one.

- Click on the 'Topology' side-bar entry, or click on the left '+Add' side bar.

- Select 'Container Image'

- Enter `quay.io/openshiftlabs/workshop-terminal:2.4.0` in the `Image Name` box,
  noting that there's no `http://`, enter exactly as shown here,
  without the quotes

- Select the magnifying glass icon to the right of the box

- Use the default `Name` for the deployment

- Uncheck the tick-box 'Create a route for the application' so that you can create the route in a follow-up step.

- Scroll down a bit, click on the 'Deployment Configuration' link and provide an environment variable.

- Add `OC_VERSION` and `4.2` as `Environment Variable` section at the bottom on the page.

- Click `Create`

- Click on the `workshop-terminal` icon in the 'Topology' view to see the deployment details

- click the tab `Resources` to watch the deployment of the pod(s)

- Navigate to the `Advanced` --> `Events` side tab to check status on each events.

Events in OpenShift Container Platform are modeled based on events that happen
to API objects in an OpenShift Container Platform cluster. Events allow OpenShift
Container Platform to record information about real-world events in a resource-
agnostic manner.

Next we need to create a route so we can access this application from the outside of OpenShift (i.e. from the internet).

- Switch to the 'Administrator' view on the top left side

- Navigate to `Networking` --> `Routes`

- Click `Create Route` in the top-left

- Enter `workshop-terminal` for the name of the route, leave the hostname and path the default

- Select `workshop-terminal` from the list of bastion in the `Service` menu

- Select the only port in the `Target Port` drop down (should be port 10080 --> 10080), like this:

- Click `Create` at the bottom and it will create the route.

- You will now be presented with a pane that shows the overview of the route:

**NOTE:** the same steps allow you to create multiple routes for the same service.

### Accessing the Terminal

- In the top right hand side of the route details page, you will see the `LOCATION`
  which will be the routable URL that will provide access to our workshop terminal.
- Click on the link under `LOCATION` and you will see the in-browser terminal
  session that we can use (if preferred, or mandated due to connectivity issues):

### Setup OC CLI in Web Terminal

- Execute the following in the terminal:

```
 wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.5.6/openshift-client-linux-4.5.6.tar.gz
 tar zvxf openshift-client-linux-4.5.6.tar.gz
 mv oc /opt/app-root/bin/
 oc version
```

> NOTE: If normal cut/paste does not work, you can try to use browser's edit menu for cut/paste.

### Login to a Remote Server

```
$ oc login https://api.ocp4.hX.rhaw.io:6443
```

```
NOTE: Username and password will be provided by your instructor.
```

Congratulations!! You now know how to create a project, an application
using an external docker image and navigate around. You also install OC CLI on
the web terminal to access the cluster via CLI.

## 
