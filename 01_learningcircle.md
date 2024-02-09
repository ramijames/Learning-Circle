# Circle Notes

These are general notes on how to think about Circle's technologies in the context of building applications around their stack.

- Payments are slow and expensive
- Settlement times are slow
- Intermediaries are expensive
- Intermediaries are not transparent

Stablecoins are a new type of cryptocurrency that are pegged to a stable asset, such as the US dollar. This means that the value of a stablecoin is not subject to the same volatility as other cryptocurrencies. This makes stablecoins a good option for people who want to use cryptocurrency for everyday transactions.

- Cryptocurrency is volatile
- Unsuitable for everyday transactions
- Stablecoins are pegged to a stable asset

USDC is one of the world's most popular stablecoins. It is pegged to the US dollar, which means that 1 USDC is always worth 1 US dollar. USDC is built on the Ethereum blockchain, which means that it is fast, secure, and cheap to use.

When you allow your users to pay in USDC, you gain all the advantages of cryptocurrencies, including:

- Near-instant settlements
- Low-cost transactions
- Easy cross-border payments
- Zero chargeback liability

---

# INTEGRATION NOTES

## Users need a wallet

- Users need to install a wallet. Metamask is acceptable.
- Save your recovery phrase somewhere safe. Ffs.
- Users must have some ETH in their wallet to pay for gas fees, but we should probably do that on a Testnet like Goerli
- This should be done via the https://goerlifaucet.com/
- There's a USDC testnet faucet from Circle here: https://faucet.circle.com/
- Users will need to import the USDC token manually.
  - you’ll probably need to import the USDC token manually into your wallet. You can do so by clicking Refresh List on MetaMask.
  - An alternative way to acquire USDC is to swap some of your goerliETH on a platform such as Uniswap. You can acquire a large amount of test USDC this way on account of the test conversion rates.

## Users need a Circle sandbox account

- https://app-sandbox.circle.com/signup/sandbox
- You'll need a sandbox API key to use the Circle API. You can get one by signing up for a Circle account and creating a sandbox application.

## Building a test app

- You'll need node and npm installed
- You'll need to install the Circle SDK

```sh
$ node -v && npm -v
$ mkdir payment-infra && cd payment-infra
$ npm init -y
$ npm install @circle-fin/circle-sdk ethers
$ touch payments.js
```

Open the payments.js file and add the following code:

```js
// Import necessary packages
const uuid = require("uuid")
const { Circle, CircleEnvironments, PaymentIntentCreationRequest } = require("@circle-fin/circle-sdk");

// Instantiate Circle SDK object with API key
const CIRCLE_API_KEY = 'SAND_API_KEY:a772d01fa660452b4fbf2d62b15692d0:ee43119fbf2e4c0677921f22f5ae66da'
const circle = new Circle(
    CIRCLE_API_KEY,
    CircleEnvironments.sandbox,
);

// Create intent to pay 1 USDC
async function createUsdcPayment(amount) {
    const reqBody = {
        // Change amount here to whatever you wish
        amount: {
            amount: amount,
            currency: "USD"
        },
        settlementCurrency: "USD",
        // Since we're using sandbox, ETH here refers to the goerli chain
        paymentMethods: [
            {
                type: "blockchain",
                chain: "ETH"
            }
        ],
        idempotencyKey: uuid.v4()
    };

    // Create payment intent using SDK
    const resp = await circle.cryptoPaymentIntents.createPaymentIntent(reqBody);
    return resp.data
}

// Get Payment intent
async function getPaymentIntent(paymentIntentId) {
    const resp = await circle.cryptoPaymentIntents.getPaymentIntent(paymentIntentId);
    return resp.data;
}

// Helper function to create a delay (in milliseconds)
function delay(time) {
    return new Promise((resolve) => setTimeout(resolve, time));
}

// Poll payment ID to get payment intent every 500 ms
async function pollPaymentIntent(paymentIntentId) {

    // Poll every 0.5 seconds or 500 ms
    const pollInterval = 500;

    let resp = undefined;
    while (true) {
        resp = await getPaymentIntent(paymentIntentId);

        // Check if deposit address has been created
        let depositAddress = resp.data?.paymentMethods[0].address;

        // If address created, break. Else check again in 500 ms
        if (depositAddress) break;
        await delay(pollInterval);
    }

    return resp.data;
}

async function main() {

    // Create payment intent
    const paymentIntent = await createUsdcPayment(“1”);
    const paymentIntentId = paymentIntent.data.id;
    const payment = await pollPaymentIntent(paymentIntentId);
    const address = payment.paymentMethods[0].address;

    // Get wallet address
    console.log(`Payment Intend ID: ${paymentIntentId}`)
    console.log(`Please pay 1 USDC to ${address}`)
}

main()
```

