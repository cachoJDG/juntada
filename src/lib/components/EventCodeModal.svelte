<script lang="ts">
	let { eventId, onClose } = $props();
	
	let copied = $state(false);
	
	async function copyToClipboard() {
		try {
			await navigator.clipboard.writeText(eventId);
			copied = true;
			setTimeout(() => {
				copied = false;
			}, 2000);
		} catch (error) {
			console.error('Error copying to clipboard:', error);
			alert('No se pudo copiar al portapapeles');
		}
	}
</script>

<div class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-sm p-4">
	<div class="bg-white rounded-xl shadow-2xl w-full max-w-md overflow-hidden">
		<div class="p-6 border-b border-slate-100 flex justify-between items-center bg-gradient-to-r from-indigo-50 to-purple-50">
			<h3 class="text-xl font-bold text-slate-800">🎉 ¡Juntada Creada!</h3>
			<button onclick={onClose} class="text-slate-400 hover:text-slate-600 transition-colors">
				✕
			</button>
		</div>
		
		<div class="p-6 space-y-4">
			<p class="text-slate-600 text-center">
				Tu juntada se creó exitosamente. Comparte este código con los participantes:
			</p>
			
			<div class="bg-slate-50 border-2 border-dashed border-indigo-300 rounded-lg p-4">
				<div class="text-center">
					<p class="text-xs text-slate-500 uppercase tracking-wide font-semibold mb-2">Código del Evento</p>
					<p class="text-2xl font-mono font-bold text-indigo-600 break-all select-all">
						{eventId}
					</p>
				</div>
			</div>
			
			<button
				onclick={copyToClipboard}
				class="w-full px-4 py-3 rounded-lg font-medium transition-all shadow-sm {copied 
					? 'bg-green-600 text-white' 
					: 'bg-indigo-600 text-white hover:bg-indigo-700'}"
			>
				{#if copied}
					✓ Copiado!
				{:else}
					📋 Copiar Código
				{/if}
			</button>
			
			<p class="text-xs text-slate-500 text-center">
				Los participantes pueden usar este código para unirse a tu juntada
			</p>
		</div>
		
		<div class="p-4 bg-slate-50 border-t border-slate-100 flex justify-end">
			<button
				onclick={onClose}
				class="px-6 py-2 rounded-lg bg-white text-slate-700 font-medium border border-slate-200 hover:bg-slate-100 transition-colors"
			>
				Cerrar
			</button>
		</div>
	</div>
</div>
