# What is Helm

Helm is a package manager for Kubernetes designed to simplify application deployment and management. While Kubernetes is highly effective at orchestrating complex infrastructures, managing individual resources for complex applications can quickly become tedious.

Managing numerous YAML files individually can lead to operational errors, especially when updating configurations, such as increasing storage sizes from 20Gi to 2200Gi, across several files.

Even if all declarations are combined into a single file, the complexity increases as the file grows, making troubleshooting more challenging.

## Enter Helm

Helm treats related resources as a single application package, enabling you to deploy and manage your entire Kubernetes application with a single command.

To install a WordPress package, simply run:

```bash
helm install wordpress
```

To upgrade process:

```bash
helm upgrade wordpress
```

If issues occur during an upgrade, Helm’s rollback feature allows you to revert to a previous, stable release:

```bash
helm rollback wordpress <release_number>
```

When it's time to remove the application, Helm ensures that all associated Kubernetes objects are tracked and deleted automatically:

```bash
helm uninstall wordpress
```

When you run the Helm command without parameters, it displays a comprehensive help menu that provides a quick reference for common actions. For example:

```bash
helm --help

The Kubernetes package manager

Common actions for Helm:
- helm search:     search for charts
- helm pull:       download a chart to your local directory to view
- helm install:    upload the chart to Kubernetes
- helm list:       list releases of charts

Usage:
  helm [command]

Available Commands:
  completion    generate autocompletion scripts for the specified shell
  create        create a new chart with the given name
  dependency    manage a chart's dependencies
  env           helm client environment information
  get           download extended information of a named release
  help          help about any command
  history       fetch release history
```

This help output serves as a handy reference; for instance, if you're considering rolling a release back to a previous version, you might initially search for a `helm restore` command. The help text quickly clarifies that the correct command for this operation is `helm rollback`.

## Exploring Subcommands

Helm provides a variety of subcommands to manage tasks, including repository-related functions. To see all commands related to chart repositories—such as adding, listing, or removing repositories execute:

```bash  theme={null}
helm repo --help

helm repo update --help
```

## Deploying a Real-World Application: WordPress

Helm simplifies the deployment of applications on Kubernetes. As a practical example, let's deploy a WordPress website using a prepackaged chart from the Bitnami repository. Follow these steps:

1. Add the Bitnami repository to your Helm configuration.
2. Deploy the WordPress application using the `helm install` command.

Here are the commands to execute:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install my-release bitnami/wordpress
```

To list summary information, including the current revision:

```bash
helm list
```

To shows detailed revision insights:
```bash
helm history my-release
```

If an upgrade introduces an unexpected change, you can easily roll back to a previous configuration:
```bash
helm rollback my-release 1
```

To upgrade to new version:
```bash
helm upgrade my-release
```

To completely remove the deployed WordPress website and its associated Kubernetes objects, use the uninstall command:

```bash
helm uninstall my-release
```

## Managing Helm Repositories

Managing chart repositories with Helm is intuitive. You can add, list, remove, or update repositories using the `helm repo` commands. For example, to view your current repositories and update their information, run:

```bash
helm repo list
```

This process is akin to updating package manager repositories on Linux systems, ensuring that your local cache of available charts is always up-to-date.

### Helm’s lifecycle management empowers you to:
* Install releases that package and manage multiple Kubernetes objects.
* Upgrade releases seamlessly in one command with robust tracking of configuration changes.
* Roll back to previous states by creating new revisions that reflect past stable configurations.
* Inspect release history to monitor and audit configuration modifications over time.
