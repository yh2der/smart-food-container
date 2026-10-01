<script>
	let {
		foodName,
		daysLeft,
		status,
		ledColor,
		foodType,
		bacteria,
		bacteriaLevel,
		bacteriaHistory,
		onAdjust,
		onClear
	} = $props();

	let dayLabel = $derived(daysLeft === 1 ? 'DAY LEFT' : 'DAYS LEFT');

	// Lid screen shows either the status or the bacteria history chart
	let showHistory = $state(false);

	const LEVEL_COLORS = { normal: '#2ecc71', rising: '#f39c12', high: '#e74c3c' };
	const levelOf = (v) => (v >= 100 ? 'high' : v >= 50 ? 'rising' : 'normal');

	// Chart: 100px tall = the unsafe limit; bars shrink to fit more days
	let barWidth = $derived(Math.min(30, 180 / Math.max(bacteriaHistory.length, 1)));
</script>

<div class="device-area">
	<!-- Physical object: shows WHERE the UI lives on the container -->
	<div class="object" aria-hidden="true" title="Diagram only — not clickable">
		<p class="caption">Smart Container <span class="diagram-note">(diagram)</span></p>

		<div class="lid">
			<div class="lid-screen"></div>
			<div class="lid-buttons">
				<span></span><span></span><span></span><span></span>
			</div>
		</div>
		<p class="tag">↑ Lid screen + buttons</p>

		<!-- Container body; the glowing band is the 360° LED ring -->
		<div class="box" class:blink={bacteriaLevel === 'high'} style="--led: {ledColor}">
			<div class="ring"></div>
			<div class="sensor"></div>
			<p class="sensor-tag">Bacteria sensor (under lid)</p>
			<div class="food">
				{#if foodType === 'white-rice'}
					<svg viewBox="0 0 200 60" width="70%">
						<ellipse cx="100" cy="40" rx="90" ry="22" fill="#f4f1e8" stroke="#ddd6c4" />
					</svg>
				{:else if foodType === 'chicken-soup'}
					<svg viewBox="0 0 200 60" width="70%">
						<ellipse cx="100" cy="40" rx="90" ry="22" fill="#d9a441" stroke="#b5832a" />
						<rect x="60" y="32" width="14" height="10" fill="#f3e2c0" />
						<rect x="115" y="38" width="14" height="10" fill="#f3e2c0" />
						<circle cx="95" cy="36" r="4" fill="#e69138" />
					</svg>
				{:else if foodType === 'salad'}
					<svg viewBox="0 0 200 60" width="70%">
						<ellipse cx="100" cy="40" rx="90" ry="22" fill="#93c47d" stroke="#6aa84f" />
						<circle cx="75" cy="34" r="7" fill="#b6d7a8" />
						<circle cx="125" cy="40" r="6" fill="#e06666" />
						<circle cx="100" cy="30" r="6" fill="#6aa84f" />
					</svg>
				{:else if foodType === 'fried-rice'}
					<svg viewBox="0 0 200 60" width="70%">
						<ellipse cx="100" cy="40" rx="90" ry="22" fill="#e0b25c" stroke="#c4933c" />
						<circle cx="70" cy="35" r="5" fill="#6aa84f" />
						<circle cx="120" cy="42" r="5" fill="#e06666" />
						<circle cx="95" cy="30" r="4" fill="#f6d55c" />
						<circle cx="145" cy="36" r="4" fill="#6aa84f" />
					</svg>
				{/if}
			</div>
		</div>
		<p class="tag">↑ 360° LED ring (around the sides)</p>
	</div>

	<!-- Enlarged view of the lid screen -->
	<div class="screen-view">
		<p class="caption">Lid Screen (enlarged)</p>
		<div class="screen" title="Display only — use the lid buttons below">
			{#if foodName && showHistory}
				<p class="food-name">BACTERIA HISTORY</p>
				<svg class="chart" viewBox="0 0 200 120">
					<!-- threshold lines -->
					<line x1="0" x2="200" y1="10" y2="10" stroke="#e74c3c" stroke-dasharray="4" />
					<text x="198" y="8" text-anchor="end" fill="#e74c3c">100% unsafe</text>
					<line x1="0" x2="200" y1="60" y2="60" stroke="#f39c12" stroke-dasharray="4" />
					<text x="198" y="58" text-anchor="end" fill="#f39c12">50%</text>
					{#each bacteriaHistory as value, day}
						<rect
							x={10 + day * barWidth}
							y={110 - value}
							width={barWidth - 4}
							height={value}
							fill={LEVEL_COLORS[levelOf(value)]}
						/>
						<text x={10 + day * barWidth + (barWidth - 4) / 2} y="119" text-anchor="middle" fill="#aaa">
							D{day}
						</text>
					{/each}
				</svg>
				<p class="sensor-line">
					Stored {bacteriaHistory.length - 1} day{bacteriaHistory.length === 2 ? '' : 's'} · now {bacteria}%
				</p>
			{:else if foodName}
				<p class="food-name">{foodName.toUpperCase()}</p>
				<p class="days">{Math.max(daysLeft, 0)}</p>
				<p class="days-label">{dayLabel}</p>
				<p class="status" style="color: {ledColor}">{status}</p>
				<p class="sensor-line" class:alert={bacteriaLevel !== 'normal'}>
					BACTERIA: {bacteria}% {bacteriaLevel.toUpperCase()}
				</p>
			{:else}
				<p class="empty">
					NO FOOD DETECTED
					<span>Choose a food and press Take Photo →</span>
				</p>
			{/if}
		</div>

		<p class="caption buttons-caption">Lid Buttons</p>
		<div class="buttons">
			<button onclick={() => onAdjust(-1)} disabled={!foodName || showHistory}>−</button>
			<button onclick={() => onAdjust(1)} disabled={!foodName || showHistory}>+</button>
			<button onclick={() => (showHistory = !showHistory)} disabled={!foodName}>
				{showHistory ? 'Status' : 'History'}
			</button>
			<button onclick={onClear} disabled={!foodName}>Clear</button>
		</div>
	</div>
</div>

<style>
	.device-area {
		display: flex;
		align-items: center;
		gap: 60px;
	}

	.object,
	.screen-view {
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	/* Not interactive: hovering shows a "not allowed" cursor */
	.object,
	.screen {
		cursor: not-allowed;
		user-select: none;
	}

	.object {
		width: 360px;
	}

	.diagram-note {
		font-weight: normal;
		font-size: 13px;
		opacity: 0.6;
	}

	.caption {
		font-weight: bold;
		color: var(--text-h);
		margin-bottom: 12px;
	}

	.tag {
		font-size: 13px;
		margin: 6px 0;
	}

	/* Lid seen slightly from above, with the screen's position marked */
	.lid {
		width: 100%;
		height: 60px;
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 14px;
		background: #d9dde2;
		border: 2px solid #b6bcc4;
		border-radius: 16px;
		box-sizing: border-box;
		transform: perspective(600px) rotateX(35deg);
	}

	.lid-screen {
		width: 80px;
		height: 36px;
		background: #111;
		border-radius: 4px;
	}

	.lid-buttons {
		display: flex;
		flex-direction: column;
		gap: 3px;
	}

	.lid-buttons span {
		width: 16px;
		height: 8px;
		background: #555;
		border-radius: 3px;
	}

	.box {
		position: relative;
		width: 94%;
		height: 180px;
		background: rgba(200, 220, 235, 0.35);
		border: 2px solid rgba(150, 170, 190, 0.6);
		border-radius: 0 0 22px 22px;
		display: flex;
		align-items: flex-end;
		justify-content: center;
		padding-bottom: 20px;
		box-sizing: border-box;
	}

	/* LED ring wrapping the top of the container */
	.ring {
		position: absolute;
		top: 8px;
		left: -2px;
		right: -2px;
		height: 10px;
		background: var(--led);
		box-shadow: 0 0 16px 4px var(--led);
		transition:
			background 0.4s,
			box-shadow 0.4s;
	}

	/* High bacteria: the ring blinks to show it's more urgent than just "old" */
	.blink .ring {
		animation: blink 0.8s infinite;
	}

	@keyframes blink {
		50% {
			opacity: 0.2;
		}
	}

	/* Bacteria sensor mounted on the underside of the lid */
	.sensor {
		position: absolute;
		top: 24px;
		left: 50%;
		transform: translateX(-50%);
		width: 40px;
		height: 10px;
		background: #555;
		border-radius: 0 0 6px 6px;
	}

	.sensor-tag {
		position: absolute;
		top: 38px;
		left: 0;
		right: 0;
		text-align: center;
		font-size: 12px;
	}

	.food {
		width: 100%;
		display: flex;
		justify-content: center;
	}

	.screen {
		width: 240px;
		padding: 20px;
		background: #111;
		color: #fff;
		border-radius: 12px;
		font-family: var(--mono);
		text-align: center;
		box-shadow: inset 0 0 0 4px #333;
	}

	.food-name {
		font-size: 16px;
		letter-spacing: 1px;
	}

	.days {
		font-size: 72px;
		line-height: 1;
		font-weight: bold;
		margin: 10px 0 2px;
	}

	.days-label {
		font-size: 13px;
		opacity: 0.8;
	}

	.status {
		margin-top: 10px;
		font-weight: bold;
	}

	.sensor-line {
		margin-top: 10px;
		font-size: 12px;
		opacity: 0.7;
	}

	.sensor-line.alert {
		color: #e74c3c;
		opacity: 1;
		font-weight: bold;
	}

	.chart {
		width: 100%;
		margin: 8px 0 4px;
		font-size: 9px;
	}

	.buttons-caption {
		margin: 16px 0 8px;
	}

	.buttons {
		display: flex;
		gap: 8px;
	}

	.buttons button {
		min-width: 52px;
		padding: 10px 14px;
		font: 600 16px var(--sans);
		color: #222;
		background: linear-gradient(#f4f5f7, #d9dde2);
		border: 1px solid #a9b0b8;
		border-bottom-width: 3px;
		border-radius: 10px;
		cursor: pointer;
	}

	.buttons button:hover:not(:disabled) {
		background: linear-gradient(#ffffff, #e3e7eb);
	}

	.buttons button:active:not(:disabled) {
		transform: translateY(2px);
		border-bottom-width: 1px;
	}

	.buttons button:disabled {
		opacity: 0.4;
		cursor: not-allowed;
	}

	.empty {
		padding: 50px 0;
		font-size: 14px;
	}

	.empty span {
		display: block;
		margin-top: 10px;
		font-size: 12px;
		opacity: 0.7;
	}
</style>
