# Building your first web client

In this tutorial, we will build a simple web app to interact with the Counter contract we've just deployed.
We'll also configure it as a [Telegram Web App](https://core.telegram.org/bots/webapps).

## Step 1 - Install Tonkeeper 
Install Tonkeeper on your mobile device. Download it from [https://tonkeeper.com/](https://tonkeeper.com/)

---
network:testnet
---
Switch tonkeeper to testnet:
* Go to Settings
* Scroll down until you see the version
* Tap on the version 5 times quickly
* Tap switch to testnet

---

## Step 2 - Create a React project
Create the project using [vite](https://vitejs.dev/) (a quick way of jumpstarting react)

```
npm create vite@latest my-twa -- --template react-ts
cd my-twa
npm install
npm install ton ton-crypto ton-core
npm install @orbs-network/ton-access
```

## Step 3 - Add node.js polyfills
We need to polyfill node.js `Buffer` in order to work with the `ton` NPM package in the browser. 

First, install the node.js polyfills.

```
npm install vite-plugin-node-polyfills
```
  
Modify `vite.config.ts` so it looks like this: 

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import { nodePolyfills } from "vite-plugin-node-polyfills";

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react(), nodePolyfills()],
});
```

## Step 4 - Set up TON Connect
TON Connect is the dapp library to enable wallet interaction

Install TON Connect UI. It's still in beta, but it will handle all wallet interaction for us:

* Getting the list of Ton-Connect2 supported wallets
* Getting the address from the wallet
* Sending a transaction through the wallet

```
npm i @tonconnect/ui-react
```

Modify main.tsx to use TON Connect provider

```tsx
import { TonConnectUIProvider } from '@tonconnect/ui-react';

// We use this temporarily for development purposes and will replace it later in this tutorial.
const manifestUrl = "https://raw.githubusercontent.com/ton-community/tutorials/03-twa/03-client/tonconnect-manifest.json";

ReactDOM.createRoot(document.getElementById("root") as HTMLElement).render(
  <TonConnectUIProvider manifestUrl={manifestUrl}>
    <App />
  </TonConnectUIProvider>
);
```

## Step 5 - Connect to your wallet from the webapp

Update your ```<App/>``` code
```tsx
import { TonConnectButton } from '@tonconnect/ui-react';
...
function App() {
  return (
      <div>
        <TonConnectButton />
      </div>
  );
}
```

Run `npm run dev` and open your web app. You should be able to connect at this point with your wallet and view your address within the webapp.

## Step 6 - Read the Counter value

Now we'll interact with the counter contract. Add to your project the `Counter` class from the previous tutorial, and store it under `src/contracts/counter.ts`.

useAsyncInitialize - Will help us initialize dependencies. Put it under `src/hooks/useAsyncInitialize.ts`.
```ts
import { useEffect, useState } from "react";

function useAsyncInitialize<T>(func: () => Promise<T>, deps: any[] = []) {
  const [state, setState] = useState<T | undefined>();

  useEffect(() => {
    (async () => {
      setState(await func());
    })();
  }, deps);

  return state;
}
```

useTonClient - will help us interact with TON. Put it under `src/hooks/useTonClient.ts`.

---
network:mainnet
---
```ts
import { getHttpEndpoint } from "@orbs-network/ton-access";
import { TonClient } from "ton";

function useTonClient() {
  return useAsyncInitialize(
    async () =>
      new TonClient({
        endpoint: await getHttpEndpoint(),
      })
  );
}
```
---

---
network:testnet
---
```ts
import { getHttpEndpoint } from "@orbs-network/ton-access";
import { TonClient } from "ton";

function useTonClient() {
  return useAsyncInitialize(
    async () =>
      new TonClient({
        endpoint: await getHttpEndpoint({ network: "testnet" }),
      })
  );
}
```
---

Add the useCounterContract hook, which will get the counter's value. Put it under `src/hooks/useCounterContract.ts`

```ts
import Counter from './contracts/counter.ts';

function useCounterContract() {
  const tc = useTonClient();
  const [val, setVal] = useState<null | number>();
  
  const ctrct = useAsyncInitialize(async () => {
    if (!tc) return;
    const contract = new Counter(
      Address.parse(<COUNTER_CONTRACT_ADDRESS>)
    );
    return tc.open(contract) as OpenedContract<Counter>;
  }, [tc]);

  useEffect(() => {
    async function getValue() {
      if (!ctrct) return;
      setVal(null);
      const val = await ctrct.getCounter();
      setVal(Number(val));
    }
    getValue();
  }, [ctrct]);

  return {
    value: val,
    address: ctrct?.address.toString(),
  };
}
```

Update your App.tsx:

```tsx
function App() {
  const { value, address } = useCounterContract();
  const { connected } = useTonConnect();

  return (
    <div className="App">
      <div className="Container">
        <TonConnectButton />

        <div className="Card">
          <b>Counter Address</b>
          <div className="Hint">{address?.slice(0, 30) + "..."}</div>
        </div>

        <div className="Card">
          <b>Value</b>
          <div>{value ?? "Loading..."}</div>
        </div>
      </div>
    </div>
  );
}
```

Run your webapp with `npm run dev`, and you should see the counter's value.
Don't worry about styling, we will handle this later.


## Step 7 - Increment the counter

add the `useTonConnect` hook, which will expose whether we are connected with the wallet, and the Sender interface which is needed to send operations via our Counter class. Put it under `src/hooks/useTonConnect.ts`.
```ts
import { useTonConnectUI } from "@tonconnect/ui-react";
import { Sender, SenderArguments } from "ton";

