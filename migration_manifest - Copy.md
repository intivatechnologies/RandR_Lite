# Migration Manifest — Backend Debrief

## Stack
- Node, Express, Axios, Stripe, Nodemailer, Multer, PdfKit, Socket.io
- Firebase (session/auth persistence)
- Google OAuth2
- Stripe CLI (webhook testing)
- UPS API (mocked — real UPS objects/properties that get used with API, no live credentials or authorization required)

---

## Admins - Authentication & Session Management

### Functionality
- Admin login via username/password (POST) or Google OAuth2
- Returns boolean authorization response to client based on
    a) username and password validation
    b) Google authentication and recognized as an admin account
- Session persists via Firebase — survives app reboot so long as app is running
- WebSocket connection allows client to actively poll auth status
- Multi-instance aware: logging out on one dashboard instance broadcasts logout state
  to all open instances via web socket

### Storage
- Firebase Auth — handles Google OAuth2 and session token persistence
- Firebase Firestore or Realtime Database — stores admin credential record and session state

### API Calls
- POST /api/auth/login — receives username + password, validates against Firebase,
  returns auth boolean + session token
- POST /api/auth/google — receives Google OAuth token, validates via Firebase Auth,
  returns auth boolean + session token
- POST /api/auth/logout — invalidates Firebase session
- GET /api/auth/status — returns current auth status for active session
- WS /ws/auth — Socket.io channel; broadcasts auth state changes to all open
  client instances

### Frontend Connection
- Admin dashboard login page consumes POST /api/auth/login and POST /api/auth/google
- Dashboard layout maintains WS /ws/auth Socket.io connection to reactively handle
  logout across tabs

---

## Newsletter

### Functionality
- Admin composes emails with embedded images and text
- Admin can view archive of all previously sent newsletters
- Sends to all stored follower email addresses via Nodemailer

### Storage
- Firebase Firestore:
  - newsletter_followers collection — follower email accounts; populated
    on signup when opted in, and on PUT /api/accounts/:id/newsletter
    when user opts in post-signup; removed on opt-out or account deletion
  - newsletter_archive collection — each sent newsletter stored as document
    with HTML content, subject, and timestamp metadata

### API Calls
- GET /api/newsletter/archive — returns list of all previously sent newsletters
- GET /api/newsletter/archive/:id — returns full content of a specific past newsletter
- POST /api/newsletter/send — composes and delivers email to all followers,
  stores to archive
- POST /api/newsletter/followers — adds a new follower email
- DELETE /api/newsletter/followers/:id — removes a follower

### Frontend Connection
- NewsletterEditor.vue consumes POST /api/newsletter/send
- Public newsletter signup form consumes POST /api/newsletter/followers

---

## Accounts

### Functionality
- User can log in to a pre-existing account using saved credentials.
- On signup, backend requires a referral/source survey response (typical
  "How did you hear about us" spinner or equivalent single-select field).
- On signup, backend requires home/mailing address fields; that address is stored
  on the user profile and used as the destination for shipping estimates and
  order flow (no separate address step on the product order UI).
- Backend evaluates local `Pickup` eligibility by testing the user's saved
  home/mailing address against the admin's configured pickup radius from the
  admin's home location.
- User can view account data, including current shipment package statuses and
  a summary of recent package activity tied to their account.
- User can update newsletter preferences (opt in/opt out) from account settings.
- User can delete their account and is prompted to complete an account-deletion
  survey before final deletion is processed.

### Storage
- Firebase Auth — stores user login credentials and authentication identity.
- Firebase Firestore:
  - users collection — profile/account metadata, newsletter preference
    (e.g., `newsletterOptIn: boolean`), `hearAboutUs` (or equivalent) survey
    answer from signup, and `homeMailingAddress: { street, city, state, zip, country }`
  - newsletter_followers collection — user email is added when opted in at signup

### API Calls
- POST /api/accounts/signup — creates user account, persists hearAboutUs,
  home/mailing address, evaluates and stores pickup eligibility, stores user
  profile record, conditionally adds email to newsletter_followers when opted in
- POST /api/accounts/login — authenticates user credentials via Firebase Auth,
  returns session token
