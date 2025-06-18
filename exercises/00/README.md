# Exercise 00 - Setting up

This workshop is all about local development, but we want to ensure that everyone has the same experience regardless of the machine, operating system and admin rights to install software. So there are different options you can take to set up and get ready to run through the exercises.

If you have everything set up for CAP Node.js development already, including Node.js 22 or 24, the latest release of CAP Node.js ([9.0.0+]), and an editor you're comfortable using, then you're all set. If not, then we have various options for you from which to choose.

## Set up working environment

Here are the options.

### Option 1 - dev container and VS Code installed locally

If you already have or can install VS Code on your machine, then this option may be for you.

- Ensure you have installed the [Dev Containers] extension in VS Code.
- Clone this repo `git clone https://github.com/SAP-samples/cap-local-development-workshop`.
- Open the directory containing the clone with VS Code `code cap-local-development-workshop`.
- Choose to "Reopen in Container" when asked:
  ![reopen in container dialogue box](assets/vscode-reopen-in-container.png)

### Option 2 - dev container in a GitHub codespace

For our purposes, [GitHub codespaces] are essentially the same as a locally running container and VS Code ... but provided by GitHub and accessed via the browser.

- At the [home of this repo] on GitHub, use the "Code" button.
- Select the "Codespaces" tab.
- Choose to "Create codespace on main":
  ![github-create-codespace](assets/github-create-codespace.png)
- When the codespace is ready (in another browser tab), you're all set.

### Option 3 - dev space in SAP Business Application Studio

This option is very much similar to the previous two options, in that it provides a VS Code based development environment and (Linux-based) container. If you have a [trial account on the SAP Business Technology Platform], a [subscription to the SAP Business Application Studio], and the appropriate role collections assigned, then you can use this option.

- Go to the SAP Business Application Studio from your [trial landing page](https://account.hanatrial.ondemand.com/trial/).
- Choose to "Create Dev Space", giving it a name and selecting the "Full Stack Cloud Application" type:
  ![creating a dev space](assets/bas-create-dev-space.png)
- Once the dev space is started, enter it, use the "Clone from Git" option to clone this repo, and choose to open it when prompted:
  ![cloning this repo from git](assets/bas-clone-from-git.png)

## Check CAP Node.js version

Once you have everything set up, check that CAP Node.js is installed (it should be) by opening up a terminal (menu option "Terminal -> New Terminal") and running `cds v`. The version for `@sap/cds-dk` should be [9.0.0+]. Here's an example from a terminal prompt from Option 1, but regardless of the option you chose, it should look similar:

![running cds v in a terminal prompt](assets/vscode-shell-cds-version.png)

<!-- TODO: BAS dev spaces are still on CAP Node.js 8 - check in July -->

[GitHub codespaces]: https://github.com/features/codespaces
[9.0.0+]: https://cap.cloud.sap/docs/releases/may25
[Dev Containers]: https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers
[home of this repo]: https://github.com/SAP-samples/cap-local-development-workshop
[trial account on the SAP Business Technology Platform]: https://developers.sap.com/tutorials/hcp-create-trial-account.html
[subscription to the SAP Business Application Studio]: https://developers.sap.com/tutorials/appstudio-onboarding.html
[trial landing page]: https://account.hanatrial.ondemand.com/trial/
