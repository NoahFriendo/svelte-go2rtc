<script lang="ts">
	import VideoRTC from '$lib/VideoRTC.svelte';

	let stream_name = 'zed';

	let error = '';
	let mode = '';
	let message = '';

	let component: VideoRTC | null = null;
</script>

<h1>VideoRTC Component</h1>

<VideoRTC
	bind:this={component}
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

{#if component}
	<p>VideoRTC: {component}</p>
	<button
		on:click={() => {
			component?.seekToEnd();
		}}
	>
		Seek to end
	</button>
	<button
		on:click={() => {
			console.log(component?.getDelay());
		}}
	>
		Get Delay
	</button>
{/if}