- GET /api/accounts/:id — returns user profile data including shipment package
  statuses and recent package activity (authenticated)
- PUT /api/accounts/:id — updates user profile fields (authenticated)
- PUT /api/accounts/:id/newsletter — updates newsletterOptIn preference,
  adds or removes email from newsletter_followers collection accordingly
- DELETE /api/accounts/:id — prompts deletion survey, processes account
  deletion from Firebase Auth and Firestore users collection on survey
  submission
- POST /api/accounts/:id/deletion-survey — stores deletion survey response
  before account deletion is finalized

### Frontend Connection
- Public signup form consumes POST /api/accounts/signup
- Login page consumes POST /api/accounts/login
- Account settings page consumes GET /api/accounts/:id,
  PUT /api/accounts/:id, PUT /api/accounts/:id/newsletter
- Account deletion flow consumes POST /api/accounts/:id/deletion-survey
  then DELETE /api/accounts/:id


---

## Projects

### Functionality
- Admin stores and manages restored antique projects
- Each project has two image galleries filtered as "before" and "after"
- Metadata per project

### Storage
- Firebase Firestore — projects collection, one document per project:
  - name: string
  - about: string
  - images: { before: string[], after: string[] } — Firebase Storage URLs
- Firebase Storage — hosts uploaded project images

### API Calls
- GET /api/projects — returns all projects (public)
- GET /api/projects/:id — returns single project with full image galleries (public)
- POST /api/projects — creates new project with metadata + images (admin)
- PUT /api/projects/:id — updates project metadata or images (admin)
- DELETE /api/projects/:id — removes project and associated images (admin)

### Frontend Connection
- ProjectCard.vue consumes GET /api/projects
- Project detail page consumes GET /api/projects/:id

---

## Products

### Functionality
- Admin stores and manages restored antique products available for purchase
- Single image gallery per product, no filters
- Full dimensional and pricing metadata per product

### Storage
- Firebase Firestore — products collection, one document per product:
  - name: string
  - about: string
  - price: number
  - weight: number
  - width: number
  - height: number
  - length: number
  - images: string[] — Firebase Storage URLs
- Firebase Storage — hosts uploaded product images
- Note: price in storage is never mutated by shipping estimates or quote
  adjustments. All price modifications exist only at the quote/invoice level.

### API Calls
- GET /api/products — returns all products (public)
- GET /api/products/:id — returns single product with full metadata (public)
- POST /api/products — creates new product with metadata + images (admin)
- PUT /api/products/:id — updates product metadata or images (admin)
- DELETE /api/products/:id — removes product and associated images (admin)

### Frontend Connection
- ProductCard.vue consumes GET /api/products
- Product detail page consumes GET /api/products/:id
- Product detail page initiates the order flow using the logged-in user's saved
  home/mailing address (no address re-entry on the product page)

---

## Order Flow

### Functionality

#### Client Side (order permissions)
- If the client has not signed up for an account, they are informed they can't start an order until they sign up.
- The client signs up or logs in, continuing with the acccount flow (See Above).

#### Client Side (order flow)
- Client views product (price, weight, dimensions)
- System uses the client's saved home/mailing address from their account profile
  to check pickup eligibility (distance from admin's home) and mocked UPS
  availability, then presents available options (`Pickup` and/or `Shipping`)
- If client selects `Pickup`, checkout can proceed immediately
- If client selects `Shipping`, checkout is blocked until admin review is complete
  and a confirmed offer is sent

#### Client Side (pickup handoff)
- Client is shown ready-to-pickup date/time before checkout
- Client proceeds directly to embedded Stripe checkout
- Client receives pickup confirmation details via email

#### Client Side (shipping handoff)
- Client submits quote request and sees confirmation that admin review is required
- Client receives email also informing them that admin review is required along with an invoice with the total estimated amount (product price + estimated shipping amount)
- Invoice record is created in Firestore with status: "PENDING_REVIEW"
- Client and admin both receive notification emails with invoice review details

#### Admin Side
- Admin views incoming shipping invoice review request on dashboard
- Admin reviews address and provisional shipping details
- Admin confirms or adjusts shipping amount and submits offer
- Invoice record is updated to status: "OFFER_SENT"
- Client is emailed offer details with a link to accept or reject

#### Client Response
- Client visits offer review page, views confirmed invoice offer
- Client either:
  a) Accepts offer → proceeds to Stripe embedded checkout →
     invoice status updated to "ACCEPTED"
  b) Rejects offer → invoice status updated to "REJECTED" →
     client is redirected to a survey page polling reason for rejection →
     deal is permanently closed, no further resumption possible

