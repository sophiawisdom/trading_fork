<script lang="ts">
	import { actingAs, portfolio, sendClientMessage, users } from '$lib/api';
	import { user } from '$lib/auth';
	import { Slider } from '$lib/components/ui/slider';
	import { cn } from '$lib/utils';
	import { HistoryIcon, LineChartIcon } from 'lucide-svelte';
	import { websocket_api } from 'schema-js';
	import FlexNumber from './flexNumber.svelte';
	import CreateOrder from './forms/createOrder.svelte';
	import SettleMarket from './forms/settleMarket.svelte';
	import PriceChart from './priceChart.svelte';
	import Button from './ui/button/button.svelte';
	import * as Table from './ui/table';
	import Toggle from './ui/toggle/toggle.svelte';
	import { markets } from '$lib/api';

	export let market: websocket_api.IMarket;
	let showChart = true;
	let displayTransactionIdBindable: number[] = [];

	$: displayTransactionId = market.hasFullHistory ? displayTransactionIdBindable[0] : undefined;

	$: maxTransactionId = Math.max(
		...(market.orders?.map((o) => o.transactionId) || []),
		...(market.trades?.map((t) => t.transactionId) || []),
		market.transactionId
	);

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
	$: trades =
		displayTransactionId === undefined
			? market.trades || []
			: market.trades?.filter((t) => t.transactionId <= displayTransactionId) || [];
	$: bids = orders.filter((order) => order.side === websocket_api.Side.BID);
	$: bids.sort((a, b) => Number(b.price) - Number(a.price));
	$: offers = orders.filter((order) => order.side === websocket_api.Side.OFFER);
	$: offers.sort((a, b) => Number(a.price) - Number(b.price));
	$: position =
		Number($portfolio?.marketExposures?.find((me) => me.marketId === market.id)?.position) || 0;
	$: lastPrice = trades[trades.length - 1]?.price || '';
	$: midPrice = bids[0]
		? offers[0]
			? ((Number(bids[0].price) + Number(offers[0].price)) / 2).toFixed(2)
			: bids[0].price
		: offers[0]
			? offers[0].price
			: '';

	const cancelOrder = (id: number) => {
		sendClientMessage({ cancelOrder: { id } });
	};

	var important_markets = [34, 35, 36, 37, 38, 39];
	let names = {
		34: "A", 35: "B", 36: "C",
		37: "etf", 38: "bond", 39: "dice"};

		let mins_maxes = {
		34: [0, 240], 35: [0, 240], 36: [0, 240],
		37: [1, 20], 38: [1, 20], 39: [0, 360]
	};
	/*
	var fairs = new Map(); // [fair, interval, time set]
	var important_markets = [34, 35, 36, 37, 38, 39];
	let mins_maxes = {
		34: [0, 20], 35: [1, 20], 36: [4, 80],
		37: [1, 20], 38: [1, 20], 39: [4, 80]
	};
	let names = {31: "min", 32: "max", 33: "sum"};
	for (let i = 0; i < important_markets.length; i++) {
		let market = important_markets[i];
		fairs.set(market, [mins_maxes[market][0], mins_maxes[market][1], new Date()]);
	}
	*/

	function getMarket(id: number): websocket_api.Portfolio.IMarketExposure | null {
		let exposures = ($portfolio?.marketExposures ?? []);
		for (let i = 0; i < exposures.length; i++) {
			let exposure = exposures[i];
			if (exposure.marketId == id) {
				return exposure;
			}
		}
		return null;
	}

		let trades = market.trades ?? [];
		let pos_exp = new Map();
		let neg_exp = new Map();
		let traders = new Set();
		trades.forEach(trade => {
			if (!pos_exp.get(trade.buyerId)) {
				pos_exp.set(trade.buyerId, 0);
			}
			if (!neg_exp.get(trade.sellerId)) {
				neg_exp.set(trade.sellerId, 0);
			}

			pos_exp.set(trade.buyerId, pos_exp.get(trade.buyerId) + parseFloat(trade.size ?? "0")*100);
			neg_exp.set(trade.sellerId, neg_exp.get(trade.sellerId) + parseFloat(trade.size ?? "0")*100);
			traders.add(trade.buyerId);
			traders.add(trade.sellerId);
		});
		let sorted_traders = Array.from(traders);
		sorted_traders.sort((a, b) => -(pos_exp.get(a) || 0) - (neg_exp.get(a) || 0) + (pos_exp.get(b) || 0) + (neg_exp.get(b) || 0));
