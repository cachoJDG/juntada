<script lang="ts">
	import { onMount } from 'svelte';
	import { supabase } from '$lib/supabaseClient';
	import { page } from '$app/stores';

	let { data } = $props();
	let eventId = data.id;

	let event: any = $state(null);
	let attendees: any[] = $state([]);
	let rideRequests: any[] = $state([]);
	let loading = $state(true);
	let joinLoading = $state(false);
	let loginLoading = $state(false);
	
	// Session State
	let sessionEmail = $state<string | null>(null);
	let currentAttendee = $state<any>(null);
	let showLoginForm = $state(false);
	let loginEmail = $state('');

	// Join Form State
	let name = $state('');
	let email = $state('');
	let origin = $state('');
	let arrival_time = $state('');
	let is_driver = $state(false);
	let car_capacity = $state(0);
	let notes = $state('');

	// Ride Request Modal State
	let showRideRequestModal = $state(false);
	let selectedDriverId = $state<string | null>(null);
	let rideRequestMessage = $state('');
	let requestLoading = $state(false);

	// Derived State
	let drivers = $derived(attendees.filter(a => a.is_driver));
	let passengers = $derived(attendees.filter(a => !a.is_driver));
	let mySentRequests = $derived(
		currentAttendee && !currentAttendee.is_driver
			? rideRequests.filter(r => r.passenger_attendee_id === currentAttendee.id)
			: []
	);

	onMount(async () => {
		await fetchEventDetails();
		await fetchAttendees();
		await fetchRideRequests();
		
		// Check for existing session
		const storedEmail = localStorage.getItem(`session_email_${eventId}`);
		if (storedEmail) {
			await loginWithEmail(storedEmail);
		}
		
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

	async function fetchRideRequests() {
		const { data, error } = await supabase
			.from('ride_requests')
			.select(`
				*,
				passenger:event_attendees!ride_requests_passenger_attendee_id_fkey(name, origin),
				driver:event_attendees!ride_requests_driver_attendee_id_fkey(name, car_capacity)
			`)
			.eq('event_id', eventId);

		if (error) {
			console.error('Error fetching ride requests:', error);
		} else {
			rideRequests = data || [];
		}
	}

	function copyLink() {
		navigator.clipboard.writeText(window.location.href);
		alert('¡Link copiado al portapapeles!');
	}

	async function loginWithEmail(emailToLogin: string) {
		const { data: attendeeData, error } = await supabase
			.from('event_attendees')
			.select('*')
			.eq('event_id', eventId)
			.eq('email', emailToLogin)
			.single();

		if (error || !attendeeData) {
			console.error('Login error:', error);
			return false;
		}

		sessionEmail = emailToLogin;
		currentAttendee = attendeeData;
		localStorage.setItem(`session_email_${eventId}`, emailToLogin);
		return true;
	}

	async function handleLogin() {
		if (!loginEmail.trim()) {
			alert('Por favor ingresá tu email');
			return;
		}

		loginLoading = true;
		const success = await loginWithEmail(loginEmail.trim());
		
		if (success) {
			showLoginForm = false;
			loginEmail = '';
		} else {
			alert('No encontramos tu email en esta juntada. ¿Ya te sumaste?');
		}
		loginLoading = false;
	}

	function handleLogout() {
		sessionEmail = null;
		currentAttendee = null;
		localStorage.removeItem(`session_email_${eventId}`);
	}

	async function handleJoin() {
		if (!email.trim()) {
			alert('El email es obligatorio');
			return;
		}

		joinLoading = true;
		try {
			const { data: newAttendee, error } = await supabase
				.from('event_attendees')
				.insert([
					{
						event_id: eventId,
						name: name.trim(),
						email: email.trim(),
						origin: origin.trim(),
						arrival_time: arrival_time || null,
						is_driver,
						car_capacity: is_driver ? car_capacity : null,
						notes: notes ? notes.trim() : null
					}
				])
				.select()
				.single();

			if (error) throw error;

			// Auto-login after joining
			await loginWithEmail(email.trim());

			// Show success message with access link for drivers
			if (is_driver && newAttendee.access_token) {
				const accessLink = `${window.location.origin}/driver/${newAttendee.access_token}`;
				alert(`¡Te has unido a la juntada!\n\nTu link de acceso para ver solicitudes:\n${accessLink}\n\nGuardá este link para recibir notificaciones de solicitudes.`);
			} else {
				alert('¡Te has unido a la juntada!');
			}
			
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
		} catch (error: any) {
			console.error('Error joining event:', error);
			if (error.code === '23505') { // Unique constraint violation
				alert('Este email ya está registrado en esta juntada. Usá "Ingresar" para acceder.');
			} else {
				alert('Hubo un error al unirse. Intenta de nuevo.');
			}
		} finally {
			joinLoading = false;
		}
	}

	function openRideRequestModal(driverId: string) {
		if (!currentAttendee) {
			alert('Debes ingresar primero');
			return;
		}
		selectedDriverId = driverId;
		showRideRequestModal = true;
	}

	async function sendRideRequest() {
		if (!currentAttendee || !selectedDriverId) {
			alert('Debes ingresar primero');
			return;
		}

		requestLoading = true;
		try {
			const { error } = await supabase
				.from('ride_requests')
				.insert([
					{
						event_id: eventId,
						driver_attendee_id: selectedDriverId,
						passenger_attendee_id: currentAttendee.id,
						message: rideRequestMessage ? rideRequestMessage.trim() : null,
						status: 'pending'
					}
				]);

			if (error) throw error;

			alert('¡Solicitud enviada!');
			showRideRequestModal = false;
			rideRequestMessage = '';
			await fetchRideRequests();
		} catch (error) {
			console.error('Error sending ride request:', error);
			alert('Hubo un error al enviar la solicitud.');
		} finally {
			requestLoading = false;
		}
	}

	async function handleRideRequest(requestId: string, status: 'accepted' | 'rejected') {
		try {
			// Update ride request status
			const { error: requestError } = await supabase
				.from('ride_requests')
				.update({ 
					status,
					responded_at: new Date().toISOString()
				})
				.eq('id', requestId);

			if (requestError) throw requestError;

			// If accepted, decrement the driver's car capacity
			if (status === 'accepted' && currentAttendee && currentAttendee.car_capacity > 0) {
				const { error: capacityError } = await supabase
					.from('event_attendees')
					.update({ 
						car_capacity: currentAttendee.car_capacity - 1
					})
					.eq('id', currentAttendee.id);

				if (capacityError) throw capacityError;

				// Update local state
				currentAttendee.car_capacity -= 1;
			}

			alert(status === 'accepted' ? '¡Solicitud aceptada!' : 'Solicitud rechazada');
			await fetchRideRequests();
			await fetchAttendees(); // Refresh to show updated capacity
		} catch (error) {
			console.error('Error updating ride request:', error);
			alert('Hubo un error al actualizar la solicitud.');
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
				<div class="flex justify-between items-start mb-2">
					<h1 class="text-3xl font-extrabold text-slate-900">{event.title}</h1>
					<button
						onclick={copyLink}
						class="flex items-center gap-2 px-3 py-1.5 text-sm font-medium text-indigo-600 bg-indigo-50 hover:bg-indigo-100 rounded-lg transition-colors"
						title="Copiar link del evento"
					>
						🔗 <span class="hidden sm:inline">Compartir</span>
					</button>
				</div>
				<div class="grid grid-cols-1 sm:grid-cols-2 gap-4 text-slate-600">
					<div>
					<span class="font-semibold block text-xs uppercase tracking-wider text-slate-400 mb-1">Organiza</span>
					<div class="font-medium">{event.creator_name}</div>
					{#if event.creator_email}
						<a href="mailto:{event.creator_email}" class="text-sm text-indigo-600 hover:underline">
							{event.creator_email}
						</a>
					{/if}
				</div>
					<div>
						<span class="font-semibold block text-xs uppercase tracking-wider text-slate-400 mb-1">Cuándo</span>
						{new Date(event.event_datetime).toLocaleString('es-AR', { 
							dateStyle: 'long', 
							timeStyle: 'short',
							timeZone: 'America/Argentina/Buenos_Aires'
						})}
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
				<!-- Session / Join / Login Section -->
				<div class="bg-white rounded-xl shadow-sm border border-slate-100 p-6">
					{#if currentAttendee}
						<!-- Logged In State -->
						<div class="space-y-4">
							<div class="flex justify-between items-start">
								<div>
									<h2 class="text-xl font-bold text-slate-900">¡Hola, {currentAttendee.name}!</h2>
									<p class="text-sm text-slate-600 mt-1">{currentAttendee.email}</p>
								</div>
								<button
									onclick={handleLogout}
									class="text-sm text-slate-500 hover:text-slate-700 underline"
								>
									Cerrar sesión
								</button>
							</div>
							
							<div class="pt-4 border-t border-slate-100 space-y-2">
								<div class="text-sm">
									<span class="font-medium text-slate-700">Desde:</span>
									<span class="text-slate-600">{currentAttendee.origin || 'No especificado'}</span>
								</div>
								{#if currentAttendee.is_driver}
									<div class="flex items-center gap-2">
										<span class="inline-flex items-center px-2 py-0.5 rounded text-xs font-medium bg-green-100 text-green-800">
											Conductor 🚗
										</span>
										<span class="text-sm text-slate-600">{currentAttendee.car_capacity} lugares</span>
									</div>
								{:else}
									<!-- Passenger Dashboard -->
									{#if mySentRequests.length > 0}
										<div class="mt-3 pt-3 border-t border-slate-100">
											<h4 class="text-sm font-bold text-slate-900 mb-2">Mis Solicitudes de Viaje</h4>
											<div class="space-y-2">
												{#each mySentRequests as request}
													<div class="bg-slate-50 rounded p-2 text-sm border border-slate-200">
														<div class="flex justify-between items-center mb-1">
															<span class="font-medium text-slate-700">
																A: {request.driver?.name || 'Conductor'}
															</span>
															{#if request.status === 'pending'}
																<span class="px-2 py-0.5 rounded-full text-xs font-medium bg-yellow-100 text-yellow-800">
																	⏳ Pendiente
																</span>
															{:else if request.status === 'accepted'}
																<span class="px-2 py-0.5 rounded-full text-xs font-medium bg-green-100 text-green-800">
																	✅ Aceptada
																</span>
															{:else}
																<span class="px-2 py-0.5 rounded-full text-xs font-medium bg-red-100 text-red-800">
																	❌ Rechazada
																</span>
															{/if}
														</div>
														{#if request.status === 'accepted'}
															<div class="text-xs text-green-700 mt-1">
																¡Viajas con {request.driver?.name}! 🎉
															</div>
														{/if}
													</div>
												{/each}
											</div>
										</div>
									{/if}
								{/if}
								{#if currentAttendee.notes}
									<div class="text-sm">
										<span class="font-medium text-slate-700">Notas:</span>
										<span class="text-slate-600 italic">"{currentAttendee.notes}"</span>
									</div>
								{/if}
							</div>
						</div>
					{:else}
						<!-- Not Logged In - Show Tabs -->
						<div class="space-y-6">
							<div class="flex gap-2 border-b border-slate-200">
								<button
									onclick={() => (showLoginForm = false)}
									class="px-4 py-2 text-sm font-medium transition-colors {!showLoginForm ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-600 hover:text-slate-900'}"
								>
									Sumarme
								</button>
								<button
									onclick={() => (showLoginForm = true)}
									class="px-4 py-2 text-sm font-medium transition-colors {showLoginForm ? 'text-indigo-600 border-b-2 border-indigo-600' : 'text-slate-600 hover:text-slate-900'}"
								>
									Ingresar
								</button>
							</div>

							{#if showLoginForm}
								<!-- Login Form -->
								<form onsubmit={(e) => { e.preventDefault(); handleLogin(); }} class="space-y-4">
									<div>
										<label for="loginEmail" class="block text-sm font-medium text-slate-700 mb-1">Email</label>
										<input
											type="email"
											id="loginEmail"
											bind:value={loginEmail}
											required
											class="w-full px-3 py-2 border border-slate-300 rounded-md focus:ring-indigo-500 focus:border-indigo-500"
											placeholder="tu@email.com"
										/>
										<p class="text-xs text-slate-500 mt-1">Ingresá el email con el que te sumaste</p>
									</div>
									<button
										type="submit"
										disabled={loginLoading}
										class="w-full py-2 px-4 border border-transparent rounded-md shadow-sm text-sm font-medium text-white bg-indigo-600 hover:bg-indigo-700 disabled:opacity-50"
									>
										{loginLoading ? 'Ingresando...' : 'Ingresar'}
									</button>
								</form>
							{:else}
								<!-- Join Form -->
								<form onsubmit={(e) => { e.preventDefault(); handleJoin(); }} class="space-y-4">
									<div>
										<label for="name" class="block text-sm font-medium text-slate-700 mb-1">Nombre</label>
										<input type="text" id="name" bind:value={name} required class="w-full px-3 py-2 border border-slate-300 rounded-md focus:ring-indigo-500 focus:border-indigo-500" placeholder="Tu nombre" />
									</div>
									<div>
										<label for="email" class="block text-sm font-medium text-slate-700 mb-1">Email <span class="text-red-500">*</span></label>
										<input type="email" id="email" bind:value={email} required class="w-full px-3 py-2 border border-slate-300 rounded-md focus:ring-indigo-500 focus:border-indigo-500" placeholder="tu@email.com" />
										<p class="text-xs text-slate-500 mt-1">Necesario para ingresar después</p>
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
							{/if}
						</div>
					{/if}
				</div>

				<!-- Attendees List -->
				<!-- Attendees List -->
				<div class="space-y-8">
					<!-- Drivers Section -->
					<div>
						<h2 class="text-xl font-bold text-slate-900 mb-4 flex items-center gap-2">
							🚗 Conductores <span class="text-sm font-normal text-slate-500">({drivers.length})</span>
						</h2>
						{#if drivers.length === 0}
							<p class="text-slate-500 italic bg-slate-50 p-4 rounded-lg border border-slate-100 text-center">
								No hay conductores anotados todavía.
							</p>
						{:else}
							<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
								{#each drivers as driver}
									<div class="bg-white rounded-xl p-5 border border-slate-200 shadow-sm hover:shadow-md transition-shadow relative overflow-hidden">
										<!-- Capacity Indicator Bar -->
										<div class="absolute top-0 left-0 right-0 h-1 bg-slate-100">
											<div 
												class="h-full {driver.car_capacity > 0 ? 'bg-green-500' : 'bg-red-500'}" 
												style="width: {driver.car_capacity > 0 ? '100%' : '100%'}"
											></div>
										</div>

										<div class="flex justify-between items-start mt-2">
											<div>
												<div class="font-bold text-lg text-slate-900">{driver.name}</div>
												{#if driver.origin}
													<div class="text-sm text-slate-600 flex items-center gap-1 mt-1">
														📍 {driver.origin}
													</div>
												{/if}
												{#if driver.email}
													<a href="mailto:{driver.email}" class="text-xs text-indigo-600 hover:underline mt-1 block">
														{driver.email}
													</a>
												{/if}
												{#if driver.notes}
													<div class="text-sm text-slate-500 italic mt-2 bg-slate-50 p-2 rounded">
														"{driver.notes}"
													</div>
												{/if}
											</div>
											<div class="text-right">
												<div class="text-xs text-slate-500 font-medium uppercase tracking-wide">Lugares</div>
												<div class="text-2xl font-bold {driver.car_capacity > 0 ? 'text-green-600' : 'text-red-500'}">
													{driver.car_capacity}
												</div>
											</div>
										</div>

										<div class="mt-4 pt-4 border-t border-slate-100 flex justify-end">
											{#if currentAttendee && currentAttendee.id !== driver.id}
												{#if driver.car_capacity > 0}
													<button
														onclick={() => openRideRequestModal(driver.id)}
														class="w-full sm:w-auto px-4 py-2 bg-indigo-600 text-white text-sm font-medium rounded-lg hover:bg-indigo-700 transition-colors shadow-sm"
													>
														Pedir Lugar 🙋‍♂️
													</button>
												{:else}
													<button disabled class="w-full sm:w-auto px-4 py-2 bg-slate-100 text-slate-400 text-sm font-medium rounded-lg cursor-not-allowed">
														Completo 🚫
													</button>
												{/if}
											{/if}
										</div>
									</div>
								{/each}
							</div>
						{/if}
					</div>

					<!-- Passengers Section -->
					<div>
						<h2 class="text-xl font-bold text-slate-900 mb-4 flex items-center gap-2">
							🚶 Pasajeros <span class="text-sm font-normal text-slate-500">({passengers.length})</span>
						</h2>
						{#if passengers.length === 0}
							<p class="text-slate-500 italic">No hay pasajeros anotados todavía.</p>
						{:else}
							<div class="bg-white rounded-xl border border-slate-200 shadow-sm divide-y divide-slate-100">
								{#each passengers as passenger}
									<div class="p-4 flex justify-between items-center hover:bg-slate-50 transition-colors">
										<div>
											<div class="font-medium text-slate-900">{passenger.name}</div>
											{#if passenger.origin}
												<div class="text-sm text-slate-600">Desde: {passenger.origin}</div>
											{/if}
											{#if passenger.notes}
												<div class="text-xs text-slate-500 italic mt-1">"{passenger.notes}"</div>
											{/if}
										</div>
									</div>
								{/each}
							</div>
						{/if}
					</div>

					<!-- Ride Requests Section (for drivers) -->
					{#if currentAttendee}
						{@const myRequests = rideRequests.filter(r => r.driver_attendee_id === currentAttendee.id && r.status === 'pending')}
						{#if myRequests.length > 0}
							<div class="mt-6 pt-6 border-t border-slate-200">
								<h3 class="text-lg font-bold text-slate-900 mb-3">Solicitudes de Viaje ({myRequests.length})</h3>
								<div class="space-y-2">
									{#each myRequests as request}
										<div class="bg-amber-50 border border-amber-200 rounded-lg p-3">
											<div class="flex justify-between items-start">
												<div class="flex-1">
													<div class="font-medium text-slate-900">{request.passenger.name}</div>
													{#if request.passenger.origin}
														<div class="text-sm text-slate-600">Desde: {request.passenger.origin}</div>
													{/if}
													{#if request.message}
														<div class="text-sm text-slate-600 italic mt-1">"{request.message}"</div>
													{/if}
												</div>
												<div class="flex gap-2 ml-3">
													<button
														onclick={() => handleRideRequest(request.id, 'accepted')}
														class="px-3 py-1 bg-green-600 text-white text-sm rounded hover:bg-green-700"
													>
														✓ Aceptar
													</button>
													<button
														onclick={() => handleRideRequest(request.id, 'rejected')}
														class="px-3 py-1 bg-red-600 text-white text-sm rounded hover:bg-red-700"
													>
														✗ Rechazar
													</button>
												</div>
											</div>
										</div>
									{/each}
								</div>
							</div>
						{/if}
					{/if}
				</div>
			</div>
		{:else}
			<div class="text-center text-red-500">Juntada no encontrada</div>
		{/if}
	</div>
</div>

<!-- Ride Request Modal -->
{#if showRideRequestModal}
	<div class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-sm p-4">
		<div class="bg-white rounded-xl shadow-2xl w-full max-w-md overflow-hidden p-6 space-y-6">
			<div class="flex justify-between items-center">
				<h3 class="text-xl font-bold text-slate-900">Pedir Lugar</h3>
				<button onclick={() => (showRideRequestModal = false)} class="text-slate-400 hover:text-slate-600">
					✕
				</button>
			</div>
			
			<div class="space-y-4">
				<p class="text-sm text-slate-600">
					Envía un mensaje al conductor para solicitar un lugar en su auto.
				</p>
				<div>
					<label for="rideMessage" class="block text-sm font-medium text-slate-700 mb-1">Mensaje (Opcional)</label>
					<textarea
						id="rideMessage"
						bind:value={rideRequestMessage}
						rows="3"
						class="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 outline-none"
						placeholder="Ej: Paso por tu zona, ¿me podés llevar?"
					></textarea>
				</div>
			</div>

			<div class="flex justify-end gap-3">
				<button
					onclick={() => (showRideRequestModal = false)}
					class="px-4 py-2 rounded-lg text-slate-600 hover:bg-slate-100 font-medium transition-colors"
				>
					Cancelar
				</button>
				<button
					onclick={sendRideRequest}
					disabled={requestLoading}
					class="px-4 py-2 rounded-lg bg-indigo-600 text-white font-medium hover:bg-indigo-700 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
				>
					{requestLoading ? 'Enviando...' : 'Enviar Solicitud'}
				</button>
			</div>
		</div>
	</div>
{/if}
