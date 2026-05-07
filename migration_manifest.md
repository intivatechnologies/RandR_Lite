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
- Admin login via:
  - Email/password authentication through Firebase Auth
  - Google OAuth2 through Firebase Auth
- Backend validates admin authorization separately from authentication
- Returns authorization response to client based on:
    a) successful Firebase token validation
    b) successful admin profile lookup in Firestore
- Firebase Auth handles:
  - session persistence
  - token refresh
  - token expiration
  - identity lifecycle
- Backend remains stateless regarding authentication sessions
- Sessions survive backend restarts automatically through Firebase Auth persistence
- Optional Socket.io integration may broadcast logout/account revocation events
  across open admin dashboard instances
- Socket.io is never authoritative for authentication state
- Firebase Auth + Firestore remain the single source of truth
- Multi-instance aware:
  - if logout or admin revocation occurs on one dashboard instance,
    all connected instances reactively clear local auth state

### Storage
- Firebase Auth:
  - handles admin identity
  - email/password authentication
  - Google OAuth2 authentication
  - token/session lifecycle
- Firebase Firestore:
  - admin_profiles collection — stores authorization metadata:
    - uid: string
    - email: string
    - role: string
    - permissions: string[]
    - active: boolean
    - createdAt: string
  - optional audit_logs collection:
    - login events
    - logout events
    - failed access attempts
    - admin permission changes
- No server-side session persistence storage required unless:
  - forced logout
  - device tracking
  - audit history
  - concurrent session management
  are later introduced

### API Calls
- POST /api/auth/login — receives email + password,
  authenticates through Firebase Auth,
  validates admin authorization through Firestore,
  returns auth boolean + Firebase session token
- POST /api/auth/google — receives Firebase-authenticated Google token,
  validates token server-side,
  validates admin authorization through Firestore,
  returns auth boolean + Firebase session token
- POST /api/auth/logout — frontend signs out through Firebase Auth;
  optional backend route may revoke refresh tokens for forced logout support
- GET /api/auth/status — validates Firebase token,
  returns current authenticated + authorized admin state
- Optional:
  WS /ws/auth — Socket.io channel used only for reactive logout or
  authorization-change broadcasts across connected dashboard instances;
  never authoritative for auth state

### Frontend Connection
- Admin dashboard login page consumes:
  - POST /api/auth/login
  - POST /api/auth/google
- Frontend Firebase SDK maintains session persistence automatically
- Dashboard layout validates auth state through:
  - Firebase auth listeners
  - GET /api/auth/status
- Optional Socket.io connection reacts to:
  - logout broadcasts
  - admin revocation events
  - permission updates
- All protected frontend routes depend on backend token validation
  rather than local frontend auth state

---

## Newsletter

### Functionality
- Admin composes and sends text-based newsletters
- Newsletter content supports:
  - subject line
  - plain text content
  - optional lightweight HTML formatting
- Embedded images are not supported
- Admin can:
  - preview newsletter content before sending
  - view archive of all previously sent newsletters
  - retrieve individual historical newsletters
- Newsletters are delivered to all active newsletter subscribers via Nodemailer
- Newsletter sending is processed asynchronously to avoid request blocking
- Newsletter delivery lifecycle supports:
  - DRAFT
  - SENDING
  - SENT
  - FAILED
- Newsletter subscribers may originate from:
  - public newsletter signup form
  - account signup opt-in
  - account settings opt-in flow
- Unsubscribed users are preserved as inactive records rather than deleted

### Storage
- Firebase Firestore:

  - newsletter_followers collection — authoritative newsletter subscriber list:
    - email: string
    - userId: string | null
    - active: boolean
    - source: string ("PUBLIC_FORM" | "ACCOUNT_SIGNUP" | "ACCOUNT_SETTINGS")
    - subscribedAt: string
    - unsubscribedAt: string | null

  - newsletter_archive collection — one document per sent newsletter:
    - subject: string
    - content: string
    - previewText: string
    - status: string ("DRAFT" | "SENDING" | "SENT" | "FAILED")
    - recipientCount: number
    - sentBy: string
    - sentAt: string
    - createdAt: string

  - optional newsletter_jobs collection — tracks background send progress:
    - newsletterId: string
    - processedCount: number
    - failedCount: number
    - startedAt: string
    - completedAt: string | null
    - status: string

