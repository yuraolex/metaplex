# Candy Machine With Whitelist

## How to Build and Deploy Candy Machine

1. Clone project from [GitHub](https://github.com/yuraolex/metaplex)
1. Install Anchor enviorment. See [Installing Dependencies](https://project-serum.github.io/anchor/getting-started/installation.html), for details.
1. Run `anchor build`
1. Run `anchor deploy`
1. Run `anchor idl init --filepath {path-to-metaplex}/metaplex/rust/target/idl/nft_candy_machine.json {candy_machine_pubkey}`
   1. Run `anchor idl upgrade --filepath {path-to-metaplex}/metaplex/rust/target/idl/nft_candy_machine.json {candy_machine_pubkey}` if candy machine alredy uploaded.

## How to Create Candy Machine using CLI

Use the same repo from [GitHub](https://github.com/yuraolex/metaplex) to build and run CLI commands. Candy machine cli located in `metaplex/js/packages/cli/src/candy-machine-cli.ts`.

1. Before creating candy machine, assets should be prepared. All assets should have related `json` metadata files.
1. To upload prepared assets and related metadata, use `candy-machine-cli.ts upload` command from metaplex project. Use `-s` or `--storage` to select database that will be used for storage (`arweave-bundle`, `arweave-sol`,`arweave`, `ipfs`, `aws`). Default: `arweave-sol`. Use `-h` option to see all possible options for this command.
   1. How to implement upload could be found [HERE](https://github.com/yuraolex/metaplex/blob/1064ba3a0c7c564019364be3de91a86306874c69/js/packages/cli/src/candy-machine-cli.ts#L206) and [HERE](https://github.com/yuraolex/metaplex/blob/1064ba3a0c7c564019364be3de91a86306874c69/js/packages/cli/src/commands/upload.ts#L271).
1. To create new candy machine, that will handle assets as NFTs, use `candy-machine-cli.ts create_candy_machine`. Use `-h` option to see all possible options for this command.
1. After candy machine was successfuly created, use `candy-machine-cli.ts update_candy_machine` to update price with `-p` or `--price` option and set start sale date with `-d` or `--date` option. Date should be in format `"14 Nov 2021 00:00:00 EST"`. To enable private sale, use `-w` or `--whitelist` option and pass as second parameter file with whitelisted addresses, separated by coma. It is possible to add maximum 1000 addresess to whitelist. Use `-h` option to see all possible options for this command.

```cl
~ ts-node {path-to-metaplex}/metaplex/js/packages/cli/src/candy-machine-cli.ts upload {path-to-assets}/assets --storage arweave --env devnet --keypair {path-to-wallet}/devnet.json

~ ts-node {path-to-metaplex}/metaplex/js/packages/cli/src/candy-machine-cli.ts create_candy_machine --env devnet --keypair {path-to-wallet}/devnet.json

~ ts-node {path-to-metaplex}/metaplex/js/packages/cli/src/candy-machine-cli.ts update_candy_machine --keypair {path-to-wallet}/devnet.json --price 0.5 --date "14 Nov 2021 00:00:00 EST" --whitelist {path-to-whitelist-file}/whitelist.csv

```

### Whitelist file example

```

AfMymK6UXLMX6N8poHrbqusxtcWCcjH8rPfRTbMGQmup,
VVMtyH4sVYs9AJL2qhQwbvq9g7Cedny3D6JCeRQY3nZ,
GfyAPMytYmKVe7LJqX1TTiEHFLPYCBsCFC3MFikA24qe,
2pHvfroXp4fKp5JE8HKxgwSEVBpA5f85LHwbN415t972,
GdawPh99KPfN9GWaHjDRsJmptBfpHhSzUeFrz3jv9uzS,
2zPaA2wTzrGyVKDTZSTEA2JkZaPURKKaHqNxBW5KKSQM,
...

```

## How to Use Deployed Contract in Web App

### How to Create Whitelist

To create whitelist, follow next steps:

- First of all the array of addresses(public keys) should be created(see `whitelistAddresses` below as example).
- The maximum number of adresses that could be in array is 1000. This is predefined constant by us, so if it needs to be changed, it also needs to be changed in contract. We should set it, as Solana should know upfront what size we want to store in account (see `WHITELIST_RESERVED_BYTES` below).
- The keypair(publick and secret keys) should be generatetd for new whitelist data account(see `whitelistAccount`).
- Call `anchor.Program.rpc.initMintingWhitelist()` instruction with parameters to fill in the list(see `await anchorProgram.rpc.initMintingWhitelist` example below).
  - **NOTE**: Transaction to Solana network is limited in size, so the maximum size of serialized transaction could be only `1232` byte. Thats mean that we could not put entire huge array of `whitelistAddresses` as `initMintingWhitelist` argument if its size biger than `26` elements. If there are more than `26` elements in array, take first 26 to pass as arguments in this transaction, and send other elements using `whitelistAddMultiple` instructions. See [Edit entries in whitelist](#edit-entries-in-the-whitelist) for more info.
  - The `payer` variable has `Keypair` type.
  - **NOTE**: Be sure to check `whitelistAddresses` array for duplicates adresses upfront, before passing to contract instruction.

```ts

// Example of whitelisted adresses. Try to send no more than 26 items in init whitelist instruction
const whitelistAddresses = [
    new PublicKey("C2D2HUAWMNheYZ85mEEfLPWDM8k12U6BuHERjYCW2Giy"),
    new PublicKey("U2DH2AUMWNheZY85mEELPWDM8kf12YCW2GU6BuHERjiy"),
    ...
]

const whitelistAccount = Keypair.generate(); // Whitelist account keypair

//The size of whitelist account.
const WHITELIST_RESERVED_BYTES = 8 //DISCRIMINATOR_BYTES
  + 8 // PUBLIC_KEY_BYTES
  + 4 // LENGTH_PREFIX_BYTES
  + 1000 * 32; // WHITELIST_MAX_LEN * PUBLIC_KEY_BYTES

await anchorProgram.rpc.initMintingWhitelist(
  whitelistAddresses, // array of PublicKey that should be whitelisted
  {
    accounts: {
      whitelist: whitelistAccount.publicKey, //Whitelist PubKey
      owner: payer.publicKey, //Owner pubkey
    },
    signers: [payer, whitelistAccount],
    instructions: [
      // Here the whitelist account creates before filling it in with data
      SystemProgram.createAccount({
        fromPubkey: payer.publicKey, // Payer public key
        lamports: await anchor.Program.provider.connection.getMinimumBalanceForRentExemption(WHITELIST_RESERVED_BYTES), // Calculate rent Exempt sum for this account
        newAccountPubkey: whitelistAccount.publicKey, // Pubkey for this account
        programId: anchor.Program.programId, // Candy machine program ID
        space: WHITELIST_RESERVED_BYTES // Size of whitelist accoun. Should be set here as accounts size in solana are static and uncanged, so they should be set upfront.
      })
    ]
  },
)
```

### Enable Private Sale for Candy Machine

To enable private sale for candy machine, argument `privateSaleEnabled` should be set to `true` for `anchor.Program.rpc.updateCandyMachine` and the whitelist account should be added as the last item of `remainingAccounts`. If regular sale should be available, `privateSaleEnabled` should be set to `false` for `anchor.Program.rpc.updateCandyMachine`. The private sale argument is optional, so other parts of the candy machine could be updated, and the whitelist could remain untouched.

**NOTE:** Full source code of updating candy machine with private sale enabled could be found [HERE](https://github.com/yuraolex/metaplex/blob/464c47bef081fcf5ea36f91854ea2fd7db120d45/js/packages/cli/src/candy-machine-cli.ts#L856).

```ts
const remainingAccounts = [];

//...

let privateSaleEnabled = true;

if (privateSaleEnabled) {
  remainingAccounts.push({
    pubkey: whitelistAddress,
    isWritable: false,
    isSigner: false,
  });
}

const tx = await anchorProgram.rpc.updateCandyMachine(
  lamports ? new anchor.BN(lamports) : null,
  secondsSinceEpoch ? new anchor.BN(secondsSinceEpoch) : null,
  whitelistAddress ? privateSaleEnabled : null,
  {
    accounts: {
      candyMachine,
      authority: walletKeyPair.publicKey,
    },
    remainingAccounts,
  }
);
```

### Edit entries in the whitelist

It is possible to extend range of addresses by adding addresses to whitelist or remove addresses from whitelist, but only owner of whitelist could do this. For this next instructions should be used:

1. `whitelistAdd` to add one address to the whitelist
1. `whitelistAddMultiple` to add the array of addresses to the whitelist. It is possible to add only `28` addresses in one transaction due to transaction limitation, so if array of addresses is bigger, split for smaller chanks and do this instaction multiple times. **NOTE:** the restriction number of addresses is not the same for different instructions, as it depends from number of arguments.
1. `whitelistDelete` to remove one address from the whitelist
1. `whitelistDeleteMultiple` to remove the array of addresses from the whitelist. It is possible to remove only `28` addresses in one transaction due to transaction limitation, so if array of addresses is bigger, split for smaller chanks and do this instaction multiple times. **NOTE:** the restriction number of addresses is not the same for different instructions, as it depends from number of arguments.

In `accounts` object the whitelist public key and the owner public key of whitelist should be spceified. The owner of whitelist should also sign the transaction. See examples below.

**NOTE**: Be sure to check array of adresses for duplicates upfront, before adding them to contract. Also check updront if this addresses are not in whiotelist already. To fetch whitelist addressdes use `const wh_account = await anchor.Program.account.whiteList.fetch(whitelistAccount.publicKey); const wh_addresses = wh_account.addresses;`

```ts
const whitelisted_address = new PublicKey(
  "C2DH2AUWMNheYZ85mEEfLPWDM8k12U6BuHERjYCW2Giy"
);

//To Add the new adress to whitelist
await anchorProgram.rpc.whitelistAdd(whitelisted_address, {
  accounts: {
    whitelist: whitelistAccount.pubkey,
    owner: payer.publicKey,
  },
  signers: [payer],
});

//To Remove the adress from whitelist
await anchor.Program.rpc.whitelistRemove(whitelisted_address, {
  accounts: {
    whitelist: whitelistAccount.pubkey,
    owner: payer.publicKey,
  },
  signers: [payer],
});

//To Add new adresses to whitelist
await anchorProgram.rpc.whitelistAddMultiple([whitelisted_address], {
  accounts: {
    whitelist: whitelistAccount.pubkey,
    owner: payer.publicKey,
  },
  signers: [payer],
});

//To Remove adresses from whitelist
await anchor.Program.rpc.whitelistRemoveMultiple([whitelisted_address], {
  accounts: {
    whitelist: whitelistAccount.pubkey,
    owner: payer.publicKey,
  },
  signers: [payer],
});
```

### Mint token with whitelist

When call minting instruction, and whitelist is available in candy machine, add it as a last item of `remainingAccounts`. See example below and full code example [HERE](https://github.com/yuraolex/metaplex/blob/608e1c0143fad6816279896ae08238327ae99b0c/js/packages/cli/src/commands/mint.ts#L122).

```ts
const candyMachine: any = await anchor.Program.account.candyMachine.fetch(
  candyMachineAddress
);

const remainingAccounts = [];

// Be shure, that always add whitelist as the las item to remainingAccounts array.
if (candyMachine.whitelist) {
  remainingAccounts.push({
    pubkey: candyMachine.whitelist,
    isWritable: false,
    isSigner: false,
  });
}

await anchor.Program.instruction.mintNft({
  accounts: {
    config: configAddress,
    candyMachine: candyMachineAddress,
    payer: userKeyPair.publicKey,
    //@ts-ignore
    wallet: candyMachine.wallet,
    mint: mint.publicKey,
    metadata: metadataAddress,
    masterEdition,
    mintAuthority: userKeyPair.publicKey,
    updateAuthority: userKeyPair.publicKey,
    tokenMetadataProgram: TOKEN_METADATA_PROGRAM_ID,
    tokenProgram: TOKEN_PROGRAM_ID,
    systemProgram: SystemProgram.programId,
    rent: anchor.web3.SYSVAR_RENT_PUBKEY,
    clock: anchor.web3.SYSVAR_CLOCK_PUBKEY,
  },
  remainingAccounts,
});
```

## Integrations with website

1. After creating candy machine, clone [candy-machine-mint]() example web app.
1. run `yarn install`
1. Open the `.env.example` file. And change enviorment variables based on cache file:
   1. REACT_APP_CANDY_MACHINE_CONFIG=HLJVYSyF9shHXvsFXXEXhZzC9tTzG6cpFr4gn1MhzsZN - this could be found in `metaplex/.cache/{cluster-name}-temp.json` in object `{"program":{"uuid":"HLJVYS","config":"HLJVYSyF9shHXvsFXXEXhZzC9tTzG6cpFr4gn1MhzsZN"}, ..`.
   1. REACT_APP_CANDY_MACHINE_ID=DNCVemRhoVRTr8TpWPxnRaoPMTMTEmzWYDwn7g3YQE3j - this could be found in `metaplex/.cache/{cluster-name}-temp.json` in field `...,"candyMachineAddress":"DNCVemRhoVRTr8TpWPxnRaoPMTMTEmzWYDwn7g3YQE3j",...`
   1. REACT_APP_TREASURY_ADDRESS=9YziGgwWcT...VopuZkiPKw7y - this is publicc address of your wallet, that is authority for candy machine
   1. REACT_APP_CANDY_START_DATE=1640235600 - start date of candy machine, could be found in `metaplex/.cache/{cluster-name}-temp.json` in field `...,"startDate":1640235600}`
1. Than run `yarn start` to start website for minting.
