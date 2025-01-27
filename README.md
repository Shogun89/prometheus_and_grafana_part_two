# Adding monitoring with Prometheus and Grafana : Part 2
### Setting up a notification interface and alerts

## Introduction

This is a continuation of the work I did in the [previous article](https://open.substack.com/pub/invariantsubspace/p/adding-monitoring-with-prometheus?r=4l2b0p&utm_campaign=post&utm_medium=web). As usual, I recommend reading the previous article to understand the context, however it is not required to follow this article. In the previous article, I added Docker components for Prometheus, Grafana and the associated data exporters needed. I also set up some basic dashboards in Grafana. In this article, I'll be looking at adding alerts to the system, creating a chat interface for the alerts and testing the system. 

In contrast to previous projects I won't be using the complete project from before. Instead I'll be focusing strictly on the Grafana and Prometheus components to make running the project easier. I won't cover all of the changes in detail, but I'll provide enough information to understand the changes and how they work. You can find the previous article [here](https://open.substack.com/pub/invariantsubspace/p/adding-monitoring-with-prometheus?r=4l2b0p&utm_campaign=post&utm_medium=web) along with the code for the project [here](https://github.com/Shogun89/prometheus_and_grafana). The repo for the new project will be [here](https://github.com/Shogun89/prometheus_and_grafana_part_two). Feel free to clone the repo and follow along. As mentioned before there is a docs folder in the repo that contains the documentation for the project. Feel free to read it to understand the changes and how they work.

As an added note I’d like to state ahead of time that a large chunk of this depends upon setting up Slack and other things which are inherently manual. In the past I’ve tried to make following along as simple as running a Docker command but couldn’t do that here. There may be ways of automating the installation and configuration of Slack but I won’t be looking into them for the purposes of this article. Finally, the use of Slack is not really an endorsement of it nor a statement about it being the only way to do this.  You could do this with most any chat interface or even use other notification methods.

## Architecture

As usual I'll be reviewing the architecture of the previous project and the new project to understand the changes but in this case I'll be focusing on the changes to the Grafana and Prometheus components as well as the new chat interface for the alerts.

Below is a diagram of the project architecture up to this point.

![Project Architecture](./images/architecture_diagram_previous.png)

Below is a diagram of the project architecture after the changes.

![Project Architecture](./images/architecture_diagram_current.png)

As mentioned above I’m focusing strictly on Grafana, Prometheus and Slack in this picture so in reality it’s the just the top right hand side. As you can see the basic idea is to create an alert inside of Grafana which which trigger a webhook in Slack. This will then create a notification inside of a Slack workplace. 

## Setting up Slack

### Creating a Slack app

In order to send alerts to Slack, we need to set up a Slack app. This involves creating a new Slack workspace or using an existing one, creating a new app, and configuring the app to send alerts to a specific channel. To do this we need to follow these steps:

1. Download the Slack app from the [Slack website](https://slack.com/intl/en-in/help/articles/218080037-Create-a-Slack-app)
2. Create a new Slack workspace or use an existing one
3. Go to [api.slack.com](https://api.slack.com/) and create a new app
   - Click on "Your Apps"
   ![Your Apps](./images/your_apps.png)
   - Click "Create New App"
   ![New App](./images/create_app.png)
   - Choose "From scratch"
   ![From Scratch](./images/from_scratch.png)
   - Name your app (e.g. "Grafana Alerts")
   ![App Name](./images/app_name.png)
   - Select your workspace

### Configuring the webhook
Now we have an application set up so we can deliver the notifications. After that we need to configure the webhook used to trigger the notification.

- Under "Features", select "Incoming Webhooks"
![Incoming Webhooks](./images/incoming_webhooks.png)
- Activate incoming webhooks
![Activate Incoming Webhooks](./images/activate_webhook.png)
- Click "Add New Webhook to Workspace"
![Add New Webhook to Workspace](./images/add_new_webhook.png)
- Select the channel where you want the alerts to appear
![Select Channel](./images/select_channel.png)
- Copy the curl command and save the webhook URL for later
![Copy Webhook URL](./images/copy_curl.png)
- Now test the command and check your workspace
![Test Command](./images/test_curl.png)

For reference the curl command should look like this:
```bash
curl -X POST -H 'Content-type: application/json' --data '{"text":"Hello, World!"}' https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

Now we’ve configured most of what we need for Slack to generate the notification. We’ve created an app for our workspace, we’ve added a webhook for that workspace, we’ve given it the proper permissions and we’ve tested triggering that webhook.

### Configuring Grafana

For the next portion we need to configure Grafana. If you want to do this you’ll have to download the repo and then run the Docker command to start Grafana and Prometheus to configure the alerts. In order to do this we will do the following.

1. Run the following Docker command:
```bash
docker compose up -d
```
2. Open Grafana in your browser
   - Go to http://localhost:3000/
   - Enter admin for both the user and password
   - Skip creating a new password
3. In Grafana, go to Home > Administration > Alerting 
![Grafana Alerting](./images/grafana_alerting.png)
4. Click on "Contact points" in the left sidebar
5. Click "New contact point"
6. Configure the contact point:
   - Name: "Slack Alerts"
   - Integration: Select "Slack"
   - Webhook URL: Paste your Slack webhook URL
      - If you forgot this then go back to api.slack.com and check the webhook
   - Message Settings: You can customize the message format
   ![Message Settings](./images/grafana_contact_point.png)
7. Try testing the contact 
![Test Contact Point](./images/grafana_test.png)
8. Check your alerting channel to confirm it was sent
![Test Contact Point](./images/grafana_alert_channel.png)
9. Click "Save contact point"

### Creating an alert

Now we need to create an alert rule. Let's create a basic alert that triggers when a service is down:

1. Go to Home > Alerting > Alert rules
![Grafana Alerting](./images/alert_rules.png)
2. Click "New alert rule"
3. Configure the alert:
   - In the "Name" section:
      - Enter a name (e.g. “API A up”)
   - In the "Query" section:
     - Select Prometheus as your data source
     - Choose a metric from the "Metric" dropdown (for example, "up")
   ![New Alert Rule](./images/new_alert_rule.png)
   - In the "Reduce" section:
      - Input : A
      - Operation : Last
   - In the "Threshold" section:
      - Input : B
      - Threshold : IS BELOW
      - Value : 1
   ![Define Rule](./images/define_rule.png)
4. In the "Evaluation Behavior" section:
   - Create a new folder (e.g. alerts)
   - Create a new evaluation group (e.g. service alerts)
   - Set the time period to 5 m
   ![Evaluation Behavior](./images/evaluation_behavior.png)
5. In the "Configure labels and notifications" section:
   - Change the contact point to Slack Alerts
   ![Configure Labels and Notifications](./images/configure_notifications.png)
6. Click "Configure notification message"
   - Set the Summary to ```{{ $labels.job }} is Down```
   - Set the Description to ```{{ $labels.job }} has been down for {{ $duration }}. Current value: {{ $value }}```
   ![Configure Notification Message](./images/configure_message.png)
7. Click "Save rule and exit"

This alert will trigger if the chosen monitored service is down for more than 5 minutes. When triggered, it will send a notification to your configured Slack channel.

## Testing the Alert

There are multiple ways to test this however the simplest way is to simply wait. In the rule I configured I chose API A. If you remember this was one of the FastAPI shard instances however it does not exist in this Docker configuration. It is however still noted in the monitoring Docker config. So our rule will attempt to look for it and since it isn’t on then it will fail and fire. I decided to wait myself and I see this.

### Grafana Alerts

![Grafana Alerting](./images/grafana_alert_fire.png)

### Slack Alert Channel

![Slack Alerting](./images/slack_alert_fire.png)

## Conclusion

In this article, we've successfully:
- Set up Slack integration with Grafana for alert notifications
- Created a basic but effective alert rule to monitor service availability
- Configured alert messages and notification channels
- Learned different ways to test our alerting system

This monitoring setup provides immediate notification when services go down, allowing for faster response times to potential issues. While this is a basic implementation, it serves as a foundation that can be expanded with more sophisticated alerts for metrics like CPU usage, memory consumption, or custom application metrics.

The complete code for this project is available on [GitHub](https://github.com/Shogun89/prometheus_and_grafana_part_two). If you enjoyed this please consider subscribing.


