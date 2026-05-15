# 📧 Salesforce Apex Email Notification with Visualforce — Complete Guide

## Problem Statement
Develop an Apex program that sends an email notification to a specified email address using Salesforce email services. The program should define the recipient email, subject, and message body, and send the email (with/without attachment) using the built-in Messaging class. Display appropriate messages for invalid email IDs using a Visualforce page frontend.

---

## Prerequisites
- Salesforce Developer Account (free at [developer.salesforce.com](https://developer.salesforce.com))
- Basic knowledge of Apex and Visualforce
- Access to Salesforce Setup

---

## Architecture Overview

```
Visualforce Page (Frontend)
        ↓ (user fills form)
Apex Controller (Backend)
        ↓ (validates email)
Messaging.SingleEmailMessage
        ↓ (sends email)
    Recipient Inbox
```

---

## FILES TO CREATE

| File | Type | Purpose |
|------|------|---------|
| `EmailSenderController.cls` | Apex Class | Backend logic |
| `EmailSenderPage.vfp` | Visualforce Page | Frontend UI |

---

## STEP 1 — Create the Apex Controller Class

### Navigate to:
**Setup → Developer Console → File → New → Apex Class**

Name it: `EmailSenderController`

### Code: `EmailSenderController.cls`

```apex
public class EmailSenderController {

    // Properties bound to Visualforce page
    public String recipientEmail { get; set; }
    public String emailSubject  { get; set; }
    public String emailBody     { get; set; }
    public String statusMessage { get; set; }
    public Boolean isSuccess    { get; set; }
    public Boolean hasAttachment { get; set; }
    public transient Blob attachmentBody { get; set; }
    public String attachmentName { get; set; }

    // File upload support
    public transient Blob fileBody { get; set; }
    public String fileName { get; set; }

    // Constructor
    public EmailSenderController() {
        recipientEmail  = '';
        emailSubject    = '';
        emailBody       = '';
        statusMessage   = '';
        isSuccess       = false;
        hasAttachment   = false;
        attachmentName  = '';
        fileName        = '';
    }

    // Email validation method
    private Boolean isValidEmail(String email) {
        if (String.isBlank(email)) return false;
        String emailRegex = '^[a-zA-Z0-9._|\\\\%#~`=?&/$^*!}{+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z]{2,4}$';
        Pattern emailPattern = Pattern.compile(emailRegex);
        Matcher emailMatcher = emailPattern.matcher(email);
        return emailMatcher.matches();
    }

    // Main method to send email
    public PageReference sendEmail() {
        isSuccess = false;
        statusMessage = '';

        // Validate email
        if (String.isBlank(recipientEmail)) {
            statusMessage = '❌ Error: Recipient email address cannot be empty.';
            return null;
        }

        if (!isValidEmail(recipientEmail)) {
            statusMessage = '❌ Error: Invalid email address format. Please enter a valid email (e.g., user@example.com).';
            return null;
        }

        if (String.isBlank(emailSubject)) {
            statusMessage = '❌ Error: Email subject cannot be empty.';
            return null;
        }

        if (String.isBlank(emailBody)) {
            statusMessage = '❌ Error: Email body cannot be empty.';
            return null;
        }

        try {
            // Create email message
            Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();

            // Set recipient
            email.setToAddresses(new String[] { recipientEmail });

            // Set subject
            email.setSubject(emailSubject);

            // Set body (plain text + HTML)
            email.setPlainTextBody(emailBody);
            email.setHtmlBody('<p>' + emailBody.replace('\n', '<br/>') + '</p>');

            // Handle attachment if provided
            if (hasAttachment && fileBody != null && !String.isBlank(fileName)) {
                Messaging.EmailFileAttachment attachment = new Messaging.EmailFileAttachment();
                attachment.setFileName(fileName);
                attachment.setBody(fileBody);
                attachment.setContentType('application/octet-stream');
                email.setFileAttachments(new Messaging.EmailFileAttachment[] { attachment });
            }

            // Send the email
            Messaging.SendEmailResult[] results = Messaging.sendEmail(
                new Messaging.SingleEmailMessage[] { email }
            );

            // Check result
            if (results[0].isSuccess()) {
                isSuccess = true;
                statusMessage = '✅ Success! Email has been sent to ' + recipientEmail;
                // Clear form after success
                recipientEmail = '';
                emailSubject   = '';
                emailBody      = '';
                fileBody       = null;
                fileName       = '';
                hasAttachment  = false;
            } else {
                Messaging.SendEmailError[] errors = results[0].getErrors();
                statusMessage = '❌ Failed to send email: ' + errors[0].getMessage();
            }

        } catch (Exception e) {
            statusMessage = '❌ Exception: ' + e.getMessage();
        }

        return null;
    }

    // Clear/reset the form
    public PageReference clearForm() {
        recipientEmail = '';
        emailSubject   = '';
        emailBody      = '';
        statusMessage  = '';
        isSuccess      = false;
        hasAttachment  = false;
        fileBody       = null;
        fileName       = '';
        return null;
    }
}
```

---

## STEP 2 — Create the Visualforce Page

### Navigate to:
**Setup → Developer Console → File → New → Visualforce Page**

Name it: `EmailSenderPage`

### Code: `EmailSenderPage.vfp`

```xml
<apex:page controller="EmailSenderController" showHeader="true" sidebar="false">

    <!-- Inline CSS Styling -->
    <style>
        body {
            font-family: 'Salesforce Sans', Arial, sans-serif;
            background-color: #f3f3f3;
            margin: 0;
            padding: 20px;
        }
        .email-container {
            max-width: 650px;
            margin: 30px auto;
            background: #ffffff;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.15);
            overflow: hidden;
        }
        .email-header {
            background: linear-gradient(135deg, #1589EE, #0070D2);
            color: white;
            padding: 25px 30px;
            text-align: center;
        }
        .email-header h1 {
            margin: 0;
            font-size: 24px;
            font-weight: 600;
        }
        .email-header p {
            margin: 8px 0 0;
            font-size: 14px;
            opacity: 0.85;
        }
        .email-body {
            padding: 30px;
        }
        .form-group {
            margin-bottom: 20px;
        }
        .form-group label {
            display: block;
            font-weight: 600;
            font-size: 13px;
            color: #444;
            margin-bottom: 6px;
            text-transform: uppercase;
            letter-spacing: 0.5px;
        }
        .form-group input[type="text"],
        .form-group input[type="email"],
        .form-group textarea {
            width: 100%;
            padding: 10px 14px;
            border: 1px solid #ddd;
            border-radius: 4px;
            font-size: 14px;
            color: #333;
            box-sizing: border-box;
            transition: border-color 0.2s;
        }
        .form-group input:focus,
        .form-group textarea:focus {
            outline: none;
            border-color: #1589EE;
            box-shadow: 0 0 0 3px rgba(21,137,238,0.15);
        }
        .form-group textarea {
            height: 130px;
            resize: vertical;
        }
        .attachment-section {
            background: #f8f9fa;
            border: 1px dashed #ccc;
            border-radius: 4px;
            padding: 15px;
            margin-bottom: 20px;
        }
        .attachment-section label {
            font-weight: 600;
            font-size: 13px;
            color: #444;
        }
        .checkbox-group {
            display: flex;
            align-items: center;
            gap: 8px;
            margin-bottom: 10px;
        }
        .btn-send {
            background: linear-gradient(135deg, #1589EE, #0070D2);
            color: white;
            border: none;
            padding: 12px 30px;
            border-radius: 4px;
            font-size: 15px;
            font-weight: 600;
            cursor: pointer;
            width: 100%;
            transition: opacity 0.2s;
        }
        .btn-send:hover {
            opacity: 0.9;
        }
        .btn-clear {
            background: #f4f4f4;
            color: #555;
            border: 1px solid #ddd;
            padding: 10px 20px;
            border-radius: 4px;
            font-size: 14px;
            cursor: pointer;
            width: 100%;
            margin-top: 10px;
        }
        .btn-clear:hover {
            background: #e8e8e8;
        }
        .status-message {
            padding: 12px 16px;
            border-radius: 4px;
            margin-bottom: 20px;
            font-size: 14px;
            font-weight: 500;
        }
        .status-success {
            background: #e8f5e9;
            border-left: 4px solid #4CAF50;
            color: #2e7d32;
        }
        .status-error {
            background: #fdecea;
            border-left: 4px solid #e53935;
            color: #b71c1c;
        }
        .required {
            color: #e53935;
            margin-left: 2px;
        }
        .divider {
            border: none;
            border-top: 1px solid #eee;
            margin: 25px 0;
        }
    </style>

    <div class="email-container">

        <!-- Header -->
        <div class="email-header">
            <h1>📧 Email Notification Sender</h1>
            <p>Send emails with or without attachments using Salesforce</p>
        </div>

        <!-- Body -->
        <div class="email-body">

            <apex:form enctype="multipart/form-data">

                <!-- Status Message -->
                <apex:outputPanel rendered="{!NOT(ISBLANK(statusMessage))}">
                    <div class="status-message {!IF(isSuccess, 'status-success', 'status-error')}">
                        <apex:outputText value="{!statusMessage}" />
                    </div>
                </apex:outputPanel>

                <!-- Recipient Email -->
                <div class="form-group">
                    <label>Recipient Email Address <span class="required">*</span></label>
                    <apex:inputText
                        value="{!recipientEmail}"
                        styleClass="email-input"
                        style="width:100%; padding:10px 14px; border:1px solid #ddd; border-radius:4px; font-size:14px; box-sizing:border-box;"
                        html-placeholder="e.g. recipient@example.com"
                        html-type="email"
                    />
                </div>

                <!-- Email Subject -->
                <div class="form-group">
                    <label>Subject <span class="required">*</span></label>
                    <apex:inputText
                        value="{!emailSubject}"
                        style="width:100%; padding:10px 14px; border:1px solid #ddd; border-radius:4px; font-size:14px; box-sizing:border-box;"
                        html-placeholder="Enter email subject"
                    />
                </div>

                <!-- Email Body -->
                <div class="form-group">
                    <label>Message Body <span class="required">*</span></label>
                    <apex:inputTextarea
                        value="{!emailBody}"
                        style="width:100%; padding:10px 14px; border:1px solid #ddd; border-radius:4px; font-size:14px; box-sizing:border-box; height:130px; resize:vertical;"
                        html-placeholder="Type your message here..."
                    />
                </div>

                <hr class="divider" />

                <!-- Attachment Section -->
                <div class="attachment-section">
                    <div class="checkbox-group">
                        <apex:inputCheckbox value="{!hasAttachment}" id="attachCheck" />
                        <label for="{!$Component.attachCheck}">📎 Add Attachment</label>
                    </div>

                    <apex:outputPanel rendered="{!hasAttachment}">
                        <div class="form-group" style="margin-bottom:0;">
                            <label>Select File</label>
                            <apex:inputFile
                                value="{!fileBody}"
                                filename="{!fileName}"
                                style="font-size:13px; margin-top:5px;"
                            />
                            <p style="font-size:12px; color:#888; margin:5px 0 0;">
                                Supported: PDF, DOC, DOCX, XLS, PNG, JPG, TXT (Max 5MB)
                            </p>
                        </div>
                    </apex:outputPanel>
                </div>

                <hr class="divider" />

                <!-- Send Button -->
                <apex:commandButton
                    value="Send Email"
                    action="{!sendEmail}"
                    styleClass="btn-send"
                    style="background:linear-gradient(135deg,#1589EE,#0070D2); color:white; border:none; padding:12px 30px; border-radius:4px; font-size:15px; font-weight:600; cursor:pointer; width:100%;"
                    rerender="statusPanel"
                />

                <!-- Clear Button -->
                <apex:commandButton
                    value="Clear Form"
                    action="{!clearForm}"
                    styleClass="btn-clear"
                    style="background:#f4f4f4; color:#555; border:1px solid #ddd; padding:10px 20px; border-radius:4px; font-size:14px; cursor:pointer; width:100%; margin-top:10px;"
                    immediate="true"
                />

            </apex:form>

        </div>
    </div>

</apex:page>
```

---

## STEP 3 — Enable Deliverability Settings

Before emails can be sent, configure Salesforce deliverability:

1. Go to **Setup → Search "Deliverability"**
2. Click **"Deliverability"**
3. Set **Access level** to:
   - **"All Email"** (for production)
   - **"System Email Only"** is default in sandboxes
4. Click **Save**

---

## STEP 4 — Access the Visualforce Page

### Method A — Direct URL:
```
https://YOUR-ORG.salesforce.com/apex/EmailSenderPage
```

### Method B — Add to App:
1. Go to **Setup → Apps → App Manager**
2. Select your app → **Edit**
3. Add **Visualforce Tab** pointing to `EmailSenderPage`

### Method C — Create a Tab:
1. **Setup → Tabs → New (Visualforce Tabs)**
2. Select `EmailSenderPage` → name it "Email Sender"
3. Add to your app

---

## STEP 5 — Test the Application

### Test Case 1 — Valid Email (No Attachment):
- Email: `test@example.com`
- Subject: `Test Email`
- Body: `Hello, this is a test email from Salesforce!`
- Expected: ✅ Green success message

### Test Case 2 — Invalid Email:
- Email: `invalidemail@`
- Expected: ❌ Red error — "Invalid email address format"

### Test Case 3 — Empty Email:
- Email: *(blank)*
- Expected: ❌ Red error — "email address cannot be empty"

### Test Case 4 — Valid Email (With Attachment):
- Check "Add Attachment" checkbox
- Upload a PDF or image file
- Expected: ✅ Email received with attachment

### Test Case 5 — Empty Subject:
- Email: `test@example.com`
- Subject: *(blank)*
- Expected: ❌ Red error — "subject cannot be empty"

---

## Validation Rules Summary

| Scenario | Message Shown |
|----------|--------------|
| Empty email field | ❌ Recipient email address cannot be empty |
| Invalid email format | ❌ Invalid email address format. Please enter a valid email (e.g., user@example.com) |
| Empty subject | ❌ Email subject cannot be empty |
| Empty body | ❌ Email body cannot be empty |
| Send success | ✅ Email has been sent to [email] |
| Salesforce API error | ❌ Failed to send email: [error details] |

---

## Key Salesforce Classes Used

```apex
// 1. Create email object
Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();

// 2. Set recipients
email.setToAddresses(new String[]{ 'recipient@example.com' });

// 3. Set content
email.setSubject('Subject here');
email.setPlainTextBody('Plain text body');
email.setHtmlBody('<p>HTML body</p>');

// 4. Add attachment (optional)
Messaging.EmailFileAttachment att = new Messaging.EmailFileAttachment();
att.setFileName('file.pdf');
att.setBody(Blob.valueOf('file content'));
email.setFileAttachments(new Messaging.EmailFileAttachment[]{ att });

// 5. Send
Messaging.SendEmailResult[] results = Messaging.sendEmail(
    new Messaging.SingleEmailMessage[]{ email }
);

// 6. Check result
Boolean success = results[0].isSuccess();
```

---

## Project Folder Structure

```
EmailNotification/
├── classes/
│   ├── EmailSenderController.cls
│   └── EmailSenderController.cls-meta.xml
└── pages/
    ├── EmailSenderPage.page
    └── EmailSenderPage.page-meta.xml
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Email not received | Check **Deliverability** setting → set to "All Email" |
| "SendEmail failed" error | Verify recipient email exists and is reachable |
| Attachment not uploading | Use `enctype="multipart/form-data"` in `<apex:form>` |
| Page not found | Check Visualforce page name matches URL exactly |
| "Insufficient privileges" | Check user profile has "Send Email" permission |
| Emails going to spam | Configure SPF/DKIM records in your org's email settings |
| Daily email limit hit | Free orgs: 15 emails/day. Check **Setup → Email Log Files** |

---

## Email Limits in Salesforce

| Edition | Daily Email Limit |
|---------|-----------------|
| Developer Edition | 15 emails/day |
| Professional | 1,000/day |
| Enterprise | 5,000/day |
| Unlimited | 5,000/day |

> Check current usage: **Setup → Email Log Files**

---

## Quick Reference — Developer Console Shortcuts

```
Open Developer Console  : Setup → Developer Console
New Apex Class          : File → New → Apex Class
New Visualforce Page    : File → New → Visualforce Page
Run Anonymous Apex      : Debug → Open Execute Anonymous Window
View Logs              : Debug → Open Execute Anonymous Window → View logs
```
