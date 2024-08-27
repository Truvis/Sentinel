# OPNsense > LogStash > Azure Sentinel

>NOTE: This guide does not touch on the parsing of the other log types from other services within OPNSense(expect that to come later).

>NOTE: This guide currently is primarily for using a Custom Log Analytics table. Expect a modification to use ASIM-formatted NetworkSessions log later.

>UPDATE: Suricata parsing was added. Make sure you have EVE logging checked in the IDPS configuration in OPNSense.

### Ubuntu (v22.04) Server in Azure
  
1. Install Ubuntu Server (v22.04) on an Azure Virtual Machine (or maybe computer connected to Azure with ARC?).

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

### Install MaxMind Database

> NOTE: I had to install Maxmind for the pipeline to work.

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
    sudo wget https://raw.githubusercontent.com/HartD92/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/01-inputs.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/HartD92/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/05-apps.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/HartD92/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/03-filter.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/HartD92/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/20-interfaces.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/HartD92/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/30-geoip.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/HartD92/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/49-cleanup.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/HartD92/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/50-output.conf -P /etc/logstash/conf.d/
    sudo wget https://raw.githubusercontent.com/HartD92/Sentinel/main/Parsers/LogStash/OPNSense/conf.d/patterns/pfelk.grok -P /etc/logstash/conf.d/patterns/
    ```

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

1. Login to Entra and browse to your `App Registrations` settings.
2. Create a new `App Registration` per the [Microsoft Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#create-microsoft-entra-application).
3. Create a new `App Secret` and make a note of it, as well as the `Application (client) ID` and `Directory (tenant) ID`.
    
    ![image](https://raw.githubusercontent.com/HartD92/Sentinel/3857dbf6623c033f8415599ae00b74551ccbf4ae/entraAppReg.png)

4. Navigate to `Azure Monitor`, select `Data Collection Endpoint`, and create a new one as per the [Microsoft Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#create-data-collection-endpoint). Make note of the `Logs Ingestion URL`.

    ![image](https://raw.githubusercontent.com/HartD92/Sentinel/3857dbf6623c033f8415599ae00b74551ccbf4ae/DCEEndpoint.png)

5. Run the command to install the [Microsoft Logstash LogAnalytics](https://github.com/Azure/Azure-Sentinel/tree/master/DataConnectors/microsoft-logstash-output-azure-loganalytics) plugin.

    ```BASH
    sudo /usr/share/logstash/bin/logstash-plugin install microsoft-sentinel-log-analytics-logstash-output-plugin
    ```

6. If using a Custom Table, generate a sample file.
    1. Edit the Logstash configuration to generate a sample file.

        ```BASH
        sudo nano /etc/logstash/conf.d/50-outputs.conf
        ```

        ```BASH
        output {
            microsoft-sentinel-log-analytics-logstash-output-plugin {
                create_sample_file => true
                sample_file_path => "/tmp"
            }
        }
        ```

    2. Enable and Start Logstash to generate a sample file.

        ```BASH
        sudo systemctl enable logstash
        sudo systemctl start logstash
        ```

    3. Once you have enabled the Logstash service and it has been started check `logstash-plain.log` to confirm there are no errors. Wait until you see confirmation that `sampleFile<epoch seconds>.json` is created.

        ```BASH
        tail -f /var/log/logstash/logstash-plain.log
        ```

        >Note: If you are seeing errors in the log please refer to the [troubleshooting](#troubleshooting) steps below.

    4. Stop Logstash once the sample file has been generated.

        ```BASH
        sudo systemctl stop logstash
        ```

7. Create the `Data Collection Rule`.
    1. If creating a custom Log Analytics table, create one as per the [Microsoft Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#create-new-table-in-log-analytics-workspace). When providing the schema, use the sample file generated in `Step 8`.
    
    2. If not creating a custom table, create the `Data Collection Rule` using the [Microsoft Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-api#create-data-collection-rule). Take note of the `immutableId` from the `JSON View`.

        ![image](https://raw.githubusercontent.com/HartD92/Sentinel/3857dbf6623c033f8415599ae00b74551ccbf4ae/DCRJson.png)
        ![image](https://raw.githubusercontent.com/HartD92/Sentinel/3857dbf6623c033f8415599ae00b74551ccbf4ae/DCRImmutableId.png)


8. Reconfigure the output plugin file to send data to the Sentinel Log Ingestion API using the data gathered in `steps 3, 4, and 7`.

    ```BASH
    sudo nano /etc/logstash/conf.d/50-outputs.conf
    ```

    ```BASH
    output {
        microsoft-sentinel-log-analytics-logstash-output-plugin {
            client_app_Id => ""
            client_app_secret => ""
            tenant_id => ""
            data_collection_endpoint => ""
            dcr_immutable_id => ""
            dcr_stream_name => ""

            create_sample_file => false
            sample_file_path => "/tmp"
        }
    }
    ```

    > NOTE: It is recommended to store this information in the Logstash KeyStore instead of in the pipeline configuration.

9. Restart Logstash

    ```BASH
    sudo systemctl start logstash
    ```

### View Logs in Azure Sentinel

1. Wait for logs to arrive in Azure Sentinel.

  
- It can take up to 30 minutes for the role assignment granted to the Entra ID App registration on the DCR to take effect. Until then, you may see 403 errors in the Logstash log.

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
