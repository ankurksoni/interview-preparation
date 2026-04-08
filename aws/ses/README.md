# Amazon SES Interview Questions & Answers

A curated list of **Amazon Simple Email Service (SES) interview questions** with practical, production-grade answers. Covers email sending, deliverability, authentication, templates, event tracking, and real-world integration patterns.

---

### 1. **What is Amazon SES?**

**Answer:** Amazon SES is a cloud-based email sending and receiving service. It's used for transactional emails (order confirmations, password resets), marketing emails, and bulk notifications. SES handles the underlying email infrastructure — SMTP, DKIM, SPF, bounce handling, and reputation management.

---

### 2. **What is the difference between SES sandbox and production?**

**Answer:**

| Feature     | Sandbox                  | Production                    |
| ----------- | ------------------------ | ----------------------------- |
| Send to     | Verified addresses only  | Any address                   |
| Daily limit | 200 emails/day           | Configurable (up to millions) |
| Rate limit  | 1 email/second           | 50+/second (adjustable)       |
| Setup       | Default for new accounts | Requires manual request       |

Request production access via AWS Console → SES → Account Dashboard → Request Production Access.

---

### 3. **How do you send an email using the SES v2 SDK?**

**Answer:**

```ts
import { SESv2Client, SendEmailCommand } from "@aws-sdk/client-sesv2";

const client = new SESv2Client({});

await client.send(
  new SendEmailCommand({
    FromEmailAddress: "noreply@myapp.com",
    Destination: {
      ToAddresses: ["user@example.com"],
    },
    Content: {
      Simple: {
        Subject: { Data: "Your order has shipped!" },
        Body: {
          Html: { Data: "<h1>Order #123 shipped</h1><p>Track it here...</p>" },
          Text: { Data: "Order #123 shipped. Track it here..." },
        },
      },
    },
  }),
);
```

> **Note:** Always include both HTML and plain text body for maximum deliverability.

---

### 4. **What are SPF, DKIM, and DMARC, and why do they matter?**

**Answer:** These are email authentication protocols that prevent spoofing and improve deliverability.

- **SPF (Sender Policy Framework)** — DNS TXT record declaring which servers can send email for your domain
- **DKIM (DomainKeys Identified Mail)** — Cryptographic signature added to email headers; SES generates CNAME records you add to DNS
- **DMARC (Domain-based Message Authentication)** — Policy that tells receivers what to do when SPF/DKIM fail

```
; SPF record
v=spf1 include:amazonses.com ~all

; DMARC record
_dmarc.myapp.com  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@myapp.com"
```

Without these, your emails are likely to land in **spam folders** or be **rejected entirely** by Gmail, Outlook, etc.

---

### 5. **How do you handle bounces and complaints in production?**

**Answer:** SES publishes bounce and complaint notifications via SNS. You **must** handle these to maintain sender reputation.

```ts
// SNS → Lambda handler for bounce/complaint processing
export const handler = async (event: any) => {
  for (const record of event.Records) {
    const message = JSON.parse(record.Sns.Message);

    if (message.notificationType === "Bounce") {
      const bounced = message.bounce.bouncedRecipients;
      for (const recipient of bounced) {
        // ✅ Remove hard bounces from your mailing list immediately
        await suppressEmail(recipient.emailAddress);
        console.log(`Suppressed bounced email: ${recipient.emailAddress}`);
      }
    }

    if (message.notificationType === "Complaint") {
      const complained = message.complaint.complainedRecipients;
      for (const recipient of complained) {
        // ✅ Unsubscribe users who mark you as spam
        await unsubscribeUser(recipient.emailAddress);
      }
    }
  }
};
```

**Failure to handle bounces/complaints will result in AWS suspending your SES account.**

---

### 6. **What is the SES suppression list?**

**Answer:** SES maintains an **account-level suppression list** that automatically suppresses sending to addresses that previously bounced or complained. You can also manage it manually.