### Storage
- Firebase Firestore — quotes collection, one document per quote/invoice
  review record:
  - productId: string
  - productName: string
  - productPrice: number
  - weight: number
  - width: number
  - height: number
  - length: number
  - clientEmail: string
  - clientName: string
  - userId: string — authenticated user account reference
  - destinationAddress: { street, city, state, zip, country } — pulled
    from user account profile at time of order, not re-entered
  - eligibilityType: string ("LOCAL_PICKUP" | "UPS_DELIVERY")
  - selectedRate: { serviceLevel, estimatedCost, estimatedTransitDays }
    — null if LOCAL_PICKUP
  - adminShippingCost: number — admin's confirmed shipping cost
  - pickupLabel: boolean — true if pickup, false if shipping
  - provisionalTotal: number — product price + estimated shipping
  - confirmedTotal: number — product price + admin confirmed shipping
  - status: string ("PENDING_REVIEW" | "OFFER_SENT" | "ACCEPTED" |
    "REJECTED" | "COMPLETED")
  - rejectionSurveyResponse: string — populated if client rejects
  - timestamp: string
  - offeredAt: string
  - resolvedAt: string

### API Calls
- POST /api/shipping/estimate — receives productId (authenticated session);
  server loads the user's saved home/mailing address as destination, checks
  eligibility against admin residence radius, returns either LOCAL_PICKUP flag
  or array of mocked UPS rate options
- POST /api/checkout/session/pickup — client selects pickup and starts
  immediate Stripe embedded checkout session
- POST /api/quotes — client submits selected rate (authenticated); server uses
  account home/mailing address on the invoice review record, creates Firestore
  invoice review record with status "PENDING_REVIEW", emails client
  confirmation and admin notification via Nodemailer
- GET /api/quotes — returns all quote records (admin)
- GET /api/quotes/:id — returns single quote record (admin + client
  via token)
- PUT /api/quotes/:id/offer — admin submits confirmed shipping cost or
  pickup revocation, updates invoice review record to status "OFFER_SENT",
  emails client confirmed offer via Nodemailer
- PUT /api/quotes/:id/accept — client accepts offer, updates status to
  "ACCEPTED", initiates Stripe checkout session
- PUT /api/quotes/:id/reject — client rejects offer, updates status to
  "REJECTED", returns survey prompt to frontend
- POST /api/quotes/:id/survey — stores client rejection survey response
  to Firestore quote record

### Frontend Connection
- Product detail page order UI consumes POST /api/shipping/estimate
  (session carries account address; no address form on this page)
- Product detail page pickup action consumes POST /api/checkout/session/pickup
- Pickup confirmation screen renders pickup date/time and consumes
  POST /api/checkout/session for embedded Stripe checkout
- Product detail page shipping action consumes POST /api/quotes
- Client confirmation screen renders provisional invoice from quote record
- Offer review page consumes GET /api/quotes/:id,
  PUT /api/quotes/:id/accept, PUT /api/quotes/:id/reject
- Survey page consumes POST /api/quotes/:id/survey
- Admin dashboard quote panel consumes GET /api/quotes,
  PUT /api/quotes/:id/offer

---

## Stripe Checkout

### Functionality
- Triggered from two paths:
  - Pickup flow: immediate checkout after pickup selection
  - Shipping flow: checkout only after client accepts confirmed offer
- Stripe checkout is embedded directly on this web app (not redirected)
- Checkout billed amount is calculated at session creation time as
  product price + live shipping amount
