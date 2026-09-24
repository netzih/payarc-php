# PayArc port: plan and open questions

Plan agreed on 2026-09-23. The goal is PayArc versions of the two USAePay
integrations (`usaepayjs` for CiviCRM and `usaepay-wordpress`).

## Decisions

- **Separate forks.** Three new repositories under `~/dev`, copied from the
  USAePay ones and renamed:
  - `payarc-php`: this library.
  - `payarc-wordpress`: the "PayArc Payments" plugin.
  - `payarcjs`: the CiviCRM extension.

  Every prefix, meta key, hook and gateway id is renamed, so both plugins can
  run on one site.
- **Scope of v1.** The CiviCRM extension plus the WooCommerce, Gravity Forms
  and GiveWP modules. Apple Pay and Google Pay are included, through the
  Hosted Fields wallet buttons. The CiviCRM transaction importer is deferred.
- **Recurring stays site-managed.** Saved cards are charged by cron, as in
  the USAePay versions. PayArc's own plans have a fixed amount, which does not
  suit donor-chosen gifts.
- **Card entry is PayArc Hosted Fields** (`iframeprocess.js`, Client ID in
  the browser). Hosted Checkout, a modal driven by a server-side order, is
  the fallback if Hosted Fields turn out unusable.
- **The CiviCRM extension ships this library** in its own `vendor/`, instead
  of keeping a separate copy of the client (the USAePay extension did that,
  and the copy drifted).

## Phases

1. **This library.** Done: client, response readers, donor wording, unit
   tests, credential check and sandbox probes.
2. **`payarc-wordpress`.**
   - Fork and rename.
   - Settings: Client ID and bearer token for live and sandbox, plus extra
     accounts.
   - `assets/js/payarc-hostedfields.js`: the same `{mount, tokenize,
     applePay, errorText}` surface as `UsaepayPayJs`, wrapping PayArc's
     global-function script.
   - Then WooCommerce, Gravity Forms and GiveWP. Wallets last.
   - `Gateway::approved()` and its siblings map onto `Charge::outcome()`.
     `Reconcile` uses the idempotent resend or `findChargeByReference()`,
     depending on what the probe finds.
3. **`payarcjs`.**
   - Fork and rename, then bundle this library.
   - Rewrite `assertApproved`, the card-entry JS, the processor type fields
     (bearer token and Client ID), the status checks and the wallets.
   - The recurring cron, pre-approval, token storage and refunds carry over.
4. **Sandbox gate.** Finish the open questions below, then run the USAePay
   acceptance checklist against PayArc. Finish with a live smoke test: a
   small charge, then a refund.

## Open questions (answered by the probes)

| # | Question | Why it matters | Probe |
|---|---|---|---|
| 1 | Can a Hosted Fields token be attached to a customer, and is the card `is_verified`? | Unverified cards need a CVV on later charges, and recurring charges have no CVV | web harness: save |
| 2 | Does a saved card charge without a CVV, with `eci_indicator` 2? | Recurring depends on it | probe `charge_saved_*`, web harness |
| 3 | Does resending with the same `Idempotency-Key` return the original charge? What happens if the body differs? How long is a key kept? | Decides how `Reconcile` settles lost responses | probe `idempotent_*` |
| 4 | Is a token single-use even across different keys? | Signup flows: save first, then charge the saved card | probe `token_reuse` |
| 5 | How is metadata returned on GET and in the list? What format is `created_at` in, and which zone? | `findChargeByReference()` and time cutoffs | probe `get_charge`, `list_shape`, `find_by_reference` |
| 6 | Is the refund refusal on an unsettled charge exactly "Return Not Allowed."? What does voiding twice return? | `reverse()` depends on the wording | probe `reverse_*`, `refund_raw_unsettled`, `void_twice` |
| 7 | Is `description` accepted and shown as `charge_description`? | Staff find charges in the dashboard | probe `charge_token` |
| 8 | Where does a decline appear: HTTP error or `failure_code`? Which test inputs decline? | `Charge::outcome()` and `DonorMessage` | probe `decline_*` |
| 9 | Do the `do_not_send_*` flags actually stop PayArc's emails and texts? | Donors must not get double receipts | check the probe inbox and dashboard after a run |
| 10 | Does PayArc create or email anything for a customer at `saveCard()`? | Same | same |
| 11 | Does the Hosted Fields script work on `localhost`, or only on registered domains? | Local development and the browser tests | web harness |
| 12 | How does the wallet popup behave on a real HTTPS domain? | Apple Pay and Google Pay in v1 | Phase 4, on the staging site |

Record each answer here with the date and the report file it came from.
Then update the code comments that say "assumed", "not documented" or
"INFERRED".
