# Email HTML content (Orchestrate journeys)

**Authoritative:** `htmlContent` JSON Schema and tool description on `updateOrchestrateEmailContent`.
**This file:** when to run, workflow ordering, and copy craft — not validation rules (those live in MCP).

---

## When to run

Run Step 5 in `SKILL.md` when the user wants to **write, draft, add, or update email content** — e.g.
"write the welcome email", "add copy to each step", "fill in the body", paste content, personalization,
or revise a draft.

Also when email copy was captured in intake, once the journey exists and steps are verified.

---

## Workflow

1. Journey already created — `getOrchestrateJourneySteps`.
2. Each `messageType` `Email` step: `messageId` → `emailId` on `updateOrchestrateEmailContent`.
3. Draft HTML; iterate with the user; call the tool per approved email.
4. `getOrchestrateEmail` does **not** return HTML — confirm via tool response or Orchestrate UI.
5. Remind: **subject**, **from**, **reply-to** are UI-only.

**Multi-email journeys:** one save per email, in journey order; label drafts by step **name**.

**Conditional splits:** each branch email has its own `messageId`.

---

## Generating HTML (craft, not enforcement)

Match Orchestrate **Edit email** intent — table-based layout, inline styles, simple structure. The save
path sanitizes HTML the same way as the UI; unsafe tags and `javascript:` URLs are stripped automatically.

Before drafting, skim the UI's HTML best practices (same as
[Create and send an HTML email](https://support.pendo.io/hc/en-us/articles/52945752520731-Create-and-send-an-HTML-email)):
table layout, no scripts, visible unsubscribe link for marketing.

**Author personalization as single-brace tokens** (same as typing in Edit email): `{visitor.agent.field/}`,
`{field|default/}`. Call `visitorMetadataSchema` / `accountMetadataSchema` before using fields. Do not
author `{{visitor.name}}` — the pipeline converts single-brace on save.

**Marketing unsubscribe** — exact token in an anchor (spacing matters):

```html
<a href="{{ UnsubscribeURI }}">Unsubscribe</a>
```

Transactional emails do not require unsubscribe. Details and limits are on the tool schema.

**Do not include:** polls, guide building blocks, or interactive widgets.

---

## Worked example — minimal marketing email

Starting template for a welcome or onboarding step. Customize copy, CTA URL, and styles for the user;
keep table layout, inline styles, allowed tags, and the unsubscribe anchor for marketing emails.

```html
<!DOCTYPE html>
<html>
<head>
  <title>Welcome</title>
</head>
<body style="margin: 0; padding: 0; background-color: #f4f4f4;">
  <table role="presentation" cellpadding="0" cellspacing="0" width="100%" style="background-color: #f4f4f4;">
    <tr>
      <td align="center" style="padding: 24px 16px;">
        <table role="presentation" cellpadding="0" cellspacing="0" width="600" style="background-color: #ffffff;">
          <tr>
            <td style="padding: 32px 24px; font-family: Arial, Helvetica, sans-serif; font-size: 16px; line-height: 24px; color: #333333;">
              <p style="margin: 0 0 16px;">Hi {visitor.agent.fullName|there/},</p>
              <p style="margin: 0 0 16px;">Thanks for signing up. Here is one quick win to try in your first week.</p>
              <p style="margin: 0 0 24px;">
                <a href="https://app.example.com/getting-started" target="_blank" style="display: inline-block; padding: 12px 24px; background-color: #128297; color: #ffffff; text-decoration: none; border-radius: 4px;">Get started</a>
              </p>
              <p style="margin: 0; font-size: 14px; color: #666666;">
                <a href="{{ UnsubscribeURI }}" style="color: #666666; text-decoration: underline;">Unsubscribe</a>
              </p>
            </td>
          </tr>
        </table>
      </td>
    </tr>
  </table>
</body>
</html>
```

**Save via MCP** (after `getOrchestrateJourneySteps`):

```
updateOrchestrateEmailContent({
  subId, appId,
  emailId: "<messageId from journey step>",
  htmlContent: "<html above>"
})
```

Use the journey step's `emailType` — marketing emails require the visible `{{ UnsubscribeURI }}` link.
On success, confirm in the Orchestrate UI preview (or the tool response); `getOrchestrateEmail` does not
return HTML.

---

## Copy and tone (plugin-only)

- Match the user's brand voice when described; otherwise clear, professional lifecycle tone.
- Lead with value; one primary CTA per email.
- Keep subject lines out of the HTML body.
- On revision, regenerate full HTML for affected emails and re-call the tool — do not ask the user to patch fragments.

On save failure, read the **tool error message** first, then adjust and retry once.
