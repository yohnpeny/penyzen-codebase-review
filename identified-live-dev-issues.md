# Identified Live Development Environment Issues & Solutions

**Document Version:** 1.0  
**Environment:** `dev` branch (Live Dev Environment)  
**Date:** November 12, 2026

---

## Executive Summary

During testing of the live development environment on the `dev` branch, multiple issues were identified across authentication, payout readiness gating, and frontend user state management:

1. **MFA / Verification Code Sender Email Failure**: Cognito SES identity mismatch in dev causing verification email delivery failures.
2. **Missing MFA Code Retry / Resend Capability**: When an MFA code fails to reach the user, there is no resend or retry button on the code prompt, forcing the user to return to the sign-in page to start over.
3. **Campaign List Filtering vs. Campaign Details Passing**: Payout gating divergence between list and details views, combined with Stripe Connect webhook synchronization lag.
4. **Frontend User Role Resolution Bug**: Double `/api/proxy` prefix and envelope double-unwrapping in `use-auth-state.ts`, causing signed-in users to resolve with a `null` role.
5. **Action Buttons Missing Confirmation Dialogs**: Destructive actions like removing an uploaded profile photo execute immediately without a modal dialog; these UI/UX experiences have already been resolved on the `refactor-frontend` branch.
6. **Donate Panel Integration & Component State**: Residual merge artifacts on campaign details.

Each issue is accompanied by an architectural root-cause analysis, impacts, and concrete preferred solutions.

---

## Issue 1: MFA Code Sender Email Is Not Working

### 1. Description & Symptoms

- Users attempting to sign in using **Two-Step Verification (Email OTP)**, reset forgotten passwords, or confirm account registration do not receive verification emails in the live development environment.
- Setting up Email MFA in the user dashboard requires phone verification first (`/v1/phone/send-otp`). When SMS origination is not configured (or fails in dev), the OTP fallback email also fails to arrive.

### 2. Root Cause Analysis

In `auth-stack.ts` (lines 79–88), the Cognito User Pool email configuration is set as follows:

```typescript
email: cognito.UserPoolEmail.withSES({
  fromEmail: 'support@penyzen.com',
  fromName: 'Penyzen',
  replyTo: 'support@penyzen.com',
  ...(props.envName === 'prod' && { sesVerifiedDomain: 'penyzen.com' }),
}),
```

- **Domain verification split**: In AWS SES, the production account owns and verifies `penyzen.com`. The development account owns and verifies `dev.penyzen.com` (as correctly documented and configured in `api-stack.ts` lines 168–170):
  ```typescript
  // Each account can only verify its own domain (prod owns penyzen.com,
  // dev owns dev.penyzen.com), so the FROM address is env-specific.
  SES_FROM_EMAIL: props.envName === 'prod' ? 'noreply@penyzen.com' : 'noreply@dev.penyzen.com',
  ```
- **Synthesis failure in dev**: Because `sesVerifiedDomain: 'penyzen.com'` is applied **only** when `props.envName === 'prod'`, CDK synthesizes the dev CloudFormation template with:
  ```json
  "SourceArn": "arn:aws:ses:us-east-1:<dev-account>:identity/support@penyzen.com"
  ```
  In the dev account, `support@penyzen.com` is unverified (only `dev.penyzen.com` is verified). Consequently, Amazon SES rejects email sends triggered by Cognito for MFA OTP and password resets.
- **User-service phone OTP fallback**: In `phone.service.ts` (line 11), `FROM_EMAIL` defaults to `process.env['SES_FROM_EMAIL'] ?? 'noreply@penyzen.com'`. If the environment variable is omitted in dev, fallback emails also attempt sending from `@penyzen.com` instead of `@dev.penyzen.com`.

### 3. Preferred Solution

#### A. Make Cognito UserPool SES Configuration Environment-Aware

In `auth-stack.ts`:

