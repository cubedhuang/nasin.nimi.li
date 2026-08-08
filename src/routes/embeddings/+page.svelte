<script lang="ts">
	import { onMount } from 'svelte';
	import linku from '$lib/linku.json';

	type CorpusType = 'full' | 'pure';
	type RawModel = { dim: number; words: string[]; vectors: number[][] };
	type Model = RawModel & { index: Map<string, number> };
	type Filter = {
		id: string;
		label: string;
		limit?: number;
		set?: Set<string>;
	};

	const MAX_SUGGESTIONS = 50;

	let models = $state<Record<CorpusType, Model> | null>(null);
	let currentModel = $state<CorpusType>('pure');
	let query = $state('');
	let selected = $state('pona');
	let highlighted = $state(0);
	let open = $state(false);
	let topN = $state(200);
	let filterId = $state('core');

	async function load(name: CorpusType): Promise<Model> {
		const m: RawModel = await fetch(`/models/${name}.json`).then((r) =>
			r.json()
		);
		return { ...m, index: new Map(m.words.map((w, i) => [w, i])) };
	}

	onMount(async () => {
		const [full, pure] = await Promise.all([load('full'), load('pure')]);
		models = { full, pure };
	});

	const model = $derived(models ? models[currentModel] : null);

	const filters = $derived<Filter[]>([
		{
			id: 'core',
			label: 'Core + Common',
			set: models ? new Set(models.pure.words) : new Set<string>()
		},
		{ id: 'linku', label: 'All Linku', set: new Set(Object.keys(linku)) },
		{ id: 'top500', label: 'Top 500', limit: 500 },
		{ id: 'top1000', label: 'Top 1000', limit: 1000 },
		{ id: 'top2000', label: 'Top 2000', limit: 2000 },
		{ id: 'top4000', label: 'Top 4000', limit: 4000 },
		{ id: 'all', label: `All ${model ? model.words.length : 'Words'}` }
	]);
	const filter = $derived(
		filters.find((f) => f.id === filterId) ?? filters[4]
	);

	const suggestions = $derived.by(() => {
		if (!model) return [];
		const q = query.trim().toLowerCase();
		if (!q) return [];
		const equals: string[] = [];
		const starts: string[] = [];
		const contains: string[] = [];
		for (const w of model.words) {
			const lw = w.toLowerCase();
			if (lw === q) equals.push(w);
			else if (lw.startsWith(q)) starts.push(w);
			else if (lw.includes(q)) contains.push(w);
			if (starts.length >= MAX_SUGGESTIONS) break;
		}
		return [...equals, ...starts, ...contains].slice(0, MAX_SUGGESTIONS);
	});

	const allowed = $derived.by(() => {
		if (!model) return null;
		const n = model.words.length;
		const mask = new Uint8Array(n);
		for (let i = 0; i < n; i++) {
			if (filter.limit !== undefined) mask[i] = i < filter.limit ? 1 : 0;
			else if (filter.set)
				mask[i] = filter.set.has(model.words[i]) ? 1 : 0;
			else mask[i] = 1;
		}
		return mask;
	});

	const neighbors = $derived.by(() => {
		if (!model) return [];
		const i = model.index.get(selected);
		if (i === undefined) return [];
		const v = model.vectors[i];
		const scored: { word: string; score: number }[] = [];
		for (let j = 0; j < model.vectors.length; j++) {
			if (j === i || (allowed && !allowed[j])) continue;
			const u = model.vectors[j];
			let s = 0;
			for (let d = 0; d < v.length; d++) s += v[d] * u[d];
			scored.push({ word: model.words[j], score: s });
		}
		scored.sort((a, b) => b.score - a.score);
		return scored.slice(0, topN);
	});

	const missing = $derived(
		!!model && model.index.get(selected) === undefined
	);

	function pick(word: string) {
		selected = word;
		query = '';
		open = false;
		highlighted = 0;
	}

	function onKeydown(e: KeyboardEvent) {
		if (!open || suggestions.length === 0) return;
		if (e.key === 'ArrowDown') {
			e.preventDefault();
			highlighted = (highlighted + 1) % suggestions.length;
		} else if (e.key === 'ArrowUp') {
			e.preventDefault();
			highlighted =
				(highlighted - 1 + suggestions.length) % suggestions.length;
		} else if (e.key === 'Enter') {
			e.preventDefault();
			pick(suggestions[highlighted]);
		} else if (e.key === 'Escape') {
			open = false;
		}
	}
</script>

<svelte:head>
	<title>sona kon</title>
</svelte:head>

<h1 class="text-3xl font-bold">sona kon</h1>

{#if !models}
	<p class="mt-4">Loading models...</p>
{:else}
	<p class="mt-4">
		<label>
			<span class="mr-2 font-semibold">Model</span>
			<select
				class="cursor-pointer rounded border-2 border-gray-200 px-1 py-0.5 transition hover:border-gray-400"
				bind:value={currentModel}
			>
				<option value="full">Full Corpus</option>
				<option value="pure">Pure Toki Pona</option>
			</select>
		</label>
		{#if currentModel === 'full'}
			<label class="ml-6">
				<span class="mr-2 font-semibold">Vocabulary</span>
				<select
					class="cursor-pointer rounded border-2 border-gray-200 px-1 py-0.5 transition hover:border-gray-400"
					bind:value={filterId}
				>
					{#each filters as f}
						<option value={f.id}>{f.label}</option>
					{/each}
				</select>
			</label>
		{/if}
	</p>

	<div class="relative mt-4 max-w-sm">
		<input
			class="mt-4 w-full rounded-lg border-2 px-4 py-1 outline-none transition focus:border-gray-400"
			placeholder="o pana e nimi..."
			bind:value={query}
			oninput={() => {
				open = true;
				highlighted = 0;
			}}
			onfocus={() => (open = true)}
			onblur={() => (open = false)}
			onkeydown={onKeydown}
		/>

		{#if open && suggestions.length > 0}
			<ul
				class="absolute z-10 mt-1 max-h-96 w-full overflow-auto rounded-lg border-2 border-gray-400 bg-white shadow"
				role="listbox"
			>
				{#each suggestions as word, i}
					<li>
						<button
							type="button"
							class="block w-full px-4 py-1 text-left {i ===
							highlighted
								? 'bg-gray-100'
								: 'hover:bg-gray-100'}"
							onmousedown={(e) => {
								e.preventDefault();
								pick(word);
							}}
						>
							{word}
						</button>
					</li>
				{/each}
			</ul>
		{/if}
	</div>

	{#if missing}
		<p class="mt-4 text-gray-600">
			<span class="font-semibold">{selected}</span> not found!
		</p>
	{:else}
		<h2 class="mt-6 text-xl font-semibold">
			nimi seme li poka tawa nimi <span class="italic">{selected}</span>?
		</h2>
		<ol class="mt-2 max-w-sm border-t border-gray-200">
			{#each neighbors as n}
				<li class="relative border-b border-gray-200 py-0.5">
					<div
						class="absolute inset-y-0 left-0 bg-blue-50"
						style="width: {Math.max(0, n.score) * 100}%"
					></div>
					<div
						class="relative flex items-baseline justify-between px-1"
					>
						<button
							type="button"
							class="font-medium hover:underline"
							onclick={() => pick(n.word)}
						>
							{n.word}
						</button>
						<span class=" text-sm tabular-nums text-gray-600"
							>{n.score.toFixed(3)}</span
						>
					</div>
				</li>
			{/each}
		</ol>
	{/if}
{/if}
