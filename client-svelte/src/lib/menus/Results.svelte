<script lang="ts">
	import Button from '$lib/Button.svelte';
	import type { Answer } from '$lib/datatypes/answer';
	import type { Guess } from '$lib/datatypes/guess';
	import { onMount } from 'svelte';
	import { getGame, getScore } from '$lib/functions/requests';
	import type { Game } from '$lib/datatypes/game';
	import type { Round } from '$lib/datatypes/round';

	export let setGameState: (new_state: string) => void;
	export let name: string | null;
	export let game_name: string | null;

	let question: string;
	let answers: Array<Answer> = [];
	let players: Array<string> = [];
	let correct_answer_map: Map<string, string> = new Map();
	let my_answer: string;
	let my_guess: Array<Answer> = [];
	let score_map: Map<string, number> = new Map();
	let knows_score_map: Map<string, number> = new Map();
	let people_who_guessed_you_correct: Set<string> = new Set([]);
	let round_count: number;
	let game: Game;

	// One row per answer that was not mine: who really gave it, and who I put on it.
	$: my_rows = answers
		.filter((answer) => answer.player != name)
		.slice()
		.sort((a, b) => a.player.localeCompare(b.player))
		.map((answer) => {
			const mine = my_guess.find((guess_answer) => guess_answer.answer == answer.answer);
			return {
				answer: answer.answer,
				actual: answer.player,
				guessed: mine ? mine.player : ''
			};
		});
	$: my_correct = my_rows.filter((row) => row.guessed == row.actual).length;
	$: other_players = players.filter((player) => player != name);
	$: rounds_played = round_count > 1 ? round_count - 1 : 1;

	function onNextRoundClick() {
		setGameState('answer');
	}

	function getScores() {
		getScore(game_name)
			.then((response) => response.json())
			.then((data) => {
				for (var prop in data) {
					score_map.set(prop, data[prop]);
				}
				score_map = new Map(
					[...score_map.entries()].sort((a, b) => b[1] - a[1] || a[0].localeCompare(b[0]))
				);
			});
	}

	async function readGame() {
		getGame(game_name)
			.then((response) => response.json())
			.then((data) => {
				game = data;
				round_count = data.rounds.length;
				players = data.players;
				question = data.rounds[data.rounds.length - 2].question;
				answers = data.rounds[data.rounds.length - 2].answers;

				answers.forEach((answer: Answer) => {
					correct_answer_map.set(answer.player, answer.answer);
				});
				correct_answer_map = correct_answer_map;
				my_answer = correct_answer_map.get(name);

				data.rounds[data.rounds.length - 2].guesses.forEach((guess: Guess) => {
					if (guess.player == name) {
						my_guess = guess.answers;
					} else {
						guess.answers.forEach((answer) => {
							if (answer.player == name && answer.answer == my_answer) {
								people_who_guessed_you_correct.add(guess.player);
							}
						});
					}
				});
				people_who_guessed_you_correct = people_who_guessed_you_correct;

				data.players.forEach((player: string) => {
					if (player != name) {
						getKnowScore(player);
					}
				});
				knows_score_map = new Map(
					[...knows_score_map.entries()].sort((a, b) => b[1] - a[1] || a[0].localeCompare(b[0]))
				);
			});
	}

	onMount(() => {
		readGame();
		getScores();
	});

	function getKnowScore(player: string) {
		let count: number = 0;
		game?.rounds.forEach((round: Round) => {
			let correct_answer = '';
			round.answers.forEach((answer: Answer) => {
				if (answer.player == name) {
					correct_answer = answer.answer;
					return;
				}
			});

			round.guesses.forEach((guess: Guess) => {
				if (guess.player == player) {
					return guess.answers.forEach((answer: Answer) => {
						if (answer.answer == correct_answer && answer.player == name) {
							count += 1;
						}
					});
				}
			});
		});
		knows_score_map.set(player, count);
	}
</script>

