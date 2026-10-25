# 2C2P Payment Provider – Investigation & Implementation Plan

Status: proposal (no code yet). Target: a new provider project in this solution,
`EPiServer.Business.Commerce.Payment.TwoC2P`, built the same way as the existing DIBS provider.

> **Verification note.** The 2C2P developer portal (`developer.2c2p.com`) was not reachable from the
> environment this plan was written in. The API facts below come from portal search results and from an
> open-source client (`laraditz/2c2p`). Items marked **(verify)** must be checked against the live
> v4.3 docs and your merchant account before coding.

---

## 1. Recommendation

Integrate **2C2P PGW v4.3 using the Redirect API** (2C2P-hosted payment page).

* No card data is handled by the site, so PCI scope stays minimal (SAQ-A). It also enables every 2C2P
  channel (cards, QR, e-wallets, installments, 123/over-the-counter) without extra UI work.
* It fits the redirect pattern this repo already uses for DIBS (`DIBSPaymentGateway` → `DIBSPaymentController` →
  hosted page → return to the site), so most of the cart-completion code can be reused.
* Direct API / Secure Fields (card inputs on our own page) and the Payment Maintenance API
  (settle/void/refund from Commerce Manager) can be added later. See the phases in §7.

---

## 2. 2C2P PGW v4.x – what we need to know

### 2.1 Hosts & transport

| Environment | Base URL |
|---|---|
| Sandbox | `https://sandbox-pgw.2c2p.com/payment/4.3/` |
| Production | `https://pgw.2c2p.com/payment/4.3/` |

The Payment Inquiry docs show a `/payment/4.5/` path while Payment Token shows `4.3` **(verify)**.
The base URL and version therefore belong in configuration, not in code (DIBS hard-codes its endpoints:
`DIBSPaymentController.cs:21`, `DIBSRequestHelper.cs:14`, and the `test` flag at `DIBSRequestHelper.cs:55`.
We should not repeat that.)

### 2.2 Authentication / message format

* Every request is `POST` JSON: `{ "payload": "<JWT>" }`.
* The JWT is **HS256**, signed with the **merchant Secret Key**. There is no `Authorization` header;
  the merchant is identified by `merchantID` inside the payload.
* A successful response is also `{ "payload": "<JWT>" }`, signed the same way: verify it, then decode it.
  On some errors the response is plain JSON `{ "respCode": "...", "respDesc": "..." }` with no `payload`,
  so handle both shapes.
* The reference client signs with the secret as a raw string (UTF-8 bytes). DIBS hex-decodes its key
  (`DIBS/Helpers/Utilities.cs`), which is a different scheme, so don't copy it **(verify)**.

### 2.3 APIs used in the Redirect flow

**Payment Token**: `POST {base}/paymentToken`

| Field | Notes |
|---|---|
| `merchantID` | from config |
| `invoiceNo` | merchant reference, must be **unique per token request** (verify max length/charset; ≤ 50 alphanumeric is expected) |
| `description` | short order description |
| `amount` | **decimal major units** in v4 (e.g. `1000.00`), unlike DIBS's minor units (verify) |
| `currencyCode` | ISO-4217 alpha (`THB`, `SGD`, `USD`…) |
| `paymentChannel` | optional array, e.g. `["CC"]`; omit to show every channel enabled for the merchant |
| `request3DS` | optional (`Y`/`N`/`F`) (verify) |
| `frontendReturnUrl` | browser comes back here |
| `backendReturnUrl` | server-to-server notification |
| `userDefined1..5` | free fields echoed back in inquiry and backend results (verify). We store the cart id here |
| `locale` | optional, UI language |

Response: `respCode` `"0000"` = success, plus `webPaymentUrl` (send the shopper here) and `paymentToken`.

**Frontend return**: the shopper's browser is sent to `frontendReturnUrl` with a **base64url-encoded
JSON** (`paymentResponse`, via POST form or query string (verify)):
`{"invoiceNo":"…","channelCode":"CC","respCode":"2000","respDesc":"Transaction is completed, please do payment inquiry request for full payment information."}`

