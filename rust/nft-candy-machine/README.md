# Candy Machine With Whitelist

## How to Build and Deploy Candy Machine

1. Clone project from [GitHub](https://github.com/yuraolex/metaplex)
1. Install Anchor enviorment. See [Installing Dependencies](https://project-serum.github.io/anchor/getting-started/installation.html), for details.
1. Run `anchor build`
1. Run `anchor deploy`
1. Run  `anchor idl init --filepath {path-to-metaplex}/metaplex/rust/target/idl/nft_candy_machine.json {candy_machine_pubkey}`
   1. Run  `anchor idl upgrade --filepath {path-to-metaplex}/metaplex/rust/target/idl/nft_candy_machine.json {candy_machine_pubkey}` if candy machine alredy uploaded.

## How to Use Deploied Contract

### How to Create Whitelist 

To create whitelist, follow next steps:
* First of all the array of addresses(public keys) should be created(see `whitelistAddresses` below as example).
* The maximum number of adresses that could be in array is 1000. This is predefined constant by us, so if it needs to be changed, it also needs to be changed in contract. We should set it, as Solana should know upfront what size we want to store in account (see `WHITELIST_RESERVED_BYTES` below). 
* The keypair(publick and secret keys) should be generatetd for new whitelist data account(see `whitelistAccount`).
* Call `anchor.Program.rpc.initMintingWhitelist()` instruction with parameters to fill in the list(see `await anchorProgram.rpc.initMintingWhitelist` example below).
  * **NOTE**, that transaction to Solana network is limited in size, so the maximum size of serialized transaction could be only `1232` byte. Thats mean that we could not put entire huge array of `whitelistAddresses` as `initMintingWhitelist` argument if its size biger than `26` elements. If there are more than `26` elements in array, take first 26 to pass as arguments in this transaction, and send other elements using `whitelistAddMultiple` instructions. See [Edit entries in whitelist](#edit-entries-in-the-whitelist) for more info.
  * The `payer` variable has `Keypair` type. 


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

To enable private sale for candy machine, and restriction, who could buy NFTs from it, the whitelist address(public key) should be passed as argument to `anchor.Program.rpc.initializeCandyMachine`. If regular sale should be available, create new candy machine with `null` whitelist address argument. (see example below).

**NOTE:** Full source code of initializing candy machine could be found [HERE](https://github.com/yuraolex/metaplex/blob/608e1c0143fad6816279896ae08238327ae99b0c/js/packages/cli/src/candy-machine-cli.ts#L776).

```ts

let isPrivateSale = true;

await anchor.Program.rpc.initializeCandyMachine(
  bump,
  {
    uuid: cacheContent.program.uuid,
    price: new anchor.BN(parsedPrice),
    itemsAvailable: new anchor.BN(Object.keys(cacheContent.items).length),
    goLiveDate: null,
  },
  isPrivateSale ? whitelistAccount.pubkey : null, // The third parameter should be whitelist address. If null, candymachine will ignore whitelist.
  {
    accounts: {
      candyMachine,
      wallet,
      config: config,
      authority: walletKeyPair.publicKey,
      payer: walletKeyPair.publicKey,
      systemProgram: anchor.web3.SystemProgram.programId,
      rent: anchor.web3.SYSVAR_RENT_PUBKEY,
    },
    signers: [],
    remainingAccounts,
  },
);
```

### Edit entries in the whitelist

It is possible to extend range of addresses by adding addresses to whitelist or remove addresses from whitelist, but only owner of whitelist could do this. For this next instructions should be used:

1. `whitelistAdd` to add one address to the whitelist
1. `whitelistAddMultiple` to add the array of addresses to the whitelist. It is possible to add only `28` addresses in one transaction due to transaction limitation, so if array of addresses is bigger, split for smaller chanks and do this instaction multiple times. **NOTE:** the restriction number of addresses is not the same for different instructions, as it depends from number of arguments.
1. `whitelistDelete` to remove one address from the whitelist
1. `whitelistDeleteMultiple` to remove the array of addresses from the whitelist. It is possible to remove only `28` addresses in one transaction due to transaction limitation, so if array of addresses is bigger, split for smaller chanks and do this instaction multiple times.  **NOTE:** the restriction number of addresses is not the same for different instructions, as it depends from number of arguments.

In `accounts` object the whitelist public key and the owner public key of whitelist should be spceified. The owner of whitelist should also sign the transaction. See examples below.

```ts

const whitelisted_address = new PublicKey("C2DH2AUWMNheYZ85mEEfLPWDM8k12U6BuHERjYCW2Giy");

//To Add the new adress to whitelist
await anchorProgram.rpc.whitelistAdd(
  whitelisted_address,
  {
    accounts: {
      whitelist: whitelistAccount.pubkey,
      owner: payer.publicKey,
    },
    signers: [payer]
  }
);

//To Remove the adress from whitelist
await anchor.Program.rpc.whitelistRemove(
  whitelisted_address,
  {
    accounts: {
      whitelist: whitelistAccount.pubkey,
      owner: payer.publicKey,
    },
    signers: [payer]
  }
);

//To Add new adresses to whitelist
await anchorProgram.rpc.whitelistAddMultiple(
  [
    whitelisted_address,
  ],
  {
    accounts: {
      whitelist: whitelistAccount.pubkey,
      owner: payer.publicKey,
    },
    signers: [payer]
  }
);

//To Remove adresses from whitelist
await anchor.Program.rpc.whitelistRemoveMultiple(
  [
    whitelisted_address,
  ],
  {
    accounts: {
      whitelist: whitelistAccount.pubkey,
      owner: payer.publicKey,
    },
    signers: [payer]
  }
);

```

### Mint token with whitelist

When call minting instruction, and whitelist is available in candy machine, add it as a last item of `remainingAccounts`. See example below and full code example [HERE](https://github.com/yuraolex/metaplex/blob/608e1c0143fad6816279896ae08238327ae99b0c/js/packages/cli/src/commands/mint.ts#L122).

```ts

const candyMachine: any = await anchor.Program.account.candyMachine.fetch(
    candyMachineAddress,
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