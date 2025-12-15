<script lang="ts">
  // IntersectionObserver options the Svelte action understands
  type IntersectionParams = {
    threshold?: number | number[];
    root?: Element | null;
    rootMargin?: string;
    once?: boolean;
    onEnter?: (entry: IntersectionObserverEntry) => void;
    onLeave?: (entry: IntersectionObserverEntry) => void;
  };

  // Svelte 5-friendly action: rebuilds the observer when options change
  function intersect(node: HTMLElement, params: IntersectionParams = {}) {
    let opts = {
      root: params.root ?? null,
      rootMargin: params.rootMargin ?? '0px',
      threshold: params.threshold ?? 0.25
    } satisfies IntersectionObserverInit;

    let current = { ...params } satisfies IntersectionParams;
    let observer: IntersectionObserver;

    const init = () => {
      observer?.disconnect();
      observer = new IntersectionObserver((entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            current.onEnter?.(entry);
            if (current.once) observer.unobserve(entry.target);
          } else {
            current.onLeave?.(entry);
          }
        });
      }, opts);
      observer.observe(node);
    };

    // create observer immediately
    init();

    return {
      update(next: IntersectionParams) {
        // merge new params and recreate the observer with fresh options
        current = { ...current, ...next } satisfies IntersectionParams;
        opts = {
          root: next.root ?? opts.root ?? null,
          rootMargin: next.rootMargin ?? opts.rootMargin ?? '0px',
          threshold: next.threshold ?? opts.threshold
        } satisfies IntersectionObserverInit;
        init();
      },
      destroy() {
        observer?.disconnect();
      }
    };
  }

  type LogEntry = {
    id: string;
    ratio: number;
    direction: 'enter' | 'leave';
    time: string;
  };

  const navSections = [
    { id: 'overview', label: 'Overview' },
    { id: 'lazy', label: 'Lazy Images' },
    { id: 'reveal', label: 'Reveal' },
    { id: 'spy', label: 'Scroll Spy' },
    { id: 'log', label: 'Live Log' }
  ] as const;

  type SectionId = (typeof navSections)[number]['id'];

  const gallery = [
    {
      id: 'aurora',
      title: 'Aurora Cache',
      description: 'Preload offscreen assets with rootMargin and a single observer.',
      blur:
        'https://images.unsplash.com/photo-1444703686981-a3abbc4d4fe3?auto=format&fit=crop&w=40&q=20&blur=40',
      full:
        'https://images.unsplash.com/photo-1444703686981-a3abbc4d4fe3?auto=format&fit=crop&w=1400&q=80'
    },
    {
      id: 'desert',
      title: 'Desert Streams',
      description: 'Observe many targets at once; IO batches them for you.',
      blur:
        'https://images.unsplash.com/photo-1500534314209-a25ddb2bd429?auto=format&fit=crop&w=40&q=20&blur=40',
      full:
        'https://images.unsplash.com/photo-1500534314209-a25ddb2bd429?auto=format&fit=crop&w=1400&q=80'
    },
    {
      id: 'city',
      title: 'City Highlights',
      description: 'Use threshold arrays to stagger when UI states flip.',
      blur:
        'https://images.unsplash.com/photo-1444084316824-dc26d6657664?auto=format&fit=crop&w=40&q=20&blur=40',
      full:
        'https://images.unsplash.com/photo-1444084316824-dc26d6657664?auto=format&fit=crop&w=1400&q=80'
    },
    {
      id: 'forest',
      title: 'Forest Signals',
      description: 'Once-only observations keep observers lean over time.',
      blur:
        'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=40&q=20&blur=40',
      full:
        'https://images.unsplash.com/photo-1500530855697-b586d89ba3ee?auto=format&fit=crop&w=1400&q=80'
    }
  ];

  const revealBlocks = [
    {
      id: 'io-batching',
      title: 'Batched callbacks',
      detail: 'The browser coalesces visibility changes; your callback sees a pack of entries.'
    },
    {
      id: 'precision',
      title: 'Fine-grained thresholds',
      detail: 'Set arrays like [0, 0.25, 0.5, 0.75, 1] to animate in multiple phases.'
    },
    {
      id: 'off-main',
      title: 'Layout-friendly',
      detail: 'No layout thrash from scroll listeners; IO runs outside the hot path.'
    },
    {
      id: 'cleanup',
      title: 'Cleanup discipline',
      detail: 'Unobserve targets after first intersect to free memory and work.'
    }
  ];

  const spySections = [
    {
      id: 'use-case-1',
      heading: 'Lazy Load',
      body: 'Use rootMargin to preload before the fold and avoid pop-in.'
    },
    {
      id: 'use-case-2',
      heading: 'Scroll Animations',
      body: 'Drive CSS states only; let Tailwind transitions do the heavy lifting.'
    },
    {
      id: 'use-case-3',
      heading: 'Scroll Spy',
      body: 'Observe sections at 50% threshold to keep nav aligned with the viewport.'
    }
  ];

  // Snippet we display in the UI
  const actionSnippet = `const action = (node, options) => {
  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          options.onEnter?.(entry);
          if (options.once) observer.unobserve(entry.target);
        } else options.onLeave?.(entry);
      });
    },
    options
  );
  observer.observe(node);
  return { destroy: () => observer.disconnect() };
};`;



