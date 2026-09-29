<script>
	import FoodStatus from './lib/FoodStatus.svelte';

	// 4 scenarios. days = estimated storage time from the photo;
	// bacteria = sensor reading at detection (% of the unsafe limit); growth = % added per day
	const SAMPLES = {
		'white-rice': {
			name: 'White Rice', label: 'White Rice', icon: '🍚',
			days: 3, bacteria: 10, growth: 30,
			note: 'Ages as expected'
		},
		'fried-rice': {
			name: 'Fried Rice', label: 'Leftover Fried Rice', icon: '🥡',
			days: 1, bacteria: 40, growth: 60,
			note: 'Already a few days old'
		},
		'chicken-soup': {
			name: 'Chicken Soup', label: 'Chicken Soup', icon: '🍲',
			days: 4, bacteria: 15, growth: 45,
			note: 'Fridge too warm, spoils early'
		},
		salad: {
			name: 'Salad', label: 'Salad', icon: '🥗',
			days: 2, bacteria: 5, growth: 15,
			note: 'Low bacteria, but wilts'
		}
	};

	let selectedFood = $state('white-rice');
	let detectedFood = $state(null); // key of the food currently in the container
	let daysLeft = $state(0);
	let isAnalyzing = $state(false);
	let showInfo = $state(false);
	let bacteriaHistory = $state([]); // one sensor reading (%) per day, last = current
	let simDay = $state(0); // days passed since the food was detected
	let isPlaying = $state(false);
	let timer = null;

	let foodName = $derived(detectedFood ? SAMPLES[detectedFood].name : null);

	let bacteria = $derived(bacteriaHistory.at(-1) ?? 0);
	let bacteriaLevel = $derived(bacteria >= 100 ? 'high' : bacteria >= 50 ? 'rising' : 'normal');

	// Days are only an estimate; the bacteria sensor can only make things worse
	let shownDays = $derived(
		bacteriaLevel === 'high' ? 0 : bacteriaLevel === 'rising' ? Math.min(daysLeft, 1) : daysLeft
	);

	let status = $derived(
		!detectedFood
			? ''
			: bacteriaLevel === 'high'
				? 'SPOILED'
				: shownDays <= 0
					? 'EXPIRED'
					: bacteriaLevel === 'rising'
						? 'EAT TODAY'
						: shownDays >= 3
							? 'SAFE'
							: 'EAT SOON'
	);

	// 3+ days green, 2 days orange, 1 or 0 days red
	let ledColor = $derived(
		!detectedFood ? '#bbb' : shownDays >= 3 ? '#2ecc71' : shownDays === 2 ? '#f39c12' : '#e74c3c'
	);

	function takePhoto() {
		stopPlay();
		isAnalyzing = true;
		setTimeout(() => {
			detectedFood = selectedFood;
			daysLeft = SAMPLES[selectedFood].days;
			bacteriaHistory = [SAMPLES[selectedFood].bacteria];
			simDay = 0;
			isAnalyzing = false;
		}, 1200);
	}

	function advanceDay() {
		if (shownDays > 0) {
			daysLeft = Math.max(daysLeft - 1, 0);
			simDay += 1;
			// As food ages, the sensor picks up more bacteria
			bacteriaHistory.push(Math.min(bacteria + SAMPLES[detectedFood].growth, 100));
		}
		if (shownDays <= 0) stopPlay();
	}

	// Testing only: override today's sensor reading
	function setBacteria(value) {
		bacteriaHistory[bacteriaHistory.length - 1] = value;
	}

	// Auto time simulation: 1 second = 1 day, stops when the food expires
	function togglePlay() {
		if (isPlaying) {
			stopPlay();
		} else {
			isPlaying = true;
			timer = setInterval(advanceDay, 1000);
		}
	}

	function stopPlay() {
		clearInterval(timer);
		isPlaying = false;
	}

	// Physical buttons on the lid
	function adjustDays(delta) {
		daysLeft = Math.min(Math.max(daysLeft + delta, 0), 14);
	}

	function clearFood() {
		stopPlay();
		detectedFood = null;
		daysLeft = 0;
		bacteriaHistory = [];
		simDay = 0;
	}
</script>

