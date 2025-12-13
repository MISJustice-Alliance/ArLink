---
description: "Check delivery status for mail job"
allowed-tools: ["Read", "Bash(npm:*)"]
author: "ArweaveStamp Team"
version: "1.0"
---

# Track Delivery - Delivery Status

## Purpose
Check delivery status for a PostGrid mail job with carrier tracking.

## Parameters
- $1: Mail job ID

## Workflow

1. **Query job status**
   ```bash
   !npm run cli -- mail status $JOB_ID
   ```

2. **Display job details**
   - Sender address
   - Recipient address
   - Documents mailed
   - QR code data

3. **Display tracking information**
   - Carrier: USPS/UPS/FedEx
   - Tracking number(s)
   - Current status
   - Last update timestamp
   - Estimated delivery date

4. **Show tracking events**
   Display timeline:
   - [Date/Time] Printed
   - [Date/Time] Picked up by carrier
   - [Date/Time] In transit
   - [Date/Time] Out for delivery
   - [Date/Time] Delivered (if complete)

5. **Delivery receipt** (if delivered)
   ```bash
   !npm run cli -- mail delivery-receipt $JOB_ID --format pdf
   ```

6. **Watch mode** (optional)
   If --watch flag:
   - Poll every 60 seconds
   - Display updates in real-time
   - Exit on delivery confirmation

## Success Criteria
- Job status retrieved
- Tracking events displayed
- Delivery receipt available (if delivered)

## Version History
- 1.0: Initial release