* This message is **not signed**, so it must never be used to complete an order.
* `2000` only means "the flow finished". The real result has to come from Payment Inquiry or the backend notification.

**Backend notification**: 2C2P `POST`s `{ "payload": "<JWT HS256>" }` to `backendReturnUrl` once the payment
reaches a final state. The payload holds the same fields as the inquiry response (`merchantID`, `invoiceNo`,
`amount`, `currencyCode`, `respCode`, `tranRef`, `accountNo`, …). Verify the signature before trusting it.
This is the only path that still completes the order when the shopper closes the browser after paying.

**Payment Inquiry**: `POST {base}/paymentInquiry` with JWT `{ merchantID, invoiceNo | paymentToken, locale }`.
The response JWT includes `respCode` (`"0000"` = paid), `respDesc`, `amount`, `currencyCode`, `invoiceNo`,
`tranRef`, `referenceNo`, `approvalCode`, `eci`, `transactionDateTime`, `channelCode`, `accountNo` (masked),
and `userDefined1..5`.

### 2.4 Payment Maintenance (settle / void / refund)

This is a separate API and much heavier:

* XML `PaymentProcessRequest` with `processType` = `S` (settle), `V` (void), `R` (refund), `actionAmount`, `invoiceNo`.
* v4: **JWE (RSA-OAEP + A256GCM)**, signed with **JWS PS256** using a merchant RSA private key, with responses
  verified using 2C2P's public certificate. Sandbox endpoint is `https://demo2.2c2p.com/PaymentAction/2.0/action` (verify current URL).
* Business rules: a void must happen the same day, before the acquirer cut-off. A pre-authorisation that isn't settled
  in about 7 days is voided automatically. The sum of refunds cannot exceed the captured amount.
* .NET Framework 4.6.1 has no built-in AES-GCM, so this needs a library (e.g. `jose-jwt` or BouncyCastle).
  This is the main reason maintenance is planned as Phase 2.

---

## 3. How it maps onto the Episerver Commerce payment lifecycle

| Episerver event | 2C2P action | Phase |
|---|---|---|
| Checkout: `ProcessPayment` on the **cart** (`TransactionType.Authorization`) | Save the cart and return a redirect to the 2C2P payment page type (same as `DIBSPaymentGateway.cs:150`) | 1 |
| Payment page controller `Index` (GET) | Call `paymentToken`, then `302` to `webPaymentUrl` (plain redirect, no auto-post form needed) | 1 |
| Browser returns to `frontendReturnUrl` | Decode it, **ignore its status**, call `paymentInquiry`, validate, then complete the cart into a PO, or send the shopper back to checkout with an error | 1 |
| 2C2P calls `backendReturnUrl` | Verify the JWT, load the cart from `userDefined1`, validate, complete idempotently, return `200` | 1 |
| PO `ProcessPayment` with `TransactionType.Capture` (shipment released) | Phase 1: return success when the merchant account auto-settles (config flag). Phase 2: `S` settle | 1 / 2 |
| PO `ProcessPayment` with `TransactionType.Credit` (return / refund) | Phase 1: unsuccessful result with a clear "refund in the 2C2P portal" message. Phase 2: `R` refund | 1 / 2 |
| PO `ProcessPayment` with `TransactionType.Void` | Phase 2: `V` void | 2 |
| Order created in Commerce Manager | Not supported (same as DIBS: "does not support this order type") | — |

---

## 4. Project layout

New SDK-style project `EPiServer.Business.Commerce.Payment.TwoC2P` (`net461`, `EPiServer.Commerce.Core 13.29.0`,
same as `EPiServer.Business.Commerce.Payment.DIBS.csproj`). C# identifiers can't start with a digit, so code uses
`TwoC2P`. The payment method **system keyword** can still be `2C2P`.

```
EPiServer.Business.Commerce.Payment.TwoC2P/
  TwoC2PPaymentGateway.cs          AbstractPaymentGateway + IPaymentPlugin (cart → redirect; capture/credit on PO)
  TwoC2PPaymentMethod.cs           IPaymentMethod, [ServiceConfiguration(typeof(IPaymentMethod))]
  Controllers/
    TwoC2PPaymentController.cs     PageController<TwoC2PPage>: Index (start), Return (frontend)
    TwoC2PNotificationController.cs   anonymous POST endpoint for backendReturnUrl (route registered in an InitializationModule)
  PageTypes/TwoC2PPage.cs          CMS page type (new GUID), GroupName "Payment"
  Helpers/
    TwoC2PConfiguration.cs         reads PaymentMethodParameter rows (pattern: DIBSConfiguration)
    TwoC2PClient.cs                paymentToken / paymentInquiry over HttpClient
    JwtHs256.cs                    minimal HS256 encode/verify/decode
    PaymentResult.cs               typed inquiry/notification model + validation
    TwoC2PAmountFormatter.cs       currency-aware decimal formatting
    Utilities.cs                   Translate(), GetUrlFromStartPageReferenceProperty() (copy from DIBS)
  CommerceManager/ConfigurePayment.ascx(.cs/.designer.cs)   settings UI (pattern: DIBS)
  CommerceManager/EditTab.ascx(.cs/.designer.cs)
  Frontend/_TwoC2P.cshtml, _TwoC2PConfirmation.cshtml       MVC (Quicksilver-based)
  Frontend/PaymentMethod.ascx(.cs/.designer.cs)             WebForms parity (optional, see §8)
  lang/TwoC2P_EN.xml (+ other languages as needed)          keys under /Commerce/Checkout/TwoC2P/
  images/2C2P-logo.png
EPiServer.Business.Commerce.Payment.TwoC2P.Tests/          xunit + Moq + FluentAssertions (pattern: DIBS.Tests)
```

Also add both projects to `PaymentProviders.sln` (the test project goes in the `Tests` solution folder), and add a bullet to `README.md`.

### 4.1 Commerce Manager parameters (`TwoC2PConfiguration`)

| Parameter | Purpose |
|---|---|
| `MerchantID` | 2C2P merchant id |
| `SecretKey` | HS256 key (password textbox) |
| `ApiBaseUrl` | e.g. `https://sandbox-pgw.2c2p.com/payment/4.3/` |
| `PaymentChannels` | optional, comma-separated → `paymentChannel` array |
| `Request3DS` | optional |
| `AutoSettle` | `true` → Capture returns success without calling 2C2P |
| *(Phase 2)* `MaintenanceUrl`, `MerchantPrivateKey` (cert thumbprint or PFX path + password), `TwoC2PPublicCert` | Payment Maintenance |

Start-page reference properties used for URLs, same convention as DIBS: `TwoC2PPaymentPage`,
`TwoC2PPaymentLandingPage`, `CheckoutPage`.

---

## 5. Detailed flow (Phase 1)

1. **Gateway `ProcessPayment(cart)`**: guard clauses as in DIBS (`HttpContext`, payment, order form). Save the cart and return
   `PaymentProcessingResult.CreateSuccessfulResult("", url(TwoC2PPaymentPage))`.
2. **`TwoC2PPaymentController.Index`**: load the current cart and the 2C2P payment.
   * Generate a **new** `invoiceNo` with `IOrderNumberGenerator` on every attempt, because 2C2P rejects reused invoice numbers. Store it
     on the payment (`payment.ProviderTransactionID`, or a payment property) and save the cart.
   * Build the token request: amount = `payment.Amount` formatted for the currency, `userDefined1` = `cart.OrderLink.OrderGroupId`,
     absolute `frontendReturnUrl` (`…/TwoC2PPaymentPage/Return`) and `backendReturnUrl` (`/twoc2p/notify`).
   * If `respCode == "0000"`, `Redirect(webPaymentUrl)`. Otherwise go back to checkout with a localized error and log `respCode`/`respDesc`.
3. **`Return` action** (accept GET and POST): decode `paymentResponse` only to get `invoiceNo`. Call `paymentInquiry(invoiceNo)` and
   **validate**:
   * the JWT signature is valid,
   * `respCode == "0000"`,
   * `invoiceNo` equals the value stored on the cart payment,
   * `amount` and `currencyCode` equal the cart payment,
   * `merchantID` equals config.

   On success, complete the cart through the shared completion service (step 5) and redirect to the landing page.
   On failure or pending, redirect to checkout with `success=false&paymentmethod=2c2p&message=…`.
   For pending channels (e.g. counter or bank transfer), show a "payment pending" message and let the backend notification finish the order.
4. **Notification endpoint**: read `payload`, verify HS256 (reject on failure with 400 and log), and run the same validation as above.
   Load the cart by `userDefined1`, complete it idempotently, and always answer `200` for an already-processed invoice so 2C2P stops retrying.
5. **Shared completion service** (refactored from `DIBSPaymentGateway.ProcessSuccessfulTransaction`/`MakePurchaseOrder`), with these changes:
   * It must work **without a shopper session**. DIBS uses `CustomerContext.Current.CurrentContact` and
     `PrincipalInfo.CurrentPrincipal` (`DIBSPaymentGateway.cs:220,223,326`). Use `cart.CustomerId` and `CustomerContext.Current.GetContactById` instead.
   * **Idempotency / race**: the browser return and the backend notification can arrive at the same moment. Serialize by cart id (a named lock; for
     web farms, also rely on "cart no longer exists → look up the PO by `OrderNumber == invoiceNo` and treat it as done").
   * Set `payment.TransactionID = tranRef`, `payment.ProviderTransactionID = invoiceNo`, and `PurchaseOrder.OrderNumber = invoiceNo`
     (same as DIBS). Add an order note with `channelCode` and `approvalCode`.

---

## 6. Implementation details & pitfalls

* **JWT**: hand-roll HS256 (about 60 lines: base64url, `HMACSHA256`, Newtonsoft is already available). Pin `alg == "HS256"` (reject `none`/others) and
  compare signatures in constant time. DIBS compares its MAC with `string.Equals` (`TransactionRequest.cs:45`); don't copy that. This avoids adding
  `System.IdentityModel.Tokens.Jwt` and the binding-redirect friction it brings on `net461` / Episerver 11.
* **TLS 1.2**: on `net461`, ensure `ServicePointManager.SecurityProtocol |= SecurityProtocolType.Tls12`, otherwise the calls to 2C2P will fail.
* **HTTP**: use one static `HttpClient` with an explicit timeout, rather than DIBS's `HttpWebRequest`. Don't rethrow with `throw ex` (DIBS does this, and it loses the stack trace).
* **Amounts**: format with `CultureInfo.InvariantCulture` and the currency's minor digits (0 for JPY/KRW/VND/IDR) **(verify 2C2P rules per currency)**.
  Compare amounts as decimals, not strings.
* **Never trust the frontend payload.** It is unsigned. Use it only to get the `invoiceNo`.
* **Notification endpoint**: anonymous, no antiforgery token, and fast to respond. Don't log the secret or the full payload (it contains masked PAN, name, and email).
* **Configuration errors**: if `MerchantID`/`SecretKey`/`ApiBaseUrl` are missing, show the localized "settings error" (pattern: `DIBSSettingsError`).
* **Localization**: `/Commerce/Checkout/TwoC2P/*` keys: `PaymentNotSpecified`, `CancelMessage`, `PendingMessage`, `GenericError`,
  `NotEnoughStockWarning`, `CartValidationWarning`, `TwoC2PSettingsError`, `RefundNotSupported`.

---

## 7. Phased work plan

**Phase 1: checkout via the hosted page (MVP)**
1. Scaffold the project and test project, add them to the `.sln`, and add `TwoC2PConfiguration` and the Commerce Manager `ConfigurePayment` control.
2. Add `JwtHs256` and its unit tests (round-trip, a known vector from the 2C2P docs sample, tampered signature, `alg:none`).
3. Add `TwoC2PClient` (`paymentToken`, `paymentInquiry`) behind an interface so it can be mocked.
4. Add `TwoC2PPaymentMethod`, `TwoC2PPage`, and `TwoC2PPaymentGateway` (cart path plus Capture/Credit behaviour from §3).
5. Add `TwoC2PPaymentController` (`Index`, `Return`) and the shared completion service, refactored to be session-independent and idempotent.
6. Add the notification endpoint and its route registration.
7. Add MVP views, the language file, and the logo. Update the README with install steps (deploy the CM folder, create the payment method with
   system keyword `2C2P` and class names, create the page and start-page properties, configure the parameters).
8. Run a sandbox end-to-end test: success, decline, cancel, browser closed after paying (notification only), duplicate notification,
   amount tampering (inquiry mismatch), and a pending channel.

**Phase 2: back-office operations**
9. Add a JWE/JWS crypto helper (choose `jose-jwt` or BouncyCastle) and certificate configuration.
10. Implement Settle on Capture, Refund on Credit (supports partial amounts), and Void.
11. Run a sandbox test from Commerce Manager: release shipment → settle, return → refund, and same-day cancel → void.

**Phase 3 (optional)**: Secure Fields/Direct API embedded card form, card tokenization for returning customers, installment (IPP) options,
and support for the CSR UI (as `AuthorizeTokenEx.CSR` does).

---

## 8. Tests (in `EPiServer.Business.Commerce.Payment.TwoC2P.Tests`)

* `JwtHs256Tests`: encode/decode, signature mismatch, wrong alg, malformed token.
* `TwoC2PRequestBuilderTests`: field mapping, unique invoiceNo, `userDefined1` = cart id, absolute return URLs, amount formatting per currency.
* `FrontendResponseTests`: base64url decoding with and without padding, POST and GET forms.
* `PaymentResultValidatorTests`: success, non-0000 codes, amount mismatch, currency mismatch, invoice mismatch, merchant mismatch.
* `TwoC2PPaymentGatewayTests`: cart → redirect, PO Capture with `AutoSettle` on/off, Credit → unsupported (Phase 1).
* Reuse the fake order classes in `DIBS.Tests/TestSupport`. Copy them, since every test project in this repo carries its own copy.

Note: these are `net461` projects that reference Episerver/Mediachase (`System.Web`, WebForms), so building and testing needs Windows / Visual Studio
(or a Windows CI agent).

---

## 9. Open questions (for the merchant / product owner)

1. Which PGW version is the merchant account on: 4.3 or 4.5? Do sandbox credentials (MerchantID + Secret Key) exist yet?
2. Is the account set to **auto-settle** or **pre-auth + manual settle**? This decides whether Phase 2 is required before go-live.
3. Must refunds be issued from Commerce Manager at launch, or is the 2C2P merchant portal acceptable for now?
4. Which currencies and payment channels are needed (cards only, or also QR, e-wallets, or counter payments that settle later)?
5. Is the site a web farm? This affects the idempotency/locking strategy in §5.5.
6. Is the WebForms front-end (`Frontend/PaymentMethod.ascx`) still needed, or only MVP?

---

## Sources

* 2C2P – Payment Token: https://developer.2c2p.com/docs/api-payment-token
* 2C2P – JWT: https://developer.2c2p.com/docs/json-web-tokens-jwt
* 2C2P – Sandbox Setup: https://developer.2c2p.com/docs/sandbox-setup
* 2C2P – Redirect: How to integrate: https://developer.2c2p.com/docs/redirect-api-integrate-with-payment
* 2C2P – Payment Response (Frontend): https://developer.2c2p.com/docs/api-payment-response-frontend
* 2C2P – Payment Response (Backend): https://developer.2c2p.com/docs/api-payment-response-backend
* 2C2P – Payment Inquiry: https://developer.2c2p.com/docs/api-payment-inquiry
* 2C2P – Payment Inquiry Response Parameters: https://developer.2c2p.com/docs/api-payment-inquiry-response-parameter
* 2C2P – Payment Flow Codes: https://developer.2c2p.com/docs/response-code-payment-flow
* 2C2P – Payment Maintenance: https://developer.2c2p.com/docs/payment-action-api-spec
* 2C2P – Refund / Void / Settle (v3.2.6): https://developer.2c2p.com/v3.2.6/docs/refund-request, https://developer.2c2p.com/v3.2.6/docs/void-cancel-request, https://developer.2c2p.com/v3.2.6/docs/settle-request
* Third-party API profile: https://github.com/api-evangelist/2c2p
* Reference client (PHP): https://github.com/laraditz/2c2p
