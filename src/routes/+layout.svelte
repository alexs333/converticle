<script lang="ts">
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<meta name="color-scheme" content="light dark" />
	<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-9859754269579381" crossorigin="anonymous"></script>
</svelte:head>

{@render children?.()}

<style>
	:root {
		/* Light theme blue/cyan palette */
		--gradient-start: #3b82f6; /* blue-500 */
		--gradient-end: #0ea5e9;   /* cyan-500 */
		--accent: #2563eb;         /* blue-600 */
		--surface: rgba(255,255,255,0.78);
		--surface-alt: rgba(255,255,255,0.58);
		--border: rgba(37,99,235,0.22);
		--text-primary: #1f2937;
		--text-secondary: #4b5563;
		--radius-sm: 6px;
		--radius-md: 12px;
		--radius-lg: 18px;
		--focus-ring: 0 0 0 3px color-mix(in srgb, var(--accent) 55%, transparent), 0 1px 2px rgba(0,0,0,0.15);
		--shadow-lg: 0 8px 24px -4px rgba(31,41,55,0.15), 0 4px 12px -2px rgba(31,41,55,0.12);
		--code-font: 'Monaco','Menlo','Ubuntu Mono',monospace;
		--danger: #dc2626;
		--success: #059669;
		--warn: #d97706;
	}

	:root[data-theme="dark"] {
		/* Classic dark theme */
		--gradient-start: #1f2937; /* gray-800 */
		--gradient-end: #111827;   /* gray-900 */
		--accent: #6b7280;         /* gray-500 */
		--surface: rgba(31,41,55,0.85); /* gray-800 w/ alpha */
		--surface-alt: rgba(17,24,39,0.75); /* gray-900 */
		--border: rgba(107,114,128,0.3);
		--text-primary: #f9fafb;
		--text-secondary: #d1d5db;
		--shadow-lg: 0 8px 28px -4px rgba(0,0,0,0.7), 0 4px 14px -2px rgba(0,0,0,0.5);
	}

	:global(body) {
		background: radial-gradient(circle at 20% 20%, var(--gradient-start), var(--gradient-end));
		background-attachment: fixed;
		color: var(--text-primary);
	}
</style>
