<script>
  import { afterUpdate, onDestroy, onMount, tick } from 'svelte';
  import ApiCard from './components/ApiCard.svelte';
  import Sidebar from './components/Sidebar.svelte';

  const CONFIG_URL = 'https://cdn.ferdev.my.id/assets/configure.json';
  const BASE_URL = 'https://api.ferdev.my.id';
  const MIN_SPLASH_MS = 1800;

  let status = 'loading';
  let errorMessage = '';
  let categories = [];
  let searchTerm = '';
  let activeCat = 'hero';
  let sidebarOpen = false;
  let sidebarCollapsed = false;
  let showSplash = true;
  let splashHiding = false;
  let showBackTop = false;
  let toastMessage = '';
  let toastVisible = false;
  let themeWipeActive = false;

  let toastTimer;
  let splashTimer;
  let splashRemoveTimer;
  let scrollSpyObserver;
  let scrollSpyTimer;

  const slugify = (input) => input.toLowerCase().replace(/\s+/g, '-').replace(/[^a-z0-9-]/g, '');

  const isValidConfig = (payload) =>
    Array.isArray(payload) &&
    payload.every(
      (category) =>
        category &&
        typeof category.name === 'string' &&
        Array.isArray(category.apis)
    );

  const isLightPreferred = () => window.matchMedia('(prefers-color-scheme: light)').matches;
  const isMobileViewport = () => window.innerWidth <= 768;

  function applyInitialTheme() {
    const savedTheme = localStorage.getItem('theme');
    if (savedTheme) {
      document.documentElement.dataset.theme = savedTheme;
      return;
    }

    if (isLightPreferred()) {
      document.documentElement.dataset.theme = 'light';
    }
  }

  function toggleTheme() {
    if (themeWipeActive) return;

    const html = document.documentElement;
    const nextTheme = html.dataset.theme === 'dark' ? 'light' : 'dark';
    const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

    if (prefersReducedMotion || typeof document.startViewTransition !== 'function') {
      html.dataset.theme = nextTheme;
      localStorage.setItem('theme', nextTheme);
      return;
    }

    themeWipeActive = true;
    html.classList.add('theme-transition');

    const transition = document.startViewTransition(() => {
      html.dataset.theme = nextTheme;
      localStorage.setItem('theme', nextTheme);
    });

    transition.finished.finally(() => {
      html.classList.remove('theme-transition');
      themeWipeActive = false;
    });
  }

  function toggleSidebar() {
    if (isMobileViewport()) {
      sidebarOpen = !sidebarOpen;
      return;
    }

    sidebarCollapsed = !sidebarCollapsed;
  }

  function closeSidebar() {
    sidebarOpen = false;
  }

  function showToast(message) {
    toastMessage = message;
    toastVisible = true;

    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => {
      toastVisible = false;
    }, 2500);
  }

  function onNavigate(event) {
    const { slug } = event.detail;
    activeCat = slug;
    closeSidebar();

    const section = document.getElementById(slug);
    if (section) {
      section.scrollIntoView({ behavior: 'smooth', block: 'start' });
    }
  }

  function startSplashExit(delayMs) {
    clearTimeout(splashTimer);
    clearTimeout(splashRemoveTimer);

    splashTimer = setTimeout(() => {
      splashHiding = true;
      splashRemoveTimer = setTimeout(() => {
        showSplash = false;
      }, 700);
    }, Math.max(0, delayMs));
  }

  async function fetchConfig() {
    const response = await fetch(CONFIG_URL, { cache: 'no-store' });
    if (!response.ok) {
      throw new Error(`Failed to fetch config (${response.status})`);
    }

    const payload = await response.json();
    if (!isValidConfig(payload)) {
      throw new Error('Invalid configure.json format');
    }

    return payload;
  }

  async function loadData() {
    status = 'loading';
    errorMessage = '';
    splashHiding = false;
    showSplash = true;

    const startedAt = Date.now();

    try {
      const payload = await fetchConfig();
      categories = payload;
      status = 'ready';
      activeCat = 'hero';

      await tick();
      scheduleScrollSpy();

      const elapsed = Date.now() - startedAt;
      startSplashExit(MIN_SPLASH_MS - elapsed);
    } catch (error) {
      status = 'error';
      errorMessage = error?.message || 'Failed to load API documentation data.';

      const elapsed = Date.now() - startedAt;
      startSplashExit(MIN_SPLASH_MS - elapsed);
    }
  }

  function handleScroll() {
    requestAnimationFrame(() => {
      showBackTop = window.scrollY > 400;
    });
  }

  function initScrollSpy() {
    if (scrollSpyObserver) {
      scrollSpyObserver.disconnect();
    }

    const sections = document.querySelectorAll('.section-anchor');
    if (!sections.length) return;

    scrollSpyObserver = new IntersectionObserver(
      (entries) => {
        for (const entry of entries) {
          if (entry.isIntersecting) {
            activeCat = entry.target.id;
          }
        }
      },
      { rootMargin: '-30% 0px -60% 0px' }
    );

    for (const section of sections) {
      scrollSpyObserver.observe(section);
    }
  }

  function scheduleScrollSpy() {
    clearTimeout(scrollSpyTimer);
    scrollSpyTimer = setTimeout(() => {
      initScrollSpy();
    }, 120);
  }

  function initCodeTabs() {
    const tabGroups = document.querySelectorAll('.code-examples');
    
    tabGroups.forEach(group => {
      const tabs = group.querySelectorAll('.code-tab');
      const contents = group.querySelectorAll('.code-content');
      
      tabs.forEach(tab => {
        tab.addEventListener('click', () => {
          const lang = tab.dataset.lang;
          
          tabs.forEach(t => t.classList.remove('active'));
          contents.forEach(c => c.classList.remove('active'));
          
          tab.classList.add('active');
          const targetContent = group.querySelector(`.code-content[data-lang="${lang}"]`);
          if (targetContent) {
            targetContent.classList.add('active');
          }
        });
      });
    });
  }

  onMount(() => {
    applyInitialTheme();
    loadData();
    handleScroll();
    initCodeTabs();
    window.addEventListener('scroll', handleScroll);
    window.addEventListener('resize', closeSidebar);

    return () => {
      window.removeEventListener('scroll', handleScroll);
      window.removeEventListener('resize', closeSidebar);
    };
  });

  afterUpdate(() => {
    if (status === 'ready') {
      scheduleScrollSpy();
    }
  });

  onDestroy(() => {
    clearTimeout(toastTimer);
    clearTimeout(splashTimer);
    clearTimeout(splashRemoveTimer);
    clearTimeout(scrollSpyTimer);

    if (scrollSpyObserver) {
      scrollSpyObserver.disconnect();
    }
  });

  $: categoriesWithApis = categories
    .map((category) => ({
      ...category,
      slug: slugify(category.name),
      apis: Array.isArray(category.apis) 
        ? category.apis.filter((api) => api.isAktif !== false)
        : []
    }))
    .filter((category) => category.apis.length > 0);

  $: allApis = categoriesWithApis.flatMap((category) => category.apis);

  $: stats = [
    { n: allApis.length, l: 'Total Endpoints' },
    { n: categoriesWithApis.length, l: 'Categories' },
    {
      n: allApis.filter((api) => String(api.method || '').toUpperCase() === 'GET').length,
      l: 'GET Requests'
    },
    {
      n: allApis.filter((api) => String(api.method || '').toUpperCase() === 'POST').length,
      l: 'POST Requests'
    }
  ];

  $: normalizedSearch = searchTerm.trim().toLowerCase();

  $: filteredSections = categoriesWithApis
    .map((category) => {
      const filteredApis = normalizedSearch
        ? category.apis.filter((api) => {
            const title = String(api.title || '').toLowerCase();
            const endpoint = String(api.endpointPath || '').toLowerCase();
            const description = String(api.description || '').toLowerCase();
            return (
              title.includes(normalizedSearch) ||
              endpoint.includes(normalizedSearch) ||
              description.includes(normalizedSearch)
            );
          })
        : category.apis;

      return {
        ...category,
        apis: filteredApis
      };
    })
    .filter((category) => category.apis.length > 0);
