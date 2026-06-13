# Standing Up an Nginx Web Server on EC2 and Sending Logs to Amazon CloudWatch

## Introduction

In this hands-on lab, you will configure an Amazon EC2 instance running **Nginx** to push custom log files to Amazon CloudWatch using the CloudWatch agent. Once the logs are flowing into CloudWatch, you will build a simple alerting system that detects **HTTP 500 Internal Server Error** responses using Metric Filters, and delivers notifications via **Amazon SNS email**.

---

## Solution

Log in to the AWS Management Console using the credentials provided on the lab instructions page. Make sure you are using the **us-east-1** region.

---

## 1. Install and Configure the Amazon CloudWatch Agent

1. In the console, navigate to **Amazon EC2**.

2. Locate the **StagingWebServer** instance and select it.

3. Under the **Details** tab, click the link next to **Public IP address** to open it in a new tab.

4. Edit the URL to use `http://` (not `https://`) — you should see the **"Great, you did it and it works!"** webpage. Leave this tab open for later.

5. Back in the EC2 console, click **Connect**.

6. Under the **Session Manager** tab, click **Connect**.

7. Once connected, install the CloudWatch agent:

   ```bash
   sudo yum install -y amazon-cloudwatch-agent
   ```

8. Create the agent configuration file:

   ```bash
   sudo vim /tmp/cloudwatch-agent-config.json
   ```

9. Inside the file, paste the following configuration:

   ```json
   {
     "logs": {
       "logs_collected": {
         "files": {
           "collect_list": [
             {
               "file_path": "/var/log/nginx/access.log",
               "log_group_name": "Nginx-Access-Logs",
               "log_stream_name": "{instance_id}"
             },
             {
               "file_path": "/var/log/nginx/error.log",
               "log_group_name": "Nginx-Error-Logs",
               "log_stream_name": "{instance_id}"
             }
           ]
         }
       }
     }
   }
   ```

10. Save and exit the file by pressing `Esc`, then typing `:wq!` and pressing `Enter`.

11. Start the CloudWatch agent and load the new configuration:

    ```bash
    sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
      -a fetch-config -m ec2 \
      -c file:/tmp/cloudwatch-agent-config.json -s
    ```

12. Navigate back to your browser tab with the Nginx page and refresh it several times to generate access log entries.

13. In the AWS console, navigate to **Amazon CloudWatch**.

14. In the left menu under **Logs**, select **Log Groups**.

15. Select **Nginx-Access-Logs**.

16. Under the **Log streams** tab at the bottom, select the stream that matches your instance ID.

17. Click **Start tailing** in the top right corner.

18. Switch back to the Nginx page and refresh it several times (ensure you are using `http://`, not `https://`).

    You should see new log entries appearing in real time in the tailing session.

---

## 2. Generate 500 Errors and Create a Metric Filter

Now let's deliberately trigger some HTTP 500 errors so we have something to filter on.

1. In your Session Manager terminal, create a broken PHP-style endpoint by writing a script that Nginx cannot execute:

   ```bash
   echo '<?php this_is_broken; ?>' | sudo tee /usr/share/nginx/html/broken.html
   ```

2. In your browser, navigate to:

   ```
   http://YOUR_PUBLIC_IP/broken.html
   ```

   Nginx will return an **HTTP 500** response. Refresh this page a few times to generate multiple entries.

3. Navigate back to the **Nginx-Access-Logs** log stream in CloudWatch and check the tailing session. You should see entries containing `500`.

4. Cancel the tailing session and navigate back to the **Nginx-Access-Logs** Log Group.

5. In the bottom pane, select the **Metric filters** tab.

6. Click **Create metric filter**.

7. For **Filter pattern**, enter the following to match any HTTP 500 status codes:

   ```
   %\b500\b%
   ```

8. Under **Test pattern**, select your instance ID from the **Select log data to test** dropdown, then click **Test pattern**. Confirm that filtered results appear.

9. Click **Next**.

10. For **Filter name**, enter: `500-Errors`

11. Under **Metric details**:
    - **Metric namespace**: `Nginx` (enable **Create new**)
    - **Metric name**: `500`
    - **Metric value**: `1`
    - **Default value**: leave blank
    - **Unit**: `Count`

12. Click **Next**, review the settings, then click **Create metric filter**.

13. Navigate to **CloudWatch → Metrics → All Metrics**.

14. Refresh your broken URL a few more times to generate metrics. You should soon see the **Nginx** custom namespace appear.

---

## 3. Create an SNS Topic and CloudWatch Alarm

Now let's wire up an alarm that fires whenever a 500 error is detected.

1. In **CloudWatch**, confirm you are in **us-east-1**, then select the **Nginx** custom namespace.

2. Select **Metrics with no dimensions**.

3. Check the **500** metric so it appears on the graph.

4. Click the **Custom calendar** button at the top of the graph and select **5 minutes** from the dropdown.

5. Click the dropdown next to the refresh button and set auto-refresh to **10 seconds**.

6. Refresh the CloudWatch page to reveal new options in the **Graphed metrics** section.

7. Locate the **Period** for the `500` metric and change it to **1 minute**.

8. Change the **Statistic** to **Sum**.

9. Under the **Actions** column, click the alarm bell icon to **Create alarm**.

10. Confirm the metric pane shows:
    - **Metric name**: `500`
    - **Period**: 1 minute
    - **Statistic**: Sum

11. Under **Conditions**:
    - **Threshold type**: Static
    - **Whenever 500 is…**: Greater
    - **than**: `0`

12. Click **Next**.

13. For **Notification**:
    - **Alarm state trigger**: In alarm
    - **SNS topic**: Create new topic
    - **Topic name**: `500-Alerts`
    - **Email endpoint**: enter your email address
    - Click **Create topic**

14. **Important**: open the confirmation email from AWS and click the subscription confirmation link before proceeding.

15. Click **Next**.

16. For **Alarm name**, enter: `500-Detections`. Optionally add a description.

17. Click **Next**, review all settings, then click **Create alarm**.

18. Navigate back to `http://YOUR_PUBLIC_IP/broken.html` and refresh several times to generate more 500 errors.

19. In the CloudWatch console, wait a few minutes for the alarm to move out of the **Insufficient data** state and into **In alarm** (refresh as needed).

    Once triggered, you will receive an SNS email notification at the address you subscribed to the `500-Alerts` topic.

---

## Summary

In this lab you:

- Configured the **CloudWatch agent** to ship Nginx access and error logs to CloudWatch Log Groups
- Created a **Metric Filter** to detect HTTP **500** status codes in the log stream
- Set up a **CloudWatch Alarm** with a **Sum > 0** condition on a 1-minute period
- Wired the alarm to an **SNS email topic** (`500-Alerts`) for real-time notifications
