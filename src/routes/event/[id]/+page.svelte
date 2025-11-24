<script lang="ts">
	import { onMount } from 'svelte';
	import { supabase } from '$lib/supabaseClient';
	import { page } from '$app/stores';

	let { data } = $props();
	let eventId = data.id;

	let event: any = $state(null);
	let attendees: any[] = $state([]);
	let loading = $state(true);
	let joinLoading = $state(false);

	// Join Form State
	let name = $state('');
	let email = $state('');
	let origin = $state('');
	let arrival_time = $state('');
	let is_driver = $state(false);
	let car_capacity = $state(0);
	let notes = $state('');

	onMount(async () => {
		await fetchEventDetails();
		await fetchAttendees();
		loading = false;
	});

	async function fetchEventDetails() {
		const { data, error } = await supabase
			.from('events')
			.select('*')
			.eq('id', eventId)
			.single();

		if (error) {
			console.error('Error fetching event:', error);
			alert('No se pudo cargar la juntada. Verifica el ID.');
		} else {
			event = data;
		}
	}

	async function fetchAttendees() {
		const { data, error } = await supabase
			.from('event_attendees')
			.select('*')
			.eq('event_id', eventId)
			.order('created_at', { ascending: true });

		if (error) {
			console.error('Error fetching attendees:', error);
		} else {
			attendees = data || [];
		}
	}

	async function handleJoin() {
		joinLoading = true;
		try {
			const { error } = await supabase
				.from('event_attendees')
				.insert([
					{
						event_id: eventId,
						name,
						email: email || null,
						origin,
						arrival_time: arrival_time || null,
						is_driver,
						car_capacity: is_driver ? car_capacity : null,
						notes: notes || null
					}
				]);

			if (error) throw error;

			alert('¡Te has unido a la juntada!');
			// Reset form
			name = '';
			email = '';
			origin = '';
			arrival_time = '';
			is_driver = false;
			car_capacity = 0;
			notes = '';
			
			// Refresh list
			await fetchAttendees();
		} catch (error) {
			console.error('Error joining event:', error);
			alert('Hubo un error al unirse. Intenta de nuevo.');
		} finally {
			joinLoading = false;
		}
	}
</script>

