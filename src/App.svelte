<script lang="ts">
  import { onMount } from 'svelte';
  import { parse } from 'marked';
  import DOMPurify from 'dompurify';
  import { Home, Briefcase, FolderGit2, ArrowLeft, ExternalLink, Code2, Clock, X } from '@lucide/svelte';

  // Navigation Items
  const navItems = [
    { key: 'h', label: 'home', id: 'home', hash: '#/', icon: Home },
    { key: 'e', label: 'experience', id: 'experience', hash: '#/experience', icon: Briefcase },
    { key: 'p', label: 'projects', id: 'projects', hash: '#/projects', icon: FolderGit2 }
  ];

  let currentTab = $state('home');
  let currentTime = $state('');

  // Bio State
  let bioHtml = $state('');
  let loadingMd = $state(true);

  // Modal Image Overlay State
  let activeImageOverlay = $state<string | null>(null);

  // Experience Dynamic Items State
  interface ExperienceMeta {
    slug: string;
    role: string;
    type: string;
    period: string;
    botUrl?: string;
    botName?: string;
    botDesc?: string;
    techStack: string[];
    htmlContent?: string;
  }

  let experiences = $state<ExperienceMeta[]>([]);
  let loadingExp = $state(true);

  // Selected Projects State
  interface ProjectMeta {
    slug: string;
    name: string;
    description: string;
    language: string;
    stars: number;
    tags: string[];
    githubUrl: string;
  }

  let projects = $state<ProjectMeta[]>([]);
  let loadingProjects = $state(true);
  let selectedProject = $state<ProjectMeta | null>(null);
  let selectedProjectSlug = $state<string | null>(null);
  let projectDetailHtml = $state('');
  let loadingProjectDetail = $state(false);

  // Configure DOMPurify to allow standard HTML plus dataset & class attributes
  const purifyConfig = {
    ADD_ATTR: ['target', 'rel', 'data-src'],
    ADD_TAGS: ['figcaption', 'figure']
  };

  async function sanitizeMarkdown(mdText: string): Promise<string> {
    const rawHtml = await parse(mdText);
    return DOMPurify.sanitize(rawHtml, purifyConfig);
  }

  function syncRouteFromHash() {
    const hash = window.location.hash || '#/';
    activeImageOverlay = null;

    if (hash === '#/' || hash === '') {
      currentTab = 'home';
      selectedProject = null;
      selectedProjectSlug = null;
    } else if (hash === '#/experience') {
      currentTab = 'experience';
      selectedProject = null;
      selectedProjectSlug = null;
    } else if (hash === '#/projects') {
      currentTab = 'projects';
      selectedProject = null;
      selectedProjectSlug = null;
    } else if (hash.startsWith('#/projects/')) {
      currentTab = 'projects';
      const slug = hash.replace('#/projects/', '');
      selectedProjectSlug = slug;
      if (projects.length > 0) {
        const found = projects.find(p => p.slug === slug);
        if (found) {
          openProjectDetail(found, false);
        }
      }
    }
  }

  function updateTime() {
    const now = new Date();
    currentTime = now.toLocaleTimeString('en-US', {
      hour12: false,
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit'
    }) + ' UTC';
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.target instanceof HTMLInputElement || e.target instanceof HTMLTextAreaElement) return;
    
    // Esc key closes modal image overlay or project detail view
    if (e.key === 'Escape') {
      if (activeImageOverlay) {
        activeImageOverlay = null;
        return;
      }
      if (selectedProject) {
        window.location.hash = '#/projects';
        return;
      }
    }

    const pressed = e.key.toLowerCase();
    const match = navItems.find(item => item.key === pressed);
    if (match) {
      window.location.hash = match.hash;
    }
  }

  async function loadBio() {
    try {
      loadingMd = true;
      const bioRes = await fetch('./content/bio.md');
      const bioText = await bioRes.text();
      bioHtml = await sanitizeMarkdown(bioText);
    } catch (err) {
      console.error('Error loading bio markdown:', err);
    } finally {
      loadingMd = false;
    }
  }

  async function loadExperiences() {
    try {
      loadingExp = true;
      const res = await fetch('./content/experience/experience.json');
      if (!res.ok) throw new Error('Failed to load experience index');
      const list: ExperienceMeta[] = await res.json();

      const loaded = await Promise.all(
        list.map(async (item) => {
          try {
            const mdRes = await fetch(`./content/experience/${item.slug}.md`);
            if (mdRes.ok) {
              const mdText = await mdRes.text();
              item.htmlContent = await sanitizeMarkdown(mdText);
            }
          } catch (e) {
            console.error(`Error loading experience md for ${item.slug}`, e);
          }
          return item;
        })
      );
      experiences = loaded;
    } catch (err) {
      console.error('Error loading experiences:', err);
    } finally {
      loadingExp = false;
    }
  }

  async function fetchSelectedProjects() {
    try {
      loadingProjects = true;
      const res = await fetch('./content/projects/projects.json');
      if (!res.ok) throw new Error('Failed to load selected projects index');
      projects = await res.json();
      
      // If URL hash points to a specific project on direct page refresh
      const hash = window.location.hash;
      if (hash.startsWith('#/projects/')) {
        const slug = hash.replace('#/projects/', '');
        const found = projects.find(p => p.slug === slug);
        if (found) {
          openProjectDetail(found, false);
        }
      }
    } catch (err) {
      console.error('Error loading selected projects:', err);
    } finally {
      loadingProjects = false;
    }
  }

  async function openProjectDetail(project: ProjectMeta, updateHash = true) {
    selectedProject = project;
    selectedProjectSlug = project.slug;
    if (updateHash) {
      window.location.hash = `#/projects/${project.slug}`;
    }
    loadingProjectDetail = true;
    try {
      const res = await fetch(`./content/projects/${project.slug}.md`);
      if (!res.ok) throw new Error('Failed to load project detail markdown');
      const mdText = await res.text();
      projectDetailHtml = await sanitizeMarkdown(mdText);
    } catch (err) {
      projectDetailHtml = `<p>Error loading details for ${project.name}.</p>`;
    } finally {
      loadingProjectDetail = false;
    }
  }

  function closeProjectDetail() {
    window.location.hash = '#/projects';
  }

  function handleContainerClick(e: MouseEvent) {
    const target = e.target as HTMLElement;
    const item = target.closest('.screenshot-item') as HTMLElement;
    if (item && item.dataset.src) {
      activeImageOverlay = item.dataset.src;
    }
  }

  // Interactive Custom Cursor State
  let cursorX = $state(-100);
  let cursorY = $state(-100);
  let ringX = $state(-100);
  let ringY = $state(-100);
  let isHovered = $state(false);
  let isClicking = $state(false);
  let cursorVisible = $state(false);

  let targetX = -100;
  let targetY = -100;
  let animFrameId: number;

  function updateCursorPos(e: MouseEvent) {
    cursorVisible = true;
    targetX = e.clientX;
    targetY = e.clientY;
    cursorX = e.clientX;
    cursorY = e.clientY;

    const target = e.target as HTMLElement;
    if (target) {
      const interactive = target.closest('a, button, input, textarea, [data-src], .project-card, .screenshot-item, .nav-item');
      isHovered = !!interactive;
    }
  }

  function handleMouseDown() {
    isClicking = true;
  }

  function handleMouseUp() {
    isClicking = false;
  }

  function handleMouseLeave() {
    cursorVisible = false;
  }

  function animateRing() {
    ringX += (targetX - ringX) * 0.18;
    ringY += (targetY - ringY) * 0.18;
    animFrameId = requestAnimationFrame(animateRing);
  }

  onMount(() => {
    updateTime();
    loadBio();
    loadExperiences();
    fetchSelectedProjects();
    syncRouteFromHash();

    animFrameId = requestAnimationFrame(animateRing);

    const interval = setInterval(updateTime, 1000);
    window.addEventListener('keydown', handleKeydown);
    window.addEventListener('hashchange', syncRouteFromHash);
    window.addEventListener('mousemove', updateCursorPos);
    window.addEventListener('mousedown', handleMouseDown);
    window.addEventListener('mouseup', handleMouseUp);
    document.addEventListener('mouseleave', handleMouseLeave);

    return () => {
      clearInterval(interval);
      cancelAnimationFrame(animFrameId);
      window.removeEventListener('keydown', handleKeydown);
      window.removeEventListener('hashchange', syncRouteFromHash);
      window.removeEventListener('mousemove', updateCursorPos);
      window.removeEventListener('mousedown', handleMouseDown);
      window.removeEventListener('mouseup', handleMouseUp);
      document.removeEventListener('mouseleave', handleMouseLeave);
    };
  });
