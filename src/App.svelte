<script lang="ts">
	import { onMount } from "svelte";
	import * as idl from "./idl/gm_solana.json";
	import type { GmSolana } from "./types/gm_solana";
	import { Connection, PublicKey, clusterApiUrl } from "@solana/web3.js";
	import { Idl, Program, Provider, web3 } from "@project-serum/anchor";
	const { SystemProgram, Keypair } = web3;
	const programID = new PublicKey(idl.metadata.address);
	const network = clusterApiUrl('devnet');
	const connection = new Connection(network, "confirmed");
	const getProvider = () => {
		const provider = new Provider(connection, wallet, {
		preflightCommitment: "confirmed",
		});
		return provider;
	};
	const getProgram = () => {
		const program = new Program(
		idl as Idl,
		programID,
		getProvider()
		) as Program<GmSolana>;
		return program;
	};
	// ======== APPLICATION STATE ========
  
	let wallet: any;
	let account = "";
	let baseAccountPublicKey: PublicKey;
	let baseAccountPublicKeyInput = ""; // UI state used for the in
	let gmList = [];
	let gmMessage = "";
  
	// reactively log the wallet connection when account state changes,
	// if you don't know what this is, check out https://svelte.dev/tutorial/reactive-declarations
	$: account && console.log(`Connected to wallet: ${account}`);
  
	// ======== PAGE LOAD CHECKS ========
  
	const onLoad = async () => {
	  const { solana } = window as any;
	  wallet = solana;
  
	  // set up handlers for wallet events
	  wallet.on("connect", () => (account = wallet.publicKey.toString()));
	  wallet.on("disconnect", () => (account = ""));
  
	  // eagerly connect wallet if the user already has connected before, otherwise do nothing
	  const resp = await wallet.connect({ onlyIfTrusted: true });
	};
  
	// life cycle hook for when the component is mounted
	onMount(() => {
	  // run the onLoad function when the page completes loading
	  window.addEventListener("load", onLoad);
  
	  // return a cleanup function to remove the event listener to avoid memory leaks when the page unloads
	  return () => window.removeEventListener("load", onLoad);
	});
  
	// ======== CONNECT WALLET ========
	const handleConnectWallet = async () => {
	  const resp = await wallet.connect();
	};
	// ======== INITIALIZE WALLET ========
	const initializeAccount = async () => {
		const provider = getProvider();
		const program = getProgram();
		const _baseAccount = Keypair.generate();
		Keypair;

		await program.rpc.initialize({
		accounts: {
			baseAccount: _baseAccount.publicKey,
			user: provider.wallet.publicKey,
			systemProgram: SystemProgram.programId,
		},
		signers: [_baseAccount],
		});
		baseAccountPublicKey = _baseAccount.publicKey;
		console.log("New BaseAccount:", baseAccountPublicKey.toString());
		await getGmList(); // first fetch
	};

  // alternative to initializeAccount(), loadAccount() allows you to pick up a previously created baseAccount
  // so we can share the same "gm Solana" instance!
	const loadAccount = async () => {
		baseAccountPublicKey = new PublicKey(baseAccountPublicKeyInput);
		console.log("Loaded BaseAccount:", baseAccountPublicKey.toString());
		await getGmList(); // first fetch
	};
	  // ======== PROGRAM INTERACTION ========

  // interacts with our program and updates local the gm list
  const getGmList = async () => {
    const program = getProgram();
    const account = await program.account.baseAccount.fetch(
      baseAccountPublicKey
    );

    console.log("Got the account", account);
    gmList = account.gmList as any[];
  };

  // interacts with our program and submits a new gm message
  const sayGm = async () => {
    const provider = getProvider();
    const program = getProgram();

    await program.rpc.sayGm(gmMessage, {
      accounts: {
        baseAccount: baseAccountPublicKey,
        user: provider.wallet.publicKey,
      },
      // if we don't supply a signer, it will try to use the connected wallet by default
    });
    console.log("gm successfully sent", gmMessage);
    gmMessage = ""; // clears the input field

    await getGmList(); // updates the local gm list
  };

  $: console.log("gmList:", gmList); // just some extra logging when the gm list changes
  </script>
  
  <main>
	  <div class="top">
		<h1>vartalap </h1>
	
		<!-- Conditionally render the user account, connect button, or just a warning -->
		{#if account}
		<h3>Your wallet:</h3>
		<p>{account}</p>
		{:else if wallet} {#if wallet.isPhantom}
		<h2>Phantom Wallet found!</h2>
		<button on:click="{handleConnectWallet}">Connect wallet</button>
		{:else}
		<h2>Solana wallet found but not supported.</h2>
		{/if} {:else}
		<h2>Solana wallet not found.</h2>
		{/if}
	</div>

	<!-- initialize -->
	<div class="ini">
	{#if account}
		{#if !baseAccountPublicKey}
		<button on:click={initializeAccount}>Initialize account</button>
		or
		<input
			type="text"
			placeholder="use existing account..."
			bind:value={baseAccountPublicKeyInput}
		/>
		<button on:click={loadAccount}>Load</button>
		{:else}
		Using gm solana base account: {baseAccountPublicKey.toString()}
		{/if}
	{/if}
	</div>
  <!-- Gm stuf -->
  <div class="gm-stuff">
  {#if baseAccountPublicKey}
	<div>
		<h3>Thread list:</h3>
		<ul>
		{#each gmList as gm}
			<li>
			<b>{gm.message}</b>, said {gm.user.toString().slice(0, 6)}... at {new Date(
				gm.timestamp.toNumber() * 1000
			).toLocaleTimeString()}
			</li>
		{/each}
		</ul>
		<button on:click={getGmList}>Refresh Thread!</button>
	</div>

	<div>
		<h3>Add to Thread:</h3>
		<input
		type="text"
		placeholder="write something..."
		bind:value={gmMessage}
		/>
		<button on:click={sayGm} disabled={!gmMessage}>Say to vartalap!</button>
	</div>
	{/if}

  </div>
  </main>
  <footer>
		<div class="main-con" style="background: black; color:wheat; text-align:center; padding: 2% 0% ;">
			Built by <a href="https://twitter.com/chidam333" style="color:green;" target="_blank">@chidam333</a> (Special thanks to <a href="https://twitter.com/0xMuse" style="color:green;" target="_blank">@0xMuse</a>)
		</div>
  </footer>
  <style>
	main {
      position: relative;
	  padding: 1em;
	  margin: 0 auto;
	  background: linear-gradient(33deg,rgb(226, 250, 122) 70% ,wheat);
	  min-height: 500px;
	}
	.top {
		text-align: center;
	}
	.top button{
		color: wheat;
		text-transform: lowercase;
		background: rgb(54, 134, 0);
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
		font-weight: 400;
		border-radius: 12px;

	}
	.top button:hover{
		background: black;
		border-radius: 17px;
	}

    .ini button{
		background: rgb(240, 117, 2);
		color: black;
		box-shadow: 2px 2px black;
	}
	.ini button:hover{
		background: black;
		color:rgb(240, 117, 2);
		box-shadow: 5px 5px 5px rgb(97, 65, 6);
	}
	.gm-stuff button:hover{
		background: black;
		color:rgb(240, 117, 2);
		box-shadow: 5px 5px 5px rgb(97, 65, 6);
	}
	.gm-stuff button{
		background: rgb(240, 117, 2);
		color: black;
		box-shadow: 2px 2px black;
	}
	h1 {
	  color: #bb600a;
	  font-size: 4em;
	  font-weight: 300;
	}
  
	@media (min-width: 640px) {
	  main {
		max-width: none;
	  }
	}
  </style>
  