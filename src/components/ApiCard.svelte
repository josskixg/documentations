<script>
  import { createEventDispatcher, onDestroy, onMount } from 'svelte';

  export let api;
  export let index = 0;
  export let baseUrl;

  const dispatch = createEventDispatcher();
  
  // Generate unique ID for this card instance
  const uniqueCardId = `card-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;

  let cardEl;
  let bodyEl;
  let revealed = false;
  let isOpen = false;
  let isSending = false;
  let response = { visible: false };
  let responseImageUrl = '';
  let formValues = {};
  let paramsSignature = '';
  let activeCodeTab = 'curl';
  let isTabSwitching = false;
  let showParams = true;
  let showCodeExamples = true;
  let showResponseExample = true;
  let resizeObserver;
  let resizeTimeout;

  $: method = (api?.method || 'GET').toUpperCase();
  $: params = Array.isArray(api?.parameters) ? api.parameters : [];
  $: {
    const nextSignature = params
      .map((param) => `${param.name}:${param.defaultValue ?? ''}`)
      .join('|');
    if (nextSignature !== paramsSignature) {
      paramsSignature = nextSignature;
      const nextValues = {};
      for (const param of params) {
        nextValues[param.name] = param.defaultValue ?? '';
      }
      formValues = nextValues;
    }
  }

  function updateCardHeight() {
    if (!bodyEl || !isOpen) return;
    
    // Clear any pending updates
    if (resizeTimeout) clearTimeout(resizeTimeout);
    
    resizeTimeout = setTimeout(() => {
      // Temporarily set to auto to get real height
      const currentTransition = bodyEl.style.transition;
      bodyEl.style.transition = 'none';
      bodyEl.style.height = 'auto';
      const newHeight = bodyEl.scrollHeight;
      bodyEl.style.height = `${newHeight}px`;
      
      // Restore transition after a frame
      requestAnimationFrame(() => {
        bodyEl.style.transition = currentTransition;
      });
    }, 100);
  }

  function switchTab(tab) {
    if (isTabSwitching || activeCodeTab === tab) return;
    isTabSwitching = true;
    activeCodeTab = tab;
    setTimeout(() => {
      isTabSwitching = false;
    }, 300);
  }

  function generateCodeExamples() {
    const currentParams = {};
    for (const [name, value] of Object.entries(formValues)) {
      const trimmed = String(value).trim();
      if (trimmed) currentParams[name] = trimmed;
    }

    const apikey = currentParams.apikey || 'YOUR_API_KEY';
    delete currentParams.apikey;

    const endpoint = `${baseUrl}${api.endpointPath}`;
    
    if (method === 'GET') {
      const queryParams = { ...currentParams, apikey };
      const queryString = new URLSearchParams(queryParams).toString();
      const fullUrl = `${endpoint}?${queryString}`;

      return {
        curl: `curl -X GET "${fullUrl}"`,
        javascript: `// Fetch API
fetch('${fullUrl}')
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error('Error:', error));

// Axios
axios.get('${endpoint}', {
  params: ${JSON.stringify(queryParams, null, 2)}
})
.then(response => console.log(response.data))
.catch(error => console.error('Error:', error));`,
        php: `<?php
// cURL
$url = '${fullUrl}';
$ch = curl_init($url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = curl_exec($ch);
curl_close($ch);
$data = json_decode($response, true);
print_r($data);

// file_get_contents
$response = file_get_contents('${fullUrl}');
$data = json_decode($response, true);
print_r($data);
?>`,
        python: `import requests

url = '${endpoint}'
params = ${JSON.stringify(queryParams, null, 2).replace(/"([^"]+)":/g, "'$1':")}

response = requests.get(url, params=params)
data = response.json()
print(data)`
      };
    } else {
      // POST
      const paramsStr = JSON.stringify(currentParams, null, 2);
      const paramsOneLine = JSON.stringify(currentParams);
      
      return {
        curl: `curl -X POST "${endpoint}" \\
  -H "Content-Type: application/json" \\
  -H "Authorization: Bearer ${apikey}" \\
  -d '${paramsOneLine}'`,
        javascript: `// Fetch API
fetch('${endpoint}', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ${apikey}'
  },
  body: JSON.stringify(${paramsStr})
})
.then(response => response.json())
.then(data => console.log(data))
.catch(error => console.error('Error:', error));