```ts
import {
  SESv2Client,
  PutSuppressedDestinationCommand,
} from "@aws-sdk/client-sesv2";

// Manually add an address to suppression list
await client.send(
  new PutSuppressedDestinationCommand({
    EmailAddress: "invalid@example.com",
    Reason: "BOUNCE",
  }),
);
```

---

### 7. **How do you use SES email templates for transactional emails?**

**Answer:** Templates allow reusable email layouts with dynamic data, processed server-side by SES.

```ts
import {
  SESv2Client,
  CreateEmailTemplateCommand,
  SendEmailCommand,
} from "@aws-sdk/client-sesv2";

// Create template once
await client.send(
  new CreateEmailTemplateCommand({
    TemplateName: "OrderConfirmation",
    TemplateContent: {
      Subject: "Order {{orderId}} confirmed!",
      Html: `
      <h1>Thanks, {{name}}!</h1>
      <p>Your order <b>{{orderId}}</b> for <b>{{amount}}</b> is confirmed.</p>
    `,
      Text: "Thanks, {{name}}! Order {{orderId}} for {{amount}} is confirmed.",
    },
  }),
);

// Send using template
await client.send(
  new SendEmailCommand({
    FromEmailAddress: "orders@myapp.com",
    Destination: { ToAddresses: ["customer@example.com"] },
    Content: {
      Template: {
        TemplateName: "OrderConfirmation",
        TemplateData: JSON.stringify({
          name: "Ankur",
          orderId: "ORD-123",
          amount: "$99.99",
        }),
      },
    },
  }),
);
```

---

### 8. **How do you send bulk emails efficiently with SES?**

**Answer:** Use `SendBulkEmail` API to send personalized emails to many recipients in a single API call.

```ts
import { SESv2Client, SendBulkEmailCommand } from "@aws-sdk/client-sesv2";

await client.send(
  new SendBulkEmailCommand({
    FromEmailAddress: "marketing@myapp.com",
    DefaultContent: {
      Template: {
        TemplateName: "WeeklyNewsletter",
        TemplateData: JSON.stringify({ defaultName: "Subscriber" }),
      },
    },
    BulkEmailEntries: [
      {
        Destination: { ToAddresses: ["user1@example.com"] },
        ReplacementEmailContent: {
          ReplacementTemplate: {
            ReplacementTemplateData: JSON.stringify({ name: "Alice" }),
          },
        },
      },
      {
        Destination: { ToAddresses: ["user2@example.com"] },
        ReplacementEmailContent: {
          ReplacementTemplate: {
            ReplacementTemplateData: JSON.stringify({ name: "Bob" }),
          },
        },
      },
    ],
  }),
);
```

---

### 9. **How do you track email opens, clicks, and delivery events?**

**Answer:** Enable **event publishing** with a configuration set. SES can send events to SNS, Kinesis Firehose, CloudWatch, or EventBridge.

Tracked event types: `send`, `delivery`, `open`, `click`, `bounce`, `complaint`, `reject`, `renderingFailure`

```ts
// CDK example: SES → EventBridge → Lambda
const configSet = new ses.ConfigurationSet(this, "EmailTracking", {
  configurationSetName: "my-tracking-set",
});

// Add EventBridge as event destination
configSet.addEventDestination("EventBridgeDest", {
  destination: ses.EventDestination.eventBridge(
    events.EventBus.fromEventBusName(this, "Default", "default"),
  ),
  events: [
    ses.EmailSendingEvent.OPEN,
    ses.EmailSendingEvent.CLICK,
    ses.EmailSendingEvent.BOUNCE,
    ses.EmailSendingEvent.DELIVERY,
  ],
});
```

Then send emails with the configuration set:

```ts
await client.send(
  new SendEmailCommand({
    FromEmailAddress: "noreply@myapp.com",
    Destination: { ToAddresses: ["user@example.com"] },
    Content: {
      /* ... */
    },
    ConfigurationSetName: "my-tracking-set",
  }),
);
```

---

### 10. **What are dedicated IP addresses in SES and when should you use them?**

**Answer:**

