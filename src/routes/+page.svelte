<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import * as THREE from 'three';
	import gsap from 'gsap';
	import { ScrollTrigger } from 'gsap/ScrollTrigger';
	import { ScrollToPlugin } from 'gsap/ScrollToPlugin';
	import Lenis from 'lenis';
	import 'lenis/dist/lenis.css';

	let mobileNavOpen = $state(false);
	let activeSection = $state('');

	const navLinks = [
		{ label: 'Capabilities', href: '#capabilities' },
		{ label: 'Architecture', href: '#architecture' },
		{ label: 'Impact', href: '#impact' }
	];

	type NavItem = { label: string; href: string; desc: string; external?: boolean };
	type NavDropdown = { label: string; items: NavItem[] };

	const navDropdowns: NavDropdown[] = [
		{
			label: 'Product',
			items: [
				{ label: 'Download beta', href: '#download', desc: 'Get the signed APK' },
				{ label: 'Changelog', href: '#', desc: 'Release history' },
				{ label: 'Roadmap', href: '#', desc: 'What is coming next' }
			]
		},
		{
			label: 'Developers',
			items: [
				{ label: 'API Docs', href: '#', desc: 'REST & gRPC reference' },
				{ label: 'Android SDK', href: '#', desc: 'Kotlin & Java libraries' },
				{ label: 'Model card', href: '#', desc: 'YOLO nano INT8 details' },
				{
					label: 'Contributing',
					href: 'https://github.com',
					desc: 'Guide & code of conduct',
					external: true
				}
			]
		},
		{
			label: 'Community',
			items: [
				{ label: 'GitHub', href: 'https://github.com', desc: 'Source code', external: true },
				{ label: 'Telegram', href: 'https://t.me', desc: 'Chat with the team', external: true },
				{ label: 'X / Twitter', href: 'https://x.com', desc: 'Release updates', external: true },
				{ label: 'LinkedIn', href: 'https://linkedin.com', desc: 'Company page', external: true }
			]
		}
	];

	const ticker = [
		'YOLO nano',
		'INT8 quantized',
		'NNAPI delegate',
		'Oboe SDK',
		'Visual place recognition',
		'Android 10+',
		'Airplane mode',
		'48 kHz DSP'
	];

	const capabilities = [
		{
			id: '01',
			title: 'On-device vision',
			body: 'A quantized YOLO model runs on the handset and names what stands in front of the user — rupiah notes and coins, bags, doorways, furniture, people in the path. Frames are processed locally and discarded. Nothing is uploaded.',
			tags: ['YOLO nano', 'INT8 quantized', '~38 ms / frame']
		},
		{
			id: '02',
			title: 'The third ear',
			body: 'A C++ DSP pipeline built on the Oboe SDK cleans classroom noise in real time, flags aggressive speech aimed at the user, and produces a short spoken summary of a lecturer’s explanation.',
			tags: ['C++ / Oboe SDK', '48 kHz stream', 'Voice activity detection']
		},
		{
			id: '03',
			title: 'Spatial navigation',
			body: 'Visual place recognition matches the camera feed against a map of campus that ships inside the app, follows guiding blocks, and returns turn-by-turn audio cues. A single gesture sends an SOS to a guardian or campus security.',
			tags: ['Visual place recognition', 'Bundled offline map', 'One-tap SOS']
		}
	];

	const steps = [
		{
			step: '01',
			title: 'Capture',
			body: 'Camera frames at 30 fps and a dual-microphone stream feed a single local pipeline.'
		},
		{
			step: '02',
			title: 'Inference',
			body: 'Quantized models execute on the NPU through the Android NNAPI delegate. No cloud round-trip.'
		},
		{
			step: '03',
			title: 'Fusion',
			body: 'Vision, audio and location results are scored together, then ranked by urgency before anything is spoken.'
		},
		{
			step: '04',
			title: 'Output',
			body: 'Speech, spatialised tones and haptics are delivered inside the user’s reaction window.'
		}
	];

	const specs: [string, string][] = [
		['Platform', 'Android 10 and above · AArch64'],
		['Runtime', '100% on-device · works in airplane mode'],
		['Vision model', 'YOLO nano, INT8 quantized, NNAPI delegate'],
		['Audio stack', 'C++ / Oboe, 48 kHz, sub-5 ms buffer'],
		['Navigation', 'Visual place recognition + guiding block detection'],
		['Bundle size', '42 MB (models included in the APK)'],
		['Licence', 'CC BY-NC 4.0 — free for non-commercial use'],
		['Maintainer', 'PT. Sentinel Project Indonesia']
	];

	const principles = [
		{
			title: 'Offline is a right, not a feature',
			body: 'Every model ships inside the APK. NetraSense works in a basement lecture hall, on a campus shuttle, in a dead zone — and nothing about the user’s surroundings ever leaves the device.'
		},
		{
			title: 'Non-commercial by design',
			body: 'Released under CC BY-NC 4.0 so universities, disability units and NGOs can adopt it without a licence fee, and so it cannot be repackaged and resold as a product.'
		},
		{
			title: 'Co-designed with students',
			body: 'The roadmap comes from interviews and field tests with blind students and campus disability services, not from assumptions made in isolation.'
		}
	];

	let heroGlobeHostEl: HTMLDivElement | undefined = $state();
	let heroGlobeCanvasEl: HTMLCanvasElement | undefined = $state();
	let heroCleanup = () => {};

	let hostEl: HTMLDivElement | undefined = $state();
	let canvasEl: HTMLCanvasElement | undefined = $state();
	let webglFailed = $state(false);

	const CENTER_LON = 106.8;
	const HUB = { lat: -6.2, lon: 106.8 };

	const NODES = [
		{ lat: 39.9, lon: 116.4 },
		{ lat: 31.78, lon: 35.22 },
		{ lat: 35.69, lon: 51.39 },
		{ lat: 6.52, lon: 3.38 },
		{ lat: 51.51, lon: -0.13 },
		{ lat: -23.55, lon: -46.63 },
		{ lat: 40.71, lon: -74.01 }
	];

	const MAP_W = 100;
	const MAP_H = 50;
	const MAP_SOURCES = [
		'https://unpkg.com/three-globe/example/img/earth-water.png',
		'https://cdn.jsdelivr.net/npm/three-globe/example/img/earth-water.png'
	];

	let cleanup = () => {};

	const loadImage = (url: string): Promise<HTMLImageElement> =>
		new Promise((resolve, reject) => {
			const img = new Image();
			img.crossOrigin = 'anonymous';
			img.onload = () => resolve(img);
			img.onerror = () => reject(new Error('image load failed'));
			img.src = url;
		});

	/* =================== GLOBAL — GSAP orchestration + Lenis =================== */
	onMount(() => {
		gsap.registerPlugin(ScrollTrigger, ScrollToPlugin);

		const prefersReduced = window.matchMedia?.('(prefers-reduced-motion: reduce)').matches ?? false;
		const finePointer = window.matchMedia?.('(hover: hover) and (pointer: fine)').matches ?? false;

		const cleanups: Array<() => void> = [];

		/* ========== LENIS SMOOTH SCROLL ========== */
		let lenis: Lenis | null = null;

		if (!prefersReduced) {
			lenis = new Lenis({
				// Hapus autoRaf: false (secara default bernilai true di versi terbaru)
				duration: 1.2,
				easing: (t: number) => Math.min(1, 1.001 - Math.pow(2, -10 * t)),
				smoothWheel: true,
				wheelMultiplier: 1,
				touchMultiplier: 1.5,
				syncTouch: false,
				prevent: (node: HTMLElement) => node.hasAttribute('data-lenis-prevent')
			});

			// Cukup hubungkan event scroll Lenis ke ScrollTrigger
			lenis.on('scroll', ScrollTrigger.update);

			cleanups.push(() => {
				lenis?.destroy();
				lenis = null;
			});
		}

		/* ---------- Custom cursor (GSAP quickTo for buttery follow) ---------- */
		if (finePointer) {
			const cursor = document.querySelector<HTMLElement>('[data-cursor]');
			const cursorInner = document.querySelector<HTMLElement>('[data-cursor-inner]');
			if (cursor && cursorInner) {
				const xTo = gsap.quickTo(cursor, 'x', { duration: 0.35, ease: 'power3' });
				const yTo = gsap.quickTo(cursor, 'y', { duration: 0.35, ease: 'power3' });

				const onMove = (e: MouseEvent) => {
					xTo(e.clientX);
					yTo(e.clientY);
				};
				window.addEventListener('mousemove', onMove, { passive: true });
				cleanups.push(() => window.removeEventListener('mousemove', onMove));

				const onOver = (e: MouseEvent) => {
					const t = e.target as HTMLElement | null;
					const interactive = !!t?.closest('a, button, [data-cursor-hover]');
					gsap.to(cursorInner, {
						scale: interactive ? 4.3 : 1,
						opacity: interactive ? 0.14 : 0.72,
						duration: 0.4,
						ease: 'power3.out',
						overwrite: 'auto'
					});
				};
				document.addEventListener('mouseover', onOver);
				cleanups.push(() => document.removeEventListener('mouseover', onOver));
			}
		}

		/* ---------- Scroll progress (scaleX — jauh lebih mulus dari width) ---------- */
		const progressEl = document.querySelector<HTMLElement>('[data-scroll-progress]');
		if (progressEl) {
			gsap.set(progressEl, { scaleX: 0 });
			const setProgress = gsap.quickTo(progressEl, 'scaleX', {
				duration: 0.18,
				ease: 'power2.out'
			});
			const onScroll = () => {
				const h = document.documentElement.scrollHeight - window.innerHeight;
				setProgress(h > 0 ? Math.min(1, Math.max(0, window.scrollY / h)) : 0);
			};
			if (lenis) {
				lenis.on('scroll', onScroll);
			} else {
				window.addEventListener('scroll', onScroll, { passive: true });
				cleanups.push(() => window.removeEventListener('scroll', onScroll));
			}
			onScroll();
		}

		/* ---------- Section observer untuk nav active state ---------- */
		const sections = document.querySelectorAll('section[id]');
		const sectionIO = new IntersectionObserver(
			(entries) => {
				entries.forEach((entry) => {
					if (entry.isIntersecting) activeSection = (entry.target as HTMLElement).id;
				});
			},
			{ rootMargin: '-45% 0px -45% 0px', threshold: 0 }
		);
		sections.forEach((el) => sectionIO.observe(el));
		cleanups.push(() => sectionIO.disconnect());

		/* ---------- Smooth anchor scroll ---------- */
		const onAnchorClick = (e: MouseEvent) => {
			const target = (e.target as HTMLElement | null)?.closest(
				'a[href^="#"]'
			) as HTMLAnchorElement | null;
			if (!target) return;
			const href = target.getAttribute('href');
			if (!href || href === '#') return;
			const dest = document.querySelector(href);
			if (!dest) return;
			e.preventDefault();

			if (lenis) {
				lenis.scrollTo(dest as HTMLElement, {
					offset: -96,
					duration: 1.2,
					easing: (t: number) => 1 - Math.pow(1 - t, 4)
				});
			} else {
				gsap.to(window, {
					scrollTo: { y: dest, offsetY: 96, autoKill: true },
					duration: 1.05,
					ease: 'power3.inOut',
					overwrite: 'auto'
				});
			}
		};
		document.addEventListener('click', onAnchorClick);
		cleanups.push(() => document.removeEventListener('click', onAnchorClick));

		/* ---------- GSAP context untuk semua ScrollTrigger ---------- */
		const ctx = gsap.context(() => {
			if (prefersReduced) return;

			/* Hero entrance timeline */
			const heroTl = gsap.timeline({ defaults: { ease: 'power3.out' } });
			heroTl
				.from('[data-hero-badge]', { y: 16, opacity: 0, duration: 0.7 })
				.from('[data-hero-title]', { y: 40, opacity: 0, duration: 1.0 }, '-=0.45')
				.from('[data-hero-desc]', { y: 24, opacity: 0, duration: 0.9 }, '-=0.7')
				.from('[data-hero-cta] > *', { y: 20, opacity: 0, stagger: 0.1, duration: 0.7 }, '-=0.55')
				.from('[data-hero-meta] > *', { y: 14, opacity: 0, stagger: 0.08, duration: 0.6 }, '-=0.5')
				.from('[data-hero-panel]', { y: 32, opacity: 0, duration: 0.95 }, '-=1.0');

			/* Generic reveal via data-reveal */
			gsap.utils.toArray<HTMLElement>('[data-reveal]').forEach((el) => {
				gsap.from(el, {
					y: 36,
					opacity: 0,
					duration: 1.0,
					ease: 'power3.out',
					scrollTrigger: { trigger: el, start: 'top 88%', once: true }
				});
			});

			/* Capabilities stagger */
			const caps = gsap.utils.toArray<HTMLElement>('[data-capability]');
			if (caps.length) {
				gsap.from(caps, {
					y: 44,
					opacity: 0,
					duration: 0.9,
					ease: 'power3.out',
					stagger: 0.12,
					scrollTrigger: { trigger: caps[0], start: 'top 82%', once: true }
				});
			}

			/* Architecture steps */
			const stepEls = gsap.utils.toArray<HTMLElement>('[data-step]');
			if (stepEls.length) {
				gsap.from(stepEls, {
					y: 40,
					opacity: 0,
					duration: 0.85,
					ease: 'power3.out',
					stagger: 0.1,
					scrollTrigger: { trigger: stepEls[0], start: 'top 85%', once: true }
				});
			}

			/* Spec rows */
			const specRows = gsap.utils.toArray<HTMLElement>('[data-spec]');
			if (specRows.length) {
				gsap.from(specRows, {
					y: 18,
					opacity: 0,
					duration: 0.55,
					ease: 'power2.out',
					stagger: 0.05,
					scrollTrigger: { trigger: specRows[0], start: 'top 85%', once: true }
				});
			}

			/* Principles */
			const principleEls = gsap.utils.toArray<HTMLElement>('[data-principle]');
			if (principleEls.length) {
				gsap.from(principleEls, {
					y: 40,
					opacity: 0,
					duration: 0.9,
					ease: 'power3.out',
					stagger: 0.12,
					scrollTrigger: { trigger: principleEls[0], start: 'top 82%', once: true }
				});
			}

			/* Download card scale-in */
			const downloadCard = document.querySelector<HTMLElement>('[data-download-card]');
			if (downloadCard) {
				gsap.from(downloadCard, {
					y: 60,
					opacity: 0,
					duration: 1.15,
					ease: 'power3.out',
					scrollTrigger: { trigger: downloadCard, start: 'top 84%', once: true }
				});
			}

			/* Map card */
			const mapCard = document.querySelector<HTMLElement>('[data-map-card]');
			if (mapCard) {
				gsap.from(mapCard, {
					y: 50,
					opacity: 0,
					duration: 1.1,
					ease: 'power3.out',
					scrollTrigger: { trigger: mapCard, start: 'top 84%', once: true }
				});
			}

			/* Footer cards */
			const footerCards = gsap.utils.toArray<HTMLElement>('[data-footer-card]');
			if (footerCards.length) {
				gsap.from(footerCards, {
					y: 40,
					opacity: 0,
					duration: 1.0,
					ease: 'power3.out',
					stagger: 0.1,
					scrollTrigger: { trigger: footerCards[0], start: 'top 88%', once: true }
				});
			}

			/* Hero globe parallax (scrub) */
			const heroGlobe = document.querySelector<HTMLElement>('[data-hero-globe]');
			const heroSection = document.querySelector<HTMLElement>('[data-hero-section]');
			if (heroGlobe && heroSection) {
				gsap.to(heroGlobe, {
					yPercent: 22,
					ease: 'none',
					scrollTrigger: {
						trigger: heroSection,
						start: 'top top',
						end: 'bottom top',
						scrub: 0.6
					}
				});
			}

			/* Magnetic buttons */
			document.querySelectorAll<HTMLElement>('[data-magnetic]').forEach((btn) => {
				const strength = 10;
				const onMove = (e: MouseEvent) => {
					const rect = btn.getBoundingClientRect();
					const dx = e.clientX - (rect.left + rect.width / 2);
					const dy = e.clientY - (rect.top + rect.height / 2);
					gsap.to(btn, {
						x: (dx / rect.width) * strength,
						y: (dy / rect.height) * strength,
						duration: 0.55,
						ease: 'power3.out',
						overwrite: 'auto'
					});
				};
				const onLeave = () => {
					gsap.to(btn, {
						x: 0,
						y: 0,
						duration: 0.8,
						ease: 'elastic.out(1, 0.4)',
						overwrite: 'auto'
					});
				};
				btn.addEventListener('mousemove', onMove);
				btn.addEventListener('mouseleave', onLeave);
				cleanups.push(() => {
					btn.removeEventListener('mousemove', onMove);
					btn.removeEventListener('mouseleave', onLeave);
				});
			});
		});

		cleanups.push(() => ctx.revert());

		/* ---------- Refresh ScrollTrigger setelah semua aset loaded ---------- */
		if (lenis) {
			const refresh = () => ScrollTrigger.refresh();
			window.addEventListener('load', refresh);
			const refreshTimer = setTimeout(refresh, 300);
			cleanups.push(() => {
				window.removeEventListener('load', refresh);
				clearTimeout(refreshTimer);
			});
		}

		return () => {
			cleanups.forEach((fn) => fn());
			ScrollTrigger.getAll().forEach((t) => t.kill());
		};
	});

	/* =================== HERO GLOBE =================== */
	onMount(() => {
		const host = heroGlobeHostEl;
		const canvas = heroGlobeCanvasEl;
		if (!host || !canvas) return;

		let renderer: THREE.WebGLRenderer;
		try {
			renderer = new THREE.WebGLRenderer({
				canvas,
				antialias: true,
				alpha: true,
				powerPreference: 'low-power'
			});
		} catch {
			return;
		}

		renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
		renderer.setClearColor(0x000000, 0);

		const scene = new THREE.Scene();
		const camera = new THREE.PerspectiveCamera(45, 1, 0.1, 100);
		camera.position.set(0, 0, 5);

		const clock = new THREE.Clock();
		const prefersReduced = window.matchMedia?.('(prefers-reduced-motion: reduce)').matches ?? false;

		const R = 1.75;

		const dotMaterial = new THREE.ShaderMaterial({
			uniforms: {
				uColorA: { value: new THREE.Color(0x1c1917) },
				uColorB: { value: new THREE.Color(0x065f46) },
				uSize: { value: 2.2 },
				uOpacity: { value: 0.0 },
				uPixelRatio: { value: 1.0 }
			},
			vertexShader: `
				uniform float uSize;
				uniform float uPixelRatio;
				varying float vDepth;
				void main() {
					vec4 mv = modelViewMatrix * vec4(position, 1.0);
					gl_Position = projectionMatrix * mv;
					gl_PointSize = uSize * uPixelRatio * (4.0 / -mv.z);
					vDepth = -mv.z;
				}
			`,
			fragmentShader: `
				uniform vec3 uColorA;
				uniform vec3 uColorB;
				uniform float uOpacity;
				varying float vDepth;
				void main() {
					vec2 d = gl_PointCoord - vec2(0.5);
					float r = length(d);
					if (r > 0.5) discard;
					float a = smoothstep(0.5, 0.2, r);
					float mixv = clamp(1.0 - vDepth / 7.0, 0.0, 1.0);
					vec3 col = mix(uColorA, uColorB, mixv);
					gl_FragColor = vec4(col, a * uOpacity);
				}
			`,
			transparent: true,
			depthWrite: false,
			blending: THREE.NormalBlending
		});

		const points = new THREE.Points(new THREE.BufferGeometry(), dotMaterial);
		scene.add(points);

		async function buildSphere() {
			let data: Uint8ClampedArray | null = null;
			const CW = 1024;
			const CH = 512;

			for (const url of MAP_SOURCES) {
				try {
					const img = await loadImage(url);
					const c = document.createElement('canvas');
					c.width = CW;
					c.height = CH;
					const ctx = c.getContext('2d', { willReadFrequently: true });
					if (ctx) {
						ctx.drawImage(img, 0, 0, CW, CH);
						data = ctx.getImageData(0, 0, CW, CH).data;
						break;
					}
				} catch {
					/* next */
				}
			}

			const N = 9000;
			const golden = Math.PI * (3 - Math.sqrt(5));
			const positions: number[] = [];

			for (let i = 0; i < N; i++) {
				const y = 1 - (i / (N - 1)) * 2;
				const rad = Math.sqrt(1 - y * y);
				const theta = golden * i;
				const x = Math.cos(theta) * rad;
				const z = Math.sin(theta) * rad;

				if (data) {
					const lat = (Math.asin(y) * 180) / Math.PI;
					const lon = (Math.atan2(z, x) * 180) / Math.PI;
					const u = ((((lon + 180) / 360) % 1) + 1) % 1;
					const v = (90 - lat) / 180;
					const px = Math.min(CW - 1, Math.max(0, Math.floor(u * CW)));
					const py = Math.min(CH - 1, Math.max(0, Math.floor(v * CH)));
					const idx = (py * CW + px) * 4;
					const lum = (data[idx] * 0.299 + data[idx + 1] * 0.587 + data[idx + 2] * 0.114) / 255;
					if (lum > 0.5) continue;
				}

				positions.push(x * R, y * R, z * R);
			}

			const geo = new THREE.BufferGeometry();
			geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
			points.geometry.dispose();
			points.geometry = geo;
		}

		function resize() {
			if (!host) return;
			const w = host.clientWidth;
			const h = host.clientHeight;
			if (!w || !h || w <= 1 || h <= 1) return;

			renderer.setSize(w, h, false);
			camera.aspect = w / h;
			camera.updateProjectionMatrix();
			dotMaterial.uniforms.uPixelRatio.value = Math.min(window.devicePixelRatio || 1, 2);

			const dist = camera.position.z;
			const fovRad = (camera.fov * Math.PI) / 180;
			const visHalfH = Math.tan(fovRad / 2) * dist;
			const visHalfW = visHalfH * camera.aspect;

			points.position.x = visHalfW * 0.72;
			points.position.y = 0.05;
		}

		let rafId: number | null = null;

		function tick() {
			rafId = requestAnimationFrame(tick);
			const t = clock.getElapsedTime();

			const fadeIn = prefersReduced ? 1 : THREE.MathUtils.smoothstep(t, 0.2, 1.6);
			dotMaterial.uniforms.uOpacity.value = 0.55 * fadeIn;

			if (!prefersReduced) points.rotation.y = t * 0.12;

			renderer.render(scene, camera);
		}

		requestAnimationFrame(() => {
			resize();
			clock.start();
			tick();
		});

		const ro = new ResizeObserver(() => resize());
		ro.observe(host);

		let rt: ReturnType<typeof setTimeout> | undefined;
		const onResize = () => {
			clearTimeout(rt);
			rt = setTimeout(resize, 140);
		};
		window.addEventListener('resize', onResize);

		const io = new IntersectionObserver(
			(entries) => {
				const next = entries[0]?.isIntersecting ?? true;
				if (next && rafId === null) {
					clock.start();
					tick();
				} else if (!next && rafId !== null) {
					cancelAnimationFrame(rafId);
					rafId = null;
				}
			},
			{ threshold: 0 }
		);
		io.observe(host);

		buildSphere();

		heroCleanup = () => {
			window.removeEventListener('resize', onResize);
			clearTimeout(rt);
			ro.disconnect();
			if (rafId !== null) cancelAnimationFrame(rafId);
			io.disconnect();
			points.geometry.dispose();
			dotMaterial.dispose();
			renderer.dispose();
		};
	});

	/* =================== FOOTER MAP =================== */
	onMount(() => {
		const host = hostEl;
		const canvas = canvasEl;
		if (!host || !canvas) return;

		let renderer: THREE.WebGLRenderer;
		try {
			renderer = new THREE.WebGLRenderer({
				canvas,
				antialias: true,
				alpha: true,
				powerPreference: 'low-power'
			});
		} catch {
			webglFailed = true;
			return;
		}

		const dpr = Math.min(window.devicePixelRatio || 1, 2);
		renderer.setPixelRatio(dpr);
		renderer.setClearColor(0x000000, 0);

		const scene = new THREE.Scene();
		const camera = new THREE.OrthographicCamera(-50, 50, 25, -25, 0.1, 100);
		camera.position.z = 20;

		const clock = new THREE.Clock();
		const prefersReduced = window.matchMedia?.('(prefers-reduced-motion: reduce)').matches ?? false;
		const CAMERA_Y = 4;

		const wrapLon = (lon: number): number => {
			const min = CENTER_LON - 180;
			const max = CENTER_LON + 180;
			let l = lon;
			while (l < min) l += 360;
			while (l > max) l -= 360;
			return l;
		};

		const project = (lat: number, lon: number): THREE.Vector3 =>
			new THREE.Vector3(((wrapLon(lon) - CENTER_LON) / 360) * MAP_W, (lat / 90) * (MAP_H / 2), 0);

		const dotMaterial = new THREE.ShaderMaterial({
			uniforms: {
				uColor: { value: new THREE.Color(0x78716c) },
				uSize: { value: 1.6 },
				uOpacity: { value: 0.0 },
				uScale: { value: dpr }
			},
			vertexShader: `
				uniform float uSize;
				uniform float uScale;
				void main() {
					vec4 mv = modelViewMatrix * vec4(position, 1.0);
					gl_Position = projectionMatrix * mv;
					gl_PointSize = uSize * uScale;
				}
			`,
			fragmentShader: `
				uniform vec3 uColor;
				uniform float uOpacity;
				void main() {
					vec2 d = gl_PointCoord - vec2(0.5);
					float r = length(d);
					if (r > 0.5) discard;
					float a = smoothstep(0.5, 0.4, r);
					gl_FragColor = vec4(uColor, a * uOpacity);
				}
			`,
			transparent: true,
			depthTest: false,
			depthWrite: false
		});

		const dotPoints = new THREE.Points(new THREE.BufferGeometry(), dotMaterial);
		dotPoints.renderOrder = 1;
		scene.add(dotPoints);

		const IMG_W = 1024;
		const IMG_H = 512;
		let imageData: Uint8ClampedArray | null = null;
		let mapReady = false;

		async function loadMapData(): Promise<boolean> {
			for (const url of MAP_SOURCES) {
				try {
					const img = await loadImage(url);
					const c = document.createElement('canvas');
					c.width = IMG_W;
					c.height = IMG_H;
					const ctx = c.getContext('2d', { willReadFrequently: true });
					if (!ctx) continue;
					ctx.drawImage(img, 0, 0, IMG_W, IMG_H);
					imageData = ctx.getImageData(0, 0, IMG_W, IMG_H).data;
					return true;
				} catch {
					/* next */
				}
			}
			return false;
		}

		function isLand(u: number, v: number): boolean {
			if (!imageData) return false;
			const px = Math.min(IMG_W - 1, Math.max(0, Math.floor(u * IMG_W)));
			const py = Math.min(IMG_H - 1, Math.max(0, Math.floor(v * IMG_H)));
			const i = (py * IMG_W + px) * 4;
			const lum =
				(imageData[i] * 0.299 + imageData[i + 1] * 0.587 + imageData[i + 2] * 0.114) / 255;
			return lum < 0.5;
		}

		let lastCols = 0;
		let lastRows = 0;

		function buildDots(cols: number, rows: number) {
			if (!imageData) return;
			lastCols = cols;
			lastRows = rows;

			const arr: number[] = [];
			for (let r = 0; r < rows; r++) {
				const v = (r + 0.5) / rows;
				const lat = (0.5 - v) * 180;
				if (lat < -58) continue;
				for (let c = 0; c < cols; c++) {
					const u = (c + 0.5) / cols;
					if (!isLand(u, v)) continue;
					const lon = (u - 0.5) * 360;
					const p = project(lat, lon);
					arr.push(p.x, p.y, 0);
				}
			}

			const geo = new THREE.BufferGeometry();
			geo.setAttribute('position', new THREE.Float32BufferAttribute(arr, 3));
			dotPoints.geometry.dispose();
			dotPoints.geometry = geo;
		}

		const arcGroup = new THREE.Group();
		arcGroup.renderOrder = 3;
		scene.add(arcGroup);

		type ArcEntry = { mat: THREE.ShaderMaterial };
		const arcs: ArcEntry[] = [];

		const makeCurve = (a: THREE.Vector3, b: THREE.Vector3): THREE.CubicBezierCurve3 => {
			const dist = a.distanceTo(b);
			const lift = Math.min(dist * 0.32, 18);
			const c1 = new THREE.Vector3().lerpVectors(a, b, 0.25);
			const c2 = new THREE.Vector3().lerpVectors(a, b, 0.75);
			c1.y += lift;
			c2.y += lift;
			return new THREE.CubicBezierCurve3(a.clone(), c1, c2, b.clone());
		};

		const TUBULAR = 140;
		const RADIAL = 6;

		function makeArcGeometry(curve: THREE.CubicBezierCurve3, radius: number): THREE.TubeGeometry {
			const geo = new THREE.TubeGeometry(curve, TUBULAR, radius, RADIAL, false);
			const pos = geo.attributes.position.array as Float32Array;
			const count = geo.attributes.position.count;
			const perRing = RADIAL + 1;
			const ringCount = TUBULAR + 1;

			const prog = new Float32Array(count);
			const center = new Float32Array(count * 3);

			for (let ring = 0; ring < ringCount; ring++) {
				let cx = 0,
					cy = 0,
					cz = 0;
				for (let j = 0; j <= RADIAL; j++) {
					const idx = (ring * perRing + j) * 3;
					cx += pos[idx];
					cy += pos[idx + 1];
					cz += pos[idx + 2];
				}
				cx /= perRing;
				cy /= perRing;
				cz /= perRing;
				for (let j = 0; j <= RADIAL; j++) {
					const i = ring * perRing + j;
					prog[i] = ring / TUBULAR;
					center[i * 3] = cx;
					center[i * 3 + 1] = cy;
					center[i * 3 + 2] = cz;
				}
			}

			geo.setAttribute('aProgress', new THREE.BufferAttribute(prog, 1));
			geo.setAttribute('aCenter', new THREE.BufferAttribute(center, 3));
			return geo;
		}

		function arcMaterial(): THREE.ShaderMaterial {
			return new THREE.ShaderMaterial({
				uniforms: {
					uProgress: { value: 0 },
					uPulse: { value: -1 },
					uColor: { value: new THREE.Color(0xa8a29e) },
					uPulseColor: { value: new THREE.Color(0x065f46) },
					uOpacity: { value: 0.7 }
				},
				vertexShader: `
					attribute float aProgress;
					attribute vec3 aCenter;
					uniform float uProgress;
					varying float vProgress;
					void main() {
						vProgress = aProgress;
						float denom = max(uProgress, 0.001);
						float local = clamp(aProgress / denom, 0.0, 1.0);
						float taper = smoothstep(0.9, 1.0, local);
						vec3 p = mix(position, aCenter, taper * 0.98);
						gl_Position = projectionMatrix * modelViewMatrix * vec4(p, 1.0);
					}
				`,
				fragmentShader: `
					uniform float uProgress;
					uniform float uPulse;
					uniform vec3 uColor;
					uniform vec3 uPulseColor;
					uniform float uOpacity;
					varying float vProgress;
					void main() {
						if (vProgress > uProgress) discard;
						float head = smoothstep(uProgress - 0.08, uProgress, vProgress);
						float d = vProgress - uPulse;
						float pulse = exp(-(d * d) / 0.0025);
						vec3 col = mix(uColor, uPulseColor, clamp(pulse, 0.0, 1.0));
						float a = uOpacity + head * 0.35 + pulse * 2.5;
						gl_FragColor = vec4(col, clamp(a, 0.0, 1.0));
					}
				`,
				transparent: true,
				depthTest: false,
				depthWrite: false
			});
		}

		const hubPos = project(HUB.lat, HUB.lon);

		const hubDot = new THREE.Mesh(
			new THREE.PlaneGeometry(0.8, 0.8),
			new THREE.MeshBasicMaterial({
				color: 0x1c1917,
				transparent: true,
				depthTest: false,
				depthWrite: false
			})
		);
		hubDot.position.copy(hubPos);
		hubDot.renderOrder = 2;
		scene.add(hubDot);

		const hubRing = new THREE.Mesh(
			new THREE.RingGeometry(0.55, 0.7, 40),
			new THREE.MeshBasicMaterial({
				color: 0x065f46,
				transparent: true,
				opacity: 0.5,
				side: THREE.DoubleSide,
				depthTest: false,
				depthWrite: false
			})
		);
		hubRing.position.copy(hubPos);
		hubRing.renderOrder = 2;
		scene.add(hubRing);

		type NodeDot = { mesh: THREE.Mesh; mat: THREE.MeshBasicMaterial };
		const nodeDots: NodeDot[] = [];

		NODES.forEach((n) => {
			const p = project(n.lat, n.lon);
			const mat = new THREE.MeshBasicMaterial({
				color: 0x065f46,
				transparent: true,
				opacity: 0,
				depthTest: false,
				depthWrite: false
			});
			const dot = new THREE.Mesh(new THREE.PlaneGeometry(0.5, 0.5), mat);
			dot.position.copy(p);
			dot.renderOrder = 2;
			scene.add(dot);
			nodeDots.push({ mesh: dot, mat });
		});

		let cssW = 1;
		let cssH = 1;
		let ppu = 1;
		let uiScale = 1;

		function rebuildArcs() {
			arcGroup.children.slice().forEach((m) => {
				arcGroup.remove(m);
				const mesh = m as THREE.Mesh;
				mesh.geometry.dispose();
				(mesh.material as THREE.Material).dispose();
			});
			arcs.length = 0;

			const radius = Math.max(0.06, 0.9 / ppu);

			NODES.forEach((n) => {
				const target = project(n.lat, n.lon);
				const curve = makeCurve(hubPos, target);
				const geo = makeArcGeometry(curve, radius);
				const mat = arcMaterial();
				const mesh = new THREE.Mesh(geo, mat);
				mesh.renderOrder = 3;
				arcGroup.add(mesh);
				arcs.push({ mat });
			});
		}

		function resize() {
			if (!host) return;
			cssW = Math.max(1, host.clientWidth);
			cssH = Math.max(1, host.clientHeight);
			renderer.setSize(cssW, cssH, false);

			const aspect = cssW / cssH;
			const visW = MAP_W;
			const visH = visW / aspect;

			camera.left = -visW / 2;
			camera.right = visW / 2;
			camera.top = visH / 2 + CAMERA_Y;
			camera.bottom = -visH / 2 + CAMERA_Y;
			camera.updateProjectionMatrix();

			ppu = cssW / visW;
			uiScale = 12 / ppu;

			dotMaterial.uniforms.uScale.value = dpr;

			const spacingPx = 2.6;
			const cols = Math.round(Math.min(720, Math.max(120, cssW / spacingPx)));
			const rows = Math.round(cols / 2);

			if (mapReady && (cols !== lastCols || rows !== lastRows)) buildDots(cols, rows);

			rebuildArcs();
			hubDot.scale.set(uiScale, uiScale, 1);
		}

		let rafId: number | null = null;

		function tick() {
			rafId = requestAnimationFrame(tick);
			const t = clock.getElapsedTime();

			const dotOp = prefersReduced ? 1 : THREE.MathUtils.smoothstep(t, 0.15, 1.1);
			dotMaterial.uniforms.uOpacity.value = 0.85 * dotOp;

			const START = 0.4;
			const STAGGER = 0.5;
			const DUR = 1.8;
			const ARC_END = 0.94;

			arcs.forEach((a, i) => {
				const start = START + i * STAGGER;
				const p = prefersReduced ? 1 : THREE.MathUtils.clamp((t - start) / DUR, 0, 1);
				a.mat.uniforms.uProgress.value = p * ARC_END;

				if (prefersReduced || p < 1) {
					a.mat.uniforms.uPulse.value = -1;
				} else {
					const cycle = 2.4;
					const local = (t - (start + DUR) + i * 0.35) % cycle;
					a.mat.uniforms.uPulse.value = (local / cycle) * ARC_END;
				}
			});

			nodeDots.forEach(({ mesh, mat }, i) => {
				const arriveT = START + i * STAGGER + DUR;
				const fade = prefersReduced ? 1 : THREE.MathUtils.clamp((t - arriveT) / 0.35, 0, 1);
				const eased = fade * fade * (3 - 2 * fade);
				mat.opacity = eased * 0.95;
				const s = (0.3 + eased * 0.7) * uiScale;
				mesh.scale.set(s, s, 1);
			});

			const ringCycle = 2.8;
			const rp = ((t + 0.4) % ringCycle) / ringCycle;
			const s = (1 + rp * 3.2) * uiScale;
			hubRing.scale.set(s, s, 1);
			(hubRing.material as THREE.MeshBasicMaterial).opacity = 0.5 * (1 - rp) * (1 - rp);

			renderer.render(scene, camera);
		}

		resize();

		const ro = new ResizeObserver(() => resize());
		ro.observe(host);

		let resizeTimer: ReturnType<typeof setTimeout> | undefined;
		const onResize = () => {
			clearTimeout(resizeTimer);
			resizeTimer = setTimeout(resize, 140);
		};
		window.addEventListener('resize', onResize);
		window.addEventListener('orientationchange', onResize);

		let visible = false;
		const io = new IntersectionObserver(
			(entries) => {
				const next = entries[0]?.isIntersecting ?? true;
				if (next === visible) return;
				visible = next;
				if (visible && rafId === null) {
					clock.start();
					tick();
				} else if (!visible && rafId !== null) {
					cancelAnimationFrame(rafId);
					rafId = null;
				}
			},
			{ threshold: 0 }
		);
		io.observe(host);

		loadMapData().then(() => {
			mapReady = true;
			lastCols = 0;
			lastRows = 0;
			resize();
		});

		cleanup = () => {
			window.removeEventListener('resize', onResize);
			window.removeEventListener('orientationchange', onResize);
			clearTimeout(resizeTimer);
			ro.disconnect();
			if (rafId !== null) cancelAnimationFrame(rafId);
			io.disconnect();
			arcGroup.children.slice().forEach((m) => {
				const mesh = m as THREE.Mesh;
				mesh.geometry.dispose();
				(mesh.material as THREE.Material).dispose();
			});
			nodeDots.forEach(({ mesh, mat }) => {
				mesh.geometry.dispose();
				mat.dispose();
			});
			dotPoints.geometry.dispose();
			dotMaterial.dispose();
			hubDot.geometry.dispose();
			(hubDot.material as THREE.Material).dispose();
			hubRing.geometry.dispose();
			(hubRing.material as THREE.Material).dispose();
			renderer.dispose();
		};
	});

	onDestroy(() => {
		cleanup();
		heroCleanup();
	});
