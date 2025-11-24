<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';
	import '../app.css';
	import { goto } from '$app/navigation';
	import { showJoinModal } from '$lib/stores/modalStore';

	let { children } = $props();
	let joinId = $state('');

	function handleJoin() {
		if (joinId.trim()) {
			goto(`/event/${joinId.trim()}`);
			$showJoinModal = false;
		}
	}
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
</svelte:head>

{@render children()}

{#if $showJoinModal}
	<div class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-sm p-4">
		<div class="bg-white rounded-xl shadow-2xl w-full max-w-md overflow-hidden p-6 space-y-6">
			<div class="flex justify-between items-center">
				<h3 class="text-xl font-bold text-slate-900">Unirse a una Juntada</h3>
				<button onclick={() => ($showJoinModal = false)} class="text-slate-400 hover:text-slate-600">
					✕
				</button>
			</div>
			
			<div class="space-y-2">
				<label for="joinId" class="block text-sm font-medium text-slate-700">ID de la Juntada</label>
				<input
					id="joinId"
					type="text"
					bind:value={joinId}
					placeholder="Ej: 602eaeeb-..."
					class="w-full px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 outline-none transition-all"
				/>
				<p class="text-xs text-slate-500">Pídele el ID al organizador</p>
			</div>

			<div class="flex justify-end gap-3 pt-2">
				<button
					onclick={() => ($showJoinModal = false)}
					class="px-4 py-2 rounded-lg text-slate-600 hover:bg-slate-100 font-medium transition-colors"
				>
					Cancelar
				</button>
				<button
					onclick={handleJoin}
					disabled={!joinId.trim()}
					class="px-4 py-2 rounded-lg bg-indigo-600 text-white font-medium hover:bg-indigo-700 disabled:opacity-50 disabled:cursor-not-allowed transition-colors shadow-sm cursor-pointer"
				>
					Ir a la Juntada
				</button>
			</div>
		</div>
	</div>
{/if}
