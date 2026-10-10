# Social Messenger Native Discuss Integration (v2 Architecture)

## Context
The current version of the `social_messenger` module uses custom `social.conversation` and `social.message` models. While functional, it forces users to click a "Reply" button which opens a popup wizard to send messages back to Facebook/Instagram. 

To create a true, real-time chat experience, the architecture is being redesigned to mimic the official Odoo Enterprise `whatsapp` module. We will integrate directly with Odoo's native `discuss.channel`.

## 1. Data Model & Discuss Extension
The custom conversation models will be removed and replaced by extending core Odoo models:

- **`discuss.channel` (Extended)**:
  - `channel_type`: Extended to include `'facebook'` and `'instagram'`.
  - `social_account_id`: (Many2one -> `social.account`) Links the channel to the specific Meta App / Page credentials required to send replies.
  - `social_sender_id`: (Char) Stores the platform-scoped ID (PSID/IGSID) of the external customer messaging the page.

- **`mail.message` (Extended)**:
  - `social_meta_message_id`: (Char) Stores the unique message ID from Meta to prevent processing duplicate webhook deliveries.

- **`discuss.channel.member`**:
  - We will manage channel memberships dynamically. When a new chat comes in, the associated Odoo internal users (like the page managers) and the customer (if linked to a `res.partner`) will be added to the channel so it appears in their Discuss sidebar.

## 2. Incoming Messages (Webhooks)
When a webhook payload hits the `controllers/main.py` endpoint from Meta:
1. Extract the `sender_id` (customer) and `recipient_id` (page).
2. Search for an existing `discuss.channel` where `social_sender_id = sender_id` and `social_account_id` matches the receiving page.
3. If no channel exists, create one, give it a name based on the sender's Meta profile, and add the appropriate Odoo users as channel members.
4. Call `channel.message_post()` with the incoming message text and attachments. Because it's posted to a `discuss.channel`, it instantly pops up in the Odoo Discuss UI for the users.

## 3. Outgoing Messages (Real-Time Reply)
To allow real-time replies from the Discuss text box, we will override the `message_post` method in `discuss.channel`:
1. When a user types a message and hits send, `message_post` fires.
2. If the `channel_type` is `'facebook'` or `'instagram'`, our overridden method intercepts the call.
3. We strip HTML formatting from the Odoo message body to get plain text.
4. We asynchronously (or synchronously, depending on failure handling) call `self.social_account_id._send_message(self.social_sender_id, plain_text)`.
5. The message is successfully sent to Meta, and Odoo logs the sent message in the chat history.

## Migration Strategy
When upgrading to this new architecture, a data migration script will be needed to convert all existing `social.conversation` records into `discuss.channel` records, and move `social.message` records over to `mail.message`.