There is a lot of code here, but the core logic is extremely simple:

1. We have a function `createUSDCPayment` that creates a payment intent using Circle for a certain amount of USDC.

2. We have a helper function that gets the latest status of the payment intent we create (by intent ID).

3. Finally, we have a poll intent function that polls the helper function from (2) until a wallet address is generated.

Run this code using the following:

```sh
$ node payments.js
```

You should see output that looks something like this:

```sh
Payment Intend ID: 76075284-212d-4268-9dfe-4780a6744beb
Please pay 1 USDC to 0x0d53d534bb7bb10b7bfe50095a113a579759900a
```

Next you'll be making the payment as the customer.

## Making a payment

- in the extension, click on the USDC token
- send GOERLI USDC to the address above (0x0d53d534bb7bb10b7bfe50095a113a579759900a in this case, check your output)

You can check the status of the payment either via the merchant payments dashboard (https://app-sandbox.circle.com/platform/payments) or by using the Circle API programmatically.

For the programmatic approach (which is what you’ll likely use when dealing with thousands of payments), let’s write a script in a new file called confirmation.js.

```js
const { Circle, CircleEnvironments } = require("@circle-fin/circle-sdk");

const circle = new Circle(
  "SAND_API_KEY:a772d01fa660452b4fbf2d62b15692d0:ee43119fbf2e4c0677921f22f5ae66da",
  CircleEnvironments.sandbox
);

async function getPayment(paymentId) {
  const resp = await circle.payments.getPayment(paymentId);
  console.log(resp.data);
}

async function getPaymentIntent() {
  const paymentIntentId = "76075284-212d-4268-9dfe-4780a6744beb";
  const paymentIntent = await circle.cryptoPaymentIntents.getPaymentIntent(
    paymentIntentId
  );

  const paymentIds = paymentIntent.data.data.paymentIds;
  let paymentId;

  // If payment has been made, paymentsId will be a non-empty list
  if (paymentIds.length > 0) {
    paymentId = paymentIds[0];
  } else {
    console.log("Payment hasn't been made yet!");
    return;
  }

  getPayment(paymentId);
}

getPaymentIntent();
```

This code should be familiar to you from the last step. All we’re doing here is accessing the payment intent by ID and checking if a payment has been made.

Run the confirmation script using:

```sh
$ node confirmation.js
```

You should see an output that looks something like this:

```json
{
  data: {
    id: 'babec85e-00b0-374c-ad6f-a854218cccbd',
    type: 'payment',
    status: 'paid',
    amount: { amount: '1.00', currency: 'USD' },
    fees: { amount: '0.01', currency: 'USD' },
    createDate: '2023-08-13T12:39:29.455894Z',
    updateDate: '2023-08-13T12:43:07.156878Z',
    merchantId: 'bfdb53c3-9e63-4116-b3da-bf357e78bbba',
    merchantWalletId: '1016320978',
    paymentIntentId: '76075284-212d-4268-9dfe-4780a6744beb',
    settlementAmount: { amount: '1.00', currency: 'USD' },
    fromAddresses: { chain: 'ETH', addresses: [Array] },
    depositAddress: {
      chain: 'ETH',
      address: '0x0d53d534bb7bb10b7bfe50095a113a579759900a'
    },
    transactionHash: '0x52df384329672112a2782715187409bbe8ad6fa7a6670b24b3a1c14a8f4d3cb4'
  }
}
```

Accessing data[status] and checking if it is paid will allow you to code any custom logic from here.

## Payment Gateway

### Prerequisites

You’ll require the following things:

1. A MetaMask wallet with at least 0.01 ETH and at least 1 USDC on the Goerli testnet
2. A Circle Sandbox Developer Account and API Key
3. Node and NPM installed on your local machine

First let’s create a boilerplate Next app using `create-next-app.` We’ll also need to install the following packages:

- @circle-fin/circle-sdk: To access Circle API endpoints from the app
- @types/uuid: To create UUIDs (required for creating payment intents)
- wagmi: To implement connect wallet functionality

To do all of the above, run the following commands in order:

```sh
$ node -v && npm -v
$ npm install create-next-app
$ npx create-next-app circle-app
$ npm install @circle-fin/circle-sdk @types/uuid wagmi
$ cd circle-app
$ npm run dev
```

When creating the Next app, accept all defaults except for the modern app router. If all goes well, once complete you should see a template page when you visit localhost:3000.

Create a new file called constants.js. This is where you will add your Circle Sandbox API key.

```js
// Circle sandbox API key
const circleSandboxApiKey = "< CIRCLE API KEY >";

export { circleSandboxApiKey };
```

Now let’s implement the ability for a user to connect their wallet to our app. We’ll do this using the wagmi library. The wagmi library is a collection of React hooks for Ethereum. It makes it easy for us to do things like connect to a wallet, display balance information, sign messages, and more.

Note that our app will only connect to the MetaMask wallet. If you want to support other wallets such as Coinbase, you can check out the documentation to see how.

In pages/\_app.js, add the boilerplate configuration code:

```js
import { WagmiConfig, createConfig, configureChains } from "wagmi";
import { goerli } from "@wagmi/core/chains";

import { publicProvider } from "wagmi/providers/public";
import { MetaMaskConnector } from "wagmi/connectors/metaMask";

// Configure chains & provider
const { chains, publicClient, webSocketPublicClient } = configureChains(
  [goerli],
  [publicProvider()]
);

// Set up wagmi config
const config = createConfig({
  autoConnect: true,
  connectors: [new MetaMaskConnector({ chains })],
  publicClient,
  webSocketPublicClient,
});

export default function App({ Component, pageProps }) {
  return (
    <WagmiConfig config={config}>
      <Component {...pageProps} />
    </WagmiConfig>
  );
}
```

Next, create a new file in the pages folder called connect.js. The main connect logic will reside here:

```js
import Head from "next/head";
import { Fragment } from "react";
import styles from "../styles/connect.module.css";
import { useConnect, useAccount } from "wagmi";
import { useState, useEffect } from "react";
import { useRouter } from "next/router";
import { goerli } from "@wagmi/core/chains";

export default function Connect() {
  const { connect, connectors, error, isLoading, pendingConnector } =
    useConnect({
      chainId: goerli.id,
    });
  const { isConnected } = useAccount();
  const [hasMounted, setHasMounted] = useState(false);
  const router = useRouter();

  // Prevent hydration errors
  useEffect(() => {
    setHasMounted(true);
  }, []);

  // If connected, go back to the main page
  if (!hasMounted) return null;
  if (isConnected) router.replace("/");

  return (
    <Fragment>
      <Head>
        <title>Connect Wallet</title>
      </Head>
      <div className={styles.jumbotron}>
        <h1>My USDC Payment Gated App</h1>
        {connectors.map((connector) => (
          <button
            disabled={!connector.ready}
            key={connector.id}
            onClick={() => connect({ connector })}
          >
            Connect Wallet
            {!connector.ready && " (unsupported)"}
            {isLoading &&
              connector.id === pendingConnector?.id &&
              " (connecting)"}
          </button>
        ))}
        {error && <div className={styles.error}>{error.message}</div>}
      </div>
    </Fragment>
  );
}
```

Now let’s move on to the more interesting code which is specific to our use case. In this step, we’ll implement two major pieces of functionality:
Creation of payment intent and a corresponding deposit address
Ability to check if a payment associated with a particular payment ID has been made

The details of these features were covered in the previous tutorial, so they should look familiar. The main difference here is that we’re creating an actual app and using best practices for a frontend framework using Next.

First, in the root directory, create a new folder called utils. Add a new file called payments.js and add the code to implement creation of payment IDs.

```js
import { v4 as uuid } from "uuid";
import { Circle, CircleEnvironments } from "@circle-fin/circle-sdk";
import { circleSandboxApiKey } from "@/data/constants";

// Instantiate Circle SDK object with API key
const circle = new Circle(circleSandboxApiKey, CircleEnvironments.sandbox);

// Create intent to pay 1 USDC
async function createUSDCPayment() {
  const reqBody = {
    amount: {
      amount: "1",
      currency: "USD",
    },
    settlementCurrency: "USD",
    // Since we're using sandbox, ETH here refers to the goerli chain
    paymentMethods: [
      {
        type: "blockchain",
        chain: "ETH",
      },
    ],
    idempotencyKey: uuid(),
  };

  // Create payment intent using SDK
  const resp = await circle.cryptoPaymentIntents.createPaymentIntent(reqBody);
  return resp.data;
}

// Get Payment intent
async function getPaymentIntent(paymentInentId) {
  const resp = await circle.cryptoPaymentIntents.getPaymentIntent(
    paymentInentId
  );
  return resp.data;
}

// Helper function to create a delay (in milliseconds)
function delay(time) {
  return new Promise((resolve) => setTimeout(resolve, time));
}

// Poll payment ID to get payment intent every 500 ms
async function pollPaymentIntent(paymentIntentId) {
  // Poll every 0.5 seconds or 500 ms
  const pollInterval = 500;

  let resp = undefined;
  while (true) {
    resp = await getPaymentIntent(paymentIntentId);

    // Check if deposit address has been created
    let depositAddress = resp.data?.paymentMethods[0].address;

    // If address created, break. Else check again in 500 ms
    if (depositAddress) break;
    await delay(pollInterval);
  }

  return resp.data;
}

// Create wallet address and return address + payment intent
async function createWalletAddress() {
  // Create payment intent for 1 USDC
  const paymentIntent = await createUSDCPayment();
  const paymentIntentId = paymentIntent.data.id;
  const payment = await pollPaymentIntent(paymentIntentId);
  const address = payment.paymentMethods[0].address;

  // Get wallet address where customer will deposit I USDC
  console.log(`Payment Intend ID: ${paymentIntentId}`);
  console.log(`Please pay 1 USDC to ${address}`);

  return [paymentIntentId, address];
}

export { createWalletAddress };
```

This code should all be very familiar.

Now in the same folder, create another file called confirmation.js where we add the logic to set the status of payments.

```js
import { Circle, CircleEnvironments } from "@circle-fin/circle-sdk";
import { circleSandboxApiKey } from "@/data/constants";

// Instantiate Circle SDK object with API key
const circle = new Circle(circleSandboxApiKey, CircleEnvironments.sandbox);

// Check for payment status
async function checkPaymentStatus(paymentIntentId) {
  const paymentIntent = await circle.cryptoPaymentIntents.getPaymentIntent(
    paymentIntentId
  );

  const paymentIds = paymentIntent.data.data.paymentIds;
  let paymentId;

  // If payment has been made, paymentsId will be a non-empty list
  if (paymentIds.length > 0) {
    paymentId = paymentIds[0];
    return true;
  }

  console.log("Payment hasn't been made yet!");
  return false;
}

export { checkPaymentStatus };
```

Now that we have all the pieces in place for us to code our main app logic, the flow for our app will be:

1. If the user is not connected to the app, they are redirected to the /connect page.
2. Once connected, the user is redirected to the home page and informed that the content is gated and they must first make a payment.
3. When the user clicks the Make Payment button, a payment intent is created.
4. Once the intent creation is complete, inform the user that they need to make a 1 USDC payment to a particular address.
5. User makes payment through MetaMask.
6. Once the payment is made, the user clicks Confirm. The payment confirmation flow is activated.
7. If payment is confirmed, the token gated content is displayed.

In the index.js file, add the following code:

```js
// Standard Next and CSS imports
import Head from "next/head";
import { Fragment, useState, useEffect } from "react";
import styles from "../styles/mainpage.module.css";
import { useRouter } from "next/router";
import { Inter } from "next/font/google";

// Wagmi import for connected wallet info
import { useAccount } from "wagmi";

// Custom imports
import { createWalletAddress } from "@/utils/payments";
import { checkPaymentStatus } from "@/utils/confirmation";

const inter = Inter({ subsets: ["latin"] });

export default function Home() {
  // Standard Next router definition
  const router = useRouter();

  // Get connected wallet address and connection status
  const { isConnected } = useAccount();

  // Prevent Hydration errors
  const [hasMounted, setHasMounted] = useState(false);

  // Payment intent and deposit address
  const [intent, setIntent] = useState(null);
  const [depositAddress, setDepositAddress] = useState(null);

  // Payment status and error checker (ie payment hasn't been made yet)
  const [paymentStatus, setPaymentStatus] = useState(false);
  const [error, setError] = useState(null);

  // Hydration error fix
  useEffect(() => {
    setHasMounted(true);
  }, []);

  // Do not render until entire UI is mounted
  if (!hasMounted) return null;

  // Redirect to Connect page if wallet is not connected
  if (!isConnected) {
    router.replace("/connect");
  }

  // Create payment intent and set intent ID + deposit address
  const createPaymentIntent = async (e) => {
    e.preventDefault();

    let [intent, address] = await createWalletAddress();
    setIntent(intent);
    setDepositAddress(address);
  };

  // Confirm if payment has been made
  const confirmPayment = async (e) => {
    e.preventDefault();

    let paymentStatus = await checkPaymentStatus(intent);
    if (paymentStatus) {
      setIntent(null);
      setDepositAddress(null);
      setError(null);
    } else {
      setError(
        "Payment hasn't been confirmed yet. Try again in sometime in case it's already done."
      );
    }
    setPaymentStatus(paymentStatus);
  };

  return (
    <Fragment>
      <Head>
        <title>My USDC Payment Gated App</title>
      </Head>

      <main className={inter.className}>
        <div className={styles.jumbotron}>
          <h1>USDC Gated App</h1>
          {/* Display when no intent to pay has been made */}
          {!intent && !depositAddress && !paymentStatus && (
            <div>
              <p>
                You will need to make a payment to access this gated content.
              </p>
              <form onSubmit={createPaymentIntent} className={styles.mint_form}>
                <button type="submit">Pay 1 USDC</button>
              </form>
            </div>
          )}

          {/* Display when intent to pay is created but payment is pending */}
          {intent && depositAddress && !paymentStatus && (
            <div>
              <p>Please pay 1 USDC to {depositAddress}</p>
              <p>Once done, click for confirmation</p>
              <form onSubmit={confirmPayment} className={styles.mint_form}>
                <button type="submit">Confirm</button>
              </form>
            </div>
          )}

          {/* Display when payment is made */}
          {!intent && !depositAddress && paymentStatus && (
            <div>
              <p>Thank you for the payment!</p>
              <p>Here is some super exclusive content!</p>
            </div>
          )}
          {error && <p>{error}</p>}
        </div>
      </main>
    </Fragment>
  );
}
```

Note a couple of key sections of code above:
`createPaymentIntent` is triggered when we click on the Pay 1 USDC button. This will create a payment intent and set the intent ID and deposit address on the frontend.
`confirmPayment` is triggered when we click the Confirm button. This will use the payment intent ID created in (1) and check if a payment has been made.

If your app is not running already, do so by running:

```sh
$ npm run dev
```

## Programmable Wallets

- https://www.circle.com/en/programmable-wallets

Is a wallet-as-a-service solution that allows developers to create and manage digital wallets for their users. It enables developers to easily support USDC payments for their users behind the scenes.

There are two main types:

1. Developer-controlled wallets
2. User-controlled wallets

The first option—developer-controlled wallets—creates wallets behind the scenes for users and allows your app to retain all control over the wallet. You as the app owner manage the interactions, sending and receiving of USDC, and more, all through the API and SDK. Your users don’t have to understand anything about web3 or blockchain, don’t need to remember a PIN, and don’t even have to know that a wallet exists.

The second option—user-controlled wallets—allows you to accept USDC and store it securely using your app, but still allows the user to have full custody of the wallet. In this scenario, during the wallet creation process, the user sets a PIN and 2FA using security questions just as they do in many familiar web2 workflows. The PIN and 2FA give them control over the wallet and its USDC, but still allow you to onboard them to your app, create the wallet seamlessly for them, securely store their USDC, and help manage their activity.

```curl
// Create a set of wallets
POST https://api.circle.com/v1/w3s/user/wallets
{
"blockchains": [
“ETH”,
“AVAX”,
“MATIC”
],
“metadata”: [
{
“name”: “My First Wallet”
}
],
"idempotencyKey":
"{{idempotencyKey}}"
}
```

Programmable Wallets eschew the notions of keys and wallet recovery phrases, and instead provide the end-user with familiar paradigms such as a username, email, PIN, and security questions. In doing so, Programmable Wallets provide the following advantages:

- Familiar UX - With Programmable Wallets, users can’t tell the difference between their web2 and web3 experience; by using mechanisms like PINs and security questions, conducting secure transactions through wallets becomes as simple as using a well-designed iOS or Android app.

- Simple access - In order to access their wallet, users will only need to remember a PIN or answers to security questions; they do not remember a random set of words.

- Advanced security - Circle ensures that the easy UX does not come at the cost of security by employing advanced MPC technology.

## Smart Contract Platform

Circle allows the import of external smart contracts on any major platform (Polygon, Avalanche, Ethereum, etc.). You can convert the core logic into an easy-to-use REST API.

Reading and writing from the contract (USDC contract on Testnet is 0xeddf1273fc35b0beda96bddea280c218869bc351) is now as simple as sending GET and POST requests. You no longer need knowledge of Solidity to work with this contract.

The Smart Contract Platform also provides code snippets that work with the contract’s functions.

For example, in order to call the transfer function, you would use the following snippet:

```js
import axios from "axios";

const options = {
  method: "POST",
  url: "https://api.circle.com/v1/w3s/developer/transactions/contractExecution",
  headers: {
    Authorization: "Bearer ${API_KEY}",
    accept: "application/json",
    "content-type": "application/json",
  },
  data: {
    abiFunctionSignature: "transfer(address,uint256)",
    idempotencyKey: "250f7f11-fc5b-491a-8623-ef0d3f02aa50",
    contractAddress: "0xeddf1273fc35b0beda96bddea280c218869bc351",
    feeLevel: "MEDIUM",
    walletId: "${WALLET_ID}",
    entitySecretCiphertext: "${ENTITY_SECRET_CIPHERTEXT}",
  },
};

try {
  const { data } = await axios.request(options);
  console.log(data);
} catch (error) {
  console.error(error);
}
```

## Smart Contract Composability

The concept of composability means that you can extend the functionality of your smart contract by integrating it with other smart contracts. This is a powerful concept that allows you to create complex financial products and services by combining different smart contracts.

For example, you can create a lending protocol by combining a smart contract that allows users to deposit funds and earn interest with a smart contract that allows users to borrow funds. This is just one example of the many possibilities that composability enables.
