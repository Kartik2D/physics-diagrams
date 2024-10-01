<script lang="ts">
	import { onMount } from 'svelte';
	import { writable, type Writable } from 'svelte/store';

	type Component = new (...args: any[]) => any;

	// Store to hold the current component
	const currentComponent: Writable<Component | null> = writable(null);

	// Store to hold the list of available pages
	const pages: Writable<string[]> = writable([]);

	onMount(async () => {
		// Dynamically import all files from the /pages directory
		const modules = import.meta.glob<{ default: Component }>('/src/pages/*.svelte');
		const pageNames = Object.keys(modules).map(key => key.split('/').pop()?.replace('.svelte', '') || '');
		pages.set(pageNames);

		// Load the first page by default
		if (pageNames.length > 0) {
			await loadComponent(pageNames[0]);
		}
	});

	async function loadComponent(pageName: string): Promise<void> {
		try {
			const module = await import(`/src/pages/${pageName}.svelte`);
			currentComponent.set(module.default);
		} catch (error) {
			console.error(`Failed to load component: ${pageName}`, error);
		}
	}

	function handleSelectChange(event: Event): void {
		const target = event.target as HTMLSelectElement;
		loadComponent(target.value);
	}
</script>

<div class="overlay">
	<div class="page-selector">
		<select id="page-select" on:change={handleSelectChange}>
			{#each $pages as page}
				<option value={page}>{page}</option>
			{/each}
		</select>
	</div>
</div>

<main>
	<div class="page-content">
		{#if $currentComponent}
			<svelte:component this={$currentComponent} />
		{:else}
			<p>Loading...</p>
		{/if}
	</div>
</main>

<style>
	.overlay {
		position: fixed;
		bottom: 20px;
		left: 20px;
		z-index: 1000;
		background-color: rgba(51, 51, 51, 0.8);
		padding: 10px;
		border-radius: 5px;
		box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
	}

	.page-selector {
		display: flex;
		align-items: center;
	}

	select {
		padding: 5px;
		border-radius: 3px;
		border: 1px solid #444;
		background-color: #2a2a2a;
		color: #e0e0e0;
	}
</style>
