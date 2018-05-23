## SkyEnterprise Zero Touch Provisioning (ZTP)
This section explains some of the quirks of the ZTP process for SkyEnterprise and provides templates to use when deploying your own config.

### ZTP templates
The provided ZTP templates contain only the required SkyEnterprise configuration. You can add your own XML config in freely.

The provided template permits root login via SSH and configures the root user with the password `lab123`. __DO NOT USE THESE VALUES IN PRODUCTION!__

Do not try and create your own XML config from scratch - it is best to create the config in Junos then have it display it as XML. Copy what is displayed into the ZTP template, add variables where required, then paste into SkyEnterprise.

To display your config as XML in Junos and save it to file:
```
show configuration | display xml | save /tmp/xmlconfig.xml
```

Get the saved file from Junos:
```
scp username@device-hostname:/tmp/xmlconfig.xml ~/path/to/save/file/to
```

### Overriding existing configuration
Currently SkyEnterprise cannot override any existing configuration (ie: that from the factory default configuration).

The current workaround is to delete the section of config you wish to override, and then re-add your new config.

To delete sections of config in XML:
```xml
<device xmlns="http://juniper.net/zerotouch-bootstrap-server">
  ...
  <configuration>
    <config>
      <configuration>
         ...
         ...
         <security delete="delete"/>
         <security>
            <log>
               <mode>stream</mode>
               <report></report>
            </log>
            <pki>
               ...
            </pki>
         </security>
         ...
         ...
      </configuration>
    </config>
  </configuration>

</device>
```

Sections of the snippet above have been omitted for brevity.

In the example above, the entire *security* section of the config is deleted. A user defined *security* config is then added.

### Variables
There wouldn't be much point to ZTP if the same hardcoded config had to be used for every device.

You can replace hardcoded values in your ZTP template with variables. Then, when assigning a specific ZTP template to a device, SkyEnterprise will prompt you to fill in a value for that variable. When pushing the template to the device, SkyEnterprise renders the template with the variable values given.


#### Format
The variables used by SkyEnterprise are similar to those uses by Jinja templates.

To create a variable, enclose the name of the variable within __two__ curly brackets. For example:
```
{{ management_ip_and_cidr }}
```

#### Rules and restrictions
When naming your variables, Keep It Simple Stupid:
- Do not use special characters or symbols other than underscore _
- Do not use spaces.
- Numbers are ok, but try not to start the variable with a number.
- Be descriptive so you can remember what the variable is for.
- Consider using two variables for ip/cidr notation, eg: ```{{ ip }}/{{ cidr }}```
- Do not use a SkyEnterprise Special/Reserved variable name (see below).


#### SkyEnterprise Reserved Variables
One of the reasons for using the provided ZTP templates is that they include variables that will be populated by SkyEnterprise itself for the purposes of communicating with the device and applying the ZTP template. These are:

```
{{ ztp_username }}
{{ ztp_username }}
{{ ztp_host_id }}
{{ ztp_secret }}
```

#### Example template using variables
The below snippet describes how to use the variables within a template. The full example in the file  ```example.xml```

The example simply sets the hostname, DNS nameservers, `fxp0` management interface, and a default route to the gateway.

```xml
<device xmlns="http://juniper.net/zerotouch-bootstrap-server">
  <unique-id>{{ serial }}</unique-id>
  <configuration>
    <config>
      <configuration>
        <system>
          <host-name>{{ hostname }}</host-name>
          <name-server>
            <name>{{ dns_server_1 }}</name>
          </name-server>
          <name-server>
            <name>{{ dns_server_1 }}</name>
          </name-server>
        </system>
        <interfaces>
           <interface delete="delete"/>
              <name>fxp0</name>
           </interface>
           <interface>
              <name>fxp0</name>
              <unit>
                 <name>0</name>
                 <family>
                    <inet>
                       <address>
                          <name>{{ oob_management_ip_and_cidr }}</name>
                       </address>
                    </inet>
                 </family>
              </unit>
           </interface>
        </interfaces>
        <routing-options>
           <static>
              <route>
                 <name>0.0.0.0/0</name>
                 <next-hop>{{ gateway_ip }}</next-hop>
              </route>
            </static>
        </routing-options>
      </configuration>
    </config>
  </configuration>
</device>
```

Now, when deploying this template to SkyEnterprise, we will be prompted to fill in values for each of the above variables.

SkyEnterprise will then render the template with these variables and push the config to the device when it contacts the phone-home server.

Note that the `fxp0` interface is first delete and then re-added via the template config. This is in case there is an existing `fxp0` configuration: SkyEnterprise cannot currently override existing configuration and so this workaround is needed.