// Axios
axios.post('${endpoint}', 
  ${paramsStr},
  {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer ${apikey}'
    }
  }
)
.then(response => console.log(response.data))
.catch(error => console.error('Error:', error));`,
        php: `<?php
$url = '${endpoint}';
$data = json_encode(${paramsStr});

$ch = curl_init($url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
curl_setopt($ch, CURLOPT_HTTPHEADER, [
    'Content-Type: application/json',
    'Authorization: Bearer ${apikey}'
]);

$response = curl_exec($ch);
curl_close($ch);
$result = json_decode($response, true);
print_r($result);
?>`,
        python: `import requests

url = '${endpoint}'
headers = {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer ${apikey}'
}
data = ${paramsStr.replace(/"([^"]+)":/g, "'$1':")}

response = requests.post(url, headers=headers, json=data)
result = response.json()
print(result)`
      };
    }
  }

  $: codeExamples = (() => {
    try {
      return generateCodeExamples();
    } catch (error) {
      console.error('Error generating code examples:', error);
      return {
        curl: '// Error generating code example',
        javascript: '// Error generating code example',
        php: '// Error generating code example',
        python: '# Error generating code example'
      };
    }
  })();

  function updateParam(name, value) {
    formValues = { ...formValues, [name]: value };
  }

  function toggleCard() {
    if (!bodyEl) return;

    if (isOpen) {
      bodyEl.style.height = `${bodyEl.scrollHeight}px`;
      requestAnimationFrame(() => {
        bodyEl.style.height = '0px';
      });
      isOpen = false;
      return;
    }

    bodyEl.style.height = '0px';
    isOpen = true;
    requestAnimationFrame(() => {
      bodyEl.style.height = `${bodyEl.scrollHeight}px`;
    });

    const handleEnd = () => {
      if (isOpen) bodyEl.style.height = 'auto';
      bodyEl.removeEventListener('transitionend', handleEnd);
    };

    bodyEl.addEventListener('transitionend', handleEnd);
  }

  function escHtml(input) {
    return String(input)
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
  }

  function syntaxHighlight(jsonText) {
    const safe = escHtml(jsonText);
    return safe.replace(
      /("(\\u[a-zA-Z0-9]{4}|\\[^u]|[^\\\"])*"(\s*:)?|\b(true|false|null)\b|-?\d+(?:\.\d*)?(?:[eE][+\-]?\d+)?)/g,
      (match) => {
        let style = 'color:#98D8C8';
        if (/^"/.test(match)) {
          style = /:$/.test(match) ? 'color:var(--accent-2)' : 'color:#A8D8A8';
        } else if (/true|false/.test(match)) {
          style = 'color:var(--post)';
        } else if (/null/.test(match)) {
          style = 'color:var(--text-muted)';
        } else {
          style = 'color:#D8A8FF';
        }

        return `<span style="${style}">${match}</span>`;
      }
    );
  }

  async function sendRequest() {
    const currentParams = {};
    for (const [name, value] of Object.entries(formValues)) {
      currentParams[name] = String(value).trim();
    }

    const apikey = currentParams.apikey || '';
    delete currentParams.apikey;

    let url = `${baseUrl}${api.endpointPath}`;
    const options = { method };

    response = {
      visible: true,
      badge: '',
      badgeClass: '',
      type: 'html',
      html: '<span style="color:var(--text-dim)">Waiting for response...</span>'
    };

    isSending = true;

    if (responseImageUrl) {
      URL.revokeObjectURL(responseImageUrl);
      responseImageUrl = '';
    }

    try {
      if (method === 'GET') {
        const query = new URLSearchParams({ ...currentParams, apikey });
        url += `?${query.toString()}`;
      } else {
        options.headers = {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apikey}`
        };
        options.body = JSON.stringify(currentParams);
      }

      const res = await fetch(url, options);
      const contentType = res.headers.get('content-type') || '';

      if (contentType.includes('image')) {
        const blob = await res.blob();
        responseImageUrl = URL.createObjectURL(blob);
        response = {
          visible: true,
          badge: `${res.status} ${res.ok ? 'OK' : 'Error'}`,
          badgeClass: res.ok ? 'status-ok' : 'status-err',
          type: 'image',
          imageUrl: responseImageUrl
        };
      } else if (contentType.includes('json')) {
        const json = await res.json();
        response = {
          visible: true,
          badge: `${res.status} ${res.ok ? 'OK' : 'Error'}`,
          badgeClass: res.ok ? 'status-ok' : 'status-err',
          type: 'html',
          html: syntaxHighlight(JSON.stringify(json, null, 2))
        };
      } else {
        const text = await res.text();
        response = {
          visible: true,
          badge: `${res.status} ${res.ok ? 'OK' : 'Error'}`,
          badgeClass: res.ok ? 'status-ok' : 'status-err',
          type: 'html',
          html: escHtml(text)
        };
      }
    } catch (error) {
      response = {
        visible: true,
        badge: 'Error',
        badgeClass: 'status-err',
        type: 'html',
        html: escHtml(error?.message || 'Request failed')
      };
    } finally {
      isSending = false;
      if (bodyEl && bodyEl.style.height !== 'auto') {
        bodyEl.style.height = `${bodyEl.scrollHeight}px`;
      }
    }
  }

  async function copyEndpoint() {
    const queryParams = {};
    for (const [name, value] of Object.entries(formValues)) {
      const trimmed = String(value).trim();
      if (trimmed) queryParams[name] = trimmed;
    }

    const query = new URLSearchParams(queryParams);
    const url = `${baseUrl}${api.endpointPath}${query.toString() ? `?${query.toString()}` : ''}`;

    try {
      await navigator.clipboard.writeText(url);
      dispatch('toast', { message: 'URL copied to clipboard' });
    } catch {
      dispatch('toast', { message: 'Failed to copy URL' });
    }
  }

  async function copyCode(lang) {
    try {
      await navigator.clipboard.writeText(codeExamples[lang]);
      dispatch('toast', { message: `${lang.toUpperCase()} code copied to clipboard` });
    } catch {
      dispatch('toast', { message: 'Failed to copy code' });
    }
  }

  async function copyResponse() {
    try {
      await navigator.clipboard.writeText(api.response.body);
      dispatch('toast', { message: 'Response copied to clipboard' });
    } catch {
      dispatch('toast', { message: 'Failed to copy response' });
    }
  }

  onMount(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        for (const entry of entries) {
          if (entry.isIntersecting) {
            revealed = true;
            observer.unobserve(entry.target);
          }
        }
      },
      { threshold: 0.05, rootMargin: '0px 0px -40px 0px' }
    );

    if (cardEl) observer.observe(cardEl);

    // Add resize observer to update card height on viewport change
    if (typeof ResizeObserver !== 'undefined') {
      resizeObserver = new ResizeObserver(() => {
        if (isOpen) {
          updateCardHeight();
        }
      });
      
      if (bodyEl) {
        resizeObserver.observe(bodyEl);
      }
    }

    // Fallback: window resize listener
    const handleResize = () => {
      if (isOpen) {
        updateCardHeight();
      }
    };
    
    window.addEventListener('resize', handleResize);

    return () => {
      observer.disconnect();
      if (resizeObserver) resizeObserver.disconnect();
      window.removeEventListener('resize', handleResize);
    };
  });

  onDestroy(() => {
    if (responseImageUrl) URL.revokeObjectURL(responseImageUrl);
    if (resizeObserver) resizeObserver.disconnect();
    if (resizeTimeout) clearTimeout(resizeTimeout);
  });
