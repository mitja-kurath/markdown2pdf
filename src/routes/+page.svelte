<script lang="ts">
	import { marked } from 'marked';
	import { onMount } from 'svelte';

	type ViewMode = 'edit' | 'split' | 'preview';

	const SAMPLE = `# My Document

This is a **markdown to PDF** converter that runs entirely in your browser — no uploads, no servers.

## Features

- Live preview as you type
- Drag & drop \`.md\` files to open them
- Clean, print-optimized PDF export

## Code

\`\`\`javascript
function greet(name) {
  return \`Hello, \${name}!\`;
}
\`\`\`

Inline \`code\` works too.

## Tables

| Name       | Type    | Required |
|------------|---------|----------|
| title      | string  | yes      |
| body       | string  | yes      |
| published  | boolean | no       |

## Blockquotes

> "Simplicity is the soul of efficiency."
> — Austin Freeman

## Lists

1. Write your document in Markdown
2. Preview it on the right
3. Click **Export PDF** when ready

---

Happy writing!
`;

	let markdown = $state(SAMPLE);
	let viewMode = $state<ViewMode>('split');
	let isDragging = $state(false);
	let isExporting = $state(false);
	let filename = $state('document');
	let textareaEl: HTMLTextAreaElement | null = $state(null);

	function renderMarkdown(md: string): string {
		return marked.parse(md, { async: false, gfm: true, breaks: false }) as string;
	}

	let rendered = $derived(renderMarkdown(markdown));
	let wordCount = $derived(markdown.trim() ? markdown.trim().split(/\s+/).length : 0);
	let charCount = $derived(markdown.length);

	onMount(() => {
		textareaEl?.focus();
	});

	function handleDragOver(e: DragEvent) {
		e.preventDefault();
		isDragging = true;
	}

	function handleDragLeave(e: DragEvent) {
		const target = e.currentTarget as Element;
		if (!target.contains(e.relatedTarget as Node)) {
			isDragging = false;
		}
	}

	async function handleDrop(e: DragEvent) {
		e.preventDefault();
		isDragging = false;
		const file = e.dataTransfer?.files[0];
		if (file) {
			const text = await file.text();
			markdown = text;
			filename = file.name.replace(/\.(md|txt|markdown)$/i, '') || 'document';
		}
	}

	function handleFileInput(e: Event) {
		const file = (e.target as HTMLInputElement).files?.[0];
		if (!file) return;
		file.text().then((text) => {
			markdown = text;
			filename = file.name.replace(/\.(md|txt|markdown)$/i, '') || 'document';
			(e.target as HTMLInputElement).value = '';
		});
	}

	function handleKeydown(e: KeyboardEvent) {
		if (e.key === 'Tab') {
			e.preventDefault();
			const ta = e.target as HTMLTextAreaElement;
			const start = ta.selectionStart;
			const end = ta.selectionEnd;
			markdown = markdown.substring(0, start) + '  ' + markdown.substring(end);
			requestAnimationFrame(() => {
				ta.selectionStart = ta.selectionEnd = start + 2;
			});
		}
	}

	function exportPDF() {
		isExporting = true;

		const printCSS = `
      @page { size: A4; margin: 2.2cm 2.5cm; }

      *, *::before, *::after { box-sizing: border-box; }

      body {
        font-family: Georgia, 'Times New Roman', serif;
        font-size: 11pt;
        line-height: 1.75;
        color: #1a1a1a;
        margin: 0;
        padding: 0;
        -webkit-print-color-adjust: exact;
        print-color-adjust: exact;
      }

      h1, h2, h3, h4, h5, h6 {
        font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif;
        font-weight: 600;
        line-height: 1.3;
        margin: 1.6em 0 0.5em;
        color: #111;
        page-break-after: avoid;
      }

      h1 { font-size: 22pt; margin-top: 0; padding-bottom: 0.3em; border-bottom: 2px solid #e5e7eb; }
      h2 { font-size: 16pt; padding-bottom: 0.2em; border-bottom: 1px solid #e5e7eb; }
      h3 { font-size: 13pt; }
      h4 { font-size: 11.5pt; }
      h5, h6 { font-size: 11pt; color: #374151; }

      p { margin: 0 0 0.9em; orphans: 3; widows: 3; }

      a { color: #2563eb; text-decoration: underline; word-break: break-all; }

      strong { font-weight: 700; }
      em { font-style: italic; }
      del { text-decoration: line-through; color: #6b7280; }

      code {
        font-family: 'Courier New', 'Lucida Console', monospace;
        font-size: 9.5pt;
        background: #f3f4f6;
        border: 1px solid #e5e7eb;
        border-radius: 3px;
        padding: 0.1em 0.35em;
      }

      pre {
        background: #f8f9fa;
        border: 1px solid #e5e7eb;
        border-radius: 5px;
        padding: 0.9em 1.1em;
        margin: 0.8em 0 1em;
        overflow-x: auto;
        page-break-inside: avoid;
      }

      pre code {
        background: none;
        border: none;
        padding: 0;
        font-size: 9pt;
        line-height: 1.55;
      }

      blockquote {
        border-left: 3px solid #d1d5db;
        margin: 1em 0;
        padding: 0.4em 1em;
        color: #4b5563;
        background: #f9fafb;
      }

      blockquote p:last-child { margin-bottom: 0; }

      table {
        width: 100%;
        border-collapse: collapse;
        margin: 0.8em 0 1em;
        font-size: 10.5pt;
        page-break-inside: avoid;
      }

      th {
        background: #f3f4f6;
        font-weight: 600;
        font-family: -apple-system, sans-serif;
        text-align: left;
        padding: 0.45em 0.75em;
        border: 1px solid #d1d5db;
        font-size: 10pt;
      }

      td {
        padding: 0.35em 0.75em;
        border: 1px solid #e5e7eb;
        vertical-align: top;
      }

      tr:nth-child(even) td { background: #f9fafb; }

      ul, ol {
        margin: 0 0 0.9em;
        padding-left: 1.6em;
      }

      li { margin-bottom: 0.2em; }
      li > ul, li > ol { margin: 0.25em 0 0.25em; }

      hr {
        border: none;
        border-top: 1px solid #e5e7eb;
        margin: 1.5em 0;
      }

      img {
        max-width: 100%;
        height: auto;
        page-break-inside: avoid;
      }
    `;

		const iframe = document.createElement('iframe');
		iframe.style.cssText =
			'position:fixed;top:-9999px;left:-9999px;width:210mm;height:297mm;border:none;visibility:hidden;';
		document.body.appendChild(iframe);

		const doc = iframe.contentDocument!;
		doc.open();
		doc.write(`<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>${filename}</title>
<style>${printCSS}</style>
</head>
<body>${rendered}</body>
</html>`);
		doc.close();

		const cleanup = () => {
			if (document.body.contains(iframe)) document.body.removeChild(iframe);
			isExporting = false;
		};

		iframe.contentWindow!.onafterprint = cleanup;

		setTimeout(() => {
			iframe.contentWindow!.print();
			setTimeout(cleanup, 3000);
		}, 250);
	}
