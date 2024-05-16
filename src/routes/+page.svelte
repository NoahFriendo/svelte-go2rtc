<script lang="ts">
	import VideoRTC from '$lib/VideoRTC.svelte';

	let stream_name = 'zed';

	let error = '';
	let mode = '';
	let message = '';
</script>

<h1>VideoRTC Component</h1>

<VideoRTC
	src="http://localhost:1984/api/ws?src={stream_name}"
	on:error={(e) => (error = e.detail)}
	on:mode={(e) => (mode = e.detail)}
	on:message={(e) => (message = e.detail)}
/>

{#if error}
	<p>Error: {error}</p>
{/if}

{#if mode}
	<p>Mode: {mode}</p>
{/if}

{#if message}
	<p>Message: {JSON.stringify(message)}</p>
{/if}
