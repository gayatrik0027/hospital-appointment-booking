# Hospital Appointment Booking – Camunda 8 (Exercise 8)

Built for Camunda 8.6+ (SaaS or Self-Managed). Open the files in Camunda Modeler 5.30+ (Camunda 8 mode) or Web Modeler.

## Contents

| File | What it is |
|---|---|
| `bpmn/send-notification.bpmn` | Child process: Start → REST Outbound Connector (Telegram) → End |
| `dmn/slot_decision.dmn` | Decision `slot_decision`, hit policy UNIQUE |
| `bpmn/appointment-booking.bpmn` | Parent process, **propagation OFF** + explicit mappings (recommended) |
| `bpmn/appointment-booking-propagate-all.bpmn` | Same parent, **propagation ON** (for the comparison, Task 4) |
| `test-data/` | Variables for the 3 test cases and the cancellation message |
| `screenshots/` | Put your Operate screenshots here |

## Setup before running

1. Create the cluster secret `TELEGRAM_BOT_TOKEN` (Console → Cluster → Connector Secrets). The child uses it as `{{secrets.TELEGRAM_BOT_TOKEN}}`.
2. Create a bot with @BotFather and send it any message from your own Telegram account. `patientId` must be your numeric Telegram chat id (get it from `https://api.telegram.org/bot<TOKEN>/getUpdates`), because the parent passes `patientId` as `recipientId`. The test files use dummy ids, so edit them first.
3. Make sure the Connectors runtime is running (SaaS has it built in).

## Deploy (order matters)

Deploy in one go or in this order: `slot_decision.dmn`, `send-notification.bpmn`, `appointment-booking.bpmn`.
Only deploy `appointment-booking-propagate-all.bpmn` when you do the Task 4 comparison.

## Design notes

- **Child (`send-notification`)**: connector `io.camunda:http-json:1`, POST to `https://api.telegram.org/bot{{secrets.TELEGRAM_BOT_TOKEN}}/sendMessage`, body `={chat_id: recipientId, text: messageText}`. Result expression returns `notificationStatus` and `telegramMessageId`.
- **DMN**: inputs `department` (String), `bookedCount` (Number); outputs `slotStatus` (AVAILABLE/FULL), `suggestedAction`. Capacity rules: Cardiology 10, Orthopedics 8, General Medicine 15. Cardiology with `bookedCount >= 10` is FULL. UNIQUE means exactly one rule may match, so departments not in the table raise a "no rule matched" incident.
- **Parent**: "Check slot" writes the result to `slotDecision`. The gateway tests `slotDecision.slotStatus = "AVAILABLE"`; FULL is the default flow.
- **"Book slot" / "Release slot"** are FEEL script tasks (`slotBooked = true`, `slotReleased = true`), so no job worker is needed. Replace them with service tasks and real workers for production.
- **"Appointment pending (patient attends)"** is a user task I added after the confirmation. Without a waiting state the instance would finish immediately and there would be nothing left to cancel. Do not complete it before testing the cancellation.
- **Cancellation** is an *interrupting* message event subprocess: message `cancel-booking`, correlation key `=patientId`. Interrupting is the right choice here because the booking should stop as soon as the patient cancels. Switch it to non-interrupting in the Modeler if you want to compare.

## Task 4 – Variable propagation: comparison

Both variants have the same input mappings on every call activity:

- `recipientId` = `=patientId`
- `messageText` = a FEEL string built from `patientName`, `department`, `preferredSlot` (and `slotDecision.suggestedAction` for slot full)

| | Propagation ON (`...-propagate-all.bpmn`) | Propagation OFF (`appointment-booking.bpmn`) |
|---|---|---|
| Parent → child | Child receives **all** parent variables (`patientId`, `patientName`, `department`, `bookedCount`, `slotDecision`, ...) plus `recipientId` and `messageText` | Child receives **only** `recipientId` and `messageText` |
| Child → parent | **All** child variables (`recipientId`, `messageText`, `notificationStatus`, `telegramMessageId`) are copied back to the parent scope | Only what the output mapping lists: `telegramMessageId` → `confirmationMessageId` / `slotFullMessageId` / `cancellationMessageId` |
| Side effects | Parent variables `recipientId` and `messageText` appear and are overwritten by each call; `telegramMessageId` from the confirmation is overwritten by the cancellation message id | No pollution; each call keeps its own result variable |
| Coupling | Child depends implicitly on parent variable names; leaking patient data into a reusable child is a privacy concern | Child is a clean, reusable contract (`recipientId`, `messageText`) |
| Effort | Least configuration | Explicit mappings needed |

**Conclusion**: propagation OFF is better for a reusable, hospital-grade process: minimal data exposure, no accidental overwrites, easier to test. Propagation ON is only convenient for quick prototypes.

What to check in Operate: open the child instance → Variables tab (ON shows ~9 variables, OFF shows 2 + results); open the parent → Variables tab (ON shows `recipientId`/`messageText`/`telegramMessageId`, OFF shows `confirmationMessageId` etc.).

## Task 5 – Test cases

Start each with "Start instance" in Operate/Modeler using the JSON, or with the CLI (e.g. `zbctl create instance appointment-booking --variables "$(cat test-data/case1-cardiology-count3.json)"`).

| # | Input file | Expected |
|---|---|---|
| 1 | `case1-cardiology-count3.json` | `slotStatus = AVAILABLE` → Book slot → call activity "Send confirmation" completes → instance waits at "Appointment pending" |
| 2 | `case2-cardiology-count12.json` | `slotStatus = FULL` → "Send slot full message" → ends at "Slot full" |
| 3 | `case3-booking-then-cancel.json`, then publish `case3-cancel-message.json` | Instance waits at "Appointment pending"; after the message, the event subprocess interrupts it → "Release slot" → "Send cancellation message" → "Booking cancelled" |

Publish the message with e.g. `zbctl publish message cancel-booking --correlationKey 987654321 --variables '{"cancelReason":"Patient request"}'` (the correlation key must equal the instance's `patientId`).

## Task 6 – Screenshots to capture in Operate (save in `screenshots/`)

1. Parent instance (case 1) showing the token on "Appointment pending" and the variables panel.
2. Call activity "Send confirmation" in the parent → click **View called instance** (the child `send-notification` instance, shown with its parent instance link in the header).
3. Parent instance from case 3 after cancellation: path through "Handle cancellation", the second child instance, and the incident-free completed state.
4. Case 2 (slot full) instance, completed at "Slot full".
5. Decision instance of `slot_decision` (Operate → Decisions) showing the matched rule.
6. Optional for Task 4: child variables with propagation ON vs OFF.

## Troubleshooting

- Incident at "Send Telegram message": wrong/missing secret, or the chat id is not a real Telegram chat (HTTP 400/401).
- Incident at "Check slot": department not in the decision table (UNIQUE finds no match).
- Message not correlating: the correlation key must be exactly the instance's `patientId`, and the instance must still be waiting.