<main>
	<div class="topleft">
		{game_name}
	</div>
	<div class="topright">
		Round #{round_count}
	</div>

	<div class="question">
		{question}
	</div>
	<div class="summary">
		you got {my_correct} of {my_rows.length} · {people_who_guessed_you_correct.size} of {other_players.length}
		got you
	</div>

	<h2>How You Did</h2>
	<div class="stack">
		{#each my_rows as row (row.answer)}
			<div class="row {row.guessed == row.actual ? 'correct' : 'incorrect'}">
				<div class="line answer">"{row.answer}"</div>
				<div class="line who">
					<span class="mark">{row.guessed == row.actual ? '✓' : '✗'}</span>
					{row.actual} said it
					{#if row.guessed != row.actual}
						· you said {row.guessed == '' ? 'nobody' : row.guessed}
					{/if}
				</div>
			</div>
		{/each}
	</div>

	<h2>Who Knows You</h2>
	<div class="stack">
		{#if my_answer}
			<div class="row me">
				<div class="line who">you said</div>
				<div class="line answer">"{my_answer}"</div>
			</div>
		{/if}
		{#each knows_score_map as [player, score] (player)}
			<div class="row {people_who_guessed_you_correct.has(player) ? 'correct' : 'incorrect'}">
				<div class="line who">
					<span class="mark">{people_who_guessed_you_correct.has(player) ? '✓' : '✗'}</span>
					{player}
				</div>
				<div class="line note">
					got you {score} of {rounds_played} {rounds_played == 1 ? 'round' : 'rounds'}
				</div>
			</div>
		{/each}
	</div>

	<h2>Leader Board</h2>
	<div class="stack">
		{#each score_map as [player, score], index (player)}
			<div class="row leader {player == name ? 'me' : 'neutral'}">
				<span class="rank">{index + 1}</span>
				<span class="who">{player}</span>
				<span class="points">{score}</span>
			</div>
		{/each}
	</div>

	<div>
		<Button text="Next Round" onClick={onNextRoundClick} />
	</div>
</main>

<style>
	@import '../../app.css';

	.question {
		font-size: 18px;
		padding: 0 10px;
	}
	.summary {
		font-size: 14px;
		opacity: 0.7;
		padding: 0 10px 5px 10px;
	}
	.stack {
		max-width: 460px;
		margin-inline: auto;
		padding: 0;
	}
	.row {
		padding: 10px 14px;
		overflow-wrap: anywhere;
	}
	.row:first-child {
		border-top-right-radius: 50px;
		border-top-left-radius: 50px;
	}
	.row:last-child {
		border-bottom-right-radius: 50px;
		border-bottom-left-radius: 50px;
	}
	.line {
		padding: 0;
	}
	.answer {
		font-size: 18px;
	}
	.who {
		font-size: 16px;
	}
	.note {
		font-size: 14px;
		opacity: 0.8;
	}
	.mark {
		font-size: 18px;
		padding-right: 4px;
	}
	.leader {
		display: flex;
		align-items: center;
		gap: 10px;
	}
	.leader .rank {
		opacity: 0.7;
		min-width: 20px;
		text-align: left;
	}
	.leader .who {
		flex: 1;
		text-align: left;
	}
	.leader .points {
		font-size: 18px;
	}
	.correct {
		background-color: rgba(116, 185, 95, 0.75);
	}
	.incorrect {
		background-color: rgba(202, 96, 96, 0.75);
	}
	.me {
		background-color: rgba(28, 188, 252, 0.75);
	}
	@media (prefers-color-scheme: dark) {
		.neutral {
			background-color: rgba(255, 255, 255, 0.1);
		}
		.row + .row {
			border-top: 1px solid #111;
		}
	}
	@media (prefers-color-scheme: light) {
		.neutral {
			background-color: rgba(0, 0, 0, 0.15);
		}
		.row + .row {
			border-top: 1px solid #fff;
		}
	}
</style>
