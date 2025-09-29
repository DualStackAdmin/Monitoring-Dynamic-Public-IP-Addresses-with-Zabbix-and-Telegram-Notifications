# Monitoring Dynamic Public IP Addresses with Zabbix and Telegram Notifications

This repository provides a complete guide and the necessary configuration files to monitor dynamic public IP address changes on servers using Zabbix and send instant, formatted notifications to Telegram. This solution is ideal for environments with dynamic IPs, such as Starlink, failover internet connections, or other similar setups.

### Final Notification Format Example:
![Successful Telegram Notification](image_2ad36d.png)

---

## ⚙️ Architecture and Workflow
The system consists of the following components and works according to this logic:
1.  **Zabbix Agent (`UserParameter`)**: The agent on the monitored server learns its own public IP address using a custom command.
2.  **Zabbix Template (`Item` & `Trigger`)**: A Zabbix template collects this IP data from the agent regularly (`Item`) and creates an alert (`Trigger`) whenever the IP address changes.
3.  **Zabbix Action**: As soon as the alert is triggered, an Action rule is executed.
4.  **Telegram Webhook**: The Action sends a pre-formatted message to your Telegram bot using Zabbix's built-in Telegram media type.

---

## 🛠️ Step-by-Step Installation Guide

### Stage 1: Agent-Side Configuration (On the Monitored Server)
These steps "teach" the Zabbix agent how to find its own public IP address.

1.  **Create the Custom Parameter (UserParameter):**
    Create a new key named `public.ip` in the agent's configuration directory. This key will return the server's public IP via a `curl` command when called.
    ```bash
    echo 'UserParameter=public.ip,curl -s ifconfig.me' | sudo tee /etc/zabbix/zabbix_agent2.d/public_ip.conf
    ```
2.  **Restart the Agent:**
    Restart the agent for the new configuration to take effect.
    ```bash
    sudo systemctl restart zabbix-agent2
    ```
3.  **Verify the Configuration:**
    Test that the parameter is working correctly. This command should return your server's public IP address.
    ```bash
    zabbix_agent2 -t public.ip
    ```

### Stage 2: Importing the Zabbix Template
The `Public IP Monitoring.yaml` file provided in this repository contains the necessary Item and Trigger configuration.

1.  Download the `Public IP Monitoring.yaml` file to your computer.
2.  In the Zabbix frontend, navigate to `Data collection` -> `Templates`.
3.  Click the **`Import`** button in the top-right corner.
4.  Choose the YAML file you downloaded and complete the import by clicking the **`Import`** button.

> **Template Contents:**
> * **Item (`Server Public IP Address`):** Collects data using the `public.ip` key every 5 minutes.
> * **Trigger (`Public IP Address on {HOST.NAME} has CHANGED!`):** Activates when the last received IP value is different from the previous one. The **"Problem event generation mode"** is set to **`Multiple`** to ensure that every IP change (even flapping between two IPs) generates a new notification.

### Stage 3: Setting up the Telegram Notification Channel
1.  Create a bot with **`@BotFather`** in Telegram to get your **HTTP API Token**, and get your unique **Chat ID** from **`@userinfobot`**.
2.  In Zabbix, navigate to `Alerts` -> `Media types` and select `Telegram`. Configure the **`Parameters`** tab with the following 8 parameters (delete any old ones and add these):

    * `alert_message`:
      ```html
      <b>{EVENT.SEVERITY}</b>: {EVENT.NAME}<b>Host</b>: {HOST.NAME}<b>Time</b>: {EVENT.TIME} on {EVENT.DATE}<pre>{ITEM.NAME}: {ITEM.VALUE}{TRIGGER.URL}</pre>
      ```
    * `alert_subject`: `{EVENT.STATUS}: {EVENT.NAME} on {HOST.NAME}`
    * `api_chat_id`: Your **Chat ID**.
    * `api_parse_mode`: `HTML`
    * `api_token`: Your **Bot Token**.
    * `event_nseverity`: `{EVENT.NSEVERITY}`
    * `event_update_status`: `{EVENT.UPDATE.STATUS}`
    * `event_value`: `{EVENT.VALUE}`

3.  Navigate to `Alerts` -> `Users`, go to your user profile, and add your `Telegram` media with your **Chat ID** in the `Media` tab.

### Stage 4: Creating the Action (Trigger Action)
This is the logic that connects the trigger to the Telegram notification.

1.  Navigate to `Alerts` -> `Actions` -> `Trigger actions` and create a new Action.
2.  In the **`Conditions`** tab, create the following universal condition to ensure the action works for any host using the template:
    * **Type:** `Event name`
    * **Operator:** `contains`
    * **Value:** `Public IP Address has CHANGED!`
3.  In the **`Operations`** tab, create message templates for problem and recovery notifications.
    * **Problem Message:**
      * **Subject:** `🚨 IP Changed: {HOST.NAME}`
      * **Message:**
        ```html
        🚨 <b>Public IP Address Changed!</b> 🚨<b>Server:</b> {HOST.NAME}<b>Problem:</b> {TRIGGER.NAME}<b>New IP Address:</b> {ITEM.LASTVALUE1}<b>Time of Change:</b> {EVENT.DATE} {EVENT.TIME}
        ```
    * **Recovery Message:**
      * **Subject:** `✅ OK: IP Address Stable on {HOST.NAME}`
      * **Message:**
        ```html
        ✅ <b>Resolved: Public IP Address is Stable</b><b>Server:</b> {HOST.NAME}<b>Problem:</b> {TRIGGER.NAME}<b>Time of Recovery:</b> {EVENT.RECOVERY.DATE} {EVENT.RECOVERY.TIME}
        ```

### Stage 5: Putting It All Together
1.  Navigate to `Data collection` -> `Hosts` and select the host you want to monitor.
2.  Go to the `Templates` tab.
3.  In the "Link new templates" field, find and add the **`Public IP Monitoring`** template you just imported.
4.  Click the **`Update`** button.

**Setup is complete!** Now, any host to which you apply this template will automatically detect public IP changes and send you a formatted notification via Telegram, just like the one shown in the example.