function useTonConnect(): { sender: Sender; connected: boolean } {
  const [tonConnectUI] = useTonConnectUI();

  return {
    sender: {
      send: async (args: SenderArguments) => {
        tonConnectUI.sendTransaction({
          messages: [
            {
              address: args.to.toString(),
              amount: args.value.toString(),
              payload: args.body?.toBoc().toString("base64"),
              // Note that this does not send a StateInit, but it is not needed for our needs
            },
          ],
          validUntil: Date.now() + 5 * 60 * 1000, // TODO
        });
      },
    },
    connected: tonConnectUI.connected,
  };
}
```

Update the `useCounterContract` hook, which will poll the counter value every 3 seconds and expose a function to send an increment op, using TON Connect 2, to your wallet. The hook's content after the update should be:

```ts
import Counter from './contracts/counter.ts';

function useCounterContract() {
  const tc = useTonClient();
  const [val, setVal] = useState<null | number>();
  const { sender } = useTonConnect();

  const sleep = (time: number) => new Promise((resolve) => setTimeout(resolve, time));

  const ctrct = useAsyncInitialize(async () => {
    if (!tc) return;
    const contract = new Counter(
      Address.parse(<COUNTER_CONTRACT_ADDRESS>)
    );
    return tc.open(contract) as OpenedContract<Counter>;
  }, [tc]);

  useEffect(() => {
    async function getValue() {
      if (!ctrct) return;
      setVal(null);
      const val = await ctrct.getCounter();
      setVal(Number(val));
      await sleep(3000);
      getValue();
    }
    getValue();
  }, [ctrct]);

  return {
    sendIncrement: () => {
      return ctrct?.sendIncrement(sender);
    },
    value: val,
    address: ctrct?.address.toString(),
  };
}
```

Update your App.tsx, so it will look like this:

```tsx
function App() {
  const { sendIncrement, value, address } = useCounterContract();
  const { connected } = useTonConnect();

  return (
    <div className="App">
      <div className="Container">
        <TonConnectButton />

        <div className="Card">
          <b>Counter Address</b>
          <div className="Hint">{address?.slice(0, 30) + "..."}</div>
        </div>

        <div className="Card">
          <b>Value</b>
          <div>{value ?? "Loading..."}</div>
        </div>
        <div
          className={`Button ${connected ? "Active" : "Disabled"}`}
          onClick={() => {
            sendIncrement();
          }}
        >
          Increment
        </div>
      </div>
    </div>
  );
}
```

## Step 8 - Style the app

First, delete content of your index.css file so that it's completely empty.

Then, replace the contents of your App.css file with:
```css
:root {
  --tg-theme-bg-color: #efeff3;
  --tg-theme-button-color: #63d0f9;
  --tg-theme-button-text-color: black;
}

.App {
  height: 100vh;
  background-color: var(--tg-theme-bg-color);
  color: var(--tg-theme-text-color);
}

.Container {
  padding: 2rem;
  max-width: 500px;
  display: flex;
  flex-direction: column;
  gap: 30px;
  align-items: center;
  margin: 0 auto;
  text-align: center;
}

.Button {
  background-color: var(--tg-theme-button-color);
  color: var(--tg-theme-button-text-color);
  display: inline-block;
  padding: 10px 20px;
  border-radius: 10px;
  cursor: pointer;
  font-weight: bold;
  width: 100%;
}

.Disabled {
  filter: brightness(50%);
  cursor: initial;
}

.Button.Active:hover {
  filter: brightness(105%);
}

.Hint {
  color: var(--tg-theme-hint-color);
}

.Card {
  width: 100%;
  padding: 10px 20px;
  border-radius: 10px;
  background-color: white;
}

@media (prefers-color-scheme: dark) {
  :root {
    --tg-theme-bg-color: #131415;
    --tg-theme-text-color: #fff;
    --tg-theme-button-color: #32a6fb;
    --tg-theme-button-text-color: #fff;
  }

  .Card {
    background-color: var(--tg-theme-bg-color);
    filter: brightness(165%);
  }

  .CounterValue {
    border-radius: 16px;
    color: white;
    padding: 10px;
  }


}
```

## Step 9 - Increment the counter from your app
You should now have a working webapp. Run it with `npm run dev`, then connect your wallet with Tonkeeper.

After connecting, increment the counter and watch its value change.


## Step 10 - Publish to Github pages

Now add the real TON Connect manifest file. This file tells the wallet details about your app when connecting with it.
Add a `tonconnect-manifest.json` in your `public` folder with the following contents:

Don't worry about the broken icon url for now, in your production app you will just change it to a publicly available URL of your logo.

```json
{
  "url": "",
  "name": "My TWA",
  "iconUrl": ""
}
```

Update main.tsx to instruct the TON Connect provider to consume the manifest from your github pages url (if running in production).

```tsx
const manifestUrl = "https://myuser.github.io/my-twa/tonconnect-manifest.json";
```

Follow the vite publish to [github pages tutorial](https://vitejs.dev/guide/static-deploy.html#github-pages)

Take note of your github pages URL, such as `https://myuser.github.io/my-twa`

## Step 11 - Add the Telegram Web App SDK

Add the Telegram Web App SDK, so we can get theming properties from Telegram.

Run
```
npm install @twa-dev/sdk
```

And in your App.tsx, add
```
import "@twa-dev/sdk";
```

## Step 12 - Connect the webapp to your bot

Create a Telegram bot.

* Go to [botfather](https://t.me/botfather) and create a bot

* Set the menu button URL to your github pages site:
  
  * In Telegram's chat with botfather, type /mybots
  
  * Select your bot

  * Select 'Bot Settings'

  * Select 'Menu Button'

  * Select 'Edit Menu Button URL'

  * Type your github pages URL

* Go to your bot in Telegram and launch the webapp