- newsletter_followers collection remains authoritative for delivery eligibility
- users collection newsletterOptIn field remains synchronized with
  newsletter_followers state
- Newsletter records are never deleted automatically
- Unsubscribed emails are never hard deleted in order to:
  - preserve suppression history
  - prevent accidental resubscription imports
  - preserve auditability

### API Calls
- GET /api/newsletter/archive — returns paginated newsletter summaries:
  - id
  - subject
  - status
  - recipientCount
  - sentAt

- GET /api/newsletter/archive/:id — returns full newsletter record

- POST /api/newsletter/send — receives newsletter content,
  creates newsletter archive record with status "SENDING",
  begins asynchronous delivery process,
  updates final delivery status on completion

- POST /api/newsletter/followers — validates and normalizes email,
  creates or reactivates newsletter subscriber record

- DELETE /api/newsletter/followers/:id — marks subscriber as inactive
  rather than permanently deleting the record

### Frontend Connection
- NewsletterEditor.vue consumes:
  - POST /api/newsletter/send
  - GET /api/newsletter/archive
  - GET /api/newsletter/archive/:id

- Newsletter editor supports:
  - preview rendering
  - send confirmation flow
  - delivery status feedback

- Public newsletter signup form consumes:
  - POST /api/newsletter/followers

- Account settings newsletter toggle consumes:
  - PUT /api/accounts/:id/newsletter

- Admin dashboard newsletter archive panel consumes:
  - GET /api/newsletter/archive
  - GET /api/newsletter/archive/:id

---

## Accounts

### Functionality
- User account authentication is handled through Firebase Auth
- Users can:
  - sign up for an account
  - log into an existing account
  - manage account/profile information
  - manage newsletter preferences
  - view shipment/package activity tied to their account
  - delete their account after completing a deletion survey
- Signup requires:
  - email/password credentials
  - "How did you hear about us" survey response
  - home/mailing address information
- Home/mailing address becomes the authoritative destination address
  for:
  - shipping estimates
  - quote generation
  - shipment records
  - pickup eligibility checks
- Product pages do not request shipping address re-entry
- Backend evaluates local pickup eligibility during signup and order flow
  using:
  - user's saved home/mailing address
  - admin-configured pickup radius
  - admin-configured pickup origin location
- Shipment activity attached to the user account updates dynamically as:
  - invoices progress
  - shipments are created
  - shipment statuses change
- Newsletter opt-in state synchronizes with the newsletter_followers collection
- Account deletion requires:
  - deletion survey submission
  - Firebase Auth account removal
  - Firestore profile cleanup
  - newsletter state cleanup/suppression update

### Storage
- Firebase Auth:
  - handles user identity
  - email/password authentication
  - token/session lifecycle

- Firebase Firestore:

  - users collection — authoritative user profile records:
    - uid: string
    - email: string
    - firstName: string
    - lastName: string
    - phoneNumber: string | null
    - newsletterOptIn: boolean
    - hearAboutUs: string
    - pickupEligible: boolean
    - createdAt: string
    - updatedAt: string
    - homeMailingAddress:
      {
        street: string,
        city: string,
        state: string,
        zip: string,
        country: string
      }

  - deletion_surveys collection — stores account deletion feedback:
    - userId: string
    - email: string
    - response: string
    - submittedAt: string

- Newsletter subscriber state is synchronized with:
  - newsletter_followers collection
- Shipment/package activity is derived from:
  - quotes collection
  - shipments collection
  - invoice metadata references

- User profile records remain authoritative for:
  - shipping destination address
  - pickup eligibility
  - newsletter preferences

### API Calls
- POST /api/accounts/signup — creates Firebase Auth account,
  validates required signup fields,
  stores Firestore user profile,
  evaluates pickup eligibility,
  synchronizes newsletter_followers subscription state if opted in,
  returns authenticated session token

