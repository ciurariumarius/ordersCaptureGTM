# Send completed orders from GTM to Google Sheets

Your site already has a GA4 `purchase` event. Use that event and add **two GTM Custom HTML tags**: one to retain click IDs from the landing page, and one to send the completed order to the sheet. Do not add another purchase push.

## Set up the sheet and Apps Script

1. In the `REVERSEECZ` sheet, put these exact headers in row 1: `Timestamp`, `transaction_id`, `value`, `currency`, `tax`, `shipping`, `coupon`, `items`, `gclid`, `fbc`, `email`, `phone_number`. Their order does not matter, and extra columns are fine. Format `transaction_id`, `gclid`, `fbc`, `email`, and `phone_number` as plain text; the `items` cell will contain JSON text.
2. If [gsheetCode](gsheetCode) is already deployed, adding the new headers is enough; its header-based mapping needs no code change. Otherwise, paste it into Apps Script and deploy a **new version** of the Web app. Set **Execute as: Me** and **Who has access: Anyone**. Copy its `/exec` URL.
3. In [gtm-purchase-to-sheet.html](gtm-purchase-to-sheet.html), replace `PASTE_APPS_SCRIPT_WEB_APP_EXEC_URL_HERE` with that `/exec` URL.

## Set up GTM

1. Create a **Custom HTML** tag with [gtm-capture-click-ids.html](gtm-capture-click-ids.html). Trigger: **Initialization - All Pages**, including ad landing pages. It keeps `gclid` from the URL and `fbc` from `fbclid` or the Meta `_fbc` cookie in first-party browser storage for up to 90 days. Change that retention if your policy requires it.
2. Create another **Custom HTML** tag with [gtm-purchase-to-sheet.html](gtm-purchase-to-sheet.html). Trigger: **Custom Event** with event name `purchase` (exact case). Fire after the purchase is confirmed.
3. Configure both tags consistently with your site's advertising consent setup before publishing. If your consent system blocks these tags until consent is granted, make sure the capture tag can run on the landing page after that grant.
4. In GTM Preview, land on a test URL containing `?gclid=TEST-GCLID&fbclid=TEST-FBCLID`, navigate to checkout on the same origin, and complete a test order. Inspect the sheet: `gclid` should be `TEST-GCLID`, and `fbc` should begin `fb.1.` and end in `TEST-FBCLID`. The purchase event must contain `ecommerce.transaction_id`, `ecommerce.value`, `ecommerce.currency`, `ecommerce.items`, `user_data.email`, and `user_data.phone`. Check the sheet's email and phone cells too.
5. Repeat the same transaction ID. The Apps Script should leave the sheet at one row for that order.

The fields sent by the tag must match the sheet headers exactly. The tag reads `ecommerce` on the same `dataLayer.push({event: 'purchase', ecommerce: ...})` entry, or the parameters of `gtag('event', 'purchase', ...)`. If your purchase data is stored elsewhere, adjust the tag's `order` mapping to match it. Browser storage is scoped to the site's origin; a checkout on another domain needs explicit cross-domain passing. The web app URL is public, so do not treat this browser request as verified order data or send customer personal data through it. For an authoritative order ledger, write from your store backend instead.

Email and phone are read only from `user_data` on the matching purchase event and sent in the POST body. `user_data.phone` is written to the `phone_number` sheet column without changing its format, so a leading `0` is preserved. `user_data.phone_number` remains a fallback. These values are not saved in browser storage or added to the URL. Restrict access to the sheet, and do not map these raw values to ordinary GA4 event parameters. Google has separate user-provided data features for those fields.

`gclid` and `fbc` are observed click identifiers, not an attribution decision. Both can appear on one order if the browser had clicks from both platforms during the retention period. An empty cell means no qualifying identifier was available to these tags.

The browser cannot read the Apps Script response from this cross-origin beacon. GTM Preview can show that the tag fired, but **the new sheet row is the delivery check**.
