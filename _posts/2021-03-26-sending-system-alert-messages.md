---
layout: post
title: Sending System Alert Messages
date: 2021-03-26 19:00 +0000
---

A common issue I found myself faced with in the past was logging and error handling within your NetSuite scripts.

Adding in error logs and exception handling is really cool and all, but, what about if you don't check your logs?

For example, say you have a RESTlet for receiving third-party transactional data and suddenly the payload of that request changes - sure, it may log out an error after attempting to store it but how long would it take you to notice that log? As a full time developer, you can't be expected to have each and every one of your scripts execution logs open and refresh it occasionally to check for new logs.

To overcome this, I thought of a solution that involved using the company's communication tool - MS Teams in this case.
Whenever an error occurred that we deemed critical we would then send out a message to a Teams channel via a webhook. For example, a new employee failed to be created in NetSuite through the RESTlet - send out an alert with some valuable information so it's then visible to people who are maintaining their NetSuite environment.

Personally I think the biggest benefit is having that instant feedback feeding directly into a tool you already use on a regular basis and you're able to address potential issues before they're raised from other colleagues in the business.

## Implementation

I started this off by creating a very simple custom record in NetSuite named 'Teams Webhooks' which consists of two fields: webhook name/title and the webhook URL.

Once that was set-up, I went on to create a 'library' script (outlined below) which could then be re-used in any existing SuiteScripts with the benefit of not needing to re-write the implementation every time we needed to send out a Teams alert.

I'm a big fan of using interfaces as it helps any other developers understand what exactly is needed in order to send out a system alert.

Please get in touch with me via the NetSuite Professionals Slack channel (Ryan Jones) and let me know your thoughts about this post/concept.

#### Usage

```typescript
sendTeamsAlert({
    title: 'Employee RESTlet Error'
    message: 'Employee A failed to store in NS: ' + exception message/reason
    webhookRecordID: 1
});
```

```typescript
/**
 * @name - Teams Alerter
 * @description - A re-usable lib to be included in NS scripts for posting errors to Teams
 */
import https = require('N/https');
import log = require('N/log');
import record = require('N/record');

/**
 *
 * @param title
 * @param message
 * @param webhookRecordID
 */
export function sendTeamsAlert(options: sendTeamsAlertOptions): number {
    try {
        let webhookRecord = record.load({
            type: record.Type.CUSTOM_RECORD + '_teams_webhooks',
            id: options.webhookRecordID,
        });

        let webhookUrl = webhookRecord.getValue({
            fieldId: 'custrecord_webhook_url',
        }) as string;

        const headers = {
            name: 'Content-Type',
            value: 'application/json',
        };

        let body = {
            title: options.title,
            text: options.message,
        };

        const response = https.post({
            url: webhookUrl,
            headers: headers,
            body: JSON.stringify(body),
        });

        return response.code;
    } catch (error) {
        log.error({
            title: 'Error Posting Message',
            details: error.message,
        });
    }
}

interface sendTeamsAlertOptions {
    /** A title for the alert. */
    title: string;
    /** A message for the alert. */
    message: string;
    /** NetSuite Record ID of the webook to be used. */
    webhookRecordID: number;
}
```