- For pickup, live shipping amount is 0
- Product price in storage remains immutable
- Stripe CLI used for local webhook testing
- On successful checkout (webhook confirmed):
  - Unofficial invoice is upgraded to official invoice
  - Official invoice metadata written to Firebase Storage
  - Invoice PDF generated via PdfKit with all product, address, pricing,
    shipping, and quote information
  - PDF emailed to client with personalized message via Nodemailer confirming that shipment has started
  - PDF emailed to admin with personalized message via Nodemailer confirming that shipment has started
  - PDF is not stored — generated at send time only
  - UPS shipment record created in Firestore, shipment lifecycle begins

### Storage
- Firebase Storage — invoice metadata JSON files:
  - quoteId, productId, productName, clientEmail, clientName,
    destinationAddress, productPrice, liveShippingAmount, billedTotal,
    tax, pickupLabel, timestamp, stripePaymentIntentId
- Note: Invoice PDFs are generated and emailed at checkout time only.
  Not stored anywhere. Regenerated on demand from metadata if retrieved
  via dashboard.

### API Calls
- POST /api/checkout/session/pickup — receives productId and authenticated
  userId, validates pickup eligibility, computes product price only
  (shipping = 0), returns Stripe embedded checkout session client secret
- POST /api/checkout/session — receives quoteId, validates quote status
  is "ACCEPTED", computes product price + confirmed shipping amount,
  returns Stripe embedded checkout session client secret
- POST /api/checkout/webhook — Stripe webhook listener (Stripe CLI for
  local testing); on payment success:
    - Writes invoice metadata to Firebase Storage
    - Generates and emails PDF invoice to client and admin via Nodemailer
    - Creates shipment record in Firestore
    - Updates quote or pickup record status to "COMPLETED"

### Frontend Connection
- Pickup confirmation screen consumes POST /api/checkout/session/pickup
  and mounts Stripe embedded checkout widget
- Offer review page accept flow consumes POST /api/checkout/session and
  mounts Stripe embedded checkout widget
- Stripe webhook drives all post-payment automation server-side

---

## MRR & Invoice Reporting

### Functionality
- Admin access to monthly revenue reports derived from invoice metadata
- Admin access to full invoice list and individual invoice retrieval
- On individual invoice retrieval, PDF is regenerated on demand from
  stored metadata via PdfKit — not retrieved from storage
- Admin access to deleting invoices via select / select all functionality
- Deletion recalculates and updates monthly sum and total sum in
  Firebase Storage
- Regenerated PDFs are built strictly from stored metadata and do not
  recalculate invoice amounts at retrieval time
- Metadata integrity validation runs before PDF regeneration

### Storage
- Firebase Storage — single source of truth, structured as JSON files:
  - invoices/:id.json — individual invoice metadata records
  - reports/monthly.json — total revenue summed per calendar month
  - reports/total.json — total revenue across all invoices

### API Calls
- GET /api/reports/mrr — reads reports/monthly.json from Firebase
  Storage, returns grouped monthly revenue
- GET /api/reports/invoices — reads all invoices/:id.json records,
  returns full metadata list
- GET /api/reports/invoices/:id — reads single invoices/:id.json,
  validates metadata integrity, regenerates PDF on demand from metadata via
  PdfKit, returns metadata + PDF as download
- DELETE /api/reports/invoices/:id — removes invoices/:id.json,
  recalculates and rewrites reports/monthly.json and reports/total.json

### Frontend Connection
- Admin dashboard MRR panel consumes GET /api/reports/mrr
- Admin dashboard invoice list consumes GET /api/reports/invoices
- Admin dashboard invoice detail consumes GET /api/reports/invoices/:id

---

## Shipping (UPS — Mocked)

### Functionality

#### Pre-Checkout Shipping Quote Flow
- Client selects a product and views its pricing, weight, and dimensions
- System loads client's saved home/mailing address from account profile
  as destination — no address entry on the product page
- System calculates an estimated shipping cost based on product
  dimensions/weight and stored destination address using mocked UPS
  rate logic
- Client views estimated shipping cost and selects preferred rate
- Client submits quote request — admin is notified via Nodemailer
- Admin reviews and confirms shipping terms in-app and sends offer
  to client
- Stripe checkout does not proceed until admin has responded and
  client confirms
- Mock UPS objects stay aligned to live UPS request/response shapes
  so migration to live UPS can occur without data-model refactors

