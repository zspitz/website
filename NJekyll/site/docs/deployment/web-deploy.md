---
title: Deploying using Web Deploy
---

# Deploying using Web Deploy

Application can be automatically deployed during the build to staging environment and promoted to specific environment later from UI or through API. AppVeyor can deploy web application from Web Deploy package or zip archive.

* [Creating Web Deploy package](#creating-package)
	* [Automatic packaging](#automatic-packaging)
	* [Custom packaging](#custom-packaging)
* [Web Deploy deployment settings](#provider-settings)
* [Specifying correct connection details](#connection-details)
	*  [Remote agent service](#connection-details-agent)
	*  [Web Deploy Handler](#connection-details-handler)
	*  [Azure Web Sites](#connection-details-waws)
* [Web Deploy Parametrization](#web-deploy-parametrization)


<a id="creating-package"></a>
## Creating Web Deploy package



<a id="automatic-packaging"></a>
### Automatic packaging

During “MSBuild” phase AppVeyor automatically detects Web Application projects in the solution and publish them as Web Deploy packages to build artifacts:

![project-settings](/content/docs/deployment/images/web-deploy/project-settings.png)

> Here you may also set required configuration to choose proper web.config transformation.

AppVeyor uses the following command to create Web Deploy package:

    msbuild <web_app.csproj> /t:Package /p:PackageLocation=<web-deploy-package.zip> /p:PackageAsSingleFile=True



<a id="custom-packaging"></a>
### Custom packaging

If you build your app using a script or build framework like MSBuild, PSake or rake you can push the package to artifacts using the command:

    appveyor PuthArtifact <path-to-package.zip>

<a id="provider-settings"></a>
## Web Deploy deployment settings

Web deploy provider settings are specified on Deployment tab of project settings, `appveyor.yml` or on environment settings page.

* **Server** (`server`) - server name with remote agent service installed or URL of Web Deploy handler.
* **Website name** (`website`) - web site name to deploy to, e.g. Default Web Site or myserver.com
* **Username** (`username`)
* **Password** (`password`)
* **NTLM authentication** (`ntlm`) - NTLM authentication is primarily used by Remote Agent Service. Usually, IIS 7 and up web servers use Web Deploy Handler approach with Basic authentication.
* **Remove additional files at destination** (`remove_files`) - when set selected provider performs full content synchronization, i.e. deletes files at destination that don't exist in the package.
* **Skip directories** (`skip_dirs`) - semicolon list of regular expressions specifying the list of directories to skip while synchronizing web site contents, for example `\\App_data;\\uploads`.
* **Skip files** (`skip_files`) - semicolon list of regular expressions specifying the list of files to skip while synchronizing web site contents, for example `web.config` (all web.configs) or only the root config for MVC apps `^((?!Views).)*web\.config$` (thanks to [this blog post](http://www.keza.net/2011/11/15/skipping-mvc-web-config-files-with-msdeploy/)).
* **Take ASP.NET application offline during deployment** (`app_offline`) - places app_offline.htm page into the root of web application before sync to take app offline and then remove the page when deployment has finished.
* **Artifact to deploy** (`artifact`) - artifact name containing application package to deploy.

<a id="connection-details"></a>
## Specifying correct connection details

<a id="connection-details-agent"></a>
### Remote agent service

If you’re deploying to the remote agent service on the destination web server, you can specify the target computer name (for example, **TESTWEB1** or **TESTWEB1.fabrikam.net**), or you can specify the remote agent endpoint (for example, **http://TESTWEB1/MSDEPLOYAGENTSERVICE**). The deployment works the same way in each case.

Typically, NTLM should be enabled when deploying to remote agent service.

See  [Configuring Deployment Properties for a Target Environment](http://www.asp.net/web-forms/tutorials/deployment/configuring-server-environments-for-web-deployment/configuring-deployment-properties-for-a-target-environment) for more details.

<a id="connection-details-handler"></a>
### Web Deploy Handler

If you’re deploying to the Web Deploy Handler on the destination web server, you should specify the service endpoint and include the name of the IIS website as a query string parameter (for example, **https://STAGEWEB1:8172/MSDeploy.axd?site=DemoSite**).

Typically, Web Deployment Handler uses Basic authentication which is enabled by default.

See  [Configuring Deployment Properties for a Target Environment](http://www.asp.net/web-forms/tutorials/deployment/configuring-server-environments-for-web-deployment/configuring-deployment-properties-for-a-target-environment) for more details.

<a id="connection-details-waws"></a>
### Azure Web Sites

Open website dashboard in Azure Management Portal and download *publish profile*:

![publish-profile](/content/docs/deployment/images/web-deploy/waws-publish-profile.png) 

Specify the following deployment settings in AppVeyor:

* Server: `https://<publishUrl>/msdeploy.axd?site=<msdeploySite>`
* Website name: `<msdeploySite>`
* Username: `<userName>`
* Password: `<userPWD>`
* NTLM: disabled

Replace `<publishUrl>`, `<msdeploySite>`, `<userName>` and `<userPWD>` with values from downloaded publishing profile XML file like in example below:

![publish-profile-xml](/content/docs/deployment/images/web-deploy/waws-publish-profile-xml.png) 

External links:

* [Configuring web server to use remote agent](http://www.asp.net/web-forms/tutorials/deployment/configuring-server-environments-for-web-deployment/configuring-a-web-server-for-web-deploy-publishing-(remote-agent))
* [Configuring web service to use Web Deploy handler](http://www.asp.net/web-forms/tutorials/deployment/configuring-server-environments-for-web-deployment/configuring-a-web-server-for-web-deploy-publishing-(web-deploy-handler))


Configuring in `appveyor.yml`:

    deploy:
      provider: WebDeploy
      server: 
      website:
      username:
      password:
      ntlm: true|false
      remove_files: true|false
      artifact:

<a id="web-deploy-parametrization"></a>
## Web Deploy Parametrization

When deploying web application to different environments you don’t want to re-build application package every time with different configurations, but you want to deploy the same package (artifact) with some environment-specific settings configured during deployment. When using Web Deploy the problem can be easily solved by Web Deploy parametrization.

### Usage scenarios

Most common use cases for Web Deploy parametrization is updating node/attribute value in XML files or replacing a token in text files, for example:

- appSettings in `web.config`
- connection strings in `web.config`
- WCF endpoints
- Paths to log files
- Database name in SQL install script

### Parameters.xml
To enable Web Deploy parametrization add `parameters.xml` file in the root of your web application.

![vs-solution-explorer](/content/docs/deployment/images/web-deploy/vs-solution-explorer.png)

`Parameters.xml` contains the list of parameters required (or supported) by your Web Deploy package. In the example below we introduce two parameters - one to update path to log file in `appSettings` section of `web.config` and another one to set database name in SQL script.

Parameter element describes the name, default value and the places where and how this parameter must be applied.

`Parameters.xml` for our example:

	<?xml version="1.0" encoding="utf-8" ?>
	<parameters>
	  <parameter name="LogsPath" defaultValue="logs">
	    <parameterEntry kind="XmlFile" scope="\\web.config$" match="/configuration/appSettings/add[@key='LogsPath']/@value" />
	  </parameter>
	  <parameter name="DatabaseName">
	    <parameterEntry kind="TextFile" scope="\\Database\\install_db.sql$" match="@@database_name@@" />
	  </parameter>
	</parameters>

When Web Deploy package is built you can open it in the explorer and see `parameters.xml` in the root:

![webdeploy-package](/content/docs/deployment/images/web-deploy/webdeploy-package.png)

Resulting `parameters.xml` combines your custom parameters and system ones such as `IIS Web Application Name`. You don’t have to set `IIS Web Application Name` parameter explicitly - AppVeyor does that for you.

Read more about defining parameters: [http://technet.microsoft.com/en-us/library/dd569084(v=ws.10).aspx](http://technet.microsoft.com/en-us/library/dd569084(v=ws.10).aspx)

### Setting parameters during deployment

Web Deploy provider analyzes Web Deploy package and looks into environment variables to set parameter values with matching names.

When promoting specific build from Environment page you set variables on environment settings page:

![environment-variables](/content/docs/deployment/images/web-deploy/environment-variables.png)

When deploying during the build session environment variables are used instead. You can set build environment variables on Environment tab of project settings, `appveyor.yml` or programmatically during the build.

![project-environment-variables](/content/docs/deployment/images/web-deploy/project-environment-variables.png)

Variables defined during the build override those ones defined on Environment level.