## Odoo Delivery Email Notification Implementation Journey

**Project Goal:** Implement an automated email notification to customers when a delivery order associated with the "OUT" stage of a PICK/PACK/OUT/SIGN multi-step delivery process is validated. The email should use a standard Odoo delivery template and be logged in the delivery order's Chatter.

**Odoo Environment:** Odoo.sh, Enterprise version 18.

---

### **Phase 1: Initial Solution Proposal & Clarification**

*   **Initial Request:** User wants email when "OUT" stage is validated.
*   **Consultant's Initial Assumption:** "OUT" stage corresponds to a single `stock.picking` becoming `Done`.
*   **Proposed Solution:** Automated Action (Server Action) on `stock.picking` model, triggered by state change to `Done`, with a filter for the specific operation type.
*   **Challenge Encountered:** Initial understanding of the "OUT" stage was incomplete.
    *   **User Clarification:** The process involves `PICK`/`PACK`/`OUT`/`SIGN` as **separate `stock.picking` records** in a multi-step routing. "OUT" is a specific `stock.picking` (Operation Type "Deliver" with "OUT" prefix) that transitions to `Done`, but it's followed by a `SIGN` stage.
*   **Successes:**
    *   Confirmed "OUT" stage aligns with a specific `stock.picking` record reaching the `Done` state.
    *   Confirmed the use of the `stock.picking` model for the Automated Action.
    *   Identified the correct email template: "Shipping: Send by Email" (XML ID: `stock.mail_template_data_delivery_confirmation`).

---

### **Phase 2: First Automated Action Attempt - `State is set to Done` Trigger with Python `old_values`**

*   **Attempted Configuration:**
    *   **Model:** `Transfer (stock.picking)`
    *   **Trigger:** `State is set to Done`
    *   **Apply On:**
        *   `Operation Type > Name > is equal to > Deliver`
        *   `State > is equal to > Done` (Initially redundant, but harmless)
    *   **Actions To Do:**
        *   `Execute Code`: `record.state == 'done' and record.state != old_values.get('state')` (for duplicate prevention)
        *   `Send Email`: `Shipping: Send by Email` (with `Send Email As: Email`)
*   **Challenge Encountered:** `NameError: name 'old_values' is not defined`
    *   **Root Cause:** The `old_values` variable is **not available** when the Automated Action's trigger is `State is set to Done`. This trigger inherently handles the transition, so `old_values` is not passed to the Python context.
*   **Successes:**
    *   Precisely diagnosed the Python `NameError`.
    *   Learned about the specific context available with different Automated Action triggers in Odoo 18.

---

### **Phase 3: Trigger Refinement - `On save` with Explicit State Filters**

*   **Attempted Configuration Adjustment:**
    *   **Removed** the problematic `Execute Code` action.
    *   **Consultant's Misstep:** Initially suggested `On Update` / `Values Updated` trigger.
    *   **User Clarification:** Confirmed these options were **not available** in the Odoo instance. The closest equivalent was **`On save`**.
*   **Solution Adopted:** Using `On save` trigger with a more robust set of "Apply On" filters to detect state transition.
*   **Configuration:**
    *   **Model:** `Transfer (stock.picking)`
    *   **Trigger:** `On save`
    *   **When updating:** Added `Operation Type` and `Status` (newly discovered field, helps optimize trigger).
    *   **Apply On:**
        *   `Operation Type > Name > is equal to > WH: Deliver`
        *   `State > is equal to > Done`
        *   `State > is not equal to > Done` (This pair, with `On save`, detects the exact transition to `Done`).
    *   **Actions To Do:**
        *   `Send Email`: `Shipping: Send by Email` (configured with `Send Email As: Message` – clarified this meant "send email AND log in Chatter" for Odoo 18).
*   **Challenges Encountered:**
    *   **Initial Problem:** The `State is set to Done` trigger caused premature Chatter entries and delays in the multi-step routing. It was firing when `WH/OUT` went `Ready`, not just `Done`.
    *   **Confusion:** The `State is equal to Done` AND `State is not equal to Done` filters seemed contradictory at first glance.