- POST /api/accounts/login — authenticates through Firebase Auth,
  returns Firebase session token

- GET /api/accounts/:id — returns authenticated user profile,
  shipment summaries,
  package activity,
  newsletter preference state

- PUT /api/accounts/:id — updates editable user profile fields:
  - name
  - phone number
  - home/mailing address
  - newsletter preferences

- PUT /api/accounts/:id/newsletter — updates newsletterOptIn state,
  synchronizes newsletter_followers active state

- POST /api/accounts/:id/deletion-survey — stores deletion survey response
  before deletion is finalized

- DELETE /api/accounts/:id — deletes Firebase Auth account,
  removes or anonymizes Firestore profile data,
  disables newsletter subscription state,
  finalizes account deletion

### Frontend Connection
- Public signup form consumes:
  - POST /api/accounts/signup

- Login page consumes:
  - POST /api/accounts/login

- Account settings page consumes:
  - GET /api/accounts/:id
  - PUT /api/accounts/:id
  - PUT /api/accounts/:id/newsletter

- Shipment/activity panel consumes:
  - GET /api/accounts/:id

- Account deletion flow consumes:
  - POST /api/accounts/:id/deletion-survey
  - DELETE /api/accounts/:id

- Product order flow consumes authenticated account state
  for:
  - saved destination address
  - pickup eligibility
  - shipment history linkage

---

## Projects

### Functionality
- Admin manages restored antique showcase projects
- Projects are public-facing portfolio/display entries
- Each project contains:
  - project metadata
  - "before" image gallery
  - "after" image gallery
- Galleries are intentionally separated to support:
  - restoration comparisons
  - visual storytelling
  - portfolio presentation
- Admin can:
  - create projects
  - update project metadata
  - upload/remove images
  - delete projects
- Public users can:
  - browse all projects
  - open individual project detail pages
  - view before/after galleries independently

### Storage
- Firebase Firestore:

  - projects collection — one document per project:
    - id: string
    - name: string
    - about: string
    - createdAt: string
    - updatedAt: string
    - featured: boolean
    - images:
      {
        before: string[],
        after: string[]
      }

- Firebase Storage:
  - stores uploaded project image assets
  - Firestore stores only Storage URLs or Storage path references
  - Images are grouped logically by project ID:
    - /projects/{projectId}/before/
    - /projects/{projectId}/after/

- Project documents remain lightweight by storing only:
  - metadata
  - image references
- Binary image assets are never stored directly inside Firestore documents

### API Calls
- GET /api/projects — returns public project summaries:
  - id
  - name
  - preview image
  - short description
  - featured status

- GET /api/projects/:id — returns full project record:
  - metadata
  - before gallery
  - after gallery

- POST /api/projects — authenticated admin route;
  creates new project record,
  uploads images to Firebase Storage,
  stores Storage references in Firestore

- PUT /api/projects/:id — authenticated admin route;
  updates project metadata and/or image galleries,
  synchronizes Firebase Storage references

- DELETE /api/projects/:id — authenticated admin route;
  removes Firestore project record,
  deletes associated Firebase Storage image assets

### Frontend Connection
- Public project gallery page consumes:
  - GET /api/projects

- Project detail page consumes:
  - GET /api/projects/:id

- Admin dashboard project management panel consumes:
  - POST /api/projects
  - PUT /api/projects/:id
  - DELETE /api/projects/:id

- Admin project editor supports:
  - metadata editing
  - before/after gallery management
  - image upload/removal
  - project preview functionality

---

## Products

### Functionality
- Admin manages restored antique products available for purchase
- Products are public-facing purchasable inventory entries
- Each product contains:
  - pricing information
  - dimensional metadata
  - shipping metadata
  - image gallery
  - descriptive information
- Admin can:
  - create products
  - update metadata
  - upload/remove product images
  - delete products
- Public users can:
  - browse all products
  - open individual product detail pages
  - initiate authenticated order flow
