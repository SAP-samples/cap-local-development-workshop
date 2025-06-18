# Exercise 00 - Setting up

This workshop is all about local development, but we want to ensure that everyone has the same experience regardless of the machine, operating system and admin rights to install software. So there are different options you can take to set up and get ready to run through the exercises.

If you have everything set up for CAP Node.js development already, including Node.js 22 or 24, the latest release of CAP Node.js ([9.0.0+]), and an editor you're comfortable using, then you're all set. If not, then we have various options for you from which to choose.

## Option 1 - dev container and VS Code installed locally

If you already have or can install VS Code on your machine, then this option may be for you.

- Ensure you have installed the [Dev Containers] extension in VS Code.
- Clone this repo `git clone https://github.com/SAP-samples/cap-local-development-workshop`.
- Open the directory containing the clone with VS Code `code cap-local-development-workshop`.
- Choose to "Reopen in Container" when asked:
  ![reopen in container dialogue box](assets/vscode-reopen-in-container.png)
- Open up a terminal and check the CAP version (should be [9.0.0+]) with `cds v`:
  ![running cds v in a terminal prompt](assets/vscode-shell-cds-version.png)

[9.0.0+]: https://cap.cloud.sap/docs/releases/may25
[Dev Containers]: https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers
