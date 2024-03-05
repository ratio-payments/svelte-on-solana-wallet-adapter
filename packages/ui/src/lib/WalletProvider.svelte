<script lang="ts">
	import { onMount } from 'svelte';
	import { WalletReadyState, isWalletAdapterCompatibleStandardWallet } from '@solana/wallet-adapter-base';
	import { StandardWalletAdapter } from '@solana/wallet-standard-wallet-adapter-base';
	import {
		createDefaultAddressSelector,
		createDefaultAuthorizationResultCache,
		createDefaultWalletNotFoundHandler,
		SolanaMobileWalletAdapter,
		SolanaMobileWalletAdapterWalletName
	} from '@solana-mobile/wallet-adapter-mobile';
	import { getChainForEndpoint } from '@solana/wallet-standard-util';
	import { getWallets } from '@wallet-standard/app';
	import { initialize } from '@aztemi/svelte-on-solana-wallet-adapter-core';
	import type { Adapter, WalletError } from '@solana/wallet-adapter-base';
	import { workSpace } from './workSpace';

	export let localStorageKey: string,
		wallets: Adapter[] = [],
		autoConnect = false,
		onError = (error: WalletError) => console.error(error);

	$: wallets.length && updateWallets();

	function updateWallets() {
		// get installed wallets compatible with the standard
		const { get } = getWallets();
		const standardWallets = get()
			.filter(isWalletAdapterCompatibleStandardWallet)
			.map((wallet) => new StandardWalletAdapter({ wallet }));

		// filter out non standard wallets
		const nonStandardWallets = wallets.filter(
			(wallet) => !standardWallets.some(({ name }) => wallet.name === name)
		);

		const allWallets = [...standardWallets, ...nonStandardWallets];

		// merge standard mobile wallet if available
		const mobileWallet = getMobileWallet(allWallets);
		if (mobileWallet) allWallets.unshift(mobileWallet);

		// sort and initialize wallets store
		allWallets.sort(installedAdaptersFirst);
		initialize({ wallets: allWallets, autoConnect, localStorageKey, onError });
	}

	function installedAdaptersFirst(a: Adapter, b: Adapter): number {
		let sort: number = 0;
		const isInstalled = (c: Adapter) => c.readyState === WalletReadyState.Installed;

		if (isInstalled(a) && !isInstalled(b)) sort = -1;
		if (!isInstalled(a) && isInstalled(b)) sort = 1;
		return sort;
	}

	function getMobileWallet(wallets: Adapter[]) {
		/**
		 * Return null if Mobile wallet adapter is already in the list or if ReadyState is Installed.
		 *
		 * There are only two ways a browser extension adapter should be able to reach `Installed` status:
		 *   1. Its browser extension is installed.
		 *   2. The app is running on a mobile wallet's in-app browser.
		 * In either case, we consider the environment to be desktop-like and not 'mobile'.
		 */
		if (
			wallets.some(
				({ name, readyState }) =>
					name === SolanaMobileWalletAdapterWalletName || readyState === WalletReadyState.Installed
			)
		) {
			return null;
		}

		const ua = globalThis.navigator?.userAgent ?? null;

		const isOnAndroid = (ua: string) => /android/i.test(ua);

		const isInWebView = (ua: string) =>
			/(WebView|Version\/.+(Chrome)\/(\d+)\.(\d+)\.(\d+)\.(\d+)|; wv\).+(Chrome)\/(\d+)\.(\d+)\.(\d+)\.(\d+))/i.test(
				ua
			);

		const getUriForAppIdentity = () => {
			const { location } = globalThis;
			if (location) return `${location.protocol}//${location.host}`;
		};

		/**
		 * Return Mobile Wallet Adapter object if we are running
		 * on a device that supports MWA like Android
		 * and we are *not* running in a WebView.
		 */
		if (ua && isOnAndroid(ua) && !isInWebView(ua)) {
			return new SolanaMobileWalletAdapter({
				addressSelector: createDefaultAddressSelector(),
				appIdentity: {
					uri: getUriForAppIdentity()
				},
				authorizationResultCache: createDefaultAuthorizationResultCache(),
				chain: getChainForEndpoint($workSpace?.connection?.rpcEndpoint || ''),
				onWalletNotFound: createDefaultWalletNotFoundHandler()
			});
		}
	}

	onMount(() => {
		const { on } = getWallets();
		const removeRegisterListener = on('register', updateWallets);
		const removeUnregisterListener = on('unregister', updateWallets);
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
