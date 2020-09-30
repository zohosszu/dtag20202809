## Setting up Client Tools

Now after we have installed our Openshift 4 cluster we need to install the Openshift Client Tools.

After completing this section, you should be able to:

* Locate the binaries for the OpenShift Enterprise command-line
  interface (OC CLI)
* Install the OpenShift CLI tools.
* CLI basic configuration

### Locating the binaries

The OSE CLI exposes commands for managing applications, as well as
lower-level tools to interact with each component of a system. The
binaries for Mac, Windows, and Linux are available for download from the
Red Hat Customer Portal via the following link, selecting the same exact
version of your cluster:

https://access.redhat.com/downloads/content/290/ver=4.5/rhel---8/4.5.6/x86_64/product-software

The CLI is provided as compressed files that can be decompressed to any
directory. In order to make it simple for any user to access the OSE
CLI, it is recommended that it is made available in a directory mapped
to the environment variable called `PATH` from the OS.

It is possible as well and easier to install the client tools from openshift.com.

1. OSX and Linux:
   
   * OSX
     https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.5.6/openshift-client-mac-4.5.6.tar.gz
   
   * Linux
     https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.5.6/openshift-client-linux-4.5.6.tar.gz

Copy the binary to the `/usr/local/bin` directory, or one of the
paths listed in the `PATH` environment variable.

2. Windows:
   
   * https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.5.6/openshift-client-windows-4.5.6.zip

Use `oc.exe` to open an OpenShift shell. If you getting error from
running oc, go to http://git-scm.com to download git bash for Windows (during
installation you need to specify in the selection to integrate with the
command prompt)

> **Important notes**
> Download and install Notepad++ and install the JSON plugin or use
> http://jsonlint.com/ to edit and validate JSON.
> http://ammonsonline.com/formatted-json-in-notepad/
> Configure your default editor to be Notepad++*

### CLI basic configuration*

The easiest way to initially setup the OpenShift CLI is to use the
`oc login` command. It'll interactively ask you a server URL, username
and password. The information is automatically saved in a CLI
configuration file that is then used for subsequent commands.

To login to a remote server use:

```
oc login -u USERNAME -p PASSWORD https://api.ocp4.hX.rhaw.io:6443```

> NOTE: Username and password have been created earlier in the authentication step.