```typescript
const isProd = props.envName === 'prod';
const emailDomain = isProd ? 'penyzen.com' : 'dev.penyzen.com';

this.userPool = new cognito.UserPool(this, 'UserPool', {
  // ... other properties
  email: cognito.UserPoolEmail.withSES({
    fromEmail: `support@${emailDomain}`,
    fromName: 'Penyzen',
    replyTo: `support@${emailDomain}`,
    sesVerifiedDomain: emailDomain,
  }),
  // ...
});
```

#### B. Ensure Service Fallbacks Default by Environment

In `phone.service.ts` and `sesClient.ts`:

```typescript
const isProd =
  process.env['NODE_ENV'] === 'production' && !process.env['NEXT_PUBLIC_API_URL']?.includes('dev');
const defaultFrom = isProd ? 'noreply@penyzen.com' : 'noreply@dev.penyzen.com';
const FROM_EMAIL = process.env['SES_FROM_EMAIL'] ?? defaultFrom;
```

---

## Issue 2: Missing MFA Code Retry / Resend Option on Login Page

### 1. Description & Symptoms

- When a user enters their credentials and reaches the 6-digit MFA code challenge, if the email OTP does not arrive (due to email delays, spam filtering, or sender errors), there is **no way to request a new code or retry sending**.
- The user is completely trapped on the code input screen. The only workaround is to reload the browser or navigate back to the sign-in page to re-enter their username and password from scratch.

### 2. Root Cause Analysis

In `page.tsx` (the web app sign-in page `login/page.tsx`), the code submission form only renders the input field and a submit button:

```tsx
<form onSubmit={onSubmitCode} className="mt-6 space-y-4">
  <p className="text-sm text-slate-700">
    {codeStep.kind === 'TOTP'
      ? 'Enter the 6-digit code from your authenticator app.'
      : `Enter the code we emailed${codeStep.sentTo ? ` to ${codeStep.sentTo}` : ' you'}.`}
  </p>
  <Field
    label="Code"
    type="text"
    value={code}
    onChange={setCode}
    required
    autoComplete="one-time-code"
  />
  {error && <p className="rounded-md bg-red-50 px-3 py-2 text-sm text-red-700">{error}</p>}
  <button type="submit" disabled={submitting} className="...">
    {submitting ? 'Checking…' : 'Continue'}
  </button>
</form>
```

- There is no "Resend code" button or link for the `EMAIL` factor.
- There is no option to switch back to factor selection (e.g. choose Authenticator app instead of Email) without discarding the session cookie and starting over.
- In `auth-flows.ts`, the `PendingChallenge` cookie contains the active session valid for 3 minutes, but the server exposes no endpoint to re-dispatch the challenge response without resetting the session.

### 3. Preferred Solution

1. **Add Resend & Factor Switch Controls in `page.tsx` (`login/page.tsx`)**:
   Add a secondary action section below the submit button:
   - For email OTPs: A "Didn't receive a code? Resend code" button equipped with a 60-second cooldown timer.
   - A "Choose a different sign-in method" button to switch between Authenticator app (`TOTP`) and Email (`EMAIL`).
   - A "Back to sign in" button that safely clears the challenge cookie and returns the user to the email/password form cleanly without unexpected session state.

```tsx
{
  codeStep.kind === 'EMAIL' && (
    <div className="flex items-center justify-between text-xs text-slate-500 pt-2">
      <button
        type="button"
        disabled={resendCooldown > 0 || submitting}
        onClick={handleResendMfaCode}
        className="text-brand-600 hover:underline disabled:opacity-50"
      >
        {resendCooldown > 0 ? `Resend code in ${resendCooldown}s` : 'Resend code'}
      </button>
      <button
        type="button"
        onClick={() => setCodeStep(null)}
        className="text-slate-500 hover:text-slate-700 hover:underline"
      >
        Back to sign in
      </button>
    </div>
  );
}
```

---

## Issue 3: Campaign List Filtering vs. Campaign Details Passing

