<script lang="ts">
  import { onMount } from 'svelte';
  import { fade } from 'svelte/transition';
  import { parse } from 'marked';
  import DOMPurify from 'dompurify';
  import { Home, Briefcase, FolderGit2, ArrowLeft, ExternalLink, Code2, Clock, X, ChevronRight, Cpu, GraduationCap, Database, Eye, Tv, Binary, CircuitBoard, BarChart3 } from '@lucide/svelte';

  const projectIconsMap: Record<string, any> = {
    Cpu,
    GraduationCap,
    Database,
    Eye,
    Tv,
    Binary,
    CircuitBoard,
    BarChart3
  };

  // Navigation Items
  const navItems = [
    { key: 'h', label: 'home', id: 'home', hash: '#/', icon: Home },
    { key: 'e', label: 'experience', id: 'experience', hash: '#/experience', icon: Briefcase },
    { key: 'p', label: 'projects', id: 'projects', hash: '#/projects', icon: FolderGit2 }
  ];

  let currentTab = $state('home');
  let currentTime = $state('');
  let mobileMenuOpen = $state(false);

  function closeMobileMenu() {
    mobileMenuOpen = false;
  }

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

  function getProjectIcon(iconName?: string) {
    if (iconName && projectIconsMap[iconName]) {
      return projectIconsMap[iconName];
    }
    return Code2;
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

  // Derived: deduplicated tech stack for home page display
  let uniqueTechs = $derived.by(() => {
    const seen = new Map<string, string>();
    for (const exp of experiences ?? []) {
      for (const t of exp?.techStack ?? []) {
        const key = t.toLowerCase();
        if (!seen.has(key) || t !== t.toLowerCase()) seen.set(key, t);
      }
    }
    for (const p of projects ?? []) {
      if (p?.language) {
        const key = p.language.toLowerCase();
        if (!seen.has(key) || p.language !== p.language.toLowerCase()) seen.set(key, p.language);
      }
      for (const t of p?.tags ?? []) {
        const key = t.toLowerCase();
        if (!seen.has(key)) seen.set(key, t);
      }
    }
    return [...seen.values()].sort((a, b) => a.toLowerCase().localeCompare(b.toLowerCase()));
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

  // Brute Force Matrix / Decrypt Scramble Text Action
  const GLYPHS = 'abcdefghijklmnopqrstuvwxyz0123456789+-_/~!#$&';

  function scramble(node: HTMLElement, targetText?: string) {
    const finalTarget = (targetText || node.getAttribute('data-text') || node.textContent || '').trim();
    node.setAttribute('data-text', finalTarget);
    let iteration = 0;
    let frameId: number;
    let lastTime = performance.now();
    const totalChars = finalTarget.length;
    const intervalMs = 32;

    function step(now: number) {
      if (now - lastTime >= intervalMs) {
        lastTime = now;
        const currentProgress = Math.floor(iteration);

        node.innerText = finalTarget
          .split('')
          .map((char, index) => {
            if (char === ' ') return ' ';
            if (index < currentProgress) {
              return finalTarget[index];
            }
            if (index === currentProgress && Math.random() > 0.45) {
              return finalTarget[index];
            }
            return GLYPHS[Math.floor(Math.random() * GLYPHS.length)];
          })
          .join('');

        iteration += 0.35;
      }

      if (iteration <= totalChars + 1) {
        frameId = requestAnimationFrame(step);
      } else {
        node.innerText = finalTarget;
      }
    }

    frameId = requestAnimationFrame(step);

    return {
      update(newTarget: string) {
        cancelAnimationFrame(frameId);
        iteration = 0;
        node.setAttribute('data-text', newTarget);
        frameId = requestAnimationFrame(step);
      },
      destroy() {
        cancelAnimationFrame(frameId);
      }
    };
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

  function getProjectTags(project: ProjectMeta): string[] {
    if (!project || !project.tags) return [];
    const lang = (project.language || '').toLowerCase().trim();
    const seen = new Set<string>();
    if (lang) {
      seen.add(lang);
      if (lang === 'c++') {
        seen.add('cpp');
        seen.add('cpp17');
        seen.add('cpp20');
        seen.add('c++17');
        seen.add('c++20');
      } else if (lang === 'c') {
        seen.add('c');
      } else if (lang === 'python') {
        seen.add('python');
        seen.add('py');
      } else if (lang === 'shell') {
        seen.add('shell');
      }
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

  // Interactive Custom Cursor State (optimized for 60-120fps smooth performance)
  let isHovered = $state(false);
  let isClicking = $state(false);
  let cursorVisible = $state(false);

  let targetX = -100;
  let targetY = -100;
  let currentCursorX = -100;
  let currentCursorY = -100;
  let currentRingX = -100;
  let currentRingY = -100;
  let animFrameId: number;

  function updateCursorPos(e: MouseEvent) {
    if (!cursorVisible) cursorVisible = true;
    targetX = e.clientX;
    targetY = e.clientY;
    currentCursorX = e.clientX;
    currentCursorY = e.clientY;

    const dotEl = document.getElementById('custom-cursor-dot');
    if (dotEl) {
      dotEl.style.transform = `translate3d(${currentCursorX}px, ${currentCursorY}px, 0)`;
    }

    const target = e.target as HTMLElement;
    if (target) {
      const interactive = target.closest('a, button, input, textarea, [data-src], .project-card, .screenshot-item, .nav-item');
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
    cursorVisible = false;
  }

  function animateRing() {
    currentRingX += (targetX - currentRingX) * 0.22;
    currentRingY += (targetY - currentRingY) * 0.22;
    const ringEl = document.getElementById('custom-cursor-ring');
    if (ringEl) {
      ringEl.style.transform = `translate3d(${currentRingX}px, ${currentRingY}px, 0)`;
    }
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
    window.addEventListener('click', handleContainerClick);
    document.addEventListener('mouseleave', handleMouseLeave);

    return () => {
      clearInterval(interval);
      cancelAnimationFrame(animFrameId);
      window.removeEventListener('keydown', handleKeydown);
      window.removeEventListener('hashchange', syncRouteFromHash);
      window.removeEventListener('mousemove', updateCursorPos);
      window.removeEventListener('mousedown', handleMouseDown);
      window.removeEventListener('mouseup', handleMouseUp);
      window.removeEventListener('click', handleContainerClick);
      document.removeEventListener('mouseleave', handleMouseLeave);
    };
  });
</script>

<main class="portfolio-container">
  <!-- Top Navigation Header -->
  <nav class="nav-bar">
    <!-- Desktop nav items -->
    <div class="nav-desktop">
      {#each navItems as item}
        {@const Icon = item.icon}
        <a
          href={item.hash}
          class="nav-item {currentTab === item.id && !selectedProject ? 'active' : ''}"
          onclick={closeMobileMenu}
        >
          <span class="nav-key">[{item.key}]</span>
          <Icon size={15} class="nav-icon" />
          <span class="nav-label">{item.label}</span>
        </a>
      {/each}
    </div>

    <!-- Mobile hamburger button -->
    <button
      class="hamburger-btn"
      onclick={() => mobileMenuOpen = !mobileMenuOpen}
      aria-label="Toggle navigation menu"
    >
      <span class="hamburger-line {mobileMenuOpen ? 'open' : ''}"></span>
      <span class="hamburger-line {mobileMenuOpen ? 'open' : ''}"></span>
      <span class="hamburger-line {mobileMenuOpen ? 'open' : ''}"></span>
    </button>
  </nav>

  <!-- Mobile dropdown menu -->
  {#if mobileMenuOpen}
    <div class="mobile-menu" in:fade={{ duration: 120 }}>
      {#each navItems as item}
        {@const Icon = item.icon}
        <a
          href={item.hash}
          class="mobile-nav-item {currentTab === item.id && !selectedProject ? 'active' : ''}"
          onclick={closeMobileMenu}
        >
          <Icon size={18} class="mobile-nav-icon" />
          <span>{item.label}</span>
        </a>
      {/each}
    </div>
  {/if}

  <!-- TAB CONTENT: HOME -->
  {#if currentTab === 'home' && !selectedProject}
    <section class="tab-content bio-section">
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
          <h1 class="profile-name">Vladyslav Panasiuk</h1>
          <p class="profile-handle">@th0truth · <span class="profile-role">software engineer</span></p>
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

      <!-- Work Experience Preview -->
      <div class="home-section-block" style="margin-top: 3.5rem;">
        <div class="section-title-wrapper">
          <span class="section-bar"></span>
          <h2>experience</h2>
        </div>
        {#if loadingExp}
          <p class="status-msg">Loading...</p>
        {:else if experiences.length > 0}
          {#each experiences as exp}
            <a href="#/experience" class="exp-preview-card">
              <div class="exp-header-row">
                <div class="exp-title-row">
                  <h3 class="exp-role">{exp.role}</h3>
                  <span class="badge-freelance">{exp.type}</span>
                </div>
                <div class="exp-period">{exp.period}</div>
              </div>
              {#if exp.techStack && exp.techStack.length > 0}
                <div class="exp-preview-tags">
                  {#each exp.techStack as tech}
                    <span class="tag-item">{tech}</span>
                  {/each}
                </div>
              {/if}
            </a>
          {/each}
          <a href="#/experience" class="show-more-link">
            Show More
            <ChevronRight size={14} />
          </a>
        {/if}
      </div>


      <!-- Featured Projects -->
      <div class="home-section-block">
        <div class="section-title-wrapper">
          <span class="section-bar"></span>
          <h2>selected projects</h2>
        </div>
        {#if loadingProjects}
          <p class="status-msg">Loading...</p>
        {:else if featuredProjects.length > 0}
          <div class="featured-project-list">
            {#each featuredProjects as project}
              {@const Icon = getProjectIcon(project.icon)}
              <button class="featured-project-row" onclick={() => openProjectDetail(project)}>
                <div class="row-left">
                  <div class="row-header">
                    <Icon size={16} class="card-title-icon" />
                    <span class="row-title">{project.name}</span>
                    {#if project.language}
                      <span class="tag-primary compact">{project.language}</span>
                    {/if}
                  </div>
                  <p class="row-desc">{project.description}</p>
                </div>
                <div class="row-right">
                  <ChevronRight size={16} class="row-arrow" />
                </div>
              </button>
            {/each}
          </div>
          <a href="#/projects" class="show-more-link">
            View all projects
            <ChevronRight size={14} />
          </a>
        {/if}
      </div>
    </section>



  {:else if currentTab === 'experience' && !selectedProject}
    <section class="tab-content">
      <div class="section-title-wrapper">
        <span class="section-bar"></span>
        <h2 use:scramble={'experience'} onmouseenter={(e) => scramble(e.currentTarget)}>experience</h2>
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
                <div class="exp-header-row">
                  <div class="exp-title-row">
                    <h3 class="exp-role">{exp.role}</h3>
                    <span class="badge-freelance">{exp.type}</span>
                  </div>
                  <div class="exp-period">{exp.period}</div>
                </div>

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
        <span class="section-bar"></span>
        <h2 use:scramble={'projects'} onmouseenter={(e) => scramble(e.currentTarget)}>projects</h2>
      </div>
      
      {#if loadingProjects}
        <p class="status-msg">Loading selected projects...</p>
      {:else if projects.length === 0}
        <p class="status-msg">No selected projects listed.</p>
      {:else}
        <!-- Language Switcher/Filter -->
        <div class="lang-filter-container">
          {#each availableLanguages as lang}
            <button
              class="lang-filter-btn {selectedLanguageFilter === lang ? 'active' : ''}"
              onclick={() => selectedLanguageFilter = lang}
            >
              {lang}
            </button>
          {/each}
        </div>

        <div class="project-grid">
          {#each filteredProjects as project}
            {@const Icon = getProjectIcon(project.icon)}
            <button class="project-card" onclick={() => openProjectDetail(project)}>
              <div class="card-header">
                <div class="card-title-group">
                  <Icon size={16} class="card-title-icon" />
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
                {#each getProjectTags(project) as tag}
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
      <h2 class="footer-title" onmouseenter={(e) => { const s = e.currentTarget.querySelector('.scramble-target'); if (s) scramble(s as HTMLElement); }}>
        <span class="section-bar"></span>
        <span class="scramble-target" use:scramble={'find me here ~'}>find me here ~</span>
      </h2>
      <div class="time-wrapper">
        <Clock size={14} class="clock-icon" />
        <span class="time-display">{currentTime}</span>
      </div>
    </div>

    <div class="links-grid">
      <div class="link-item">
        <span class="link-label">LinkedIn</span>
        <a href="https://www.linkedin.com/in/vladyslav-panasiuk-481582370" target="_blank" rel="noreferrer" class="link-url">
          <svg viewBox="0 0 24 24" width="15" height="15" fill="currentColor">
            <path d="M19 3a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h14m-.5 15.5v-5.3a3.26 3.26 0 0 0-3.26-3.26c-.85 0-1.84.52-2.28 1.3v-1.11h-2.79v8.37h2.79v-4.93c0-.77.62-1.4 1.39-1.4a1.4 1.4 0 0 1 1.4 1.4v4.93h2.75M6.46 10.9v8.37H9.25V10.9H6.46M7.86 6.6a1.4 1.4 0 1 0 1.4 1.4 1.4 1.4 0 0 0-1.4-1.4z"/>
          </svg>
          <span>vladyslav-panasiuk</span>
        </a>
      </div>
      <div class="link-item">
        <span class="link-label">GitHub</span>
        <a href="https://github.com/th0truth" target="_blank" rel="noreferrer" class="link-url">
          <svg viewBox="0 0 24 24" width="15" height="15" fill="currentColor">
            <path d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.905 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57A12.02 12.02 0 0024 12c0-6.63-5.37-12-12-12z"/>
          </svg>
          <span>@th0truth</span>
        </a>
      </div>
      <div class="link-item">
        <span class="link-label">DEV Community</span>
        <a href="https://dev.to/th0truth" target="_blank" rel="noreferrer" class="link-url">
          <svg viewBox="0 0 448 512" width="15" height="15" fill="currentColor">
            <path d="M120.12 208.29c-3.88-2.9-7.77-4.35-11.65-4.35H91.03v104.47h17.45c3.88 0 7.77-1.45 11.65-4.35 3.88-2.9 5.82-7.25 5.82-13.06v-69.65c.01-5.8-1.93-10.16-5.83-13.06zM400 32H48C21.5 32 0 53.5 0 80v352c0 26.5 21.5 48 48 48h352c26.5 0 48-21.5 48-48V80c0-26.5-21.5-48-48-48zM143.67 296.8c-7.2 9.68-18.06 14.52-32.57 14.52H67.89V200.73h43.21c14.51 0 25.37 4.84 32.57 14.52 7.2 9.68 10.8 22.01 10.8 37.02.01 15-3.59 27.34-10.8 37.02zm86.2-70.18h-41.69v30.48h37.08v24.16h-37.08v30.48h41.69v24.16h-69.96V200.73h69.96v25.89zm101.44 87.81c-15.65 0-25.37-12.58-29.23-37.75l-12.1-76.45h27.98l7.98 56.52c1.78 12.58 4.77 24.16 8.98 24.16 4.2 0 7.2-11.58 8.98-24.16l7.98-56.52h27.98l-12.1 76.45c-3.86 25.17-13.58 37.75-29.23 37.75z"/>
          </svg>
          <span>@th0truth</span>
        </a>
      </div>
    </div>
  </footer>

  <!-- FULLSCREEN IMAGE OVERLAY MODAL -->
  {#if activeImageOverlay}
    <div
      class="image-modal-overlay"
      role="button"
      tabindex="0"
      onclick={() => activeImageOverlay = null}
      onkeydown={(e) => { if (e.key === 'Enter' || e.key === ' ' || e.key === 'Escape') activeImageOverlay = null; }}
    >
      <button class="modal-close-btn" onclick={() => activeImageOverlay = null} aria-label="Close enlarged image">
        <X size={20} />
      </button>
      <!-- svelte-ignore a11y_no_static_element_interactions -->
      <!-- svelte-ignore a11y_click_events_have_key_events -->
      <div class="modal-content-wrapper" onclick={(e) => e.stopPropagation()}>
        <img src={activeImageOverlay} alt="Enlarged screenshot preview" class="modal-img" />
      </div>
    </div>
  {/if}

  <!-- CUSTOM ANIMATED CURSOR -->
  {#if cursorVisible}
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
    width: 100%;
    max-width: 760px;
    margin: 0 auto;
    box-sizing: border-box;
    padding: 0 1.25rem;
    position: relative;
    overflow-x: hidden;
  }

  /* Navigation Header at Top */
  .nav-bar {
    display: flex;
    justify-content: flex-start;
    align-items: center;
    margin-bottom: 3.5rem;
    font-family: var(--font-mono);
    font-size: 0.95rem;
    width: 100%;
  }

  .nav-desktop {
    display: flex;
    align-items: center;
    gap: 1.75rem;
    flex-wrap: wrap;
  }

  .nav-item {
    position: relative;
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    color: var(--text-muted);
    transition: color 0.15s ease;
    text-decoration: none !important;
    padding-bottom: 4px;
  }

  .nav-item::after {
    content: '';
    position: absolute;
    bottom: 0;
    right: 0;
    width: 100%;
    height: 1px;
    background: #ffffff;
    opacity: 0;
    transform: scaleX(0);
    transform-origin: right;
    transition: opacity 0.2s ease, transform 0.2s cubic-bezier(0.16, 1, 0.3, 1);
  }

  .nav-key {
    color: var(--text-muted);
  }

  @media (max-width: 768px), (pointer: coarse) {
    .nav-key {
      display: none !important;
    }
  }

  :global(.nav-icon) {
    opacity: 0.6;
    transition: opacity 0.15s ease, color 0.15s ease;
  }

  .nav-item.active :global(.nav-icon),
  .nav-item:hover :global(.nav-icon) {
    opacity: 1;
    color: #ffffff !important;
  }

  .nav-item.active .nav-label {
    color: var(--text-primary);
  }

  .nav-item.active::after,
  .nav-item:hover::after {
    opacity: 1;
    transform: scaleX(1);
  }

  .nav-item:hover {
    color: var(--text-primary);
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

  .profile-handle {
    font-family: var(--font-mono);
    color: var(--text-secondary);
    font-size: 0.95rem;
    margin-top: 0.35rem;
  }

  .profile-role {
    font-family: var(--font-mono);
    color: var(--text-secondary);
    font-size: 0.95rem;
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
    animation: nodePulse 3.5s ease-in-out infinite;
  }

  .timeline-vertical-line {
    position: absolute;
    top: 0.35rem;
    bottom: -2.25rem;
    left: 8px;
    width: 2px;
    background: linear-gradient(180deg, #52525b 0%, #27272a 100%);
    z-index: 1;
    overflow: hidden;
  }

  .timeline-vertical-line::after {
    content: '';
    position: absolute;
    top: -100%;
    left: 0;
    width: 100%;
    height: 100%;
    background: linear-gradient(180deg, transparent 0%, rgba(255, 255, 255, 0.35) 50%, transparent 100%);
    animation: lineLightPulse 3s cubic-bezier(0.4, 0, 0.6, 1) infinite;
  }

  .timeline-item:nth-child(1) .timeline-vertical-line::after,
  .timeline-item:nth-child(1) .timeline-node { animation-delay: 0s; }
  .timeline-item:nth-child(2) .timeline-vertical-line::after,
  .timeline-item:nth-child(2) .timeline-node { animation-delay: 0.3s; }
  .timeline-item:nth-child(3) .timeline-vertical-line::after,
  .timeline-item:nth-child(3) .timeline-node { animation-delay: 0.6s; }
  .timeline-item:nth-child(4) .timeline-vertical-line::after,
  .timeline-item:nth-child(4) .timeline-node { animation-delay: 0.9s; }
  .timeline-item:nth-child(5) .timeline-vertical-line::after,
  .timeline-item:nth-child(5) .timeline-node { animation-delay: 1.2s; }
  .timeline-item:nth-child(6) .timeline-vertical-line::after,
  .timeline-item:nth-child(6) .timeline-node { animation-delay: 1.5s; }
  .timeline-item:nth-child(7) .timeline-vertical-line::after,
  .timeline-item:nth-child(7) .timeline-node { animation-delay: 1.8s; }
  .timeline-item:nth-child(8) .timeline-vertical-line::after,
  .timeline-item:nth-child(8) .timeline-node { animation-delay: 2.1s; }
  .timeline-item:nth-child(9) .timeline-vertical-line::after,
  .timeline-item:nth-child(9) .timeline-node { animation-delay: 2.4s; }

  @keyframes lineLightPulse {
    0% {
      top: -100%;
      opacity: 0;
    }
    30% {
      opacity: 1;
    }
    70% {
      opacity: 1;
    }
    100% {
      top: 100%;
      opacity: 0;
    }
  }

  @keyframes nodePulse {
    0%, 100% {
      border-color: #71717a;
      box-shadow: 0 0 6px rgba(113, 113, 122, 0.4);
    }
    50% {
      border-color: #a1a1aa;
      box-shadow: 0 0 10px rgba(255, 255, 255, 0.4);
    }
  }

  .timeline-item:last-child .timeline-vertical-line {
    bottom: 0;
  }

  .timeline-card {
    flex-grow: 1;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .exp-header-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 100%;
    gap: 0.75rem;
    margin-bottom: 0.35rem;
  }

  .exp-title-row {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    flex-wrap: wrap;
  }

  .exp-role {
    font-size: 1.22rem;
    font-weight: 600;
    color: var(--text-primary);
    line-height: 1.25;
  }

  .badge-freelance {
    font-family: var(--font-mono);
    font-size: 0.76rem;
    color: var(--text-secondary);
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid var(--border-color);
    padding: 0.15rem 0.5rem;
    border-radius: 4px;
  }

  .exp-period {
    font-family: var(--font-mono);
    font-size: 0.82rem;
    color: var(--text-muted);
    text-transform: uppercase;
    letter-spacing: 0.05em;
    white-space: nowrap;
    text-align: right;
    margin-left: auto;
  }

  .exp-meta-line {
    font-size: 0.96rem;
    color: var(--text-secondary);
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
    align-items: center;
    line-height: 1.5;
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
    font-size: 0.94rem;
  }

  .bot-desc {
    color: var(--text-muted);
    font-style: italic;
    font-size: 0.92rem;
  }

  .tech-line {
    margin-bottom: 0.45rem;
  }

  .tech-val {
    color: var(--text-secondary);
    font-size: 0.94rem;
  }

  .exp-bullets {
    margin-top: 0.35rem;
    font-size: 0.98rem;
    line-height: 1.65;
  }

  /* Bio & Markdown Content Styling */
  .markdown-content {
    display: flex;
    flex-direction: column;
    color: var(--text-secondary);
    font-size: 1.02rem;
    line-height: 1.7;
  }

  :global(.markdown-content p) {
    margin-bottom: 0.95rem;
  }

  :global(.markdown-content p:last-child) {
    margin-bottom: 0;
  }

  :global(.markdown-content ul) {
    margin-left: 1.35rem;
    padding-left: 0.25rem;
    display: flex;
    flex-direction: column;
    gap: 0.45rem;
    margin-bottom: 0.85rem;
  }

  :global(.markdown-content li) {
    padding-left: 0.25rem;
  }

  :global(.project-detail-body h1) {
    color: var(--text-primary);
    font-size: 1.75rem;
    font-weight: 600;
    margin-bottom: 1.1rem;
    line-height: 1.25;
  }

  :global(.project-detail-body h2) {
    color: var(--text-primary);
    font-size: 1.22rem;
    font-weight: 600;
    margin-top: 1.65rem;
    margin-bottom: 0.75rem;
  }

  :global(.project-detail-body p) {
    margin-bottom: 1.2rem;
    font-size: 1.02rem;
    line-height: 1.7;
  }

  :global(.project-detail-body ul) {
    margin-left: 1.35rem;
    padding-left: 0.25rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    margin-bottom: 1.35rem;
    font-size: 0.98rem;
    line-height: 1.65;
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
    height: 180px !important;
    width: auto !important;
    max-width: 100% !important;
    object-fit: contain !important;
    filter: drop-shadow(0 0 8px rgba(255, 255, 255, 0.15));
    animation: logoGlowPulse 3.5s ease-in-out infinite alternate;
    transition: transform 0.3s cubic-bezier(0.16, 1, 0.3, 1), filter 0.3s ease;
  }

  :global(.project-logo-wrapper:hover .hwmonitor-logo-img) {
    filter: drop-shadow(0 0 24px rgba(255, 255, 255, 0.55));
  }

  @keyframes logoGlowPulse {
    0% {
      filter: drop-shadow(0 0 6px rgba(255, 255, 255, 0.12));
    }
    100% {
      filter: drop-shadow(0 0 16px rgba(255, 255, 255, 0.35));
    }
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
    font-size: 0.78rem;
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
    border: 1px solid rgba(255, 255, 255, 0.3);
    color: #ffffff;
    border-radius: 50%;
    width: 40px;
    height: 40px;
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
    transition: background 0.2s ease, border-color 0.2s ease;
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
    gap: 0.65rem;
    margin-bottom: 1.35rem;
    color: var(--text-primary);
  }

  .section-bar {
    display: inline-block;
    width: 24px;
    height: 2.5px;
    background: linear-gradient(90deg, #e4e4e7 0%, #a1a1aa 55%, #52525b 100%);
    border-radius: 1px;
    flex-shrink: 0;
    transform-origin: left center;
    animation: sectionBarIn 0.65s cubic-bezier(0.16, 1, 0.3, 1) both;
    box-shadow: 0 0 8px rgba(228, 228, 231, 0.35);
  }

  @keyframes sectionBarIn {
    0% {
      width: 0;
      opacity: 0;
      transform: scaleX(0);
    }
    50% {
      opacity: 1;
    }
    100% {
      width: 24px;
      opacity: 1;
      transform: scaleX(1);
    }
  }

  .section-title-wrapper h2 {
    font-size: 1.52rem;
    font-weight: 600;
    color: var(--text-primary);
    line-height: 1.2;
    letter-spacing: -0.015em;
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
    margin-bottom: 1.5rem;
  }

  .footer-title {
    display: flex;
    align-items: center;
    gap: 0.65rem;
    font-size: 1.35rem;
    font-weight: 600;
    color: var(--text-primary);
  }

  .time-wrapper {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    color: var(--text-muted);
  }

  .time-display {
    font-family: var(--font-mono);
    font-size: 0.86rem;
  }

  .links-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 1.75rem 1.5rem;
  }

  .link-item {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }

  .link-label {
    font-size: 0.98rem;
    color: var(--text-primary);
  }

  .link-url {
    display: inline-flex;
    align-items: center;
    gap: 0.4rem;
    font-family: var(--font-mono);
    font-size: 0.9rem;
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
    width: 100%;
    animation: tabEnter 0.65s cubic-bezier(0.16, 1, 0.3, 1) both;
    will-change: opacity, transform, filter;
  }

  @keyframes tabEnter {
    0% {
      opacity: 0;
      transform: translateY(24px);
      filter: blur(3px);
    }
    50% {
      filter: blur(0);
    }
    100% {
      opacity: 1;
      transform: translateY(0);
      filter: blur(0);
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .tab-content {
      animation: none;
    }
  }

  .status-msg {
    font-family: var(--font-mono);
    color: var(--text-muted);
    font-size: 0.92rem;
  }

  /* Language Filter Switcher */
  .lang-filter-container {
    display: flex;
    flex-wrap: wrap;
    gap: 0.45rem;
    margin: 0.5rem 0 1.75rem 0;
    padding-bottom: 0.75rem;
    border-bottom: 1px solid var(--border-color);
  }

  .lang-filter-btn {
    font-family: var(--font-mono);
    font-size: 0.85rem;
    padding: 0.35rem 0.8rem;
    border-radius: 6px;
    border: 1px solid var(--border-color);
    background: rgba(255, 255, 255, 0.02);
    color: var(--text-secondary);
    transition: all 0.15s ease;
  }

  .lang-filter-btn:hover {
    color: var(--text-primary);
    border-color: #52525b;
    background: rgba(255, 255, 255, 0.05);
  }

  .lang-filter-btn.active {
    color: #ffffff;
    border-color: var(--text-primary);
    background: rgba(255, 255, 255, 0.08);
    box-shadow: 0 0 8px rgba(255, 255, 255, 0.04);
  }

  /* Selected Projects Grid */
  .project-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 1.15rem;
    width: 100%;
    max-width: 100%;
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
    width: 100%;
    max-width: 100%;
    box-sizing: border-box;
    overflow: hidden;
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
    max-width: 100%;
    min-width: 0;
    gap: 0.5rem;
    flex-wrap: wrap;
  }

  .card-title-group {
    display: flex;
    align-items: center;
    gap: 0.4rem;
    min-width: 0;
    max-width: 100%;
    flex-wrap: wrap;
  }

  :global(.card-title-icon) {
    color: var(--text-muted);
    flex-shrink: 0;
  }

  .card-header h3 {
    font-size: 1.08rem;
    color: var(--text-primary);
    font-family: var(--font-mono);
    font-weight: 600;
    overflow-wrap: break-word;
    word-break: break-word;
    min-width: 0;
    max-width: 100%;
  }

  .stars {
    font-family: var(--font-mono);
    font-size: 0.82rem;
    color: var(--text-secondary);
    flex-shrink: 0;
  }

  .project-card p {
    font-size: 0.98rem;
    color: var(--text-secondary);
    margin-bottom: 0.95rem;
    line-height: 1.6;
    overflow-wrap: break-word;
    word-break: break-word;
  }

  .tags,
  .exp-preview-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
    font-family: var(--font-mono);
    font-size: 0.82rem;
    color: var(--text-secondary);
    max-width: 100%;
  }

  .tag-primary {
    font-family: var(--font-mono);
    font-size: 0.82rem;
    color: var(--text-primary);
    border: 1px solid var(--border-color);
    padding: 0.15rem 0.48rem;
    border-radius: 4px;
    background: rgba(255, 255, 255, 0.03);
    max-width: 100%;
    overflow-wrap: break-word;
    word-break: break-word;
  }

  .tag-item {
    font-family: var(--font-mono);
    font-size: 0.82rem;
    color: var(--text-secondary);
    border: 1px solid var(--border-color);
    padding: 0.15rem 0.48rem;
    border-radius: 4px;
    background: rgba(255, 255, 255, 0.015);
    max-width: 100%;
    overflow-wrap: break-word;
    word-break: break-word;
  }

  /* Dedicated Project Detail Section */
  .project-detail-section {
    display: flex;
    flex-direction: column;
    gap: 1.35rem;
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
    font-size: 0.94rem;
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
    padding: 0.35rem 0.75rem;
    border-radius: 6px;
    color: var(--text-primary);
    font-size: 0.88rem;
    transition: border-color 0.2s ease;
  }

  .github-btn:hover {
    border-color: #5e5e65;
  }

  /* Mobile & Tablet Responsiveness */
  /* ═══ Mobile Navigation ═══ */
  .hamburger-btn {
    display: none;
    flex-direction: column;
    justify-content: center;
    gap: 5px;
    width: 36px;
    height: 36px;
    background: none;
    border: 1px solid var(--border-color);
    border-radius: 7px;
    cursor: pointer;
    padding: 8px 7px;
    transition: border-color 0.15s ease;
  }

  .hamburger-btn:hover {
    border-color: #52525b;
  }

  .hamburger-line {
    display: block;
    width: 100%;
    height: 1.5px;
    background: #ffffff;
    border-radius: 2px;
    transition: transform 0.25s ease, opacity 0.2s ease, background 0.15s ease;
    transform-origin: center;
  }

  .hamburger-btn:hover .hamburger-line {
    background: #ffffff;
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

  :global(.markdown-content pre) {
    max-width: 100%;
    overflow-x: auto;
    background: #18181c;
    border: 1px solid var(--border-color);
    border-radius: 6px;
    padding: 0.85rem 1.1rem;
    font-family: var(--font-mono);
    font-size: 0.9rem;
    white-space: pre-wrap;
    word-break: break-all;
  }

  :global(.markdown-content code) {
    font-family: var(--font-mono);
    font-size: 0.88em;
    background: rgba(255, 255, 255, 0.05);
    padding: 0.15em 0.35em;
    border-radius: 4px;
    word-break: break-word;
  }

  :global(.markdown-content img) {
    max-width: 100%;
    height: auto;
  }

  :global(.markdown-content table) {
    width: 100%;
    max-width: 100%;
    display: block;
    overflow-x: auto;
    border-collapse: collapse;
  }

  .mobile-menu {
    position: absolute;
    top: 3.5rem;
    left: 0.5rem;
    right: 0.5rem;
    background: rgba(20, 20, 24, 0.98);
    backdrop-filter: blur(20px);
    -webkit-backdrop-filter: blur(20px);
    border: 1px solid var(--border-color);
    border-radius: 10px;
    padding: 0.6rem;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    z-index: 999;
    box-shadow: 0 16px 40px rgba(0, 0, 0, 0.65);
  }

  .mobile-nav-item {
    display: flex;
    align-items: center;
    gap: 0.9rem;
    padding: 0.85rem 1.1rem;
    border-radius: 8px;
    color: var(--text-muted);
    text-decoration: none;
    font-family: var(--font-mono);
    font-size: 1rem;
    transition: background 0.15s ease, color 0.15s ease;
    border: 1px solid transparent;
  }

  .mobile-nav-item.active {
    color: var(--text-primary);
    background: rgba(255,255,255,0.05);
    border-color: var(--border-color);
  }

  .mobile-nav-item:hover {
    background: rgba(255,255,255,0.06);
    color: #ffffff;
  }

  :global(.mobile-nav-icon) {
    opacity: 0.7;
  }

  .mobile-nav-item.active :global(.mobile-nav-icon),
  .mobile-nav-item:hover :global(.mobile-nav-icon) {
    opacity: 1;
  }

  /* ═══ Tablet & Mobile Responsive ═══ */
  @media (max-width: 768px) {
    .portfolio-container {
      position: relative;
      padding: 0 1.25rem;
    }

    .nav-desktop {
      display: none;
    }

    .hamburger-btn {
      display: flex;
    }

    .nav-bar {
      justify-content: flex-end;
      align-items: center;
      margin-bottom: 2rem;
      padding: 0.5rem 0;
      position: relative;
      z-index: 600;
    }

    .mobile-nav-item {
      font-size: 1.05rem;
      padding: 0.9rem 1.15rem;
    }

    .profile-header {
      flex-direction: column;
      align-items: flex-start;
      gap: 1.25rem;
    }

    .profile-name {
      font-size: 2.2rem;
    }

    .profile-handle, .profile-role {
      font-size: 0.98rem;
    }

    .markdown-content, .bio-text {
      font-size: 1.02rem;
      line-height: 1.7;
    }

    .section-title-wrapper h2 {
      font-size: 1.46rem;
    }

    .section-bar {
      width: 22px;
      height: 2.5px;
    }


    .exp-role {
      font-size: 1.20rem;
    }

    .badge-freelance {
      font-size: 0.76rem;
      padding: 0.15rem 0.48rem;
    }

    .exp-period {
      font-size: 0.88rem;
    }

    .exp-meta-line {
      font-size: 0.95rem;
    }

    .bot-link, .bot-desc, .tech-val {
      font-size: 0.92rem;
    }

    .exp-bullets {
      font-size: 0.98rem;
      line-height: 1.65;
    }

    .tag-item, .tag-primary {
      font-size: 0.82rem;
      padding: 0.15rem 0.48rem;
    }

    .show-more-link {
      font-size: 0.9rem;
    }

    .featured-project-row {
      padding: 0.95rem 1.2rem;
    }

    .featured-project-row .row-title {
      font-size: 1.02rem;
    }

    .featured-project-row .row-desc {
      font-size: 0.94rem;
    }

    .lang-filter-btn {
      font-size: 0.88rem;
      padding: 0.38rem 0.85rem;
    }

    .project-grid {
      grid-template-columns: 1fr;
    }

    .project-card {
      padding: 1.25rem;
    }

    .project-card .card-header h3 {
      font-size: 1.08rem;
    }

    .stars {
      font-size: 0.82rem;
    }

    .project-card p {
      font-size: 0.98rem;
      line-height: 1.6;
    }

    :global(.screenshots-group) {
      grid-template-columns: repeat(2, 1fr);
    }

    :global(.screenshot-item figcaption) {
      font-size: 0.78rem;
      padding: 0.2rem 0;
    }

    .back-btn {
      font-size: 0.94rem;
    }

    .github-btn {
      font-size: 0.88rem;
      padding: 0.35rem 0.75rem;
    }

    .footer-header {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.5rem;
    }

    .footer-title {
      font-size: 1.28rem;
    }

    .time-display {
      font-size: 0.86rem;
    }

    .link-label {
      font-size: 0.98rem;
    }

    .link-url {
      font-size: 0.9rem;
    }

    .timeline-line-col {
      display: none;
    }

    .timeline-item {
      gap: 0;
    }

    .tab-content {
      padding-bottom: 2rem;
    }
  }

  @media (max-width: 480px) {
    .portfolio-container {
      padding: 0 1rem;
    }

    .profile-name {
      font-size: 1.95rem;
    }

    .profile-handle, .profile-role {
      font-size: 0.94rem;
    }

    .avatar-radar-wrapper {
      width: 88px;
      height: 88px;
    }

    .avatar-container {
      width: 70px;
      height: 70px;
    }

    .markdown-content, .bio-text {
      font-size: 0.98rem;
      line-height: 1.65;
    }

    .section-title-wrapper h2 {
      font-size: 1.40rem;
    }

    .section-bar {
      width: 20px;
      height: 2px;
    }


    .exp-header-row {
      display: flex;
      justify-content: space-between;
      align-items: flex-start;
      gap: 0.5rem;
    }

    .exp-title-row {
      flex-direction: column;
      align-items: flex-start;
      gap: 0.25rem;
    }

    .exp-role {
      font-size: 1.16rem;
    }

    .badge-freelance {
      font-size: 0.74rem;
      padding: 0.12rem 0.42rem;
    }

    .exp-period {
      font-size: 0.78rem;
      text-align: right;
      white-space: nowrap;
      margin-top: 0.15rem;
    }

    .exp-meta-line {
      font-size: 0.92rem;
    }

    .bot-link, .bot-desc, .tech-val {
      font-size: 0.9rem;
    }

    .exp-bullets {
      font-size: 0.94rem;
      line-height: 1.6;
    }

    .tag-item, .tag-primary {
      font-size: 0.78rem;
      padding: 0.12rem 0.42rem;
    }

    .show-more-link {
      font-size: 0.86rem;
    }



    .featured-project-row .row-desc {
      white-space: normal;
      font-size: 0.90rem;
    }

    .featured-project-row .row-title {
      font-size: 0.98rem;
    }

    .lang-filter-btn {
      font-size: 0.84rem;
      padding: 0.35rem 0.8rem;
    }

    .project-card .card-header h3 {
      font-size: 1.04rem;
    }

    .project-card p {
      font-size: 0.95rem;
      line-height: 1.55;
    }

    .stars {
      font-size: 0.8rem;
    }

    :global(.screenshots-group) {
      grid-template-columns: 1fr;
    }

    :global(.screenshot-item figcaption) {
      font-size: 0.76rem;
    }

    .links-grid {
      grid-template-columns: 1fr;
      gap: 1.25rem;
    }

    .link-label {
      font-size: 0.94rem;
    }

    .link-url {
      font-size: 0.88rem;
    }

    .footer-title {
      font-size: 1.22rem;
    }

    .time-display {
      font-size: 0.84rem;
    }

    .mobile-nav-item {
      font-size: 1rem;
      padding: 0.85rem 1.05rem;
    }

    .timeline-line-col {
      display: none;
    }

    .timeline-item {
      gap: 0;
    }
  }

  /* ═══ Home Page Enhancement Styles ═══ */

  /* Home Section Blocks */
  .home-section-block {
    margin-bottom: 3.5rem;
  }

  .home-section-block .section-title-wrapper {
    margin-bottom: 1.25rem;
  }

  /* Experience Preview Card */
  .exp-preview-card {
    display: block;
    text-decoration: none;
    color: inherit;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    padding: 1.1rem 1.35rem;
    border-radius: 8px;
    margin-bottom: 0.65rem;
    transition: border-color 0.25s ease, transform 0.25s ease;
    cursor: pointer;
  }

  .exp-preview-card:hover {
    border-color: #5e5e65;
    transform: translateX(2px);
  }

  .exp-preview-tags {
    margin-top: 0.55rem;
  }

  /* Show More Link */
  .show-more-link {
    display: inline-flex;
    align-items: center;
    gap: 0.3rem;
    font-family: var(--font-mono);
    font-size: 0.9rem;
    color: var(--text-secondary);
    text-decoration: none;
    margin-top: 1.15rem;
    transition: color 0.15s ease, gap 0.15s ease;
  }

  .show-more-link:hover {
    color: var(--text-primary);
    gap: 0.5rem;
  }


  /* Concise Featured Projects List */
  .featured-project-list {
    display: flex;
    flex-direction: column;
    gap: 0.65rem;
  }

  .featured-project-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    background: var(--bg-card);
    border: 1px solid var(--border-color);
    padding: 0.95rem 1.2rem;
    border-radius: 8px;
    text-align: left;
    transition: border-color 0.25s ease, transform 0.25s ease;
    cursor: pointer;
  }

  .featured-project-row:hover {
    border-color: #5e5e65;
    transform: translateX(3px);
  }

  .featured-project-row .row-left {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    flex: 1;
    min-width: 0;
  }

  .featured-project-row .row-header {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .featured-project-row .row-title {
    font-family: var(--font-mono);
    font-size: 1.02rem;
    font-weight: 600;
    color: var(--text-primary);
  }

  .featured-project-row .row-desc {
    font-size: 0.94rem;
    color: var(--text-secondary);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .tag-primary.compact {
    font-size: 0.74rem;
    padding: 0.12rem 0.38rem;
  }

  .featured-project-row .row-right {
    display: flex;
    align-items: center;
    padding-left: 0.85rem;
    color: var(--text-muted);
    transition: color 0.15s ease;
  }

  .featured-project-row:hover :global(.card-title-icon),
  .featured-project-row:hover .row-right {
    color: #ffffff !important;
    opacity: 1 !important;
  }

  /* Global Card & Row Icon Hover Enhancements */
  :global(.card-title-icon),
  :global(.section-icon),
  :global(.nav-icon) {
    transition: color 0.2s ease, opacity 0.2s ease;
  }

  .project-card:hover :global(.card-title-icon) {
    color: #ffffff !important;
  }
</style>
