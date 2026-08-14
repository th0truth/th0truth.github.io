<script lang="ts">
  import { onMount } from 'svelte';
  import { parse } from 'marked';
  import DOMPurify from 'dompurify';

  // Navigation Items
  const navItems = [
    { key: 'h', label: 'home', id: 'home', hash: '#/' },
    { key: 'e', label: 'experience', id: 'experience', hash: '#/experience' },
    { key: 'p', label: 'projects', id: 'projects', hash: '#/projects' },
  ];

  let currentTab = $state('home');
  let mobileMenuOpen = $state(false);

  function closeMobileMenu() {
    mobileMenuOpen = false;
  }

  // Bio State
  let bioHtml = $state('');
  let loadingMd = $state(true);

  // Skills State
  interface SkillCategory {
    category: string;
    primary?: string[];
    familiar?: string[];
    items?: string[];
  }
  interface SoftSkill {
    title: string;
    icon: string;
    description: string;
  }
  interface SkillsData {
    technical: SkillCategory[];
    soft: SoftSkill[];
  }
  let skillsData = $state<SkillsData | null>(null);

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

  interface ProjectMeta {
    slug: string;
    name: string;
    icon?: string;
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

  // Programming language filtering state
  let selectedLanguageFilter = $state('All');

  let availableLanguages = $derived.by(() => {
    const langs = new Set<string>();
    for (const p of projects ?? []) {
      if (p.language) {
        langs.add(p.language);
      }
    }
    return ['All', ...Array.from(langs).sort()];
  });

  let filteredProjects = $derived.by(() => {
    if (selectedLanguageFilter === 'All') {
      return projects;
    }
    return projects.filter(p => p.language === selectedLanguageFilter);
  });

  // Derived: first 3 projects for featured section on home page
  let featuredProjects = $derived((projects ?? []).slice(0, 3));

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
    let hash = (window.location.hash || '#/').toLowerCase();
    if (hash.length > 2 && hash.endsWith('/')) {
      hash = hash.slice(0, -1);
    }
    activeImageOverlay = null;

    if (hash === '#/' || hash === '' || hash === '#') {
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

  function handleKeydown(e: KeyboardEvent) {
    if (e.target instanceof HTMLInputElement || e.target instanceof HTMLTextAreaElement) return;

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

  async function loadSkills() {
    try {
      const res = await fetch('./content/skills/skills.json');
      if (res.ok) {
        skillsData = await res.json();
      }
    } catch (err) {
      console.error('Error loading skills:', err);
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

  function getProjectTags(project: ProjectMeta): string[] {
    if (!project || !project.tags) return [];
    const lang = (project.language || '').toLowerCase().trim();
    const seen = new Set<string>();
    if (lang) {
      seen.add(lang);
      if (lang === 'c++') { seen.add('cpp'); seen.add('cpp17'); seen.add('cpp20'); seen.add('c++17'); seen.add('c++20'); }
      else if (lang === 'c') { seen.add('c'); }
      else if (lang === 'python') { seen.add('python'); seen.add('py'); }
      else if (lang === 'shell') { seen.add('shell'); }
    }
    const result: string[] = [];
    for (const t of project.tags) {
      const lower = t.toLowerCase().trim();
      if (!seen.has(lower)) {
        seen.add(lower);
        result.push(t);
      }
    }
    return result;
  }

  function handleContainerClick(e: MouseEvent) {
    const target = e.target as HTMLElement;
    const item = target.closest('.screenshot-item') as HTMLElement;
    if (item && item.dataset.src) {
      activeImageOverlay = item.dataset.src;
    }
  }

  // Custom cursor state & smooth tracking
  let isHovered = $state(false);
  let isClicking = $state(false);
  let isCursorVisible = $state(false);

  let mouseX = -100;
  let mouseY = -100;
  let currentRingX = -100;
  let currentRingY = -100;
  let animFrameId: number;

  function updateCursorPos(e: MouseEvent) {
    if (!isCursorVisible) isCursorVisible = true;
    mouseX = e.clientX;
    mouseY = e.clientY;

    const dotEl = document.getElementById('custom-cursor-dot');
    if (dotEl) {
      dotEl.style.transform = `translate3d(${mouseX}px, ${mouseY}px, 0)`;
    }

    const target = e.target as HTMLElement;
    if (target) {
      const interactive = target.closest('a, button, [role="button"], input, textarea, select, label, [data-src], .screenshot-item, .project-card, .tag, .filter-chip');
      const shouldHover = !!interactive;
      if (isHovered !== shouldHover) {
        isHovered = shouldHover;
      }
    }
  }

  function handleMouseDown() {
    isClicking = true;
  }

  function handleMouseUp() {
    isClicking = false;
  }

  function handleMouseLeave() {
    isCursorVisible = false;
  }

  function animateRing() {
    currentRingX += (mouseX - currentRingX) * 0.22;
    currentRingY += (mouseY - currentRingY) * 0.22;
    const ringEl = document.getElementById('custom-cursor-ring');
    if (ringEl) {
      ringEl.style.transform = `translate3d(${currentRingX}px, ${currentRingY}px, 0)`;
    }
    animFrameId = requestAnimationFrame(animateRing);
  }

  onMount(() => {
    loadBio();
    loadSkills();
    loadExperiences();
    fetchSelectedProjects();
    syncRouteFromHash();

    animFrameId = requestAnimationFrame(animateRing);

    window.addEventListener('keydown', handleKeydown);
    window.addEventListener('hashchange', syncRouteFromHash);
    window.addEventListener('click', handleContainerClick);
    window.addEventListener('mousemove', updateCursorPos);
    window.addEventListener('mousedown', handleMouseDown);
    window.addEventListener('mouseup', handleMouseUp);
    document.addEventListener('mouseleave', handleMouseLeave);

    return () => {
      cancelAnimationFrame(animFrameId);
      window.removeEventListener('keydown', handleKeydown);
      window.removeEventListener('hashchange', syncRouteFromHash);
      window.removeEventListener('click', handleContainerClick);
      window.removeEventListener('mousemove', updateCursorPos);
      window.removeEventListener('mousedown', handleMouseDown);
      window.removeEventListener('mouseup', handleMouseUp);
      document.removeEventListener('mouseleave', handleMouseLeave);
    };
  });
</script>

<main class="site-wrapper">
  <!-- Header -->
  <header class="site-header">
    <a href="#/" class="header-brand">
      <span class="brand-name">th0truth</span>
    </a>

    <!-- Desktop nav -->
    <nav class="nav-desktop">
      {#each navItems as item}
        <a
          href={item.hash}
          class="nav-link {currentTab === item.id && !selectedProject ? 'active' : ''}"
          onclick={closeMobileMenu}
        >
          {item.label}
        </a>
      {/each}
    </nav>

    <!-- Mobile hamburger -->
    <button
      class="hamburger-btn"
      onclick={() => mobileMenuOpen = !mobileMenuOpen}
      aria-label="Toggle navigation menu"
    >
      <span class="hamburger-line {mobileMenuOpen ? 'open' : ''}"></span>
      <span class="hamburger-line {mobileMenuOpen ? 'open' : ''}"></span>
      <span class="hamburger-line {mobileMenuOpen ? 'open' : ''}"></span>
    </button>
  </header>

  <!-- Mobile dropdown -->
  {#if mobileMenuOpen}
    <nav class="mobile-menu">
      {#each navItems as item}
        <a
          href={item.hash}
          class="mobile-nav-link {currentTab === item.id && !selectedProject ? 'active' : ''}"
          onclick={closeMobileMenu}
        >
          {item.label}
        </a>
      {/each}
    </nav>
  {/if}

  <div class="content-area">

    <!-- ═══════════════════ HOME ═══════════════════ -->
    {#if currentTab === 'home' && !selectedProject}
      <section class="page-section" id="home-section">
        <!-- Profile intro -->
        <div class="profile-intro">
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
                alt="Vladyslav Panasiuk" 
                class="avatar-img"
              />
            </div>
          </div>
          <div class="profile-text">
            <h1 class="profile-name">Vladyslav Panasiuk</h1>
            <p class="profile-tagline">software engineer</p>
          </div>
        </div>

        <!-- Bio -->
        {#if loadingMd}
          <p class="loading-text">loading...</p>
        {:else}
          <div class="prose">
            {@html bioHtml}
          </div>
        {/if}

        <!-- Skills -->
        {#if skillsData}
          <div class="section-block">
            <h2 class="section-heading"><span class="heading-hash">#</span>skills</h2>
            <div class="skills-grid">
              {#each skillsData.technical as cat}
                <div class="skill-category">
                  <span class="skill-cat-title">{cat.category}:</span>
                  <div class="skill-cat-content">
                    {#if cat.primary}
                      <div class="skill-subline">
                        <span class="skill-sublabel">Primary:</span>
                        <span class="skill-subitems">{cat.primary.join(', ')}</span>
                      </div>
                      {#if cat.familiar}
                        <div class="skill-subline">
                          <span class="skill-sublabel">Familiar:</span>
                          <span class="skill-subitems">{cat.familiar.join(', ')}</span>
                        </div>
                      {/if}
                    {:else if cat.items}
                      <div class="skill-subline">
                        <span class="skill-subitems">{cat.items.join(', ')}</span>
                      </div>
                    {/if}
                  </div>
                </div>
              {/each}
            </div>
          </div>
        {/if}

        <!-- Experience Preview -->
        <div class="section-block">
          <h2 class="section-heading"><span class="heading-hash">#</span>experience</h2>
          {#if loadingExp}
            <p class="loading-text">loading...</p>
          {:else if experiences.length > 0}
            {#each experiences as exp}
              <a href="#/experience" class="preview-card">
                <div class="preview-card-top">
                  <div class="preview-card-left">
                    <span class="preview-role">{exp.role}</span>
                    <span class="preview-badge">{exp.type}</span>
                  </div>
                  <span class="preview-period">{exp.period}</span>
                </div>
                {#if exp.techStack && exp.techStack.length > 0}
                  <div class="preview-tags">
                    {#each exp.techStack as tech}
                      <span class="pill">{tech}</span>
                    {/each}
                  </div>
                {/if}
                <span class="card-arrow">→</span>
              </a>
            {/each}
          {/if}
        </div>

        <!-- Featured Projects -->
        <div class="section-block">
          <h2 class="section-heading"><span class="heading-hash">#</span>selected projects</h2>
          {#if loadingProjects}
            <p class="loading-text">loading...</p>
          {:else if featuredProjects.length > 0}
            <div class="featured-list">
              {#each featuredProjects as project}
                <button class="featured-row" onclick={() => openProjectDetail(project)}>
                  <div class="featured-info">
                    <span class="featured-name">{project.name}</span>
                    {#if project.language}
                      <span class="lang-dot">{project.language}</span>
                    {/if}
                  </div>
                  <p class="featured-desc">{project.description}</p>
                  <span class="card-arrow">→</span>
                </button>
              {/each}
            </div>
            <a href="#/projects" class="view-all-link">
              view all projects →
            </a>
          {/if}
        </div>
      </section>

    <!-- ═══════════════════ EXPERIENCE ═══════════════════ -->
    {:else if currentTab === 'experience' && !selectedProject}
      <section class="page-section" id="experience-section">
        <h2 class="page-title"><span class="heading-hash">#</span>experience</h2>

        {#if loadingExp}
          <p class="loading-text">loading...</p>
        {:else if experiences.length === 0}
          <p class="loading-text">no experience entries.</p>
        {:else}
          <div class="exp-list">
            {#each experiences as exp}
              <article class="exp-card">
                <div class="exp-top-row">
                  <div class="exp-title-group">
                    <h3 class="exp-role">{exp.role}</h3>
                    <span class="exp-type-badge">{exp.type}</span>
                  </div>
                  <span class="exp-period">{exp.period}</span>
                </div>

                {#if exp.botUrl}
                  <div class="exp-meta">
                    <span class="meta-key">Bot:</span>
                    <a href={exp.botUrl} target="_blank" rel="noreferrer" class="meta-link">{exp.botName}</a>
                    {#if exp.botDesc}
                      <span class="meta-note">({exp.botDesc})</span>
                    {/if}
                  </div>
                {/if}

                {#if exp.techStack && exp.techStack.length > 0}
                  <div class="exp-meta">
                    <span class="meta-key">Stack:</span>
                    <span class="meta-val">{exp.techStack.join(', ')}</span>
                  </div>
                {/if}

                {#if exp.htmlContent}
                  <div class="prose exp-prose">
                    {@html exp.htmlContent}
                  </div>
                {/if}
              </article>
            {/each}
          </div>
        {/if}
      </section>

    <!-- ═══════════════════ PROJECTS ═══════════════════ -->
    {:else if currentTab === 'projects' && !selectedProject}
      <section class="page-section" id="projects-section">
        <h2 class="page-title"><span class="heading-hash">#</span>projects</h2>

        {#if loadingProjects}
          <p class="loading-text">loading...</p>
        {:else if projects.length === 0}
          <p class="loading-text">no projects listed.</p>
        {:else}
          <!-- Language filter -->
          <div class="filter-bar">
            {#each availableLanguages as lang}
              <button
                class="filter-btn {selectedLanguageFilter === lang ? 'active' : ''}"
                onclick={() => selectedLanguageFilter = lang}
              >
                {lang}
              </button>
            {/each}
          </div>

          <div class="project-grid">
            {#each filteredProjects as project}
              <button class="project-card" onclick={() => openProjectDetail(project)}>
                <div class="card-top">
                  <h3 class="card-name">{project.name}</h3>
                  {#if project.stars > 0}
                    <span class="card-stars">★ {project.stars}</span>
                  {/if}
                </div>
                <p class="card-desc">{project.description}</p>
                <div class="card-tags">
                  {#if project.language}
                    <span class="pill primary">{project.language}</span>
                  {/if}
                  {#each getProjectTags(project) as tag}
                    <span class="pill">{tag}</span>
                  {/each}
                </div>
                <span class="card-arrow">→</span>
              </button>
            {/each}
          </div>
        {/if}
      </section>

    <!-- ═══════════════════ PROJECT DETAIL ═══════════════════ -->
    {:else if selectedProject}
      <section class="page-section" id="project-detail-section">
        <div class="detail-bar">
          <button class="back-link" onclick={closeProjectDetail}>
            ← back to projects
          </button>
          <a href={selectedProject.githubUrl} target="_blank" rel="noreferrer" class="github-link">
            view on github →
          </a>
        </div>

        {#if loadingProjectDetail}
          <p class="loading-text">loading...</p>
        {:else}
          <div class="prose detail-prose">
            {@html projectDetailHtml}
          </div>
        {/if}
      </section>
    {/if}
  </div>

  <!-- Divider -->
  <hr class="site-divider" />

  <!-- Footer -->
  <footer class="site-footer">
    <h2 class="footer-heading"><span class="heading-hash">#</span>find me here</h2>
    <div class="footer-links">
      <div class="footer-link-item">
        <span class="footer-link-label">LinkedIn</span>
        <a href="https://www.linkedin.com/in/vladyslav-panasiuk-481582370" target="_blank" rel="noreferrer" class="footer-link-url">vladyslav-panasiuk →</a>
      </div>
      <div class="footer-link-item">
        <span class="footer-link-label">GitHub</span>
        <a href="https://github.com/th0truth" target="_blank" rel="noreferrer" class="footer-link-url">@th0truth →</a>
      </div>
      <div class="footer-link-item">
        <span class="footer-link-label">DEV Community</span>
        <a href="https://dev.to/th0truth" target="_blank" rel="noreferrer" class="footer-link-url">@th0truth →</a>
      </div>
    </div>
    <p class="footer-copy">© {new Date().getFullYear()} Vladyslav Panasiuk</p>
  </footer>

  <!-- Image overlay modal -->
  {#if activeImageOverlay}
    <div
      class="image-overlay"
      role="button"
      tabindex="0"
      onclick={() => activeImageOverlay = null}
      onkeydown={(e) => { if (e.key === 'Enter' || e.key === ' ' || e.key === 'Escape') activeImageOverlay = null; }}
    >
      <!-- svelte-ignore a11y_no_static_element_interactions -->
      <!-- svelte-ignore a11y_click_events_have_key_events -->
      <div class="overlay-content" onclick={(e) => e.stopPropagation()}>
        <img src={activeImageOverlay} alt="Enlarged screenshot" class="overlay-img" />
      </div>
      <button class="overlay-close" onclick={() => activeImageOverlay = null} aria-label="Close">✕</button>
    </div>
  {/if}

  <!-- Custom Animated Cursor -->
  {#if isCursorVisible}
    <div
      id="custom-cursor-dot"
      class="custom-cursor-dot {isHovered ? 'hovered' : ''} {isClicking ? 'clicking' : ''}"
    ></div>
    <div 
      id="custom-cursor-ring"
      class="custom-cursor-ring {isHovered ? 'hovered' : ''} {isClicking ? 'clicking' : ''}"
    ></div>
  {/if}
</main>

<style>
  /* ═══════════════════════════════════════════════════════
     CUSTOM ANIMATED CURSOR
     ═══════════════════════════════════════════════════════ */
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
    transition: width 0.2s ease, height 0.2s ease, top 0.2s ease, left 0.2s ease, background-color 0.2s ease, box-shadow 0.2s ease;
    box-shadow: 0 0 10px rgba(255, 255, 255, 0.8);
    will-change: transform;
  }

  .custom-cursor-ring {
    position: fixed;
    top: -19px;
    left: -19px;
    width: 38px;
    height: 38px;
    border: 1.5px solid rgba(255, 255, 255, 0.45);
    background-color: rgba(255, 255, 255, 0.02);
    border-radius: 50%;
    pointer-events: none;
    z-index: 99998;
    backdrop-filter: blur(1px);
    transition: width 0.22s cubic-bezier(0.16, 1, 0.3, 1), 
                height 0.22s cubic-bezier(0.16, 1, 0.3, 1), 
                top 0.22s cubic-bezier(0.16, 1, 0.3, 1), 
                left 0.22s cubic-bezier(0.16, 1, 0.3, 1), 
                border-color 0.2s ease, 
                background-color 0.2s ease,
                box-shadow 0.2s ease;
    box-shadow: 0 0 12px rgba(255, 255, 255, 0.08), inset 0 0 6px rgba(255, 255, 255, 0.03);
    will-change: transform;
  }

  /* Hover state over interactive elements (buttons, links, cards, tags) */
  .custom-cursor-dot.hovered {
    width: 6px;
    height: 6px;
    top: -3px;
    left: -3px;
    background-color: #ffffff;
    box-shadow: 0 0 10px rgba(255, 255, 255, 1);
  }

  .custom-cursor-ring.hovered {
    top: -22px;
    left: -22px;
    width: 44px;
    height: 44px;
    border-color: rgba(255, 255, 255, 0.9);
    background-color: rgba(255, 255, 255, 0.06);
    backdrop-filter: blur(1.5px);
    box-shadow: 0 0 15px rgba(255, 255, 255, 0.16), inset 0 0 8px rgba(255, 255, 255, 0.06);
  }

  /* Active Click State */
  .custom-cursor-dot.clicking {
    width: 4px;
    height: 4px;
    top: -2px;
    left: -2px;
  }

  .custom-cursor-ring.clicking {
    width: 28px;
    height: 28px;
    top: -14px;
    left: -14px;
    border-color: #ffffff;
    background-color: rgba(255, 255, 255, 0.15);
  }

  .custom-cursor-ring.hovered.clicking {
    width: 36px;
    height: 36px;
    top: -18px;
    left: -18px;
    border-color: #ffffff;
    background-color: rgba(255, 255, 255, 0.16);
  }

  @media (pointer: coarse) {
    .custom-cursor-dot, .custom-cursor-ring {
      display: none !important;
    }
  }

  /* ═══════════════════════════════════════════════════════
     LAYOUT
     ═══════════════════════════════════════════════════════ */
  .site-wrapper {
    width: 100%;
    max-width: 720px;
    margin: 0 auto;
    padding: 3rem 1.5rem 4rem;
    position: relative;
  }

  /* ═══════════════════════════════════════════════════════
     HEADER
     ═══════════════════════════════════════════════════════ */
  .site-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 4rem;
    padding-bottom: 1.5rem;
    border-bottom: 1px solid var(--border-color);
  }

  .header-brand {
    text-decoration: none;
  }

  .brand-name {
    font-family: var(--font-code);
    font-size: 1.15rem;
    font-weight: 500;
    color: var(--text-primary);
    letter-spacing: 0.02em;
  }

  .nav-desktop {
    display: flex;
    align-items: center;
    gap: 2rem;
  }

  .nav-link {
    font-family: var(--font-code);
    font-size: 0.95rem;
    color: var(--text-muted);
    text-decoration: none;
    transition: color 0.2s ease;
    position: relative;
    padding-bottom: 2px;
  }

  .nav-link::after {
    content: '';
    position: absolute;
    bottom: -2px;
    left: 0;
    width: 0;
    height: 1px;
    background: var(--text-secondary);
    transition: width 0.25s ease;
  }

  .nav-link:hover {
    color: var(--text-primary);
  }

  .nav-link:hover::after {
    width: 100%;
  }

  .nav-link.active {
    color: var(--text-primary);
  }

  .nav-link.active::after {
    width: 100%;
    background: var(--text-primary);
  }

  /* ═══════════════════════════════════════════════════════
     PROFILE
     ═══════════════════════════════════════════════════════ */
  .profile-intro {
    display: flex;
    align-items: center;
    gap: 1.75rem;
    margin-bottom: 2.75rem;
  }

  .avatar-radar-wrapper {
    position: relative;
    width: 114px;
    height: 114px;
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
    opacity: 0.55;
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
    stroke: rgba(255, 255, 255, 0.2);
    stroke-width: 1.1;
    transform-origin: center;
    transition: stroke 0.3s ease;
  }

  .avatar-radar-wrapper:hover .radar-ring {
    stroke: rgba(255, 255, 255, 0.45);
  }

  .outer-ring {
    animation: rotateOuter 24s linear infinite;
  }

  .inner-ring {
    stroke: rgba(255, 255, 255, 0.14);
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
    width: 88px;
    height: 88px;
    border-radius: 50%;
    overflow: hidden;
    border: 1px solid var(--border-color);
    background: #000;
    z-index: 1;
    transition: border-color 0.3s ease;
  }

  .avatar-radar-wrapper:hover .avatar-container {
    border-color: var(--border-hover);
  }

  .avatar-img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    filter: contrast(105%);
  }

  .profile-name {
    font-family: var(--font-body);
    font-size: 2.1rem;
    font-weight: 600;
    color: var(--text-primary);
    line-height: 1.18;
    letter-spacing: -0.015em;
  }

  .profile-tagline {
    font-family: var(--font-code);
    font-size: 1rem;
    color: var(--text-muted);
    margin-top: 0.3rem;
  }

  /* ═══════════════════════════════════════════════════════
     CONTENT SECTIONS
     ═══════════════════════════════════════════════════════ */
  .page-section {
    animation: pageIn 0.4s ease both;
  }

  @keyframes pageIn {
    from {
      opacity: 0;
      transform: translateY(12px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .page-section { animation: none; }
  }

  .page-title {
    font-family: var(--font-body);
    font-size: 1.6rem;
    font-weight: 600;
    color: var(--text-primary);
    margin-bottom: 2rem;
    letter-spacing: -0.015em;
    display: flex;
    align-items: center;
  }

  .section-block {
    margin-top: 3.5rem;
  }

  .section-heading {
    font-family: var(--font-body);
    font-size: 1.4rem;
    font-weight: 600;
    color: var(--text-primary);
    margin-bottom: 1.25rem;
    letter-spacing: -0.01em;
    display: flex;
    align-items: center;
  }

  .heading-hash {
    font-family: var(--font-code);
    color: var(--text-muted);
    font-weight: 400;
    font-size: 1.15em;
    margin-right: 0.15rem;
    opacity: 0.65;
  }

  .loading-text {
    font-family: var(--font-code);
    font-size: 0.95rem;
    color: var(--text-muted);
  }

  /* ═══════════════════════════════════════════════════════
     PROSE (Markdown content)
     ═══════════════════════════════════════════════════════ */
  .prose {
    color: var(--text-secondary);
    font-size: 1rem;
    line-height: 1.8;
  }

  :global(.prose p) {
    margin-bottom: 1rem;
  }

  :global(.prose p:last-child) {
    margin-bottom: 0;
  }

  :global(.prose ul) {
    margin-left: 1.25rem;
    margin-bottom: 1rem;
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
  }

  :global(.prose li) {
    padding-left: 0.15rem;
  }

  :global(.prose strong) {
    color: var(--text-primary);
    font-weight: 600;
  }

  :global(.prose a) {
    color: var(--text-primary);
    text-decoration: underline;
    text-underline-offset: 3px;
    text-decoration-color: var(--text-muted);
    transition: text-decoration-color 0.2s ease;
  }

  :global(.prose a:hover) {
    text-decoration-color: var(--text-primary);
  }

  :global(.prose h1) {
    color: var(--text-primary);
    font-size: 1.75rem;
    font-weight: 600;
    margin-bottom: 1rem;
    line-height: 1.3;
  }

  :global(.prose h2) {
    color: var(--text-primary);
    font-size: 1.3rem;
    font-weight: 600;
    margin-top: 1.75rem;
    margin-bottom: 0.75rem;
  }

  :global(.prose pre) {
    max-width: 100%;
    overflow-x: auto;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    border-radius: 6px;
    padding: 1rem 1.25rem;
    font-family: var(--font-code);
    font-size: 0.92rem;
    white-space: pre-wrap;
    word-break: break-all;
    margin-bottom: 1rem;
  }

  :global(.prose code) {
    font-family: var(--font-code);
    font-size: 0.88em;
    background: rgba(255, 255, 255, 0.04);
    padding: 0.15em 0.4em;
    border-radius: 3px;
  }

  :global(.prose img) {
    max-width: 100%;
    height: auto;
    border-radius: 4px;
  }

  :global(.prose table) {
    width: 100%;
    display: block;
    overflow-x: auto;
    border-collapse: collapse;
  }

  .exp-prose {
    margin-top: 0.5rem;
    font-size: 1rem;
  }

  /* ═══════════════════════════════════════════════════════
     SKILLS
     ═══════════════════════════════════════════════════════ */
  .skills-grid {
    display: flex;
    flex-direction: column;
    gap: 0.15rem;
  }

  .skill-category {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
    padding: 0.75rem 0.85rem;
    margin: 0 -0.85rem;
    border-radius: 6px;
    border-bottom: 1px solid rgba(255, 255, 255, 0.04);
    transition: background 0.2s ease;
  }

  .skill-category:last-child {
    border-bottom: none;
  }

  .skill-category:hover {
    background: rgba(255, 255, 255, 0.03);
  }

  .skill-category:hover .skill-cat-title {
    color: #f3ece2;
  }

  .skill-category:hover .skill-cat-content {
    border-left-color: rgba(255, 255, 255, 0.22);
  }

  .skill-category:hover .skill-subitems {
    color: var(--text-primary);
  }

  .skill-cat-title {
    font-family: var(--font-code);
    font-size: 1rem;
    font-weight: 600;
    color: var(--text-primary);
    letter-spacing: 0.01em;
    transition: color 0.2s ease;
    line-height: 1.3;
  }

  .skill-cat-content {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    padding: 0.05rem 0 0.05rem 0.85rem;
    margin-left: 0.1rem;
    border-left: 2px solid rgba(255, 255, 255, 0.08);
    transition: border-color 0.25s ease;
  }

  .skill-subline {
    display: flex;
    align-items: baseline;
    gap: 0.5rem;
    font-size: 1rem;
    line-height: 1.5;
    flex-wrap: wrap;
  }

  .skill-sublabel {
    font-family: var(--font-code);
    font-size: 0.92rem;
    color: var(--text-muted);
    font-weight: 500;
    flex-shrink: 0;
  }

  .skill-subitems {
    color: var(--text-secondary);
    transition: color 0.2s ease;
  }

  /* ═══════════════════════════════════════════════════════
     PREVIEW CARDS (Home page)
     ═══════════════════════════════════════════════════════ */
  .preview-card {
    display: block;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    padding: 1.35rem 1.45rem;
    margin-bottom: 0.85rem;
    text-decoration: none;
    color: inherit;
    transition: border-color 0.2s ease, background 0.2s ease;
    position: relative;
  }

  .preview-card:hover {
    border-color: var(--border-hover);
    background: var(--bg-card-hover);
  }

  .preview-card-top {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 0.75rem;
    margin-bottom: 0.5rem;
  }

  .preview-card-left {
    display: flex;
    align-items: center;
    gap: 0.65rem;
    flex-wrap: wrap;
  }

  .preview-role {
    font-weight: 600;
    font-size: 1.12rem;
    color: var(--text-primary);
  }

  .preview-badge {
    font-family: var(--font-code);
    font-size: 0.82rem;
    color: var(--text-muted);
    border: 1px solid var(--border-color);
    padding: 0.12rem 0.5rem;
    border-radius: 3px;
  }

  .preview-period {
    font-family: var(--font-code);
    font-size: 0.85rem;
    color: var(--text-muted);
    white-space: nowrap;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }

  .preview-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
  }

  .card-arrow {
    position: absolute;
    right: 1.35rem;
    bottom: 1.35rem;
    font-family: var(--font-code);
    font-size: 0.95rem;
    color: var(--text-muted);
    transition: color 0.2s ease, transform 0.2s ease;
  }

  .preview-card:hover .card-arrow,
  .featured-row:hover .card-arrow,
  .project-card:hover .card-arrow {
    color: var(--text-primary);
    transform: translateX(3px);
  }

  /* ═══════════════════════════════════════════════════════
     FEATURED PROJECTS
     ═══════════════════════════════════════════════════════ */
  .featured-list {
    display: flex;
    flex-direction: column;
    gap: 0.85rem;
  }

  .featured-row {
    display: block;
    width: 100%;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    padding: 1.35rem 1.45rem;
    text-align: left;
    transition: border-color 0.2s ease, background 0.2s ease;
    position: relative;
    cursor: pointer;
  }

  .featured-row:hover {
    border-color: var(--border-hover);
    background: var(--bg-card-hover);
  }

  .featured-info {
    display: flex;
    align-items: center;
    gap: 0.65rem;
    margin-bottom: 0.4rem;
    flex-wrap: wrap;
  }

  .featured-name {
    font-family: var(--font-code);
    font-size: 1.08rem;
    font-weight: 600;
    color: var(--text-primary);
    overflow-wrap: break-word;
    word-break: break-word;
  }

  .lang-dot {
    font-family: var(--font-code);
    font-size: 0.82rem;
    color: var(--text-muted);
    border: 1px solid var(--border-color);
    padding: 0.1rem 0.45rem;
    border-radius: 3px;
    white-space: nowrap;
    flex-shrink: 0;
  }

  .featured-desc {
    font-size: 0.98rem;
    color: var(--text-secondary);
    line-height: 1.6;
    padding-right: 2rem;
  }

  .view-all-link {
    display: inline-block;
    font-family: var(--font-code);
    font-size: 0.95rem;
    color: var(--text-muted);
    margin-top: 1.15rem;
    text-decoration: none;
    transition: color 0.2s ease;
  }

  .view-all-link:hover {
    color: var(--text-primary);
  }

  /* ═══════════════════════════════════════════════════════
     EXPERIENCE PAGE
     ═══════════════════════════════════════════════════════ */
  .exp-list {
    display: flex;
    flex-direction: column;
    gap: 2rem;
  }

  .exp-card {
    border-bottom: 1px solid var(--border-color);
    padding-bottom: 2rem;
  }

  .exp-card:last-child {
    border-bottom: none;
    padding-bottom: 0;
  }

  .exp-top-row {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 1rem;
    margin-bottom: 0.6rem;
  }

  .exp-title-group {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    flex-wrap: wrap;
  }

  .exp-role {
    font-size: 1.2rem;
    font-weight: 600;
    color: var(--text-primary);
  }

  .exp-type-badge {
    font-family: var(--font-code);
    font-size: 0.82rem;
    color: var(--text-muted);
    border: 1px solid var(--border-color);
    padding: 0.12rem 0.5rem;
    border-radius: 3px;
  }

  .exp-period {
    font-family: var(--font-code);
    font-size: 0.85rem;
    color: var(--text-muted);
    white-space: nowrap;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }

  .exp-meta {
    font-size: 0.98rem;
    color: var(--text-secondary);
    display: flex;
    flex-wrap: wrap;
    gap: 0.4rem;
    align-items: center;
    margin-bottom: 0.3rem;
  }

  .meta-key {
    font-weight: 600;
    color: var(--text-primary);
  }

  .meta-link {
    font-family: var(--font-code);
    font-size: 0.98rem;
    color: var(--text-primary);
    text-decoration: underline;
    text-underline-offset: 3px;
    text-decoration-color: var(--text-muted);
  }

  .meta-link:hover {
    text-decoration-color: var(--text-primary);
  }

  .meta-note {
    color: var(--text-muted);
    font-style: italic;
    font-size: 0.92rem;
  }

  .meta-val {
    color: var(--text-secondary);
    font-size: 0.98rem;
  }

  /* ═══════════════════════════════════════════════════════
     PROJECTS PAGE
     ═══════════════════════════════════════════════════════ */
  .filter-bar {
    display: flex;
    flex-wrap: wrap;
    gap: 0.45rem;
    margin-bottom: 1.75rem;
    padding-bottom: 1rem;
    border-bottom: 1px solid var(--border-color);
  }

  .filter-btn {
    font-family: var(--font-code);
    font-size: 0.88rem;
    padding: 0.35rem 0.8rem;
    border-radius: 4px;
    border: 1px solid var(--border-color);
    background: transparent;
    color: var(--text-muted);
    transition: all 0.15s ease;
    cursor: pointer;
  }

  .filter-btn:hover {
    color: var(--text-secondary);
    border-color: var(--border-hover);
  }

  .filter-btn.active {
    color: var(--text-primary);
    border-color: var(--text-secondary);
    background: rgba(255, 255, 255, 0.04);
  }

  .project-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 0.85rem;
  }

  .project-card {
    display: flex;
    flex-direction: column;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    padding: 1.35rem 1.45rem;
    text-align: left;
    transition: border-color 0.2s ease, background 0.2s ease;
    cursor: pointer;
    position: relative;
    min-height: 160px;
  }

  .project-card:hover {
    border-color: var(--border-hover);
    background: var(--bg-card-hover);
  }

  .card-top {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 0.5rem;
    gap: 0.5rem;
  }

  .card-name {
    font-family: var(--font-code);
    font-size: 1.05rem;
    font-weight: 600;
    color: var(--text-primary);
    overflow-wrap: break-word;
    word-break: break-word;
  }

  .card-stars {
    font-family: var(--font-code);
    font-size: 0.85rem;
    color: var(--text-muted);
    flex-shrink: 0;
  }

  .card-desc {
    font-size: 0.98rem;
    color: var(--text-secondary);
    line-height: 1.6;
    margin-bottom: 0.85rem;
    padding-right: 1.5rem;
  }

  .card-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
    margin-top: auto;
  }

  /* ═══════════════════════════════════════════════════════
     PILLS / TAGS
     ═══════════════════════════════════════════════════════ */
  .pill {
    font-family: var(--font-code);
    font-size: 0.8rem;
    color: var(--text-muted);
    border: 1px solid var(--border-color);
    padding: 0.12rem 0.48rem;
    border-radius: 3px;
    white-space: nowrap;
  }

  .pill.primary {
    color: var(--text-secondary);
  }

  /* ═══════════════════════════════════════════════════════
     PROJECT DETAIL
     ═══════════════════════════════════════════════════════ */
  .detail-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 2rem;
    font-family: var(--font-code);
    font-size: 0.95rem;
  }

  .back-link {
    color: var(--text-muted);
    font-family: var(--font-code);
    font-size: 0.95rem;
    cursor: pointer;
    transition: color 0.2s ease;
    background: none;
    border: none;
  }

  .back-link:hover {
    color: var(--text-primary);
  }

  .github-link {
    font-family: var(--font-code);
    font-size: 0.92rem;
    color: var(--text-secondary);
    border: 1px solid var(--border-color);
    padding: 0.4rem 0.85rem;
    border-radius: 5px;
    text-decoration: none;
    transition: border-color 0.2s ease, color 0.2s ease;
  }

  .github-link:hover {
    border-color: var(--border-hover);
    color: var(--text-primary);
  }

  /* ═══════════════════════════════════════════════════════
     SCREENSHOT GALLERY (from markdown content)
     ═══════════════════════════════════════════════════════ */
  :global(.screenshots-group) {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.85rem;
    margin: 1.25rem 0 2rem 0;
  }

  :global(.project-logo-wrapper) {
    display: inline-flex !important;
    margin-bottom: 1.5rem !important;
    padding: 0 !important;
    background: none !important;
    border: none !important;
    box-shadow: none !important;
    cursor: pointer;
  }

  :global(.hwmonitor-logo-img) {
    height: 160px !important;
    width: auto !important;
    max-width: 100% !important;
    object-fit: contain !important;
    opacity: 0.85;
    transition: opacity 0.3s ease;
  }

  :global(.project-logo-wrapper:hover .hwmonitor-logo-img) {
    opacity: 1;
  }

  :global(.screenshot-item) {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    padding: 0.4rem;
    border-radius: 5px;
    overflow: hidden;
    cursor: pointer;
    transition: border-color 0.2s ease;
  }

  :global(.screenshot-item:hover) {
    border-color: var(--border-hover);
  }

  :global(.screenshot-item img) {
    width: 100%;
    height: 105px;
    object-fit: cover;
    border-radius: 3px;
  }

  :global(.screenshot-item figcaption) {
    font-family: var(--font-code);
    font-size: 0.8rem;
    color: var(--text-muted);
    text-align: center;
    padding-top: 0.15rem;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  :global(.screenshot-item:hover figcaption) {
    color: var(--text-secondary);
  }

  /* ═══════════════════════════════════════════════════════
     IMAGE OVERLAY
     ═══════════════════════════════════════════════════════ */
  .image-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.88);
    backdrop-filter: blur(8px);
    z-index: 1000;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 2rem;
    animation: fadeOverlay 0.2s ease;
  }

  @keyframes fadeOverlay {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  .overlay-content {
    max-width: 90vw;
    max-height: 85vh;
    border-radius: 6px;
    overflow: hidden;
  }

  .overlay-img {
    max-width: 100%;
    max-height: 85vh;
    object-fit: contain;
    border-radius: 4px;
  }

  .overlay-close {
    position: absolute;
    top: 1.5rem;
    right: 1.5rem;
    background: rgba(255, 255, 255, 0.08);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: var(--text-primary);
    border-radius: 50%;
    width: 38px;
    height: 38px;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 0.92rem;
    cursor: pointer;
    transition: background 0.2s ease;
  }

  .overlay-close:hover {
    background: rgba(255, 255, 255, 0.15);
  }

  /* ═══════════════════════════════════════════════════════
     DIVIDER & FOOTER
     ═══════════════════════════════════════════════════════ */
  .site-divider {
    border: none;
    border-top: 1px solid var(--border-color);
    margin: 4rem 0 2.5rem 0;
  }

  .footer-heading {
    font-family: var(--font-body);
    font-size: 1.35rem;
    font-weight: 600;
    color: var(--text-primary);
    margin-bottom: 1.5rem;
    display: flex;
    align-items: center;
  }

  .footer-links {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 1.75rem;
    margin-bottom: 2.5rem;
  }

  .footer-link-item {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .footer-link-label {
    font-size: 1rem;
    color: var(--text-primary);
  }

  .footer-link-url {
    font-family: var(--font-code);
    font-size: 0.9rem;
    color: var(--text-muted);
    text-decoration: none;
    transition: color 0.2s ease;
  }

  .footer-link-url:hover {
    color: var(--text-secondary);
  }

  .footer-copy {
    font-family: var(--font-code);
    font-size: 0.85rem;
    color: var(--text-muted);
  }

  /* ═══════════════════════════════════════════════════════
     MOBILE HAMBURGER
     ═══════════════════════════════════════════════════════ */
  .hamburger-btn {
    display: none;
    flex-direction: column;
    justify-content: center;
    gap: 5px;
    width: 34px;
    height: 34px;
    background: none;
    border: 1px solid var(--border-color);
    border-radius: 5px;
    cursor: pointer;
    padding: 7px 6px;
    transition: border-color 0.15s ease;
  }

  .hamburger-btn:hover {
    border-color: var(--border-hover);
  }

  .hamburger-line {
    display: block;
    width: 100%;
    height: 1.5px;
    background: var(--text-secondary);
    border-radius: 1px;
    transition: transform 0.25s ease, opacity 0.2s ease;
    transform-origin: center;
  }

  .hamburger-line.open:nth-child(1) {
    transform: translateY(6.5px) rotate(45deg);
  }
  .hamburger-line.open:nth-child(2) {
    opacity: 0;
    transform: scaleX(0);
  }
  .hamburger-line.open:nth-child(3) {
    transform: translateY(-6.5px) rotate(-45deg);
  }

  .mobile-menu {
    position: absolute;
    top: 4.5rem;
    left: 1rem;
    right: 1rem;
    background: rgba(25, 25, 25, 0.98);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    border: 1px solid var(--border-color);
    border-radius: 8px;
    padding: 0.5rem;
    display: flex;
    flex-direction: column;
    gap: 0.2rem;
    z-index: 999;
    box-shadow: 0 12px 32px rgba(0, 0, 0, 0.5);
  }

  .mobile-nav-link {
    display: flex;
    align-items: center;
    padding: 0.75rem 1rem;
    border-radius: 6px;
    color: var(--text-muted);
    text-decoration: none;
    font-family: var(--font-code);
    font-size: 0.92rem;
    transition: background 0.15s ease, color 0.15s ease;
  }

  .mobile-nav-link.active {
    color: var(--text-primary);
    background: rgba(255, 255, 255, 0.04);
  }

  .mobile-nav-link:hover {
    background: rgba(255, 255, 255, 0.05);
    color: var(--text-primary);
  }

  /* ═══════════════════════════════════════════════════════
     RESPONSIVE
     ═══════════════════════════════════════════════════════ */
  @media (max-width: 768px) {
    .site-wrapper {
      padding: 2rem 1.25rem 3rem;
    }

    .site-header {
      margin-bottom: 2.5rem;
    }

    .nav-desktop {
      display: none;
    }

    .hamburger-btn {
      display: flex;
    }

    .profile-intro {
      flex-direction: column;
      align-items: center;
      text-align: center;
      gap: 1.25rem;
    }

    .profile-text {
      display: flex;
      flex-direction: column;
      align-items: center;
      text-align: center;
    }

    .avatar-radar-wrapper {
      width: 140px;
      height: 140px;
    }

    .avatar-container {
      width: 108px;
      height: 108px;
    }

    .profile-name {
      font-size: 1.8rem;
      text-align: center;
    }

    .profile-tagline {
      text-align: center;
    }

    .project-grid {
      grid-template-columns: 1fr;
    }

    .preview-card-top {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.45rem;
      margin-bottom: 0.65rem;
    }

    .preview-card-left {
      display: flex;
      align-items: center;
      gap: 0.6rem;
      width: 100%;
      flex-wrap: wrap;
    }

    .preview-role {
      font-size: 1.15rem;
      white-space: nowrap;
    }

    .preview-tags {
      padding-right: 1.75rem;
    }

    .preview-tags .pill:nth-child(n+5) {
      display: none;
    }

    .exp-top-row {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.4rem;
    }

    .exp-title-group {
      display: flex;
      align-items: center;
      gap: 0.6rem;
      flex-wrap: wrap;
    }

    .footer-links {
      grid-template-columns: 1fr;
      gap: 1.25rem;
    }

    .section-block {
      margin-top: 2.75rem;
    }

    :global(.project-logo-wrapper) {
      display: flex !important;
      justify-content: center !important;
      width: 100% !important;
      margin: 0 auto 1.5rem auto !important;
    }

    :global(.detail-prose h1) {
      text-align: center;
    }
  }

  @media (max-width: 480px) {
    .site-wrapper {
      padding: 1.5rem 1rem 2.5rem;
    }

    .profile-name {
      font-size: 1.65rem;
    }

    .avatar-radar-wrapper {
      width: 148px;
      height: 148px;
    }

    .avatar-container {
      width: 114px;
      height: 114px;
    }

    .preview-card {
      padding: 1.15rem 1.25rem;
    }

    .preview-role {
      font-size: 1.1rem;
      white-space: normal;
    }

    .preview-period {
      font-size: 0.8rem;
    }

    .preview-tags .pill:nth-child(n+4) {
      display: none;
    }

    .exp-role {
      font-size: 1.15rem;
    }

    .exp-period {
      font-size: 0.8rem;
    }

    .featured-row {
      padding: 1.15rem 1.25rem;
    }

    .featured-name {
      font-size: 1rem;
    }

    :global(.screenshots-group) {
      grid-template-columns: 1fr;
    }

    .skill-cat-title {
      font-size: 0.95rem;
    }

    .skill-subline {
      font-size: 0.95rem;
    }

    .detail-bar {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.75rem;
    }

    :global(.project-logo-wrapper) {
      display: flex !important;
      justify-content: center !important;
      width: 100% !important;
      margin: 0 auto 1.25rem auto !important;
    }

    :global(.detail-prose h1) {
      text-align: center;
    }
  }
</style>
