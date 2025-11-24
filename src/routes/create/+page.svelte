<script lang="ts">
	import { supabase } from '$lib/supabaseClient';
	import { goto } from '$app/navigation';

	let title = '';
	let creator_name = '';
	let creator_email = '';
	let location = '';
	let event_datetime = '';
	let loading = false;

	async function handleSubmit() {
		loading = true;
		try {
			const { data, error } = await supabase
				.from('events')
				.insert([
					{
						title,
						creator_name,
						creator_email,
						location,
						event_datetime
					}
				])
				.select();

			if (error) throw error;

			alert('¡Juntada creada con éxito!');
			goto('/');
		} catch (error) {
			console.error('Error creating event:', error);
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
					<label for="creator_email" class="block text-sm font-medium text-slate-700 mb-1">Tu Email (Opcional)</label>
					<input
						id="creator_email"
						name="creator_email"
						type="email"
						class="appearance-none relative block w-full px-3 py-2 border border-slate-300 placeholder-slate-500 text-slate-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm"
						placeholder="juan@ejemplo.com"
						bind:value={creator_email}
					/>
				</div>
				<div>
					<label for="location" class="block text-sm font-medium text-slate-700 mb-1">Ubicación</label>
					<input
						id="location"
						name="location"
						type="text"
						required
						class="appearance-none relative block w-full px-3 py-2 border border-slate-300 placeholder-slate-500 text-slate-900 rounded-md focus:outline-none focus:ring-indigo-500 focus:border-indigo-500 focus:z-10 sm:text-sm"
						placeholder="Ej: Parque Centenario / Calle Falsa 123"
						bind:value={location}
					/>
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
