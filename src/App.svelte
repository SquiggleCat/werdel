<script lang="ts">
	import {
		modeData,
		seededRandomInt,
		createDefaultStats,
		createNewGame,
		createDefaultSettings,
		createLetterStates,
		ROWS,
		getWordNumber,
		words,
	} from "./utils";
	import {
		getCubes,
	} from "./ipautils";
	import Game from "./components/Game.svelte";
	import { letterStates, settings, mode } from "./stores";
	import { GameMode } from "./enums";
	import { Toaster } from "./components/widgets";

	let stats: Stats;
	let word: string;
	let state: GameState;

	settings.set(
		(JSON.parse(localStorage.getItem("settings")) as Settings) || createDefaultSettings()
	);
	settings.subscribe((s) => localStorage.setItem("settings", JSON.stringify(s)));

	const hash = window.location.hash.slice(1).split("/");
	const modeVal: GameMode = !isNaN(GameMode[hash[0]])
		? GameMode[hash[0]]
		: parseInt(localStorage.getItem("mode")) || modeData.default;
	mode.set(modeVal);
	// If this is a link to a specific word make sure that that is the word
	modeData.modes[modeVal].historical = true;
	if (!isNaN(parseInt(hash[1])) && parseInt(hash[1]) < getWordNumber(modeVal)) {
		modeData.modes[modeVal].seed =
			(parseInt(hash[1]) - 1) * modeData.modes[modeVal].unit + modeData.modes[modeVal].start;
	}
	// also allow putting the word itself after the slash (for debugging purposes)
	else if (words.contains(hash[1])) {
		modeData.modes[modeVal].wordOverride = hash[1];
	}
	// if this branch runs, it means no word has been manually set, so turn off the historical flag
	else modeData.modes[modeVal].historical = false;
	mode.subscribe((m) => {
		localStorage.setItem("mode", `${m}`);
		window.location.hash = GameMode[m];
		stats = (JSON.parse(localStorage.getItem(`stats-${m}`)) as Stats) || createDefaultStats(m);
		// if a word has been entered by name (rather than number), choose it
		const override = modeData.modes[modeVal].wordOverride;
		if (modeData.modes[modeVal].historical && override != '') {
			word = override;
			modeData.modes[modeVal].wordOverride = '';
		}
		else word = words.words[seededRandomInt(0, words.words.length, modeData.modes[m].seed)];
		let temp: GameState;
		if (modeData.modes[m].historical === true) {
			temp = JSON.parse(localStorage.getItem(`state-${m}-h`));
			if (!temp || temp.wordNumber !== getWordNumber(m)) {
				state = createNewGame(m);
			} else {
				state = temp;
			}
		} else {
			temp = JSON.parse(localStorage.getItem(`state-${m}`));
			if (!temp || modeData.modes[m].seed - temp.time >= modeData.modes[m].unit) {
				state = createNewGame(m);
			} else {
				// This is for backwards compatibility, can be removed in a day
				if (!temp.wordNumber) {
					temp.wordNumber = getWordNumber(m);
				}
				state = temp;
			}
		}
		// Set the letter states when data for a new game mode is loaded so the keyboard is correct
		const letters = createLetterStates();
		for (let row = 0; row < ROWS; ++row) {
			const wordArr = getCubes(state.board.words[row], false);
			for (let col = 0; col < wordArr.length; ++col) {
				letters[wordArr[col]] = state.board.state[row][col];
			}
		}
		letterStates.set(letters);
	});

	$: saveState(state);
	function saveState(state: GameState) {
		if (modeData.modes[$mode].historical) {
			localStorage.setItem(`state-${$mode}-h`, JSON.stringify(state));
		} else {
			localStorage.setItem(`state-${$mode}`, JSON.stringify(state));
		}
	}
	let toaster: Toaster;
</script>

<Toaster bind:this={toaster} />
{#if toaster}
	<Game {stats} {word} {toaster} bind:game={state} />
{/if}
