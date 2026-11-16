# Kenang Bersama — Production Starter

Next.js + Supabase + Midtrans QRIS. This is a production-oriented starter, not a live deployed service. It contains the booking frontend, admin login/dashboard, database migration, approval flow, QRIS creation endpoint, and Midtrans notification verification.

## Included decisions
- Admin approval required before an invoice is created.
- DP is fixed at 30% of the agreed total price.
- One admin role; no public admin sign-up.
- Packages seeded: Intimate Rp250.000/3h, Signature Rp300.000/4h, Premium Rp350.000/5h.
- Overtime reference: Rp50.000/hour. Transport outside Malang Raya is quoted by admin.
- A booking reserves its date when admin approves it; only one active booking per event date.

## 1. Create Supabase project
1. Create a project at https://supabase.com/dashboard.
2. Open SQL Editor and run `supabase/schema.sql`.
3. Copy Project URL, publishable key, and service role key from Project Settings > API.
4. In Authentication > Users, create the single admin user yourself. Disable public sign-ups in Auth settings.
5. Add a row to `public.profiles` for that Auth user:
   ```sql
   insert into public.profiles(id, full_name, role)
   values ('AUTH_USER_UUID', 'Admin Kenang Bersama', 'admin');
   ```
   Replace the UUID with the user ID from Authentication > Users. Never expose the service role key.

## 2. Configure Midtrans
1. Register/activate a Midtrans merchant account and enable QRIS.
2. Start with Sandbox credentials and test payments.
3. Set the notification URL in Midtrans Dashboard to:
   `https://YOUR_DEPLOYMENT_HOST/api/midtrans/notification`
4. Set `MIDTRANS_SERVER_KEY` in the hosting provider's server-side environment variables.
5. Keep `MIDTRANS_IS_PRODUCTION=false` until sandbox verification is complete. Switch to `true` only after merchant production activation and successful end-to-end tests.
6. Midtrans notification signature is verified using SHA-512 of `order_id + status_code + gross_amount + ServerKey`. A booking becomes confirmed only after a valid paid notification (`settlement`, or accepted `capture`).

## 3. Local setup
```bash
npm install
cp .env.example .env.local
# Fill the variables in .env.local
npm run dev
```
Open http://localhost:3000. Sign in at `/login`.

## 4. Deploy to Vercel
1. Push this folder to a private GitHub repository.
2. Import the repository into Vercel.
3. Add all variables from `.env.example` in Vercel Project Settings > Environment Variables. Never prefix service-role or Midtrans secrets with `NEXT_PUBLIC_`.
4. Deploy a Preview first; test login, lead submission, admin approval, QRIS, webhook, and date collision.
5. Deploy Production only after tests pass.
6. Add a domain such as `app.kenangbersama.com` in Vercel Project Settings > Domains and apply the DNS records Vercel provides at your registrar.
7. Update `NEXT_PUBLIC_SITE_URL` and Midtrans notification URL to the production host, then redeploy.

## 5. Required production hardening before launch
- Add CAPTCHA/rate limiting to public lead submission.
- Configure custom SMTP and email confirmation / recovery policy in Supabase Auth.
- Enable MFA for admin if available; use a strong unique password.
- Set database backups and test restoration.
- Add privacy notice, data retention policy, terms, cancellation/refund policy.
- Add a manual payment reconciliation view and alerting for failed webhooks.
- Consider expiring unpaid DP reservations after a defined hold period. This starter keeps the date reserved while `awaiting_dp`; agree a hold window before launch.
- Confirm the exact inclusions per package and the transport pricing rule.
- Run a security review and payment sandbox tests before accepting real customer money.

## Important implementation notes
- Public form submission uses a server-only service role client; the browser cannot read leads/payments.
- Admin endpoints check Supabase Auth and `profiles.role='admin'`.
- `approve_booking` is an atomic database function with a date-level advisory lock to prevent simultaneous double booking.
- QRIS payment is created server-side through Midtrans Core API. The webhook signature is checked before payment state changes.
- The QR image URL returned by Midtrans is exposed only to the authenticated admin dashboard in this starter. A customer invoice/QR display page and WhatsApp notification are the next recommended implementation step.
- The admin can enter the agreed total amount; the DP is calculated server-side from the booking total in the database function.
