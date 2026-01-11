# Stripe Payment System

A complete, production-ready Stripe payment integration built with Next.js 14 (App Router), TypeScript, Supabase, and Stripe Checkout.

## Features

- ✅ Stripe Checkout integration
- ✅ Webhook handling for payment events
- ✅ Supabase database integration
- ✅ TypeScript throughout
- ✅ Secure webhook signature verification
- ✅ Success and cancel pages
- ✅ Modern, responsive UI

## Prerequisites

- Node.js 18+ installed
- Stripe account with test API keys
- Supabase account with database set up
- Stripe CLI (for local webhook testing)

## Setup Instructions

### 1. Install Dependencies

```bash
npm install
```

### 2. Set Up Supabase Database

Create the `orders` table in your Supabase database:

```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_name TEXT NOT NULL,
  customer_email TEXT NOT NULL,
  amount INTEGER NOT NULL,
  currency TEXT DEFAULT 'usd',
  stripe_session_id TEXT UNIQUE,
  stripe_payment_intent TEXT,
  payment_status TEXT DEFAULT 'pending' CHECK (payment_status IN ('pending', 'paid', 'failed')),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Create an index for faster lookups
CREATE INDEX idx_orders_session_id ON orders(stripe_session_id);
CREATE INDEX idx_orders_payment_intent ON orders(stripe_payment_intent);
```

### 3. Environment Variables

Create the `.env.local` file by running:

```bash
./setup-env.sh
```

Or manually create `.env.local` with the following content:

```
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
STRIPE_PUBLISHABLE_KEY=your_stripe_publishable_key
STRIPE_SECRET_KEY=your_stripe_secret_key
STRIPE_WEBHOOK_SECRET=your_stripe_webhook_secret
```

### 4. Run the Development Server

```bash
npm run dev
```

The application will be available at `http://localhost:3000`

### 5. Set Up Stripe Webhooks (Local Development)

For local development, use Stripe CLI to forward webhooks:

1. Install Stripe CLI: https://stripe.com/docs/stripe-cli

2. Login to Stripe:
```bash
stripe login
```

3. Forward webhooks to your local server:
```bash
stripe listen --forward-to localhost:3000/api/stripe-webhook
```

4. Copy the webhook signing secret that appears (starts with `whsec_`) and update your `.env.local` file with `STRIPE_WEBHOOK_SECRET`.

### 6. Production Webhook Setup

For production, you need to:

1. Deploy your application
2. Add a webhook endpoint in Stripe Dashboard: `https://yourdomain.com/api/stripe-webhook`
3. Select events to listen for:
   - `checkout.session.completed`
   - `payment_intent.succeeded`
   - `payment_intent.payment_failed`
4. Copy the webhook signing secret and add it to your production environment variables

## Project Structure

```
├── app/
│   ├── api/
│   │   ├── checkout/
│   │   │   └── route.ts          # Create Stripe Checkout session
│   │   ├── stripe-webhook/
│   │   │   └── route.ts          # Handle Stripe webhooks
│   │   └── order-status/
│   │       └── route.ts          # Get order status
│   ├── success/
│   │   └── page.tsx              # Success page
│   ├── cancel/
│   │   └── page.tsx              # Cancel page
│   ├── layout.tsx                # Root layout
│   ├── page.tsx                  # Checkout form
│   └── globals.css               # Global styles
├── lib/
│   ├── supabase.ts              # Supabase client
│   └── stripe.ts                 # Stripe client
├── types/
│   └── index.ts                  # TypeScript types
├── .env.local                    # Environment variables
├── next.config.js                # Next.js configuration
├── package.json                  # Dependencies
└── tsconfig.json                 # TypeScript configuration
```

## How It Works

1. **Checkout Flow**:
   - User fills out the checkout form (name, email, amount)
   - Form submits to `/api/checkout`
   - API creates a Stripe Checkout session and saves order to Supabase
   - User is redirected to Stripe Checkout page

2. **Webhook Processing**:
   - Stripe sends webhook events to `/api/stripe-webhook`
   - Webhook signature is verified for security
   - Order status is updated in Supabase based on event type:
     - `checkout.session.completed` → status: `paid`
     - `payment_intent.succeeded` → status: `paid`
     - `payment_intent.payment_failed` → status: `failed`

3. **Success/Cancel Pages**:
   - After payment, user is redirected to success or cancel page
   - Success page shows order details

## Testing

1. Use Stripe test cards: https://stripe.com/docs/testing
2. Recommended test card: `4242 4242 4242 4242`
3. Use any future expiry date, any CVC, and any ZIP code

## Security Notes

- Webhook signatures are verified using the webhook secret
- Environment variables are used for all sensitive data
- Never commit `.env.local` to version control
- Use HTTPS in production

## Troubleshooting

### Payment Status Stays "Pending" After Successful Payment

**This is the most common issue!** The payment succeeds but the status doesn't update because webhooks aren't working.

**Root Causes:**
1. **Stripe CLI not running** - Webhooks need to be forwarded locally
2. **Webhook secret mismatch** - The secret in `.env.local` doesn't match Stripe CLI
3. **Webhook endpoint not receiving events** - Network/firewall issues

**Solutions:**

1. **Check if Stripe CLI is running:**
   ```bash
   # In a separate terminal, run:
   stripe listen --forward-to localhost:3000/api/stripe-webhook
   ```
   You should see: `Ready! Your webhook signing secret is whsec_xxx`

2. **Update webhook secret:**
   - Copy the `whsec_xxx` secret from the Stripe CLI output
   - Update `STRIPE_WEBHOOK_SECRET` in `.env.local`
   - Restart your Next.js dev server

3. **Check webhook logs:**
   - Look at your Next.js server console for webhook events
   - You should see: `📥 Received webhook event: checkout.session.completed`
   - If you don't see this, webhooks aren't reaching your server

4. **Manual fix for existing orders:**
   - The success page now automatically tries to update pending orders
   - Or manually call: `POST /api/manual-update-order?session_id=cs_test_xxx`
   - This retrieves the session from Stripe and updates the order

5. **Test webhook manually:**
   ```bash
   # Trigger a test webhook
   stripe trigger checkout.session.completed
   ```

### Webhook not working locally?
- Make sure Stripe CLI is running and forwarding to `localhost:3000/api/stripe-webhook`
- Check that `STRIPE_WEBHOOK_SECRET` matches the secret from `stripe listen`
- Verify the webhook endpoint is accessible
- Check server console for webhook event logs

### Database errors?
- Ensure the `orders` table exists in Supabase
- Check that your Supabase credentials are correct
- Verify RLS (Row Level Security) policies allow inserts/updates

### Checkout session not creating?
- Verify Stripe API keys are correct
- Check that environment variables are loaded (restart dev server after changing `.env.local`)
- Review server logs for detailed error messages

## License

MIT