<main>
	<section class="device-ui">
		<FoodStatus
			{foodName}
			daysLeft={shownDays}
			{status}
			{ledColor}
			foodType={detectedFood}
			{bacteria}
			{bacteriaLevel}
			{bacteriaHistory}
			onAdjust={adjustDays}
			onClear={clearFood}
		/>
	</section>

	<aside class="testing-ui">
		<h1 class="title">Smart Food Container</h1>
		<div class="panel-header">
			<h2>Simulation</h2>
			<button class="info-btn" onclick={() => (showInfo = true)} aria-label="Info">i</button>
		</div>

		<p class="hint">Select a food scenario:</p>
		<div class="food-options">
			{#each Object.entries(SAMPLES) as [key, food]}
				<button
					class="food-option"
					class:selected={selectedFood === key}
					onclick={() => (selectedFood = key)}
					disabled={isAnalyzing}
				>
					<span class="icon">{food.icon}</span>
					<span>
						{food.label}
						<small>{food.note}</small>
					</span>
				</button>
			{/each}
		</div>

		<button onclick={takePhoto} disabled={isAnalyzing}>Take Photo</button>
		{#if isAnalyzing}
			<p class="hint">Analyzing food...</p>
		{/if}

		<hr />

		<p class="hint">Time simulation — day {simDay}</p>
		<button onclick={advanceDay} disabled={!detectedFood || shownDays <= 0 || isAnalyzing}>
			Advance 1 Day
		</button>
		<button onclick={togglePlay} disabled={!detectedFood || shownDays <= 0 || isAnalyzing}>
			{isPlaying ? '❚❚ Pause' : '▶ Play (1s = 1 day)'}
		</button>

		<hr />

		<p class="hint">Bacteria sensor — now {bacteria}%</p>
		<div class="level-buttons">
			{#each [['normal', 20], ['rising', 60], ['high', 100]] as [level, value]}
				<button
					class:selected={bacteriaLevel === level}
					onclick={() => setBacteria(value)}
					disabled={!detectedFood || isAnalyzing}
				>
					{level[0].toUpperCase() + level.slice(1)}
				</button>
			{/each}
		</div>

		<hr />

		<p class="credit">Created by: Howard Li</p>
		<a href="https://app.notion.com/p/Smart-Food-Container-3d56919b723280f5a7a8ffe68621d2d3?source=copy_link">
			Project write-up
		</a>
	</aside>
</main>

{#if showInfo}
	<div class="backdrop" onclick={(e) => e.target === e.currentTarget && (showInfo = false)} role="presentation">
		<div class="modal" role="dialog">
			<h2>How to use this demo</h2>
			<ul>
				<li>Select one of the 4 food scenarios. Each one has a different estimated time and a different bacteria growth rate.</li>
				<li>Take a photo to simulate food recognition.</li>
				<li>The smart container displays the detected food and remaining storage time.</li>
				<li>Advance time to simulate food aging, or press Play to let days pass automatically.</li>
				<li>The LED ring changes color based on food freshness: green (3+ days), orange (2 days), red (1 day or less).</li>
				<li>A bacteria sensor inside the lid measures the bacteria amount (% of the unsafe limit) every day:
					<ul>
						<li><strong>Normal</strong> (under 50%) — some bacteria is normal; the estimated days are used.</li>
						<li><strong>Rising</strong> (50–99%) — still edible but getting close: EAT TODAY (at most 1 day left).</li>
						<li><strong>High</strong> (100%) — unsafe: SPOILED (0 days), no matter how many days were estimated.</li>
					</ul>
					The reading grows as time passes. The Normal / Rising / High buttons override today's reading for testing.
				</li>
			</ul>
			<p><strong>Buttons on the lid</strong> (the real device controls):</p>
			<ul>
				<li><strong>− / +</strong> — correct the days left if the detection was wrong.</li>
				<li><strong>History</strong> — switch the screen to a chart of the daily bacteria readings.</li>
				<li><strong>Clear</strong> — the food was eaten or thrown away; resets the container and the sensor.</li>
			</ul>
			<button onclick={() => (showInfo = false)}>Close</button>
		</div>
	</div>
{/if}

<style>
	main {
		display: flex;
		min-height: 100vh;
	}

	.device-ui {
		flex: 0 0 78%;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 20px;
		box-sizing: border-box;
	}

	.testing-ui {
		flex: 1;
		border-left: 1px solid var(--border);
		padding: 16px;
		display: flex;
		flex-direction: column;
		gap: 10px;
		text-align: left;
		font-size: 14px;
	}

	.title {
		font-size: 22px;
		letter-spacing: normal;
		margin: 0;
	}

	.panel-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.panel-header h2 {
		margin: 0;
	}

	.info-btn {
		width: 26px;
		height: 26px;
		border-radius: 50%;
		padding: 0;
		font-weight: bold;
	}

	.food-options {
		display: flex;
		flex-direction: column;
		gap: 6px;
	}

	.food-option {
		display: flex;
		align-items: center;
		gap: 8px;
		text-align: left;
	}

	.food-option small {
		display: block;
		font-size: 11px;
		opacity: 0.7;
	}

	.level-buttons {
		display: flex;
		gap: 6px;
	}

	.level-buttons button {
		flex: 1;
	}

	.level-buttons .selected,
	.food-option.selected {
		outline: 2px solid var(--accent);
	}

	.icon {
		font-size: 22px;
	}

	.hint {
		font-size: 13px;
	}

	.credit {
		font-size: 12px;
	}

	hr {
		width: 100%;
		border: none;
		border-top: 1px solid var(--border);
	}

	.backdrop {
		position: fixed;
		inset: 0;
		background: rgba(0, 0, 0, 0.4);
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.modal {
		background: var(--bg);
		padding: 20px;
		border-radius: 8px;
		max-width: 400px;
		text-align: left;
	}
</style>