| Feature    | Shared IPs                  | Dedicated IPs           |
| ---------- | --------------------------- | ----------------------- |
| Cost       | Included                    | ~$24.95/month per IP    |
| Reputation | Shared with other SES users | Your reputation only    |
| Warm-up    | Pre-warmed                  | Must warm up manually   |
| Use case   | Low-to-medium volume        | High volume (100K+/day) |

Use dedicated IPs when you send high volumes and need full control over your IP reputation. For most applications, shared IPs with good email practices are sufficient.

---

### 11. **How do you implement email rate limiting to protect sender reputation?**

**Answer:**

```ts
// ❌ Bad: Blasting all emails at once
for (const user of users) {
  await sendEmail(user.email, template);
}

// ✅ Good: Respect SES sending rate with controlled concurrency
import pLimit from "p-limit";

const limit = pLimit(10); // 10 concurrent sends (adjust to your SES rate limit)

const promises = users.map((user) =>
  limit(() => sendEmail(user.email, template)),
);

await Promise.allSettled(promises);
```

Check your current sending rate with:

```ts
const account = await client.send(new GetAccountCommand({}));
console.log(account.SendQuota); // { Max24HourSend, MaxSendRate, SentLast24Hours }
```

---

### 12. **How do you receive incoming emails with SES?**

**Answer:** SES can receive emails for verified domains and process them with receipt rules.

**Flow:** Incoming email → SES → Receipt Rule Set → Actions (S3, Lambda, SNS, WorkMail)

```ts
// CDK: Store incoming emails in S3 and trigger Lambda
const receiptRule = new ses.ReceiptRule(this, "IncomingRule", {
  ruleSet,
  recipients: ["support@myapp.com"],
  actions: [
    new sesActions.S3({ bucket: emailBucket, objectKeyPrefix: "incoming/" }),
    new sesActions.Lambda({ function: processEmailFn }),
  ],
});
```

---

### 13. **How do you prevent your emails from going to spam?**

**Answer:** Deliverability best practices:

1. **Authenticate** — Set up SPF, DKIM, and DMARC for your domain
2. **Use a subdomain** — Send from `mail.myapp.com` instead of `myapp.com` to isolate reputation
3. **Handle bounces** — Remove hard bounces immediately
4. **Respect complaints** — Unsubscribe users who complain, aim for < 0.1% complaint rate
5. **Warm up** — Gradually increase sending volume for new domains/IPs
6. **Content quality** — Avoid spam trigger words, include unsubscribe links
7. **List hygiene** — Only send to opted-in recipients, remove inactive subscribers

---

### 14. **What are configuration sets and when should you use them?**

**Answer:** Configuration sets group delivery tracking rules. Attach one to every `SendEmail` call for:

- **Event tracking** — Opens, clicks, bounces, complaints → SNS/EventBridge/CloudWatch
- **IP pool management** — Route different email types through different IP pools
- **Sending authorization** — Delegate sending permission to other accounts

You should use configuration sets in every production SES implementation.

---

### 15. **How do you implement a transactional email service in a microservices architecture?**

**Answer:** Centralize email sending behind a dedicated service/queue to decouple email logic from business services.

```
OrderService → [OrderPlaced event] → EventBridge
    → EmailService (Lambda) → SES → Customer inbox

UserService → [PasswordResetRequested event] → EventBridge
    → EmailService (Lambda) → SES → User inbox
```

```ts
// Centralized email handler Lambda
export const handler = async (event: any) => {
  const { detailType, detail } = event;

  const templateMap: Record<string, string> = {
    OrderPlaced: "OrderConfirmation",
    PasswordResetRequested: "PasswordReset",
    WelcomeUser: "Welcome",
  };

  const templateName = templateMap[detailType];
  if (!templateName) return;

  await sesClient.send(
    new SendEmailCommand({
      FromEmailAddress: `noreply@myapp.com`,
      Destination: { ToAddresses: [detail.email] },
      Content: {
        Template: {
          TemplateName: templateName,
          TemplateData: JSON.stringify(detail),
        },
      },
      ConfigurationSetName: "transactional-tracking",
    }),
  );
};
```
