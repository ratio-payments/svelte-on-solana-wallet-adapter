<script lang="ts">
	import { onMount } from 'svelte';
	import { WalletReadyState } from '@solana/wallet-adapter-base';
	import { isWalletAdapterCompatibleWallet, StandardWalletAdapter } from '@solana/wallet-standard-wallet-adapter-base';
	import { getWallets } from '@wallet-standard/app';
	import { initialize } from '@aztemi/svelte-on-solana-wallet-adapter-core';
	import type { Adapter, WalletError } from '@solana/wallet-adapter-base';
	import type { Wallet } from '@wallet-standard/base';

	export let localStorageKey: string,
		wallets: Adapter[] = [],
		autoConnect = false,
		onError = (error: WalletError) => console.error(error);

	let nonStandardWallets: Adapter[] = [];
	let standardWallets: Adapter[] = [];

	$: {
		const allWallets = [...standardWallets, ...nonStandardWallets];
		allWallets.sort(installedAdaptersFirst);
		initialize({ wallets: allWallets, autoConnect, localStorageKey, onError });
	}

	$: {
		nonStandardWallets = wallets.filter(
			(wallet) => !standardWallets.some(({ name }) => wallet.name === name)
		);
	}

	function installedAdaptersFirst(a: Adapter, b: Adapter): number {
		let sort: number = 0;
		const isInstalled = (c: Adapter) => c.readyState === WalletReadyState.Installed;

		if (isInstalled(a) && !isInstalled(b)) sort = -1;
		if (!isInstalled(a) && isInstalled(b)) sort = 1;
		return sort;
	}

	function updateStandardWallets(wallets: Wallet[]) {
		standardWallets = wallets
			.filter(isWalletAdapterCompatibleWallet)
			.map((wallet) => new StandardWalletAdapter({ wallet }));
	}

	onMount(() => {
		const { get, on } = getWallets();
		const removeRegisterListener = on('register', () => updateStandardWallets(get()));
		const removeUnregisterListener = on('unregister', () => updateStandardWallets(get()));
		updateStandardWallets(get());

		return () => {
			removeRegisterListener();
			removeUnregisterListener();
		};
	});
</script>

<svelte:head>
	<script>
		window.global = window;
	</script>
</svelte:head>