</script>

<main class="portfolio-container" onclick={handleContainerClick}>
  <!-- Top Navigation Header -->
  <nav class="nav-bar">
    {#each navItems as item}
      {@const Icon = item.icon}
      <a 
        href={item.hash}
        class="nav-item {currentTab === item.id && !selectedProject ? 'active' : ''}"
      >
        <span class="nav-key">[{item.key}]</span>
        <Icon size={15} class="nav-icon" />
        <span class="nav-label">{item.label}</span>
      </a>
    {/each}
  </nav>

  <!-- TAB CONTENT: HOME -->
  {#if currentTab === 'home' && !selectedProject}
    <section class="bio-section">
      <!-- Profile Avatar & Title Header with Subdued Glow Radar Ring -->
      <div class="profile-header">
        <div class="avatar-radar-wrapper">
          <div class="glow-aura"></div>
          <svg class="radar-svg" viewBox="0 0 120 120">
            <!-- Outer dashed spinning ring -->
            <circle 
              cx="60" cy="60" r="56" 
              class="radar-ring outer-ring" 
              stroke-dasharray="14 12 4 12"
            />
            <!-- Inner reverse spinning ring -->
            <circle 
              cx="60" cy="60" r="50" 
              class="radar-ring inner-ring" 
              stroke-dasharray="28 8 2 8"
            />
          </svg>
          <div class="avatar-container">
            <img 
              src="./assets/avatar.jpg" 
              alt="th0truth Logo" 
              class="avatar-img"
            />
          </div>
        </div>
        <div class="profile-titles">
          <h1 class="profile-name">th0truth</h1>
          <p class="profile-role">software engineer</p>
        </div>
      </div>

      <!-- Bio Dynamic Markdown Content -->
      {#if loadingMd}
        <p class="status-msg">Loading bio...</p>
      {:else}
        <div class="bio-text markdown-content">
          {@html bioHtml}
        </div>
      {/if}
    </section>

  {:else if currentTab === 'experience' && !selectedProject}
    <section class="tab-content">
      <div class="section-title-wrapper">
        <Briefcase size={20} class="section-icon" />
        <h2>Experience</h2>
      </div>

      {#if loadingExp}
        <p class="status-msg">Loading experiences...</p>
      {:else if experiences.length === 0}
        <p class="status-msg">No experience entries listed.</p>
      {:else}
        <div class="timeline-list">
          {#each experiences as exp, idx}
            <div class="timeline-item">
              <!-- Left Vertical Line & Node Matching Reference Photo -->
              <div class="timeline-line-col">
                <div class="timeline-node"></div>
                <div class="timeline-vertical-line"></div>
              </div>

              <!-- Right Content Block with Tight Elegant Typography -->
              <div class="timeline-card">
                <div class="exp-title-row">
                  <h3 class="exp-role">{exp.role}</h3>
                  <span class="badge-freelance">{exp.type}</span>
                </div>
                <div class="exp-period">{exp.period}</div>

                {#if exp.botUrl}
                  <div class="exp-meta-line">
                    <span class="meta-label">Bot:</span>
                    <a href={exp.botUrl} target="_blank" rel="noreferrer" class="bot-link">{exp.botName}</a>
                    {#if exp.botDesc}
                      <span class="bot-desc">({exp.botDesc})</span>
                    {/if}
                  </div>
                {/if}

                {#if exp.techStack && exp.techStack.length > 0}
                  <div class="exp-meta-line tech-line">
                    <span class="meta-label">Tech Stack:</span>
                    <span class="tech-val">{exp.techStack.join(', ')}</span>
                  </div>
                {/if}

                {#if exp.htmlContent}
                  <div class="markdown-content exp-bullets">
                    {@html exp.htmlContent}
                  </div>
                {/if}
              </div>
            </div>
          {/each}
        </div>
      {/if}
    </section>

  {:else if currentTab === 'projects' && !selectedProject}
    <section class="tab-content">
      <div class="section-title-wrapper">
        <FolderGit2 size={20} class="section-icon" />
        <h2>Selected Projects</h2>
      </div>
      
      {#if loadingProjects}
        <p class="status-msg">Loading selected projects...</p>
      {:else if projects.length === 0}
        <p class="status-msg">No selected projects listed.</p>
      {:else}
        <div class="project-grid">
          {#each projects as project}
            <button class="project-card" onclick={() => openProjectDetail(project)}>
              <div class="card-header">
                <div class="card-title-group">
                  <Code2 size={16} class="card-title-icon" />
                  <h3>{project.name}</h3>
                </div>
                {#if project.stars > 0}
                  <span class="stars">★ {project.stars}</span>
                {/if}
              </div>
              <p>{project.description}</p>
              <div class="tags">
                {#if project.language}
                  <span class="tag-primary">{project.language}</span>
                {/if}
                {#each project.tags as tag}
                  <span class="tag-item">{tag}</span>
                {/each}
              </div>
            </button>
          {/each}
        </div>
      {/if}
    </section>

  <!-- DEDICATED PROJECT DETAIL PAGE -->
  {:else if selectedProject}
    <section class="tab-content project-detail-section">
      <div class="detail-top-bar">
        <button class="back-btn" onclick={closeProjectDetail}>
          <ArrowLeft size={16} class="btn-icon" />
          <span>back to projects</span>
        </button>
        <a href={selectedProject.githubUrl} target="_blank" rel="noreferrer" class="github-btn">
          <span>View on GitHub</span>
          <ExternalLink size={14} class="btn-icon-right" />
        </a>
      </div>

      {#if loadingProjectDetail}
        <p class="status-msg">Loading project details...</p>
      {:else}
        <div class="markdown-content project-detail-body">
          {@html projectDetailHtml}
        </div>
      {/if}
    </section>
  {/if}

  <hr class="divider" />

  <!-- Footer Links Section -->
  <footer class="footer-section">
    <div class="footer-header">
      <h2 class="footer-title">Find me here ~</h2>
      <div class="time-wrapper">
        <Clock size={13} class="clock-icon" />
        <span class="time-display">{currentTime}</span>
      </div>
    </div>

    <div class="links-grid">
      <div class="link-item">
        <span class="link-label">GitHub</span>
        <a href="https://github.com/th0truth" target="_blank" rel="noreferrer" class="link-url">
          <svg viewBox="0 0 24 24" width="15" height="15" fill="currentColor">
            <path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"/>
          </svg>
          <span>@th0truth</span>
        </a>
      </div>
    </div>
  </footer>

  <!-- FULLSCREEN IMAGE OVERLAY MODAL -->
  {#if activeImageOverlay}
    <div class="image-modal-overlay" onclick={() => activeImageOverlay = null}>
      <button class="modal-close-btn" onclick={() => activeImageOverlay = null}>
        <X size={20} />
      </button>
      <div class="modal-content-wrapper" onclick={(e) => e.stopPropagation()}>
        <img src={activeImageOverlay} alt="Enlarged screenshot preview" class="modal-img" />
      </div>
    </div>
  {/if}

  <!-- CUSTOM ANIMATED CURSOR -->
  {#if cursorVisible}
    <div 
      class="custom-cursor-dot {isHovered ? 'hovered' : ''} {isClicking ? 'clicking' : ''}" 
      style="transform: translate3d({cursorX}px, {cursorY}px, 0);"
    ></div>
    <div 
      class="custom-cursor-ring {isHovered ? 'hovered' : ''} {isClicking ? 'clicking' : ''}" 
      style="transform: translate3d({ringX}px, {ringY}px, 0);"
    ></div>
  {/if}
</main>

<style>
  /* Custom Animated Cursor Styling Matching Reference Photo */
  .custom-cursor-dot {
    position: fixed;
    top: -4px;
    left: -4px;
    width: 8px;
    height: 8px;
    background-color: #ffffff;
    border-radius: 50%;
    pointer-events: none;
    z-index: 99999;
    transition: transform 0.05s linear, width 0.2s ease, height 0.2s ease, background-color 0.2s ease;
    box-shadow: 0 0 10px rgba(255, 255, 255, 0.6);
    will-change: transform;
  }

  .custom-cursor-ring {
    position: fixed;
    top: -16px;
    left: -16px;
    width: 32px;
    height: 32px;
    border: 2px solid rgba(255, 255, 255, 0.85);
    border-radius: 50%;
    pointer-events: none;
    z-index: 99998;
    transition: width 0.25s cubic-bezier(0.16, 1, 0.3, 1), 
                height 0.25s cubic-bezier(0.16, 1, 0.3, 1), 
                top 0.25s cubic-bezier(0.16, 1, 0.3, 1), 
                left 0.25s cubic-bezier(0.16, 1, 0.3, 1), 
                border-color 0.2s ease, 
                background-color 0.2s ease;
    box-shadow: 0 0 14px rgba(255, 255, 255, 0.15);
    will-change: transform;
  }

  /* Hover state over interactive elements (buttons, links, screenshots) */
  .custom-cursor-dot.hovered {
    width: 6px;
    height: 6px;
    top: -3px;
    left: -3px;
    background-color: #ffffff;
  }

  .custom-cursor-ring.hovered {
    top: -24px;
    left: -24px;
    width: 48px;
    height: 48px;
    border-color: rgba(255, 255, 255, 0.95);
    background-color: rgba(255, 255, 255, 0.05);
  }

  /* Active Click State */
  .custom-cursor-dot.clicking {
    transform: scale(0.6);
  }

  .custom-cursor-ring.clicking {
    transform: scale(0.85);
    border-color: #ffffff;
  }

  @media (pointer: coarse) {
    .custom-cursor-dot, .custom-cursor-ring {
      display: none !important;
    }
  }
  .portfolio-container {
    max-width: 760px;
    width: 100%;
    margin: 0 auto;
    box-sizing: border-box;
  }

  /* Navigation Header at Top */
  .nav-bar {
    display: flex;
    justify-content: flex-start;
    align-items: center;
    flex-wrap: wrap;
    gap: 1.5rem;
    margin-bottom: 3.5rem;
    font-family: var(--font-mono);
    font-size: 0.95rem;
    width: 100%;
  }

  .nav-item {
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    color: var(--text-muted);
    transition: color 0.15s ease;
    text-decoration: none !important;
  }

  .nav-key {
    color: var(--text-muted);
  }

  :global(.nav-icon) {
    opacity: 0.7;
    transition: opacity 0.15s ease;
  }

  .nav-item.active :global(.nav-icon) {
    opacity: 1;
  }

  .nav-item.active .nav-label {
    color: var(--text-primary);
    text-decoration: underline !important;
    text-underline-offset: 4px;
  }

  .nav-item:hover {
    color: var(--text-secondary);
  }

  /* Profile Avatar Header & Subdued Radar Ring */
  .profile-header {
    display: flex;
    align-items: center;
    gap: 1.75rem;
    margin-bottom: 2.25rem;
  }

  .avatar-radar-wrapper {
    position: relative;
    width: 110px;
    height: 110px;
    display: flex;
    justify-content: center;
    align-items: center;
    flex-shrink: 0;
  }

  .glow-aura {
    position: absolute;
    inset: -2px;
    border-radius: 50%;
    background: radial-gradient(circle, rgba(255, 255, 255, 0.08) 0%, rgba(255, 255, 255, 0) 70%);
    opacity: 0.25;
    filter: blur(6px);
    transition: opacity 0.3s ease;
  }

  .avatar-radar-wrapper:hover .glow-aura {
    opacity: 0.5;
  }

  .radar-svg {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    transition: opacity 0.3s ease;
  }

  .radar-ring {
    fill: none;
    stroke: #333338;
    stroke-width: 1.1;
    transform-origin: center;
    transition: stroke 0.3s ease;
  }

  .avatar-radar-wrapper:hover .radar-ring {
    stroke: #5a5a62;
  }

  .outer-ring {
    animation: rotateOuter 24s linear infinite;
  }

  .inner-ring {
    stroke: #404046;
    animation: rotateInner 16s linear infinite reverse;
  }

  @keyframes rotateOuter {
    from {
      transform: rotate(0deg);
    }
    to {
      transform: rotate(360deg);
    }
  }

  @keyframes rotateInner {
    from {
      transform: rotate(0deg);
    }
    to {
      transform: rotate(360deg);
    }
  }

  .avatar-container {
    width: 86px;
    height: 86px;
    border-radius: 50%;
    overflow: hidden;
    border: 1px solid var(--border-color);
    background: #000;
    z-index: 1;
    transition: border-color 0.3s ease;
  }

  .avatar-radar-wrapper:hover .avatar-container {
    border-color: #5e5e65;
  }

  .avatar-img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    filter: contrast(105%);
  }

  .profile-name {
    font-size: 2.5rem;
    font-weight: 600;
    letter-spacing: -0.02em;
    line-height: 1.1;
    color: var(--text-primary);
  }

  .profile-role {
    font-family: var(--font-mono);
    color: var(--text-secondary);
    font-size: 0.95rem;
    margin-top: 0.35rem;
  }

  /* Multi-Job Dynamic Timeline Layout */
  .timeline-list {
    display: flex;
    flex-direction: column;
    gap: 2.25rem;
  }

  .timeline-item {
    display: flex;
    gap: 1.25rem;
    position: relative;
  }

  .timeline-line-col {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 18px;
    flex-shrink: 0;
    position: relative;
  }

  .timeline-node {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    background: var(--bg-main);
    border: 2px solid #71717a;
    margin-top: 0.35rem;
    z-index: 2;
    box-shadow: 0 0 6px rgba(113, 113, 122, 0.4);
  }

  .timeline-vertical-line {
    position: absolute;
    top: 0.35rem;
    bottom: -2.25rem;
    left: 8px;
    width: 2px;
    background: linear-gradient(180deg, #52525b 0%, #27272a 100%);
    z-index: 1;
  }

  .timeline-card {
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .exp-title-row {
    display: flex;
    align-items: center;
    gap: 0.6rem;
  }

  .exp-role {
    font-size: 1.25rem;
    font-weight: 600;
    color: var(--text-primary);
    line-height: 1.2;
  }

  .badge-freelance {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    color: var(--text-secondary);
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid var(--border-color);
    padding: 0.15rem 0.5rem;
    border-radius: 4px;
  }

  .exp-period {
    font-family: var(--font-mono);
    font-size: 0.85rem;
    color: var(--text-muted);
    margin-bottom: 0.35rem;
  }

  .exp-meta-line {
    font-size: 0.92rem;
    color: var(--text-secondary);
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
    align-items: center;
    line-height: 1.4;
  }

  .meta-label {
    font-weight: 600;
    color: var(--text-primary);
  }

  .bot-link {
    color: var(--text-primary);
    text-decoration: underline;
    text-underline-offset: 3px;
    font-family: var(--font-mono);
    font-size: 0.88rem;
  }

  .bot-desc {
    color: var(--text-muted);
    font-style: italic;
    font-size: 0.88rem;
  }

  .tech-line {
    margin-bottom: 0.5rem;
  }

  .tech-val {
    color: var(--text-secondary);
  }

  .exp-bullets {
    margin-top: 0.25rem;
  }

  /* Bio & Markdown Content Styling */
  .markdown-content {
    display: flex;
    flex-direction: column;
    color: var(--text-secondary);
    font-size: 1rem;
    line-height: 1.65;
  }

  :global(.markdown-content p) {
    margin-bottom: 0.85rem;
  }

  :global(.markdown-content p:last-child) {
    margin-bottom: 0;
  }

  :global(.project-detail-body h1) {
    color: var(--text-primary);
    font-size: 2.2rem;
    font-weight: 600;
    margin-bottom: 1.25rem;
    line-height: 1.2;
  }

  :global(.project-detail-body h2) {
    color: var(--text-primary);
    font-size: 1.35rem;
    font-weight: 600;
    margin-top: 1.75rem;
    margin-bottom: 0.75rem;
  }

  :global(.project-detail-body p) {
    margin-bottom: 1.25rem;
  }

  :global(.project-detail-body ul) {
    margin-left: 1.25rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    margin-bottom: 1.5rem;
  }

  :global(.markdown-content strong) {
    color: var(--text-primary);
    font-weight: 500;
  }

  :global(.markdown-content a) {
    color: var(--text-primary);
    text-decoration: underline;
    text-underline-offset: 3px;
  }

  /* Screenshot Grid Gallery */
  :global(.screenshots-group) {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.85rem;
    margin: 1.25rem 0 2rem 0;
  }

  :global(.screenshot-item) {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    padding: 0.4rem;
    border-radius: 6px;
    overflow: hidden;
    cursor: pointer;
    transition: border-color 0.2s ease, transform 0.2s ease;
  }

  :global(.screenshot-item:hover) {
    border-color: #71717a;
    transform: translateY(-2px);
  }

  :global(.screenshot-item img) {
    width: 100%;
    height: 105px;
    object-fit: cover;
    border-radius: 4px;
  }

  :global(.screenshot-item figcaption) {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    color: var(--text-muted);
    text-align: center;
    padding-top: 0.15rem;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  :global(.screenshot-item:hover figcaption) {
    color: var(--text-primary);
  }

  /* Image Modal Overlay Styling */
  .image-modal-overlay {
    position: fixed;
    inset: 0;
    background: rgba(10, 10, 12, 0.88);
    backdrop-filter: blur(8px);
    z-index: 1000;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 2rem;
    animation: fadeInOverlay 0.2s ease;
  }

  .modal-close-btn {
    position: absolute;
    top: 1.5rem;
    right: 1.5rem;
    background: rgba(255, 255, 255, 0.1);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: var(--text-primary);
    border-radius: 50%;
    width: 40px;
    height: 40px;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    transition: background 0.2s ease;
  }

  .modal-close-btn:hover {
    background: rgba(255, 255, 255, 0.2);
  }

  .modal-content-wrapper {
    max-width: 90vw;
    max-height: 85vh;
    display: flex;
    justify-content: center;
    align-items: center;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.6);
  }

  .modal-img {
    max-width: 100%;
    max-height: 85vh;
    object-fit: contain;
    border-radius: 6px;
  }

  @keyframes fadeInOverlay {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  /* Section Title Header */
  .section-title-wrapper {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    margin-bottom: 1.5rem;
    color: var(--text-primary);
  }

  :global(.section-icon) {
    color: var(--text-secondary);
  }

  /* Divider */
  .divider {
    border: none;
    border-top: 1px solid var(--border-color);
    margin: 3.5rem 0 2.5rem 0;
  }

  /* Footer */
  .footer-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1.75rem;
  }

  .footer-title {
    font-size: 1.1rem;
    font-weight: 500;
  }

  .time-wrapper {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    color: var(--text-muted);
  }

  .time-display {
    font-family: var(--font-mono);
    font-size: 0.85rem;
  }

  .links-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 2rem 1.5rem;
  }

  .link-item {
    display: flex;
    flex-direction: column;
    gap: 0.3rem;
  }

  .link-label {
    font-size: 0.95rem;
    color: var(--text-primary);
  }

  .link-url {
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    font-family: var(--font-mono);
    font-size: 0.85rem;
    color: var(--text-secondary);
    text-decoration: underline;
    text-underline-offset: 3px;
    text-decoration-color: var(--text-muted);
  }

  .link-url:hover {
    color: var(--text-primary);
  }

  .tab-content {
    min-height: 260px;
  }

  .status-msg {
    font-family: var(--font-mono);
    color: var(--text-muted);
    font-size: 0.9rem;
  }

  /* Selected Projects Grid */
  .project-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
    gap: 1.25rem;
  }

  .project-card {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    padding: 1.25rem;
    border-radius: 8px;
    text-align: left;
    transition: border-color 0.2s ease, transform 0.2s ease;
    cursor: pointer;
  }

  .project-card:hover {
    border-color: #5e5e65;
    transform: translateY(-2px);
  }

  .card-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.5rem;
    width: 100%;
  }

  .card-title-group {
    display: flex;
    align-items: center;
    gap: 0.4rem;
  }

  :global(.card-title-icon) {
    color: var(--text-muted);
  }

  .card-header h3 {
    font-size: 1.1rem;
    color: var(--text-primary);
    font-family: var(--font-mono);
  }

  .stars {
    font-family: var(--font-mono);
    font-size: 0.8rem;
    color: var(--text-secondary);
  }

  .project-card p {
    font-size: 0.9rem;
    color: var(--text-secondary);
    margin-bottom: 1rem;
    line-height: 1.5;
  }

  .tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.4rem;
    font-family: var(--font-mono);
    font-size: 0.75rem;
    color: var(--text-secondary);
  }

  .tag-primary {
    color: var(--text-primary);
    border: 1px solid var(--border-color);
    padding: 0.15rem 0.45rem;
    border-radius: 4px;
    background: rgba(255, 255, 255, 0.03);
  }

  .tag-item {
    color: var(--text-secondary);
    border: 1px solid var(--border-color);
    padding: 0.15rem 0.45rem;
    border-radius: 4px;
    background: rgba(255, 255, 255, 0.015);
  }

  /* Dedicated Project Detail Section */
  .project-detail-section {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
  }

  .detail-top-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-family: var(--font-mono);
    font-size: 0.9rem;
    margin-bottom: 0.5rem;
  }

  .back-btn {
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    color: var(--text-secondary);
    transition: color 0.15s ease;
  }

  .back-btn:hover {
    color: var(--text-primary);
  }

  .github-btn {
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    padding: 0.4rem 0.8rem;
    border-radius: 6px;
    color: var(--text-primary);
    font-size: 0.85rem;
    transition: border-color 0.2s ease;
  }

  .github-btn:hover {
    border-color: #5e5e65;
  }

  /* Mobile & Tablet Responsiveness */
  @media (max-width: 640px) {
    .nav-bar {
      gap: 0.85rem 1rem;
      margin-bottom: 2.5rem;
      font-size: 0.88rem;
    }

    .project-grid {
      grid-template-columns: 1fr;
    }

    :global(.screenshots-group) {
      grid-template-columns: repeat(2, 1fr);
    }

    .profile-header {
      gap: 1.25rem;
    }

    .avatar-radar-wrapper {
      width: 92px;
      height: 92px;
    }

    .avatar-container {
      width: 74px;
      height: 74px;
    }

    .profile-name {
      font-size: 2rem;
    }

    .footer-header {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.5rem;
    }

    .timeline-item {
      gap: 0.85rem;
    }
  }
</style>
