<script lang="ts">
	import { onMount } from 'svelte';
	import { supabase } from '$lib/supabaseClient';

	let { data } = $props();
	let token = data.token;

	let driver: any = $state(null);
	let event: any = $state(null);
	let rideRequests: any[] = $state([]);
	let loading = $state(true);

	onMount(async () => {
		await fetchDriverAndRequests();
		loading = false;
	});

	async function fetchDriverAndRequests() {
		// Fetch driver by access token
		const { data: driverData, error: driverError } = await supabase
			.from('event_attendees')
			.select('*, event:events(*)')
			.eq('access_token', token)
			.single();

		if (driverError || !driverData) {
			console.error('Error fetching driver:', driverError);
			return;
		}

		driver = driverData;
		event = driverData.event;

		// Fetch pending ride requests for this driver
		const { data: requestsData, error: requestsError } = await supabase
			.from('ride_requests')
			.select(`
				*,
				passenger:event_attendees!ride_requests_passenger_attendee_id_fkey(name, origin, email)
			`)
			.eq('driver_attendee_id', driver.id)
			.eq('status', 'pending');

		if (requestsError) {
			console.error('Error fetching requests:', requestsError);
		} else {
			rideRequests = requestsData || [];
		}
	}

	async function handleRequest(requestId: string, status: 'accepted' | 'rejected') {
		try {
			const { error } = await supabase
				.from('ride_requests')
				.update({ 
					status,
					responded_at: new Date().toISOString()
				})
				.eq('id', requestId);

			if (error) throw error;

			alert(status === 'accepted' ? '¡Solicitud aceptada!' : 'Solicitud rechazada');
			await fetchDriverAndRequests();
		} catch (error) {
			console.error('Error updating request:', error);
			alert('Hubo un error al actualizar la solicitud.');
		}
	}
</script>

<div class="min-h-screen bg-slate-50 py-12 px-4 sm:px-6 lg:px-8">
	<div class="max-w-2xl mx-auto">
		{#if loading}
			<div class="text-center text-slate-500">Cargando...</div>
		{:else if driver && event}
			<!-- Header -->
			<div class="bg-white rounded-xl shadow-sm border border-slate-100 p-6 mb-6">
				<h1 class="text-2xl font-bold text-slate-900 mb-2">Tus Solicitudes de Viaje</h1>
				<div class="text-slate-600">
					<div class="font-medium">{event.title}</div>
					<div class="text-sm">{new Date(event.event_datetime).toLocaleString()}</div>
				</div>
				<div class="mt-3 pt-3 border-t border-slate-100">
					<div class="text-sm text-slate-600">
						<span class="font-medium">Conductor:</span> {driver.name}
					</div>
					<div class="text-sm text-slate-600">
						<span class="font-medium">Lugares disponibles:</span> {driver.car_capacity}
					</div>
				</div>
			</div>

			<!-- Ride Requests -->
			<div class="space-y-4">
				<h2 class="text-xl font-bold text-slate-900">
					Solicitudes Pendientes ({rideRequests.length})
				</h2>

				{#if rideRequests.length === 0}
					<div class="bg-white rounded-lg p-8 text-center border border-slate-100">
						<div class="text-slate-400 text-5xl mb-3">✓</div>
						<p class="text-slate-600">No tenés solicitudes pendientes</p>
					</div>
				{:else}
					{#each rideRequests as request}
						<div class="bg-white rounded-lg p-5 border border-slate-200 shadow-sm">
							<div class="flex justify-between items-start gap-4">
								<div class="flex-1">
									<div class="font-semibold text-lg text-slate-900 mb-2">
										{request.passenger.name}
									</div>
									{#if request.passenger.origin}
										<div class="text-sm text-slate-600 mb-1">
											📍 Desde: <span class="font-medium">{request.passenger.origin}</span>
										</div>
									{/if}
									{#if request.passenger.email}
										<div class="text-sm text-slate-600 mb-1">
											✉️ {request.passenger.email}
										</div>
									{/if}
									{#if request.message}
										<div class="mt-3 p-3 bg-slate-50 rounded-lg border border-slate-200">
											<div class="text-xs text-slate-500 uppercase font-medium mb-1">Mensaje</div>
											<div class="text-sm text-slate-700 italic">"{request.message}"</div>
										</div>
									{/if}
								</div>
								<div class="flex flex-col gap-2">
									<button
										onclick={() => handleRequest(request.id, 'accepted')}
										class="px-4 py-2 bg-green-600 text-white text-sm font-medium rounded-lg hover:bg-green-700 transition-colors shadow-sm"
									>
										✓ Aceptar
									</button>
									<button
										onclick={() => handleRequest(request.id, 'rejected')}
										class="px-4 py-2 bg-red-600 text-white text-sm font-medium rounded-lg hover:bg-red-700 transition-colors shadow-sm"
									>
										✗ Rechazar
									</button>
								</div>
							</div>
						</div>
					{/each}
				{/if}
			</div>

			<!-- Footer -->
			<div class="mt-8 text-center">
				<a href="/event/{event.id}" class="text-indigo-600 hover:underline text-sm font-medium">
					← Ver página completa del evento
				</a>
			</div>
		{:else}
			<div class="bg-white rounded-lg p-8 text-center border border-slate-100">
				<div class="text-red-500 text-5xl mb-3">⚠️</div>
				<h2 class="text-xl font-bold text-slate-900 mb-2">Link inválido</h2>
				<p class="text-slate-600">Este link de acceso no es válido o ha expirado.</p>
			</div>
		{/if}
	</div>
</div>
