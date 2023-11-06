<script>
	import { onMount } from "svelte";
	import { marked } from 'marked';
	import { writable, get } from 'svelte/store';
	import anychart from 'anychart';

	const connections = writable({});

	const magicLink = {
		name: 'magicLink',
		level: 'inline',
		start(src) { return src.match(/@/)?.index; },
		tokenizer(src, tokens) {
			const rule = /^@\[(.+)\]/;
			const match = rule.exec(src);
			if (match) {
				// console.log('found', tokens);
				return {
					type: 'magicLink',
					raw: match[0],
					at: this.lexer.inlineTokens(match[1].trim()),
				};
			}
		},
		renderer(token) {
			const con = get(connections);
			con[selectedFile].push(token.at[0].text);
			connections.set(con);
			chartData.edges.push({
				from: selectedFile,
				to: token.at[0].text
			});

			const link = this.parser.parseInline(token.at).split(' ').join('-');
			return `\n<a href="#${link}"">${this.parser.parseInline(token.at)}</a>`;
		},
		childTokens: ['a'],
	};

	marked.use({ extensions: [magicLink] });

	let files = {};
	let fileNames = [];
	let selectedFile;
	let mds = {};
	let chart;
	let loading = true;
	const chartData = {
		nodes: [],
		edges: [],
	};

	onMount(async () => {
		loading = true;
		const response = await fetch('/files.json');
		fileNames = await response.json();
		const con = {};

		await Promise.all(
			fileNames.map(async f => {
			const res = await fetch(`/${f}.md`);
			files[f] = await res.text();
			con[f] = [];
		}));
		
		connections.set(con);
		
		Object.entries(files).forEach(([key, value]) => {
			selectedFile = key;
			chartData.nodes.push({ id: key });
			mds[key] = marked.parse(value);
		});
		
		selectedFile = fileNames[0];
		chart = anychart.graph(chartData);
		chart.nodes().labels().enabled(true);
		chart.nodes().labels().fontSize(12);
		chart.nodes().labels().fontWeight(600);
		chart.listen("click", (e) => {
			const id = e.domTarget?.tag?.id;
			if (id) {
				selectedFile = id;
			}
		})
		chart.container("anychart-container");
		chart.draw();
		loading = false;
	});

	function selectFile(name) {
		selectedFile = name;
	}

	connections.subscribe((value) => {
		if (chart) {
			console.log('got', value);
			chart = anychart.graph(chartData);
			chart.container("anychart-container");
			chart.draw();
		}
	})
</script>

<svelte:head>
	<title>Magic notes</title>
</svelte:head>

<div class="container">
	<div class="layout">
		<div class="files">
			<section>
				Files
				{#each fileNames as file}
					<p class="note" on:click={() => selectFile(file)}>{file}</p>
				{/each}
			</section>
		</div>
		<div class="editor">
			<div class="editor-open-files">
				{ selectedFile }
			</div>
			<section>
				Text
				{#if !loading}
					{ @html mds[selectedFile] }
				{/if}
			</section>
			<div id="anychart-container"></div>
		</div>
		<!--section>
			connections
			{#each Object.entries($connections) as [key, value]}
				<p>{ key } : { [...new Set(value)] }</p>
			{/each}
		</section-->
	</div>
</div>

<style>
	.container {
		height: 100%;
		display: flex;
    flex-direction: column;
    justify-content: space-between;
	}

	.layout {
		display: grid;
		height: 100%;
		grid-template-columns: 20rem 1fr;
	}

	.editor {
		display: grid;
		grid-template-rows: 2rem 1fr;
	}

	.editor-open-files {
		border-bottom: 1px solid gray;
		display: flex;
		padding-left: 1rem;
		align-items: center;
	}

	.files {
		border-right: 1px solid gray;
	}

	section {
		padding: 1rem;
	}

	.note {
		cursor: pointer;
		color: blue;
	}

	#anychart-container {
		width: 100%;
		height: 20rem;
		border-top: 1px solid gray;
	}
</style>
