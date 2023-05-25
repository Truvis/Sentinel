# pfSense/OPNsense > LogStash > Azure Sentinel

## Table of Contents

- [pfSense/OPNsense > LogStash > Azure Sentinel](#pfsenseopnsense--logstash--azure-sentinel)
  - [Table of Contents](#table-of-contents)
    - [Ubuntu (v18.04-v20.04+) Server onPrem](#ubuntu-v1804-v2004-server-onprem)
    - [Install MaxMind Database (Optional)](#install-maxmind-database-optional)
    - [Logstash Configuration](#logstash-configuration)
    - [Forwarding pfSense Logs to Logstash](#forwarding-pfsense-logs-to-logstash)
  - [Install and Configure the Log Analytics Plugin For Logstash](#install-and-configure-the-log-analytics-plugin-for-logstash)
    - [View pfSense Logs in Azure Sentinel](#view-pfsense-logs-in-azure-sentinel)
    - [Query logs in Azure Sentinel](#query-logs-in-azure-sentinel)
  - [Troubleshooting](#troubleshooting)

### Ubuntu (v18.04-v20.04+) Server onPrem
  
1. Install Ubuntu Server (v18.04-v20.04+) on a Virtual Machine or Computer and update the OS.

    ```BASH
    sudo apt update; sudo apt upgrade -y
    ```

2. Disabling Swap - Swapping should be disabled for performance and stability. (optional)

    ```BASH
    sudo swapoff -a
    ```

3. Configuration Date/Time Zone

   - The box running this configuration will reports firewall logs based on its clock. The command below will set the timezone to Eastern Standard Time (EST).
   - To view available timezones type `sudo timedatectl list-timezones`.

    ```BASH
    sudo timedatectl set-timezone Europe/London
    ```

4. Download and install the public GPG signing key.

    ```BASH
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    ```

5. Download and install apt-transport-https package.

    ```BASH
    sudo apt install apt-transport-https
    ```

6. Add Logstash Repositories (version 7+).

    ```BASH
    echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
    ```

7. Install Java 14 LTS.

    ```bash
    sudo apt install openjdk-14-jre-headless
    ```

### Install MaxMind Database

Maxmind isn't required for GeoIP lookups as this is also handled by Logstash by default.

1. Follow the steps [here](https://github.com/pfelk/pfelk/wiki/How-To:-MaxMind-via-GeoIP-with-pfELK), to install and utilise MaxMind. Otherwise the built-in GeoIP from Elastic will be utilised.

Example:

```BASH
if "IP_Private_Source" not in [tags] {
  geoip {
    source => "[truvis_sourceip]"
    database => "/usr/share/GeoIP/GeoLite2-City.mmdb"
    target => "[source][geo]"
  }
```

### Logstash Configuration

1. Install Logstash.

    ```BASH
    sudo apt update && sudo apt install logstash -y
    ```

2. Download the following configuration files. (Required)

    ```BASH
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/01-inputs.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/03-filter.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/20-interfaces.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/30-geoip.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/49-cleanup.conf /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/50-output.conf -P /etc/logstash/conf.d/

3. Update firewall interfaces.

    Amend the 20-interfaces.conf file.

    ```BASH
    sudo nano /etc/logstash/conf.d/20-interfaces.conf
    ```

    Adjust the interface name(s) `igb0` to correspond with your hardware, the interface below is referenced as igb0 with a corresponding alias `WAN`, It is also possible to add a friendly name in the `[network][name]` field.

    Add/remove sections, depending on the number of interfaces you have.

    ```BASH
    ### Change interface as desired ###
    if [truvis_interface] =~ /^ovpnc1$/ {
      mutate {
        add_field => { "[interface][alias]" => "OpenVPN-1" }
        add_field => { "[network][name]" => "Outgoing Connections" }
      }
    }
    ```

### Forwarding pfSense Logs to Logstash

4. Confirm that syslogs are being collected by listening to port 5140.

    ```BASH
    sudo tcpdump -A -ni any port 5140 -vv
    ```

## Install and Configure the Log Analytics Plugin For Logstash

Make a note of your Azure Configuration, you will need it to configure the the Log Analytics Plugin for logstash in `step 4`.

1. Login to Azure and browse to your `Log Analytics workspace` settings.
2. Select `Agents Management` and make a note of your `Workspace ID` and `Primary Key`.

    ![image](https://github.com/Truvis/Sentinel/assets/23244379/6fef583e-9409-4dc2-8201-9a33a225508d)

3. Run the command to install the [Microsoft Logstash LogAnalytics](https://github.com/Azure/Azure-Sentinel/tree/master/DataConnectors/microsoft-logstash-output-azure-loganalytics) plugin.

    ```BASH
    sudo /usr/share/logstash/bin/logstash-plugin install microsoft-logstash-output-azure-loganalytics
    ```

4. Edit the Logstash configuration.

    ```BASH
    sudo nano /etc/logstash/conf.d/50-outputs.conf
    ```

    ```BASH
    output {
        microsoft-logstash-output-azure-loganalytics {
            workspace_id => "<WORKSPACE ID>" # <your workspace id>
            workspace_key => "<Primary Key>" # <your workspace key>
            custom_log_table_name => "<Name of Log>"
            }
        }
    ```

    Using the information we previously noted down from the `Log Analytics workspace` settings, use it to populated the `50-outputs.conf` file.

    - workspace_id = `"WORKSPACE ID"`
    - workspace_key = `"Primary Key"`
    - custom_log_table_name =  `"firewall_log"` (This can be any name of your choosing)

    `Example:`

    ```BASH
    output {
        microsoft-logstash-output-azure-loganalytics {
            workspace_id => "1234567-7654321-345678-12334445"
            workspace_key => "kflsdjkgfslfjsdf0ife0f0efe0-09f0we9f-ef-w00e-0w-f0w-0fwe-f0d0-w=="
            custom_log_table_name => "firewall"
            }
        }
    ```

    - `Note: custom_log_table_name can only have these characters [a-z] [0-9] no spaces or any other characters`
    - `Note: Please use the example in your 50-outputs.conf with does not include any comments '#'`

5. Enable and Start LogStash.

    ```BASH
    sudo systemctl enable logstash
    sudo systemctl start logstash
    ```

6. Once you have enabled the Logstash service and it has been started check `logstash-plain.log` to confirm there are no errors.

    ```BASH
    cat /var/log/logstash/logstash-plain.log
    ```

  `Note: If you are seeing errors in the log please refer to the [troubleshooting](#troubleshooting) steps below.

### View Logs in Azure Sentinel

1. Wait for logs to arrive in Azure Sentinel.

  The new custom log will be created automatically by the Azure Log Analytics plugin for Logstash. You should find the pfSense/opnSense table in Azure Sentinel -> Logs -> Custom Logs.

  `You do not need to configure a custom log source in Azure Sentinel "Advanced settings".`
  
- It can take up to 20 minutes for the Custom Logs table to be populated.

    ![Azure-Sentinel](../.images/image2.png)

### Query logs in Azure Sentinel

Using the query below we can query if we are getting logs with GeoIP information into Azure Sentinel.

```BASH
// query
```

Additional queries for pfSense can be found [here](https://github.com/noodlemctwoodle/pf-azure-sentinel/tree/main/KQL/pfSense/Queries)

## Troubleshooting

1. Once you have enabled the Logstash service and it has been started check `logstash-plain.log` to confirm there are no errors.

    ```BASH
    cat /var/log/logstash/logstash-plain.log
    ```

    If you see the error below in `logstash-plain.log` you have a configuration error. Please check any edits you have made to the config files in `/etc/logstash/conf.d/`.

    ```BASH
    [2021-04-13T08:52:58,325][INFO ][logstash.runner ] Log4j configuration path used is: /etc/logstash/log4j2.properties 
    [2021-04-13T08:52:58,342][INFO ][logstash.runner ] Starting Logstash {"logstash.version"=>"7.12.0", "jruby.version"=>"jruby 9.2.13.0 (2.5.7) 2020-08-03 9a89c94bcc OpenJDK 64-Bit Server VM 11.0.10+9 on 11.0.10+9 +indy +jit [linux-x86_64]"} 
    [2021-04-13T08:52:59,955][INFO ][logstash.agent ] Successfully started Logstash API endpoint {:port=>9600} 
    [2021-04-13T08:53:00,275][ERROR][logstash.agent ] Failed to execute action {:action=>LogStash::PipelineAction::Create/pipeline_id:main, :exception=>"LogStash::ConfigurationError", :message=>"Expected one of [ \\t\\r\\n], \"#\", \"input\", \"filter\", \"output\" at line 22, column 3 (byte 873) after ", :backtrace=>["/usr/share/logstash/logstash-core/lib/logstash/compiler.rb:32:in compile_imperative'", "org/logstash/execution/AbstractPipelineExt.java:184:in initialize'", "org/logstash/execution/JavaBasePipelineExt.java:69:in initialize'", "/usr/share/logstash/logstash-core/lib/logstash/java_pipeline.rb:47:in initialize'", "/usr/share/logstash/logstash-core/lib/logstash/pipeline_action/create.rb:52:in execute'", "/usr/share/logstash/logstash-core/lib/logstash/agent.rb:389:in block in converge_state'"]} 
    [2021-04-13T08:53:00,410][INFO ][logstash.runner ] Logstash shut down. 
    [2021-04-13T08:53:00,425][FATAL][org.logstash.Logstash ] Logstash stopped processing because of an error: (SystemExit) exit org.jruby.exceptions.SystemExit: (SystemExit) exit at org.jruby.RubyKernel.exit(org/jruby/RubyKernel.java:747) ~[jruby-complete-9.2.13.0.jar:?] at org.jruby.RubyKernel.exit(org/jruby/RubyKernel.java:710) ~[jruby-complete-9.2.13.0.jar:?] at usr.share.logstash.lib.bootstrap.environment.<main>(/usr/share/logstash/lib/bootstrap/environment.rb:89) ~[?:?]
    ```

    When you configuration is working correctly your log file should look like this.

    ```BASH
    [2021-04-13T16:10:02,158][INFO ][logstash.runner          ] Log4j configuration path used is: /etc/logstash/log4j2.properties
    [2021-04-13T16:10:02,172][INFO ][logstash.runner          ] Starting Logstash {"logstash.version"=>"7.12.0", "jruby.version"=>"jruby 9.2.13.0 (2.5.7) 2020-08-03 9a89c94bcc OpenJDK 64-Bit Server VM 11.0.10+9 on 11.0.10+9 +indy +jit [linux-x86_64]"}
    [2021-04-13T16:10:03,721][INFO ][logstash.agent           ] Successfully started Logstash API endpoint {:port=>9600}
    [2021-04-13T16:10:10,438][INFO ][org.reflections.Reflections] Reflections took 44 ms to scan 1 urls, producing 23 keys and 47 values
    [2021-04-13T16:10:12,879][INFO ][logstash.outputs.azureloganalytics][main] Azure Loganalytics configuration was found valid.
    [2021-04-13T16:10:12,879][INFO ][logstash.outputs.azureloganalytics][main] Logstash Azure Loganalytics output plugin configuration was found valid
    [2021-04-13T16:10:12,922][INFO ][logstash.filters.geoip   ][main] Using geoip database {:path=>"/usr/share/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-6.0.5-java/vendor/GeoLite2-City.mmdb"}
    [2021-04-13T16:10:12,952][INFO ][logstash.filters.geoip   ][main] Using geoip database {:path=>"/usr/share/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-6.0.5-java/vendor/GeoLite2-ASN.mmdb"}
    [2021-04-13T16:10:14,338][INFO ][logstash.filters.geoip   ][main] Using geoip database {:path=>"/usr/share/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-6.0.5-java/vendor/GeoLite2-ASN.mmdb"}
    [2021-04-13T16:10:14,344][INFO ][logstash.filters.geoip   ][main] Using geoip database {:path=>"/usr/share/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-6.0.5-java/vendor/GeoLite2-ASN.mmdb"}
    [2021-04-13T16:10:14,405][INFO ][logstash.filters.geoip   ][main] Using geoip database {:path=>"/usr/share/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-6.0.5-java/vendor/GeoLite2-City.mmdb"}
    [2021-04-13T16:10:14,611][INFO ][logstash.filters.geoip   ][main] Using geoip database {:path=>"/usr/share/logstash/vendor/bundle/jruby/2.5.0/gems/logstash-filter-geoip-6.0.5-java/vendor/GeoLite2-City.mmdb"}
    [2021-04-13T16:10:15,404][INFO ][logstash.javapipeline    ][main] Starting pipeline {:pipeline_id=>"main", "pipeline.workers"=>16, "pipeline.batch.size"=>125, "pipeline.batch.delay"=>50, "pipeline.max_inflight"=>2000, "pipeline.sources"=>["/etc/logstash/conf.d/01-inputs.conf", "/etc/logstash/conf.d/02-types.conf", "/etc/logstash/conf.d/03-filter.conf", "/etc/logstash/conf.d/05-apps.conf", "/etc/logstash/conf.d/20-interfaces.conf", "/etc/logstash/conf.d/30-geoip.conf", "/etc/logstash/conf.d/35-rules-desc.conf", "/etc/logstash/conf.d/36-ports-desc.conf", "/etc/logstash/conf.d/37-enhanced_user_agent.conf", "/etc/logstash/conf.d/38-enhanced_url.conf", "/etc/logstash/conf.d/45-cleanup.conf", "/etc/logstash/conf.d/49-enhanced_private.conf", "/etc/logstash/conf.d/50-outputs.conf"], :thread=>"#<Thread:0x419d701a run>"}
    [2021-04-13T16:10:19,968][INFO ][logstash.javapipeline    ][main] Pipeline Java execution initialization time {"seconds"=>4.56}
    [2021-04-13T16:10:20,151][INFO ][logstash.inputs.beats    ][main] Starting input listener {:address=>"0.0.0.0:5044"}
    [2021-04-13T16:10:20,172][INFO ][logstash.javapipeline    ][main] Pipeline started {"pipeline.id"=>"main"}
    [2021-04-13T16:10:20,179][INFO ][logstash.inputs.tcp      ][main][pf-AzSentinel-suricata] Starting tcp input listener {:address=>"0.0.0.0:5040", :ssl_enable=>false}
    [2021-04-13T16:10:20,221][INFO ][logstash.inputs.udp      ][main][pf-AzSentinel-1] Starting UDP listener {:address=>"0.0.0.0:5140"}
    [2021-04-13T16:10:20,256][INFO ][logstash.inputs.udp      ][main][pf-AzSentinel-2] Starting UDP listener {:address=>"0.0.0.0:5141"}
    [2021-04-13T16:10:20,256][INFO ][org.logstash.beats.Server][main][Beats] Starting server on port: 5044
    [2021-04-13T16:10:20,288][INFO ][logstash.inputs.udp      ][main][pf-AzSentinel-haproxy] Starting UDP listener {:address=>"0.0.0.0:5190"}
    [2021-04-13T16:10:20,308][INFO ][logstash.inputs.udp      ][main][pf-AzSentinel-2] UDP listener started {:address=>"0.0.0.0:5141", :receive_buffer_bytes=>"106496", :queue_size=>"2000"}
    [2021-04-13T16:10:20,310][INFO ][logstash.inputs.udp      ][main][pf-AzSentinel-haproxy] UDP listener started {:address=>"0.0.0.0:5190", :receive_buffer_bytes=>"106496", :queue_size=>"2000"}
    [2021-04-13T16:10:20,313][INFO ][logstash.inputs.udp      ][main][pf-AzSentinel-1] UDP listener started {:address=>"0.0.0.0:5140", :receive_buffer_bytes=>"106496", :queue_size=>"2000"}
    ```