### 1. Description & Symptoms

- **Campaign list (`/campaigns`)**: When an organizer creates and publishes a campaign before setting up their Stripe payout account, the campaign does **not** appear in search results or category lists.
- **Campaign details (`/campaigns/:id`)**: If someone navigates directly to the campaign page via link or slug, the campaign loads completely ("passes"), displaying an amber warning banner: _"Not accepting donations yet: The organizer is still setting up their payout account."_
- **After payout setup**: When an organizer finishes setting up their Stripe Connect payout account, the campaign details page may still report that the organizer has not set up payouts, and the campaign list continues to filter it out until webhooks arrive or dashboard is visited.

### 2. Root Cause Analysis

#### A. Divergent Query Predicates

1. In `campaign.service.ts` (`listCampaigns`):
   ```typescript
   const where = {
     status: 'PUBLISHED' as const,
     ...(category && { category: category as never }),
     AND: [recipientReadyWhere, ...],
   };
   ```
   `recipientReadyWhere` excludes any campaign whose payout account lacks `stripeAccountId`, `stripeChargesEnabled`, or `stripePayoutsEnabled`.
2. In `campaign.service.ts` (`getCampaign`):
   ```typescript
   const campaign = await prisma.campaign.findFirst({
     where: { OR: [{ id }, { slug: id }], status: { in: [...PUBLIC_CAMPAIGN_STATUSES] } },
     select: { ...PUBLIC_CAMPAIGN_SELECT, ...CAMPAIGN_RECIPIENT_SELECT },
   });
   ```
   `getCampaign` does **not** include `recipientReadyWhere`.

#### B. Architectural Context & The Synchronization Gap

Under Product Decision **D4** (commit `dbef0bf`), an organizer is permitted to publish a campaign before Stripe Connect verification completes so they can preview and share the link.
However:

- **Missing Webhooks in Dev**: Stripe Connect triggers `account.updated` when payout verification completes. In the live dev environment, if the Stripe webhook endpoint is unreachable or delayed, the Postgres columns `stripeChargesEnabled` and `stripePayoutsEnabled` remain `false`.
- **Sync Discrepancy**: `connect.service.ts` (`getConnectStatus`) updates the database flags **only** when the user opens `/dashboard/payouts`. The public campaign endpoint `getCampaign` only reads Postgres columns and never queries Stripe or triggers a sync.

### 3. Preferred Solutions

There are two strategic approaches depending on business requirements:

#### Approach A (Recommended): Maintain Link Preview (D4) with Auto-Synchronization

Keep campaigns accessible by link before payout setup, but eliminate customer confusion and stale state:

1. **Live Stripe Verification on Campaign Details**:
   When `acceptingDonations` evaluates to `false` in `getCampaign`, check if `stripeAccountId` exists. If so, query or refresh account readiness (or trigger an asynchronous sync), ensuring that an organizer who just finished Stripe onboarding immediately sees their campaign accepting donations.
2. **Clearer Banner Distinction**:
   - For donors: _"This fundraiser is published and preparing to accept donations. Please check back shortly or bookmark this page."_
   - For the organizer (when signed in): _"Action Required: Complete your payout account in the dashboard to enable donations."_

#### Approach B: Strict Public Gating (Parity with Campaign List)

If campaigns without ready payouts should not be visible to the public under any circumstances:

1. In `campaign.service.ts`:
   Add `AND: [recipientReadyWhere]` to public `getCampaign` queries:
   ```typescript
   const campaign = await prisma.campaign.findFirst({
     where: {
       OR: [{ id }, { slug: id }],
       status: { in: [...PUBLIC_CAMPAIGN_STATUSES] },
       AND: [recipientReadyWhere],
     },
     select: { ...PUBLIC_CAMPAIGN_SELECT, ...CAMPAIGN_RECIPIENT_SELECT },
   });
   if (!campaign) throw new NotFoundError('Campaign', id);
   ```
