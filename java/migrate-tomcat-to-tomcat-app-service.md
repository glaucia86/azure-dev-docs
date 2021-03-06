---
title: Migrate Tomcat applications to Tomcat on Azure App Service
description: This guide describes what you should be aware of when you want to migrate an existing Tomcat application to run on Azure App Service using Tomcat.
author: yevster
ms.author: yebronsh
ms.topic: conceptual
ms.date: 1/20/2020
---

# Migrate Tomcat applications to Tomcat on Azure App Service

This guide describes what you should be aware of when you want to migrate an existing Tomcat application to run on Azure App Service using Tomcat 8.5 or 9.0.

## Before you start

If you can't meet any of the pre-migration requirements, see the following companion migration guides:

* [Migrate Tomcat applications to containers on Azure Kubernetes Service](migrate-tomcat-to-containers-on-azure-kubernetes-service.md)
* Migrate Tomcat Applications to Azure Virtual Machines (forthcoming)

## Pre-migration steps

* [Switch to a supported platform](#switch-to-a-supported-platform)
* [Inventory external resources](#inventory-external-resources)
* [Inventory secrets](#inventory-secrets)
* [Inventory persistence usage](#inventory-persistence-usage)
* [Special cases](#special-cases)

### Switch to a supported platform

App Service offers specific versions of Tomcat on specific versions of Java. To ensure compatibility, migrate your application to one of the supported versions of Tomcat and Java in its current environment prior to proceeding with any of the remaining steps. Be sure to fully test the resulting configuration. Use [Red Hat Enterprise Linux 8](https://portal.azure.com/#create/RedHat.RedHatEnterpriseLinux80-ARM) as the operating system in such tests.

#### Java

> [!NOTE]
> This validation is especially important if your current server is running on an unsupported JDK (such as Oracle JDK or IBM OpenJ9).

To obtain your current Java version, sign in to your production server and run the following command:

```bash
java -version
```

To obtain the current version used by Azure App Service, download [Zulu 8](https://www.azul.com/downloads/zulu-community/?&version=java-8-lts&os=&os=linux&architecture=x86-64-bit&package=jdk) if you intend to use the Java 8 runtime or [Zulu 11](https://www.azul.com/downloads/zulu-community/?&version=java-11-lts&os=&os=linux&architecture=x86-64-bit&package=jdk) if you intend to use the Java 11 runtime.

#### Tomcat

To determine your current Tomcat version, sign in to your production server and run the following command:

```bash
${CATALINA_HOME}/bin/version.sh
```

To obtain the current version used by Azure App Service, download [Tomcat 8.5](https://tomcat.apache.org/download-80.cgi#8.5.50) or [Tomcat 9](https://tomcat.apache.org/download-90.cgi), depending on which version you plan to use in Azure App Service.

[!INCLUDE [inventory-external-resources](includes/migration/inventory-external-resources.md)]

[!INCLUDE [inventory-secrets](includes/migration/inventory-secrets.md)]

[!INCLUDE [inventory-persistence-usage](includes/migration/inventory-persistence-usage.md)]

<!-- App-Service-specific addendum to inventory-persistence-usage -->
#### Dynamic or internal content

For files that are frequently written and read by your application (such as temporary data files), or static files that are visible only to your application, you can mount Azure Storage into the App Service file system. For more information, see [Serve content from Azure Storage in App Service on Linux](/azure/app-service/containers/how-to-serve-content-from-azure-storage).

### Identify session persistence mechanism

To identify the session persistence manager in use, inspect the *context.xml* files in your application and Tomcat configuration. Look for the `<Manager>` element, and then note the value of the `className` attribute.

Tomcat's built-in [PersistentManager](https://tomcat.apache.org/tomcat-8.5-doc/config/manager.html) implementations, such as  [StandardManager](https://tomcat.apache.org/tomcat-8.5-doc/config/manager.html#Standard_Implementation) or [FileStore](https://tomcat.apache.org/tomcat-8.5-doc/config/manager.html#Nested_Components) aren't designed for use with a distributed, scaled platform such as App Service. Because App Service may load balance among several instances and transparently restart any instance at any time, persisting mutable state to a file system isn't recommended.

If session persistence is required, you'll need to use an alternate `PersistentManager` implementation that will write to an external data store, such as Pivotal Session Manager with Redis Cache. For more information, see [Use Redis as a session cache with Tomcat](/azure/app-service/containers/configure-language-java#use-redis-as-a-session-cache-with-tomcat).

### Special cases

Certain production scenarios may require additional changes or impose additional limitations. While such scenarios can be infrequent, it is important to ensure that they are either inapplicable to your application or correctly resolved.

#### Determine whether application relies on scheduled jobs

Scheduled jobs, such as Quartz Scheduler tasks or cron jobs, can't be used with App Service. App Service will not prevent you from deploying an application containing scheduled tasks internally. However, if your application is scaled out, the same scheduled job may run more than once per scheduled period. This situation can lead to unintended consequences.

Inventory any scheduled jobs, inside or outside the application server.

#### Determine whether your application contains OS-specific code

If your application contains any code with dependencies on the host OS, then you'll need to refactor it to remove those dependencies. For example, you may need to replace any use of `/` or `\` in file system paths with [`File.Separator`](https://docs.oracle.com/javase/8/docs/api/java/io/File.html#separator) or [`Paths.get`](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Paths.html#get-java.lang.String-java.lang.String...-).

#### Determine whether Tomcat clustering is used

[Tomcat clustering](https://tomcat.apache.org/tomcat-8.5-doc/cluster-howto.html) isn't supported on Azure App Service. Instead, you can configure and manage scaling and load balancing through Azure App Service without Tomcat-specific functionality. You can persist session state to an alternate location to make it available across replicas. For more information, see [Identify session persistence mechanism](#identify-session-persistence-mechanism).

To determine whether your application uses clustering, look for the `<Cluster>` element inside the `<Host>` or `<Engine>` elements in the *server.xml* file.

#### Identify all outside processes/daemons running on the production server(s)

You will need to migrate elsewhere or eliminate any processes running outside of Application Server, such as monitoring daemons.

#### Determine whether non-HTTP connectors are used

App Service supports only a single HTTP connector. If your application requires additional connectors, such as the AJP connector, don't use App Service.

To identify HTTP connectors used by your application, look for `<Connector>` elements inside the *server.xml* file in your Tomcat configuration.

#### Determine whether MemoryRealm is used

[MemoryRealm](https://tomcat.apache.org/tomcat-8.5-doc/api/org/apache/catalina/realm/MemoryRealm.html) requires a persisted XML file. On Azure AppService, you will need to upload this file to the */home* directory or a subdirectory thereof or to mounted storage. You will have to modify the `pathName` parameter accordingly.

To determine whether `MemoryRealm` is currently used, inspect your *server.xml* and *context.xml* files and search for `<Realm>` elements where the `className` attribute is set to `org.apache.catalina.realm.MemoryRealm`.

#### Determine whether SSL session tracking is used

App Service performs session offloading outside of the Tomcat runtime. Therefore, you can't use [SSL session tracking](https://tomcat.apache.org/tomcat-8.5-doc/servletapi/javax/servlet/SessionTrackingMode.html#SSL). Use a different session tracking mode instead (`COOKIE` or `URL`). If you need SSL session tracking, don't use App Service.

#### Determine whether AccessLogValve is used

If you use [AccessLogValve](https://tomcat.apache.org/tomcat-8.5-doc/api/org/apache/catalina/valves/AccessLogValve.html), you should set the `directory` parameter to `/home/LogFiles` or a subdirectory thereof.

## Migration

### Parametrize the configuration

In the pre-migration you'll likely have identified secrets and external dependencies, such as datasources, in *server.xml* and *context.xml* files. For each item thus identified, replace any username, password, connection string or URL with an environment variable.

For example, suppose the *context.xml* file contains the following element:

```xml
<Resource
    name="jdbc/dbconnection"
    type="javax.sql.DataSource"
    url="jdbc:postgresql://postgresdb.contoso.com/wickedsecret?ssl=true"
    driverClassName="org.postgresql.Driver"
    username="postgres"
    password="t00secure2gue$$"
/>
```

In this case, you could change it as shown in the following example:

```xml
<Resource
    name="jdbc/dbconnection"
    type="javax.sql.DataSource"
    url="${postgresdb.connectionString}"
    driverClassName="org.postgresql.Driver"
    username="${postgresdb.username}"
    password="${postgresdb.password}"
/>
```

### Provision an App Service plan

From the list of available service plans at [App Service pricing](https://azure.microsoft.com/pricing/details/app-service/linux/), select the plan whose specifications meet or exceed those of the current production hardware.

> [!NOTE]
> If you plan to run staging/canary deployments or use deployment slots, the App Service plan must include that additional capacity. We recommend using Premium or higher plans for Java applications. For more information, see [Set up staging environments in Azure App Service](/azure/app-service/deploy-staging-slots).

Then, create the App Service plan. For more information, see [Manage an App Service plan in Azure](/azure/app-service/app-service-plan-manage).

### Create and deploy Web App(s)

You'll need to create a Web App on your App Service Plan for every WAR file deployed to your Tomcat server.

> [!NOTE]
> While it's possible to deploy multiple WAR files to a single web app, this is highly undesirable. Deploying multiple WAR files to a single web app prevents each application from scaling according to its own usage demands. It also adds complexity to subsequent deployment pipelines. If multiple applications need to be available on a single URL, consider using a routing solution such as [Azure Application Gateway](/azure/application-gateway/).

#### Maven applications

If your application is built from a Maven POM file, [use the Webapp plugin for Maven](/azure/app-service/containers/quickstart-java#configure-the-maven-plugin) to create the Web App and deploy your application.

#### Non-Maven applications

If you can't use the Maven plugin, you'll need to provision the Web App through other mechanisms, such as:

* [Azure portal](https://portal.azure.com/#create/Microsoft.WebSite)
* [Azure CLI](/cli/azure/webapp?view=azure-cli-latest#az-webapp-create)
* [Azure PowerShell](/powershell/module/az.websites/new-azwebapp)

Once the Web App has been created, use one of the [available deployment mechanisms](/azure/app-service/deploy-zip) to deploy your application.

### Migrate JVM runtime options

If your application requires specific runtime options, [use the most appropriate mechanism to specify them](/azure/app-service/containers/configure-language-java#set-java-runtime-options).

### Populate secrets

Use Application Settings to store any secrets specific to your application. If you intend to use the same secret(s) among multiple applications or require fine-grained access policies and audit capabilities, [use Azure Key Vault](/azure/app-service/containers/configure-language-java#use-keyvault-references) instead.

### Configure custom domain and SSL

If your application will be visible on a custom domain, you'll need to [map your web application to it](/azure/app-service/app-service-web-tutorial-custom-domain).

You'll then need to [bind the SSL certificate for that domain to your App Service Web App](/azure/app-service/app-service-web-tutorial-custom-ssl).

### Migrate data sources, libraries, and JNDI resources

Follow [these steps to migrate data sources](/azure/app-service/containers/configure-language-java#tomcat).

Migrate any additional server-level classpath dependencies by following [the same steps as for data source jar files](/azure/app-service/containers/configure-language-java#finalize-configuration).

Migrate any additional [Shared server-level JDNI resources](/azure/app-service/containers/configure-language-java#shared-server-level-resources).

> [!NOTE]
> If you're following the recommended architecture of one WAR per webapp, consider migrating server-level classpath libraries and JNDI resources into your application. This will significantly simplify component governance and change management.

### Migrate remaining configuration

Upon completing the preceding section, you should have your customizable server configuration in */home/tomcat/conf*.

Complete the migration by copying any additional configuration (such as [realms](https://tomcat.apache.org/tomcat-8.5-doc/config/realm.html), [JASPIC](https://tomcat.apache.org/tomcat-8.5-doc/config/jaspic.html))

### Migrate scheduled jobs

To execute scheduled jobs on Azure, consider using [Azure Functions with a Timer Trigger](/azure/azure-functions/functions-bindings-timer). You don't need to migrate the job code itself into a function. The function can simply invoke a URL in your application to trigger the job.

Alternatively, you can create a [Logic app](/azure/logic-apps/logic-apps-overview) with a [Recurrence trigger](/azure/logic-apps/tutorial-build-schedule-recurring-logic-app-workflow#add-the-recurrence-trigger) to invoke the URL without writing any code outside your application.

> [!NOTE]
> To prevent malicious use, you'll likely need to ensure that the job invocation endpoint requires credentials. In this case, the trigger function will need to provide the credentials.

### Restart and smoke-test

Finally, you'll need to restart your Web App to apply all configuration changes. Upon completion of the restart, verify that your application is running correctly.

## Post-migration steps

Now that you have your application migrated to Azure App Service you should verify that it works as you expect. Once you've done that we have some recommendations for you that can make your application more Cloud native.

### Recommendations

1. If you opted to use the */home* directory for file storage, consider [replacing it with Azure Storage](/azure/app-service/containers/how-to-serve-content-from-azure-storage).

1. If you have configuration in the */home* directory which contains connection strings, SSL keys, and other secret information, consider using a combination of [Azure Key Vault](/azure/app-service/app-service-key-vault-references) and/or [parameter injection with application settings](/azure/app-service/configure-common#configure-app-settings) where possible.

1. Consider [using Deployment Slots](/azure/app-service/deploy-staging-slots) for reliable deployments with zero downtime.

1. Design and implement a DevOps strategy. In order to maintain reliability while increasing your development velocity, consider [automating deployments and testing with Azure Pipelines](/azure/devops/pipelines/ecosystems/java-webapp). If using Deployment Slots, you can [automate deployment to a slot](/azure/devops/pipelines/targets/webapp?view=azure-devops&tabs=yaml#deploy-to-a-slot) and the subsequent slot swap.

1. Design and implement a business continuity and disaster recovery strategy. For mission-critical applications, consider a [multi-region deployment architecture](/azure/architecture/reference-architectures/app-service-web-app/multi-region).
