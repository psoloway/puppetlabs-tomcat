#tomcat

####Table of Contents

1. [Overview](#overview)
2. [Module Description - What the module does and why it is useful](#module-description)
3. [Setup - The basics of getting started with tomcat](#setup)
    * [Setup requirements](#setup-requirements)
    * [Beginning with tomcat](#beginning-with-tomcat)
4. [Usage - Configuration options and additional functionality](#usage)
    * [I want to install Tomcat from a specific source.](#i-want-to-install-tomcat-from-a-specific-source)
    * [I want to run multiple copies of Tomcat on a single node.](#i-want-to-run-multiple-copies-of-tomcat-on-a-single-node)
    * [I want to deploy WAR files.](#i-want-to-deploy-are-files)
5. [Reference - An under-the-hood peek at what the module is doing and how](#reference)
    * [Classes](#classes)
    * [Defines](#defined-types)
    * [Parameters](#parameters)
        * [tomcat](#tomcat-1)
        * [tomcat::config::server](#tomcatconfigserver)
        * [tomcat::config::server::connector](#tomcatconfigserverconnector)
        * [tomcat::config::server::context](#tomcatconfigservercontext)
        * [tomcat::config::server::engine](#tomcatconfigserverengine)
        * [tomcat::config::server::host](#tomcatconfigserverhost)
        * [tomcat::config::server::listener](#tomcatconfigserverlistener)
        * [tomcat::config::server::realm](#tomcatconfigserverrealm)
        * [tomcat::config::server::service](#tomcatconfigserverservice)
        * [tomcat::config::server::tomcat_users](#tomcatconfigservertomcat_users)
        * [tomcat::config::server::valve](#tomcatconfigservervalve)
        * [tomcat::instance](#tomcatinstance)
        * [tomcat::service](#tomcatservice)
        * [tomcat::setenv::entry](#tomcatsetenventry)
        * [tomcat::war](#tomcatwar)
6. [Limitations - OS compatibility, etc.](#limitations)
7. [Development - Guide for contributing to the module](#development)

##Overview

The tomcat module lets you use Puppet to install, deploy, and configure Tomcat web services.

##Module Description

Tomcat is a Java Web service provider. The tomcat module lets you use Puppet to install Tomcat, manage its configuration file, and deploy web apps to it --- including support for multiple instances of Tomcat spanning multiple versions.

##Setup

###Setup requirements

The tomcat module requires puppetlabs-stdlib version 4.0 or newer. On Puppet Enterprise, you must meet this requirement before installing the module. To update stdlib, run:

~~~
puppet module upgrade puppetlabs-stdlib
~~~

###Beginning with tomcat

The simplest way to get Tomcat up and running with the tomcat module is to install the Tomcat package from EPEL,

~~~
class { 'tomcat':
  install_from_source => false,
}
class { 'epel': }->
tomcat::instance{ 'default':
  package_name => 'tomcat',
}->
~~~

and then start the service.

~~~
tomcat::service { 'default':
  use_jsvc     => false,
  use_init     => true,
  service_name => 'tomcat',
}
~~~

##Usage

###I want to install Tomcat from a specific source

To download Tomcat from a specific source and then start the service,

~~~
class { 'tomcat': }
class { 'java': }
tomcat::instance { 'test':
  source_url => 'http://mirror.nexcess.net/apache/tomcat/tomcat-8/v8.0.8/bin/apache-tomcat-8.0.8.tar.gz'
}->
tomcat::service { 'default': }
~~~

###I want to run multiple copies of Tomcat on a single node

~~~
class { 'tomcat': }
class { 'java': }

tomcat::instance { 'tomcat8':
  catalina_base => '/opt/apache-tomcat/tomcat8',
  source_url    => 'http://mirror.nexcess.net/apache/tomcat/tomcat-8/v8.0.8/bin/apache-tomcat-8.0.8.tar.gz'
}->
tomcat::service { 'default':
  catalina_base => '/opt/apache-tomcat/tomcat8',
}

tomcat::instance { 'tomcat6':
  source_url    => 'http://apache.mirror.quintex.com/tomcat/tomcat-6/v6.0.41/bin/apache-tomcat-6.0.41.tar.gz',
  catalina_base => '/opt/apache-tomcat/tomcat6',
}->
tomcat::config::server { 'tomcat6':
  catalina_base => '/opt/apache-tomcat/tomcat6',
  port          => '8105',
}->
tomcat::config::server::connector { 'tomcat6-http':
  catalina_base         => '/opt/apache-tomcat/tomcat6',
  port                  => '8180',
  protocol              => 'HTTP/1.1',
  additional_attributes => {
    'redirectPort' => '8543'
  },
}->
tomcat::config::server::connector { 'tomcat6-ajp':
  catalina_base         => '/opt/apache-tomcat/tomcat6',
  port                  => '8109',
  protocol              => 'AJP/1.3',
  additional_attributes => {
    'redirectPort' => '8543'
  },
}->
tomcat::service { 'tomcat6':
  catalina_base => '/opt/apache-tomcat/tomcat6'
~~~

###I want to deploy WAR files

~~~
tomcat::war { 'sample.war':
  catalina_base => '/opt/apache-tomcat/tomcat8',
  war_source => '/opt/apache-tomcat/tomcat8/webapps/docs/appdev/sample/sample.war',
}
~~~

The name of the WAR file must end with '.war'.

The `war_source` can be a local path, or a `puppet:///`, `http://`, or `ftp://` URL.

###I want to change my configuration

Tomcat does not restart if you update its configuration, unless you supply a [`notify` metaparameter](https://docs.puppetlabs.com/learning/ordering.html#notify-and-subscribe).

To remove a connector, for instance, start with a manifest like this:

~~~
tomcat::config::server::connector { 'tomcat8-jsvc':
  catalina_base         => '/opt/apache-tomcat/tomcat8-jsvc',
  port                  => '80',
  protocol              => 'HTTP/1.1',
  additional_attributes => {
    'redirectPort' => '443'
  },
  connector_ensure => 'present'
}
~~~

Then set `connector_ensure` to 'absent', and set `notify` to the service resource.

~~~
tomcat::config::server::connector { 'tomcat8-jsvc':
  catalina_base         => '/opt/apache-tomcat/tomcat8-jsvc',
  port                  => '80',
  protocol              => 'HTTP/1.1',
  additional_attributes => {
    'redirectPort' => '443'
  },
  connector_ensure => 'present'
  notify => Tomcat::Service['jsvc-default'],
}
~~~

###I want to manage a Connector or Realm that already exists

Describe the Realm or HTTP Connector element using `tomcat::config::server::realm` or `tomcat::config::server::connector`, and set `purge_realms` or `purge_connectors` to 'true'.

~~~
tomcat::config::server::realm { 'org.apache.catalina.realm.LockOutRealm':
  realm_ensure => 'present',
  purge_realms => true,
}
~~~

Puppet removes any existing Connectors or Realms and leaves only the ones you've specified.

##Reference

###Classes

####Public Classes

* `tomcat`: Main class. Manages the installation and configuration of Tomcat.

####Private Classes

* `tomcat::params`: Manages Tomcat parameters.

###Defines

####Public Defines

* `tomcat::config::server`: Configures attributes for the [Server element](http://tomcat.apache.org/tomcat-8.0-doc/config/server.html) in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::config::server::connector`: Configures [Connector elements](http://tomcat.apache.org/tomcat-8.0-doc/connectors.html) in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::config::server::context`: Configures [Context elements](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::config::server::engine`: Configures [Engine elements](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Introduction) in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::config::server::host`: Configures [Host elements](http://tomcat.apache.org/tomcat-8.0-doc/config/host.html) in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::config::server::listener`: Configures [Listener elements](http://tomcat.apache.org/tomcat-8.0-doc/config/listeners.html) in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::config::server::realm`: Configures [Realm elements](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html) in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::config::server::service`: Configures a [Service element](http://tomcat.apache.org/tomcat-8.0-doc/config/service.html) element nested in the `Server` element in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::config::server::tomcat_users`: Configures user and role elements for [UserDatabaseRealm] (http://tomcat.apache.org/tomcat-8.0-doc/realm-howto.html#UserDatabaseRealm) or [MemoryRealm] (http://tomcat.apache.org/tomcat-8.0-doc/realm-howto.html#MemoryRealm) in `$CATALINA_BASE/conf/tomcat-users.xml` or any other specified file.
* `tomcat::config::server::valve`: Configures a [Valve](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html) element in `$CATALINA_BASE/conf/server.xml`.
* `tomcat::instance`: Installs a Tomcat instance.
* `tomcat::service`: Provides Tomcat service management.
* `tomcat::setenv::entry`: Adds an entry to a Tomcat configuration file (e.g., `setenv.sh` or `/etc/sysconfig/tomcat`).
* `tomcat::war`:  Manages the deployment of WAR files.

####Private Defines

* `tomcat::instance::package`: Installs Tomcat from a package.
* `tomcat::instance::source`: Installs Tomcat from source.

###Parameters

####tomcat

#####`catalina_home`

Specifies the root directory of the Tomcat installation.

#####`user`

Sets the user to run Tomcat as.

#####`group`

Sets the group to run Tomcat as.

#####`install_from_source`

*Optional.* Specifies whether to install Tomcat from source. Valid options: 'true' and 'false'. Default: 'true'.

#####`purge_connectors`

*Optional.* Specifies whether to purge any unmanaged Connector elements from the configuration file. Valid options: 'true' and 'false'. Default: 'false'.

#####`purge_realms`

*Optional.* Specifies whether to purge any unmanaged Realm elements from the configuration file. Valid options: 'true' and 'false'. Default: 'false'.

#####`manage_user`

*Optional.* Specifies whether to create the specified user, if it doesn't exist. Uses Puppet's native [`user` resource type](https://docs.puppetlabs.com/references/latest/type.html#user) with default parameters. Valid options: 'true' and 'false'. Default: 'true'.

#####`manage_group`

*Optional.* Determines whether to create the specified group, if it doesn't exist. Uses Puppet's native [`group` resource type](https://docs.puppetlabs.com/references/latest/type.html#group) with default parameters. Valid options: 'true' and 'false'. Default: 'true'.

####tomcat::config::server

#####`catalina_base`

Specifies the base directory of the Tomcat installation to manage. Valid options: a string containing an absolute path to an existing directory.

#####`class_name`

*Optional.* Specifies the Java class name of a server implementation to use. Maps to the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/server.html#Common_Attributes) in the configuration file. Valid options: a string containing a Java class name.

#####`class_name_ensure`

Specifies whether the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/server.html#Common_Attributes) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`address`

*Optional.* Specifies a TCP/IP address to listen on for the shutdown command. Maps to the [address XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/server.html#Common_Attributes).

#####`address_ensure`

Specifies whether the [address XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/server.html#Common_Attributes) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`port`

Specifies a port to listen on for the designated shutdown command.

#####`shutdown`

Designates a command to shut down Tomcat when received through the specified address and port.

####tomcat::config::server::connector

#####`catalina_base`

Specifies the base directory of the Tomcat installation to manage.

#####`connector_ensure`

Specifies whether the [Connector XML element](http://tomcat.apache.org/tomcat-8.0-doc/connectors.html) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`port`

*Required, unless `connector_ensure` is set to 'false'.* Sets a TCP port to create a server socket on. Maps to the [port XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html#Common_Attributes).

#####`protocol`

Specifies a protocol to use for handling incoming traffic. Maps to the [protocol XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html#Common_Attributes).

#####`parent_service`

*Optional.* Specifies which Service element the Connector should nest under. Default: 'Catalina'.

#####`additional_attributes`

*Optional.* Specifies any further attributes to add to the Connector. Valid options: a hash of '< attribute >' => '< value >' pairs.

#####`attributes_to_remove`

*Optional.* Specifies any attributes to remove from the Connector. Valid options: a hash of '< attribute >' => '< value >' pairs.

####tomcat::config::server::context

#####`catalina_base`

Specifies the base directory of the Tomcat installation to manage.

#####`context_ensure`

Specifies whether the [Context XML element](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`doc_base`

Specifies a Document Base (or Context Root) directory or archive file. Maps to the [docBase XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Common_Attributes). Valid options: a string containing a path (either an absolute path or a path relative to the appBase directory of the owning Host).

#####`parent_service`

*Optional.* Specifies which Service XML element the Context should nest under. Default: 'Catalina'.

#####`parent_engine`

Specifies which Engine element the Context should nest under. Only valid if `parent_host` is specified. Valid options: a string containing the name attribute of the Engine.

#####`parent_host`

Specifies which Host element the Context should nest under. Valid options: a string containing the name attribute for the Host.

#####`additional_attributes`

*Optional.* Specifies any further attributes to add to the Context. Valid options: a hash of '< attribute >' => '< value >' pairs.

#####`attributes_to_remove`

*Optional.* Specifies any attributes to remove from the Context.  Valid options: a hash of '< attribute >' => '< value >' pairs.

####tomcat::config::server::engine

#####`default_host`

*Required.* Specifies a host to handle any requests directed to hostnames that exist on the server but are not defined in this configuration file. Maps to the [defaultHost XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes) for the Engine. Valid options: a string containing a hostname.

#####`catalina_base`

Specifies the base directory of the Tomcat installation to manage.

#####`background_processor_delay`

*Optional.* Determines the delay between invoking the backgroundProcess method on this engine and its child containers. Maps to the [backgroundProcessorDelay XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes). Valid options: an integer, in seconds.

#####`background_processor_delay_ensure`

Specifies whether the [backgroundProcessorDelay XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`class_name`

*Optional.* Specifies the Java class name of a server implementation to use. Maps to the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes). Valid options: a string containing a Java class name.

#####`class_name_ensure`

Specifies whether the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`engine_name`

*Optional.* Specifies the logical name of the Engine, used in log and error messages. Maps to the [name XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes). Valid options: a string. Default: the '[name]' passed in your define.

#####`jvm_route`

*Optional.* Specifies an identifier to be used in load balancing scenarios to enable session affinity. Maps to the [jvmRoute XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes).

#####`jvm_route_ensure`

Specifies whether the [jvmRoute XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`parent_service`

*Optional.* Specifies which Service element the Engine should nest under. Default: 'Catalina'.

#####`start_stop_threads`

*Optional.* Sets how many threads the Engine should use to start child Host elements in parallel. Maps to the [startStopThreads XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes).

#####`start_stop_threads_ensure`

Specifies whether the [startStopThreads XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/engine.html#Common_Attributes) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

####tomcat::config::server::host

#####`app_base`

*Required, unless [`host_ensure`](#host_ensure) is set to 'false' or 'absent'.* Specifies the Application Base directory for the virtual host. Maps to the [appBase XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/host.html#Common_Attributes).

#####`catalina_base`

Specifies the base directory of the Tomcat installation to manage.

#####`host_ensure`

Specifies whether the virtual host (the [Host XML element](http://tomcat.apache.org/tomcat-8.0-doc/config/host.html#Introduction)) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`host_name`

*Optional.* Specifies the network name of the virtual host, as registered on your DNS server. Maps to the [name XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/host.html#Common_Attributes).  Default: the '[name]' passed in your define.

#####`parent_service`

Specifies which Service element the Host should nest under.  Default: 'Catalina'.

#####`additional_attributes`

*Optional.* Specifies any further attributes to add to the Host. Valid options: a hash of '< attribute >' => '< value >' pairs.

#####`attributes_to_remove`

*Optional.* Specifies any attributes to remove from the Host. Valid options: an array of '< attribute >' => '< value >' pairs.

####tomcat::config::server::listener

#####`catalina_base`

Specifies the base directory of the Tomcat installation.

#####`listener_ensure`

Specifies whether the [Listener XML element](http://tomcat.apache.org/tomcat-8.0-doc/config/listeners.html) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`class_name`

Specifies the Java class name of a server implementation to use. Maps to the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/listeners.html#Common_Attributes) of a Listener Element. Valid options: a string containing a Java class name.

#####`parent_service`

Specifies which Service element the Listener should nest under. Only valid if `parent_engine` or `parent_host` is specified. Default: 'Catalina'.

#####`parent_engine`

Specifies which Engine element this Listener should nest under.  Valid options: a string containing the name attribute of the Engine.

#####`parent_host`

Specifies which Host element this Listener should nest under. Valid options: a string containing the name attribute of the Host.

#####`additional_attributes`

*Optional.* Specifies any further attributes to add to the Listener. Valid options: a hash of '< attribute >' => '< value >' pairs.

#####`attributes_to_remove`

*Optional.* Specifies any attributes to remove from the Listener. Valid options: a hash of '< attribute >' => '< value >' pairs.

####tomcat::config::server::realm

#####`catalina_base`

*Optional.* Specifies the base directory of the Tomcat installation. Valid options: Specifies the Java class name of the Realm implementation to use. Maps to the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html#Common_Attributes).

#####`class_name`

*Optional.* Specifies the Java class name of a Realm implementation to use. Maps to the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/realm.html#Common_Attributes). Valid options: a string containing a Java class name. Default: the '[name]' passed in your define.

#####`realm_ensure`

Specifies whether the Realm element should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`parent_service`

Specifies which Service element this Realm element should nest under. Default: 'Catalina'.

#####`parent_engine`

*Optional.* Specifies which Engine element this Realm should nest under. Valid options: a string containing the name attribute of the Engine. Default: 'Catalina'.

#####`parent_host`

*Optional.* Specifies which Host element this Realm should nest under. Valid options: a string containing the name attribute of the Host.

#####`parent_realm`

*Optional.* Specifies which Realm element this Realm should nest under. Valid options: a string containing the className attribute of the Realm element.

#####`additional_attributes`

*Optional.* Specifies any further attributes to add to the Realm element. Valid options: a hash of '< attribute >' => '< value >' pairs.

#####`attributes_to_remove`

*Optional.* Specifies any attributes to remove from the Realm element. Valid options: an array of '< attribute >' => '< value >' pairs.

####tomcat::config::server::service

#####`catalina_base`

Specifies the base directory of the Tomcat installation.

#####`class_name`

*Optional.* Specifies the Java class name of a server implementation to use. Maps to the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/service.html#Common_Attributes). Valid options: a string containing a Java class name.

#####`class_name_ensure`

Specifies whether the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/service.html#Common_Attributes) should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`service_ensure`

Specifies whether the [Service element](http://tomcat.apache.org/tomcat-8.0-doc/config/service.html#Introduction) should exist in the configuration file.  Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

####tomcat::config::server::tomcat_users

#####`catalina_base`

Specifies the base directory of the Tomcat installation.

#####`element`

*Optional.* Specifies the type of element to manage. Valid options: 'user' or 'role'. Default: 'user'.

#####`element_name`

*Optional.* Sets the element's username (or rolename, if `element` is set to 'role'). Default: name.

#####`ensure`

Determines whether the specified XML element should exist in the configuration file. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`file`

*Optional.* Specifies the configuration file to manage. Valid options: a string containing a fully-qualified path. Default: '$catalina_base/conf/tomcat-users.xml'.

#####`manage_file`

Specifies whether to create the specified configuration file, if it doesn't exist. Uses Puppet's native [`file` resource type](https://docs.puppetlabs.com/references/latest/type.html#file) with default parameters. Valid options: 'true' and 'false'.

#####`password`

Specifies a password for user elements.

#####`roles`

Specifies one or more roles. Only valid if `element` is set to 'role'.

####tomcat::config::server::valve

#####`catalina_base`

Specifies the base directory of the Tomcat installation.

#####`class_name`

*Optional.* Specifies the Java class name of a server implementation to use. Maps to the [className XML attribute](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Access_Logging/Attributes). Valid options: a string containing a Java class name. Default: the '[name]' passed in your define.

#####`parent_host`

*Optional.* Specifies which virtual host the Valve should nest under. Valid options: a string containing the name of a Host element. Default: undef (If you don't specify a host, the Valve element nests under the Engine of your specified parent Service).

#####`parent_service`

*Optional.* Specifies which the Service element this Valve should nest under. Default: 'Catalina'.

#####`valve_ensure`

Specifies whether the valve should exist in the configuration file. Maps to the  [Valve XML element](http://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Introduction). Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`additional_attributes`

*Optional.* Specifies any further attributes to add to the Valve. Valid options: a hash of '< attribute >' => '< value >' pairs.

#####`attributes_to_remove`

*Optional.* Specifies any attributes to remove from the Valve. Valid options: a hash of '< attribute >' => '< value >' pairs.

####tomcat::instance

#####`catalina_home`

Specifies the root directory of the Tomcat installation. Only affects the instance installation if `install_from_source` is set to 'true'.

#####`catalina_base`

Specifies the base directory of the Tomcat installation. Only affects the instance installation if `install_from_source` is set to 'true'.

#####`install_from_source`

Specifies whether to install from source. If set to 'false', installation is driven by the `package_ensure` and `package_name` parameters. Valid options: 'true' and 'false'. Default: 'true'.

#####`source_url`

*Required, if `install_from_source` is set to 'true'.* Specifies the source URL to install from.

#####`source_strip_first_dir`

*Optional.* Specifies whether to strip the topmost directory of the tarball contents when unpacking it. Only valid if `install_from_source` is set to 'true'. Valid options: 'true' and 'false'. Default: 'true'.

#####`package_ensure`

Determines whether the specified package should be installed. Only valid if `install_from_source` is set to 'false'. Maps to the `ensure` parameter of Puppet's native [`package` resource type](https://docs.puppetlabs.com/references/latest/type.html#package). Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`package_name`

*Required, if `install_from_source` is set to 'false'.* Specifies the package to install. Valid options: a string containing a valid package name.

####tomcat::service

#####`catalina_home`

Specifies the root directory of the Tomcat installation.

#####`catalina_base`

Specifies the base directory of the Tomcat installation.

#####`use_jsvc`

*Optional.* Specifies whether to use Jsvc for service management. If both `use_jsvc` and `use_init` are set to 'false', tomcat uses the following commands for service management:

 * `$catalina_base/bin/catalina.sh start`
 * `$catalina_base/bin/catalina.sh stop`

 Valid options: 'true' and 'false'. Default: 'false'.

#####`java_home`

Specifies where Java is installed. Only applies if `use_jsvc` is set to 'true'. Valid options: a string containing an absolute path.

#####`service_ensure`

Specifies whether the Tomcat service should be running. Maps to the `ensure` parameter of Puppet's native [`service` resource type](https://docs.puppetlabs.com/references/latest/type.html#service). Valid options: 'running', 'stopped', 'true', and 'false'. Default: 'present'.

#####`service_enable`

*Optional.* Specifies whether to enable the Tomcat service at boot. Only valid if `use_init` is set to 'true'. Valid options: 'true' and 'false'. Default: 'true', if `use_init` is set ot true and `service_ensure` is set to 'running' or 'true'.

#####`use_init`

Specifies whether to use a package-provided init script for service management. Note that the tomcat module does not supply an init script. If both `use_jsvc` and `use_init` are set to 'false', tomcat uses the following commands for service management:

 * `$catalina_base/bin/catalina.sh start`
 * `$catalina_base/bin/catalina.sh stop`

 Valid options: 'true' and 'false'. Default: 'false'.

#####`service_name`

Specifies the name to use for the service if `use_init` is set to 'true'.

#####`start_command`

Sets the start command to use for the service.

#####`stop_command`

Sets the stop command to use for the service.

####tomcat::setenv::entry

#####`value`

Provides the value(s) of the managed parameter. Valid options: a string or an array. If passing an array, separate values with a single space.

#####`ensure`

Determines whether the fragment should exist in the configuration file.

#####`config_file`

*Optional.* Specifies the configuration file to edit. Valid options: a string containing an absolute path. Default: $'::tomcat::catalina_home/bin/setenv.sh'.

#####`base_path`

**Deprecated.** Please use `config_file` instead.

#####`param`

*Optional.* Specifies a parameter to manage. Default: the '[name]' passed in your define.

#####`quote_char`

*Optional.* Specifies a character to include before and after the specified value. Valid options: a string (usually a single or double quote). Default: (blank).

#####`order`

*Optional.* Determines the ordering of your parameters in the configuration file (parameters with lower `order` values appear first.) Default: '10'.

####tomcat::war

#####`catalina_base`

Specifies the base directory of the Tomcat installation.

#####`app_base`

*Optional.* Specifies where to deploy the WAR. Cannot be used in combination with `deployment_path`. Valid options: a string containing a path relative to $catalina_base. Default: 'webapps'.

#####`deployment_path`

*Optional.* Specifies where to deploy the WAR. Cannot be used in combination with `app_base`. Valid options: a string containing an absolute path.

#####`war_ensure`

Specifies whether the WAR should exist. Valid options: 'true', 'false', 'present', and 'absent'. Default: 'present'.

#####`war_name`

*Optional.* Specifies the name of the WAR. Valid options: a string containing a filename that ends in '.war'. Default: the '[name]' passed in your define.

#####`war_purge`

*Optional.* Specifies whether to purge the exploded WAR directory. Only applicable when `war_ensure` is set to 'absent' or 'false'.

**Note:** Setting this parameter to 'false' does not prevent Tomcat from removing the exploded WAR directory if Tomcat is running and autoDeploy is set to 'true'. Valid options: 'true' and 'false'. Default: 'true'.

#####`war_source`

*Required, unless `war_ensure` is set to 'false' or 'absent'.* Specifies the source to deploy the WAR from. Valid options: a `puppet://`, `http(s)://`, or `ftp://` URL.

##Limitations

This module only supports Tomcat installations on \*nix systems.  The `tomcat::config::server*` defines require augeas version 1.0.0 or newer.

###Multiple Instances

If you are not installing Tomcat instances from source, depending on your packaging, multiple instances may not work.

##Development

Puppet Labs modules on the Puppet Forge are open projects, and community contributions are essential for keeping them great. We can't access the huge number of platforms and myriad of hardware, software, and deployment configurations that Puppet is intended to serve.

We want to keep it as easy as possible to contribute changes so that our modules work in your environment. There are a few guidelines that we need contributors to follow so that we can have a chance of keeping on top of things.

For more information, see our [module contribution guide.](https://docs.puppetlabs.com/forge/contributing.html)

###Contributors

To see who's already involved, see the [list of contributors.](https://github.com/puppetlabs/puppetlabs-tomcat/graphs/contributors)

###Running tests

This project contains tests for both [rspec-puppet](http://rspec-puppet.com/) and [beaker-rspec](https://github.com/puppetlabs/beaker-rspec) to verify functionality. For in-depth information please see their respective documentation.

Quickstart:

    gem install bundler
    bundle install
    bundle exec rake spec
    bundle exec rspec spec/acceptance
    RS_DEBUG=yes bundle exec rspec spec/acceptance