#### Shipment Status Automation
- Once admin attaches a tracking number to an invoice, shipment is considered started
- Client receives an email (Nodemailer) when shipment starts
- Shipment status lifecycle is updated by automated job/webhook
- Client receives an email automatically whenever shipment status changes
- Client can visit a dedicated status page at any time to view live shipment status
- Status page updates in real time via WebSocket (Socket.io)

### Storage
- Firebase Firestore:
  - shipping_quotes collection — one document per quote request:
    - productId: string
    - productName: string
    - weight: number
    - width: number
    - height: number
    - length: number
    - destinationAddress: { street, city, state, zip, country }
    - estimatedShippingCost: number (mocked calculation)
    - status: string (e.g. "PENDING_REVIEW", "QUOTED", "CONFIRMED")
    - clientEmail: string
    - adminNotes: string (admin fills in confirmed price + timeline)
    - timestamp: string
  - shipments collection — one document per active shipment:
    - trackingNumber: string
    - invoiceId: string
    - clientEmail: string
    - status: string (e.g. "LABEL_CREATED", "IN_TRANSIT",
      "OUT_FOR_DELIVERY", "DELIVERED")
    - estimatedDelivery: string
    - lastNotifiedStatus: string — tracks last emailed status to prevent
      duplicate notifications
    - activity: [ { location, description, timestamp } ]

### Mock UPS Object Shapes

#### Rate Estimate Request (mirrors UPS Rate API)
  {
    package: {
      weight: number,
      dimensions: { width: number, height: number, length: number }
    },
    destination: {
      street: string,
      city: string,
      state: string,
      zip: string,
      country: string
    }
  }

#### Rate Estimate Response (mirrors UPS Rate API)
  {
    rates: [
      {
        serviceLevel: string (e.g. "UPS_GROUND", "UPS_2DAY", "UPS_NEXT_DAY"),
        estimatedCost: number,
        currency: string,
        estimatedTransitDays: number
      }
    ]
  }

#### TrackResponse (mirrors UPS Track API)
  {
    trackingNumber: string,
    status: string,
    estimatedDelivery: string,
    activity: [
      {
        location: { city: string, state: string, country: string },
        description: string,
        timestamp: string
      }
    ]
  }

### API Calls

#### Pre-Checkout Quote Flow
- POST /api/shipping/estimate — receives productId (authenticated session);
  server loads user's saved home/mailing address as destination, checks
  eligibility against admin residence radius, returns LOCAL_PICKUP flag
  or array of mocked UPS rate options
- Note: quote submission is handled by POST /api/quotes in Order Flow —
  not duplicated here
- GET /api/shipping/quotes — returns all pending quote requests (admin)
- PUT /api/shipping/quotes/:id — admin updates quote record with confirmed
  price and timeline, triggers email to client via Nodemailer

#### Shipment Tracking
- POST /api/shipping/attach — admin attaches tracking number to invoice record,
  creates shipment document in Firestore, emails client that shipment has started
- GET /api/shipping/status/:trackingNumber — queries Firestore shipment record,
  returns UPS-shaped TrackResponse object
- POST /api/shipping/status/:trackingNumber/update — admin or automated job
  (webhook/job) updates shipment status; if status differs from
  lastNotifiedStatus, triggers
  Nodemailer email to client and updates lastNotifiedStatus in Firestore
- WS /ws/shipping/:trackingNumber — Socket.io channel; pushes live status
  updates to client status page whenever shipment document changes

### Frontend Connection
- Product detail page consumes POST /api/shipping/estimate using session
  account address (no address form on this page)
- Client-facing shipment status page consumes
  GET /api/shipping/status/:trackingNumber and maintains
  WS /ws/shipping/:trackingNumber connection for real-time updates
- Admin dashboard quote management panel consumes
  GET /api/shipping/quotes and PUT /api/shipping/quotes/:id
- Admin dashboard order panel consumes POST /api/shipping/attach and
  POST /api/shipping/status/:trackingNumber/update

## Static API Calls
- POST /api/contact — contact form submission, delivers via Nodemailer to admin

---

## Status Key
- [ ] Not started
- [~] In progress
- [x] Done + tested
- [!] Blocked