2. Organizers can still preview their draft and unpublished campaigns through `/dashboard/campaigns/:id` or `/campaigns/:id/edit`, but public visitors receive `404 Not Found` until payouts are operational.

---

## Issue 4: Broken User Role Resolution in Frontend (`use-auth-state.ts`)

### 1. Description & Symptoms

- On the `dev` branch, signed-in users have their `role` set to `null` across the web application.
- Navigation menus, admin tabs, and organizer dashboard links disappear or fail to render because `useAuthState` cannot determine whether the user is a `DONOR`, `ORGANIZER`, or `ADMIN`.

### 2. Root Cause Analysis

In commit `99a278c` ("feat(web): use apiFetch in use-auth-state instead of raw fetch"), `fetchRole` in `use-auth-state.ts` was modified to:

```typescript
// use-auth-state.ts
async function fetchRole(_url: string, _init: RequestInit): Promise<Role> {
  try {
    const res = await apiFetch<{ data?: { role?: Role } }>('/api/proxy/v1/users/me', {
      auth: true,
    });
    if (!res?.data) return null;
    return res.data.role ?? null;
  } catch {
    return null;
  }
}
```

Two critical bugs exist in this implementation:

1. **Double Path Prefix**: In server-auth mode (the default), `apiFetch` in `api.ts` automatically prepends `/api/proxy`:
   ```typescript
   const url = `/api/proxy${path}`;
   ```
   Passing `'/api/proxy/v1/users/me'` results in a request to `'/api/proxy/api/proxy/v1/users/me'`, which returns `404 Not Found`.
2. **Envelope Double-Unwrap**: `apiFetch` already extracts the envelope payload in `readEnvelope` (`api.ts`):
   ```typescript
   return (parsed as ApiSuccess<T>).data;
   ```
   Therefore, `apiFetch` returns the `User` object directly (`{ id: string, role: Role, ... }`), **not** `{ data: User }`. Checking `res.data` evaluates to `undefined`, causing `fetchRole` to always return `null`.

### 3. Preferred Solution

Update `use-auth-state.ts` to pass the clean API path and directly read the user object:

```typescript
// use-auth-state.ts
async function fetchRole(_url: string, _init: RequestInit): Promise<Role> {
  try {
    const user = await apiFetch<{ role?: Role }>('/v1/users/me', { auth: true });
    return user?.role ?? null;
  } catch {
    return null;
  }
}
```

---

## Issue 5: Destructive Actions Missing Confirmation Dialogs

### 1. Description & Symptoms

- On `dev`, clicking destructive action buttons triggers immediate mutations without warning or confirmation dialogs.
- For example, in `profile-form.tsx`, clicking the **"Remove"** button next to an uploaded profile image immediately deletes the avatar from S3 and updates Postgres:
  ```tsx
  <button
    type="button"
    onClick={() => void removePhoto()}
    className="text-xs text-red-600 hover:underline"
  >
    Remove
  </button>
  ```
- Similar missing confirmation dialogs exist on other management buttons (cancelling member invites, removing bank details, etc.).

### 2. Status & Branch Parity Context

> **Resolved on `refactor-frontend` branch**:
> Every UI/UX interaction and missing confirmation dialog identified on `dev` has already been comprehensively addressed on the `yohnpeny/refactor-frontend` branch.
>
> That branch introduces:
>
> - Accessible modal confirmation dialogs for destructive actions (profile image removal, campaign deletion, member revocation).
> - Polished visual feedback states, loading spinners, and error alerts.
> - Standardized design-system components replacing ad-hoc inline action handlers.

### 3. Preferred Solution

Merge the UI/UX components and standardized modal confirmation flows from the `yohnpeny/refactor-frontend` branch into `dev`:

1. Use the unified `<ConfirmDialog>` component for destructive operations (such as profile image deletion).
2. Ensure consistent focus-trapping and keyboard accessibility (`Escape` to dismiss, `Enter` to confirm).

---

