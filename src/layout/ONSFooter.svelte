<script>
	import { themes } from '../config.js';
	import { getContext } from 'svelte';

	export let theme = getContext('theme');
</script>

<footer style="color: {themes[theme]['text']}; background-color: {themes[theme]['pale']};">
	<div class="col-wide" data-analytics="footer">
		<a href="https://uclouvain.be/fr/index.html">
			{#if theme == 'dark'}
			<img
				class="logo-img"
				src="./img/ons-logo-neg-en.svg"
				alt="Office for National Statistics" />
			{:else}
			<img
				class="logo-img"
				src="./img/ons-logo-black-en.svg"
				alt="Office for National Statistics" />
			{/if}
		</a>
		<ul>
			<li><a href="https://uclouvain.be/fr/facultes/espo/comu.html" class="link" style="color: {themes[theme]['text']}">COMU</a></li>
			<li><a href="https://uclouvain.be/fr/facultes/espo/comu/master-en-journalisme.html" class="link" style="color: {themes[theme]['text']}">Ecole de journalisme de l'UCLouvain</a></li>
		</ul>
		<hr style="border-top-color: {themes[theme]['muted']}" />
		<div class="license">
			Ce travail est un mémoire-projet mené dans le cadre du Master120 de l'école de jounralisme de l'UCLouvain
		</div>
	</div>
</footer>

<style>
	footer {
		padding-bottom: 72px;
		padding-top: 36px;
		font-size: 18px;
	}
	footer a.link {
		-webkit-box-sizing: border-box;
		box-sizing: border-box;
		text-decoration: underline;
		display: inline-block;
	}
	footer a.link:hover {
		text-decoration: none;
	}
	footer a.link:active {
		outline: 3px solid transparent;
		background-color: #fd0;
		-webkit-box-shadow: 0 -2px #fd0, 0 4px #222;
		box-shadow: 0 -2px #fd0, 0 4px #222;
		color: #222;
		text-decoration: none;
	}
	footer hr {
		margin-bottom: 27px;
		margin-top: 36px;
		border: none;
		border-top: 1px solid #777;
	}
	ul {
		padding: 0;
		margin: 0;
	}
	li {
		margin-right: 18px;
		display: inline-block;
	}
	.logo-img {
		margin-bottom: 27px;
		width: 270px;
	}
	.ogl-img {
		margin: 0 0.5rem 0.2rem 0;
		width: 59px;
		height: 24px;
		vertical-align: middle;
	}
	.license {
		vertical-align: top;
	}
	a img:hover {
		cursor: pointer;
	}
</style>