<div class="min-h-screen bg-slate-50 py-12 px-4 sm:px-6 lg:px-8">
	<div class="max-w-4xl mx-auto space-y-8">
		{#if loading}
			<div class="text-center text-slate-500">Cargando...</div>
		{:else if event}
			<!-- Event Header -->
			<div class="bg-white rounded-xl shadow-sm border border-slate-100 p-6 sm:p-8">
				<h1 class="text-3xl font-extrabold text-slate-900 mb-2">{event.title}</h1>
				<div class="grid grid-cols-1 sm:grid-cols-2 gap-4 text-slate-600">
					<div>
						<span class="font-semibold block text-xs uppercase tracking-wider text-slate-400 mb-1">Organiza</span>
						{event.creator_name}
					</div>
					<div>
						<span class="font-semibold block text-xs uppercase tracking-wider text-slate-400 mb-1">Cuándo</span>
						{new Date(event.event_datetime).toLocaleString()}
					</div>
					<div class="sm:col-span-2">
						<span class="font-semibold block text-xs uppercase tracking-wider text-slate-400 mb-1">Dónde</span>
						{#if event.location.startsWith('http')}
							<a href={event.location} target="_blank" rel="noopener noreferrer" class="text-indigo-600 hover:underline flex items-center gap-1">
								📍 Ver en mapa
							</a>
						{:else}
							{event.location}
						{/if}
					</div>
				</div>
			</div>

			<div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
				<!-- Join Form -->
				<div class="bg-white rounded-xl shadow-sm border border-slate-100 p-6">
					<h2 class="text-xl font-bold text-slate-900 mb-6">Sumarme a la Juntada</h2>
					<form onsubmit={(e) => { e.preventDefault(); handleJoin(); }} class="space-y-4">
						<div>
							<label for="name" class="block text-sm font-medium text-slate-700 mb-1">Nombre</label>
							<input type="text" id="name" bind:value={name} required class="w-full px-3 py-2 border border-slate-300 rounded-md focus:ring-indigo-500 focus:border-indigo-500" placeholder="Tu nombre" />
						</div>
						<div>
							<label for="email" class="block text-sm font-medium text-slate-700 mb-1">Email (Opcional)</label>
							<input type="email" id="email" bind:value={email} class="w-full px-3 py-2 border border-slate-300 rounded-md focus:ring-indigo-500 focus:border-indigo-500" placeholder="tu@email.com" />
						</div>
						<div>
							<label for="origin" class="block text-sm font-medium text-slate-700 mb-1">¿Desde dónde vas?</label>
							<input type="text" id="origin" bind:value={origin} class="w-full px-3 py-2 border border-slate-300 rounded-md focus:ring-indigo-500 focus:border-indigo-500" placeholder="Barrio / Zona" />
						</div>
						<div class="flex items-center gap-2">
							<input type="checkbox" id="is_driver" bind:checked={is_driver} class="h-4 w-4 text-indigo-600 focus:ring-indigo-500 border-slate-300 rounded" />
							<label for="is_driver" class="text-sm font-medium text-slate-700">Voy en auto 🚗</label>
						</div>
						{#if is_driver}
							<div>
								<label for="car_capacity" class="block text-sm font-medium text-slate-700 mb-1">Lugares disponibles</label>
								<input type="number" id="car_capacity" bind:value={car_capacity} min="1" class="w-full px-3 py-2 border border-slate-300 rounded-md focus:ring-indigo-500 focus:border-indigo-500" />
							</div>
						{/if}
						<div>
							<label for="notes" class="block text-sm font-medium text-slate-700 mb-1">Notas (Opcional)</label>
							<textarea id="notes" bind:value={notes} rows="2" class="w-full px-3 py-2 border border-slate-300 rounded-md focus:ring-indigo-500 focus:border-indigo-500" placeholder="Llevo hielo, salgo tarde, etc."></textarea>
						</div>
						<button type="submit" disabled={joinLoading} class="w-full py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 disabled:opacity-50">
							{joinLoading ? 'Uniéndome...' : 'Confirmar Asistencia'}
						</button>
					</form>
				</div>

				<!-- Attendees List -->
				<div class="space-y-4">
					<h2 class="text-xl font-bold text-slate-900">Asistentes ({attendees.length})</h2>
					<div class="space-y-3">
						{#if attendees.length === 0}
							<p class="text-slate-500 italic">Aún no hay nadie anotado. ¡Sé el primero!</p>
						{:else}
							{#each attendees as attendee}
								<div class="bg-white rounded-lg p-4 border border-slate-100 shadow-sm flex justify-between items-start">
									<div>
										<div class="font-medium text-slate-900 flex items-center gap-2">
											{attendee.name}
											{#if attendee.is_driver}
												<span class="inline-flex items-center px-2 py-0.5 rounded text-xs font-medium bg-green-100 text-green-800">
													Conductor 🚗
												</span>
											{/if}
										</div>
										{#if attendee.origin}
											<div class="text-sm text-slate-500 mt-1">Desde: {attendee.origin}</div>
										{/if}
										{#if attendee.notes}
											<div class="text-sm text-slate-500 italic mt-1">"{attendee.notes}"</div>
										{/if}
									</div>
									{#if attendee.is_driver}
										<div class="text-right">
											<div class="text-xs text-slate-500 font-medium">Lugares</div>
											<div class="text-lg font-bold text-indigo-600">{attendee.car_capacity}</div>
										</div>
									{/if}
								</div>
							{/each}
						{/if}
					</div>
				</div>
			</div>
		{:else}
			<div class="text-center text-red-500">Juntada no encontrada</div>
		{/if}
	</div>
</div>