- Product pages display:
  - product price
  - dimensions
  - weight
  - image gallery
  - pickup/shipping eligibility options after estimation
- Product base price stored in Firestore remains immutable
- Shipping adjustments and negotiated pricing exist only at:
  - quote level
  - invoice level
  - checkout session level

### Storage
- Firebase Firestore:

  - products collection — one document per product:
    - id: string
    - name: string
    - about: string
    - price: number
    - weight: number
    - width: number
    - height: number
    - length: number
    - featured: boolean
    - active: boolean
    - createdAt: string
    - updatedAt: string
    - images: string[]

- Firebase Storage:
  - stores uploaded product image assets
  - Firestore stores only Storage URLs or Storage path references
  - Images are grouped logically by product ID:
    - /products/{productId}/

- Product documents remain authoritative for:
  - base product pricing
  - dimensions
  - weight
  - storefront metadata

- Product documents are never mutated by:
  - shipping calculations
  - invoice adjustments
  - quote negotiations
  - Stripe checkout calculations

### API Calls
- GET /api/products — returns public product summaries:
  - id
  - name
  - price
  - preview image
  - featured status

- GET /api/products/:id — returns full product record:
  - metadata
  - dimensions
  - weight
  - image gallery

- POST /api/products — authenticated admin route;
  creates product record,
  uploads images to Firebase Storage,
  stores image references in Firestore

- PUT /api/products/:id — authenticated admin route;
  updates product metadata and/or image gallery,
  synchronizes Firebase Storage references

- DELETE /api/products/:id — authenticated admin route;
  removes Firestore product record,
  deletes associated Firebase Storage assets

### Frontend Connection
- Public product gallery page consumes:
  - GET /api/products

- Product detail page consumes:
  - GET /api/products/:id

- Product detail page initiates authenticated order flow:
  - POST /api/shipping/estimate
  - POST /api/quotes
  - POST /api/checkout/session/pickup

- Product detail page uses authenticated user's saved
  home/mailing address from account profile;
  no address re-entry occurs on the product page

- Admin dashboard product management panel consumes:
  - POST /api/products
  - PUT /api/products/:id
  - DELETE /api/products/:id

- Admin product editor supports:
  - metadata editing
  - pricing management
  - dimensional metadata editing
  - image upload/removal
  - storefront preview functionality

---

## Order Flow

### Functionality

#### Client Side (Order Permissions)
- Clients must be authenticated before starting any order flow
- If unauthenticated:
  - client is prompted to sign up or log in
  - order flow is blocked until authentication completes
- Authenticated account state becomes authoritative for:
  - destination address
  - pickup eligibility
  - shipment/account linkage
  - invoice ownership

#### Client Side (Order Flow)
- Client views product details:
  - price
  - dimensions
  - weight
  - gallery
- Backend loads authenticated user's saved home/mailing address
- Backend evaluates:
  - local pickup eligibility
  - mocked UPS shipping availability
- Backend returns available fulfillment methods:
  - LOCAL_PICKUP
  - UPS_DELIVERY
- If LOCAL_PICKUP is available:
  - client may proceed directly to pickup checkout
- If UPS_DELIVERY is selected:
  - checkout is blocked pending admin review
  - quote/invoice review flow begins

#### Client Side (Pickup Flow)
- Client selects pickup option
- Backend validates pickup eligibility again before checkout session creation
- Client is shown:
  - pickup instructions
  - pickup date/time estimate
- Stripe embedded checkout session is created immediately
- Client receives pickup confirmation email via Nodemailer

#### Client Side (Shipping Flow)
- Client selects shipping option and preferred estimated rate
- Client submits quote request
- Backend creates quote/invoice review record with:
  - status: "PENDING_REVIEW"
- Client receives:
  - provisional estimate summary
  - email confirmation
- Admin receives shipping review notification email

#### Admin Side (Quote Review)
- Admin views pending shipping quote requests in dashboard
- Admin reviews:
  - destination address
  - dimensions/weight
  - estimated shipping details
  - requested service level
- Admin may:
  - confirm shipping amount
  - adjust shipping amount
  - revoke shipping availability