const observer = new IntersectionObserver((entries) => {
  for (const entry of entries) {
    if (!entry.isIntersecting) continue;
    const img = entry.target;
    img.src = img.dataset.src;
    observer.unobserve(img);
  }
}, {
  root: null,
  rootMargin: "200px 0px",
  threshold: 0
});




  // Svelte 5 runes: wrap mutable values in $state so reassignments propagate
  let activeSection = $state<SectionId>('overview');
  let revealThreshold = $state(0.35);
  let imageSrc = $state<Record<string, string>>(Object.fromEntries(gallery.map((item) => [item.id, item.blur])));
  let loadedImages = $state(new Set<string>());
  let revealed = $state(new Set<string>());
  let logs = $state<LogEntry[]>([]);

  // Record an intersection event for the telemetry list
  const note = (entry: IntersectionObserverEntry, id: string) => {
    logs = [
      {
        id,
        ratio: Math.round(entry.intersectionRatio * 100) / 100,
        direction: (entry.isIntersecting ? 'enter' : 'leave') as 'enter' | 'leave',
        time: new Intl.DateTimeFormat('de-DE', {
          hour: '2-digit',
          minute: '2-digit',
          second: '2-digit'
        }).format(new Date())
      },
      ...logs
    ].slice(0, 10);
  };

  // Lazy load the high-res image and log once it appears
  const loadFull = (item: (typeof gallery)[number], entry?: IntersectionObserverEntry) => {
    if (imageSrc[item.id] === item.full) return;
    const img = new Image();
    img.src = item.full;
    img.onload = () => {
      imageSrc = { ...imageSrc, [item.id]: item.full };
      loadedImages = new Set(loadedImages).add(item.id);
      if (entry) note(entry, item.id);
    };
  };

  // Mark a reveal block as seen and add to telemetry
  const markRevealed = (id: string, entry?: IntersectionObserverEntry) => {
    if (!revealed.has(id)) {
      revealed = new Set(revealed).add(id);
      if (entry) note(entry, id);
    }
  };
</script>

<svelte:head>
  <title>IntersectionObserver Lab</title>
  <meta
    name="description"
    content="A multi-feature IntersectionObserver demo with lazy images, reveal effects, scroll spy, and live logs."
  />
</svelte:head>

