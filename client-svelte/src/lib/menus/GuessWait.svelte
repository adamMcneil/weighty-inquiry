<script lang="ts">
	import type { Round } from '$lib/datatypes/round';
	import { Answer } from '$lib/datatypes/answer';
	import { onMount } from 'svelte';
	import { getGame } from '$lib/functions/requests';
	import { sleep } from '$lib/functions/helper';
	import PlayerList from '$lib/PlayerList.svelte';
	import AnswerWait from './AnswerWait.svelte';

	export let setGameState: (new_state: string) => void;
	export let game_name: string | null;

	let question: string;
	let players: Array<string> = [];
	let waiting_for: Array<string> = [];
	let round_count: number;

	async function readGame() {
		getGame(game_name)
			.then((response) => response.json())
			.then((data) => {
				round_count = data.rounds.length
                question = data.rounds[data.rounds.length - 1].question;
				players = data.players;
                if (data.rounds[data.rounds.length - 1].guesses.length == 0) {
                    setGameState('results');
                } else {
					waiting_for = players.filter(
						(player) => !data.rounds[data.rounds.length - 1].guesses.some((guess) => guess.player === player)
					);
				}
				// players.forEach((player: string) => {
				// 	if (player != name) {
				// 		other_players.push(player);
				// 	}
				// });
			});
	}

	let get_game_interval_ms: number = 1000;
	async function getGameLoop() {
		if (localStorage.getItem('game_state') == 'guess_wait') {
			readGame();
			await sleep(get_game_interval_ms);
			getGameLoop();
		}
	}

	onMount(() => {
		getGameLoop();
	});
</script>

<main>
	<div class="topleft">
		{game_name}
	</div>
	<div class="topright">
		Round #{round_count}
	</div>
	<PlayerList players={players} waiting_for={waiting_for}/>
	<div style="padding-bottom: 100px;">
		{question}
	</div>
	<div class="loader"></div>
</main>

<style>
	@import '../../app.css';
</style>