- Backend updates quote status:
  - "OFFER_SENT"
- Client receives confirmed offer email containing:
  - confirmed totals
  - offer review link

#### Client Response Flow
- Client opens offer review page
- Client reviews confirmed invoice offer
- Client may:
  - accept offer
  - reject offer

##### Accept Flow
- Quote status updates to:
  - "ACCEPTED"
- Stripe embedded checkout session becomes available

##### Reject Flow
- Quote status updates to:
  - "REJECTED"
- Client is redirected to rejection survey flow
- Rejected quotes become permanently closed
- Rejected offers cannot be resumed or reopened

### Storage
- Firebase Firestore:

  - quotes collection — authoritative quote/invoice review records:
    - id: string
    - productId: string
    - productName: string
    - productPrice: number
    - weight: number
    - width: number
    - height: number
    - length: number
    - clientEmail: string
    - clientName: string
    - userId: string
    - destinationAddress:
      {
        street: string,
        city: string,
        state: string,
        zip: string,
        country: string
      }
    - eligibilityType: string
      ("LOCAL_PICKUP" | "UPS_DELIVERY")
    - selectedRate:
      {
        serviceLevel: string,
        estimatedCost: number,
        estimatedTransitDays: number
      } | null
    - adminShippingCost: number | null
    - pickupLabel: boolean
    - provisionalTotal: number
    - confirmedTotal: number | null
    - status: string
      ("PENDING_REVIEW" |
       "OFFER_SENT" |
       "ACCEPTED" |
       "REJECTED" |
       "COMPLETED")
    - rejectionSurveyResponse: string | null
    - createdAt: string
    - offeredAt: string | null
    - resolvedAt: string | null

- Quote records remain authoritative for:
  - negotiated shipping totals
  - shipping approval lifecycle
  - client acceptance/rejection state
  - checkout eligibility validation

- Product pricing stored in products collection remains immutable

### API Calls

#### Eligibility + Estimate
- POST /api/shipping/estimate — authenticated route;
  receives productId,
  loads authenticated user's saved address,
  evaluates pickup eligibility,
  generates mocked UPS rate estimates,
  returns:
  - LOCAL_PICKUP availability
  - UPS delivery rate options

#### Pickup Checkout Flow
- POST /api/checkout/session/pickup — authenticated route;
  validates:
  - pickup eligibility
  - product availability
  computes:
  - product price only
  creates Stripe embedded checkout session,
  returns Stripe client secret

#### Shipping Quote Flow
- POST /api/quotes — authenticated route;
  receives selected shipping rate,
  creates quote record with:
  - status: "PENDING_REVIEW"
  sends:
  - client confirmation email
  - admin notification email

- GET /api/quotes — authenticated admin route;
  returns quote list

- GET /api/quotes/:id — authenticated route;
  returns quote detail record
  (admin or owning client only)

- PUT /api/quotes/:id/offer — authenticated admin route;
  updates confirmed shipping amount,
  updates status:
  - "OFFER_SENT"
  emails client confirmed offer details

#### Client Offer Decision
- PUT /api/quotes/:id/accept — authenticated client route;
  validates quote ownership and status,
  updates status:
  - "ACCEPTED"

- PUT /api/quotes/:id/reject — authenticated client route;
  validates quote ownership and status,
  updates status:
  - "REJECTED"
  returns rejection survey prompt state

- POST /api/quotes/:id/survey — authenticated client route;
  stores rejection survey response

### Frontend Connection

#### Product Detail Page
Consumes:
- POST /api/shipping/estimate
- POST /api/quotes
- POST /api/checkout/session/pickup

Uses authenticated account state for:
- saved destination address
- pickup eligibility evaluation

No address form exists on product pages

#### Pickup Flow
- Pickup confirmation screen consumes:
  - POST /api/checkout/session/pickup
- Stripe embedded checkout mounts directly inside application

#### Shipping Flow
- Quote confirmation screen renders provisional quote data
- Offer review page consumes:
  - GET /api/quotes/:id
  - PUT /api/quotes/:id/accept
  - PUT /api/quotes/:id/reject

