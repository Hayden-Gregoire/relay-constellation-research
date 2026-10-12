<script lang="ts">
	import { onMount } from 'svelte';

	type NodeType = 'situation' | 'task' | 'person' | 'place' | 'tool';
	type AddableType = Exclude<NodeType, 'situation'>;

	type SupportNode = {
		id: string;
		label: string;
		type: NodeType;
		x: number;
		y: number;
		action: string;
		details: string;
	};

	type NextActionItem = {
		id: string;
		nodeId: string;
		label: string;
		action: string;
	};

	const NODES_KEY = 'relay-constellation-nodes';
	const ACTIONS_KEY = 'relay-constellation-next-actions';
	const CENTER = { x: 400, y: 245 };
	const ORBIT_RADIUS = 185;
	const VIEW_WIDTH = 800;
	const VIEW_HEIGHT = 500;

	const defaultNodes: SupportNode[] = [
		{
			id: 'situation',
			label: 'Start my statistics assignment',
			type: 'situation',
			x: CENTER.x,
			y: CENTER.y,
			action: 'Choose one small first step.',
			details: 'This is the current situation that feels difficult or overwhelming.'
		},
		{
			id: 'canvas',
			label: 'Open Canvas',
			type: 'task',
			x: 180,
			y: 120,
			action: 'Open the course page and find the assignment instructions.',
			details: 'A low-effort first step that makes the assignment more concrete.'
		},
		{
			id: 'grace',
			label: 'Text Grace',
			type: 'person',
			x: 620,
			y: 105,
			action: 'Ask whether Grace can body-double for twenty minutes.',
			details: 'A trusted Circle contact who can provide quiet accountability.'
		},
		{
			id: 'library',
			label: 'Library quiet floor',
			type: 'place',
			x: 185,
			y: 375,
			action: 'Move to a lower-distraction study environment.',
			details: 'A place associated with focus and fewer interruptions.'
		},
		{
			id: 'timer',
			label: 'Five-minute timer',
			type: 'tool',
			x: 625,
			y: 385,
			action: 'Set a five-minute timer and work only until it ends.',
			details: 'A tool that lowers the pressure of getting started.'
		}
	];

	const SEED_IDS = new Set(defaultNodes.map((node) => node.id));

	const typeLabels: Record<NodeType, string> = {
		situation: 'Current situation',
		task: 'Task',
		person: 'Person',
		place: 'Place',
		tool: 'Tool'
	};

	function cloneDefaults(): SupportNode[] {
		return structuredClone(defaultNodes);
	}

	function isCustomNode(node: SupportNode) {
		return !SEED_IDS.has(node.id);
	}

	function loadNodes(): SupportNode[] {
		try {
			const raw = localStorage.getItem(NODES_KEY);
			if (!raw) return cloneDefaults();

			const parsed = JSON.parse(raw) as SupportNode[];
			if (!Array.isArray(parsed) || parsed.length === 0) return cloneDefaults();
			if (!parsed.some((node) => node.type === 'situation')) return cloneDefaults();

			return parsed;
		} catch {
			return cloneDefaults();
		}
	}

	function loadNextActions(): NextActionItem[] {
		try {
			const raw = localStorage.getItem(ACTIONS_KEY);
			if (!raw) return [];

			const parsed = JSON.parse(raw) as NextActionItem[];
			return Array.isArray(parsed) ? parsed : [];
		} catch {
			return [];
		}
	}

	function orbitPosition(supportIndex: number, supportCount: number) {
		const count = Math.max(supportCount, 1);
		const angle = (supportIndex / count) * Math.PI * 2 - Math.PI / 2;

		return {
			x: Math.round(CENTER.x + Math.cos(angle) * ORBIT_RADIUS),
			y: Math.round(CENTER.y + Math.sin(angle) * ORBIT_RADIUS)
		};
	}

	function clamp(value: number, min: number, max: number) {
		return Math.min(max, Math.max(min, value));
	}

	let nodes = $state<SupportNode[]>(cloneDefaults());
	let nextActions = $state<NextActionItem[]>([]);
	let selectedId = $state('situation');
	let ready = $state(false);

	let formLabel = $state('Review formula sheet');
	let formType = $state<AddableType>('task');
	let formDetails = $state('A focused review of the key formulas before starting the problems.');
	let formAction = $state('Skim the formula sheet for five minutes and star the ones you need.');
	let formError = $state('');

	let svgEl = $state<SVGSVGElement | null>(null);
	let dragId = $state<string | null>(null);

	const selectedNode = $derived(nodes.find((node) => node.id === selectedId) ?? nodes[0]);
	const situation = $derived(nodes.find((node) => node.type === 'situation') ?? nodes[0]);
	const supportNodes = $derived(nodes.filter((node) => node.type !== 'situation'));
	const canDeleteSelected = $derived(isCustomNode(selectedNode));
	const actionAlreadySaved = $derived(
		nextActions.some(
			(item) => item.nodeId === selectedNode.id && item.action === selectedNode.action
		)
	);

	onMount(() => {
		nodes = loadNodes();
		nextActions = loadNextActions();
		selectedId = nodes.find((node) => node.type === 'situation')?.id ?? nodes[0].id;
		ready = true;
	});

	$effect(() => {
		const nodeSnapshot = nodes;
		const actionSnapshot = nextActions;
		if (!ready) return;
		localStorage.setItem(NODES_KEY, JSON.stringify(nodeSnapshot));
		localStorage.setItem(ACTIONS_KEY, JSON.stringify(actionSnapshot));
	});

	function selectNode(node: SupportNode) {
		selectedId = node.id;
	}

	function handleNodeKeydown(event: KeyboardEvent, node: SupportNode) {
		if (event.key === 'Enter' || event.key === ' ') {
			event.preventDefault();
			selectNode(node);
		}
	}

	function svgPoint(event: PointerEvent) {
		if (!svgEl) return { x: CENTER.x, y: CENTER.y };

		const point = svgEl.createSVGPoint();
		point.x = event.clientX;
		point.y = event.clientY;

		const matrix = svgEl.getScreenCTM();
		if (!matrix) return { x: CENTER.x, y: CENTER.y };

		const local = point.matrixTransform(matrix.inverse());
		return { x: local.x, y: local.y };
	}

	function startDrag(event: PointerEvent, node: SupportNode) {
		if (event.button !== 0) return;

		event.preventDefault();
		selectNode(node);
		dragId = node.id;
		(event.currentTarget as Element).setPointerCapture?.(event.pointerId);
	}

	function moveDrag(event: PointerEvent) {
		if (!dragId) return;

		const { x, y } = svgPoint(event);
		const node = nodes.find((item) => item.id === dragId);
		const radius = node?.type === 'situation' ? 76 : 58;
		const nextX = clamp(x, radius, VIEW_WIDTH - radius);
		const nextY = clamp(y, radius, VIEW_HEIGHT - radius);

		nodes = nodes.map((item) =>
			item.id === dragId ? { ...item, x: Math.round(nextX), y: Math.round(nextY) } : item
		);
	}

	function endDrag() {
		dragId = null;
	}

	function addSupportBubble(event: Event) {
		event.preventDefault();

		const label = formLabel.trim();
		const details = formDetails.trim();
		const action = formAction.trim();

		if (!label || !details || !action) {
			formError = 'Fill in label, details, and next action.';
			return;
		}

		formError = '';

		const position = orbitPosition(supportNodes.length, supportNodes.length + 1);

		const newNode: SupportNode = {
			id: `support-${crypto.randomUUID()}`,
			label,
			type: formType,
			x: position.x,
			y: position.y,
			action,
			details
		};

		nodes = [...nodes, newNode];
		selectedId = newNode.id;
		formLabel = '';
		formDetails = '';
		formAction = '';
	}

	function deleteSelectedBubble() {
		if (!canDeleteSelected) return;

		const removedId = selectedNode.id;
		nodes = nodes.filter((node) => node.id !== removedId);
		nextActions = nextActions.filter((item) => item.nodeId !== removedId);
		selectedId = 'situation';
	}

	function addToNextActions() {
		if (actionAlreadySaved) return;

		nextActions = [
			...nextActions,
			{
				id: crypto.randomUUID(),
				nodeId: selectedNode.id,
				label: selectedNode.label,
				action: selectedNode.action
			}
		];
	}

	function removeNextAction(id: string) {
		nextActions = nextActions.filter((item) => item.id !== id);
	}

	function resetConstellation() {
		nodes = cloneDefaults();
		nextActions = [];
		selectedId = 'situation';
		formLabel = 'Review formula sheet';
		formType = 'task';
		formDetails = 'A focused review of the key formulas before starting the problems.';
		formAction = 'Skim the formula sheet for five minutes and star the ones you need.';
		formError = '';
	}