</script>

<div
  bind:this={cardEl}
  class="api-card"
  class:revealed
  class:open={isOpen}
  style={`transition-delay:${index * 40}ms`}
>
  <div
    class="card-head"
    role="button"
    tabindex="0"
    on:click={toggleCard}
    on:keydown={(event) => (event.key === 'Enter' || event.key === ' ') && toggleCard()}
  >
    <span class={`card-method method-${method}`}>{method}</span>
    <div class="card-info">
      <div class="card-title">
        <span class="card-status-dot"></span>
        {api.title}
        {#if api.service && api.service !== api.title}
          <span class="card-service-badge">{api.service}</span>
        {/if}
      </div>
      <div class="card-endpoint">{baseUrl}{api.endpointPath}</div>
      {#if api.description}
        <div class="card-desc">{api.description}</div>
      {/if}
    </div>
    <button class="card-toggle" type="button" on:click|stopPropagation={toggleCard}>
      <i class="fa-solid fa-plus"></i>
    </button>
  </div>

  <div bind:this={bodyEl} class="card-body">
    <div class="card-body-inner">
      {#if params.length > 0}
        <div class="params-section">
          <div 
            class="section-collapsible-header" 
            role="button"
            tabindex="0"
            aria-expanded={showParams}
            aria-controls="params-{uniqueCardId}"
            on:click={() => showParams = !showParams}
            on:keydown={(e) => (e.key === 'Enter' || e.key === ' ') && (showParams = !showParams)}
          >
            <div class="params-title">
              <i class="fa-solid fa-list-ul"></i>
              Parameters
              <span class="param-count">{params.length}</span>
            </div>
            <div class="header-actions">
              <button class="collapse-btn" type="button" tabindex="-1">
                <i class="fa-solid fa-chevron-{showParams ? 'up' : 'down'}"></i>
              </button>
            </div>
          </div>
          {#if showParams}
            <div class="params-grid" id="params-{uniqueCardId}">
              {#each params as param}
                <div class="param-card">
                  <div class="param-header">
                    <span class="param-name">{param.name}</span>
                    <span class={param.required ? 'param-badge param-required' : 'param-badge param-optional'}>
                      {param.required ? 'Required' : 'Optional'}
                    </span>
                  </div>
                  <div class="param-meta">
                    <span class="param-type-badge">
                      <i class="fa-solid fa-tag"></i>
                      {param.type || 'string'}
                    </span>
                    {#if param.defaultValue && (param.type || 'string').toLowerCase() !== 'string'}
                      <span class="param-default">
                        <i class="fa-solid fa-circle-info"></i>
                        Default: <code>{param.defaultValue}</code>
                      </span>
                    {/if}
                  </div>
                  {#if param.description}
                    <div class="param-description">
                      {param.description}
                    </div>
                  {/if}
                </div>
              {/each}
            </div>
          {/if}
        </div>
      {/if}

      <div class="code-examples-section">
        <div 
          class="section-collapsible-header" 
          role="button"
          tabindex="0"
          aria-expanded={showCodeExamples}
          aria-controls="code-{uniqueCardId}"
          on:click={() => showCodeExamples = !showCodeExamples}
          on:keydown={(e) => (e.key === 'Enter' || e.key === ' ') && (showCodeExamples = !showCodeExamples)}
        >
          <div class="code-examples-title">
            <i class="fa-solid fa-code"></i>Code Examples
          </div>
          <div class="header-actions">
            {#if showCodeExamples}
              <button 
                class="code-copy-btn" 
                type="button"
                on:click|stopPropagation={() => copyCode(activeCodeTab)}
                title="Copy code"
              >
                <i class="fa-regular fa-copy"></i>
              </button>
            {/if}
            <button class="collapse-btn" type="button" tabindex="-1">
              <i class="fa-solid fa-chevron-{showCodeExamples ? 'up' : 'down'}"></i>
            </button>
          </div>
        </div>
        {#if showCodeExamples}
          <div class="code-examples-tabs">
            <button 
              class="code-example-tab" 
              class:active={activeCodeTab === 'curl'}
              type="button"
              disabled={isTabSwitching}
              on:click={() => switchTab('curl')}
            >
              cURL
            </button>
            <button 
              class="code-example-tab" 
              class:active={activeCodeTab === 'javascript'}
              type="button"
              disabled={isTabSwitching}
              on:click={() => switchTab('javascript')}
            >
              JavaScript
            </button>
            <button 
              class="code-example-tab" 
              class:active={activeCodeTab === 'php'}
              type="button"
              disabled={isTabSwitching}
              on:click={() => switchTab('php')}
            >
              PHP
            </button>
            <button 
              class="code-example-tab" 
              class:active={activeCodeTab === 'python'}
              type="button"
              disabled={isTabSwitching}
              on:click={() => switchTab('python')}
            >
              Python
            </button>
          </div>
          <div class="code-examples-content">
            {#if activeCodeTab === 'curl'}
              <pre><code>{codeExamples.curl}</code></pre>
            {:else if activeCodeTab === 'javascript'}
              <pre><code>{codeExamples.javascript}</code></pre>
            {:else if activeCodeTab === 'php'}
              <pre><code>{codeExamples.php}</code></pre>
            {:else if activeCodeTab === 'python'}
              <pre><code>{codeExamples.python}</code></pre>
            {/if}
          </div>
        {/if}
      </div>

      {#if api.response}
        <div class="response-example-section">
          <div 
            class="section-collapsible-header" 
            role="button"
            tabindex="0"
            aria-expanded={showResponseExample}
            aria-controls="response-{uniqueCardId}"
            on:click={() => showResponseExample = !showResponseExample}
            on:keydown={(e) => (e.key === 'Enter' || e.key === ' ') && (showResponseExample = !showResponseExample)}
          >
            <div class="response-example-title">
              Response Example
              {#if api.response.statusCode}
                <span class="response-status-badge status-{api.response.statusCode >= 200 && api.response.statusCode < 300 ? 'success' : 'error'}">
                  {api.response.statusCode} {api.response.statusCode === 200 ? 'OK' : 'Error'}
                </span>
              {/if}
            </div>
            <div class="header-actions">
              {#if showResponseExample && api.response.body}
                <button 
                  class="code-copy-btn" 
                  type="button"
                  on:click|stopPropagation={copyResponse}
                  title="Copy response"
                >
                  <i class="fa-regular fa-copy"></i>
                </button>
              {/if}
              <button class="collapse-btn" type="button" tabindex="-1">
                <i class="fa-solid fa-chevron-{showResponseExample ? 'up' : 'down'}"></i>
              </button>
            </div>
          </div>
          
          {#if showResponseExample}
            {#if api.response.headers}
              <div class="response-headers">
                <div class="response-headers-title">
                  <i class="fa-solid fa-file-lines"></i>
                  Headers
                </div>
                <div class="response-headers-list">
                  {#each Object.entries(api.response.headers) as [key, value]}
                    <div class="response-header-item">
                      <span class="header-key">{key}:</span>
                      <span class="header-value">{value}</span>
                    </div>
                  {/each}
                </div>
              </div>
            {/if}

            {#if api.response.body}
              <div class="response-body-section">
                <div class="response-body-title">
                  <i class="fa-solid fa-brackets-curly"></i>
                  Response Body
                </div>
                <pre><code>{@html syntaxHighlight(api.response.body)}</code></pre>
              </div>
            {/if}
          {/if}
        </div>
      {/if}

        <div class="try-section">
        <div class="try-title"><i class="fa-solid fa-bolt"></i>Try It Out</div>
        <div class="try-form">
          {#each params as param, paramIndex}
            {@const inputId = `input-${uniqueCardId}-${paramIndex}`}
            <div class="try-field">
              <label class="try-label" for={inputId}>{param.name}{param.required ? ' *' : ''}</label>
              <input
                id={inputId}
                name={`${uniqueCardId}-${param.name}`}
                class="try-input"
                type="text"
                placeholder={param.defaultValue || param.description || param.name}
                value={formValues[param.name] ?? ''}
                on:input={(event) => updateParam(param.name, event.currentTarget.value)}
              />
            </div>
          {/each}
        </div>

        <div class="try-actions">
          <button class="btn btn-primary" type="button" class:loading={isSending} on:click={sendRequest}>
            {#if isSending}
              <i class="fa-solid fa-circle-notch spin"></i>Sending...
            {:else}
              <i class="fa-solid fa-paper-plane"></i>Send
            {/if}
          </button>
          <button class="btn btn-ghost" type="button" on:click={copyEndpoint}>
            <i class="fa-regular fa-copy"></i>Copy URL
          </button>
        </div>

        <div class="try-response" style:display={response.visible ? 'block' : 'none'}>
          {#if response.visible}
            <div class="response-label">
              <span>Response</span>
              {#if response.badge}
                <span class={`response-status ${response.badgeClass}`}>{response.badge}</span>
              {/if}
            </div>
            {#if response.type === 'image'}
              <img src={response.imageUrl} class="response-img" alt="Response" />
            {:else}
              <div class="response-body">{@html response.html}</div>
            {/if}
          {/if}
        </div>
      </div>
    </div>
  </div>
</div>