- Rejection survey page consumes:
  - POST /api/quotes/:id/survey

#### Admin Dashboard
- Quote management panel consumes:
  - GET /api/quotes
  - GET /api/quotes/:id
  - PUT /api/quotes/:id/offer

- Admin dashboard displays:
  - pending review requests
  - accepted offers
  - rejected offers
  - completed order lifecycle states

---

## Stripe Checkout

### Functionality
- Stripe checkout is embedded directly into the web application
- Checkout is initiated from two possible flows:

  - Pickup Flow:
    - immediate checkout after pickup selection

  - Shipping Flow:
    - checkout becomes available only after:
      - admin confirms shipping offer
      - client accepts confirmed quote

- Checkout session totals are calculated dynamically at session creation time
- Product base pricing remains immutable in products collection
- Final billed totals are derived from:
  - product price
  - confirmed shipping amount
  - applicable taxes
- Pickup flow always uses:
  - shipping amount = 0

- Stripe CLI is used for:
  - local webhook testing
  - local payment event simulation

- Stripe webhook becomes authoritative for payment completion
- Frontend success states are never treated as authoritative payment confirmation

### Successful Payment Lifecycle
- Stripe webhook validates successful payment event
- Backend validates:
  - quote ownership
  - quote/payment state
  - invoice integrity
- Backend upgrades provisional quote/invoice state into finalized invoice state
- Invoice metadata JSON is written to Firebase Storage
- PDF invoice is generated dynamically through PdfKit
- PDF invoice is emailed to:
  - client
  - admin
- Shipment lifecycle begins after successful payment confirmation
- Shipment record is created in Firestore
- Quote status updates to:
  - "COMPLETED"

### Invoice PDF Behavior
- Invoice PDFs are generated dynamically at send/retrieval time only
- PDFs are not permanently stored
- Invoice metadata JSON remains the authoritative source of truth
- PDFs may be regenerated at any time from stored metadata
- Invoice regeneration never recalculates pricing;
  regeneration strictly reflects stored invoice metadata

### Storage
- Firebase Storage:

  - invoices/{invoiceId}.json — authoritative invoice metadata records:
    - invoiceId: string
    - quoteId: string | null
    - productId: string
    - productName: string
    - clientEmail: string
    - clientName: string
    - destinationAddress:
      {
        street: string,
        city: string,
        state: string,
        zip: string,
        country: string
      }
    - productPrice: number
    - shippingAmount: number
    - billedSubtotal: number
    - taxAmount: number
    - billedTotal: number
    - pickupLabel: boolean
    - stripePaymentIntentId: string
    - stripeCheckoutSessionId: string
    - paymentStatus: string
    - createdAt: string

- Firebase Firestore:

  - shipments collection:
    - shipment lifecycle records created after successful checkout

  - quotes collection:
    - updated to "COMPLETED" after successful payment webhook

- Invoice metadata JSON remains authoritative for:
  - reporting
  - invoice regeneration
  - MRR calculations
  - shipment linkage
  - accounting retrieval

- Invoice PDFs are never stored permanently

### API Calls

#### Pickup Checkout
- POST /api/checkout/session/pickup — authenticated route;
  validates:
  - authenticated ownership
  - pickup eligibility
  - product availability
  computes:
  - product price
  - shipping = 0
  creates Stripe embedded checkout session,
  returns Stripe client secret

#### Shipping Checkout
- POST /api/checkout/session — authenticated route;
  validates:
  - quote ownership
  - quote status = "ACCEPTED"
  computes:
  - product price
  - confirmed shipping amount
  - taxes
  creates Stripe embedded checkout session,
  returns Stripe client secret

#### Stripe Webhook
- POST /api/checkout/webhook — Stripe webhook endpoint;
  validates Stripe signature,
  processes successful payment events,
  writes invoice metadata JSON,
  generates PDF invoice,
  emails:
  - client invoice confirmation
  - admin invoice confirmation
  creates shipment record,
  updates quote status:
  - "COMPLETED"

