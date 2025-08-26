<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	import { tick } from 'svelte';

	type Format = 'markdown' | 'html' | 'mrkdwn';

	let inputText = '# Hello World\n\nThis is **bold** text and *italic* text.\n\n- List item 1\n- List item 2\n\n[Link to Google](https://google.com)';
	let outputText = '';
	let inputFormat: Format = 'markdown';
	let outputFormat: Format = 'html';
	let isClient = false;
	let showPreview = false;
	let theme: 'light' | 'dark' = 'light';
	let copied = false; // UI state for copy confirmation

	// accessibility / status
	let statusMessage = '';
	
	let marked: any;
	let DOMPurify: any;
	let TurndownService: any;
	let turndownService: any;

	onMount(async () => {
		if (browser) {
			// Dynamic imports for client-side only
			const markedModule = await import('marked');
			marked = markedModule.marked;
			
			const dompurifyModule = await import('dompurify');
			DOMPurify = dompurifyModule.default;
			
			const turndownModule = await import('turndown');
			TurndownService = turndownModule.default;
			turndownService = new TurndownService();
			
			isClient = true;
			convert();
			// restore theme preference
			const stored = localStorage.getItem('theme');
			if (stored === 'dark') toggleTheme(false);
		}
	});

	function convert() {
		if (!isClient || !inputText.trim() || !marked || !DOMPurify || !turndownService) {
			outputText = '';
			return;
		}

		try {
			if (inputFormat === outputFormat) {
				outputText = inputText;
				return;
			}

			if (inputFormat === 'markdown' && outputFormat === 'html') {
				outputText = DOMPurify.sanitize(marked(inputText));
			} else if (inputFormat === 'html' && outputFormat === 'markdown') {
				outputText = turndownService.turndown(inputText);
			} else if (inputFormat === 'markdown' && outputFormat === 'mrkdwn') {
				outputText = markdownToMrkdwn(inputText);
			} else if (inputFormat === 'mrkdwn' && outputFormat === 'markdown') {
				outputText = mrkdwnToMarkdown(inputText);
			} else if (inputFormat === 'html' && outputFormat === 'mrkdwn') {
				const mdContent = turndownService.turndown(inputText);
				outputText = markdownToMrkdwn(mdContent);
			} else if (inputFormat === 'mrkdwn' && outputFormat === 'html') {
				const mdContent = mrkdwnToMarkdown(inputText);
				outputText = DOMPurify.sanitize(marked(mdContent));
			}
		} catch (error) {
			outputText = 'Error converting: ' + (error as Error).message;
		}
	}

	function markdownToMrkdwn(md: string): string {
		return md
			.replace(/^#{6}\s+(.+)$/gm, '$1')
			.replace(/^#{5}\s+(.+)$/gm, '$1')
			.replace(/^#{4}\s+(.+)$/gm, '$1')
			.replace(/^#{3}\s+(.+)$/gm, '$1')
			.replace(/^#{2}\s+(.+)$/gm, '$1')
			.replace(/^#{1}\s+(.+)$/gm, '*$1*')
			.replace(/\*\*(.*?)\*\*/g, '*$1*')
			.replace(/__(.*?)__/g, '*$1*')
			.replace(/\*(.*?)\*/g, '_$1_')
			.replace(/_(.*?)_/g, '_$1_')
			.replace(/~~(.*?)~~/g, '~$1~')
			.replace(/`([^`]+)`/g, '`$1`')
			.replace(/\[([^\]]+)\]\(([^)]+)\)/g, '<$2|$1>')
			.replace(/^[-*+]\s+(.+)$/gm, '• $1')
			.replace(/^\d+\.\s+(.+)$/gm, '1. $1');
	}

	function mrkdwnToMarkdown(mrkdwn: string): string {
		return mrkdwn
			.replace(/\*([^*]+)\*/g, '**$1**')
			.replace(/_([^_]+)_/g, '*$1*')
			.replace(/~([^~]+)~/g, '~~$1~~')
			.replace(/<([^|>]+)\|([^>]+)>/g, '[$2]($1)')
			.replace(/^•\s+(.+)$/gm, '- $1')
			.replace(/^1\.\s+(.+)$/gm, '1. $1');
	}

	function copyToClipboard() {
		navigator.clipboard.writeText(outputText).then(() => {
			announce('Output copied to clipboard');
			copied = true;
			setTimeout(() => (copied = false), 1800);
		});
	}

	function downloadFile() {
		const blob = new Blob([outputText], { type: 'text/plain' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = `converted.${outputFormat === 'mrkdwn' ? 'txt' : outputFormat}`;
		a.click();
		URL.revokeObjectURL(url);
		announce('Download started');
	}

	function toggleTheme(announceChange = true) {
		if (!browser) return;
		theme = theme === 'light' ? 'dark' : 'light';
		document.documentElement.dataset.theme = theme;
		localStorage.setItem('theme', theme);
		if (announceChange) announce(`Switched to ${theme} theme`);
	}

	function announce(message: string) {
		statusMessage = message;
		// clear after a bit
		setTimeout(() => {
			if (statusMessage === message) statusMessage = '';
		}, 4000);
	}

	// Reactive conversion - triggers whenever input changes
	$: if (isClient && marked && DOMPurify && turndownService && (inputText || inputFormat || outputFormat)) {
		convert();
	}
</script>

<svelte:head>
	<title>Converticle - Text Format Converter</title>
	<meta name="description" content="Convert between Markdown, HTML, and Slack mrkdwn formats" />
</svelte:head>

<main class="container" data-theme={theme}>
	<header class="app-header">
		<div class="branding">
			<h1 aria-label="Converticle - Text Format Converter">
				<span class="logo">🔁</span> Converticle
			</h1>
			<p class="tagline">Markdown ⇆ HTML ⇆ Slack mrkdwn</p>
		</div>
		<div class="header-actions">
			<button class="ghost" on:click={toggleTheme} aria-label="Toggle color theme" title="Toggle light / dark">{theme === 'light' ? '🌙' : '☀️'}</button>
			<a href="https://github.com/" target="_blank" rel="noopener" class="ghost" aria-label="GitHub repository" title="GitHub">
				<svg width="18" height="18" viewBox="0 0 24 24" aria-hidden="true" fill="currentColor"><path d="M12 .5C5.648.5.5 5.648.5 12c0 5.086 3.292 9.385 7.865 10.905.575.103.785-.25.785-.557 0-.275-.01-1.003-.015-1.97-3.199.695-3.875-1.542-3.875-1.542-.523-1.33-1.278-1.684-1.278-1.684-1.044-.714.079-.699.079-.699 1.155.082 1.764 1.187 1.764 1.187 1.027 1.76 2.695 1.252 3.352.957.103-.744.402-1.253.732-1.54-2.554-.291-5.238-1.277-5.238-5.683 0-1.255.45-2.282 1.186-3.087-.119-.29-.514-1.46.112-3.046 0 0 .966-.309 3.166 1.18a11.02 11.02 0 0 1 2.883-.388c.978.005 1.963.133 2.883.388 2.2-1.489 3.164-1.18 3.164-1.18.628 1.586.233 2.756.114 3.046.738.805 1.184 1.832 1.184 3.087 0 4.417-2.688 5.387-5.252 5.673.414.355.78 1.057.78 2.132 0 1.54-.014 2.78-.014 3.158 0 .309.208.666.79.556C20.213 21.382 23.5 17.084 23.5 12 23.5 5.648 18.352.5 12 .5Z"/></svg>
			</a>
		</div>
	</header>

	<div class="converter" role="region" aria-label="Format conversion panels">
		<div class="panel input-panel">
			<div class="panel-header">
				<label for="input-format">Input Format:</label>
				<div class="select-wrapper">
					<select id="input-format" bind:value={inputFormat}>
					<option value="markdown">Markdown</option>
					<option value="html">HTML</option>
					<option value="mrkdwn">Slack mrkdwn</option>
					</select>
				</div>
			</div>
			<textarea
				bind:value={inputText}
				on:input={convert}
				placeholder="Paste your {inputFormat} content here..."
				class="text-area code"
				aria-label="Input text"
			></textarea>
		</div>

		<div class="conversion-controls">
			<button
				class="swap-button"
				on:click={() => {
					[inputFormat, outputFormat] = [outputFormat, inputFormat];
					[inputText, outputText] = [outputText, inputText];
				}}
				title="Swap input and output formats"
				aria-label="Swap input and output formats"
			>
				⇄
			</button>
		</div>

		<div class="panel output-panel">
			<div class="panel-header space-between">
				<div class="select-group">
					<label for="output-format">Output Format:</label>
					<div class="select-wrapper">
						<select id="output-format" bind:value={outputFormat}>
							<option value="markdown">Markdown</option>
							<option value="html">HTML</option>
							<option value="mrkdwn">Slack mrkdwn</option>
						</select>
					</div>
				</div>
				<div class="tabs" role="tablist" aria-label="Output view mode">
					<button role="tab" aria-selected={showPreview} class:active={showPreview} on:click={() => (showPreview = true)}>Preview</button>
					<button role="tab" aria-selected={!showPreview} class:active={!showPreview} on:click={() => (showPreview = false)}>Raw</button>
				</div>
			</div>
			<div class="output-container">
				{#if showPreview && outputFormat === 'html'}
					<div class="preview" aria-label="Rendered HTML preview">{@html outputText}</div>
				{:else if showPreview && outputFormat === 'markdown'}
					<div class="preview markdown" aria-label="Rendered Markdown preview">{@html DOMPurify?.sanitize(marked?.parse(outputText) || '')}</div>
				{:else if showPreview && outputFormat === 'mrkdwn'}
					<div class="preview" aria-label="Rendered mrkdwn preview"><pre>{outputText}</pre></div>
				{:else}
					<textarea readonly bind:value={outputText} class="text-area output code" aria-label="Output text (raw)"></textarea>
				{/if}
				<div class="output-actions" role="toolbar" aria-label="Output actions">
					<button on:click={copyToClipboard} class="action-button" class:copied={copied} title="Copy to clipboard" aria-label="Copy output to clipboard">
						<span>{#if copied}✓{:else}📋{/if}</span>
					</button>
					<button on:click={downloadFile} class="action-button" title="Download file" aria-label="Download output">💾</button>
				</div>
			</div>
		</div>
	</div>
	<div class="sr-only" aria-live="polite">{statusMessage}</div>
</main>

<style>
	:global(body) { margin:0; padding:0; font-family: system-ui,-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif; min-height:100dvh; }
	:global(html) { height:100%; }

	.container {
		max-width: 1200px;
		margin: 0 auto;
		padding: 2rem;
		height: 100dvh; /* Fill viewport without causing overflow from padding */
		box-sizing: border-box; /* Include padding in height calculation */
		display: flex;
		flex-direction: column;
		overflow: hidden; /* Prevent page scroll; internal areas will scroll */
	}

	.app-header { display:flex; justify-content:space-between; align-items:center; margin-bottom:1.25rem; padding:0.75rem 1rem; backdrop-filter: blur(18px) saturate(160%); background: var(--surface); border:1px solid var(--border); border-radius: var(--radius-lg); box-shadow: var(--shadow-lg); }
	.branding h1 { font-size: clamp(1.6rem,2.5vw,2.5rem); margin:0; font-weight:600; letter-spacing:.5px; display:flex; align-items:center; gap:.5rem; }
	.logo { display:inline-flex; font-size:1.4em; filter: drop-shadow(0 2px 4px rgba(0,0,0,0.25)); }
	.tagline { margin: .25rem 0 0; font-size:.85rem; letter-spacing:.5px; font-weight:500; color: var(--text-secondary); }
	.header-actions { display:flex; gap:.5rem; }
	.header-actions .ghost { background: var(--surface-alt); border:1px solid var(--border); border-radius: var(--radius-sm); padding:.55rem .7rem; cursor:pointer; font-size:.9rem; display:inline-flex; align-items:center; justify-content:center; line-height:1; transition:.25s; color: var(--text-primary); }
	.header-actions .ghost:hover { background: var(--surface); box-shadow: var(--focus-ring); }

	.converter { flex:1; display:grid; grid-template-columns:1fr auto 1fr; gap:1rem; background: var(--surface); border:1px solid var(--border); border-radius: var(--radius-lg); padding:1.1rem 1.1rem 1.25rem; box-shadow: var(--shadow-lg); min-height:0; overflow:hidden; backdrop-filter: blur(22px) saturate(180%); position:relative; }
	.converter::before { content:""; position:absolute; inset:0; border-radius: inherit; padding:1px; background:linear-gradient(140deg,var(--gradient-start),transparent 60%,var(--gradient-end)); -webkit-mask:linear-gradient(#000 0 0) content-box,linear-gradient(#000 0 0); mask:linear-gradient(#000 0 0) content-box,linear-gradient(#000 0 0); -webkit-mask-composite: xor; mask-composite: exclude; pointer-events:none; opacity:.35; }

	.panel {
		display: flex;
		flex-direction: column;
		min-height: 0;
		overflow: hidden;
	}

	.panel-header { display:flex; align-items:center; gap:.55rem; margin-bottom:.65rem; font-weight:600; color: var(--text-secondary); line-height:1; }
	.panel-header.space-between { justify-content:space-between; }
	.select-group { display:flex; align-items:center; gap:.5rem; }

	select { padding:.55rem 2.2rem .55rem .75rem; background: var(--surface-alt); border:1px solid var(--border); border-radius: var(--radius-sm); font-size:.8rem; cursor:pointer; color: var(--text-primary); transition:border-color .25s, background .25s, box-shadow .25s; backdrop-filter: blur(12px); appearance:none; -webkit-appearance:none; position:relative; line-height:1.2; background-image:none; }
	select:hover { background: var(--surface); }
	select:focus { outline:none; }
	select:focus-visible { border-color: var(--accent); box-shadow:0 0 0 2px color-mix(in srgb, var(--accent) 55%, transparent); }
	select::-ms-expand { display:none; }
	.select-wrapper { position:relative; display:inline-flex; }
	.select-wrapper::after { content:""; position:absolute; pointer-events:none; top:50%; right:.65rem; width:7px; height:7px; margin-top:-4px; border-right:2px solid var(--accent); border-bottom:2px solid var(--accent); transform:rotate(45deg); }

	.text-area { flex:1; padding:.9rem 1rem 1.15rem; border:1px solid var(--border); border-radius: var(--radius-md); font-family: var(--code-font); font-size:.82rem; line-height:1.5; resize:none; transition: border-color .2s, background .3s; min-height:0; overflow:auto; background: var(--surface-alt); color: var(--text-primary); scrollbar-width: thin; scrollbar-color: var(--accent) transparent; }
	.text-area:focus { outline:none; border-color: var(--accent); box-shadow:0 0 0 2px color-mix(in srgb, var(--accent) 55%, transparent); }
	.text-area.code { font-feature-settings: "liga" 0; }

	.output { background: var(--surface-alt); }

	.output-container {
		flex: 1;
		position: relative;
		display: flex;
		flex-direction: column;
		min-height: 0;
		overflow: hidden;
	}

	.output-actions { position:absolute; top:.45rem; right:.45rem; display:flex; gap:.35rem; }

	.action-button { padding:.45rem .55rem; border:1px solid var(--border); background: var(--surface-alt); border-radius: var(--radius-sm); cursor:pointer; font-size:.85rem; transition:.25s; line-height:1; display:inline-flex; align-items:center; justify-content:center; min-width:2rem; }
	.action-button:hover { background: var(--surface); box-shadow: var(--focus-ring); }
	.action-button.copied { background: var(--success); color:#fff; border-color: var(--success); box-shadow:0 0 0 2px color-mix(in srgb,var(--success) 55%,transparent); }

	.conversion-controls { display:flex; align-items:center; justify-content:center; }
	.swap-button { padding:.85rem; border:1px solid var(--border); background: var(--surface-alt); color: var(--accent); border-radius:50%; font-size:1.15rem; cursor:pointer; transition:.4s; box-shadow: var(--shadow-lg); position:relative; overflow:hidden; }
	.swap-button::before { content:""; position:absolute; inset:0; background:radial-gradient(circle at 30% 30%,var(--gradient-start),var(--gradient-end)); opacity:0; transition:.45s; }
	.swap-button:hover::before { opacity:.22; }
	.swap-button:hover { transform:rotate(270deg) scale(1.06); }
	.swap-button:focus { outline:none; box-shadow:0 0 0 2px color-mix(in srgb,var(--accent) 55%,transparent); }

	.tabs { display:inline-flex; background: var(--surface-alt); border:1px solid var(--border); border-radius: var(--radius-sm); overflow:hidden; }
	.tabs button { appearance:none; border:0; background:transparent; padding:.45rem .85rem; font-size:.7rem; text-transform:uppercase; letter-spacing:.5px; font-weight:600; color: var(--text-secondary); cursor:pointer; transition:.25s; }
	.tabs button.active, .tabs button:hover { background: var(--surface); color: var(--text-primary); }
	.preview { flex:1; overflow:auto; border:1px solid var(--border); border-radius: var(--radius-md); padding:1.1rem 1.4rem 1.65rem; font-size:.9rem; line-height:1.55; background: var(--surface-alt); color: var(--text-primary); scrollbar-width: thin; scrollbar-color: var(--accent) transparent; }
	.preview pre { margin:0; font-family: var(--code-font); font-size:.8rem; }
	:global(.preview h1),:global(.preview h2),:global(.preview h3),:global(.preview h4) { margin-top:1.2em; line-height:1.15; }
	:global(.preview code),:global(.preview pre) { background:rgba(0,0,0,0.08); padding:.15rem .4rem; border-radius:4px; }
	:global(.preview a) { color: var(--accent); }
	:global(.preview p) { margin:.65em 0 .75em; }
	:global(.preview ul) { margin:.55em 0 .8em; padding-left:1.05em; }
	:global(.preview li + li) { margin-top:.25em; }

	/* Thin custom scrollbar for WebKit */
	.text-area::-webkit-scrollbar, .preview::-webkit-scrollbar { width:8px; }
	.text-area::-webkit-scrollbar-track, .preview::-webkit-scrollbar-track { background:transparent; }
	.text-area::-webkit-scrollbar-thumb, .preview::-webkit-scrollbar-thumb { background:linear-gradient(var(--gradient-start),var(--gradient-end)); border-radius:4px; }
	.text-area::-webkit-scrollbar-thumb:hover, .preview::-webkit-scrollbar-thumb:hover { filter:brightness(1.15); }
	.sr-only { position:absolute; width:1px; height:1px; padding:0; margin:-1px; overflow:hidden; clip:rect(0 0 0 0); white-space:nowrap; border:0; }

	@media (max-width: 900px) { .converter { grid-template-columns:1fr; } .conversion-controls { order:-1; } }
	@media (max-width: 600px) { .branding h1 { font-size:1.6rem; } .tagline { display:none; } }
</style>