</script>

<div class="dot-pattern"></div>

{#if showSplash}
  <div id="splash" class:hide={splashHiding}>
    <div class="splash-logo">
      <span>F</span><span>e</span><span>r</span><span class="accent">D</span><span>e</span><span>v</span>
    </div>
    <div class="splash-bar">
      <div class="splash-progress"></div>
    </div>
    <div class="splash-sub">API Documentation</div>
  </div>
{/if}

<div class="app-root" class:visible={!showSplash}>
  {#if status === 'error'}
    <div class="error-state">
      <div class="error-card">
        <div class="error-title">Failed to Load Documentation</div>
        <div class="error-desc">{errorMessage}</div>
        <button class="btn btn-primary" type="button" on:click={loadData}>
          <i class="fa-solid fa-rotate-right"></i>Retry
        </button>
      </div>
    </div>
  {:else}
    <div
      class="overlay"
      class:show={sidebarOpen}
      id="overlay"
      role="button"
      tabindex="0"
      on:click={closeSidebar}
      on:keydown={(event) => (event.key === 'Enter' || event.key === ' ') && closeSidebar()}
    ></div>

    <header class="header" class:sidebar-collapsed={sidebarCollapsed}>
      <div class="header-brand">
        <button class="mobile-menu-btn" id="menuBtn" on:click={toggleSidebar}>
          <i class="fa-solid {sidebarOpen || !sidebarCollapsed ? 'fa-xmark' : 'fa-bars'}"></i>
        </button>
        <a href="#hero" class="logo" on:click|preventDefault={() => onNavigate({ detail: { slug: 'hero' } })}
          ><img src="https://cdn.ferdev.my.id/assets/img/brand_image.png" alt="FerDev" class="logo-img" /></a
        >
      </div>
      <div class="header-left">
        <div class="search-box">
          <i class="fa-solid fa-magnifying-glass"></i>
          <input
            type="text"
            id="globalSearch"
            placeholder="Search endpoints..."
            autocomplete="off"
            bind:value={searchTerm}
          />
        </div>
      </div>
      <div class="header-right">
        <a href="https://api.ferdev.my.id/register" target="_blank" class="icon-btn" title="Get API Key">
          <i class="fa-solid fa-key"></i>
        </a>
        <button class="theme-toggle" id="themeBtn" on:click={toggleTheme}>
          <i class="fa-solid fa-sun icon-sun"></i>
          <i class="fa-solid fa-moon icon-moon"></i>
        </button>
      </div>
    </header>

    <Sidebar
      categories={categoriesWithApis}
      activeCat={activeCat}
      open={sidebarOpen}
      collapsed={sidebarCollapsed}
      on:navigate={onNavigate}
    />

    <main class="main" class:sidebar-collapsed={sidebarCollapsed}>
      <div class="hero section-anchor" id="hero">
        <div class="hero-label">&#9632; REST API Documentation</div>
        <div class="hero-title">FERDEV <span>API</span></div>
        <div class="hero-desc">
          Koleksi REST API yang lengkap. AI, Downloader, Search, Tools, dan banyak lagi. Semua dalam satu
          platform.
        </div>
        <div class="hero-stats" id="heroStats">
          {#each stats as stat}
            <div class="stat">
              <div class="stat-num">{stat.n}</div>
              <div class="stat-label">{stat.l}</div>
            </div>
          {/each}
        </div>
      </div>

      <div class="usage-guide">
        <div class="usage-header">
          <i class="fa-solid fa-book"></i>
          <h2>Cara Penggunaan API</h2>
        </div>
        <p class="usage-intro">
          FerDev API mendukung metode <strong>GET</strong> dan <strong>POST</strong>. Berikut adalah panduan lengkap untuk menggunakan kedua metode tersebut.
        </p>

        <div class="method-section">
          <div class="method-header">
            <span class="method-badge method-get">GET</span>
            <h3>GET Request</h3>
          </div>
          <p class="method-desc">Untuk GET request, parameter dikirim melalui query string di URL.</p>
          
          <div class="code-examples">
            <div class="code-tabs">
              <button class="code-tab active" data-lang="curl">cURL</button>
              <button class="code-tab" data-lang="javascript">JavaScript</button>
              <button class="code-tab" data-lang="php">PHP</button>
              <button class="code-tab" data-lang="python">Python</button>
            </div>
            
            <div class="code-content active" data-lang="curl">
              <pre><code>curl -X GET "https://api.ferdev.my.id/search/pinterest?query=nature&apikey=YOUR_API_KEY"</code></pre>
            </div>
            
            <div class="code-content" data-lang="javascript">
              <pre><code>{`// Menggunakan Fetch API
fetch('https://api.ferdev.my.id/search/pinterest?query=nature&apikey=YOUR_API_KEY')
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error('Error:', error));

// Menggunakan Axios
const axios = require('axios');
axios.get('https://api.ferdev.my.id/search/pinterest', {
  params: {
    query: 'nature',
    apikey: 'YOUR_API_KEY'
  }
})
.then(response => console.log(response.data))
.catch(error => console.error('Error:', error));`}</code></pre>
            </div>
            
            <div class="code-content" data-lang="php">
              <pre><code>{`<?php
// Menggunakan cURL
$url = 'https://api.ferdev.my.id/search/pinterest?query=nature&apikey=YOUR_API_KEY';
$ch = curl_init($url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = curl_exec($ch);
curl_close($ch);
$data = json_decode($response, true);
print_r($data);

// Menggunakan file_get_contents
$url = 'https://api.ferdev.my.id/search/pinterest?query=nature&apikey=YOUR_API_KEY';
$response = file_get_contents($url);
$data = json_decode($response, true);
print_r($data);
?>`}</code></pre>
            </div>
            
            <div class="code-content" data-lang="python">
              <pre><code>{`# Menggunakan requests
import requests

url = 'https://api.ferdev.my.id/search/pinterest'
params = {
    'query': 'nature',
    'apikey': 'YOUR_API_KEY'
}
response = requests.get(url, params=params)
data = response.json()
print(data)`}</code></pre>
            </div>
          </div>
        </div>

        <div class="method-section">
          <div class="method-header">
            <span class="method-badge method-post">POST</span>
            <h3>POST Request</h3>
          </div>
          <p class="method-desc">Untuk POST request, API key dikirim melalui header Authorization, dan parameter dikirim dalam body request sebagai JSON.</p>
          
          <div class="code-examples">
            <div class="code-tabs">
              <button class="code-tab active" data-lang="curl-post">cURL</button>
              <button class="code-tab" data-lang="javascript-post">JavaScript</button>
              <button class="code-tab" data-lang="php-post">PHP</button>
              <button class="code-tab" data-lang="python-post">Python</button>
            </div>
            
            <div class="code-content active" data-lang="curl-post">
              <pre><code>{`curl -X POST "https://api.ferdev.my.id/ai/chatgpt" \\
  -H "Content-Type: application/json" \\
  -H "Authorization: Bearer YOUR_API_KEY" \\
  -d '{"text": "Hello"}'`}</code></pre>
            </div>
            
            <div class="code-content" data-lang="javascript-post">
              <pre><code>{`// Menggunakan Fetch API
fetch('https://api.ferdev.my.id/ai/chatgpt', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_API_KEY'
  },
  body: JSON.stringify({
    text: 'Hello'
  })
})
.then(response => response.json())
.then(data => console.log(data))
.catch(error => console.error('Error:', error));

// Menggunakan Axios
const axios = require('axios');
axios.post('https://api.ferdev.my.id/ai/chatgpt', 
  { text: 'Hello' },
  {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer YOUR_API_KEY'
    }
  }
)
.then(response => console.log(response.data))
.catch(error => console.error('Error:', error));`}</code></pre>
            </div>
            
            <div class="code-content" data-lang="php-post">
              <pre><code>{`<?php
// Menggunakan cURL
$url = 'https://api.ferdev.my.id/ai/chatgpt';
$data = json_encode(['text' => 'Hello']);

$ch = curl_init($url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Content-Type: application/json',
    'Authorization: Bearer YOUR_API_KEY'
]);

$response = curl_exec($ch);
curl_close($ch);
$result = json_decode($response, true);
print_r($result);

// Menggunakan file_get_contents
$url = 'https://api.ferdev.my.id/ai/chatgpt';
$data = json_encode(['text' => 'Hello']);

$options = [
    'http' => [
        'method' => 'POST',
        'header' => "Content-Type: application/json\\r\\n" .
                    "Authorization: Bearer YOUR_API_KEY\\r\\n",
        'content' => $data
    ]
];

$context = stream_context_create($options);
$response = file_get_contents($url, false, $context);
$result = json_decode($response, true);
print_r($result);
?>`}</code></pre>
            </div>
            
            <div class="code-content" data-lang="python-post">
              <pre><code>{`# Menggunakan requests
import requests
import json

url = 'https://api.ferdev.my.id/ai/chatgpt'
headers = {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer YOUR_API_KEY'
}
data = {
    'text': 'Hello'
}

response = requests.post(url, headers=headers, json=data)
result = response.json()
print(result)`}</code></pre>
            </div>
          </div>
        </div>

        <div class="usage-notes">
          <div class="note-card">
            <i class="fa-solid fa-key"></i>
            <div>
              <strong>API Key</strong>
              <p>Dapatkan API key gratis di <a href="https://api.ferdev.my.id/register" target="_blank">api.ferdev.my.id/register</a></p>
            </div>
          </div>
          <div class="note-card">
            <i class="fa-solid fa-shield-halved"></i>
            <div>
              <strong>Rate Limiting</strong>
              <p>Setiap API key memiliki batas request. Pastikan untuk menggunakan API secara bijak.</p>
            </div>
          </div>
          <div class="note-card">
            <i class="fa-solid fa-circle-info"></i>
            <div>
              <strong>Response Format</strong>
              <p>Semua response dikembalikan dalam format JSON, kecuali untuk endpoint yang mengembalikan gambar.</p>
            </div>
          </div>
        </div>
      </div>

      <div id="content">
        {#if filteredSections.length === 0 && status === 'ready'}
          <div class="no-results">
            <i class="fa-solid fa-magnifying-glass"></i>
            <p>Tidak ada hasil untuk "<strong>{normalizedSearch}</strong>"</p>
          </div>
        {:else}
          {#each filteredSections as category}
            <div class="cat-section section-anchor" id={category.slug}>
              <div class="section-header">
                <div class="section-header-left">
                  <div class="section-icon"><i class={category.icon}></i></div>
                  <div class="section-title">{category.name}</div>
                  <div class="section-desc">{category.apis.length} endpoint tersedia</div>
                </div>
                <div class="section-badge">{category.apis.length} APIs</div>
              </div>

              <div class="api-grid">
                {#each category.apis as api, apiIndex}
                  <ApiCard {api} index={apiIndex} baseUrl={BASE_URL} on:toast={(event) => showToast(event.detail.message)} />
                {/each}
              </div>
            </div>
          {/each}
        {/if}
      </div>
    </main>

    <div class="toast" class:show={toastVisible} id="toast">
      <i class="fa-solid fa-check"></i><span id="toastMsg">{toastMessage}</span>
    </div>
    <button class="back-top" class:show={showBackTop} id="backTop" on:click={() => window.scrollTo({ top: 0, behavior: 'smooth' })}>
      <i class="fa-solid fa-arrow-up"></i>
    </button>
  {/if}
</div>