## Issue 6: Donate Panel State and Merge Residue

### 1. Description & Symptoms

- In commit `77772eb`, the `DonatePanel` import and component were replaced with a placeholder comment (`{/* Donate panel – modular slice now handles donations */}`) in `page.tsx` (`campaigns/[id]/page.tsx`).
- In commit `f518064`, the component was reintroduced to support donations lists. However, care must be taken to ensure that both the `DonatePanel` and the public donations section (`DonationsList`) handle empty states and closed donation states without UI regressions.

### 2. Preferred Solution

Ensure `page.tsx` (`campaigns/[id]/page.tsx`) cleanly renders `DonatePanel` inside a `<Suspense>` boundary with full prop coverage:

```tsx
<Suspense fallback={<div className="mt-5 h-64 animate-pulse rounded-2xl bg-slate-100" />}>
  <DonatePanel
    campaignId={campaign.id}
    campaignTitle={campaign.title}
    organization={campaign.organization}
    acceptingDonations={campaign.acceptingDonations}
    fundsGoTo={campaign.fundsGoTo}
    endsAt={campaign.endsAt}
  />
</Suspense>
```

---

## Remediation Roadmap & Action Items

| Priority | Component            | File                          | Action                                                                             |
| :------- | :------------------- | :---------------------------- | :--------------------------------------------------------------------------------- |
| **P0**   | **Web App**          | `use-auth-state.ts`           | Fix path `/v1/users/me` and unwrap `user?.role`.                                   |
| **P0**   | **Infra Auth**       | `auth-stack.ts`               | Set `fromEmail` and `sesVerifiedDomain` to `dev.penyzen.com` in dev.               |
| **P1**   | **Web App**          | `page.tsx` (`login/page.tsx`) | Add MFA code resend button with cooldown and back-to-login navigation.             |
| **P1**   | **User Service**     | `phone.service.ts`            | Ensure OTP fallback sender matches `SES_FROM_EMAIL`.                               |
| **P1**   | **Campaign Service** | `campaign.service.ts`         | Align `getCampaign` and `listCampaigns` payout gating policy (Approach A or B).    |
| **P1**   | **Web UI/UX**        | `profile-form.tsx`            | Integrate modal confirmation dialogs from `refactor-frontend` branch.              |
| **P2**   | **Payment Service**  | `connect.service.ts`          | Ensure Stripe webhook listener or refresh endpoints maintain database flag parity. |

---

## Verification Plan

### 1. Automated Verification

```powershell
# 1. Typecheck all workspaces
npm run typecheck

# 2. Run unit and integration tests for modified components
npx vitest run infra/test/auth.test.ts
npx vitest run apps/web/src/lib/auth-client.test.ts
npx vitest run services/campaign-service/src/services/campaign.listCampaigns.test.ts
npx vitest run services/campaign-service/src/services/campaign.acceptingDonations.test.ts

# 3. Run full test suite
npm test
```

### 2. Manual Verification on Live Dev

1. **Email MFA & Resend**:
   - Turn on Two-Step Verification using Email OTP in the dashboard.
   - Verify receipt of the verification code sent from `support@dev.penyzen.com`.
   - On the sign-in code entry screen, click "Resend code" and confirm a new code is delivered with proper cooldown timer.
2. **Confirmation Dialogs**:
   - In profile settings, click "Remove" on an uploaded profile picture.
   - Confirm a confirmation modal prompts the user before deleting the photo.
3. **Campaign Payout Gating**:
   - Create and publish a campaign without a connected Stripe account.
   - Confirm behavior on `/campaigns` (filtered) and `/campaigns/:id` (gated or amber notice).
   - Complete onboarding in Stripe Connect and confirm the campaign appears immediately in `/campaigns` and opens for donations.
4. **User Roles**:
   - Sign in as an organizer or admin.
   - Confirm that dashboard tabs (`My campaigns`, `Payouts`, `Team`) render immediately without requiring page refresh.
