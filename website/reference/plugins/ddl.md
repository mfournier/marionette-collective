---
layout: default
title: Data Definition Language
---
[WritingAgents]: /mcollective/reference/basic/basic_agent_and_client.html
[SimpleRPCClients]: /mcollective/simplerpc/clients.html
[ResultsandExceptions]: /mcollective/simplerpc/clients.html#results_and_exceptions
[SimpleRPCAuditing]: /mcollective/simplerpc/auditing.html
[SimpleRPCAuthorization]: /mcollective/simplerpc/authorization.html
[WritingAgentsScreenCast]: http://mcollective.blip.tv/file/3808928/
[DDLScreenCast]: http://mcollective.blip.tv/file/3799653
[RPCUtil]: /mcollective/reference/plugins/rpcutil.html
[ValidatorPlugins]: /mcollective/reference/plugins/validator.html

As with other remote procedure invocation systems MCollective has a DDL that defines what remote methods are available, what inputs they take and what outputs they generate.

In addition to the usual procedure definitions we also keep meta data about author, versions, license and other key data points.

The DDL is used in various scenarios:

 * The user can access it in the form of a human readable help page
 * User interfaces can access it in a way that facilitate auto generation of user interfaces
 * The RPC client auto configures and use appropriate timeouts in waiting for responses
 * Before sending a call over the network inputs get validated so we do not send unexpected data to remote nodes.
 * Module repositories can use the meta data to display a standard view of available modules to assist a user in picking the right ones.
 * The server will validate incoming requests prior to sending it to agents

We've created [a screencast showing the capabilities of the DDL][DDLScreenCast] that might help give you a better overview.

**NOTE:** As of version 2.1.1 the DDL is required on all servers before an agent will be activated

## Examples
We'll start with a few examples as I think it's pretty simple what they do, and later on show what other permutations are allowed for defining inputs and outputs.

A helper agent called [_rpcutil_][RPCUtil] is included that helps you gather stats, inventory etc about the running daemon.  This helper has a full DDL included, see the plugins dir for this agent.

The typical service agent is a good example, it has various actions that all more or less take the same input.  All but status would have almost identical language.

### Meta Data
First we need to define the meta data for the agent itself:

{% highlight ruby linenos %}
metadata :name        => "SimpleRPC Service Agent",
         :description => "Agent to manage services using the Puppet service provider",
         :author      => "R.I.Pienaar",
         :license     => "GPLv2",
         :version     => "1.1",
         :url         => "http://projects.puppetlabs.com/projects/mcollective-plugins/wiki",
         :timeout     => 60
{% endhighlight %}

It's fairly obvious what these all do, *:timeout* is how long the MCollective daemon will let the threads run.

## Required versions
As of MCollective 2.1.2 you can indicate which is the lowest version of MCollective needed to use a plugin.  Plugins that do not meet the requirement can not be used.

{% highlight ruby linenos %}
requires :mcollective => "2.0.0"
{% endhighlight %}

You should add this right after the metadata section in the DDL

## Actions, Input and Output
Defining inputs and outputs is the hardest part, below first the *status* action:

{% highlight ruby linenos %}
action "status", :description => "Gets the status of a service" do
    display :always  # supported in 0.4.7 and newer only

    input :service,
          :prompt      => "Service Name",
          :description => "The service to get the status for",
          :type        => :string,
          :validation  => '^[a-zA-Z\-_\d]+$',
          :optional    => false,
          :maxlength   => 30

    output :status,
           :description => "The status of service",
           :display_as  => "Service Status",
           :default     => "unknown status"
end
{% endhighlight %}

As you see we can define all the major components of input and output parameters.  *:type* can be one of various values and each will have different parameters, more on that later.

As of version 2.1.1 the outputs can define a default value.  For agents the reply structures are pre-populated with all the defined outputs, if no default is supplied a default of nil will be set.

By default mcollective only show data from actions that failed, the *display* line above tells it to always show the results.  Possible values are *:ok*, *:failed* (the default behavior) and *:always*.

Finally the service agent has 3 almost identical actions - *start*, *stop* and *restart* - below we use a simple loop to define them all in one go.

{% highlight ruby linenos %}
["start", "stop", "restart"].each do |act|
    action act, :description => "#{act.capitalize} a service" do
        input :service,
              :prompt      => "Service Name",
              :description => "The service to #{act}",
              :type        => :string,
              :validation  => '^[a-zA-Z\-_\d]+$',
              :optional    => false,
              :maxlength   => 30

        output :status,
               :description => "The status of service after #{act}",
               :display_as  => "Service Status",
               :default     => "unknown status"
    end
end
{% endhighlight %}

All of this code just goes into a file, no special class or module bits needed, just save it as *service.ddl* in the same location as the *service.rb*.

