<script lang="ts">
	import { page } from '$app/state';
	import '../app.css';
	let { children } = $props();
</script>

{#snippet link(href: string, name: string)}
	{@const active = page.url.pathname === href}
	<a
		{href}
		class="font-medium underline decoration-2 underline-offset-4 transition {active
			? 'cursor-default text-black decoration-current'
			: 'decoration-transparent hover:text-black hover:decoration-current'}"
	>
		{name}
	</a>
{/snippet}

<div class="mb-20 mt-8 px-8">
	<nav class="mb-8 flex flex-wrap gap-x-8 gap-y-2 text-gray-500">
		{@render link('/', 'parser')}
		{@render link('/analyze', 'counter')}
		{@render link('/charts', 'charts')}
		{@render link('/embeddings', 'embeddings')}
		{@render link('/papers', 'papers')}

		<a
			href="https://github.com/cubedhuang/ilo-nasin-sin"
			class="ml-auto font-medium underline decoration-transparent decoration-2 underline-offset-4 transition hover:text-black hover:decoration-current"
			>GitHub</a
		>
	</nav>

	{@render children()}
</div>
