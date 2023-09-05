<script lang="ts">
	import { WalletReadyState, isWalletAdapterCompatibleStandardWallet } from '@solana/wallet-adapter-base';
	import { StandardWalletAdapter } from '@solana/wallet-standard-wallet-adapter-base';
	import { getWallets } from '@wallet-standard/app';
	import { initialize } from '@aztemi/svelte-on-solana-wallet-adapter-core';
	import type { Adapter, WalletError } from '@solana/wallet-adapter-base';

	export let localStorageKey: string,
		wallets: Adapter[] = [],
		autoConnect = false,
		onError = (error: WalletError) => console.error(error);

	$: {
		if (wallets.length) {
			// get installed wallets compatible with the standard
			const { get } = getWallets();
			const standardWallets = get()
				.filter(isWalletAdapterCompatibleStandardWallet)
				.map((wallet) => new StandardWalletAdapter({ wallet }));

			// filter out non standard wallets
			const nonStandardWallets = wallets.filter(
				(wallet) => !standardWallets.some(({ name }) => wallet.name === name)
			);

			// merge, sort and initialize wallets store
			const allWallets = [...standardWallets, ...nonStandardWallets];
			allWallets.sort(installedAdaptersFirst);
			initialize({ wallets: allWallets, autoConnect, localStorageKey, onError });
		}
	}

	function installedAdaptersFirst(a: Adapter, b: Adapter): number {
		let sort: number = 0;
		const isInstalled = (c: Adapter) => c.readyState === WalletReadyState.Installed;

		if (isInstalled(a) && !isInstalled(b)) sort = -1;
		if (!isInstalled(a) && isInstalled(b)) sort = 1;
		return sort;
	}
</script>

<svelte:head>
	<script>
		window.global = window;
	</script>
</svelte:head>
