<script lang="ts">
	import { page } from '$app/stores';
	import Button from '$lib/components/ui/button/button.svelte';
	import { websocket_api } from 'schema-js';
	import type { Readable } from 'svelte/store';
	import { Star } from 'lucide-svelte';

	export let market: Readable<websocket_api.IMarket>;
	$: marketIdParam = Number($page.params.id);
	$: closed = $market.closed;

	let starred = localStorage.getItem(`is_starred_${Number($market.id)}`) === 'true';

	function handleStarClick() {
		localStorage.setItem(`is_starred_${Number($market.id)}`, !starred ? 'true' : 'false');
		starred = !starred;
	}

	let isHovering = false;

	let displayTransactionIdBindable: number[] = [];

	$: displayTransactionId = market.hasFullHistory ? displayTransactionIdBindable[0] : undefined;

	$: orders =
		displayTransactionId === undefined
			? (market.orders || []).filter((o) => Number(o.size) !== 0)
			: (market.orders || [])
					.filter((o) => o.transactionId <= displayTransactionId)
					.map((o) => {
						let size = o.sizes?.length
							? o.sizes.findLast((s) => s.transactionId <= displayTransactionId)!.size
							: o.size;
						return { ...o, size };
					})
					.filter((o) => Number(o.size) !== 0);
	$: bids = orders.filter((order) => order.side === websocket_api.Side.BID);
	$: bids.sort((a, b) => Number(b.price) - Number(a.price));
	$: offers = orders.filter((order) => order.side === websocket_api.Side.OFFER);
	$: offers.sort((a, b) => Number(a.price) - Number(b.price));
	$: midPrice = bids[0]
		? offers[0]
			? ((Number(bids[0].price) + Number(offers[0].price)) / 2).toFixed(2)
			: bids[0].price
		: offers[0]
			? offers[0].price
			: '';
	$: console.log({ midPrice });
</script>

<li
	class:order-2={!closed && starred}
	class:order-3={!closed && !starred}
	class:order-5={closed && starred}
	class:order-6={closed && !starred}
	class="flex items-center gap-2"
>
	<button
		on:click={handleStarClick}
		on:mouseenter={() => (isHovering = true)}
		on:mouseleave={() => (isHovering = false)}
		class="mt-1 inline rounded-full p-1 focus:outline-none"
		aria-label={starred ? 'Unstar market' : 'Star market'}
	>
		<Star
			color={starred || isHovering ? 'gold' : 'slategray'}
			fill={starred ? (isHovering ? 'none' : 'gold') : 'none'}
			size="20"
		/>
	</button>

	{#if marketIdParam === $market.id}
		<span>
			<Button
				class="inline w-full whitespace-normal px-0 text-start text-lg"
				variant="link"
				disabled
			>
				{$market.name}
				{midPrice}
			</Button>
		</span>
	{:else}
		<a href="/market/{$market.id}">
			<Button class="inline whitespace-normal px-0 text-start text-lg" variant="link">
				{$market.name}
				{midPrice}
			</Button>
		</a>
	{/if}
</li>