</script>
<table style="width: 500px;">
	<caption> OUR POSITIONS </caption>
	<thead>
	{#each important_markets as marketId}
<th style="font-size:30px;"> <a color="blue" href="/market/{marketId}"> {names[marketId]} </a> </th>
{/each}
</thead>
<tbody>
<tr>
{#each important_markets as marketId}
{@const exposure = getMarket(marketId)}
<td style="text-align: center;"> {exposure ? exposure.position : 0} </td>
{/each}
</tr>
</tbody>
</table>


<div class="mb-4 flex justify-between">
	<div class="mb-4">
		<h1 class="text-2xl font-bold">{market.name}.</h1>
		<p> Min: {market.minSettlement}. Max: {market.maxSettlement} </p>
	</div>

	{#if false}
	<table width=200>
		<thead>
			<th> user </th>
			<th> bought </th>
			<th> sold </th>
		</thead>
		<tbody>
		{#each sorted_traders as trader}
		<tr>
			<td> {$users?.get(trader).name} </td>
			<td> {((pos_exp.get(trader) ?? 0) / 100).toFixed(2)} </td>
			<td> {((neg_exp.get(trader) ?? 0) / 100).toFixed(2)} </td>
		</tr>
		{/each}
		</tbody>
	</table>
	{/if}
	<!--
	<div>
		<Table.Root class="w-auto text-center font-bold">
			<Table.Header>
				<Table.Row>
					<Table.Head>
						<Toggle
							on:click={() => {
								if (displayTransactionIdBindable.length) {
									displayTransactionIdBindable = [];
								} else {
									displayTransactionIdBindable = [maxTransactionId];
									sendClientMessage({ upgradeMarketData: { marketId: market.id } });
								}
							}}
							variant="outline"
						>
							<HistoryIcon />
						</Toggle>
					</Table.Head>
					<Table.Head>
						<Toggle bind:pressed={showChart} variant="outline">
							<LineChartIcon />
						</Toggle>
					</Table.Head>
					<Table.Head class="text-center">Min Settlement</Table.Head>
					<Table.Head class="text-center">Max Settlement</Table.Head>
				</Table.Row>
			</Table.Header>
			<Table.Body>
				<Table.Row>
					<Table.Cell class="p-2"></Table.Cell>
					<Table.Cell class="p-2"></Table.Cell>
					<Table.Cell class="p-2">{market.minSettlement}</Table.Cell>
					<Table.Cell class="p-2">{market.maxSettlement}</Table.Cell>
				</Table.Row>
			</Table.Body>
		</Table.Root>
	</div>
-->	
</div>

{#if market.closed}
	<p>Market settled to <em>{market.closed.settlePrice}</em></p>
{/if}
<div class="flex justify-between gap-8">
	<div>
		<h2 class="text-center text-lg font-bold">Trades</h2>
		<Table.Root>
			<Table.Header>
				<Table.Row>
					<Table.Head class="text-center">Buyer</Table.Head>
					<Table.Head class="text-center">Seller</Table.Head>
					<Table.Head class="text-center">Price</Table.Head>
					<Table.Head class="text-center">Size</Table.Head>
				</Table.Row>
			</Table.Header>
			<Table.Body>
				{#each trades.toReversed() as trade (trade.id)}
					<Table.Row class="h-8 even:bg-accent/35">
						<Table.Cell class="px-1 py-0">
							{$users.get(trade.buyerId || '')?.name?.split(' ')[0]}
						</Table.Cell>
						<Table.Cell class="px-1 py-0">
							{$users.get(trade.sellerId || '')?.name?.split(' ')[0]}
						</Table.Cell>
						<Table.Cell class="px-1 py-0">
							<FlexNumber value={trade.price || ''} />
						</Table.Cell>
						<Table.Cell class="px-1 py-0">
							<FlexNumber value={trade.size || ''} />
						</Table.Cell>
					</Table.Row>
				{/each}
			</Table.Body>
		</Table.Root>
	</div>
	<div>
		<h2 class="text-center text-lg font-bold">Orders</h2>
		<button style="margin-right: 150px;" on:click={event => {
			let max_bid_price = Math.round(market.minSettlement * 100);
			if (bids && bids.length > 0) {
				max_bid_price = Math.max(...bids.map(order => Math.round(parseFloat(order.price ?? "0")*100)));
			}
			let our_bid_price = max_bid_price + 1;
			let our_bid_string = (our_bid_price / 100).toFixed(2);
			// console.log(max_bid_price, our_bid_price, (our_bid_price / 100).toFixed(2));

			// TOCONSIDER: CANCEL OTHER BIDS WE HAVE?

			console.log("OWNERS", bids.map(bid => bid.ownerId), $actingAs);
			bids.filter(bid => bid.ownerId == $actingAs).map(bid => bid.id).forEach(bid_id => {
				sendClientMessage({ cancelOrder: { id: bid_id } });
			})

			sendClientMessage({ createOrder: { marketId: market.id, size: "1", side: websocket_api.Side.BID, price: our_bid_string} });
		}}> Bid more </button>
		<button on:click={event => {
			console.log(market);
			let min_offer_price = Math.round(market.maxSettlement * 100);
			if (offers && offers.length > 0) {
				min_offer_price = Math.min(...offers.map(order => Math.round(parseFloat(order.price ?? "0")*100)));
			}
			let our_offer_price = min_offer_price - 1;
			let our_offer_string = (our_offer_price / 100).toFixed(2);

			offers.filter(offer => offer.ownerId == $actingAs).map(offer => offer.id).forEach(offer_id => {
				sendClientMessage({ cancelOrder: { id: offer_id } });
			})

			// TOCONSIDER: CANCEL OTHER OFFERS WE HAVE?

			sendClientMessage({ createOrder: { marketId: market.id, size: "1", side: websocket_api.Side.OFFER, price: our_offer_string} });
		}}> Offer for less </button>
		<div class="flex gap-4">
			<Table.Root>
				<Table.Header>
					<Table.Row>
						<Table.Head class="text0center">Take</Table.Head>
						<Table.Head class="text-center">Owner</Table.Head>
						<Table.Head class="text-center">Size</Table.Head>
						<Table.Head class="text-center">Bid</Table.Head>
					</Table.Row>
				</Table.Header>
				<Table.Body>
					{#each bids as order (order.id)}
						<Table.Row
							class={cn(
								'h-8 even:bg-accent/35',
								order.ownerId === $actingAs && 'outline outline-2 outline-primary'
							)}
						>
							<Table.Cell class="px-1 py-0">
								{#if order.ownerId === $actingAs && displayTransactionId === undefined}
									<Button
										variant="inverted"
										class="h-6 w-6 rounded-2xl px-2"
										on:click={() => cancelOrder(order.id)}>X</Button
									>
								{:else}
								<Table.Cell class="px-1 py-0">
									<button on:click={event => {
										// price, size, side, marketid
										sendClientMessage({ createOrder: { marketId: order.marketId, size: order.size, side: websocket_api.Side.OFFER, price: order.price } });
									}}>Take</button>
								</Table.Cell>
								{/if}
							</Table.Cell>
							<Table.Cell class="px-1 py-0">
								{$users.get(order.ownerId || '')?.name?.split(' ')[0]}
							</Table.Cell>
							<Table.Cell class="px-1 py-0">
								<FlexNumber value={order.size || ''} />
							</Table.Cell>
							<Table.Cell class="px-1 py-0">
								<FlexNumber value={order.price || ''} />
							</Table.Cell>
						</Table.Row>
					{/each}
				</Table.Body>
			</Table.Root>
			<Table.Root>
				<Table.Header>
					<Table.Row>
						<Table.Head class="text-center">Offer</Table.Head>
						<Table.Head class="text-center">Size</Table.Head>
						<Table.Head class="text-center">Owner</Table.Head>
						<Table.Head class="text0center">Take</Table.Head>
					</Table.Row>
				</Table.Header>
				<Table.Body>
					{#each offers as order (order.id)}
						<Table.Row
							class={cn(
								'h-8 even:bg-accent/35',
								order.ownerId === $actingAs && 'outline outline-2 outline-primary'
							)}
						>
							<Table.Cell class="px-1 py-0">
								<FlexNumber value={order.price || ''} />
							</Table.Cell>
							<Table.Cell class="px-1 py-0">
								<FlexNumber value={order.size || ''} />
							</Table.Cell>
							<Table.Cell class="px-1 py-0">
								{$users.get(order.ownerId || '')?.name?.split(' ')[0]}
							</Table.Cell>
							<Table.Cell class="px-1 py-0">
								{#if order.ownerId === $actingAs && displayTransactionId === undefined}
									<Button
										variant="inverted"
										class="h-6 w-6 rounded-2xl px-2"
										on:click={() => cancelOrder(order.id)}>X</Button
									>
								{:else}
								<Table.Cell class="px-1 py-0">
									<button on:click={event => {
										// price, size, side, marketid
										sendClientMessage({ createOrder: { marketId: order.marketId, size: order.size, side: websocket_api.Side.BID, price: order.price } });
									}}>Take</button>
								</Table.Cell>
								{/if}
							</Table.Cell>
						</Table.Row>
					{/each}
				</Table.Body>
			</Table.Root>
		</div>
	</div>
</div>
	<div class="flex flex-col gap-4">
		{#if showChart}
			<PriceChart
				{trades}
				minSettlement={market.minSettlement}
				maxSettlement={market.maxSettlement}
			/>
		{/if}
		{#if displayTransactionId !== undefined}
			<div class="mx-4">
				<h2 class="mb-4 ml-2 text-lg">Time Slider</h2>
				<Slider
					class="mx-4"
					bind:value={displayTransactionIdBindable}
					max={maxTransactionId}
					min={market.transactionId}
					step={1}
				/>
			</div>
		{/if}
		{#if market.open || displayTransactionId !== undefined}
			<Table.Root class="font-bold">
				<Table.Header>
					<Table.Row>
						<Table.Head class="text-center">Last price</Table.Head>
						<Table.Head class="text-center">Mid price</Table.Head>
						<Table.Head class="text-center">Your Position</Table.Head>
					</Table.Row>
				</Table.Header>
				<Table.Body class="text-center">
					<Table.Row>
						<Table.Cell class="pt-2">{lastPrice}</Table.Cell>
						<Table.Cell class="pt-2">{midPrice}</Table.Cell>
						<Table.Cell class="pt-2">{Number(position.toFixed(2))}</Table.Cell>
					</Table.Row>
				</Table.Body>
			</Table.Root>
		{/if}
		<div
			class={cn(
				'flex justify-between gap-8 text-center',
				displayTransactionId !== undefined && 'min-h-screen'
			)}
		>
	{#if displayTransactionId !== undefined}
		<div class="mx-4">
			<h2 class="mb-4 ml-2 text-lg">Time Slider</h2>
			<Slider
				class="mx-4"
				bind:value={displayTransactionIdBindable}
				max={maxTransactionId}
				min={market.transactionId}
				step={1}
			/>
		</div>
	{/if}
	{#if market.open || displayTransactionId !== undefined}
		<Table.Root class="font-bold">
			<Table.Header>
				<Table.Row>
					<Table.Head class="text-center">Last price</Table.Head>
					<Table.Head class="text-center">Mid price</Table.Head>
					<Table.Head class="text-center">Your Position</Table.Head>
				</Table.Row>
			</Table.Header>
			<Table.Body class="text-center">
				<Table.Row>
					<Table.Cell class="pt-2">{lastPrice}</Table.Cell>
					<Table.Cell class="pt-2">{midPrice}</Table.Cell>
					<Table.Cell class="pt-2">{Number(position.toFixed(4))}</Table.Cell>
				</Table.Row>
			</Table.Body>
		</Table.Root>
	{/if}
	</div>
	{#if market.open && displayTransactionId === undefined}
		<div>
			<CreateOrder
				marketId={market.id}
				minSettlement={market.minSettlement}
				maxSettlement={market.maxSettlement}
			/>
			<div class="pt-8">
				<Button
					variant="inverted"
					class="w-full"
					on:click={() => sendClientMessage({ out: { marketId: market.id } })}>Clear Orders</Button
				>
			</div>
			{#if market.ownerId === $user?.id}
				<div class="pt-8">
					<SettleMarket
						id={market.id}
						name={market.name}
						minSettlement={market.minSettlement}
						maxSettlement={market.maxSettlement}
					/>
				</div>
			{/if}
		</div>
	{/if}
</div>
