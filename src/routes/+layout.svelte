<script lang="ts">
	import favicon from '$lib/assets/flask.svg';
	import '../app.css';
	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<meta name="viewport" content="width=device-width, initial-scale=1" />
	<meta name="author" content="Indest Labs" />
	<title>Mandelbulb by Sean Bradley</title>
	<meta name="description" content="Mandelbulb representation in three.js using TSL" />
</svelte:head>

{@render children()}
