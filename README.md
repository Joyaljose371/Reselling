# Haat Bazaar: stationery and bookish gifts on Firebase

A cash-on-delivery stationery shop for magnetic bookmarks, books, diaries, cute notebooks, pens, art and craft supplies, and gift sets. Shoppers browse, add to cart, verify a delivery address and place orders. You list products with photos, set your margin, manage orders and coupons from the Seller panel. Data lives in Firebase (Authentication + Firestore) and the site is served by Firebase Hosting. There is no build step.

```
public/index.html          the whole app
public/firebase-config.js  your Firebase keys (the only file you edit)
firestore.rules            who can read and write what
firebase.json              hosting + rules deployment
```

## Try it first (no Firebase needed)

While `firebase-config.js` still has `YOUR_API_KEY`, the store runs in **demo mode** and saves everything in the browser. A banner says so. Serve the folder over http (module scripts do not run from `file://`):

```
npx serve public
```

## Go live in 6 steps

1. **Create a project** at https://console.firebase.google.com. Add a **Web app** (the `</>` icon) and copy its config into `public/firebase-config.js`.
2. **Authentication**: Build > Authentication > Get started > enable **Email/Password**.
3. **Firestore**: Build > Firestore Database > Create database, production mode. A region near your customers (for India, `asia-south1` Mumbai) keeps it fast.
4. **Install the CLI and log in**: `npm i -g firebase-tools` then `firebase login`.
5. **Link and deploy** from this folder:
   ```
   firebase use --add          # pick your project
   firebase deploy --only firestore:rules,hosting
   ```
6. **Install the Trigger Email extension** from Firebase Extensions. Configure its watched collection as `mail` and configure Gmail SMTP with a Gmail App Password (requires 2-Step Verification; never put the password in this repository). The extension needs a billing-enabled Firebase project. The app queues email for order placed, confirmed and delivered.
7. **Open the site and create your own account first.** On a fresh store the first account becomes the owner and the **Seller panel** button appears in the header. Then use Seller panel > Overview > **Load sample products** to see the store working, and **Remove sample products** when you are ready to list your own.

The sample catalog is stationery-themed. Changing the site files does not change products already saved in Firestore; remove old sample products from Seller panel > Overview, or edit/delete old listings in Seller panel > Products, then add your stationery inventory.

## Owner account

By default the first account created becomes the owner (`claimOwnerOnFirstSignup = true`). Anyone who signs up before you would win that claim, so create your account right after deploying.

To set the owner by hand instead: set `claimOwnerOnFirstSignup = false`, create your account on the site, then in the Firestore console create document `config/store` with a field `ownerUid` (string) holding your user's UID (Authentication > Users). To transfer ownership later, edit that field in the console.

## What is stored

| Collection | Contents | Who can write |
|---|---|---|
| `products` | name, prices, stock, sizes, photos, rating totals | owner (shoppers can only lower stock when ordering and add one rating when reviewing) |
| `products/{id}/reviews/{uid}` | one review per shopper | that shopper |
| `coupons` | codes and rules | owner |
| `orders` | items, address, totals, status | shopper creates; owner updates status; shopper can only request a cancellation |
| `users/{uid}` | name, phone, verified addresses | that shopper |
| `config/store` | owner UID, shipping settings, customer support contacts | owner settings |
| `mail` | transactional email requests for the Trigger Email extension | order owner/customer via constrained rules; extension processes delivery |

Cart and wishlist stay in each shopper's browser.

## Delivery, returns, and customer emails

- In Seller panel > Overview, set one delivery charge per order, a free-delivery order minimum (set to `0` to disable the minimum), and customer-facing support email/WhatsApp details. Free delivery applies if the cart reaches the minimum, every item in the cart is marked **Free delivery**, or a free-shipping coupon applies.
- Choose **This product qualifies for free delivery** while adding or editing an item. In a mixed cart, every item must have this flag for product-based free delivery to apply.
- A customer can request a return within 7 days of an order being marked Delivered. After the request, the seller advances the order through **Return accepted**, **Return processed**, and **Refund initiated**, in that order; the customer sees these milestones in order tracking. “Refund initiated” records the seller's action but does not issue a payment automatically. For older delivered orders without a delivery timestamp, use **Record delivery date** on the seller order card; this starts the seven-day return window from when it is recorded.
- PIN lookups use the `api.postalpincode.in` directory to show matching post offices, district and state. The customer selects the matching locality. This identifies postal delivery areas, not a house or street address, and sends the entered PIN to that lookup service.
- The Trigger Email extension sends email from the `mail` collection. Deploy the included Firestore rules after installing it. Emails are queued when an order is placed and when the seller moves it to Confirmed or Delivered; test SMTP delivery from the extension before going live.

## How the main flows work

- **Cash on delivery only.** Orders are created as unpaid. Marking an order **Delivered** records it as paid in cash.
- **Cancellations.** A shopper asks to cancel while the order is Placed or Confirmed. You approve with **Cancel and restock** on the order, which returns the stock.
- **Stock** is checked and reduced inside a Firestore transaction when an order is placed, so two shoppers cannot buy the last item.
- **Address verification** checks name, mobile number, PIN code range, PIN-to-state match and address completeness. It runs in the browser from a built-in PIN prefix table, not a postal database, so a few border-area PINs can be rejected wrongly.

## Limits to know about

- **Prices are calculated in the browser.** Firestore rules cannot recompute a cart, so a technical user could send an order with lower prices. Orders show a **Price check** badge when an item price differs from the current catalogue price. Check these before dispatching. Stopping this fully needs a Cloud Function, which requires the paid Blaze plan.
- **Coupon rules** (minimum order, first order only) are also enforced in the browser.
- **Photos** are stored inside the product document (Firestore allows about 1 MB per document), so each product takes up to 4 photos that are shrunk to 640 px. This keeps you on the free Spark plan. If you later move to Blaze, Firebase Storage is the better place for photos.
- Every visitor downloads the product list, so keep the catalogue and photo sizes modest.
- This was tested end to end in demo mode (sign-up, cart, coupons, address verification, order, status change, review, listing a product). The Firebase connection code could not be run against a live project from where it was written, so if something fails after deploy, open the browser console and read the first error. Common causes: Email/Password sign-in not enabled, rules not deployed, or a wrong value in `firebase-config.js`.