<main class="min-h-screen bg-slate-950 text-slate-100">
  <div class="relative isolate overflow-hidden">
    <!-- Soft gradient backdrop to give depth without affecting hit-testing -->
    <div class="pointer-events-none absolute inset-0 bg-[radial-gradient(circle_at_20%_20%,rgba(52,211,153,0.12),transparent_35%),radial-gradient(circle_at_80%_0%,rgba(59,130,246,0.12),transparent_30%),radial-gradient(circle_at_50%_70%,rgba(236,72,153,0.12),transparent_30%)]"></div>

    <!-- Sticky header + nav to visualize scroll-spy updates -->
    <header class="sticky top-0 z-20 backdrop-blur bg-slate-950/70 border-b border-slate-800/50">
      <div class="mx-auto flex max-w-6xl items-center justify-between px-6 py-3">
        <div class="flex items-center gap-3 text-sm font-semibold tracking-tight">
          <span class="flex h-9 w-9 items-center justify-center rounded-xl bg-emerald-500/10 text-emerald-300 shadow-inner shadow-emerald-500/30">IO</span>
          <div>
            <p class="text-xs uppercase text-slate-400">IntersectionObserver</p>
            <p class="text-sm text-slate-100">Advanced demo</p>
          </div>
        </div>
        <nav class="hidden items-center gap-2 text-xs font-medium md:flex">
          {#each navSections as item}
            <a
              href={`#${item.id}`}
              class={`rounded-full px-3 py-1 transition hover:bg-slate-800/80 hover:text-white ${
                activeSection === item.id
                  ? 'bg-slate-800 text-emerald-300 shadow-sm shadow-emerald-500/20'
                  : 'text-slate-300/80'
              }`}
            >
              {item.label}
            </a>
          {/each}
        </nav>
      </div>
    </header>

    <!-- Overview hero: explains the reusable action that powers the rest -->
    <section
      id="overview"
      use:intersect={{ threshold: 0.6, onEnter: () => (activeSection = 'overview') }}
      class="mx-auto max-w-6xl px-6 py-14 md:py-18"
    >
      <div class="grid gap-10 md:grid-cols-[1.1fr,0.9fr] md:items-center">
        <div class="space-y-5">
          <p class="inline-flex items-center gap-2 rounded-full bg-slate-900/70 px-3 py-1 text-xs text-emerald-200 ring-1 ring-emerald-500/30">
            <span class="h-2 w-2 rounded-full bg-emerald-400"></span>
            Drei Beobachter, ein Toolkit
          </p>
          <h1 class="text-3xl font-semibold leading-tight text-white md:text-4xl">
            IntersectionObserver Lab: Lazy Loading, Scroll-FX & Live Telemetrie
          </h1>
          <p class="text-base text-slate-300">
            Wir nutzen drei voneinander unabhaengige Observer: Bilder laden mit rootMargin, Karten animieren
            mit dynamischem Threshold und ein Scroll-Spy, das die Navigation synchron haelt. Live-Logs zeigen,
            wann und mit welcher Schnittmenge Events feuern.
          </p>
          <div class="grid gap-4 sm:grid-cols-2">
            <div class="rounded-2xl border border-slate-800 bg-slate-900/60 p-4 shadow-lg shadow-slate-900/80">
              <p class="text-sm font-semibold text-white">Ein Observer, viele Ziele</p>
              <p class="text-sm text-slate-300">Keine Scroll-Listener, keine Debounce-Hacks. Der Browser buendelt Aenderungen.</p>
            </div>
            <div class="rounded-2xl border border-slate-800 bg-slate-900/60 p-4 shadow-lg shadow-slate-900/80">
              <p class="text-sm font-semibold text-white">Cleanup & Optionen</p>
              <p class="text-sm text-slate-300">Threshold-Tuning, rootMargin-Preload, einmalige Beobachtung pro Element.</p>
            </div>
          </div>
        </div>
        <div class="rounded-3xl border border-slate-800 bg-slate-900/70 p-6 shadow-2xl shadow-slate-900/80">
          <p class="mb-2 text-sm font-semibold text-emerald-300">Reusable Svelte action</p>
          <pre class="overflow-auto rounded-2xl bg-slate-950/70 p-4 text-xs text-slate-200 ring-1 ring-slate-800"><code>{actionSnippet}</code></pre>
          <p class="mt-3 text-xs text-slate-400">Das gesamte Beispiel nutzt genau diese Action an verschiedenen Stellen.</p>
        </div>
      </div>
    </section>

    <!-- Lazy-loading grid: single observer instance handles all cards -->
    <section
      id="lazy"
      use:intersect={{ threshold: 0.5, onEnter: () => (activeSection = 'lazy') }}
      class="mx-auto max-w-6xl px-6 py-14"
    >
      <div class="mb-6 flex items-center justify-between gap-4">
        <div>
          <p class="text-sm font-semibold text-emerald-300">Observer 1 - Lazy Loading</p>
          <h2 class="text-2xl font-semibold text-white">rootMargin preloads, once detaches</h2>
          <p class="text-sm text-slate-300">Alle Karten teilen sich denselben Observer. Sobald geladen, wird das Ziel aus der Beobachtung entfernt.</p>
        </div>
        <div class="hidden text-xs text-slate-400 md:block">rootMargin: 0px 0px 240px 0px - threshold: 0.15</div>
      </div>

      <div class="grid gap-6 md:grid-cols-2 xl:grid-cols-3">
        {#each gallery as shot}
          <article
            class="group overflow-hidden rounded-3xl border border-slate-800/80 bg-slate-900/60 shadow-xl shadow-slate-900/70 transition hover:border-emerald-500/30 hover:shadow-emerald-500/10"
            use:intersect={{
              threshold: 0.15,
              rootMargin: '0px 0px 240px 0px',
              once: true,
              onEnter: (entry) => loadFull(shot, entry)
            }}
          >
            <div class="relative h-56 overflow-hidden">
              <img
                src={imageSrc[shot.id]}
                alt={shot.title}
                class={`h-full w-full object-cover transition duration-700 ease-out ${
                  loadedImages.has(shot.id)
                    ? 'scale-100 opacity-100 blur-0'
                    : 'scale-105 opacity-40 blur-sm'
                }`}
                loading="lazy"
              />
              <div class="pointer-events-none absolute inset-0 bg-gradient-to-t from-slate-950/60 via-transparent to-transparent"></div>
              <div class="absolute left-4 top-4 flex items-center gap-2 text-[11px] font-medium text-slate-200">
                <span class="rounded-full bg-slate-950/70 px-3 py-1 ring-1 ring-slate-800">rootMargin +240px</span>
                <span
                  class={`rounded-full px-3 py-1 ring-1 ${
                    loadedImages.has(shot.id)
                      ? 'bg-emerald-500/20 text-emerald-200 ring-emerald-500/40'
                      : 'bg-amber-500/10 text-amber-200 ring-amber-500/30'
                  }`}
                >
                  {loadedImages.has(shot.id) ? 'geladen' : 'wartet'}
                </span>
              </div>
            </div>
            <div class="space-y-1 p-4">
              <p class="text-lg font-semibold text-white">{shot.title}</p>
              <p class="text-sm text-slate-300">{shot.description}</p>
            </div>
          </article>
        {/each}
      </div>
    </section>

    <!-- Reveal cards re-run the action when threshold slider moves -->
    <section
      id="reveal"
      use:intersect={{ threshold: 0.55, onEnter: () => (activeSection = 'reveal') }}
      class="mx-auto max-w-6xl px-6 py-14"
    >
      <div class="flex flex-col gap-4 md:flex-row md:items-center md:justify-between">
        <div>
          <p class="text-sm font-semibold text-sky-300">Observer 2 - Scroll Reveal</p>
          <h2 class="text-2xl font-semibold text-white">Dynamic threshold = spaeteres oder frueheres Triggern</h2>
          <p class="text-sm text-slate-300">Aendere den Threshold live; die Action wird neu initialisiert und verschiebt den Moment der Sichtbarkeit.</p>
        </div>
        <div class="w-full max-w-sm rounded-2xl border border-slate-800 bg-slate-900/60 p-4 shadow-lg shadow-slate-900/60">
          <label for="reveal-slider" class="flex items-center justify-between text-xs text-slate-300">
            <span>threshold</span>
            <span class="font-mono text-emerald-300">{revealThreshold.toFixed(2)}</span>
          </label>
          <input
            id="reveal-slider"
            type="range"
            min="0.05"
            max="0.8"
            step="0.05"
            value={revealThreshold}
            on:input={(e) => (revealThreshold = parseFloat(e.currentTarget.value))}
            class="mt-2 h-2 w-full cursor-pointer appearance-none rounded-full bg-slate-800 accent-emerald-400"
          />
          <p class="mt-2 text-[11px] text-slate-400">Kleine Werte: fruehes Erscheinen - Grosse Werte: spaeteres Erscheinen</p>
        </div>
      </div>

      <div class="mt-8 grid gap-4 md:grid-cols-2">
        {#each revealBlocks as block}
          <article
            class={`rounded-3xl border border-slate-800 bg-slate-900/60 p-5 transition duration-500 ease-out ${
              revealed.has(block.id)
                ? 'translate-y-0 border-emerald-500/30 bg-slate-900/80 shadow-xl shadow-emerald-500/10 opacity-100'
                : 'translate-y-6 opacity-0'
            }`}
            use:intersect={{
              threshold: revealThreshold,
              onEnter: (entry) => markRevealed(block.id, entry)
            }}
          >
            <p class="text-sm font-semibold text-white">{block.title}</p>
            <p class="text-sm text-slate-300">{block.detail}</p>
            <div class="mt-3 flex items-center gap-2 text-[11px] text-slate-400">
              <span class="rounded-full bg-slate-800/80 px-3 py-1">threshold {revealThreshold.toFixed(2)}</span>
              <span class="rounded-full bg-slate-800/80 px-3 py-1">unobserve: false</span>
            </div>
          </article>
        {/each}
      </div>
    </section>

    <!-- Scroll spy section: every article toggles activeSection via action -->
    <section
      id="spy"
      use:intersect={{ threshold: 0.55, onEnter: () => (activeSection = 'spy') }}
      class="mx-auto max-w-6xl px-6 py-14"
    >
      <div class="mb-8 flex items-center justify-between">
        <div>
          <p class="text-sm font-semibold text-fuchsia-300">Observer 3 - Scroll Spy</p>
          <h2 class="text-2xl font-semibold text-white">Sections beobachten, Navigation synchronisieren</h2>
          <p class="text-sm text-slate-300">Wir beobachten jede Section bei 50% Sichtbarkeit und togglen die aktive Markierung.</p>
        </div>
        <div class="hidden text-xs text-slate-400 md:block">threshold: 0.5 - once: false</div>
      </div>




      <div class="grid gap-6 lg:grid-cols-[1fr,0.55fr]">
        <div class="space-y-6">
          {#each spySections as section}
            <article
              id={section.id}
              class="rounded-3xl border border-slate-800 bg-slate-900/60 p-6 shadow-lg shadow-slate-900/70 transition"
              use:intersect={{
                threshold: 0.5,
                onEnter: (entry) => {
                  activeSection = 'spy';
                  note(entry, section.id);
                },
                onLeave: (entry) => note(entry, section.id)
              }}
            >
              <p class="text-sm font-semibold text-white">{section.heading}</p>
              <p class="text-sm text-slate-300">{section.body}</p>
            </article>
          {/each}
        </div>
        <div class="rounded-3xl border border-slate-800 bg-slate-900/70 p-5 shadow-xl shadow-slate-900/80">
          <p class="text-sm font-semibold text-white">Aktive Section</p>
          <div class="mt-3 flex flex-wrap gap-2 text-xs">
            {#each navSections as item}
              <span
                class={`rounded-full px-3 py-1 ring-1 transition ${
                  activeSection === item.id
                    ? 'bg-emerald-500/15 text-emerald-200 ring-emerald-500/40'
                    : 'bg-slate-900 text-slate-400 ring-slate-800'
                }`}
              >
                {item.label}
              </span>
            {/each}
          </div>
          <p class="mt-4 text-sm text-slate-300">Scroll und beobachte, wie der Zustand ohne manuelle Scroll-Events wechselt.</p>
        </div>
      </div>
    </section>

    <!-- Live log pulls the last 10 intersection events for debugging -->
    <section
      id="log"
      use:intersect={{ threshold: 0.4, onEnter: () => (activeSection = 'log') }}
      class="mx-auto max-w-6xl px-6 pb-16"
    >
      <div class="flex flex-col gap-4 md:flex-row md:items-center md:justify-between">
        <div>
          <p class="text-sm font-semibold text-amber-300">Telemetry</p>
          <h2 class="text-2xl font-semibold text-white">Live Intersection Logs</h2>
          <p class="text-sm text-slate-300">Letzte 10 Events mit ratio &amp; Richtung. Gut, um Thresholds zu verstehen.</p>
        </div>
        <div class="text-xs text-slate-400">Events werden geloggt, wenn ein beobachtetes Element enter/leave triggert.</div>
      </div>

      <div class="mt-6 overflow-hidden rounded-3xl border border-slate-800 bg-slate-900/70 shadow-xl shadow-slate-900/80">
        <div class="grid grid-cols-[1.2fr,0.6fr,0.7fr,0.9fr] bg-slate-900/60 px-4 py-2 text-[11px] font-semibold uppercase tracking-wide text-slate-400">
          <span>target</span>
          <span>ratio</span>
          <span>direction</span>
          <span>time</span>
        </div>
        {#if logs.length === 0}
          <p class="px-4 py-4 text-sm text-slate-400">Noch keine Eintraege - scrolle durch die Seite.</p>
        {:else}
          {#each logs as entry (entry.time + entry.id + entry.direction)}
            <div class="grid grid-cols-[1.2fr,0.6fr,0.7fr,0.9fr] border-t border-slate-800/80 px-4 py-2 text-sm text-slate-200">
              <span class="font-mono text-[12px]">{entry.id}</span>
              <span class="font-mono text-emerald-300">{entry.ratio.toFixed(2)}</span>
              <span class={`font-semibold ${entry.direction === 'enter' ? 'text-emerald-300' : 'text-slate-400'}`}>
                {entry.direction}
              </span>
              <span class="font-mono text-[12px] text-slate-300">{entry.time}</span>
            </div>
          {/each}
        {/if}
      </div>
    </section>
  </div>
</main>
