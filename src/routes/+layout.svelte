<script lang="ts">
	import './layout.css';
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();

	// =========================
	// Site metadata
	// =========================
	const siteUrl = 'https://netrasense.id';
	const siteName = 'NetraSense';

	const title =
		'NetraSense — Offline-first AI for blind and low-vision students';

	const description =
		'An offline-first, multi-sensory assistant for blind and low-vision university students: on-device vision, zero-latency classroom audio and 360° spatial navigation.';

	const ogImage = `${siteUrl}/og-image.png`;
</script>

<svelte:head>
	<!-- =========================
	     Basic
	     ========================= -->

	<link rel="icon" href={favicon} />

	<meta
		name="viewport"
		content="width=device-width, initial-scale=1, viewport-fit=cover"
	/>

	<meta name="theme-color" content="#065f46" />
	<meta name="color-scheme" content="light" />

	<!-- =========================
	     SEO
	     ========================= -->

	<title>{title}</title>

	<meta
		name="description"
		content={description}
	/>

	<meta
		name="author"
		content="NetraSense"
	/>

	<link
		rel="canonical"
		href={siteUrl}
	/>

	<!-- =========================
	     Open Graph
	     ========================= -->

	<meta property="og:type" content="website" />

	<meta
		property="og:site_name"
		content={siteName}
	/>

	<meta
		property="og:title"
		content={title}
	/>

	<meta
		property="og:description"
		content={description}
	/>

	<meta
		property="og:url"
		content={siteUrl}
	/>

	<meta
		property="og:image"
		content={ogImage}
	/>

	<meta
		property="og:image:width"
		content="1200"
	/>

	<meta
		property="og:image:height"
		content="630"
	/>

	<meta
		property="og:image:alt"
		content="NetraSense — Offline-first assistive AI"
	/>

	<meta
		property="og:locale"
		content="en_US"
	/>

	<meta
		property="og:locale:alternate"
		content="id_ID"
	/>

	<!-- =========================
	     Twitter / X
	     ========================= -->

	<meta
		name="twitter:card"
		content="summary_large_image"
	/>

	<meta
		name="twitter:title"
		content={title}
	/>

	<meta
		name="twitter:description"
		content={description}
	/>

	<meta
		name="twitter:image"
		content={ogImage}
	/>

	<meta
		name="twitter:image:alt"
		content="NetraSense — Offline-first assistive AI"
	/>

	<!-- =========================
	     Mobile
	     ========================= -->

	<meta
		name="apple-mobile-web-app-capable"
		content="yes"
	/>

	<meta
		name="apple-mobile-web-app-title"
		content={siteName}
	/>

	<meta
		name="format-detection"
		content="telephone=no"
	/>
</svelte:head>

{@render children()}