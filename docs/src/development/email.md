---
title: "Sending E-mail"
weight: 8
sidebarTitle: "E-mail"
description: |
  By default only your production environment can send emails. For non-production environments, you can configure outgoing emails via the [management console](/administration/web/configure-environment.md#outgoing-emails). Emails from Platform.sh are sent via a SendGrid-based SMTP proxy.
---

{{< description >}}

Each Platform.sh project is provisioned as a SendGrid sub-account.
You can use `/usr/sbin/sendmail` on your application container to send emails with the assigned SendGrid sub-account.
Alternatively, you can use the `PLATFORM_SMTP_HOST` environment variable to use in your SMTP configuration."

We do not guarantee the deliverability of emails, and we do not support white-labeling them.
Our SMTP proxy is intended as a zero-configuration, best effort service.
If needed, you can instead use your own SMTP server or email delivery service provider.
In that case, bear in mind that TCP port 25 is blocked for security reasons; use TCP port 465 or 587 instead.

{{< note>}}

You may follow SendGrid's SPF setup guidelines to improve email deliverability with our SMTP proxy, by including the following TXT record to the domain's DNS records:

```txt
>v=spf1 include:sendgrid.net -all
```
{{< /note >}}

## Email domain validation

{{< tiered-feature "Enterprise" >}}

Enterprise or Elite customers can request for DomainKeys Identified Mail (DKIM) to be enabled on their domain.

DKIM improves the delivery rate as an email sender and can be enabled on either your Dedicated or Grid sites.

### Enable DKIM on your domain

To have DKIM enabled for your domain:

1. Open a support ticket with the domain where you want DKIM.
2. Update your DNS configuration with the `CNAME` and `TXT` records that you get in the ticket.
3. Checks for the expected DNS records run every 15 minutes before validation.

{{< note>}}

The TXT record to include your account ID (see [SendGrid's SPF setup guidelines](https://docs.sendgrid.com/ui/account-and-settings/spf-records#custom-spf-records))
looks similar to the following:

```txt
>v=spf1 include:u17504801.wl.sendgrid.net -all
```

{{< /note >}}

## Enabling/disabling email

Email support can be enabled/disabled per-environment.
By default, it's enabled on your production environment and disabled elsewhere.
That can be toggled in through the management console or via the command line, like so:

```bash
platform environment:info enable_smtp true

platform environment:info enable_smtp false
```

When SMTP support is enabled,
the environment variable `PLATFORM_SMTP_HOST` will be populated with the address of the SMTP host that should be used.
When SMTP support is disabled,
that environment variable will be empty.

## Ports

- Port 465 and 587 should be used to send email to your own external email server.
- Port 25 should be used to send through PLATFORM_SMTP_HOST. (this is the default in most mailers).

We proxy your emails through our own SMTP host, and encrypt them over port 465 before sending them through to the outside world.

## Testing the email service

Before testing that the email service is working, make sure that:

- E-mail has been [enabled](#enablingdisabling-email) on the environment
- The environment has been redeployed
- You have accessed the environment using SSH and verified that the `PLATFORM_SMTP_HOST` environment variable is visible

To test the email service, first connect to your cluster through [SSH](/development/ssh/_index.md)
using the [CLI](/development/cli/_index.md) command `platform ssh`.
Run the following command (replacing the email addresses with the ones you want):

```bash
$ php -r 'mail("mail@example.com", "test message", "just testing", "From: tester@example.com");'
```

After a couple minutes you should receive the "test message" in your mailbox.

## Sending email in PHP

When you send email, you can use the built-in `mail()` function in PHP.
The PHP runtime is configured to send email automatically via the assigned SendGrid sub-account.
Note that the `From` header is required; email will not send if that header is missing.

Beware of the potential security problems when using the `mail()` function,
which arise when using user-supplied input in the fifth (`$additional_parameters`) argument.
See the [PHP `mail()` documentation](http://php.net/manual/en/function.mail.php) for more information.

### SwiftMailer

In Symfony, if you use the default `SwiftMailer` service,
we recommend the following settings in your `app/config/parameters.yaml`:

```yaml
parameters:
    mailer_transport: smtp
    mailer_host: "%env(PLATFORM_SMTP_HOST)%"
    mailer_user: null
    mailer_password: null
```

If you are using a file spool facility, you will probably need to setup a read/write mount for it in `.platform.app.yaml`, for example:

```yaml
mounts:
    'app/spool':
        source: local
        source_path: spool
```

## Sending email in Java

JavaMail is a Java API used to send and receive email via SMTP, POP3, and IMAP.
JavaMail is built into the [Jakarta EE](https://jakarta.ee/) platform, but also provides an optional package for use in Java SE.

Jakarta Mail defines a platform-independent and protocol-independent framework to build mail and messaging applications.

Below the sample code that uses [Jakarta Mail](https://projects.eclipse.org/projects/ee4j.mail):

```java
import sh.platform.config.Config;

import javax.mail.Message;
import javax.mail.MessagingException;
import javax.mail.Session;
import javax.mail.Transport;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeMessage;
import java.util.Properties;
import java.util.logging.Level;
import java.util.logging.Logger;

public class JavaEmailSender {

    private static final Logger LOGGER = Logger.getLogger(JavaEmailSender.class.getName());

    public void send() {
        Config config = new Config();
        String to = "";//change accordingly
        String from = "";//change accordingly
        String host = config.getSmtpHost();
        //or IP address
        //Get the session object
        Properties properties = System.getProperties();
        properties.setProperty("mail.smtp.host", host);
        Session session = Session.getDefaultInstance(properties);

        //compose the message
        try {
            MimeMessage message = new MimeMessage(session);
            message.setFrom(new InternetAddress(from));
            message.addRecipient(Message.RecipientType.TO, new InternetAddress(to));
            message.setSubject("Ping");
            message.setText("Hello, this is example of sending email  ");

            // Send message
            Transport.send(message);
            System.out.println("message sent successfully....");

        } catch (MessagingException exp) {
            exp.printStackTrace();
            LOGGER.log(Level.SEVERE, "there is an error to send an message", exp);
        }
    }
}

```



There is plenty of additional documentation about using JavaMail,
like this one [that shows how to send email with HTML formatting and attachments](https://mkyong.com/java/java-how-to-send-email/).

### References

- [Platform.sh Email documentation](/development/email.md)
- [https://mkyong.com/java/java-how-to-send-email/](https://mkyong.com/java/java-how-to-send-email/)
- [JavaMail API](https://javaee.github.io/javamail/)