</script>

<!-- svelte-ignore a11y_no_static_element_interactions -->
<div
	class="flex h-screen flex-col bg-white"
	ondragover={handleDragOver}
	ondragleave={handleDragLeave}
	ondrop={handleDrop}
>
	<!-- Drag overlay -->
	{#if isDragging}
		<div
			class="fixed inset-0 z-50 flex items-center justify-center border-4 border-dashed border-blue-400 bg-blue-50/95"
		>
			<div class="text-center">
				<div class="mb-3 text-5xl opacity-60">📄</div>
				<p class="text-xl font-medium text-blue-700">Drop to open</p>
				<p class="mt-1 text-sm text-blue-500">.md · .txt · .markdown</p>
			</div>
		</div>
	{/if}

	<!-- Header -->
	<header class="flex shrink-0 items-center justify-between border-b border-gray-200 px-5 py-3">
		<!-- Brand -->
		<div class="flex items-center gap-2">
			<span class="text-[13px] font-semibold tracking-tight text-gray-900">markdown</span>
			<span class="text-[13px] text-gray-400">→</span>
			<span class="text-[13px] font-semibold tracking-tight text-gray-900">pdf</span>
		</div>

		<!-- View tabs -->
		<div class="flex rounded-lg bg-gray-100 p-0.5">
			{#each [['edit', 'Edit'], ['split', 'Split'], ['preview', 'Preview']] as [mode, label]}
				<button
					class="rounded-md px-3.5 py-1.5 text-xs font-medium transition-all duration-150 {viewMode ===
					mode
						? 'bg-white text-gray-900 shadow-sm'
						: 'text-gray-500 hover:text-gray-800'}"
					onclick={() => (viewMode = mode as ViewMode)}
				>
					{label}
				</button>
			{/each}
		</div>

		<!-- Actions -->
		<div class="flex items-center gap-1">
			<label
				class="cursor-pointer rounded-md px-3 py-1.5 text-xs font-medium text-gray-500 transition-colors hover:bg-gray-100 hover:text-gray-800"
			>
				<input
					type="file"
					accept=".md,.txt,.markdown"
					class="hidden"
					onchange={handleFileInput}
				/>
				Open file
			</label>

			<button
				onclick={exportPDF}
				disabled={isExporting || !markdown.trim()}
				class="flex items-center gap-1.5 rounded-md bg-gray-900 px-3.5 py-1.5 text-xs font-medium text-white transition-colors hover:bg-gray-700 disabled:cursor-not-allowed disabled:opacity-40"
			>
				{#if isExporting}
					<svg class="h-3 w-3 animate-spin" viewBox="0 0 24 24" fill="none">
						<circle
							class="opacity-25"
							cx="12"
							cy="12"
							r="10"
							stroke="currentColor"
							stroke-width="4"
						></circle>
						<path
							class="opacity-75"
							fill="currentColor"
							d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4z"
						></path>
					</svg>
					Preparing…
				{:else}
					<svg class="h-3 w-3" viewBox="0 0 16 16" fill="currentColor">
						<path
							d="M2.5 1A1.5 1.5 0 001 2.5v11A1.5 1.5 0 002.5 15h11a1.5 1.5 0 001.5-1.5V6.5L10 1H2.5zM10 2l4 4h-3.5A.5.5 0 0110 5.5V2zM4.5 8h7a.5.5 0 010 1h-7a.5.5 0 010-1zm0 2.5h7a.5.5 0 010 1h-7a.5.5 0 010-1zm0 2.5h4a.5.5 0 010 1h-4a.5.5 0 010-1z"
						/>
					</svg>
					Export PDF
				{/if}
			</button>
		</div>
	</header>

	<!-- Main panels -->
	<main class="flex min-h-0 flex-1">
		<!-- Editor -->
		{#if viewMode === 'edit' || viewMode === 'split'}
			<div
				class="flex flex-col {viewMode === 'split'
					? 'w-1/2 border-r border-gray-200'
					: 'w-full'}"
			>
				<textarea
					bind:this={textareaEl}
					bind:value={markdown}
					onkeydown={handleKeydown}
					spellcheck="false"
					placeholder="Start writing Markdown…"
					class="editor-textarea flex-1 resize-none bg-gray-50/60 px-7 py-6 font-mono text-[13px] leading-relaxed text-gray-800 outline-none placeholder:text-gray-300"
				></textarea>
			</div>
		{/if}

		<!-- Preview -->
		{#if viewMode === 'preview' || viewMode === 'split'}
			<div class="flex-1 overflow-y-auto px-10 py-8">
				<div class="prose mx-auto max-w-2xl">
					{@html rendered}
				</div>
			</div>
		{/if}
	</main>

	<!-- Status bar -->
	<footer
		class="flex shrink-0 items-center justify-between border-t border-gray-100 bg-gray-50 px-5 py-2"
	>
		<span class="text-[11px] text-gray-400">Drop a .md file anywhere to open it</span>
		<div class="flex gap-4 text-[11px] text-gray-400">
			<span>{wordCount} {wordCount === 1 ? 'word' : 'words'}</span>
			<span>{charCount} chars</span>
		</div>
	</footer>
</div>

<style>
	.editor-textarea::selection {
		background: #dbeafe;
	}

	/* Markdown preview prose styles */
	.prose :global(h1) {
		font-size: 1.75rem;
		font-weight: 700;
		line-height: 1.25;
		margin: 0 0 0.5em;
		color: #111827;
		border-bottom: 2px solid #f3f4f6;
		padding-bottom: 0.35em;
	}

	.prose :global(h2) {
		font-size: 1.35rem;
		font-weight: 600;
		line-height: 1.3;
		margin: 1.75em 0 0.5em;
		color: #1f2937;
		border-bottom: 1px solid #f3f4f6;
		padding-bottom: 0.25em;
	}

	.prose :global(h3) {
		font-size: 1.1rem;
		font-weight: 600;
		line-height: 1.35;
		margin: 1.5em 0 0.4em;
		color: #1f2937;
	}

	.prose :global(h4),
	.prose :global(h5),
	.prose :global(h6) {
		font-size: 0.95rem;
		font-weight: 600;
		margin: 1.25em 0 0.35em;
		color: #374151;
	}

	.prose :global(p) {
		margin: 0 0 0.9em;
		color: #374151;
		line-height: 1.75;
	}

	.prose :global(a) {
		color: #2563eb;
		text-decoration: underline;
		text-underline-offset: 2px;
	}

	.prose :global(a:hover) {
		color: #1d4ed8;
	}

	.prose :global(strong) {
		font-weight: 700;
		color: #111827;
	}

	.prose :global(em) {
		font-style: italic;
	}

	.prose :global(del) {
		text-decoration: line-through;
		color: #9ca3af;
	}

	.prose :global(code) {
		font-family: 'Menlo', 'Monaco', 'Consolas', monospace;
		font-size: 0.84em;
		background: #f3f4f6;
		border: 1px solid #e5e7eb;
		border-radius: 4px;
		padding: 0.1em 0.4em;
		color: #1f2937;
	}

	.prose :global(pre) {
		background: #f8f9fa;
		border: 1px solid #e5e7eb;
		border-radius: 8px;
		padding: 1em 1.25em;
		margin: 0.75em 0 1em;
		overflow-x: auto;
	}

	.prose :global(pre code) {
		background: none;
		border: none;
		padding: 0;
		font-size: 0.85em;
		line-height: 1.6;
		color: #1f2937;
	}

	.prose :global(blockquote) {
		border-left: 3px solid #d1d5db;
		margin: 1em 0;
		padding: 0.5em 1em;
		color: #6b7280;
		background: #f9fafb;
		border-radius: 0 6px 6px 0;
	}

	.prose :global(blockquote p) {
		margin: 0;
		color: #6b7280;
	}

	.prose :global(ul),
	.prose :global(ol) {
		margin: 0 0 0.9em;
		padding-left: 1.5em;
		color: #374151;
	}

	.prose :global(li) {
		margin-bottom: 0.25em;
		line-height: 1.7;
	}

	.prose :global(li > ul),
	.prose :global(li > ol) {
		margin: 0.25em 0;
	}

	.prose :global(table) {
		width: 100%;
		border-collapse: collapse;
		margin: 0.75em 0 1em;
		font-size: 0.9rem;
	}

	.prose :global(th) {
		background: #f3f4f6;
		font-weight: 600;
		text-align: left;
		padding: 0.5em 0.75em;
		border: 1px solid #e5e7eb;
		color: #1f2937;
		font-size: 0.85rem;
	}

	.prose :global(td) {
		padding: 0.4em 0.75em;
		border: 1px solid #e5e7eb;
		color: #374151;
		vertical-align: top;
	}

	.prose :global(tr:nth-child(even) td) {
		background: #f9fafb;
	}

	.prose :global(hr) {
		border: none;
		border-top: 1px solid #e5e7eb;
		margin: 1.5em 0;
	}

	.prose :global(img) {
		max-width: 100%;
		height: auto;
		border-radius: 6px;
	}
</style>
