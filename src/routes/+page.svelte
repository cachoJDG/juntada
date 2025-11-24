<script lang="ts">
	// No script needed for now
</script>

<div class="min-h-screen bg-slate-50 flex flex-col items-center justify-center p-4 selection:bg-indigo-100 selection:text-indigo-900">
	<main class="max-w-2xl w-full text-center space-y-8">
		<div class="space-y-4">
			<h1 class="text-6xl font-extrabold tracking-tight text-slate-900 sm:text-7xl">
				Juntada <span class="text-indigo-600">app</span>
			</h1>
			<p class="text-xl text-slate-600 max-w-lg mx-auto leading-relaxed">
				La forma más simple de organizar tus encuentros. <br class="hidden sm:block" />
				Sin complicaciones, solo buenos momentos.
			</p>
		</div>

		<div class="flex flex-col sm:flex-row gap-4 justify-center items-center pt-8">
			<a href="/create" class="px-8 py-3 rounded-full bg-indigo-600 text-white font-medium hover:bg-indigo-700 transition-colors shadow-lg shadow-indigo-200 hover:shadow-xl hover:shadow-indigo-200 transform hover:-translate-y-0.5 duration-200">
				Crear Juntada
			</a>
			<button class="px-8 py-3 rounded-full bg-white text-slate-700 font-medium border border-slate-200 hover:border-slate-300 hover:bg-slate-50 transition-all shadow-sm hover:shadow-md">
				Unirse a una
			</button>
		</div>
	</main>

	<footer class="absolute bottom-8 text-slate-400 text-sm font-medium">
		&copy; {new Date().getFullYear()} Juntada App
	</footer>
</div>
