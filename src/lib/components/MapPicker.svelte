<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	let { onConfirm, onCancel } = $props();

	let mapElement: HTMLElement;
	let map: any;
	let marker: any;
	let selectedLat = $state<number | null>(null);
	let selectedLng = $state<number | null>(null);
	let searchQuery = $state('');
	let searchResults = $state<any[]>([]);
	let searching = $state(false);

	onMount(async () => {
		if (browser) {
			const L = (await import('leaflet')).default;

			// Import Leaflet CSS
			const link = document.createElement('link');
			link.rel = 'stylesheet';
			link.href = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css';
			document.head.appendChild(link);

			// Default to Buenos Aires or a neutral location if geolocation fails
			const defaultLat = -34.6037;
			const defaultLng = -58.3816;

			map = L.map(mapElement).setView([defaultLat, defaultLng], 13);

			L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
				attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
			}).addTo(map);

			// Try to get current location to center map
			if (navigator.geolocation) {
				navigator.geolocation.getCurrentPosition((position) => {
					const lat = position.coords.latitude;
					const lng = position.coords.longitude;
					map.setView([lat, lng], 15);
				});
			}

			map.on('click', (e: any) => {
				const { lat, lng } = e.latlng;
				setMarker(lat, lng);
			});
		}
	});

	onDestroy(() => {
		if (map) {
			map.remove();
		}
	});

	function setMarker(lat: number, lng: number) {
		selectedLat = lat;
		selectedLng = lng;

		if (browser) {
			import('leaflet').then(({ default: L }) => {
				if (marker) {
					marker.setLatLng([lat, lng]);
				} else {
					marker = L.marker([lat, lng]).addTo(map);
				}
			});
		}
	}

	async function searchLocation() {
		if (!searchQuery.trim()) return;

		searching = true;
		try {
			// Use Nominatim (OpenStreetMap's geocoding service)
			const response = await fetch(
				`https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(searchQuery.trim())}&limit=5`
			);
			const results = await response.json();
			searchResults = results;
		} catch (error) {
			console.error('Error searching location:', error);
			alert('Error al buscar la ubicación');
		} finally {
			searching = false;
		}
	}

	function selectSearchResult(result: any) {
		const lat = parseFloat(result.lat);
		const lng = parseFloat(result.lon);
		
		map.setView([lat, lng], 15);
		setMarker(lat, lng);
		searchResults = [];
		searchQuery = result.display_name;
	}

	function handleConfirm() {
		if (selectedLat !== null && selectedLng !== null) {
			onConfirm(selectedLat, selectedLng);
		} else {
			alert('Por favor selecciona una ubicación en el mapa');
		}
	}
</script>

<div class="fixed inset-0 z-50 flex items-center justify-center bg-black/50 backdrop-blur-sm p-4">
	<div class="bg-white rounded-xl shadow-2xl w-full max-w-3xl overflow-hidden flex flex-col max-h-[90vh]">
		<div class="p-4 border-b border-slate-100 flex justify-between items-center relative z-10 bg-white">
			<h3 class="text-lg font-semibold text-slate-800">Seleccionar Ubicación</h3>
			<button onclick={onCancel} class="text-slate-400 hover:text-slate-600">
				✕
			</button>
		</div>
		
		<!-- Search Bar -->
		<div class="p-4 bg-slate-50 border-b border-slate-200 relative z-10">
			<div class="flex gap-2">
				<input
					type="text"
					bind:value={searchQuery}
					onkeydown={(e) => e.key === 'Enter' && searchLocation()}
					placeholder="Buscar ubicación (ej: Palermo, Buenos Aires)"
					class="flex-1 px-4 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 outline-none"
				/>
				<button
					onclick={searchLocation}
					disabled={searching}
					class="px-4 py-2 bg-indigo-600 text-white rounded-lg hover:bg-indigo-700 disabled:opacity-50 transition-colors"
				>
					{searching ? 'Buscando...' : 'Buscar'}
				</button>
			</div>
			
			<!-- Search Results -->
			{#if searchResults.length > 0}
				<div class="mt-2 bg-white border border-slate-200 rounded-lg shadow-lg max-h-48 overflow-y-auto">
					{#each searchResults as result}
						<button
							onclick={() => selectSearchResult(result)}
							class="w-full text-left px-4 py-2 hover:bg-slate-50 border-b border-slate-100 last:border-b-0 transition-colors"
						>
							<div class="text-sm font-medium text-slate-900">{result.display_name}</div>
						</button>
					{/each}
				</div>
			{/if}
		</div>

		<div class="flex-1 relative min-h-[400px]">
			<div bind:this={mapElement} class="absolute inset-0 z-0"></div>
		</div>

		<div class="p-4 border-t border-slate-100 flex justify-end gap-3 bg-slate-50 relative z-10">
			<button
				onclick={onCancel}
				class="px-4 py-2 rounded-lg text-slate-600 hover:bg-slate-200 font-medium transition-colors"
			>
				Cancelar
			</button>
			<button
				onclick={handleConfirm}
				disabled={selectedLat === null}
				class="px-4 py-2 rounded-lg bg-indigo-600 text-white font-medium hover:bg-indigo-700 disabled:opacity-50 disabled:cursor-not-allowed transition-colors shadow-sm cursor-pointer disabled:cursor-not-allowed"
			>
				Confirmar Ubicación
			</button>
		</div>
	</div>
</div>

<style>
	:global(.leaflet-control-attribution) {
		display: none;
	}
</style>
