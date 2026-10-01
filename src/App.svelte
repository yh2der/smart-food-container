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

<header class="banner">
	<div>
		<h1>Smart Food Container</h1>
		<p>Interface to a Smart Object · Yun Hao Li</p>
	</div>
	<a href="https://app.notion.com/p/Smart-Food-Container-3d56919b723280f5a7a8ffe68621d2d3?source=copy_link">Project write-up ↗</a>
</header>

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
		<div class="panel-header">
			<h2>Simulation</h2>
			<button class="info-btn" onclick={() => (showInfo = true)} aria-label="Info">i</button>
		</div>

		<p class="step">1. Choose a food</p>
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

		<p class="step">2. Put it in the container</p>
		<button class="primary" onclick={takePhoto} disabled={isAnalyzing}>📷 Take Photo</button>
		{#if isAnalyzing}
			<p class="hint">Analyzing food...</p>
		{/if}

		<p class="step">3. Let time pass <span class="hint">day {simDay}</span></p>
		<div class="button-row">
			<button onclick={advanceDay} disabled={!detectedFood || shownDays <= 0 || isAnalyzing}>
				Advance 1 Day
			</button>
			<button class="primary" title="1 second = 1 day" onclick={togglePlay} disabled={!detectedFood || shownDays <= 0 || isAnalyzing}>
				{isPlaying ? '❚❚ Pause' : '▶ Play'}
			</button>
		</div>

		<p class="step">Testing: bacteria sensor <span class="hint">{detectedFood ? `${bacteria}%` : ''}</span></p>
		<div class="level-buttons">
			{#each [['normal', 20], ['rising', 60], ['high', 100]] as [level, value]}
				<button
					class:selected={detectedFood && bacteriaLevel === level}
					onclick={() => setBacteria(value)}
					disabled={!detectedFood || isAnalyzing}
				>
					{level[0].toUpperCase() + level.slice(1)}
				</button>
			{/each}
		</div>
	</aside>
</main>

{#if showInfo}
	<div class="backdrop" onclick={(e) => e.target === e.currentTarget && (showInfo = false)} role="presentation">
		<div class="modal" role="dialog">
			<h2>How to use</h2>
			<ol class="info-steps">
				<li>Choose a food</li>
				<li>Press <b>Take Photo</b></li>
				<li>Press <b>Advance 1 Day</b> or <b>Play</b></li>
			</ol>

			<h3>LED ring</h3>
			<dl>
				<dt><span class="dot" style="background:#2ecc71"></span>Green</dt><dd>3+ days</dd>
				<dt><span class="dot" style="background:#f39c12"></span>Orange</dt><dd>2 days</dd>
				<dt><span class="dot" style="background:#e74c3c"></span>Red</dt><dd>1 day or less · blinks when spoiled</dd>
			</dl>

			<h3>Bacteria sensor</h3>
			<dl>
				<dt>Normal</dt><dd>under 50%</dd>
				<dt>Rising</dt><dd>50–99% → EAT TODAY</dd>
				<dt>High</dt><dd>100% → SPOILED</dd>
			</dl>
			<p class="info-note">The Normal / Rising / High buttons set today's reading for testing.</p>

			<h3>Lid buttons</h3>
			<dl>
				<dt>− / +</dt><dd>Correct the days</dd>
				<dt>History</dt><dd>Bacteria chart</dd>
				<dt>Clear</dt><dd>Empty the container</dd>
			</dl>

			<button class="close-btn" onclick={() => (showInfo = false)}>Close</button>
		</div>
	</div>
{/if}

<style>
	.banner {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 16px;
		padding: 12px 24px;
		background: #1f2a30;
		color: #fff;
		text-align: left;
	}

	.banner h1 {
		margin: 0;
		font-size: 24px;
		line-height: 1.2;
		letter-spacing: normal;
		color: #fff;
	}

	.banner p {
		font-size: 14px;
		opacity: 0.75;
	}

	.banner a {
		color: #fff;
		font-size: 14px;
		white-space: nowrap;
	}

	main {
		display: flex;
		flex: 1;
	}

	/* Every clickable thing in the panel looks like a button */
	.testing-ui button {
		font: inherit;
		padding: 8px 12px;
		border: 1px solid #b9c0c7;
		border-radius: 8px;
		background: var(--bg);
		color: var(--text-h);
		cursor: pointer;
		box-shadow: 0 1px 2px rgba(0, 0, 0, 0.08);
		transition:
			border-color 0.15s,
			background 0.15s;
	}

	.testing-ui button:hover:not(:disabled) {
		border-color: var(--accent);
		background: var(--accent-bg);
	}

	.testing-ui button:disabled {
		opacity: 0.4;
		cursor: not-allowed;
		box-shadow: none;
	}

	.testing-ui button.primary {
		background: var(--accent);
		border-color: var(--accent);
		color: #fff;
		font-weight: 600;
	}

	.testing-ui button.primary:hover:not(:disabled) {
		background: var(--accent);
		filter: brightness(1.1);
	}

	.step {
		display: flex;
		justify-content: space-between;
		align-items: baseline;
		margin-top: 6px;
		font-size: 14px;
		font-weight: 600;
		color: var(--text-h);
	}

	.step .hint {
		font-weight: normal;
		font-size: 12px;
	}

	.device-ui {
		flex: 0 0 75%;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 20px;
		box-sizing: border-box;
	}

	.testing-ui {
		flex: 1;
		border-left: 1px solid var(--border);
		padding: 14px 16px;
		display: flex;
		flex-direction: column;
		gap: 8px;
		text-align: left;
		font-size: 14px;
	}

	.panel-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.panel-header h2 {
		margin: 0;
	}

	.testing-ui .info-btn {
		width: 28px;
		height: 28px;
		flex: none;
		display: grid;
		place-items: center;
		padding: 0;
		border-radius: 50%;
		font: italic 700 15px/1 Georgia, serif;
	}

	.food-options {
		display: flex;
		flex-direction: column;
		gap: 6px;
	}

	.testing-ui .food-option {
		display: flex;
		align-items: center;
		gap: 8px;
		padding: 4px 10px;
		text-align: left;
		line-height: 1.25;
	}

	.button-row {
		display: flex;
		gap: 6px;
	}

	.button-row button {
		flex: 1;
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

	.testing-ui .level-buttons .selected,
	.testing-ui .food-option.selected {
		border-color: var(--accent);
		background: var(--accent-bg);
		box-shadow: inset 0 0 0 1px var(--accent);
	}

	.icon {
		font-size: 20px;
	}

	.hint {
		font-size: 13px;
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
		padding: 24px;
		border-radius: 12px;
		width: 340px;
		text-align: left;
		font-size: 14px;
		line-height: 1.5;
	}

	.modal h2 {
		margin: 0 0 8px;
	}

	.modal h3 {
		margin: 16px 0 6px;
		font-size: 12px;
		font-weight: 600;
		letter-spacing: 0.08em;
		text-transform: uppercase;
		color: var(--text);
	}

	.info-steps {
		margin: 0;
		padding-left: 20px;
		color: var(--text-h);
	}

	.modal dl {
		display: grid;
		grid-template-columns: 80px 1fr;
		gap: 4px 12px;
		margin: 0;
	}

	.modal dt {
		font-weight: 600;
		color: var(--text-h);
		display: flex;
		align-items: center;
		gap: 6px;
	}

	.modal dd {
		margin: 0;
	}

	.dot {
		width: 10px;
		height: 10px;
		border-radius: 50%;
	}

	.info-note {
		margin-top: 6px;
		font-size: 12px;
		opacity: 0.8;
	}

	.close-btn {
		margin-top: 20px;
		width: 100%;
		padding: 8px;
		font: inherit;
		font-weight: 600;
		border: none;
		border-radius: 8px;
		background: var(--accent);
		color: #fff;
		cursor: pointer;
	}
</style>
