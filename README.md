# Model Driven Telemetry lab example

## Contacts
* Christian Tollefsen

## Solution Components
* IOS-XE devices

## Installation/Configuration

### Telegraf installation instructions
1. Install Telegraf from InfluxData's [install page](https://portal.influxdata.com/downloads/ "Click here to download Telegraf!"):
  
2. (Optional) To see all the configuration options, generate a default config file with the following command:
  
   `$ telegraf config > telegraf.conf`
  
   For more information about the .conf file, see Telegraf's [GitHub docs](https://github.com/influxdata/telegraf/blob/master/docs/CONFIGURATION.md)
  
3. Verify that Telegraf is properly installed by reading in your host's CPU utilization:
   `$ sudo telegraf -test -config /location/of/telegraf.conf --input-filter cpu`
  
4. Now run Telegraf:
   `$ sudo telegraf -config /location/of/telegraf.conf`
   
   Use the telegraf.conf in this repository for a quick start.
   
   We'll also be using the [cisco_telemetry_mdt plugin](https://github.com/influxdata/telegraf/tree/master/plugins/inputs/cisco_telemetry_mdt). Please see the attached github for configuration options.
   
### Telemetry setup instructions
While Telegraf is running, let's configure telemetry on the networking device:

1. Enter config mode
   `configure terminal`
   
2. Paste the following:
   ```
   telemetry ietf subscription 101
     encoding encode-kvgpb
     filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
     stream yang-push
     update-policy periodic 500
     receiver ip address x.x.x.x 57000 protocol grpc-tcp
   ```
   Note that: 
   * `filter xpath` denotes which resource we're requesting from the device
   * We're sending it in the yang format
   * We're sending it periodicially, every 5 seconds. 

3. Verify that the config is correct:

3a.
   `show telemetry ietf subscription 101`
   
   The output should look like:
   ```
       Telemetry subscription brief

    ID               Type        State       Filter type
    --------------------------------------------------------
    101              Configured  Valid       xpath
   ```
  Make sure that the state is valid. If not, remove the telemetry subscription with `no telemetry ietf subscription 101` and go back to step 2.
  
3b. Verify that the gRPC dialout has been subscribed:
   `show telemetry ietf subscription 101 receiver`
   
   The output should like like:
   ```
    Subscription ID: 101
    Address: x.x.x.x
    Port: 57000
    Protocol: grpc-tcp
    Profile:
    State: Connected
    Explanation:
   ```
   If the state is still `Connecting`, wait for the flush-time you set in the telegraf.conf file, and run the command again.
   
4. Your Telegraf should now display that it accepted a gRPC dialout from your networking device:
   `2020-08-20T11:33:27Z D! [inputs.cisco_telemetry_mdt] Accepted Cisco MDT GRPC dialout connection from x.x.x.x:yyyyy`
   
   After the data has been serialized by telegraf, it is printed to stdout:
   ```json
   {"_value":1,"metric_name":"Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization.five_seconds","path":"Cisco-IOS-XE-process-cpu-oper:cpu-usage/cpu-utilization","subscription":"101","time":1597921707.633}2020-08-20T11:33:28Z
   ```
5. If you used the `telegraf.conf` file included in this repository, it will also write this to `/tmp/metrics.out/.

If you set your Splunk input to read from `/tmp/metrics.out`, you will now have the data safely in Splunk. If you would rather send the data over HTTPS, you can use the `outputs.https` plugin from the Telegraf default config file.



### Usage

#### Get Yang Explorer up and running

To find the correct x-path filter, download [Yang Explorer](https://github.com/CiscoDevNet/yang-explorer). (NOTE: currently end of support, but it still works.

Before starting on the install guide, do the following steps to make sure you have the correct Python version:

1. Install PyEnv to manage Python versions

```bash
$ pip install pyenv
```

2. Install `pyenv-virtualenv` [plugin](https://github.com/pyenv/pyenv-virtualenv "pyenv-virtualenv github") to manage virtual environments with pyenv

```bash
$ git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv
```

3. Reset your terminal session

```bash
$ exec "$SHELL"
```

4. Download Python 2.7 to through PyEnv

```bash
$ pyenv install 2.7.15
```

5. Create your virtual environment and activate it through pyenv-virtualenv

```bash
$ pyenv virtualenv 2.7.15 mdt-venv
$ pyenv activate 2.7.15/envs/mdt-venv
```

6. Follow the install guide to get yang-explorer running.

#### Synchronize models from device

When it's running, you're going to want to import models so you can find which xpath filter to use. There are multiple ways to do this, many of which are bugged. The solution I verified is to sync them from the device.

1. Open yang-explorer (default is `localhost:8088`)

2. Log in to the top right with either username/password of admin/admin or guest/guest

3. Go to Manage Models -> Device, and press Create device profile.

![Manage models](https://i.imgur.com/B85bf1f.png)

4. For this next step, NetConf needs to be enabled on the remote device. Create the device profile including the SSH login information.

![Create device profile](https://i.imgur.com/xwSCYvv.png)

You don't need to fill out the RestConf information, but make sure you filled out the NetConf IP, port, username, and password, and selected the user you previously logged in with. You also need a description.

5. The remaining steps are explained in the [Yang explorer config guide](https://github.com/CiscoDevNet/yang-explorer#sync-from-device). Keep in mind that some models are based on others, so you might run in to some issues if you don't sync all dependant models. 

6. Find the model you want, and select the specific variable to get that information. 

![Xpath filter](https://i.imgur.com/NjjeGCj.png)

To use a model, copy the field to the bottom right called "Xpath filter" into the config on the IOS device with the following commands:

```
configure terminal
telemetry subscription ietf <subscription number>
  xpath filter <copy xpath filter here>
end
```

If you want to specify a single entry in an XPath filter, the [Catalyst 9300 config guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/prog/configuration/172/b_172_programmability_cg/model_driven_telemetry.html#id_90652) has this to say:

> The dataset within the yang-push stream to be subscribed to should be specified by the use of an XPath filter. The following guidelines apply to the XPath expression:
XPath expressions can have keys to specify a single entry in a list or container. The supported key specification syntax is
`[{key name}={key value}]`

>The following is an example of an XPath expression:
```
filter xpath /rt:routing-state/routing-instance[name="default"]/ribs/rib[name="ipv4-default"]/routes/route 
# VALID!
```

Compound keys are supported by the use of multiple key specifications. Key names and values must be exact; no ranges or wildcard values are supported.

And that's it! I hope you enjoyed this tutorial. 

# Legal 

![/IMAGES/0image.png](/IMAGES/0image.png)

### LICENSE

Provided under Cisco Sample Code License, for details see [LICENSE](LICENSE.md)

### CODE_OF_CONDUCT

Our code of conduct is available [here](CODE_OF_CONDUCT.md)

### CONTRIBUTING

See our contributing guidelines [here](CONTRIBUTING.md)

#### DISCLAIMER:
<b>Please note:</b> This script is meant for demo purposes only. All tools/ scripts in this repo are released for use "AS IS" without any warranties of any kind, including, but not limited to their installation, use, or performance. Any use of these scripts and tools is at your own risk. There is no guarantee that they have been through thorough testing in a comparable environment and we are not responsible for any damage or data loss incurred with their use.
You are responsible for reviewing and testing any scripts you run thoroughly before use in any non-testing environment.
