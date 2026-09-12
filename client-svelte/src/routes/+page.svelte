<script lang="ts">
	import { onMount } from 'svelte';
	import Join from '$lib/menus/Join.svelte';
	import Answer from '$lib/menus/Answer.svelte';
	import AnswerWait from '$lib/menus/AnswerWait.svelte';
	import Guess from '$lib/menus/Guess.svelte';
	import GuessWait from '$lib/menus/GuessWait.svelte';
	import Results from '$lib/menus/Results.svelte';

	import { deleteGame, deletePlayerFromGame, getGame } from '$lib/functions/requests';
	import Button from '$lib/Button.svelte';

	let game_state: string | null;

	let production_url: string = 'https://weight-inquiries.onrender.com/api/v1/game/';
	let test_url: string = 'http://127.0.0.1:8172/api/v1/game/';

	function setGameState(new_state: string) {
		localStorage.setItem('game_state', new_state);
		game_state = new_state;
	}

	onMount(() => {
		if (!localStorage.getItem('game_state')) {
			setGameState('join');
		} else {
			game_state = localStorage.getItem('game_state');
		}
		if (window.location.href == 'http://localhost:5173/') {
			localStorage.setItem('base_server_path', test_url);
		} else {
			localStorage.setItem('base_server_path', production_url);
		}
		if (localStorage.getItem('game_name')) {
			const response: Promise<Response> = getGame(localStorage.getItem('game_name'));
			response.then((response) => {
				if (!response.ok) {
					setGameState('join');
				}
			}) 
		}
	});

	let other_players: Array<string> = [];

	$: if (game_state == 'answer') {
		readOtherPlayers();
	}

	async function readOtherPlayers() {
		getGame(localStorage.getItem('game_name'))
			.then((response) => response.json())
			.then((data) => {
				other_players = data.players.filter(
					(player: string) => player != localStorage.getItem('name')
				);
			})
			.catch(() => {
				other_players = [];
			});
	}

	function onLeave() {
		if (confirm('Do you really want to leave the game?') == true) {
			const response: Promise<Response> = deletePlayerFromGame(
				localStorage.getItem('game_name'),
				localStorage.getItem('name')
			);
			response.then((response) => {
				if (response.ok) {
					setGameState('join');
				} else {
				}
			});
		}
	}

	function onKick(player_to_kick: string) {
		if (confirm('Do you really what to kick ' + player_to_kick + '?') == true) {
			const response: Promise<Response> = deletePlayerFromGame(
				localStorage.getItem('game_name'),
				player_to_kick
			);
			response.then(() => readOtherPlayers());
		}
	}

	function onEndGame() {
		if (confirm('Do you want to end the game for everybody?')) {
			const response: Promise<Response> = deleteGame(localStorage.getItem('game_name'));
			setGameState('join');
		}
	}
</script>

<main>
	<h1>
		weighty inquiries
	</h1>
	{#if game_state == 'join'}
		<Join {setGameState} />
	{:else if game_state == 'answer'}
		<Answer
			{setGameState}
			name={localStorage.getItem('name')}
			game_name={localStorage.getItem('game_name')}
		/>
	{:else if game_state == 'answer_wait'}
		<AnswerWait {setGameState} game_name={localStorage.getItem('game_name')} />
	{:else if game_state == 'guess'}
		<Guess
			{setGameState}
			name={localStorage.getItem('name')}
			game_name={localStorage.getItem('game_name')}
		/>
	{:else if game_state == 'guess_wait'}
		<GuessWait {setGameState} game_name={localStorage.getItem('game_name')} />
	{:else if game_state == 'results'}
		<Results
			{setGameState}
			name={localStorage.getItem('name')}
			game_name={localStorage.getItem('game_name')}
		/>
	{/if}
	<div style="padding: 50px;">

	</div>
	{#if game_state == 'answer'}
		<div>
			<Button text="Leave Game" onClick={onLeave} />
		</div>
		{#if other_players.length > 0}
			<div class="kick-label">tap a player to kick them</div>
			<div class="kick-list">
				{#each other_players as player (player)}
					<button class="kick-name shadow" on:click={() => onKick(player)}>
						{player}
					</button>
				{/each}
			</div>
		{/if}
	{/if}
	{#if game_state != 'join'}
		<div>
			<Button text="End Game" onClick={onEndGame} />
		</div>
	{/if}
</main>

<style>
	@import '../app.css';

	.kick-label {
		font-size: 14px;
		opacity: 0.7;
		padding: 10px 0 0 0;
	}
	.kick-list {
		display: flex;
		flex-wrap: wrap;
		justify-content: center;
		gap: 8px;
		padding: 8px;
	}
	.kick-name {
		font-family: inherit;
		font-size: 16px;
		color: inherit;
		text-transform: uppercase;
		padding: 10px;
		border: none;
		border-radius: 5px;
		background-color: #387b96;
		cursor: pointer;
		transition-duration: 0.2s;
	}
	.kick-name::before {
		content: '✕ ';
	}
	.kick-name:hover {
		background-color: #e86a6a;
	}
</style>
