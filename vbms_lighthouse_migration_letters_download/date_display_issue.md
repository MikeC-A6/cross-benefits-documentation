### Lighthouse Claim Letters: 1-day Earlier Date Display Analysis

- **Issue**: Dates on the claim letters list show one day earlier when using the Lighthouse implementation, but not with VBMS.
- **Impact**: Veterans in US time zones see the prior calendar day for letter dates.
- **Root cause**: A date-only string (YYYY-MM-DD) from Lighthouse is parsed/treated as UTC midnight and then formatted in the user’s local time, which shifts the date back for time zones behind UTC.

### Where it happens
- **Frontend display (vets-website)**
  - Component: `ClaimLetterListItem.jsx` formats the letter date by calling a formatter built in `utils/helpers.js`.
    - `ClaimLetterListItem.jsx` (formats `letter.receivedAt`): [link](https://github.com/department-of-veterans-affairs/vets-website/blob/main/src/applications/claims-status/components/claim-letters/ClaimLetterListItem.jsx)
    - `utils/helpers.js` → `buildDateFormatter()` uses date-fns `parseISO` then `format`; `parseISO('YYYY-MM-DD')` treats the date as UTC midnight: [link](https://github.com/department-of-veterans-affairs/vets-website/blob/main/src/applications/claims-status/utils/helpers.js)

- **Backend data shape (vets-api)**
  - Migration PR to Lighthouse provider (adds `LighthouseClaimLettersProvider`): [vets-api PR #22643](https://github.com/department-of-veterans-affairs/vets-api/pull/22643)
  - Lighthouse provider constructs `received_at` from LH’s date-only `receivedAt` using `Time.zone.parse`, which typically becomes midnight UTC when serialized and consumed by the frontend:
    - `lib/claim_letters/providers/claim_letters/lighthouse_claim_letters_provider.rb` (maps `receivedAt` → `received_at`): [link](https://github.com/department-of-veterans-affairs/vets-api/blob/master/lib/claim_letters/providers/claim_letters/lighthouse_claim_letters_provider.rb)
  - The older VBMS flow historically returned a zone-aware timestamp (not a bare date), which did not shift when formatted locally:
    - VBMS downloader: `lib/claim_letters/claim_letter_downloader.rb` (VBMS path): [link](https://github.com/department-of-veterans-affairs/vets-api/blob/master/lib/claim_letters/claim_letter_downloader.rb)

### Why the 1-day shift only with Lighthouse
- Lighthouse returns a date-only value (`YYYY-MM-DD`). The frontend pipeline does `parseISO(dateOnly)` → treats it as `YYYY-MM-DDT00:00:00Z` → then formats in local time. In US time zones, UTC midnight is the prior calendar day locally, so the UI shows “one day earlier.”
- VBMS returned a timezone-aware timestamp (or at least a time that preserved the intended local calendar day when formatted), so no shift occurred.

### How to reproduce
- Feed the frontend `receivedAt: "2023-06-15"` (date-only) from Lighthouse. With `parseISO`, a user in ET/PT will see “June 14” in the UI due to UTC→local conversion.
- Feed the frontend `receivedAt: "2023-06-15T00:00:00-04:00"` (or similar zone-aware timestamp) as VBMS did; UI remains “June 15.”

### Recommendations

- **Frontend (safest in the long run)**
  - Teach the formatter to treat date-only values as calendar dates, not UTC instants:
    - Detect `YYYY-MM-DD` and use date-fns `parse(dateStr, 'yyyy-MM-dd', new Date())` instead of `parseISO` for those inputs, then `format` as today.
    - Continue using `parseISO` for full timestamps.
  - Benefit: works regardless of upstream provider; avoids regressions if another service emits date-only values.

- **Backend (normalize Lighthouse to match VBMS semantics)**
  - Emit a timezone-aware timestamp that preserves the intended calendar date when formatted by clients, e.g.:
    - Use local midnight with explicit offset (e.g., `2023-06-15T00:00:00-05:00`) or
    - Use noon UTC (`2023-06-15T12:00:00Z`) so all US time zones still render the same calendar date.
  - Alternatively, return the date-only field and codify that clients must treat it as a calendar date (i.e., pair with the frontend change).

### Notes on related logic
- The Lighthouse provider’s BOA filtering and sorting logic already tolerates `nil` dates and does not cause the 1-day shift. The shift strictly results from parsing date-only inputs as UTC instants on the client.

### References (code URLs)
- vets-website
  - `ClaimLetterListItem.jsx` (uses the date formatter): [vets-website ClaimLetterListItem.jsx](https://github.com/department-of-veterans-affairs/vets-website/blob/main/src/applications/claims-status/components/claim-letters/ClaimLetterListItem.jsx)
  - `utils/helpers.js` (date formatting helpers): [vets-website helpers.js](https://github.com/department-of-veterans-affairs/vets-website/blob/main/src/applications/claims-status/utils/helpers.js)
- vets-api
  - Migration PR: [PR #22643](https://github.com/department-of-veterans-affairs/vets-api/pull/22643)
  - Lighthouse provider mapping `receivedAt` → `received_at`: [lighthouse_claim_letters_provider.rb](https://github.com/department-of-veterans-affairs/vets-api/blob/master/lib/claim_letters/providers/claim_letters/lighthouse_claim_letters_provider.rb)
  - VBMS downloader (legacy path): [claim_letter_downloader.rb](https://github.com/department-of-veterans-affairs/vets-api/blob/master/lib/claim_letters/claim_letter_downloader.rb)

### TL;DR
- **Root cause**: Date-only strings from Lighthouse are parsed as UTC and then formatted in local time on the frontend → calendar date shifts back in US time zones.
- **Fix**: Either (a) frontend: parse date-only values as local calendar dates; or (b) backend: return zone-aware timestamps (e.g., noon UTC) to preserve the calendar date across time zones.

### Validation: VBMS provided datetime; Lighthouse provided date-only

- **VBMS (legacy) evidence**
  - The VBMS downloader returns both `received_at` and `upload_date` and sorts by `received_at`, which is a time value (not just a string date). See the legacy downloader:
    - `lib/claim_letters/claim_letter_downloader.rb`: [link](https://github.com/department-of-veterans-affairs/vets-api/blob/master/lib/claim_letters/claim_letter_downloader.rb)
  - The fixture used by the legacy endpoint shows `received_at` and `upload_date` fields. Although the fixture values are in `YYYY-MM-DD` format, these are mapped into time objects by the app code and were historically emitted to the frontend as timestamps. Fixture example entries (note `received_at` and `upload_date`):
    - `spec/fixtures/claim_letter/claim_letter_list.json`: [link](https://github.com/department-of-veterans-affairs/vets-api/blob/master/spec/fixtures/claim_letter/claim_letter_list.json)

- **Lighthouse (new) evidence**
  - Lighthouse `claim-letters/search` returns `receivedAt` as a date-only value and `uploadedDateTime` as an ISO datetime. The provider maps `receivedAt` → `received_at` via `Time.zone.parse(letter['receivedAt'])` and passes it downstream:
    - `lib/claim_letters/providers/claim_letters/lighthouse_claim_letters_provider.rb` (mapping and parse): [link](https://github.com/department-of-veterans-affairs/vets-api/blob/master/lib/claim_letters/providers/claim_letters/lighthouse_claim_letters_provider.rb)
  - Because `receivedAt` is date-only, parsing yields a midnight instant (typically UTC). On the web, the formatter uses `parseISO` + local `format`, which shifts the visible date for US time zones.

- **Conclusion**
  - VBMS path produced a date+time (zone-aware when serialized to the client) that preserved the intended calendar date during client formatting.
  - Lighthouse path introduced a date-only `receivedAt`, which—when parsed and then formatted in local time—causes the observed 1-day earlier display for users behind UTC.
