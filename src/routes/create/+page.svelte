<script lang="ts">
	import { supabase } from '$lib/supabaseClient';
	import { goto } from '$app/navigation';
	import MapPicker from '$lib/components/MapPicker.svelte';
	import EventCodeModal from '$lib/components/EventCodeModal.svelte';

	let title = '';
	let creator_name = '';
	let creator_email = '';
	let location = '';
	let event_datetime = '';
	let loading = false;
	let showMapPicker = false;
	let showEventCodeModal = false;
	let createdEventId = '';

	function getLocation() {
		if (!navigator.geolocation) {
			alert('Tu navegador no soporta geolocalización');
			return;
		}

		navigator.geolocation.getCurrentPosition(
			(position) => {
				const lat = position.coords.latitude;
				const lng = position.coords.longitude;
				location = `https://www.google.com/maps/search/?api=1&query=${lat},${lng}`;
			},
			(error) => {
				console.error('Error getting location:', error);
				alert('No se pudo obtener tu ubicación. Por favor ingrésala manualmente.');
			}
		);
	}

	async function handleSubmit() {
		loading = true;
		try {
			// Convert datetime-local to ISO string with timezone
			// datetime-local gives us "2025-11-28T22:50" (no timezone)
			// We need to append the local timezone offset
			const datetimeWithTimezone = event_datetime ? new Date(event_datetime).toISOString() : null;
			
			const eventData = {
				title: title.trim(),
				creator_name: creator_name.trim(),
				creator_email: creator_email.trim(),
				location: location.trim(),
				event_datetime: datetimeWithTimezone
			};
			
			console.log('🚀 Intentando crear juntada con los siguientes datos:', eventData);
			console.log('📅 Fecha original:', event_datetime);
			console.log('📅 Fecha convertida a ISO:', datetimeWithTimezone);
			
			const { data, error } = await supabase
				.from('events')
				.insert([eventData])
				.select();

			console.log('📊 Respuesta de Supabase:', { data, error });

			if (error) {
				console.error('❌ Error de Supabase:', error);
				console.error('❌ Detalles del error:', JSON.stringify(error, null, 2));
				throw error;
			}

			console.log('✅ Juntada creada exitosamente:', data);
		
			// Get the event ID from the created event
			const eventId = data && data[0] ? data[0].id : null;
			
			if (eventId) {
				createdEventId = eventId;
				showEventCodeModal = true;
			} else {
				alert('¡Juntada creada con éxito!');
				goto('/');
			}
		} catch (error) {
			console.error('💥 Error capturado en catch:', error);
			console.error('💥 Tipo de error:', typeof error);
			console.error('💥 Error completo:', JSON.stringify(error, null, 2));
			alert('Hubo un error al crear la juntada. Por favor intenta de nuevo.');
		} finally {
			loading = false;
		}
	}
</script>

<div class="min-h-screen bg-slate-50 py-12 px-4 sm:px-6 lg:px-8 flex flex-col items-center">
	<div class="max-w-md w-full space-y-8">
		<div>
			<h2 class="mt-6 text-center text-3xl font-extrabold text-slate-900">
				Crear nueva Juntada
			</h2>
			<p class="mt-2 text-center text-sm text-slate-600">
				Completa los datos para organizar tu encuentro
			</p>
		</div>
		<form class="mt-8 space-y-6 bg-white p-8 rounded-xl shadow-lg border border-slate-100" on:submit|preventDefault={handleSubmit}>
			<div class="rounded-md shadow-sm space-y-4">
				<div>
					<label for="title" class="block text-sm font-medium text-slate-700 mb-1">Nombre de la Juntada</label>
					<input
						id="title"
						name="title"
						type="text"
						required
						class="appearance-none relative block w-full px-3 py-2 border border-slate-300 placeholder-slate-500 text-slate-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm"
						placeholder="Ej: Asado fin de año"
						bind:value={title}
					/>
				</div>
				<div>
					<label for="creator_name" class="block text-sm font-medium text-slate-700 mb-1">Tu Nombre</label>
					<input
						id="creator_name"
						name="creator_name"
						type="text"
						required
						class="appearance-none relative block w-full px-3 py-2 border border-slate-300 placeholder-slate-500 text-slate-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm"
						placeholder="Ej: Juan Pérez"
						bind:value={creator_name}
					/>
				</div>
				<div>
					<label for="creator_email" class="block text-sm font-medium text-slate-700 mb-1">Tu Email</label>
					<input
						id="creator_email"
						name="creator_email"
						type="email"
						required
						class="appearance-none relative block w-full px-3 py-2 border border-slate-300 placeholder-slate-500 text-slate-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm"
						placeholder="juan@ejemplo.com"
						bind:value={creator_email}
					/>
				</div>
				<div>
					<label for="location" class="block text-sm font-medium text-slate-700 mb-1">Ubicación</label>
					<div class="flex gap-2">
						<input
							id="location"
							name="location"
							type="text"
							required
							class="appearance-none relative block w-full px-3 py-2 border border-slate-300 placeholder-slate-500 text-slate-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm"
							placeholder="Ej: Parque Centenario / Calle Falsa 123"
							bind:value={location}
						/>
						<button
							type="button"
							on:click={getLocation}
							class="px-3 py-2 border border-slate-300 rounded-md text-sm font-medium text-slate-700 hover:bg-slate-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
							title="Usar mi ubicación actual"
						>
							📍
						</button>
						<button
							type="button"
							on:click={() => (showMapPicker = true)}
							class="px-3 py-2 border border-slate-300 rounded-md text-sm font-medium text-slate-700 hover:bg-slate-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500"
							title="Seleccionar en mapa"
						>
							🗺️
						</button>
					</div>
				</div>
				<div>
					<label for="event_datetime" class="block text-sm font-medium text-slate-700 mb-1">Fecha y Hora</label>
					<input
						id="event_datetime"
						name="event_datetime"
						type="datetime-local"
						required
						class="appearance-none relative block w-full px-3 py-2 border border-slate-300 placeholder-slate-500 text-slate-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm"
						bind:value={event_datetime}
					/>
				</div>
			</div>

			<div>
				<button
					type="submit"
					disabled={loading}
					class="group relative w-full flex justify-center py-2 px-4 border border-transparent text-sm font-medium rounded-md text-white bg-indigo-600 hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-indigo-500 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
				>
					{#if loading}
						Creando...
					{:else}
						Crear Juntada
					{/if}
				</button>
			</div>
		</form>
		<div class="text-center">
			<a href="/" class="font-medium text-indigo-600 hover:text-indigo-500">
				&larr; Volver al inicio
			</a>
		</div>
	</div>
</div>

{#if showMapPicker}
	<MapPicker
		onConfirm={(lat: number, lng: number) => {
			location = `https://www.google.com/maps/search/?api=1&query=${lat},${lng}`;
			showMapPicker = false;
		}}
		onCancel={() => (showMapPicker = false)}
	/>
{/if}

{#if showEventCodeModal}
	<EventCodeModal
		eventId={createdEventId}
		onClose={() => {
			showEventCodeModal = false;
			goto('/');
		}}
	/>
{/if}