</script>

<svelte:head>
	<title>NetraSense — Offline-first AI for blind and low-vision students</title>
	<meta
		name="description"
		content="NetraSense is an offline-first, multi-sensory assistant for blind and low-vision university students: on-device vision, zero-latency classroom audio and 360° spatial navigation. Built by Sentinel Project ID."
	/>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&family=Archivo:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&family=Instrument+Serif:ital@0;1&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div
	class="min-h-screen bg-[#FAFAF9] text-stone-900 antialiased selection:bg-emerald-900 selection:text-white"
>
	<!-- ==================== SCROLL PROGRESS BAR ==================== -->
	<div
		data-scroll-progress
		class="fixed top-0 left-0 z-[100] h-[2px] w-full origin-left bg-emerald-800"
		aria-hidden="true"
	></div>

	<!-- ==================== CUSTOM CURSOR ==================== -->
	<div
		data-cursor
		class="cursor-dot pointer-events-none fixed top-0 left-0 z-[99]"
		aria-hidden="true"
	>
		<div
			data-cursor-inner
			class="absolute h-1.5 w-1.5 -translate-x-1/2 -translate-y-1/2 rounded-full bg-emerald-800"
			style="opacity: 0.72;"
		></div>
	</div>

	<!-- ==================== HEADER — DYNAMIC ISLAND ==================== -->
	<header class="sticky top-3 z-50 md:top-4">
		<div class="mx-auto w-[90%] max-w-[1720px]">
			<div
				class="rounded-full border border-stone-200/70 bg-[#FAFAF9]/85 shadow-[0_8px_24px_-16px_rgba(15,23,42,0.18)] backdrop-blur-md"
			>
				<div class="flex h-14 items-center justify-between gap-3 px-3 md:h-16 md:px-5">
					<div class="flex items-center gap-7">
						<a
							href="/"
							class="flex items-center gap-2.5 rounded-full focus:outline-none focus-visible:ring-2 focus-visible:ring-emerald-800"
						>
							<img
								src="/netra_logo.png"
								alt="NetraSense"
								class="h-6 w-auto object-contain md:h-7"
							/>
						</a>

						<nav class="hidden items-center gap-0.5 lg:flex" aria-label="Primary">
							{#each navLinks as link}
								{@const isActive = activeSection === link.href.slice(1)}
								<a
									href={link.href}
									aria-current={isActive ? 'true' : undefined}
									class="relative rounded-full px-3 py-2 text-[13.5px] font-medium transition-colors hover:bg-stone-200/50 {isActive
										? 'text-stone-900'
										: 'text-stone-600 hover:text-stone-900'}"
								>
									{link.label}
									{#if isActive}
										<span
											class="absolute -bottom-0.5 left-1/2 h-[3px] w-[3px] -translate-x-1/2 rounded-full bg-emerald-800"
										></span>
									{/if}
								</a>
							{/each}

							{#each navDropdowns as dropdown}
								<div class="group relative">
									<button
										type="button"
										class="inline-flex items-center gap-1.5 rounded-full px-3 py-2 text-[13.5px] font-medium text-stone-600 transition-colors hover:bg-stone-200/50 hover:text-stone-900 focus:outline-none focus-visible:ring-2 focus-visible:ring-emerald-800"
										aria-haspopup="true"
										aria-expanded="false"
									>
										{dropdown.label}
										<svg
											class="h-3 w-3 transition-transform duration-200 group-focus-within:rotate-180 group-hover:rotate-180"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											aria-hidden="true"
										>
											<path d="m6 9 6 6 6-6" />
										</svg>
									</button>

									<div
										class="invisible absolute top-full left-0 z-[60] pt-2 opacity-0 transition-all duration-150 group-focus-within:visible group-focus-within:opacity-100 group-hover:visible group-hover:opacity-100"
									>
										<div
											class="w-[280px] rounded-2xl border border-stone-200 bg-white py-1.5 shadow-lg shadow-stone-900/5"
										>
											{#each dropdown.items as item}
												<a
													href={item.href}
													target={item.external ? '_blank' : undefined}
													rel={item.external ? 'noreferrer' : undefined}
													class="block rounded-xl px-3.5 py-2.5 transition-colors hover:bg-stone-50"
												>
													<div class="flex items-center gap-1.5">
														<span class="text-[13px] font-medium text-stone-900">{item.label}</span>
														{#if item.external}
															<svg
																class="h-3 w-3 text-stone-400"
																viewBox="0 0 24 24"
																fill="none"
																stroke="currentColor"
																stroke-width="2"
																stroke-linecap="round"
																stroke-linejoin="round"
																aria-hidden="true"
															>
																<path d="M7 17 17 7M7 7h10v10" />
															</svg>
														{/if}
													</div>
													<p class="mt-0.5 text-[12px] text-stone-500">{item.desc}</p>
												</a>
											{/each}
										</div>
									</div>
								</div>
							{/each}
						</nav>
					</div>

					<div class="flex items-center gap-2">
						<a
							href="#download"
							class="hidden items-center gap-2 rounded-full bg-stone-900 px-4 py-2 text-[13px] font-medium text-white transition-colors hover:bg-stone-800 md:inline-flex"
						>
							Download beta
						</a>
						<button
							type="button"
							class="rounded-full p-2 text-stone-700 transition-colors hover:bg-stone-200/60 lg:hidden"
							on:click={() => (mobileNavOpen = !mobileNavOpen)}
							aria-label={mobileNavOpen ? 'Close menu' : 'Open menu'}
							aria-expanded={mobileNavOpen}
						>
							<svg
								class="h-5 w-5"
								fill="none"
								stroke="currentColor"
								stroke-width="1.75"
								viewBox="0 0 24 24"
								aria-hidden="true"
							>
								{#if mobileNavOpen}
									<path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" />
								{:else}
									<path
										stroke-linecap="round"
										stroke-linejoin="round"
										d="M4 7h16M4 12h16M4 17h16"
									/>
								{/if}
							</svg>
						</button>
					</div>
				</div>
			</div>

			{#if mobileNavOpen}
				<div
					class="mt-3 overflow-hidden rounded-3xl border border-stone-200/70 bg-white/95 shadow-lg shadow-stone-900/5 backdrop-blur-md lg:hidden"
				>
					<nav class="flex flex-col p-3" aria-label="Mobile">
						{#each navLinks as link}
							<a
								href={link.href}
								on:click={() => (mobileNavOpen = false)}
								class="rounded-2xl px-4 py-3 text-[15px] font-medium text-stone-800 transition-colors hover:bg-stone-100"
								>{link.label}</a
							>
						{/each}

						{#each navDropdowns as dropdown}
							<p
								class="mono mt-3 px-4 pt-2 pb-1 text-[10.5px] tracking-[0.18em] text-stone-500 uppercase"
							>
								{dropdown.label}
							</p>
							{#each dropdown.items as item}
								<a
									href={item.href}
									target={item.external ? '_blank' : undefined}
									rel={item.external ? 'noreferrer' : undefined}
									on:click={() => (mobileNavOpen = false)}
									class="flex items-center justify-between rounded-2xl px-4 py-3 text-[15px] text-stone-700 transition-colors hover:bg-stone-100"
								>
									{item.label}
									{#if item.external}
										<svg
											class="h-3.5 w-3.5 text-stone-400"
											viewBox="0 0 24 24"
											fill="none"
											stroke="currentColor"
											stroke-width="2"
											stroke-linecap="round"
											stroke-linejoin="round"
											aria-hidden="true"
										>
											<path d="M7 17 17 7M7 7h10v10" />
										</svg>
									{/if}
								</a>
							{/each}
						{/each}

						<a
							href="#download"
							on:click={() => (mobileNavOpen = false)}
							class="mt-3 inline-flex items-center justify-center gap-2 rounded-2xl bg-stone-900 px-4 py-3 text-[14px] font-medium text-white"
						>
							<svg class="h-4 w-4" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
								<path
									d="M17.6 9.48l1.84-3.18a.38.38 0 0 0-.14-.52.38.38 0 0 0-.52.14L16.9 9.1a11.31 11.31 0 0 0-9.8 0L5.22 5.92a.38.38 0 0 0-.52-.14.38.38 0 0 0-.14.52l1.84 3.18A11.02 11.02 0 0 0 .5 19h23a11.02 11.02 0 0 0-5.9-9.52ZM7 16.5a1.5 1.5 0 1 1 0-3 1.5 1.5 0 0 1 0 3Zm10 0a1.5 1.5 0 1 1 0-3 1.5 1.5 0 0 1 0 3Z"
								/>
							</svg>
							Download beta
						</a>
					</nav>
				</div>
			{/if}
		</div>
	</header>

	<!-- ==================== HERO ==================== -->
	<section data-hero-section class="relative isolate overflow-hidden pt-6 lg:pt-8">
		<div
			class="pointer-events-none absolute inset-0 z-0 opacity-[0.55]"
			style="background-image: linear-gradient(to right, rgba(28,25,23,0.045) 1px, transparent 1px); background-size: 90px 100%;"
			aria-hidden="true"
		></div>

		<div
			data-hero-globe
			bind:this={heroGlobeHostEl}
			class="pointer-events-none absolute inset-0 z-0 hidden opacity-0 transition-opacity duration-700 lg:block lg:opacity-100"
		>
			<canvas
				bind:this={heroGlobeCanvasEl}
				class="absolute inset-0 h-full w-full"
				aria-hidden="true"
			></canvas>
			<div
				class="absolute inset-y-0 left-0 w-[55%] bg-gradient-to-r from-[#FAFAF9] via-[#FAFAF9]/85 to-transparent"
			></div>
		</div>

		<div class="relative z-10 mx-auto w-[90%] max-w-[1720px] pt-14 pb-24 lg:pt-24 lg:pb-32">
			<div class="grid items-center gap-16 lg:grid-cols-12 lg:gap-20">
				<div class="lg:col-span-7">
					<div data-hero-badge class="flex items-baseline gap-4">
						<span class="mono text-[11px] tracking-[0.2em] text-stone-400">—</span>
						<p class="mono text-[11px] font-medium tracking-[0.2em] text-emerald-900 uppercase">
							Sentinel Project ID · Accessibility
						</p>
					</div>

					<h1
						data-hero-title
						class="mt-6 text-[2.75rem] leading-[0.98] tracking-[-0.02em] text-stone-900 sm:text-[3.5rem] lg:text-[4.25rem] xl:text-[4.75rem]"
						style="font-family: 'Instrument Serif', ui-serif, Georgia, serif; font-weight: 400;"
					>
						Independence that runs <em class="italic">offline</em>, on the phone already in their
						pocket.
					</h1>

					<p
						data-hero-desc
						class="mt-8 max-w-xl text-[17px] leading-[1.65] text-stone-600 xl:max-w-2xl xl:text-[18px]"
					>
						NetraSense is a multi-sensory assistant for blind and low-vision university students. It
						identifies obstacles, rupiah denominations and personal belongings in real time, maps
						campus routes in 360°, and filters classroom audio — without a single network request.
					</p>

					<div data-hero-cta class="mt-10 flex flex-col gap-3 sm:flex-row">
						<a
							href="#download"
							data-magnetic
							class="group inline-flex items-center justify-center gap-2 rounded-md bg-emerald-900 px-5 py-3 text-[14px] font-medium text-white transition-all hover:bg-emerald-950 focus:outline-none focus-visible:ring-2 focus-visible:ring-emerald-800 focus-visible:ring-offset-2 focus-visible:ring-offset-[#FAFAF9]"
						>
							<svg class="h-4 w-4" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
								<path
									d="M17.6 9.48l1.84-3.18a.38.38 0 0 0-.14-.52.38.38 0 0 0-.52.14L16.9 9.1a11.31 11.31 0 0 0-9.8 0L5.22 5.92a.38.38 0 0 0-.52-.14.38.38 0 0 0-.14.52l1.84 3.18A11.02 11.02 0 0 0 .5 19h23a11.02 11.02 0 0 0-5.9-9.52ZM7 16.5a1.5 1.5 0 1 1 0-3 1.5 1.5 0 0 1 0 3Zm10 0a1.5 1.5 0 1 1 0-3 1.5 1.5 0 0 1 0 3Z"
								/>
							</svg>
							Download the beta APK
							<svg
								class="h-3.5 w-3.5 transition-transform group-hover:translate-x-0.5"
								viewBox="0 0 24 24"
								fill="none"
								stroke="currentColor"
								stroke-width="2"
								stroke-linecap="round"
								stroke-linejoin="round"
								aria-hidden="true"
							>
								<path d="M5 12h14M13 6l6 6-6 6" />
							</svg>
						</a>
						<a
							href="https://github.com"
							target="_blank"
							rel="noreferrer"
							data-magnetic
							class="inline-flex items-center justify-center gap-2 rounded-md border border-stone-300 bg-white px-5 py-3 text-[14px] font-medium text-stone-800 transition-colors hover:border-stone-400 hover:bg-stone-50"
						>
							<svg
								class="h-4 w-4 fill-current text-stone-700"
								viewBox="0 0 24 24"
								aria-hidden="true"
							>
								<path
									d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"
								/>
							</svg>
							Read the source
						</a>
					</div>

					<dl
						data-hero-meta
						class="mt-12 grid max-w-xl grid-cols-2 gap-x-8 gap-y-5 border-t border-stone-200/80 pt-6 sm:grid-cols-3 xl:max-w-2xl"
					>
						<div>
							<dt class="mono text-[10.5px] tracking-[0.16em] text-stone-500 uppercase">Version</dt>
							<dd class="mt-1.5 text-[13.5px] text-stone-800">v0.9.2 beta</dd>
						</div>
						<div>
							<dt class="mono text-[10.5px] tracking-[0.16em] text-stone-500 uppercase">
								Platform
							</dt>
							<dd class="mt-1.5 text-[13.5px] text-stone-800">Android 10+</dd>
						</div>
						<div>
							<dt class="mono text-[10.5px] tracking-[0.16em] text-stone-500 uppercase">Licence</dt>
							<dd class="mt-1.5 text-[13.5px] text-stone-800">CC BY-NC 4.0</dd>
						</div>
					</dl>
				</div>

				<div data-hero-panel class="lg:col-span-5">
					<div class="overflow-hidden rounded-md border border-stone-800 bg-[#0A0A0A]">
						<div class="flex items-center justify-between border-b border-stone-800 px-4 py-2.5">
							<span class="mono text-[11px] text-stone-500">netrasense — local pipeline</span>
							<span class="mono flex items-center gap-2 text-[11px] text-emerald-500">
								<span class="h-1 w-1 rounded-full bg-emerald-500"></span>
								airplane mode
							</span>
						</div>
						<div class="mono space-y-3 px-4 py-4 text-[11.5px] leading-[1.65] text-stone-400">
							<div>
								<span class="text-stone-600">12:04:18.204</span>
								<span class="ml-3 text-emerald-500">vision.infer</span>
								<span class="ml-3 text-stone-500">yolo_nano_int8 · 38ms</span>
								<p class="mt-0.5 pl-5 text-stone-500">→ rupiah_50k · tote_bag · doorway</p>
							</div>
							<div>
								<span class="text-stone-600">12:04:18.259</span>
								<span class="ml-3 text-emerald-500">audio.dsp</span>
								<span class="ml-3 text-stone-500">oboe_stream · 4.1ms</span>
								<p class="mt-0.5 pl-5 text-stone-500">→ vad · speech_detected (near field)</p>
							</div>
							<div>
								<span class="text-stone-600">12:04:19.871</span>
								<span class="ml-3 text-emerald-500">vpr.match</span>
								<span class="ml-3 text-stone-500">campus/blok-c · 0.94</span>
								<p class="mt-0.5 pl-5 text-stone-500">→ "Turn right in 12 metres"</p>
							</div>
							<div class="pt-1">
								<span class="text-stone-600">12:04:20.003</span>
								<span class="ml-3 text-stone-700">network.out</span>
								<span class="ml-3 text-stone-600">— skipped</span>
							</div>
						</div>
					</div>
					<p class="mono mt-3 text-[11px] text-stone-500">
						Recorded on a Pixel 6a, airplane mode, zero network calls.
					</p>
				</div>
			</div>
		</div>
	</section>

	<!-- ==================== TICKER MARQUEE ==================== -->
	<section class="overflow-hidden border-y border-stone-200/80 bg-white py-4">
		<div class="marquee flex whitespace-nowrap">
			{#each [0, 1] as _}
				<div class="flex shrink-0 items-center gap-10 pr-10">
					{#each ticker as word}
						<span class="mono text-[10.5px] tracking-[0.22em] text-stone-500 uppercase">
							{word}
						</span>
						<span class="h-1 w-1 shrink-0 rounded-full bg-emerald-800/60" aria-hidden="true"></span>
					{/each}
				</div>
			{/each}
		</div>
	</section>

	<!-- ==================== STATS ==================== -->
	<section class="border-b border-stone-200/80 bg-white">
		<div class="mx-auto w-[90%] max-w-[1720px]">
			<dl class="grid grid-cols-2 lg:grid-cols-4">
				<div
					data-reveal
					class="border-b border-stone-200/80 py-8 pr-6 lg:border-r lg:border-b-0 lg:pr-8"
				>
					<dd class="mono text-[28px] leading-none font-medium tracking-tight text-stone-900">
						&lt;5<span class="ml-1 text-[16px] text-stone-500">ms</span>
					</dd>
					<dt class="mt-3 text-[12.5px] leading-snug text-stone-500">
						Audio DSP latency, C++ / Oboe
					</dt>
				</div>
				<div
					data-reveal
					class="border-b border-l border-stone-200/80 py-8 pl-6 lg:border-r lg:border-b-0 lg:border-l-0 lg:px-8"
				>
					<dd class="mono text-[28px] leading-none font-medium tracking-tight text-stone-900">0</dd>
					<dt class="mt-3 text-[12.5px] leading-snug text-stone-500">
						Network requests at runtime
					</dt>
				</div>
				<div
					data-reveal
					class="border-b border-stone-200/80 py-8 pr-6 lg:border-r lg:border-b-0 lg:px-8"
				>
					<dd class="mono text-[28px] leading-none font-medium tracking-tight text-stone-900">
						360<span class="ml-0.5 text-[16px] text-stone-500">°</span>
					</dd>
					<dt class="mt-3 text-[12.5px] leading-snug text-stone-500">
						Spatial navigation coverage
					</dt>
				</div>
				<div
					data-reveal
					class="border-b border-l border-stone-200/80 py-8 pl-6 lg:border-b-0 lg:border-l-0 lg:pl-8"
				>
					<dd class="mono text-[28px] leading-none font-medium tracking-tight text-stone-900">
						BY-NC
					</dd>
					<dt class="mt-3 text-[12.5px] leading-snug text-stone-500">Creative Commons licence</dt>
				</div>
			</dl>
		</div>
	</section>

	<!-- ==================== CAPABILITIES ==================== -->
	<section id="capabilities" class="border-b border-stone-200/80">
		<div class="mx-auto w-[90%] max-w-[1720px] py-20 lg:py-28">
			<div class="grid gap-16 lg:grid-cols-12 lg:gap-20">
				<div class="lg:col-span-4">
					<div class="lg:sticky lg:top-28">
						<div data-reveal class="flex items-baseline gap-4">
							<span class="mono text-[11px] tracking-[0.2em] text-stone-400">01</span>
							<p class="mono text-[11px] font-medium tracking-[0.2em] text-emerald-900 uppercase">
								Capabilities
							</p>
						</div>
						<h2
							data-reveal
							class="mt-4 text-[1.9rem] leading-[1.1] font-semibold tracking-[-0.025em] text-stone-900 sm:text-[2.15rem] xl:text-[2.4rem]"
						>
							Three systems, one pipeline
						</h2>
						<p data-reveal class="mt-5 text-[15px] leading-[1.7] text-stone-600 xl:text-[15.5px]">
							Each subsystem answers a specific problem reported by students in the field. They
							share a single inference budget and a single audio output, so the user hears one
							coherent voice instead of three competing ones.
						</p>
					</div>
				</div>

				<div class="lg:col-span-8">
					<div class="border-t border-stone-200/80">
						{#each capabilities as item}
							<article
								data-capability
								class="group grid gap-x-10 gap-y-4 border-b border-stone-200/80 py-10 sm:grid-cols-[3rem_1fr] xl:py-12"
							>
								<span
									class="mono pt-1 text-[12px] text-stone-400 transition-colors group-hover:text-emerald-800"
								>
									{item.id}
								</span>
								<div>
									<h3
										class="text-[19px] font-semibold tracking-[-0.01em] text-stone-900 xl:text-[21px]"
									>
										{item.title}
									</h3>
									<p
										class="mt-3 max-w-2xl text-[15px] leading-[1.7] text-stone-600 xl:max-w-3xl xl:text-[15.5px]"
									>
										{item.body}
									</p>
									<ul
										class="mono mt-5 flex flex-wrap gap-x-6 gap-y-2 text-[10.5px] tracking-[0.14em] text-stone-500 uppercase"
									>
										{#each item.tags as tag}<li>{tag}</li>{/each}
									</ul>
								</div>
							</article>
						{/each}
					</div>
				</div>
			</div>
		</div>
	</section>

	<!-- ==================== ARCHITECTURE ==================== -->
	<section id="architecture" class="border-b border-stone-200/80 bg-white">
		<div class="mx-auto w-[90%] max-w-[1720px] py-20 lg:py-28">
			<div class="grid gap-12 lg:grid-cols-12 lg:gap-20">
				<div class="lg:col-span-4">
					<div data-reveal class="flex items-baseline gap-4">
						<span class="mono text-[11px] tracking-[0.2em] text-stone-400">02</span>
						<p class="mono text-[11px] font-medium tracking-[0.2em] text-emerald-900 uppercase">
							Architecture
						</p>
					</div>
					<h2
						data-reveal
						class="mt-4 text-[1.9rem] leading-[1.1] font-semibold tracking-[-0.025em] text-stone-900 sm:text-[2.15rem] xl:text-[2.4rem]"
					>
						Four stages, none of them online
					</h2>
					<p data-reveal class="mt-5 text-[15px] leading-[1.7] text-stone-600 xl:text-[15.5px]">
						The pipeline is deliberately linear. Every stage has a hard latency budget, and any
						stage that cannot meet it degrades to a simpler local model rather than reaching for the
						network.
					</p>
				</div>

				<div class="lg:col-span-8">
					<ol
						class="grid gap-px overflow-hidden rounded-md border border-stone-200/80 bg-stone-200/80 sm:grid-cols-2"
					>
						{#each steps as step}
							<li data-step class="bg-white p-6 lg:p-7 xl:p-8">
								<span class="mono text-[10.5px] tracking-[0.18em] text-stone-500 uppercase">
									{step.step}
								</span>
								<h3 class="mt-3 text-[15.5px] font-semibold text-stone-900">{step.title}</h3>
								<p class="mt-2 text-[13.5px] leading-[1.65] text-stone-600">{step.body}</p>
							</li>
						{/each}
					</ol>
				</div>
			</div>

			<div class="mt-20 grid gap-12 border-t border-stone-200/80 pt-16 lg:grid-cols-12 lg:gap-20">
				<div class="lg:col-span-4">
					<h3 class="text-[17px] font-semibold tracking-[-0.01em] text-stone-900">
						Technical specification
					</h3>
					<p class="mt-3 text-[14px] leading-[1.7] text-stone-600">
						Everything below is verified against the release build published on GitHub.
					</p>
				</div>
				<div class="lg:col-span-8">
					<dl class="border-t border-stone-200/80">
						{#each specs as [term, value]}
							<div
								data-spec
								class="grid gap-1.5 border-b border-stone-200/80 py-3.5 sm:grid-cols-[12rem_1fr] sm:gap-8 xl:py-4"
							>
								<dt class="mono text-[10.5px] tracking-[0.16em] text-stone-500 uppercase sm:pt-1">
									{term}
								</dt>
								<dd class="text-[14px] leading-[1.6] text-stone-700">{value}</dd>
							</div>
						{/each}
					</dl>
				</div>
			</div>
		</div>
	</section>

	<!-- ==================== IMPACT ==================== -->
	<section id="impact" class="border-b border-stone-200/80">
		<div class="mx-auto w-[90%] max-w-[1720px] py-20 lg:py-28">
			<div class="grid gap-12 lg:grid-cols-12 lg:gap-20">
				<div class="lg:col-span-4">
					<div data-reveal class="flex items-baseline gap-4">
						<span class="mono text-[11px] tracking-[0.2em] text-stone-400">03</span>
						<p class="mono text-[11px] font-medium tracking-[0.2em] text-emerald-900 uppercase">
							Social impact
						</p>
					</div>
					<h2
						data-reveal
						class="mt-4 text-[1.9rem] leading-[1.1] font-semibold tracking-[-0.025em] text-stone-900 sm:text-[2.15rem] xl:text-[2.4rem]"
					>
						Assistive technology should not require a subscription
					</h2>
					<p data-reveal class="mt-5 text-[15px] leading-[1.7] text-stone-600 xl:text-[15.5px]">
						Most vision-assistance apps depend on a cloud service, a data plan and a monthly fee.
						For a student navigating a campus between lectures, all three are unreliable. NetraSense
						was built around the opposite assumption.
					</p>
				</div>

				<div class="lg:col-span-8">
					<div
						class="grid gap-px overflow-hidden rounded-md border border-stone-200/80 bg-stone-200/80 sm:grid-cols-2"
					>
						{#each principles as principle, i}
							<div data-principle class="bg-white p-7 lg:p-8 {i === 2 ? 'sm:col-span-2' : ''}">
								<h3 class="text-[15.5px] leading-snug font-semibold text-stone-900">
									{principle.title}
								</h3>
								<p class="mt-3 text-[13.5px] leading-[1.7] text-stone-600">{principle.body}</p>
							</div>
						{/each}
					</div>
				</div>
			</div>
		</div>
	</section>

	<!-- ==================== DOWNLOAD ==================== -->
	<section id="download" class="bg-white">
		<div class="mx-auto w-[90%] max-w-[1720px] py-20 lg:py-24">
			<div
				data-download-card
				class="relative isolate overflow-hidden rounded-2xl bg-emerald-950 px-8 py-12 sm:px-12 sm:py-16 lg:px-16"
			>
				<div class="pointer-events-none absolute inset-0 overflow-hidden">
					<div
						class="absolute inset-0 opacity-[0.055]"
						style="background-image: radial-gradient(circle at 1px 1px, #6ee7b7 1px, transparent 0); background-size: 22px 22px;"
					></div>
					<div
						class="absolute -top-44 -right-44 h-[560px] w-[560px] rounded-full border border-emerald-400/[0.10]"
					></div>
					<div
						class="absolute -top-36 -right-36 h-[440px] w-[440px] rounded-full border border-emerald-400/[0.08]"
					></div>
					<div
						class="absolute -top-28 -right-28 h-[320px] w-[320px] rounded-full border border-emerald-400/[0.06]"
					></div>
					<div
						class="absolute -bottom-56 -left-40 h-[500px] w-[500px] rounded-full border border-emerald-400/[0.08]"
					></div>
					<div
						class="absolute -bottom-44 -left-32 h-[380px] w-[380px] rounded-full border border-emerald-400/[0.06]"
					></div>
					<div
						class="absolute top-1/2 -right-20 h-px w-[60%] -translate-y-1/2 rotate-[-18deg] bg-gradient-to-r from-transparent via-emerald-400/20 to-transparent"
					></div>
					<div class="absolute top-1/4 right-1/4 h-1 w-1 rounded-full bg-emerald-300/60"></div>
					<div
						class="absolute bottom-1/4 left-1/4 h-1.5 w-1.5 rounded-full bg-emerald-400/40"
					></div>
					<div class="absolute top-2/3 right-1/3 h-1 w-1 rounded-full bg-emerald-300/50"></div>
					<div
						class="absolute -right-24 -bottom-32 h-[400px] w-[400px] rounded-full bg-emerald-500/10 blur-3xl"
					></div>
				</div>

				<div class="relative grid gap-12 lg:grid-cols-12 lg:gap-16">
					<div class="lg:col-span-7">
						<div class="flex items-baseline gap-4">
							<span class="mono text-[11px] tracking-[0.2em] text-emerald-400/40">—</span>
							<p
								class="mono text-[11px] font-medium tracking-[0.2em] text-emerald-400/80 uppercase"
							>
								Beta release
							</p>
						</div>
						<h2
							class="mt-4 text-[1.9rem] leading-[1.15] font-semibold tracking-[-0.02em] text-white sm:text-[2.15rem] xl:text-[2.4rem]"
						>
							Try NetraSense on an Android device
						</h2>
						<p
							class="mt-5 max-w-lg text-[14.5px] leading-[1.7] text-emerald-100/70 xl:max-w-xl xl:text-[15.5px]"
						>
							The beta is signed and distributed directly. It is intended for testing and feedback —
							please report issues on GitHub rather than relying on it in unsupervised settings.
						</p>
						<div class="mt-9 flex flex-col gap-3 sm:flex-row">
							<a
								href="https://github.com"
								target="_blank"
								rel="noreferrer"
								data-magnetic
								class="group inline-flex items-center justify-center gap-2.5 rounded-md bg-white px-5 py-3 text-[14px] font-medium text-emerald-950 transition-colors hover:bg-emerald-50"
							>
								<svg class="h-4 w-4" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
									<path
										d="M17.6 9.48l1.84-3.18a.38.38 0 0 0-.14-.52.38.38 0 0 0-.52.14L16.9 9.1a11.31 11.31 0 0 0-9.8 0L5.22 5.92a.38.38 0 0 0-.52-.14.38.38 0 0 0-.14.52l1.84 3.18A11.02 11.02 0 0 0 .5 19h23a11.02 11.02 0 0 0-5.9-9.52ZM7 16.5a1.5 1.5 0 1 1 0-3 1.5 1.5 0 0 1 0 3Zm10 0a1.5 1.5 0 1 1 0-3 1.5 1.5 0 0 1 0 3Z"
									/>
								</svg>
								Download APK (42 MB)
								<svg
									class="h-3.5 w-3.5 transition-transform group-hover:translate-y-0.5"
									viewBox="0 0 24 24"
									fill="none"
									stroke="currentColor"
									stroke-width="2"
									stroke-linecap="round"
									stroke-linejoin="round"
									aria-hidden="true"
								>
									<path d="M12 4v12m0 0-4-4m4 4 4-4M5 20h14" />
								</svg>
							</a>
							<a
								href="https://github.com"
								target="_blank"
								rel="noreferrer"
								data-magnetic
								class="inline-flex items-center justify-center gap-2.5 rounded-md border border-emerald-400/25 px-5 py-3 text-[14px] font-medium text-emerald-50 transition-colors hover:border-emerald-400/60 hover:bg-emerald-900/40"
							>
								<svg class="h-4 w-4 fill-current" viewBox="0 0 24 24" aria-hidden="true">
									<path
										d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"
									/>
								</svg>
								Build from source
							</a>
						</div>
						<p class="mono mt-7 text-[11px] leading-relaxed text-emerald-200/40">
							SHA-256 checksums are published alongside every release tag.
						</p>
					</div>

					<div class="lg:col-span-5">
						<dl class="border-t border-emerald-400/20">
							{#each [['Version', '0.9.2 (beta)'], ['Requires', 'Android 10+ · 4 GB RAM'], ['Permissions', 'Camera, microphone, foreground location'], ['Bundle', '42 MB, models included'], ['Licence', 'CC BY-NC 4.0']] as [term, value]}
								<div
									class="flex items-baseline justify-between gap-6 border-b border-emerald-400/20 py-3.5"
								>
									<dt class="mono text-[10.5px] tracking-[0.16em] text-emerald-300/60 uppercase">
										{term}
									</dt>
									<dd class="text-right text-[13.5px] text-emerald-50">{value}</dd>
								</div>
							{/each}
						</dl>
					</div>
				</div>
			</div>
		</div>
	</section>

	<!-- ==================== MAP ==================== -->
	<section class="relative isolate border-t border-stone-200/80">
		<div class="mx-auto w-[90%] max-w-[1720px] py-14 lg:py-20">
			<div
				data-map-card
				bind:this={hostEl}
				class="relative h-[440px] w-full overflow-hidden rounded-2xl border border-stone-200/80 bg-white sm:h-[520px] lg:h-[600px]"
			>
				<canvas bind:this={canvasEl} class="absolute inset-0 h-full w-full" aria-hidden="true"
				></canvas>

				<div
					class="pointer-events-none absolute inset-x-0 top-0 z-[4] h-20 bg-gradient-to-b from-white via-white/60 to-transparent"
				></div>
				<div
					class="pointer-events-none absolute inset-x-0 bottom-0 z-[4] h-56 bg-gradient-to-t from-white via-white/95 to-transparent"
				></div>

				<div
					class="pointer-events-none absolute inset-x-0 bottom-0 z-10 flex flex-col items-center px-6 pb-10 text-center sm:pb-14"
				>
					<div class="flex items-baseline gap-4">
						<span class="mono text-[10.5px] tracking-[0.22em] text-stone-400">—</span>
						<p class="mono text-[10.5px] font-medium tracking-[0.22em] text-emerald-900 uppercase">
							Built in Jakarta
						</p>
					</div>

					<h2
						class="mt-4 max-w-2xl text-[1.5rem] leading-[1.15] font-semibold tracking-[-0.025em] text-stone-900 sm:text-[1.9rem]"
					>
						Built in Jakarta. Designed to work anywhere.
					</h2>

					<p class="mono mt-4 text-[10.5px] tracking-[0.22em] text-stone-500 uppercase">
						Offline by default · On-device inference
					</p>

					{#if webglFailed}
						<p class="mt-4 max-w-md text-[12.5px] leading-relaxed text-stone-400">
							Visualisation unavailable — your browser does not support WebGL.
						</p>
					{/if}
				</div>
			</div>
		</div>
	</section>

	<!-- ==================== FOOTER — MODAL STYLE ==================== -->
	<footer class="bg-[#FAFAF9] pb-6 lg:pb-8">
		<div class="mx-auto w-[90%] max-w-[1720px]">
			<div
				data-footer-card
				class="rounded-3xl bg-[#0A0A0A] px-8 py-14 text-stone-400 sm:px-12 sm:py-16 lg:px-16"
			>
				<div class="grid gap-12 lg:grid-cols-12 lg:gap-16">
					<div class="lg:col-span-5">
						<img
							src="/netrafull_logo.png"
							alt="NetraSense"
							class="h-8 w-auto brightness-0 invert"
						/>
						<p class="mt-6 max-w-sm text-[13.5px] leading-[1.75] text-stone-500">
							An offline-first, multi-sensory assistant for blind and low-vision university
							students. Built and maintained in Jakarta by Sentinel Project ID.
						</p>

						<div class="mt-7 flex flex-wrap items-center gap-2">
							<a
								href="https://github.com"
								target="_blank"
								rel="noreferrer"
								aria-label="GitHub"
								class="flex h-9 w-9 items-center justify-center rounded-md border border-stone-800 text-stone-400 transition-colors hover:border-stone-700 hover:bg-stone-900 hover:text-stone-100"
							>
								<svg class="h-4 w-4" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
									<path
										d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"
									/>
								</svg>
							</a>
							<a
								href="https://t.me"
								target="_blank"
								rel="noreferrer"
								aria-label="Telegram"
								class="flex h-9 w-9 items-center justify-center rounded-md border border-stone-800 text-stone-400 transition-colors hover:border-stone-700 hover:bg-stone-900 hover:text-stone-100"
							>
								<svg class="h-4 w-4" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
									<path
										d="M21.94 4.72 19.6 19.87c-.17 1.19-.94 1.48-1.9.92l-5.26-3.88-2.54 2.44c-.28.28-.52.52-1.06.52l.38-5.4L19.06 5.5c.43-.38-.09-.6-.67-.22L6.9 12.81 1.7 11.18c-1.13-.35-1.15-1.13.24-1.67L20.5 3.1c.94-.35 1.76.22 1.44 1.62Z"
									/>
								</svg>
							</a>
							<a
								href="https://x.com"
								target="_blank"
								rel="noreferrer"
								aria-label="X"
								class="flex h-9 w-9 items-center justify-center rounded-md border border-stone-800 text-stone-400 transition-colors hover:border-stone-700 hover:bg-stone-900 hover:text-stone-100"
							>
								<svg class="h-4 w-4" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
									<path
										d="M18.24 2H21.5l-7.13 8.15L22.75 22h-6.56l-5.14-6.72L5.17 22H1.9l7.63-8.72L1.25 2h6.73l4.64 6.14L18.24 2Zm-1.14 18.06h1.81L7.02 3.85H5.08l12.02 16.21Z"
									/>
								</svg>
							</a>
							<a
								href="https://linkedin.com"
								target="_blank"
								rel="noreferrer"
								aria-label="LinkedIn"
								class="flex h-9 w-9 items-center justify-center rounded-md border border-stone-800 text-stone-400 transition-colors hover:border-stone-700 hover:bg-stone-900 hover:text-stone-100"
							>
								<svg class="h-4 w-4" viewBox="0 0 24 24" fill="currentColor" aria-hidden="true">
									<path
										d="M20.5 2h-17A1.5 1.5 0 0 0 2 3.5v17A1.5 1.5 0 0 0 3.5 22h17a1.5 1.5 0 0 0 1.5-1.5v-17A1.5 1.5 0 0 0 20.5 2ZM8 19H5v-9h3ZM6.5 8.25A1.75 1.75 0 1 1 8.3 6.5 1.78 1.78 0 0 1 6.5 8.25ZM19 19h-3v-4.74c0-1.42-.6-1.93-1.38-1.93A1.74 1.74 0 0 0 13 14.19a.66.66 0 0 0 0 .14V19h-3v-9h2.9v1.3a3.11 3.11 0 0 1 2.7-1.4c1.55 0 3.36.86 3.36 3.66Z"
									/>
								</svg>
							</a>
						</div>
					</div>

					<div class="lg:col-span-2">
						<h3 class="mono text-[10.5px] tracking-[0.18em] text-stone-500 uppercase">Product</h3>
						<ul class="mt-6 space-y-3 text-[13.5px]">
							<li>
								<a href="#capabilities" class="text-stone-300 transition-colors hover:text-white"
									>Capabilities</a
								>
							</li>
							<li>
								<a href="#architecture" class="text-stone-300 transition-colors hover:text-white"
									>Architecture</a
								>
							</li>
							<li>
								<a href="#impact" class="text-stone-300 transition-colors hover:text-white"
									>Social impact</a
								>
							</li>
							<li>
								<a href="#download" class="text-stone-300 transition-colors hover:text-white"
									>Download</a
								>
							</li>
						</ul>
					</div>

					<div class="lg:col-span-2">
						<h3 class="mono text-[10.5px] tracking-[0.18em] text-stone-500 uppercase">Resources</h3>
						<ul class="mt-6 space-y-3 text-[13.5px]">
							<li>
								<a
									href="https://github.com"
									target="_blank"
									rel="noreferrer"
									class="text-stone-300 transition-colors hover:text-white">GitHub</a
								>
							</li>
							<li>
								<a href="#" class="text-stone-300 transition-colors hover:text-white">API Docs</a>
							</li>
							<li>
								<a href="#" class="text-stone-300 transition-colors hover:text-white">Changelog</a>
							</li>
							<li>
								<a
									href="https://creativecommons.org/licenses/by-nc/4.0/"
									target="_blank"
									rel="noreferrer"
									class="text-stone-300 transition-colors hover:text-white">Licence</a
								>
							</li>
						</ul>
					</div>

					<div class="lg:col-span-3">
						<h3 class="mono text-[10.5px] tracking-[0.18em] text-stone-500 uppercase">Contact</h3>
						<ul class="mt-6 space-y-4 text-[13.5px]">
							<li class="flex gap-3">
								<span
									class="mt-0.5 flex h-6 w-6 shrink-0 items-center justify-center rounded-md border border-stone-800 text-stone-500"
								>
									<svg
										class="h-3 w-3"
										viewBox="0 0 24 24"
										fill="none"
										stroke="currentColor"
										stroke-width="1.75"
										stroke-linecap="round"
										stroke-linejoin="round"
										aria-hidden="true"
									>
										<path d="M12 22s7-6.5 7-12a7 7 0 1 0-14 0c0 5.5 7 12 7 12Z" />
										<circle cx="12" cy="10" r="2.5" />
									</svg>
								</span>
								<span class="leading-[1.7] text-stone-500">
									Jl. Jend. Sudirman Kav. 52-53<br />
									SCBD, Jakarta 12190, Indonesia
								</span>
							</li>
							<li class="flex items-center gap-3">
								<span
									class="flex h-6 w-6 shrink-0 items-center justify-center rounded-md border border-stone-800 text-stone-500"
								>
									<svg
										class="h-3 w-3"
										viewBox="0 0 24 24"
										fill="none"
										stroke="currentColor"
										stroke-width="1.75"
										stroke-linecap="round"
										stroke-linejoin="round"
										aria-hidden="true"
									>
										<path
											d="M22 16.92v3a2 2 0 0 1-2.18 2 19.79 19.79 0 0 1-8.63-3.07 19.5 19.5 0 0 1-6-6A19.79 19.79 0 0 1 2.12 4.18 2 2 0 0 1 4.11 2h3a2 2 0 0 1 2 1.72c.13.96.36 1.9.7 2.81a2 2 0 0 1-.45 2.11L8.09 9.91a16 16 0 0 0 6 6l1.27-1.27a2 2 0 0 1 2.11-.45c.91.34 1.85.57 2.81.7A2 2 0 0 1 22 16.92Z"
										/>
									</svg>
								</span>
								<a
									href="tel:+6281234567890"
									class="text-stone-300 transition-colors hover:text-white">+62 812 3456 7890</a
								>
							</li>
							<li class="flex items-center gap-3">
								<span
									class="flex h-6 w-6 shrink-0 items-center justify-center rounded-md border border-stone-800 text-stone-500"
								>
									<svg
										class="h-3 w-3"
										viewBox="0 0 24 24"
										fill="none"
										stroke="currentColor"
										stroke-width="1.75"
										stroke-linecap="round"
										stroke-linejoin="round"
										aria-hidden="true"
									>
										<rect x="3" y="5" width="18" height="14" rx="2" />
										<path d="m3.5 7 8.5 6 8.5-6" />
									</svg>
								</span>
								<a
									href="mailto:hello@sentinelproject.id"
									class="text-stone-300 transition-colors hover:text-white"
									>hello@sentinelproject.id</a
								>
							</li>
						</ul>
					</div>
				</div>
			</div>

			<div data-footer-card class="mt-3 rounded-3xl bg-[#0A0A0A] px-8 py-5 sm:px-12 lg:px-16">
				<div class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
					<p class="text-[12px] text-stone-500">
						Copyright © 2026 PT. Sentinel Project Indonesia. All rights reserved.
					</p>
					<div class="flex flex-wrap items-center gap-x-6 gap-y-2">
						<a href="#" class="text-[12px] text-stone-500 transition-colors hover:text-white"
							>Terms of Service</a
						>
						<a href="#" class="text-[12px] text-stone-500 transition-colors hover:text-white"
							>Privacy Statement</a
						>
						<a
							href="https://creativecommons.org/licenses/by-nc/4.0/"
							target="_blank"
							rel="noreferrer"
							class="text-[12px] text-stone-500 transition-colors hover:text-white">CC BY-NC 4.0</a
						>
					</div>
				</div>
			</div>
		</div>
	</footer>
</div>

<style>
	/* scroll-behavior: smooth DIHAPUS — Lenis yang handle scroll halus */
	:global(body) {
		font-family:
			'Poppins',
			'Archivo',
			ui-sans-serif,
			system-ui,
			-apple-system,
			sans-serif;
		-webkit-font-smoothing: antialiased;
		text-rendering: optimizeLegibility;
		background-color: #fafaf9;
	}
	:global(.mono) {
		font-family: 'IBM Plex Mono', ui-monospace, SFMono-Regular, Menlo, monospace;
		font-feature-settings: 'tnum' 1;
	}

	/* Custom cursor only on devices with a real pointer */
	.cursor-dot {
		display: none;
	}
	@media (hover: hover) and (pointer: fine) {
		.cursor-dot {
			display: block;
		}
	}

	/* Marquee ticker */
	.marquee {
		animation: marquee 42s linear infinite;
	}
	@keyframes marquee {
		from {
			transform: translate3d(0, 0, 0);
		}
		to {
			transform: translate3d(-50%, 0, 0);
		}
	}
	@media (prefers-reduced-motion: reduce) {
		.marquee {
			animation: none;
		}
	}
</style>
