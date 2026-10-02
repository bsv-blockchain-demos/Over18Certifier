# Over18Certifier

A BSV identity certificate prototype with a Next.js frontend and an Express signing service. It explores wallet-based identity, DID certificates and identity credentials containing an `over18` claim.

## Current status

Age eligibility is calculated from the date of birth entered by the user. Independent identity-document or age verification is not implemented.

The main page currently starts with email verification marked as complete. The verification API also contains unfinished signature, ownership-proof and revocation checks. These limitations make the repository a development prototype; its certificates and verification responses should not be used as an age-access control.

## Application flow

1. Connect a compatible BSV wallet and look for existing DID and identity certificates.
2. Create a DID certificate when required.
3. Enter identity details, including a date of birth in `DD/MM/YYYY` format.
4. Calculate the age claim and request an identity certificate from the signing service.
5. Store and inspect the certificate through the connected wallet.

The signing service checks a client nonce, creates a transaction containing a revocation output and signs the certificate. Certificate issuance therefore needs a configured, funded certifier wallet and can create real blockchain transactions.

## Requirements

- Node.js 22 and npm.
- MongoDB for the database-backed routes.
- A BSV wallet accessible to the browser.
- A certifier private key and a compatible wallet storage service.
- Brevo credentials if enabling the email verification flow.

## Install and configure

```sh
git clone https://github.com/bsv-blockchain-demos/Over18Certifier.git
cd Over18Certifier
npm install
npm install --prefix server
cp .env.example .env
```

The repository has no committed npm lockfiles, so installation uses `npm install`.

Edit the root `.env` before starting the services:

| Variable | Used by | Purpose |
| --- | --- | --- |
| `SERVER_PRIVATE_KEY` | Signing and revocation code | Certifier's hex-encoded private key. |
| `WALLET_STORAGE_URL` | Server wallet | Wallet storage service endpoint. |
| `CHAIN` | Certificate signing and deletion | Use `main` with the current server configuration. |
| `NEXT_PUBLIC_SERVER_PUBLIC_KEY` | Frontend and verification routes | Public identity key corresponding to the certifier's private key. |
| `NEXT_PUBLIC_CERTIFIER_URL` | Frontend certificate requests | Signing service URL, normally `http://localhost:8080` locally. |
| `MONGODB_URI` | Database-backed routes | MongoDB connection string. |
| `PORT` | Express server | Optional signing-service port; defaults to `8080`. |
| `BREVO_API_KEY`, `SENDER_EMAIL` | Email route | Required when enabling verification emails. |

Add `NEXT_PUBLIC_CERTIFIER_URL` explicitly: the example file currently lists `NEXT_PUBLIC_SERVER_URL`, while the certificate acquisition code reads `NEXT_PUBLIC_CERTIFIER_URL`.

The authentication wallet in [server/index.js](server/index.js) selects the main chain directly. Changing `CHAIN` alone does not switch the whole application to another network. Keep private keys out of variables prefixed with `NEXT_PUBLIC_`.

The database code selects the `Over18Certifier` database. For a local MongoDB instance without TLS, explicitly include `tls=false` in the connection string; otherwise the connection helper adds TLS options. The existing [Compose file](docker-compose.yml) starts MongoDB only and uses development credentials that must match the connection string.

## Start the services

Run these commands in separate terminals, both from the repository root.

Start the certificate signing service:

```sh
node --env-file=.env server/index.js
```

Start the frontend:

```sh
npm run dev -- --hostname 127.0.0.1
```

Open `http://localhost:3000`. The signing service listens on port 8080 by default and provides `/.well-known/auth`, `/acquireCertificate` and `/signCertificate`.

For the frontend production build:

```sh
npm run build
npm start
```

The signing service remains a separate process. A valid `MONGODB_URI` setting is required when the frontend build imports its database module.

## Known implementation gaps

- The `over18` value comes from client-supplied information; the signer does not independently establish the person's age.
- Email verification is bypassed in the main page. The email flow also generates its verification code in the browser.
- `/api/verify-certificate` checks the expected certifier and presence of a signature, but cryptographic signature verification and ownership challenge-response remain TODOs.
- Revocation checking uses database presence as a placeholder. The signer currently does not persist the certificate record it prepares, so the database-backed verification flow is incomplete.
- Development logs include the server private key and personal information. Remove that logging before supplying live credentials or personal data.
- The Dockerfile uses `npm ci` without a committed lockfile and installs production dependencies before building. Its build setup needs updating before it can serve as a reproducible deployment path.

## Source guide

| Location | Responsibility |
| --- | --- |
| [src/app/page.js](src/app/page.js) | Identity form and certificate acquisition. |
| [src/context/DidContext.js](src/context/DidContext.js) | DID lifecycle and wallet certificate handling. |
| [src/lib/bsv/](src/lib/bsv/) | DID and identity credential helpers. |
| [server/index.js](server/index.js) | Express server and authentication middleware. |
| [server/signCertificate.js](server/signCertificate.js) | Nonce handling, revocation output and certificate signing. |
| [src/app/api/verify-certificate/route.js](src/app/api/verify-certificate/route.js) | Prototype verification API. |
| [src/app/emailVerify/route.js](src/app/emailVerify/route.js) | Email code storage and verification. |

## Licence

**Server licence declaration: ISC.** See [server/package.json](server/package.json). The [root package](package.json) has no licence declaration. No standalone licence file is included in this repository.
