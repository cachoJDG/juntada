<script lang="ts">
	import { supabase } from '$lib/supabaseClient';
	
	let email = '';
	let loading = false;
	let events: any[] = [];
	let searched = false;
	
	async function searchEvents() {
		if (!email.trim()) {
			alert('Por favor ingresa un email');
			return;
		}
		
		loading = true;
		searched = false;
		try {
			const { data, error } = await supabase
				.from('events')
				.select('*')
				.eq('creator_email', email.trim())
				.order('created_at', { ascending: false });
			
			if (error) throw error;
			
			events = data || [];
			searched = true;
		} catch (error) {
			console.error('Error searching events:', error);
			alert('Hubo un error al buscar las juntadas. Por favor intenta de nuevo.');
		} finally {
			loading = false;
		}
	}
	
	function formatDate(dateString: string) {
		const date = new Date(dateString);
		return date.toLocaleDateString('es-AR', {
			year: 'numeric',
			month: 'long',
			day: 'numeric',
			hour: '2-digit',
			minute: '2-digit'
		});
	}
	
	function copyEventId(eventId: string) {
		navigator.clipboard.writeText(eventId);
		alert('Código copiado al portapapeles!');
	}
</script>

<div class="min-h-screen bg-slate-50 py-12 px-4 sm:px-6 lg:px-8">
	<div class="max-w-3xl mx-auto">
		<div class="text-center mb-8">
			<h1 class="text-4xl font-extrabold text-slate-900 mb-2">
				Mis Juntadas
			</h1>
			<p class="text-slate-600">
				Busca las juntadas que creaste usando tu email
			</p>
		</div>
		
		<div class="bg-white rounded-xl shadow-lg border border-slate-100 p-6 mb-8">
			<form onsubmit={searchEvents} class="flex gap-3">
				<input
					type="email"
					bind:value={email}
					placeholder="tu@email.com"
					required
					class="flex-1 px-4 py-3 border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 outline-none"
				/>
				<button
					type="submit"
					disabled={loading}
					class="px-6 py-3 bg-indigo-600 text-white font-medium rounded-lg hover:bg-indigo-700 disabled:opacity-50 disabled:cursor-not-allowed transition-colors shadow-sm"
				>
					{loading ? 'Buscando...' : 'Buscar'}
				</button>
			</form>
		</div>
		
		{#if searched}
			{#if events.length > 0}
				<div class="space-y-4">
					<h2 class="text-xl font-semibold text-slate-800 mb-4">
						Encontramos {events.length} {events.length === 1 ? 'juntada' : 'juntadas'}
					</h2>
					{#each events as event}
						<div class="bg-white rounded-lg shadow border border-slate-100 p-6 hover:shadow-md transition-shadow">
							<div class="flex justify-between items-start mb-3">
								<div class="flex-1">
									<h3 class="text-lg font-semibold text-slate-900 mb-1">
										{event.title}
									</h3>
									<p class="text-sm text-slate-600">
										📅 {formatDate(event.event_datetime)}
									</p>
									{#if event.location}
										<p class="text-sm text-slate-600 mt-1">
											📍 {event.location.includes('http') ? 'Ver en mapa' : event.location}
										</p>
									{/if}
								</div>
							</div>
							
							<div class="flex gap-3 mt-4 pt-4 border-t border-slate-100">
								<div class="flex-1 bg-slate-50 rounded-lg p-3">
									<p class="text-xs text-slate-500 font-semibold mb-1">CÓDIGO DEL EVENTO</p>
									<p class="font-mono text-sm text-slate-900 break-all">{event.id}</p>
								</div>
								<button
									onclick={() => copyEventId(event.id)}
									class="px-4 py-2 bg-slate-100 hover:bg-slate-200 text-slate-700 rounded-lg transition-colors font-medium text-sm"
									title="Copiar código"
								>
									📋 Copiar
								</button>
								<a
									href="/event/{event.id}"
									class="px-4 py-2 bg-indigo-600 hover:bg-indigo-700 text-white rounded-lg transition-colors font-medium text-sm"
								>
									Ver Detalles
								</a>
							</div>
						</div>
					{/each}
				</div>
			{:else}
				<div class="bg-white rounded-lg shadow border border-slate-100 p-12 text-center">
					<div class="text-6xl mb-4">🔍</div>
					<h3 class="text-xl font-semibold text-slate-900 mb-2">
						No se encontraron juntadas
					</h3>
					<p class="text-slate-600">
						No hay juntadas creadas con el email <strong>{email}</strong>
					</p>
				</div>
			{/if}
		{/if}
		
		<div class="text-center mt-8">
			<a href="/" class="font-medium text-indigo-600 hover:text-indigo-500">
				← Volver al inicio
			</a>
		</div>
	</div>
</div>
