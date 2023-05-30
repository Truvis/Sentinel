# OPNsense > LogStash > Azure Sentinel

NOTE: This guide does not touch on the parsing of the other log types from other services within OpnSense(expect that to come later).
UPDATE: Suricata parsing was added

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
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/05-apps.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/03-filter.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/20-interfaces.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/30-geoip.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/49-cleanup.conf /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/50-output.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/Truvis/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/patterns/pfelk.grok -P /etc/logstash/conf.d/patterns/

3. Update firewall interfaces.

    Amend the 20-interfaces.conf file.

    ```BASH
    sudo nano /etc/logstash/conf.d/20-interfaces.conf
    ```

    Adjust the interface name(s) to correspond with your hardware, the interface below is referenced as `ovpnc1` with a corresponding alias `OpenVPN`, It is also possible to add a friendly name in the `[network][name]` field.

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

### Forwarding OPNsense Logs to Logstash

1. Confirm that syslogs are being collected by listening to port 5140.

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

  The new custom log will be created automatically by the Azure Log Analytics plugin for Logstash. You should find the opnSense table in Azure Sentinel -> Logs -> Custom Logs.

  `You do not need to configure a custom log source in Azure Sentinel "Advanced settings".`
  
- It can take up to 20 minutes for the Custom Logs table to be populated.

    ![image](https://github.com/Truvis/Sentinel/assets/23244379/4e586445-5681-4184-993d-aecfba31d818)

### Query logs in Azure Sentinel

Using the query below we can query if we are getting logs with GeoIP information into Azure Sentinel.

```BASH
// Find connection to countries not US
OPNSense_CL
| where destination_geo_geo_country_iso_code_s != "US"
| summarize count() by destination_geo_geo_country_iso_code_s
```
![image](https://github.com/Truvis/Sentinel/assets/23244379/1428062d-ca3d-4e72-9b51-14f6c1a1e8f4)

## Troubleshooting

1. Once you have enabled the Logstash service and it has been started check `logstash-plain.log` to confirm there are no errors.

    ```BASH
    cat /var/log/logstash/logstash-plain.log
    ```

    If you see the error below in `logstash-plain.log` you have a configuration error. Please check any edits you have made to the config files in `/etc/logstash/conf.d/`.

    ```BASH
    [2021-04-13T08:53:00,410][INFO ][logstash.runner ] Logstash shut down. 
    [2021-04-13T08:53:00,425][FATAL][org.logstash.Logstash ] Logstash stopped processing because of an error: (SystemExit) exit org.jruby.exceptions.SystemExit: (SystemExit) exit at org.jruby.RubyKernel.exit(org/jruby/RubyKernel.java:747) ~[jruby-complete-9.2.13.0.jar:?] at org.jruby.RubyKernel.exit(org/jruby/RubyKernel.java:710) ~[jruby-complete-9.2.13.0.jar:?] at usr.share.logstash.lib.bootstrap.environment.<main>(/usr/share/logstash/lib/bootstrap/environment.rb:89) ~[?:?]
    ```

    When you configuration is working correctly your log file should look like this.

    ```BASH
    [2023-05-25T18:29:49,814][INFO ][logstash.javapipeline    ][main] Pipeline Java execution initialization time {"seconds"=>1.14}
    [2023-05-25T18:29:49,835][INFO ][logstash.javapipeline    ][main] Pipeline started {"pipeline.id"=>"main"}
    [2023-05-25T18:29:49,837][INFO ][logstash.inputs.syslog   ][main][OPNSenseFirewall] Starting syslog udp listener {:address=>"0.0.0.0:5140"}
    [2023-05-25T18:29:49,840][INFO ][logstash.inputs.syslog   ][main][OPNSenseFirewall] Starting syslog tcp listener {:address=>"0.0.0.0:5140"}
    [2023-05-25T18:29:49,848][INFO ][logstash.agent           ] Pipelines running {:count=>1, :running_pipelines=>[:main], :non_running_pipelines=>[]}
    ```
