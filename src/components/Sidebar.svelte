<script>
  import { createEventDispatcher } from 'svelte';

  export let categories = [];
  export let activeCat = 'hero';
  export let open = false;
  export let collapsed = false;

  const dispatch = createEventDispatcher();

  function navigate(slug) {
    dispatch('navigate', { slug });
  }
</script>

<nav class="sidebar" class:open class:collapsed id="sidebar">
  <div class="sidebar-label">Navigation</div>
  <a
    href="#hero"
    class="sidebar-item"
    class:active={activeCat === 'hero'}
    data-cat="hero"
    on:click|preventDefault={() => navigate('hero')}
  >
    <i class="fa-solid fa-house"></i><span class="sidebar-text">Overview</span>
  </a>
  <div class="sidebar-divider"></div>
  <div class="sidebar-label">Categories</div>
  <div id="sidebarCats">
    {#each categories as cat}
      {@const slug = cat.slug}
      <a
        href={'#' + slug}
        class="sidebar-item"
        class:active={activeCat === slug}
        data-cat={slug}
        on:click|preventDefault={() => navigate(slug)}
      >
        <i class={cat.icon}></i><span class="sidebar-text">{cat.name}</span>
        <span class="sidebar-count">{cat.apis.length}</span>
      </a>
    {/each}
  </div>
</nav>
