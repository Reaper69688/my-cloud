<script lang="ts">
  import { onMount } from 'svelte';
  let { entry, onready }: { entry: any; onready: (urls?: any) => void } = $props();
  onMount(() => { onready(); });
</script>
<slot/>
