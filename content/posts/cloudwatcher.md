---
title: "AWS Cloudwatch Logs Monitoring with a Web-Based Desktop App"
date: 2023-09-16T10:30:00+01:00
tags: 
    - "aws"
    - "react"
    - "electron"
---

Check the github for this project: https://github.com/jcgloria/cloudwatcher

This project aims to remove the need to login to the AWS console each time you want to check your Cloudwatch logs. It is a desktop app built with React and Electron that displays the logs events of your log groups in a web-based desktop app. The application lets you filter by dates and log groups. 

With the left panel, the user can filter log groups by prefix. Then, the user can select a log group and see the logs in the right panel. By using the date pickers on the right, the log events will be filtered by the selected date range. With the buttons on the left, the user can set a relative date range (e.g. last 5 days, last 2 weeks, etc.).

![Cloudwatcher](/images/cloudwatcher.png)

The app requires to set aws access and secret keys using the gear icon in the top right. The application only requires the `FilterLogEvents` and `DescribeLogGroups` actions to work correctly. 