Importantly you do not need to have the *service.rb* on a machine to use the DDL, this means on machines that are just used for running client programs you can just drop the *.ddl* files into the agents directory.

You can view a human readable version of this using *mco plugin doc &lt;agent&gt;* command:

{% highlight console %}
% mco plugin doc service
SimpleRPC Service Agent
=======================

Agent to manage services using the Puppet service provider

      Author: R.I.Pienaar
     Version: 1.1
     License: GPLv2
     Timeout: 60
   Home Page: http://projects.puppetlabs.com/projects/mcollective-plugins/wiki



ACTIONS:
========
   restart, start, status, stop

   restart action:
   ---------------
       Restart a service

       INPUT:
           service:
              Description: The service to restart
                   Prompt: Service Name
                     Type: string
               Validation: ^[a-zA-Z\-_\d]+$
                   Length: 30


       OUTPUT:
           status:
              Description: The status of service after restart
               Display As: Service Status
{% endhighlight %}

### Optional Inputs
The input block has a mandatory *:optional* field, when true it would be ok if a client attempts to call the agent without this input supplied.  If it is supplied though it will be validated.

### Types of Input
As you see above the input block has *:type* option, types can be *:string*, *:list*, *:boolean*, *:integer*, *:float* or *:number*

#### :string type
The string type validates initially that the input is infact a String, then it validates the length of the input and finally matches the supplied Regular Expression.

Both *:validation* and *:maxlength* are required arguments for the string type of input.

If you want to allow unlimited length text you can make *:maxlength => 0* but use this with care.

As of version 2.2.0 a new plugin type called [Validator Plugins][ValidatorPlugins] exist that allow you to supply your own validations for *:string* types.

#### :list type
List types provide a list of valid options and only those will be allowed, see an example below:

{% highlight ruby linenos %}
input :action,
      :prompt      => "Service Action",
      :description => "The action to perform",
      :type        => :list,
      :optional    => false,
      :list        => ["stop", "start", "restart"]
{% endhighlight %}

In user interfaces this might be displayed as a drop down list selector or another kind of menu.

#### :boolean type

The value input should be either _true_ or _false_ actual boolean values.  This feature was introduced in version _0.4.9_.

#### :integer type

The value input should be an integer number like _1_ or _100_ but not _1.1_.  This feature was introduced in version _1.3.2_

#### :float type

The value input should be a floating point number like _1.0_ but not _1_.  This feature was introduced in version _1.3.2_

#### :number type

The value input should be an integer or a floating point number. This feature was introduced in version _1.3.2_

#### :any type

The value input can be any type, this allows you to send rich objects like arrays of hashes around, it effectively disables validation of the type of input

### Accessing the DDL
While programming client applications or web apps you can gain access to the DDL for any agent in several ways:

{% highlight ruby linenos %}
require 'mcollective'

config = MCollective::Config.instance
config.loadconfig(options[:config])

ddl = MCollective::DDL.new("service")
puts ddl.help("#{config.configdir}/rpc-help.erb")
{% endhighlight %}

This will produce the text help output from the above example, you can supply any ERB template to format the output however you want.

You can also access the data structures directly:

{% highlight ruby linenos %}
ddl = MCollective::DDL.new("service")
puts "Meta Data:"
pp ddl.meta

puts
puts "Status Action:"
pp ddl.action_interface("status")
{% endhighlight %}

{% highlight ruby linenos %}
Meta Data:
{:license=>"GPLv2",
 :author=>"R.I.Pienaar",
 :name=>"SimpleRPC Service Agent",
 :timeout=>60,
 :version=>"1.1",
 :url=>"http://projects.puppetlabs.com/projects/mcollective-plugins/wiki",
 :description=>"Agent to manage services using the Puppet service provider"}

Status Action:
{:action=>"status",
 :input=>
  {:service=>
    {:validation=>"^[a-zA-Z\\-_\\d]+$",
     :maxlength=>30,
     :prompt=>"Service Name",
     :type=>:string,
     :optional=>false,
     :description=>"The service to get the status for"}},
 :output=>
  {"status"=>
    {:display_as=>"Service Status", :description=>"The status of service"}},
 :description=>"Gets the status of a service"}

{% endhighlight %}

The ddl object is also available on any *rpcclient*:

{% highlight ruby linenos %}
service = rpcclient("service")
pp service.ddl.meta
{% endhighlight %}

In the case of accessing it through the service as in this example, if there was no DDL file on the machine for the service agent you'd get a *nil* back from the ddl accessor.

### Input Validation
As mentioned earlier the client does automatic input validation using the DDL, if validation fails you will get an *MCollective::DDLValidationError* exception thrown with an appropriate message.