- Webhook processing is idempotent:
  - duplicate Stripe webhook events do not create duplicate invoices,
    shipments, or emails

### Frontend Connection

#### Pickup Checkout Flow
- Pickup confirmation page consumes:
  - POST /api/checkout/session/pickup
- Stripe embedded checkout widget mounts directly inside application

#### Shipping Checkout Flow
- Offer review acceptance flow consumes:
  - POST /api/checkout/session
- Stripe embedded checkout widget mounts directly inside application

#### Post-Payment Behavior
- Frontend success UI is informational only
- Backend webhook confirmation remains authoritative for:
  - invoice creation
  - shipment creation
  - payment finalization
  - email delivery
  - reporting state updates

- Frontend may poll or retrieve finalized invoice/shipment state after
  webhook completion

---

## MRR & Invoice Reporting

### Functionality
- Admin has access to revenue reporting derived entirely from finalized invoice metadata
- System provides:
  - monthly recurring revenue (MRR) breakdown
  - total revenue aggregation
  - invoice-level inspection and retrieval
  - invoice deletion and financial recalculation controls
- All reporting is derived from immutable invoice records generated after successful Stripe webhook confirmation
- Invoice retrieval does NOT recompute financial values; it strictly reflects stored metadata
- Invoice PDFs are regenerated on demand from metadata using PdfKit
- Admin can:
  - view full invoice history
  - filter invoices by time period
  - delete individual or multiple invoices
  - trigger recalculation of aggregated revenue reports

### Storage

- Firebase Storage:

  - invoices/{id}.json — authoritative invoice records:
    - invoiceId: string
    - productId: string
    - productName: string
    - clientEmail: string
    - clientName: string
    - destinationAddress:
      {
        street: string,
        city: string,
        state: string,
        zip: string,
        country: string
      }
    - productPrice: number
    - shippingAmount: number
    - taxAmount: number
    - billedTotal: number
    - pickupLabel: boolean
    - stripePaymentIntentId: string
    - createdAt: string

  - reports/monthly.json — aggregated monthly revenue snapshot:
    - { "YYYY-MM": number }

  - reports/total.json — global revenue aggregate:
    - totalRevenue: number

- Firebase Firestore (supporting references only):
  - quotes collection (linked for traceability)
  - shipments collection (linked for lifecycle tracking)

- Invoice JSON files are the single source of truth for financial reporting
- Aggregated reports are derived artifacts and can be recalculated at any time

### API Calls

- GET /api/reports/mrr — returns monthly revenue breakdown
  - reads from reports/monthly.json
  - groups revenue by invoice createdAt date

- GET /api/reports/invoices — returns list of all invoices
  - supports pagination and filtering (date range, status, client)

- GET /api/reports/invoices/:id — returns:
  - invoice metadata
  - regenerated PDF (generated on demand via PdfKit)
  - validates metadata integrity before rendering

- DELETE /api/reports/invoices/:id — deletes invoice record
  - removes invoices/{id}.json from Firebase Storage
  - triggers recalculation of:
    - reports/monthly.json
    - reports/total.json
  - ensures financial aggregates remain consistent

### Reporting Behavior

- Revenue calculations are strictly derived from stored invoice totals
- No recalculation of Stripe or product pricing occurs during reporting
- Deleting an invoice immediately impacts:
  - monthly revenue totals
  - global revenue totals
- Recalculation process:
  - reads all remaining invoice JSON files
  - rebuilds:
    - monthly aggregates
    - total revenue snapshot
  - writes updated report files to Firebase Storage

### Frontend Connection

#### Admin Dashboard — Revenue Overview
- GET /api/reports/mrr provides:
  - chart-ready monthly revenue data
  - used for MRR visualization panels

#### Admin Dashboard — Invoice List
- GET /api/reports/invoices provides:
  - invoice table data
  - filters (date, client, status)
  - pagination support

#### Admin Dashboard — Invoice Detail View
- GET /api/reports/invoices/:id provides:
  - full invoice metadata
  - downloadable/regenerated PDF invoice

