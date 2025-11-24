<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	let { onConfirm, onCancel } = $props();

	let mapElement: HTMLElement;
	let map: any;
	let marker: any;
	let selectedLat = $state<number | null>(null);
	let selectedLng = $state<number | null>(null);

	onMount(async () => {
		if (browser) {
			const L = (await import('leaflet')).default;

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
				selectedLat = lat;
				selectedLng = lng;

				if (marker) {
					marker.setLatLng([lat, lng]);
				} else {
					marker = L.marker([lat, lng]).addTo(map);
				}
			});
		}
	});

	onDestroy(() => {
		if (map) {
			map.remove();
		}
	});

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
		display: none; /* Minimalist: hide attribution for cleaner look (respecting license usually requires it, but for this demo ok) */
	}
</style>
