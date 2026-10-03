# User manual: admin interface

This guide is for restaurant **administrators** and **operators**. Start the application as described in the [README](../README.md) and open `http://localhost:5173`. The interface works on desktop and on a phone-sized screen (on a small screen the menu opens from the button at the top right).

> **Screenshots:** none are included yet. Add them under `docs/screenshots/` and link them next to the relevant sections (suggested: login, dashboard, foods list, food form, orders list, order detail with timeline, cancel dialog, staff list, reset password).

## Roles at a glance

| Task | Administrator | Operator |
| --- | :---: | :---: |
| View the dashboard order counts | Yes | Yes |
| View foods, add a food, edit a food, mark available / unavailable | Yes | Yes |
| Archive and restore foods | Yes | No |
| View orders, change status, edit notes and fulfillment details | Yes | Yes |
| Manage staff (add, edit, reset password, deactivate, reactivate) | Yes | No (the Staff menu is hidden and the page shows "Not allowed") |

## Signing in and out

1. Open the application. You are sent to **Sign in to staff portal**.
2. Enter your email and password and choose **Sign In**.
3. An empty field or an invalid email shows a message under the field. Wrong credentials show a red message above the form. After five failed attempts in 15 minutes you must wait before trying again.
4. Choose **Sign Out** (bottom of the menu) when you finish. This also ends your session on the server.

Sessions last about two days. When a session ends (or an administrator deactivates your account or resets your password) the next action signs you out automatically and shows "Your session has expired. Please sign in again."

Local test accounts are listed in the README.

## Dashboard

The dashboard shows how many orders are in each state (Total, Placed, Accepted, Preparing, Ready, Completed, Cancelled). Choose a card to open the **Orders** list already filtered to that state.

## Foods (menu catalog)

Foods appear as cards with name, price, category, description, picture (if an image address was entered) and an availability button. The list is split into pages of 12; use **Previous** and **Next** under the list.

- **Search and filter:** type in **Search foods...** (matches name and description), or filter by availability or category. Filtering happens on the server, so it works across all pages. If nothing matches you see "No foods found."
- **Show archived foods:** tick the box to include archived foods.
- **Add a food:** choose **Add food**, fill in the form and choose **Save food**. Name, category and price are required. The price must be greater than zero with at most two decimals (for example `9.50`); otherwise a message appears under the field. The image address is optional and must start with `http://` or `https://`.
- **Edit a food:** choose the pencil on a card, change the details and choose **Save food**. Changing a price does not change orders that were already placed.
- **Mark available / unavailable:** choose the green **Available** or red **Unavailable** button on a card. Unavailable foods are hidden from customers and cannot be ordered.
- **Archive a food (administrators):** choose the archive icon and confirm. An archived food disappears from the menu and cannot be ordered, but past orders keep it in their history. Foods are never deleted. To bring one back, tick **Show archived foods** and choose the restore icon; it stays **Unavailable** until you switch it on.

## Orders

The list shows the newest orders first and refreshes quietly every 30 seconds (or choose **Refresh**). Each row shows the order reference (for example `FO-000012`), customer, pickup or delivery, total, status and when it was placed. Lists are paged (15 per page).

- **Search and filter:** the search box matches the order reference, customer name or customer email; the filters limit the list to one status and/or to pickup or delivery.
- **Open an order:** choose its reference to open the order page.

### The order page

- **Update status:** the buttons offered are only the allowed next steps:

  | Current | Buttons shown |
  | --- | --- |
  | Placed | Accept order, Cancel order |
  | Accepted | Start preparing, Cancel order |
  | Preparing | Mark ready, Cancel order |
  | Ready | Mark completed |
  | Completed, Cancelled | none (the page says no further changes are possible) |

  **Cancel order** opens a dialog that requires a reason; the reason is saved and shown in the timeline. If someone else changed the order while your page was open, the change is refused with a message and the page refreshes to the current state. The server enforces the same rules regardless of what the page shows.
- **Items:** each line shows the name and price **as they were when the order was placed**, quantity and line total, followed by the subtotal and total. Items and prices cannot be edited.
- **Customer:** name, email and phone.
- **Fulfillment:** pickup or delivery with the address. Choose **Edit** to switch type or change the address (a delivery needs an address). Not available once the order is Ready, Completed or Cancelled.
- **Internal notes:** choose **Edit** to add or change notes. They are for staff only and can be edited at any time.
- **Status timeline:** every change in order, with the previous and new status, who made it (staff name or the customer), when, and the cancellation reason if any. The timeline is a permanent record and cannot be edited.

## Staff management (administrators)

- **View staff:** a table with name, email, role, active/inactive and creation time. Search by name or email and filter by role or status.
- **Add a staff member:** choose **Add staff member**, enter name, email, an initial password (at least 8 characters) and role, then **Create staff member**. An email that is already used is rejected.
- **Edit:** choose the pencil to change name, email or role. The **last active administrator cannot be changed to operator**.
- **Reset password:** choose the key icon, enter a new password (at least 8 characters) twice and share it with the person securely. All of their sessions end. If you reset **your own** password you are signed out too.
- **Deactivate or reactivate:** choose the person-with-cross icon to deactivate (you are asked to confirm). They are signed out immediately and cannot sign in until reactivated; choose the person-with-check icon to reactivate. You cannot deactivate yourself, and the **last active administrator cannot be deactivated**.

## Keyboard and screen readers

Every control can be reached with the keyboard, focus is always visible, dialogs trap focus and close with **Esc**, form fields have labels and error messages are linked to their fields, and a "Skip to main content" link appears on the first Tab press. This has not been tested with a screen reader.

## Common messages

| Message | What it means and what to do |
| --- | --- |
| "Invalid email or password" | Wrong email or password, or the account is deactivated. |
| "Too many failed login attempts. Try again later." | Wait 15 minutes, then try again. |
| "Not allowed" page | Your role cannot open that page. Ask an administrator. |
| "Cannot transition from X to Y" / "The order is now X, not Y" | The status changed or the step is not allowed. The page refreshes; use one of the offered buttons. |
| "A reason is required to cancel an order" | Type a reason in the cancel dialog. |
| "This food is archived. Restore it first." | Restore the food before editing it. |
| "The last active administrator cannot be demoted / deactivated" | Create or activate another administrator first. |
| "Cannot reach the server" | The API is not running or the network is down. Check and try again. |
| "Your session has expired. Please sign in again." | Sign in again. |