#### Admin Dashboard — Invoice Deletion
- DELETE /api/reports/invoices/:id triggers:
  - invoice removal
  - revenue recalculation
  - UI refresh of reporting data

### System Integrity Rules
- Invoice metadata is immutable once created (except via admin deletion)
- Reports are derived artifacts and must always be recalculated from invoice source data
- Stripe is not queried during reporting; it is only used at transaction time
- PDF generation is stateless and depends only on stored invoice metadata

---

## Shipping (UPS — Mocked)

### Functionality

#### Pre-Checkout Shipping Quote Flow
- Client selects a product and views:
  - price
  - weight
  - dimensions
- System loads authenticated user's saved home/mailing address
  as the destination for shipping calculations (no manual entry required)
- Backend evaluates:
  - local pickup eligibility (based on admin-defined radius rules)
  - UPS delivery eligibility (mocked service simulation)
- System returns available fulfillment options:
  - LOCAL_PICKUP (if eligible)
  - UPS_DELIVERY (if eligible)

- If UPS_DELIVERY is available:
  - system generates mocked UPS rate options
  - client selects preferred shipping service level
  - client submits quote request
  - admin review is required before checkout is allowed

- If LOCAL_PICKUP is available:
  - client may proceed directly to checkout without admin approval

#### Admin Shipping Review Flow
- Admin receives shipping quote request
- Admin reviews:
  - destination address
  - product dimensions/weight
  - calculated mock UPS rates
- Admin may:
  - approve estimated shipping cost
  - adjust shipping cost manually
  - reject shipping option entirely (force pickup or cancel flow)
- Admin decision generates an offer:
  - updates quote status to "OFFER_SENT"
  - sends email to client with final decision

#### Shipment Lifecycle Tracking
- Once a tracking number is assigned:
  - shipment record is created in Firestore
  - shipment is considered active
- System tracks shipment lifecycle states:
  - LABEL_CREATED
  - IN_TRANSIT
  - OUT_FOR_DELIVERY
  - DELIVERED

- Each status update:
  - updates Firestore shipment record
  - triggers email notification to client (only on state change)
  - updates real-time status view (via frontend polling or refresh-based sync)

- Client can view shipment status at any time via dedicated tracking page

### Storage

- Firebase Firestore:

  - shipping_quotes collection:
    - id: string
    - productId: string
    - productName: string
    - weight: number
    - width: number
    - height: number
    - length: number
    - destinationAddress:
      {
        street: string,
        city: string,
        state: string,
        zip: string,
        country: string
      }
    - estimatedShippingCost: number
    - serviceOptions:
      [
        {
          serviceLevel: string,
          estimatedCost: number,
          estimatedTransitDays: number
        }
      ]
    - status: string ("PENDING_REVIEW" | "QUOTED" | "OFFER_SENT" | "REJECTED")
    - clientEmail: string
    - createdAt: string
    - adminNotes: string | null

  - shipments collection:
    - trackingNumber: string
    - invoiceId: string
    - quoteId: string | null
    - clientEmail: string
    - status: string
      ("LABEL_CREATED" | "IN_TRANSIT" | "OUT_FOR_DELIVERY" | "DELIVERED")
    - estimatedDelivery: string
    - activity:
      [
        {
          location: {
            city: string,
            state: string,
            country: string
          },
          description: string,
          timestamp: string
        }
      ]
    - lastNotifiedStatus: string

- UPS is fully mocked:
  - request/response shapes intentionally mirror real UPS API structures
  - allows future migration to real UPS integration without schema changes
- No real carrier API credentials are required or used

### Mock UPS Object Shapes

#### Rate Request
```json id="m4q2nv"
{
  "package": {
    "weight": 10,
    "dimensions": {
      "width": 20,
      "height": 15,
      "length": 30
    }
  },
  "destination": {
    "street": "string",
    "city": "string",
    "state": "string",
    "zip": "string",
    "country": "string"
  }
}

---

## Status Key
- [ ] Not started
- [~] In progress
- [x] Done + tested
- [!] Blocked