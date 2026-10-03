# AEM-CS API Client Library

This repository contains minimal Node.js examples for authenticating server-side applications with Adobe Identity Management Services (IMS) and calling AEM as a Cloud Service APIs.

**OAuth Server-to-Server is the recommended authentication method.** It uses the OAuth `client_credentials` grant, without a JWT, private key, or certificate. The original JWT README and examples are preserved in [README-jwt-legacy.md](README-jwt-legacy.md). JWT authentication is deprecated in favor of OAuth Server-to-Server.

## Requirements

- Node.js 18 or newer. The example uses built-in `fetch` and requires no dependencies.
- OAuth Server-to-Server credentials from an AEM Developer Console that supports this credential type. The AEM Developer Console is distinct from Adobe Developer Console.
- The technical account must have the product profiles and AEM permissions required by the API you want to call. Obtaining a token does not itself grant access to AEM content.

## Save the credentials

In the AEM Developer Console, create an **OAuth Server-to-Server (S2S)** technical account. Select **View** on an active client secret and save its credential JSON as `downloaded_integration.json`.

Use the actual `technicalAccount.clientSecret` value, not the secret ID shown in the account's secret list. Use the `imsEndpoint` and `scopes` from your credential JSON; JWT `metascopes` are not OAuth scopes.

The example accepts this structure:

```json
{
  "integration": {
    "imsEndpoint": "ims-na1.adobelogin.com",
    "technicalAccount": {
      "clientId": "your-client-id",
      "clientSecret": "your-client-secret"
    },
    "scopes": [
      "read_pc.dma_aem_ams",
      "openid",
      "AdobeID",
      "read_organizations",
      "additional_info.projectedProductContext"
    ]
  }
}
```

These are placeholders. Copy the values and scopes from your own OAuth credential JSON rather than using this example unchanged. Additional fields in the downloaded JSON are accepted but are not needed to request a token.

Keep the file private and do not commit it. `downloaded_integration.json` is ignored by Git in this repository. In a deployed application, load credentials from a secret manager or another access-controlled store instead of storing them in source code.

## Request an access token

From a checkout of this repository:

```sh
node oauth-cli.js downloaded_integration.json
```

The CLI reads the file and prints the IMS token response:

```json
{
  "access_token": "REDACTED",
  "token_type": "bearer",
  "expires_in": 86399
}
```

**The CLI output contains a sensitive access token.** Do not send it to shared logs or publish it.

The request in [oauth.js](oauth.js) is a form-encoded POST to `https://<imsEndpoint>/ims/token/v3` with:

| Parameter | Value |
| --- | --- |
| `grant_type` | `client_credentials` |
| `client_id` | `integration.technicalAccount.clientId` |
| `client_secret` | `integration.technicalAccount.clientSecret` |
| `scope` | The comma-separated values of `integration.scopes` |

The example rejects redirects and limits each token request to 30 seconds. Failed IMS requests produce an HTTP-status error without printing the response body or credentials.

## Call an AEM API

Import the OAuth example explicitly as `require('./oauth')`. The existing `index.js` export and `cli.js exchange` command remain the legacy JWT examples.

```javascript
const fs = require('node:fs');
const requestAccessToken = require('./oauth');

async function main() {
    const credentials = JSON.parse(
        fs.readFileSync('downloaded_integration.json', 'utf8')
    );
    const { access_token } = await requestAccessToken(credentials);

    // Replace this URL with your environment and an API the account can access.
    const response = await fetch(
        'https://author-p123-e456.adobeaemcloud.com/content/dam.json',
        { headers: { Authorization: `Bearer ${access_token}` } }
    );
    if (!response.ok) {
        throw new Error(`AEM API request failed: HTTP ${response.status}`);
    }
    console.log(await response.json());
}

main().catch((error) => {
    console.error(error.message);
    process.exitCode = 1;
});
```

For a long-running application, cache and reuse access tokens until shortly before they expire. IMS reports `expires_in` in seconds. OAuth Server-to-Server does not issue a refresh token; request another access token with the same client credentials when needed. This minimal example does not implement a token cache or automatic retries.

## Rotate a client secret

Create a replacement secret in the AEM Developer Console and retrieve its credential JSON. Update every application using the account, then request a fresh token and verify AEM access with the replacement secret before revoking the old secret.

The authentication code does not change: it uses the selected `technicalAccount.clientSecret`. Revocation can take time to propagate and does not necessarily invalidate already-issued access tokens. Follow the console's secret-management guidance when verifying revocation.

## Tests

Run the offline OAuth and JWT compatibility tests without real credentials or network calls:

```sh
npm run test:oauth
```

The original `npm test` command is preserved. It performs a live JWT exchange using your local `downloaded_integration.json`; its setup is documented in [the legacy README](README-jwt-legacy.md#how-to-test).

## Legacy JWT examples

See [README-jwt-legacy.md](README-jwt-legacy.md) for the original instructions and examples. `index.js`, `cli.js`, and `index.spec.js` are unchanged.

## Contributing

Contributions are welcomed! Read the [Contributing Guide](.github/CONTRIBUTING.md) for more information.

## Licensing

This project is licensed under the Apache V2 License. See [LICENSE](LICENSE) for more information.