*   **Successes:**
    *   **Critical Breakthrough:** By **archiving the problematic Automated Action**, we established a "clean slate," confirming the previous rule was indeed causing the delays and anomalous Chatter entries.
    *   **Clarified Filter Logic:** Understood how `On save` combined with `State = Done` AND `State != Done` precisely detects a state *transition*.
    *   **Validated Trigger:** Successfully confirmed that the Automated Action, with `On save` + the three "Apply On" filters, **fired at precisely the correct moment** (when `WH/OUT` transitions to `Done`), with **no anomalous Chatter postings** for other operations/states.

---

### **Phase 4: Email Dispatch Failure (Template, Related Document & Python)**

*   **Initial Observation:** Chatter updated correctly after `WH/OUT` validation, but **no email was received**.
*   **Challenge Encountered (Initial Suspect):** The `mail.mail` record (email in queue) was incorrectly referencing `WH/SIGN/00381` instead of `WH/OUT/XXXXX`. This suggested the "Related Document" for the email was misassigned.
*   **Attempted Solution (Python `Execute Code`):** To explicitly control the `res_id` (Related Document) of the email, we replaced the UI "Send Email" action with an `Execute Code` action using Python:
    ```python
    mail_template = env.ref('stock.mail_template_data_delivery_done') # (Initial error by consultant)
    mail_template.send_mail(record.id, force_send=True, email_values={'email_to': record.partner_id.id, ...}, notif_layout='mail.mail_notification_light')
    ```
*   **Challenges with Python Code:**
    *   **First Python Error:** `ValueError: External ID not found in the system: stock.mail_template_data_delivery_done`.
        *   **Root Cause:** The Python code used the incorrect XML ID for the email template. (Consultant's error).
        *   **User Clarification:** Provided the correct XML ID: `stock.mail_template_data_delivery_confirmation`.
    *   **Second Python Error:** `TypeError: MailTemplate.send_mail() got an unexpected keyword argument 'notif_layout'`.
        *   **Root Cause:** The `send_mail` method in Odoo 18 does not accept the `notif_layout` parameter. (Specific Odoo version incompatibility in Python method signature).
*   **Successes:**
    *   Identified the correct XML ID for the delivery confirmation template.
    *   Learned that direct Python code can be susceptible to minor version differences in method signatures.
    *   The Chatter was correctly updated by the `Execute Code` (when it wasn't failing).

---

### **Phase 5: Reverting to UI-based "Send Email" Action (Final Configuration)**

*   **Solution Adopted:** Removed the problematic `Execute Code` action and reverted to using the standard UI "Send Email" action, which is more stable across Odoo versions for core functionality.
*   **Configuration:**
    *   **Model:** `Transfer (stock.picking)`
    *   **Trigger:** `On save`
    *   **When updating:** `Operation Type`, `Status`
    *   **Apply On:**
        *   `Operation Type > Name > is equal to > WH: Deliver`
        *   `State > is equal to > Done`
        *   `State > is not equal to > Done`
    *   **Actions To Do:**
        *   `Send Email`: `Shipping: Send by Email` (with `Send Email As: Message`)
*   **Successes:**
    *   **Email Generation & Chatter Logging Confirmed:** The Automated Action now reliably triggers, correctly generates the email, and posts its full content to the `WH/OUT` picking's Chatter. No errors during this process. This means the `res_id` (Related Document) is now correctly handled by the UI action.

---

### **Current Status & Remaining Challenge (Step 8):**

*   **Challenge:** The emails are successfully created and logged in Odoo's Chatter, but they are **not being delivered** to the customer's inbox. They are likely stuck in the mail queue or failing during external dispatch.
*   **Next Steps:** Diagnose the actual mail delivery issue, focusing on the Outgoing Mail Server configuration, the mail queue status (and any errors there), and testing with a reliable external recipient.

---

This comprehensive summary should serve as a valuable reference for the future! It highlights the complexities of Odoo configuration, especially with multi-step processes and subtle version differences.
