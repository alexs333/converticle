# 🔄 Converticle

A modern, web-based tool to convert between Markdown, HTML, and Slack mrkdwn formats. Built with SvelteKit.

## Features

- ✨ **Modern UI**: Clean, responsive interface with gradient background
- 🔄 **Bi-directional conversion**: Convert between any supported format
- 📋 **Copy to clipboard**: One-click copying of converted text  
- 💾 **Download files**: Save converted content as files
- 🔀 **Format swapping**: Quick swap between input and output formats
- 📱 **Mobile friendly**: Responsive design works on all devices

## Supported Formats

- **Markdown** (.md) - Standard markdown format
- **HTML** - Clean HTML output with sanitization  
- **mrkdwn** - Slack's lightweight markup format

## Format Documentation

For detailed syntax and formatting rules, refer to the official documentation:

- **Markdown**: [CommonMark Spec](https://spec.commonmark.org/) | [GitHub Flavored Markdown](https://github.github.com/gfm/) | [Markdown Guide](https://www.markdownguide.org/)
- **HTML**: [MDN HTML Documentation](https://developer.mozilla.org/en-US/docs/Web/HTML) | [HTML Living Standard](https://html.spec.whatwg.org/)
- **Slack mrkdwn**: [Slack Formatting Documentation](https://api.slack.com/reference/surfaces/formatting) | [Slack Text Formatting Guide](https://slack.com/help/articles/202288908-Format-your-messages)

## Quick Start

```sh
# Install dependencies
npm install

# Start development server
npm run dev -- --open
```

The app will be available at `http://localhost:5173`

## Usage

1. **Select input format** from the left dropdown
2. **Paste or type** your content in the left textarea  
3. **Select output format** from the right dropdown
4. **View converted result** in the right textarea
5. **Copy** the result to clipboard or **download** as a file
6. Use the **swap button** (⇄) to quickly reverse input/output formats

## Examples

### Markdown to HTML
```markdown
# Hello World
**Bold text** and *italic text*
```
→
```html
<h1>Hello World</h1>
<p><strong>Bold text</strong> and <em>italic text</em></p>
```

### Markdown to Slack mrkdwn  
```markdown
**Bold** and *italic*
[Link](https://example.com)
```
→
```
*Bold* and _italic_
<https://example.com|Link>
```

## Building

To create a production version of your app:

```sh
npm run build
```

You can preview the production build with `npm run preview`.

> To deploy your app, you may need to install an [adapter](https://svelte.dev/docs/kit/adapters) for your target environment.