</script>

<svelte:head>
	<title>Relay Map</title>
	<meta
		name="description"
		content="An interactive support constellation prototype for Relay."
	/>
</svelte:head>

<svelte:window
	onpointermove={moveDrag}
	onpointerup={endDrag}
	onpointercancel={endDrag}
/>

<main>
	<header>
		<div>
			<p class="eyebrow">RELAY RESEARCH PROTOTYPE</p>
			<h1>Relay Map</h1>
			<p class="subtitle">
				See the people, places, tools, and first steps that can help you move forward.
			</p>
		</div>

		<div class="status">
			<span></span>
			{ready ? 'Saved in this browser' : 'Loading constellation'}
		</div>
	</header>

	<section class="workspace">
		<div class="map-panel">
			<div class="map-heading">
				<div>
					<h2>Your support constellation</h2>
					<p>Select a bubble, drag to rearrange, or add your own supports.</p>
				</div>
				<p class="node-count">{nodes.length} connected supports</p>
			</div>

			<svg
				bind:this={svgEl}
				viewBox="0 0 {VIEW_WIDTH} {VIEW_HEIGHT}"
				aria-label="Interactive support constellation"
				class:dragging={dragId !== null}
			>
				<defs>
					<filter id="shadow" x="-30%" y="-30%" width="160%" height="160%">
						<feDropShadow dx="0" dy="8" stdDeviation="8" flood-opacity="0.15" />
					</filter>
				</defs>

				{#each supportNodes as node (node.id)}
					<line
						x1={situation.x}
						y1={situation.y}
						x2={node.x}
						y2={node.y}
						class="connection"
					/>
				{/each}

				{#each nodes as node (node.id)}
					{@const radius = node.type === 'situation' ? 76 : 58}
					{@const textBox = radius * 1.55}
					<g
						role="button"
						tabindex="0"
						aria-label={`Select ${node.label}`}
						onpointerdown={(event) => startDrag(event, node)}
						onkeydown={(event) => handleNodeKeydown(event, node)}
						class="node-group"
						class:dragging={dragId === node.id}
					>
						<circle
							cx={node.x}
							cy={node.y}
							r={radius}
							class={`node ${node.type} ${selectedNode.id === node.id ? 'active' : ''}`}
							filter="url(#shadow)"
						/>
						<foreignObject
							x={node.x - textBox / 2}
							y={node.y - textBox / 2}
							width={textBox}
							height={textBox}
						>
							<div
								xmlns="http://www.w3.org/1999/xhtml"
								class="bubble-copy"
								class:situation-copy={node.type === 'situation'}
							>
								<span class="bubble-type">{typeLabels[node.type]}</span>
								<span class="bubble-label">{node.label}</span>
							</div>
						</foreignObject>
					</g>
				{/each}
			</svg>

			<section class="next-actions" aria-labelledby="next-actions-heading">
				<div class="next-actions-heading">
					<div>
						<h3 id="next-actions-heading">My next actions</h3>
						<p>Steps you chose from the constellation.</p>
					</div>
					<p class="node-count">{nextActions.length} saved</p>
				</div>

				{#if nextActions.length === 0}
					<p class="empty-actions">
						Select a bubble, then choose “Add to my next actions.”
					</p>
				{:else}
					<ul>
						{#each nextActions as item (item.id)}
							<li>
								<div>
									<p class="action-source">{item.label}</p>
									<p class="action-text">{item.action}</p>
								</div>
								<button
									type="button"
									class="ghost"
									onclick={() => removeNextAction(item.id)}
									aria-label={`Remove action from ${item.label}`}
								>
									Remove
								</button>
							</li>
						{/each}
					</ul>
				{/if}
			</section>
		</div>

		<aside>
			<p class="eyebrow">{typeLabels[selectedNode.type]}</p>
			<h2>{selectedNode.label}</h2>
			<p class="details">{selectedNode.details}</p>

			<div class="action-card">
				<p class="action-label">A manageable next action</p>
				<p>{selectedNode.action}</p>
			</div>

			<div class="aside-actions">
				<button type="button" onclick={addToNextActions} disabled={actionAlreadySaved}>
					{actionAlreadySaved ? 'Already in next actions' : 'Add to my next actions'}
				</button>

				{#if canDeleteSelected}
					<button type="button" class="danger" onclick={deleteSelectedBubble}>
						Delete this bubble
					</button>
				{:else}
					<p class="seed-note">Seeded demo bubble — protected from delete.</p>
				{/if}
			</div>

			<p class="note">
				This prototype keeps the user in control. Relay suggests options but never acts on the
				user’s behalf. Your constellation and next actions are saved locally so they survive a
				refresh.
			</p>
		</aside>
	</section>

	<section class="composer" aria-labelledby="add-support-heading">
		<div class="composer-heading">
			<div>
				<h2 id="add-support-heading">Add a support bubble</h2>
				<p>Connect a task, person, place, or tool to the central situation.</p>
			</div>
			<button type="button" class="secondary" onclick={resetConstellation}>
				Reset constellation
			</button>
		</div>

		<form class="support-form" onsubmit={addSupportBubble}>
			<label>
				<span>Label</span>
				<input
					type="text"
					bind:value={formLabel}
					placeholder="Review formula sheet"
					required
				/>
			</label>

			<label>
				<span>Type</span>
				<select bind:value={formType} required>
					<option value="task">Task</option>
					<option value="person">Person</option>
					<option value="place">Place</option>
					<option value="tool">Tool</option>
				</select>
			</label>

			<label class="wide">
				<span>Details</span>
				<textarea
					bind:value={formDetails}
					placeholder="Why this support helps right now"
					rows="3"
					required
				></textarea>
			</label>

			<label class="wide">
				<span>Next action</span>
				<textarea
					bind:value={formAction}
					placeholder="One small step you can take"
					rows="2"
					required
				></textarea>
			</label>

			{#if formError}
				<p class="form-error" role="alert">{formError}</p>
			{/if}

			<button type="submit" class="submit">Add to constellation</button>
		</form>
	</section>

	<section class="legend">
		<div><span class="dot situation-dot"></span> Current situation</div>
		<div><span class="dot task-dot"></span> Task</div>
		<div><span class="dot person-dot"></span> Person</div>
		<div><span class="dot place-dot"></span> Place</div>
		<div><span class="dot tool-dot"></span> Tool</div>
	</section>
</main>

<style>
	:global(*) {
		box-sizing: border-box;
	}

	:global(body) {
		margin: 0;
		background: #f5f7fb;
		color: #1f2a44;
		font-family:
			Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
	}

	main {
		max-width: 1240px;
		margin: 0 auto;
		padding: 48px 28px;
	}

	header {
		display: flex;
		align-items: flex-start;
		justify-content: space-between;
		gap: 24px;
		margin-bottom: 30px;
	}

	h1,
	h2,
	h3,
	p {
		margin-top: 0;
	}

	h1 {
		margin-bottom: 10px;
		font-size: clamp(2.2rem, 5vw, 3.8rem);
		letter-spacing: -0.06em;
	}

	h2 {
		margin-bottom: 10px;
		font-size: 1.45rem;
		letter-spacing: -0.03em;
	}

	h3 {
		margin-bottom: 6px;
		font-size: 1.05rem;
		letter-spacing: -0.02em;
	}

	.eyebrow {
		margin-bottom: 8px;
		color: #64748b;
		font-size: 0.73rem;
		font-weight: 800;
		letter-spacing: 0.12em;
	}

	.subtitle {
		max-width: 590px;
		margin-bottom: 0;
		color: #516079;
		font-size: 1.05rem;
		line-height: 1.55;
	}

	.status {
		display: flex;
		align-items: center;
		gap: 8px;
		padding: 10px 14px;
		border: 1px solid #d8e1f0;
		border-radius: 999px;
		background: white;
		color: #52617a;
		font-size: 0.9rem;
		white-space: nowrap;
	}

	.status span {
		width: 9px;
		height: 9px;
		border-radius: 50%;
		background: #59b888;
	}

	.workspace {
		display: grid;
		grid-template-columns: minmax(0, 1.75fr) minmax(260px, 0.7fr);
		overflow: hidden;
		border: 1px solid #dfe6f2;
		border-radius: 24px;
		background: white;
		box-shadow: 0 16px 40px rgba(43, 55, 86, 0.08);
	}

	.map-panel {
		min-width: 0;
		padding: 28px;
		border-right: 1px solid #e6ebf4;
	}

	.map-heading,
	.composer-heading,
	.next-actions-heading {
		display: flex;
		align-items: flex-start;
		justify-content: space-between;
		gap: 16px;
	}

	.map-heading p,
	.composer-heading p,
	.next-actions-heading p {
		color: #64748b;
	}

	.node-count {
		padding: 7px 10px;
		border-radius: 8px;
		background: #f3f6fb;
		font-size: 0.82rem;
		white-space: nowrap;
	}

	svg {
		display: block;
		width: 100%;
		min-height: 480px;
		margin-top: 12px;
		border-radius: 18px;
		touch-action: none;
		background:
			radial-gradient(circle at 50% 50%, rgba(224, 232, 250, 0.7), transparent 48%),
			linear-gradient(135deg, #fafcff, #f1f5fc);
	}

	svg.dragging {
		cursor: grabbing;
	}

	.connection {
		stroke: #ccd8ec;
		stroke-width: 3;
		stroke-linecap: round;
	}

	.node-group {
		cursor: grab;
		outline: none;
	}

	.node-group.dragging {
		cursor: grabbing;
	}

	.node {
		stroke: white;
		stroke-width: 4;
		transition: stroke-width 0.2s ease;
		transform-origin: center;
	}

	.node.active {
		stroke: #213357;
		stroke-width: 5;
	}

	.situation {
		fill: #40558b;
	}

	.task {
		fill: #76b69a;
	}

	.person {
		fill: #dc9a7f;
	}

	.place {
		fill: #8aa7d9;
	}

	.tool {
		fill: #d2a5cc;
	}

	.bubble-copy {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		box-sizing: border-box;
		width: 100%;
		height: 100%;
		padding: 18%;
		overflow: hidden;
		color: white;
		text-align: center;
		line-height: 1.2;
		pointer-events: none;
		font-family:
			Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
	}

	.bubble-type {
		display: block;
		margin-bottom: 4px;
		font-size: 10px;
		font-weight: 700;
		letter-spacing: 0.01em;
		opacity: 0.82;
	}

	.bubble-label {
		display: -webkit-box;
		overflow: hidden;
		-webkit-box-orient: vertical;
		-webkit-line-clamp: 3;
		line-clamp: 3;
		font-size: 11px;
		font-weight: 800;
		overflow-wrap: anywhere;
		word-break: break-word;
	}

	.situation-copy .bubble-type {
		font-size: 11px;
	}

	.situation-copy .bubble-label {
		-webkit-line-clamp: 4;
		line-clamp: 4;
		font-size: 12px;
	}

	.next-actions {
		margin-top: 20px;
		padding-top: 18px;
		border-top: 1px solid #e6ebf4;
	}

	.next-actions ul {
		display: grid;
		gap: 10px;
		margin: 14px 0 0;
		padding: 0;
		list-style: none;
	}

	.next-actions li {
		display: flex;
		align-items: flex-start;
		justify-content: space-between;
		gap: 12px;
		padding: 12px 14px;
		border: 1px solid #e2e8f3;
		border-radius: 14px;
		background: #f8fafc;
	}

	.action-source {
		margin-bottom: 4px;
		color: #667fae;
		font-size: 0.74rem;
		font-weight: 800;
		letter-spacing: 0.05em;
		text-transform: uppercase;
	}

	.action-text {
		margin-bottom: 0;
		color: #31466f;
		font-weight: 650;
		line-height: 1.45;
	}

	.empty-actions {
		margin: 14px 0 0;
		color: #758198;
		font-size: 0.92rem;
	}

	aside {
		display: flex;
		flex-direction: column;
		padding: 32px 28px;
		background: #fcfdff;
	}

	.details {
		color: #5b6981;
		line-height: 1.6;
	}

	.action-card {
		margin: 18px 0;
		padding: 16px;
		border-radius: 15px;
		background: #eef3fc;
		color: #31466f;
		line-height: 1.5;
	}

	.action-card p:last-child {
		margin-bottom: 0;
		font-weight: 650;
	}

	.action-label {
		margin-bottom: 7px;
		color: #667fae;
		font-size: 0.74rem;
		font-weight: 800;
		letter-spacing: 0.06em;
		text-transform: uppercase;
	}

	.aside-actions {
		display: flex;
		flex-direction: column;
		gap: 10px;
	}

	.seed-note {
		margin: 0;
		color: #8090a8;
		font-size: 0.84rem;
		line-height: 1.4;
	}

	button {
		padding: 13px 16px;
		border: 0;
		border-radius: 12px;
		background: #40558b;
		color: white;
		font: inherit;
		font-weight: 750;
		cursor: pointer;
	}

	button:hover:not(:disabled) {
		background: #314575;
	}

	button:disabled {
		cursor: default;
		opacity: 0.65;
	}

	button.secondary {
		background: #eef2f8;
		color: #40558b;
	}

	button.secondary:hover:not(:disabled) {
		background: #e2e8f3;
	}

	button.danger {
		background: #f4e4e2;
		color: #9b3b32;
	}

	button.danger:hover:not(:disabled) {
		background: #ebd3cf;
	}

	button.ghost {
		flex-shrink: 0;
		padding: 8px 10px;
		background: transparent;
		color: #667fae;
		font-size: 0.82rem;
	}

	button.ghost:hover:not(:disabled) {
		background: #eef2f8;
		color: #40558b;
	}

	.note {
		margin-top: auto;
		margin-bottom: 0;
		padding-top: 24px;
		color: #758198;
		font-size: 0.86rem;
		line-height: 1.5;
	}

	.composer {
		margin-top: 24px;
		padding: 28px;
		border: 1px solid #dfe6f2;
		border-radius: 24px;
		background: white;
		box-shadow: 0 16px 40px rgba(43, 55, 86, 0.06);
	}

	.support-form {
		display: grid;
		grid-template-columns: repeat(2, minmax(0, 1fr));
		gap: 16px;
		margin-top: 18px;
	}

	.support-form label {
		display: flex;
		flex-direction: column;
		gap: 8px;
		color: #40516f;
		font-size: 0.88rem;
		font-weight: 700;
	}

	.support-form label.wide {
		grid-column: 1 / -1;
	}

	.support-form input,
	.support-form select,
	.support-form textarea {
		width: 100%;
		padding: 12px 14px;
		border: 1px solid #d5deec;
		border-radius: 12px;
		background: #f8fafc;
		color: #1f2a44;
		font: inherit;
		font-weight: 500;
	}

	.support-form textarea {
		resize: vertical;
		min-height: 84px;
	}

	.support-form input:focus,
	.support-form select:focus,
	.support-form textarea:focus {
		outline: 2px solid rgba(64, 85, 139, 0.28);
		outline-offset: 1px;
		border-color: #8aa0c8;
		background: white;
	}

	.form-error {
		grid-column: 1 / -1;
		margin: 0;
		color: #b42318;
		font-size: 0.9rem;
		font-weight: 650;
	}

	.submit {
		grid-column: 1 / -1;
		justify-self: start;
	}

	.legend {
		display: flex;
		flex-wrap: wrap;
		gap: 14px 22px;
		padding: 20px 6px 0;
		color: #617088;
		font-size: 0.85rem;
	}

	.legend div {
		display: flex;
		align-items: center;
		gap: 7px;
	}

	.dot {
		width: 10px;
		height: 10px;
		border-radius: 50%;
	}

	.situation-dot {
		background: #40558b;
	}

	.task-dot {
		background: #76b69a;
	}

	.person-dot {
		background: #dc9a7f;
	}

	.place-dot {
		background: #8aa7d9;
	}

	.tool-dot {
		background: #d2a5cc;
	}

	@media (max-width: 840px) {
		main {
			padding: 30px 16px;
		}

		header,
		.map-heading,
		.composer-heading,
		.next-actions-heading {
			flex-direction: column;
		}

		.workspace {
			grid-template-columns: 1fr;
		}

		.map-panel {
			border-right: 0;
			border-bottom: 1px solid #e6ebf4;
		}

		svg {
			min-height: 360px;
		}

		.support-form {
			grid-template-columns: 1fr;
		}

		.next-actions li {
			flex-direction: column;
		}
	}
</style>
