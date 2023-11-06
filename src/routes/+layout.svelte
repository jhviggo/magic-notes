<script>
	import './styles.css';
</script>

<div class="app">
	<main>
		<slot />
	</main>

	<footer>
	</footer>
</div>

<style>
	.app {
		width: 100%;
	}

	main {
		height: 100%;
	}
</style>
