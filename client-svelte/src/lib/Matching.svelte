<script lang="ts">
	import { onMount } from 'svelte';
	import { slide } from 'svelte/transition';
	import type { Picture } from '$lib/datatypes/picture';

	export let baskets: Array<{ name: string; item: string }> = [];
	export let players: Array<string> = [];
	export let pictures: Array<Picture> = [];

	let open_basket_id: number | null = null;
	let picture_mode: boolean = false;
	let server_path: string = '';

	onMount(() => {
		picture_mode = localStorage.getItem('game_mode') == 'picture';
		server_path = localStorage.getItem('base_server_path')?.replace('api/v1/game/', '') ?? '';
	});

	$: matched = baskets.filter((basket) => basket.item != '').length;
	$: unplaced = players.filter((player) => !baskets.some((basket) => basket.item == player));

	function getUrl(prompt: string) {
		const picture = (pictures ?? []).find((picture) => picture.prompt == prompt);
		return picture ? server_path + picture.url : '';
	}

	function togglePicker(basket_id: number) {
		open_basket_id = open_basket_id == basket_id ? null : basket_id;
	}

	// Tapping the name already in the box takes it back out, tapping a name that
	// is sitting in another box swaps the two, anything else just places it.
	function pick(basket_id: number, player: string) {
		const previous = baskets[basket_id].item;
		if (previous == player) {
			baskets[basket_id].item = '';
		} else {
			const held_by = baskets.findIndex((basket, i) => i != basket_id && basket.item == player);
			if (held_by != -1) {
				baskets[held_by].item = previous;
			}
			baskets[basket_id].item = player;
		}
		baskets = baskets;
		open_basket_id = null;
	}

	function clear(basket_id: number) {
		baskets[basket_id].item = '';
		baskets = baskets;
		open_basket_id = null;
	}

	function onKeydown(event: KeyboardEvent) {
		if (event.key == 'Escape') {
			open_basket_id = null;
		}
	}
</script>

<svelte:window on:keydown={onKeydown} />

<div class="matching">
	<div class="tally">{matched} / {baskets.length} matched</div>
	{#if matched == 0}
		<div class="hint">tap a box, then tap a name</div>
	{:else if unplaced.length > 0}
		<div class="hint">left to place: {unplaced.join(' · ')}</div>
	{:else}
		<div class="hint ready">everyone placed</div>
	{/if}

	{#each baskets as basket, basket_id (basket)}
		{@const url = picture_mode ? getUrl(basket.name) : ''}
		<div class="row shadow">
			<div class="prompt">
				{#if url != ''}
					<img class="prompt-picture" src={url} alt={basket.name} />
				{:else}
					{basket.name}
				{/if}
			</div>
			<div class="slot-row">
				<button
					class="slot"
					class:filled={basket.item != ''}
					class:open={open_basket_id == basket_id}
					on:click={() => togglePicker(basket_id)}
				>
					{basket.item != '' ? basket.item : 'who said this?'}
				</button>
				{#if basket.item != ''}
					<button class="clear" title="take this name back" on:click={() => clear(basket_id)}>
						✕
					</button>
				{/if}
			</div>
			{#if open_basket_id == basket_id}
				<!-- reveals on open, but closes instantly so a fast tap can never land on a
				     picker that is on its way out -->
				<div class="picker" in:slide={{ duration: 120 }}>
					{#each players as player (player)}
						<button
							class="name"
							class:chosen={basket.item == player}
							class:taken={basket.item != player && baskets.some((other) => other.item == player)}
							on:click={() => pick(basket_id, player)}
						>
							{player}
						</button>
					{/each}
				</div>
			{/if}
		</div>
	{/each}
</div>

<style>
	@import '../app.css';

	.matching {
		max-width: 460px;
		margin-inline: auto;
		padding: 0;
	}
	.tally {
		font-size: 18px;
		padding: 0;
	}
	.hint {
		font-size: 14px;
		opacity: 0.7;
		padding: 0 0 12px 0;
	}
	.hint.ready {
		opacity: 1;
		color: #13ad75;
	}
	.row {
		border-radius: 5px;
		padding: 10px;
		margin-bottom: 12px;
	}
	.prompt {
		font-size: 18px;
		padding: 0 0 8px 0;
		overflow-wrap: anywhere;
	}
	.prompt-picture {
		display: block;
		max-width: 100%;
		margin-inline: auto;
		border-radius: 5px;
	}
	.slot-row {
		display: flex;
		gap: 6px;
		padding: 0;
	}
	button {
		font-family: inherit;
		font-size: 16px;
		color: inherit;
		text-transform: uppercase;
		border-radius: 5px;
		cursor: pointer;
		/* only the hover affordances ease, placing a name recolours instantly */
		transition-property: border-color, filter;
		transition-duration: 0.2s;
	}
	.slot {
		flex: 1;
		min-height: 48px;
		padding: 12px 10px;
		border: 2px dashed rgba(127, 127, 127, 0.7);
		background-color: transparent;
	}
	.slot.filled {
		border: 2px solid transparent;
		background-color: #4fcafa;
	}
	.slot.open {
		border: 2px solid #13ad75;
	}
	.clear {
		width: 48px;
		border: 2px solid rgba(127, 127, 127, 0.7);
		background-color: transparent;
	}
	.picker {
		display: flex;
		flex-wrap: wrap;
		justify-content: center;
		gap: 8px;
		padding: 10px 0 2px 0;
	}
	.name {
		padding: 10px;
		border: none;
		background-color: #4fcafa;
	}
	.name.taken {
		background-color: #387b96;
	}
	.name.taken::before {
		content: '↔ ';
	}
	.name.chosen {
		background-color: #13ad75;
	}
	.name.chosen::before {
		content: '✓ ';
	}
	button:hover {
		filter: brightness(1.15);
	}
	.slot:hover,
	.clear:hover {
		border-color: #4fcafa;
	}
	@media (prefers-color-scheme: dark) {
		.row {
			background-color: rgba(255, 255, 255, 0.1);
		}
	}
	@media (prefers-color-scheme: light) {
		.row {
			background-color: rgba(0, 0, 0, 0.15);
		}
	}
</style>
