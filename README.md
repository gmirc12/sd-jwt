# SD-JWT

This is an implementation of [SD-JWT (I-D version 05)](https://www.ietf.org/archive/id/draft-ietf-oauth-selective-disclosure-jwt-05.html) in typescript.

## Functionalities

- No cryptographic dependencies (BYOC):
  - [x] Hasher
  - [x] Signer
  - [x] Salt Generator

- Issue SD-JWT:
  - [x] Support recursive disclosures (parent object and its keys)
  - [x] Support for nested objects
  - [x] Support for arrays
  - [x] Optional: public key binding (cnf)
  - [ ] Optional: decoy digest

- Verify SD-JWT:
  - [x] Support recursive disclosures (parent object and its keys)
  - [x] Support for nested objects
  - [x] Support for arrays
  - [x] Optional: public key binding (cnf) check against the key binding JWT if one was provided
  - [x] Optional: decoy digest

- Additional:
  - [ ] Holder: separate function for a holder to be able to create a key binding JWT (6.2.2)

- Tests:
  - [x] Issuer SD-JWT tests
  - [x] Verifier SD-JWT tests
  - [x] Tests that check compatibility with SD-JWT generated by external libs
  - [x] e2e test

- Release:
  - [x] Create CommonJS and ESM builds
  - [ ] Documentation
  - [ ] Publish on npm

## Disclosure Frame
To issue or pack claims into a valid SD-JWT we use Disclosure Frame to define which properties/values should be selectively diclosable. \
It follows the following format:
```typescript
type ArrayIndex = number;
type DisclosureFrame = {
  [key: string | ArrayIndex]: DisclosureFrame | unknown;
  _sd?: Array<string | ArrayIndex>;
};
```
### Examples:

#### set property as selectively disclosable
```js
const claims = {
  firstname: 'John',
  lastname: 'Doe'
}
const diclosureFrame = {
  _sd: ['firstname'] // set firstname as selectively discloseable
}

// result
const sdjwt = {
  _sd: ['LjgwZy8TNXmmPO9mNqVDtq3jiX5r3YS-P-qw2hBNYyU']
  lastname: 'Doe',
}
```

#### nested property
```js
const claims = {
  address: {
    street: '123 Main St',
    suburb: 'Anytown',
    postcode: '1234'
  }
}

const disclosureFrame = {
  address: {
    // set address.street and address.suburb as selectively discloseable
    _sd: ['street', 'suburb'];
  }
}

// result
const sdjwt = {
  address: {
    _sd: [
      '02d7bUYevjfAzJ0Gr42ymHy66ezQVL7huNGBO68xSfs',
      'ai7P4vgPZ-Jk1QwL55BLQqtN2gwWy31-pi2VGWiIggs',
    ],
    postcode: '1234'
  }
}
```

#### Array item
```js
const claims = {
  nicknames: ['Johnny', 'JD']
}

const disclosureFrame = {
  nicknames: {
    _sd: [0, 1] // index of items in 'nicknames' Array
  }
}

// result
const sdjwt = {
  nicknames: [
    { '...': 'yfhdm_aKTMgm666j79GoXr2mer2dBW0cFfap8iXnAzY' },
    { '...': 'EU0ORASnAlqNtRwttBXsGTISxQ6myFPMBHPE0Ds8aSE' }
  ]
}
```

#### Object in Arrays
```js
const claims = {
  items: [
    {
      type: 'shirt',
      size: 'M'
    },
    'Towel',
    'Water Bottle'
  ]
}

const disclosureFrame = {
  items: {
    0: {
      _sd: ['size'] // `size` property of items[0]
    }
  }
}

// result
const sdjwt = {
  items: [
    {
      _sd: ['7aGqCE9HepzELBi59BvxxriDiV7uiB4yHTyN1im_m4M'],
      type: 'shirt'
    },
    'Towel',
    'Water Bottle'
  ]
}
```

#### Array in Arrays
```js
const claims = {
  colors: [
    ['R','G','B'], 
    ['C','Y','M','K']
  ]
}

const disclosureFrame = {
  colors: {
    0: {
      _sd: [0, 2] // `R` and `B` in colors[0]
    }
  }
}

// result
const sdjwt = {
  colors: [
    [
      { '...': '' },
      'G',
      { '...': '' }
    ],
    ['C','Y','M','K']
  ]
}
```

## issueSDJWT Example

The `issueSDJWT` function takes a JWT header, payload, and disclosure frame and returns a compact SD-JWT combined with the disclosures.

As the library is unopinionated when it comes to how you want to deal with cryptographic functions, so it requires a signer and hasher function to be provided.

### Basic Usage

Example Using `jose` lib for signer function & `crypto` for hasher;

```js
import crypto from 'crypto'
import { SignJWT, importJWK } from 'jose';

const header = {
  alg: 'ES256',
  kid: 'issuer-key-id'
};

const payload = {
  iss: 'https://example.com/issuer',
  iat: 168300000,
  exp: 188300000,
  sub: 'subject-id',
  name: 'John Doe'
};

const disclosureFrame = {
  _sd: ['name']
};

const signer = async (header, payload) => {
  const issuerPrivateKey = await importJWK(ISSUER_KEYPAIR.PRIVATE_KEY_JWK, header.alg);
  return new SignJWT(payload).setProtectedHeader(header).sign(issuerPrivateKey);
};

const hasher = (data) => {
  const digest = crypto.createHash('sha256').update(data).digest();
  const hash = Buffer.from(digest).toString('base64url');
  return Promise.resolve(hash);
};

const sdjwt = await issueSDJWT(header, payload, disclosureFrame, {
  hash: {
    alg: 'sha-256',
    callback: hasher,
  },
  signer
});

// Decoded sdjwt.payload
{
  iss: 'https://example.com/issuer',
  iat: 168300000,
  exp: 188300000,
  sub: 'subject-id',
  _sd: [
    'jlJfq0qqkvwwgPrHh6kfzO2p7hpDYX1Mve-62bHgpHE' // HASH Digest of disclosure
  ]
}

// Disclosure
"WyJ2NEVHUzhKRzlTdW9TUjVGIiwibmFtZSIsIkpvaG4gRG9lIl0" // base64url encode of ["v4EGS8JG9SuoSR5F","name","John Doe"]
```


## verifySDJWT Example

The `verifySDJWT` function takes a Compact combined SD-JWT (include optional disclosures & KB-JWT) \
**Required**: a verifier function that can verify the JWT signature \
**Required**: a getHasher function that returns a Hashed depending on the `_sd_alg` in the SD-JWT payload \
*Optional*: A Keybinding Verifier function that can verify the embedded holder key \
Returns SD-JWT with all the disclosed claims.

### Basic Usage

Example Using `jose` lib for verifier
Uses `crypto` for Hasher;

```js
import { importJWK, jwtVerify } from 'jose';

const verifier = async (jwt) => {
  const key = await getIssuerKey(); // Get SD-JWT issuer public key
  return jwtVerify(jwt, key);
};

const keyBindingVerifier = (kbjwt, holderJWK) => {
  // check against kb-jwt.aud && kb-jwt.nonce
  const { header } = decodeJWT(kbjwt);
  const holderKey = await importJWK(holderJWK, header.alg);
  const verifiedKbJWT = await jwtVerify(kbjwt, holderKey);
  return !!verifiedKbJWT;
}

const getHasher = (hashAlg) => {
  let hasher;
  // Default Hasher = Hasher for SHA-256
  if (!hashAlg || hashAlg.toLowerCase() === 'sha-256') {
    hasher = (data) => {
      const digest = crypto.createHash('sha256').update(data).digest();
      return base64encode(digest);
    };
  }
  return Promise.resolve(hasher);
};

const opts = {
  kb: {
    verifier: keyBindingVerifier
  }
}
try {
  const sdJWTwithDisclosedClaims = await verifySDJWT(compactSDJWT, verifier, getHasher, opts);
} catch (e) {
  console.log('Could not verify SD-JWT', e);
}
```

## unpackSDJWT Example

The `unpackSDJWT` function takes a SD-JWT payload with _sd digests, array of disclosures and returns the disclosed claims \
**Required**: a sd-jwt payload with `_sd` digests \
**Required**: an array of Disclosure objects \
**Required**: a getHasher function that returns a Hashed depending on the `_sd_alg` in the SD-JWT payload

### Basic Usage

```js
import crypto from 'crypto';

const getHasher = (hashAlg) => {
  let hasher;
  // Default Hasher = Hasher for SHA-256
  if (!hashAlg || hashAlg.toLowerCase() === 'sha-256') {
    hasher = (data) => {
      const digest = crypto.createHash('sha256').update(data).digest();
      return base64encode(digest);
    };
  }
  return Promise.resolve(hasher);
};

const disclosures = [{
  disclosure: 'disclosure_array_as_string', // [<salt>, <key>, <value>]
  key: 'key_of_disclosed_claim',
  value: 'value_of_disclosed_claim'
}]

const sdjwt = {
  _sd: [
    'SD_DIGEST_1',
    'SD_DIGEST_2',
  ]
}

const result = await unpackSDJWT(sdjwt, disclosures, getHasher);
```


## packSDJWT Examples

The `packSDJWT` function takes a claims object and disclosure frame and returns packed claims with selective disclosures encrypted.

### Basic Usage

```js
import { packSDJWT } from 'sd-jwt';

const claims = {
  name: 'Jane',
  ssn: '123-45-6789'
};

const disclosureFrame = {
  _sd: ['ssn']
};

const {claims: packed, disclosures} = await packSDJWT(claims, disclosureFrame, hasher);
```

This will selectively disclose `ssn` and return the packed claims and disclosures array.


### Disclosing Multiple Claims

To selectively disclose multiple claims:

```js
const claims = {
  name: 'Jane Doe',
  ssn: '123-45-6789',
  id: '1234'
};

const disclosureFrame = {
  _sd: ['ssn', 'id']
};

const {claims: packed, disclosures} = await packSDJWT(claims, disclosureFrame, hasher);

// Results
packed =  {
  "name": "Jane Doe",
  "_sd": [
      "DZkUdg_W43hB25uuSxEyt2ialCeDbweHVXcRrhQHbLY",
      "85kfxIj8lWd5WODcupbDiYEw6upYWoD1GI048JUVAHw"
  ]
}

disclosures = [
  "WyJzNnZtNTJzWjN3Y1NXNUEzIiwic3NuIiwiMTIzLTQ1LTY3ODkiXQ",
  "WyJxZEt6MURIVDRlOHBpWlZ5IiwiaWQiLCIxMjM0Il0"
]
```

### Selective Disclosable items in array

```js
const claims = {
  items: ['a', 'b', 'c']
};

const disclosureFrame = {
  items: { _sd: [1] } // item at index 1
}

const {claims: packedClaims, disclosures} = await packSDJWT(claims, disclosureFrame, hasher);

// Results
packedClaims = {
  items: [
    'a',
    {
      "...": "b64encodedhash"
    },
    'c'
  ]
}

disclosures = [
  "WyJzYWx0IiwgMV0=" // b64 encoded [salt, 'b']
]
```

## Development

### Installation

```
git clone https://github.com/Meeco/sd-jwt
npm install

npm run dev:setup
```

### Test

Runs against examples in `test/examples` directory

Examples are generated using [`sd-jwt-generate`](https://github.com/openwallet-foundation-labs/sd-jwt-python)

```
npm run test
```
