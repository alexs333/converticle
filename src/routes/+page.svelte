<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	import { tick } from 'svelte';

	// Format Documentation:
	// Markdown: https://spec.commonmark.org/ | https://github.github.com/gfm/
	// HTML: https://developer.mozilla.org/en-US/docs/Web/HTML | https://html.spec.whatwg.org/
	// Slack mrkdwn: https://api.slack.com/reference/surfaces/formatting
	// AsciiDoc: https://asciidoc.org/ | https://docs.asciidoctor.org/
	type Format = 'markdown' | 'html' | 'mrkdwn' | 'asciidoc';

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
	let Asciidoctor: any;

	onMount(async () => {
		if (browser) {
			// Dynamic imports for client-side only
			// HTML processing libraries documentation:
			// Marked (Markdown parser): https://marked.js.org/
			// DOMPurify (HTML sanitizer): https://github.com/cure53/DOMPurify
			// TurndownService (HTML to Markdown): https://github.com/mixmark-io/turndown
			// Asciidoctor (AsciiDoc processor): https://docs.asciidoctor.org/asciidoctor.js/
			const markedModule = await import('marked');
			marked = markedModule.marked;
			
			const dompurifyModule = await import('dompurify');
			DOMPurify = dompurifyModule.default;
			
			const turndownModule = await import('turndown');
			TurndownService = turndownModule.default;
			turndownService = new TurndownService();
			
			const asciidoctorModule = await import('@asciidoctor/core');
			Asciidoctor = asciidoctorModule.default();
			
			isClient = true;
			convert();
			// restore theme preference
			const stored = localStorage.getItem('theme');
			if (stored === 'dark') toggleTheme(false);
		}
	});

	function convert() {
		if (!isClient || !inputText.trim() || !marked || !DOMPurify || !turndownService || !Asciidoctor) {
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
			} else if (inputFormat === 'asciidoc' && outputFormat === 'html') {
				outputText = DOMPurify.sanitize(Asciidoctor.convert(inputText));
			} else if (inputFormat === 'html' && outputFormat === 'asciidoc') {
				const mdContent = turndownService.turndown(inputText);
				outputText = markdownToAsciidoc(mdContent);
			} else if (inputFormat === 'markdown' && outputFormat === 'asciidoc') {
				outputText = markdownToAsciidoc(inputText);
			} else if (inputFormat === 'asciidoc' && outputFormat === 'markdown') {
				const htmlContent = Asciidoctor.convert(inputText);
				outputText = turndownService.turndown(htmlContent);
			} else if (inputFormat === 'asciidoc' && outputFormat === 'mrkdwn') {
				const htmlContent = Asciidoctor.convert(inputText);
				const mdContent = turndownService.turndown(htmlContent);
				outputText = markdownToMrkdwn(mdContent);
			} else if (inputFormat === 'mrkdwn' && outputFormat === 'asciidoc') {
				const mdContent = mrkdwnToMarkdown(inputText);
				outputText = markdownToAsciidoc(mdContent);
			}
		} catch (error) {
			outputText = 'Error converting: ' + (error as Error).message;
		}
	}

	// Markdown to Slack mrkdwn conversion
	// Markdown spec: https://spec.commonmark.org/
	// Slack mrkdwn spec: https://api.slack.com/reference/surfaces/formatting
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

	// Slack mrkdwn to Markdown conversion
	// Slack mrkdwn spec: https://api.slack.com/reference/surfaces/formatting
	// Markdown spec: https://spec.commonmark.org/
	function mrkdwnToMarkdown(mrkdwn: string): string {
		return mrkdwn
			.replace(/\*([^*]+)\*/g, '**$1**')
			.replace(/_([^_]+)_/g, '*$1*')
			.replace(/~([^~]+)~/g, '~~$1~~')
			.replace(/<([^|>]+)\|([^>]+)>/g, '[$2]($1)')
			.replace(/^•\s+(.+)$/gm, '- $1')
			.replace(/^1\.\s+(.+)$/gm, '1. $1');
	}

	// Markdown to AsciiDoc conversion
	// Markdown spec: https://spec.commonmark.org/
	// AsciiDoc spec: https://asciidoc.org/
	function markdownToAsciidoc(md: string): string {
		return md
			.replace(/^#{6}\s+(.+)$/gm, '====== $1')
			.replace(/^#{5}\s+(.+)$/gm, '===== $1')
			.replace(/^#{4}\s+(.+)$/gm, '==== $1')
			.replace(/^#{3}\s+(.+)$/gm, '=== $1')
			.replace(/^#{2}\s+(.+)$/gm, '== $1')
			.replace(/^#{1}\s+(.+)$/gm, '= $1')
			.replace(/\*\*(.*?)\*\*/g, '*$1*')
			.replace(/__(.*?)__/g, '*$1*')
			.replace(/\*(.*?)\*/g, '_$1_')
			.replace(/_(.*?)_/g, '_$1_')
			.replace(/~~(.*?)~~/g, '[line-through]#$1#')
			.replace(/`([^`]+)`/g, '`$1`')
			.replace(/\[([^\]]+)\]\(([^)]+)\)/g, '$2[$1]')
			.replace(/^[-*+]\s+(.+)$/gm, '* $1')
			.replace(/^\d+\.\s+(.+)$/gm, '. $1')
			.replace(/^>\s+(.+)$/gm, '____\n$1\n____')
			.replace(/^```(\w*)\n([\s\S]*?)\n```$/gm, '[source,$1]\n----\n$2\n----');
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
		a.download = `converted.${outputFormat === 'mrkdwn' ? 'txt' : outputFormat === 'asciidoc' ? 'adoc' : outputFormat}`;
		a.click();
		URL.revokeObjectURL(url);
		announce('Download started');
	}

	function emailText() {
		const subject = `Converted ${inputFormat} to ${outputFormat}`;
		const body = encodeURIComponent(outputText);
		const mailtoLink = `mailto:?subject=${encodeURIComponent(subject)}&body=${body}`;
		window.location.href = mailtoLink;
		announce('Email client opened');
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
	$: if (isClient && marked && DOMPurify && turndownService && Asciidoctor && (inputText || inputFormat || outputFormat)) {
		convert();
	}
</script>

<svelte:head>
	<title>Converticle - Free Online Text Format Converter | Markdown ⇆ HTML ⇆ Slack</title>
	<meta name="description" content="Convert text between Markdown, HTML, AsciiDoc, and Slack mrkdwn formats instantly. Free online converter with live preview, copy/paste, and download features. Perfect for developers and content creators." />
	<meta name="keywords" content="markdown converter, html converter, asciidoc converter, slack mrkdwn, text format converter, markdown to html, html to markdown, asciidoc to html, online converter, free tool" />
	<meta name="author" content="Converticle" />
	<meta name="robots" content="index, follow" />
	<meta name="viewport" content="width=device-width, initial-scale=1.0" />
	<link rel="canonical" href="https://converticle.com" />
	
	<!-- Open Graph / Facebook -->
	<meta property="og:type" content="website" />
	<meta property="og:url" content="https://converticle.com" />
	<meta property="og:title" content="Converticle - Free Online Text Format Converter" />
	<meta property="og:description" content="Convert text between Markdown, HTML, AsciiDoc, and Slack mrkdwn formats instantly. Free online converter with live preview and download features." />
	<meta property="og:image" content="https://converticle.com/og-image.png" />
	<meta property="og:image:width" content="1200" />
	<meta property="og:image:height" content="630" />
	<meta property="og:site_name" content="Converticle" />
	
	<!-- Twitter -->
	<meta property="twitter:card" content="summary_large_image" />
	<meta property="twitter:url" content="https://converticle.com" />
	<meta property="twitter:title" content="Converticle - Free Online Text Format Converter" />
	<meta property="twitter:description" content="Convert text between Markdown, HTML, AsciiDoc, and Slack mrkdwn formats instantly. Free online converter with live preview." />
	<meta property="twitter:image" content="https://converticle.com/og-image.png" />
	
	<!-- Additional SEO -->
	<meta name="theme-color" content="#2563eb" />
	<meta name="application-name" content="Converticle" />
	<meta name="apple-mobile-web-app-title" content="Converticle" />
	<meta name="apple-mobile-web-app-capable" content="yes" />
	<meta name="apple-mobile-web-app-status-bar-style" content="default" />
	
	<!-- Structured Data -->
	<script type="application/ld+json">
		{
			"@context": "https://schema.org",
			"@type": "WebApplication",
			"name": "Converticle",
			"description": "Convert text between Markdown, HTML, AsciiDoc, and Slack mrkdwn formats instantly",
			"url": "https://converticle.com",
			"applicationCategory": "DeveloperApplication",
			"operatingSystem": "Any",
			"offers": {
				"@type": "Offer",
				"price": "0",
				"priceCurrency": "USD"
			},
			"featureList": [
				"Convert Markdown to HTML",
				"Convert HTML to Markdown", 
				"Convert AsciiDoc to HTML",
				"Convert to/from Slack mrkdwn format",
				"Live preview",
				"Copy to clipboard",
				"Download converted files",
				"Dark/Light theme support"
			]
		}
	</script>
</svelte:head>

<main class="container" data-theme={theme}>
	<header class="app-header">
		<div class="branding">
			<h1 aria-label="Converticle - Text Format Converter">
				<span class="logo">🔁</span> Converticle
			</h1>
			<p class="tagline">Convert between Markdown, HTML, AsciiDoc, and Slack mrkdwn formats</p>
		</div>
		<div class="header-actions">
			<button class="ghost" on:click={() => toggleTheme()} aria-label="Toggle color theme" title="Toggle light / dark">{theme === 'light' ? '🌙' : '☀️'}</button>
			<a href="https://github.com/alexs333/converticle/issues/" target="_blank" rel="noopener" class="ghost" aria-label="GitHub repository" title="GitHub">
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
					<option value="asciidoc">AsciiDoc</option>
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
							<option value="asciidoc">AsciiDoc</option>
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
				{:else if showPreview && outputFormat === 'asciidoc'}
					<div class="preview" aria-label="Rendered AsciiDoc preview">{@html DOMPurify?.sanitize(Asciidoctor?.convert(outputText) || '')}</div>
				{:else}
					<textarea readonly bind:value={outputText} class="text-area output code" aria-label="Output text (raw)"></textarea>
				{/if}
				<div class="output-actions" role="toolbar" aria-label="Output actions">
					<button on:click={copyToClipboard} class="action-button" class:copied={copied} title="Copy to clipboard" aria-label="Copy output to clipboard">
						<span>{#if copied}✓{:else}📋{/if}</span>
					</button>
					<button on:click={emailText} class="action-button" title="Email text" aria-label="Email output">📧</button>
					<button on:click={downloadFile} class="action-button" title="Download file" aria-label="Download output">💾</button>
				</div>
			</div>
		</div>
	</div>
	
	<footer class="app-footer">
		<section class="feature-description">
			<h2>About Converticle</h2>
			<p>A free converter for Markdown, HTML, AsciiDoc, and mrkdwn (Slack) formats with live preview.</p>
			
			<div class="format-links">
				<span class="docs-label">Format Docs:</span>
				<div class="docs-links">
					<span><strong>Markdown</strong> <a href="https://www.markdownguide.org/" target="_blank" rel="noopener">Guide</a> <a href="https://spec.commonmark.org/" target="_blank" rel="noopener">Spec</a></span> <span class="separator">|</span> 
					<span><strong>HTML</strong> <a href="https://developer.mozilla.org/en-US/docs/Web/HTML" target="_blank" rel="noopener">MDN</a> <a href="https://html.spec.whatwg.org/" target="_blank" rel="noopener">Standard</a></span> <span class="separator">|</span> 
					<span><strong>AsciiDoc</strong> <a href="https://asciidoc.org/" target="_blank" rel="noopener">Syntax</a> <a href="https://docs.asciidoctor.org/" target="_blank" rel="noopener">Docs</a></span> <span class="separator">|</span> 
					<span><strong>Slack mrkdwn</strong> <a href="https://api.slack.com/reference/surfaces/formatting" target="_blank" rel="noopener">Formatting</a> <a href="https://slack.com/help/articles/202288908-Format-your-messages" target="_blank" rel="noopener">Help</a></span>
				</div>
			</div>
		</section>
	</footer>
	
	<div class="sr-only" aria-live="polite">{statusMessage}</div>
</main>

<style>
	:global(body) { margin:0; padding:0; font-family: system-ui,-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif; min-height:100dvh; }
	:global(html) { height:100%; }

	.container {
		max-width: 1600px;
		margin: 0 auto;
		padding: 1.5rem;
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
	
	.app-footer { margin-top: 1rem; padding: 0.75rem 0; }
	.feature-description { max-width: 600px; margin: 0 auto; text-align: center; }
	.feature-description h2 { font-size: 1.2rem; margin-bottom: 0.5rem; color: var(--text-primary); font-weight: 600; }
	.feature-description p { color: var(--text-secondary); line-height: 1.4; font-size: 0.85rem; }
	
	.format-links { margin-top: 1rem; font-size: 0.72rem; color: var(--text-secondary); line-height: 1.6; }
	.docs-label { font-weight: 600; margin-right: 0.5rem; display: block; margin-bottom: 0.25rem; }
	.docs-links { display: block; }
	.docs-links span { white-space: nowrap; }
	.docs-links strong { color: var(--text-primary); font-weight: 600; margin-right: 0.35rem; }
	.docs-links a { color: var(--text-primary); text-decoration: underline; margin-right: 0.6rem; }
	.docs-links a:hover { opacity: 0.8; }
	.docs-links .separator { margin: 0 0.3rem; color: var(--text-secondary); }
	
	@media (max-width: 700px) { 
		.format-links { font-size: 0.7rem; }
		.docs-links { word-break: break-word; }
		.docs-links span { white-space: normal; }
	}

	@media (max-width: 900px) { .converter { grid-template-columns:1fr; } .conversion-controls { order:-1; } }
	@media (max-width: 600px) { .branding h1 { font-size:1.6rem; } .tagline { display:none; } }
</style>