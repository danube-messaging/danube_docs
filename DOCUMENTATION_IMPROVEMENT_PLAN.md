# Danube Documentation Improvement Plan

## OBJECTIVE 1: Navigation Structure & Repository Organization

### Current Issues
1. **Getting Started buried** - Should be more prominent for new users
2. **Architecture too early** - Heavy concepts before users try the product
3. **Client Libraries scattered** - No clear progression from basics to advanced
4. **CLI tools nested too deep** - Important tools hard to discover
5. **No clear conceptual section** - Messaging patterns mixed with architecture internals
6. **Development mixed with user docs** - Contributing info should be separate

---

### Proposed Navigation Structure (mkdocs.yml)

```yaml
nav:
  # 1. WELCOME - First impression
  - Home: index.md
  
  # 2. QUICK START - Get users running fast
  - Getting Started:
      - Docker Compose (Recommended): getting_started/Danube_docker_compose.md
      - Kubernetes: getting_started/Danube_kubernetes.md
      - Local Machine / VMs: getting_started/Danube_local.md
  
  # 3. CORE CONCEPTS - Learn fundamentals (move from architecture)
  - Core Concepts:
      - Topics: concepts/topics.md
      - Subscriptions: concepts/subscriptions.md
      - Dispatch Strategies: concepts/dispatch_strategy.md
      - Messages & Schemas: concepts/messages.md
      - Messaging Patterns:
          - Queuing vs Pub/Sub: concepts/messaging_patterns_queuing_vs_pubsub.md
          - Pub/Sub vs Streaming: concepts/messaging_modes_pubsub_vs_streaming.md
  
  # 4. CLIENT LIBRARIES - Progressive learning path
  - Client Libraries:
      - Overview & Installation: client_libraries/clients.md
      - Setup & Configuration: client_libraries/setup.md
      - Producer Guide:
          - Basic Producer: client_libraries/producer-basics.md
          - Advanced Producer: client_libraries/producer-advanced.md
      - Consumer Guide:
          - Basic Consumer: client_libraries/consumer-basics.md
          - Advanced Consumer: client_libraries/consumer-advanced.md
      - Schema Registry: client_libraries/schema-registry.md
  
  # 5. CLI TOOLS - Operations and management
  - CLI Tools:
      - Danube CLI (Client):
          - Getting Started: danube_clis/danube_cli/getting_started.md
          - Producer: danube_clis/danube_cli/producer.md
          - Consumer: danube_clis/danube_cli/consumer.md
          - Schema Registry: danube_clis/danube_cli/schema_registry.md
      - Admin CLI (Operations):
          - Brokers: danube_clis/danube_admin/brokers.md
          - Namespaces: danube_clis/danube_admin/namespaces.md
          - Topics: danube_clis/danube_admin/topics.md
          - Schema Registry: danube_clis/danube_admin/schema_registry.md
  
  # 6. ARCHITECTURE - Deep dive for advanced users
  - Architecture:
      - System Overview: architecture/architecture.md
      - Persistence (WAL + Cloud): architecture/persistence.md
      - Schema Registry Architecture: architecture/schema_registry_architecture.md
      - Internal Services: architecture/internal_danube_services.md
  
  # 7. CONTRIBUTING - Development and contribution
  - Contributing:
      - Development Environment: contributing/dev_environment.md
      - Internal Resources: contributing/internal_resources.md
```

---

### Repository Structure Changes

**File Movements Required:**

```bash
# Create new folders
docs/concepts/
docs/contributing/
docs/assets/img/architecture/
docs/assets/img/concepts/

# Move files from architecture/ to concepts/
architecture/topics.md                              → concepts/topics.md
architecture/subscriptions.md                       → concepts/subscriptions.md
architecture/dispatch_strategy.md                   → concepts/dispatch_strategy.md
architecture/messages.md                            → concepts/messages.md
architecture/messaging_patterns_queuing_vs_pubsub.md → concepts/messaging_patterns_queuing_vs_pubsub.md
architecture/messaging_modes_pubsub_vs_streaming.md  → concepts/messaging_modes_pubsub_vs_streaming.md

# Move files from development/ to contributing/
development/dev_environment.md                      → contributing/dev_environment.md
development/internal_resources.md                   → contributing/internal_resources.md

# Move images to centralized assets folder
architecture/img/Danube_architecture.png            → assets/img/architecture/Danube_architecture.png
architecture/img/Wal_Cloud.png                      → assets/img/architecture/Wal_Cloud.png
architecture/img/partitioned_topics.png             → assets/img/concepts/partitioned_topics.png
architecture/img/producers_consumers.png            → assets/img/concepts/producers_consumers.png
architecture/img/exclusive_subscription_*.png       → assets/img/concepts/exclusive_subscription_*.png
architecture/img/shared_subscription_*.png          → assets/img/concepts/shared_subscription_*.png
architecture/img/failover_subscription_*.png        → assets/img/concepts/failover_subscription_*.png

# Keep in architecture/ (advanced internals)
architecture/architecture.md                        (stays - system overview)
architecture/persistence.md                         (stays - WAL + Cloud deep dive)
architecture/schema_registry_architecture.md        (stays - schema registry internals)
architecture/internal_danube_services.md            (stays - internal components)
```

**Final Directory Structure:**

```
docs/
├── index.md
├── assets/                            # NEW - Centralized assets
│   ├── danube.png                     # Logo (already exists)
│   ├── favicon.ico                    # Favicon (to be added)
│   └── img/
│       ├── architecture/              # Architecture diagrams
│       │   ├── Danube_architecture.png
│       │   └── Wal_Cloud.png
│       ├── concepts/                  # Concept diagrams
│       │   ├── partitioned_topics.png
│       │   ├── producers_consumers.png
│       │   ├── exclusive_subscription_non_partitioned.png
│       │   ├── exclusive_subscription_partitioned.png
│       │   ├── shared_subscription_non_partitioned.png
│       │   ├── shared_subscription_partitioned.png
│       │   ├── failover_subscription_non_partitioned.png
│       │   └── failover_subscription_partitioned.png
│       └── getting_started/           # Getting started images (existing)
│           └── ... (from getting_started/img/)
├── getting_started/
│   ├── Danube_docker_compose.md
│   ├── Danube_kubernetes.md
│   └── Danube_local.md
├── concepts/                          # NEW - User-facing concepts
│   ├── topics.md
│   ├── subscriptions.md
│   ├── dispatch_strategy.md
│   ├── messages.md
│   ├── messaging_patterns_queuing_vs_pubsub.md
│   └── messaging_modes_pubsub_vs_streaming.md
├── client_libraries/
│   ├── clients.md
│   ├── setup.md
│   ├── producer-basics.md
│   ├── producer-advanced.md
│   ├── consumer-basics.md
│   ├── consumer-advanced.md
│   └── schema-registry.md
├── danube_clis/
│   ├── danube_cli/
│   │   ├── getting_started.md
│   │   ├── producer.md
│   │   ├── consumer.md
│   │   └── schema_registry.md
│   └── danube_admin/
│       ├── brokers.md
│       ├── namespaces.md
│       ├── topics.md
│       └── schema_registry.md
├── architecture/                      # REDUCED - Only deep internals
│   ├── architecture.md
│   ├── persistence.md
│   ├── schema_registry_architecture.md
│   └── internal_danube_services.md
├── contributing/                      # NEW - Development docs
│   ├── dev_environment.md
│   └── internal_resources.md
├── stylesheets/                       # NEW - Custom CSS
│   └── extra.css
├── javascripts/                       # NEW - Custom JS
│   ├── extra.js
│   └── mathjax.js
└── includes/                          # NEW - Reusable content
    └── abbreviations.md
```

---

### Image Path Updates Required

After moving files and images, the following markdown files need their image paths updated:

**Files moving to concepts/ (update image paths):**

1. **concepts/topics.md**
   ```markdown
   # Before (in architecture/)
   ![Partitioned Topics](img/partitioned_topics.png "Partitioned topics")
   
   # After (in concepts/)
   ![Partitioned Topics](../assets/img/concepts/partitioned_topics.png "Partitioned topics")
   ```

2. **concepts/subscriptions.md**
   ```markdown
   # Before (in architecture/)
   ![Producers Consumers](img/producers_consumers.png "Producers Consumers")
   ![Exclusive Non-Partitioned](img/exclusive_subscription_non_partitioned.png "...")
   ![Exclusive Partitioned](img/exclusive_subscription_partitioned.png "...")
   ![Shared Non-Partitioned](img/shared_subscription_non_partitioned.png "...")
   ![Shared Partitioned](img/shared_subscription_partitioned.png "...")
   ![Failover Non-Partitioned](img/failover_subscription_non_partitioned.png "...")
   ![Failover Partitioned](img/failover_subscription_partitioned.png "...")
   
   # After (in concepts/)
   ![Producers Consumers](../assets/img/concepts/producers_consumers.png "Producers Consumers")
   ![Exclusive Non-Partitioned](../assets/img/concepts/exclusive_subscription_non_partitioned.png "...")
   ![Exclusive Partitioned](../assets/img/concepts/exclusive_subscription_partitioned.png "...")
   ![Shared Non-Partitioned](../assets/img/concepts/shared_subscription_non_partitioned.png "...")
   ![Shared Partitioned](../assets/img/concepts/shared_subscription_partitioned.png "...")
   ![Failover Non-Partitioned](../assets/img/concepts/failover_subscription_non_partitioned.png "...")
   ![Failover Partitioned](../assets/img/concepts/failover_subscription_partitioned.png "...")
   ```

**Files staying in architecture/ (update image paths):**

3. **architecture/architecture.md**
   ```markdown
   # Before
   ![Danube Messaging Architecture](img/Danube_architecture.png "Danube Messaging Architecture")
   
   # After
   ![Danube Messaging Architecture](../assets/img/architecture/Danube_architecture.png "Danube Messaging Architecture")
   ```

4. **architecture/persistence.md**
   ```markdown
   # Before
   ![Danube Persistence Architecture](img/Wal_Cloud.png "Danube Persistence Architecture")
   
   # After
   ![Danube Persistence Architecture](../assets/img/architecture/Wal_Cloud.png "Danube Persistence Architecture")
   ```

**Note:** The getting_started/ images also need to be moved and paths updated, but we'll handle those separately based on what exists.

---

### Benefits of This Structure

1. **Progressive Disclosure**
   - Quick Start → Core Concepts → Client Usage → Advanced Architecture
   - New users aren't overwhelmed by internal architecture

2. **Task-Oriented Navigation**
   - "I want to run Danube" → Getting Started
   - "I want to understand topics" → Core Concepts
   - "I want to write code" → Client Libraries
   - "I want to manage cluster" → CLI Tools
   - "I want to understand internals" → Architecture

3. **Clear Separation of Concerns**
   - User concepts vs implementation details
   - Client usage vs operations
   - Using vs contributing

4. **Better Discoverability**
   - Important features not buried 3+ levels deep
   - CLI tools promoted to top level
   - Concepts separated from internals

5. **Logical Grouping**
   - Related topics together
   - Clear progression within each section

---

## OBJECTIVE 2: MkDocs Configuration & Professional Appearance

### Current Issues
1. **Minimal Material features** - Not using navigation tabs, instant loading, or search enhancements
2. **No visual polish** - Missing code copy buttons, last updated dates, breadcrumbs
3. **Basic markdown** - No admonitions, diagrams, or icons
4. **Missing SEO/social** - No social cards or meta descriptions
5. **Limited navigation aids** - No table of contents integration, back-to-top, or breadcrumbs

---

### Enhanced mkdocs.yml Configuration

```yaml
# ============================================================================
# SITE METADATA
# ============================================================================
site_name: Danube Messaging
site_url: https://danube-messaging.github.io/danube_docs/
site_description: >-
  Lightweight, cloud-native messaging platform built in Rust. 
  Sub-second dispatch with cloud economics powered by WAL + Object Storage.
site_author: Danube Team

# ============================================================================
# REPOSITORY
# ============================================================================
repo_url: https://github.com/danube-messaging/danube
repo_name: danube-messaging/danube
edit_uri: edit/main/docs/

# ============================================================================
# COPYRIGHT
# ============================================================================
copyright: Copyright &copy; 2024 Danube Messaging

# ============================================================================
# THEME CONFIGURATION
# ============================================================================
theme:
  name: material
  logo: assets/danube.png
  favicon: assets/favicon.ico
  
  # ==========================================================================
  # MATERIAL FEATURES
  # ==========================================================================
  features:
    # Content features
    - content.action.edit           # Edit this page button
    - content.action.view           # View source button
    - content.code.annotate         # Code annotations
    - content.code.copy             # Copy button for code blocks
    - content.tabs.link             # Linked content tabs
    - content.tooltips              # Improved tooltips
    
    # Navigation features
    - navigation.expand             # Expand navigation by default
    - navigation.footer             # Previous/Next footer navigation
    - navigation.indexes            # Section index pages
    - navigation.instant            # Instant loading (SPA-like)
    - navigation.instant.prefetch   # Prefetch pages on hover
    - navigation.instant.progress   # Progress indicator for page loads
    - navigation.path               # Breadcrumbs navigation
    - navigation.sections           # Group navigation sections
    - navigation.tabs               # Top-level navigation tabs
    - navigation.tabs.sticky        # Sticky navigation tabs
    - navigation.top                # Back to top button
    - navigation.tracking           # Update URL with active anchor
    
    # Search features
    - search.highlight              # Highlight search terms in results
    - search.share                  # Share search query link
    - search.suggest                # Search suggestions as you type
    
    # Table of contents
    - toc.follow                    # TOC follows scroll position
    - toc.integrate                 # Integrate TOC into left sidebar
  
  # ==========================================================================
  # COLOR PALETTE
  # ==========================================================================
  palette:
    # Light mode
    - media: "(prefers-color-scheme: light)"
      scheme: default
      primary: indigo               # Primary brand color
      accent: blue                  # Accent color for links/buttons
      toggle:
        icon: material/brightness-7
        name: Switch to dark mode
    
    # Dark mode
    - media: "(prefers-color-scheme: dark)"
      scheme: slate
      primary: indigo
      accent: blue
      toggle:
        icon: material/brightness-4
        name: Switch to light mode
  
  # ==========================================================================
  # TYPOGRAPHY
  # ==========================================================================
  font:
    text: Roboto
    code: Roboto Mono
  
  # ==========================================================================
  # ICONS
  # ==========================================================================
  icon:
    repo: fontawesome/brands/github
    edit: material/pencil
    view: material/eye
    logo: material/waves

# ============================================================================
# PLUGINS
# ============================================================================
plugins:
  # Search plugin with enhanced configuration
  - search:
      separator: '[\s\-,:!=\[\]()"/]+|\.(?!\d)|&[lg]t;|(?!\b)(?=[A-Z][a-z])'
      lang: en
  
  # Git revision date - shows last updated timestamp
  - git-revision-date-localized:
      enable_creation_date: true
      type: timeago
      fallback_to_build_date: true
  
  # Minify HTML/CSS/JS for faster loading
  - minify:
      minify_html: true
      minify_js: true
      minify_css: true
      htmlmin_opts:
        remove_comments: true
  
  # Social cards - beautiful link previews
  - social:
      cards_layout_options:
        background_color: "#1e3a8a"
        color: "#ffffff"

# ============================================================================
# MARKDOWN EXTENSIONS
# ============================================================================
markdown_extensions:
  # ------------------------------------------------------------------------
  # Python Markdown Extensions
  # ------------------------------------------------------------------------
  - abbr                            # Abbreviations with tooltips
  - admonition                      # Note/Warning/Tip boxes
  - attr_list                       # Add HTML/CSS attributes
  - def_list                        # Definition lists
  - footnotes                       # Footnote references
  - md_in_html                      # Markdown inside HTML blocks
  - tables                          # Table support
  - toc:                            # Table of contents
      permalink: true               # Add permalink anchors to headings
      permalink_title: Link to this section
      toc_depth: 3
  
  # ------------------------------------------------------------------------
  # PyMdown Extensions
  # ------------------------------------------------------------------------
  - pymdownx.arithmatex:            # Math equations (LaTeX)
      generic: true
  
  - pymdownx.betterem:              # Better emphasis handling
      smart_enable: all
  
  - pymdownx.caret                  # Superscript (^text^)
  - pymdownx.mark                   # Highlighting (==text==)
  - pymdownx.tilde                  # Strikethrough (~~text~~)
  
  - pymdownx.critic:                # Track changes/comments
      mode: view
  
  - pymdownx.details                # Collapsible admonition blocks
  
  - pymdownx.emoji:                 # Emoji support :emoji:
      emoji_index: !!python/name:material.extensions.emoji.twemoji
      emoji_generator: !!python/name:material.extensions.emoji.to_svg
  
  - pymdownx.highlight:             # Enhanced code highlighting
      anchor_linenums: true         # Anchor line numbers
      line_spans: __span
      pygments_lang_class: true
      auto_title: true              # Auto-add language title
      linenums: false               # Line numbers off by default
  
  - pymdownx.inlinehilite           # Inline code highlighting
  
  - pymdownx.keys                   # Keyboard keys (++ctrl+alt+del++)
  
  - pymdownx.magiclink:             # Auto-link URLs and repos
      normalize_issue_symbols: true
      repo_url_shorthand: true
      user: danube-messaging
      repo: danube
  
  - pymdownx.smartsymbols           # Smart symbols (arrows, fractions, etc.)
  
  - pymdownx.snippets:              # Include file snippets
      auto_append:
        - includes/abbreviations.md
      check_paths: true
  
  - pymdownx.superfences:           # Enhanced fenced code blocks
      custom_fences:
        - name: mermaid             # Mermaid diagrams support
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  
  - pymdownx.tabbed:                # Tabbed content blocks
      alternate_style: true
      combine_header_slug: true
      slugify: !!python/object/apply:pymdownx.slugs.slugify
        kwds:
          case: lower
  
  - pymdownx.tasklist:              # Task/Todo lists
      custom_checkbox: true
      clickable_checkbox: false

# ============================================================================
# EXTRA CONFIGURATION
# ============================================================================
extra:
  # Analytics
  analytics:
    provider: google
    property: G-2XEY5RCVVB
  
  # Social links in footer
  social:
    - icon: fontawesome/brands/github
      link: https://github.com/danube-messaging/danube
      name: Danube on GitHub
    
    - icon: fontawesome/brands/rust
      link: https://crates.io/crates/danube-client
      name: Rust Client on Crates.io
    
    - icon: fontawesome/brands/golang
      link: https://pkg.go.dev/github.com/danube-messaging/danube-go
      name: Go Client Documentation
    
    - icon: fontawesome/brands/docker
      link: https://hub.docker.com/r/danubemessaging/danube
      name: Danube on Docker Hub
  
  # Generator attribution
  generator: true
  
  # Consent (optional - for GDPR)
  # consent:
  #   title: Cookie consent
  #   description: >-
  #     We use cookies to recognize your repeated visits and preferences, as well
  #     as to measure the effectiveness of our documentation and whether users
  #     find what they're searching for. With your consent, you're helping us to
  #     make our documentation better.

# ============================================================================
# CUSTOM CSS/JS
# ============================================================================
extra_css:
  - stylesheets/extra.css

extra_javascript:
  - javascripts/extra.js
  # MathJax for math equations
  - javascripts/mathjax.js
  - https://polyfill.io/v3/polyfill.min.js?features=es6
  - https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js
```

---

### Enhanced requirements.txt

```txt
# ============================================================================
# CORE DEPENDENCIES
# ============================================================================
mkdocs==1.6.0
mkdocs-material==9.5.30
mkdocs-material-extensions==1.3.1

# ============================================================================
# MARKDOWN EXTENSIONS
# ============================================================================
pymdown-extensions==10.9
Markdown==3.6
pygments==2.18.0

# ============================================================================
# MKDOCS PLUGINS
# ============================================================================
# Git revision dates - shows last updated timestamps
mkdocs-git-revision-date-localized-plugin==1.2.4

# Minify HTML/CSS/JS for production
mkdocs-minify-plugin==0.8.0

# Social cards - beautiful link previews
pillow==10.3.0
cairosvg==2.7.1

# ============================================================================
# DEPENDENCIES
# ============================================================================
Babel==2.15.0
certifi==2024.7.4
charset-normalizer==3.3.2
click==8.1.7
colorama==0.4.6
ghp-import==2.1.0
idna==3.7
Jinja2==3.1.4
MarkupSafe==2.1.5
mergedeep==1.3.4
mkdocs-get-deps==0.2.0
packaging==24.1
paginate==0.5.6
pathspec==0.12.1
platformdirs==4.2.2
python-dateutil==2.9.0.post0
PyYAML==6.0.1
pyyaml_env_tag==0.1
regex==2024.7.24
requests==2.32.3
six==1.16.0
urllib3==2.2.2
watchdog==4.0.1
```

---

### Additional Files to Create

**1. docs/stylesheets/extra.css** - Custom styling

```css
/* Custom brand colors and refinements */
:root {
  --md-primary-fg-color: #3949ab;
  --md-primary-fg-color--light: #5e6bc4;
  --md-primary-fg-color--dark: #1e2a78;
}

/* Improve code block styling */
.highlight pre {
  border-radius: 0.4rem;
}

/* Better admonition styling */
.admonition {
  border-radius: 0.4rem;
  box-shadow: 0 0.2rem 0.5rem rgba(0, 0, 0, 0.05);
}

/* Improve table styling */
table {
  border-radius: 0.4rem;
  overflow: hidden;
}

/* Custom logo sizing */
.md-header__button.md-logo img {
  height: 1.8rem;
}
```

**2. docs/javascripts/extra.js** - Custom JavaScript

```javascript
// Custom documentation enhancements
document.addEventListener('DOMContentLoaded', function() {
  // Add external link indicators
  document.querySelectorAll('a[href^="http"]').forEach(function(link) {
    if (!link.hostname.includes('danube-messaging')) {
      link.setAttribute('target', '_blank');
      link.setAttribute('rel', 'noopener noreferrer');
    }
  });
});
```

**3. docs/javascripts/mathjax.js** - MathJax configuration

```javascript
window.MathJax = {
  tex: {
    inlineMath: [["\\(", "\\)"]],
    displayMath: [["\\[", "\\]"]],
    processEscapes: true,
    processEnvironments: true
  },
  options: {
    ignoreHtmlClass: ".*|",
    processHtmlClass: "arithmatex"
  }
};
```

**4. docs/includes/abbreviations.md** - Global abbreviations

```markdown
*[WAL]: Write-Ahead Log
*[ETCD]: Distributed key-value store
*[TLS]: Transport Layer Security
*[JWT]: JSON Web Token
*[CLI]: Command Line Interface
*[API]: Application Programming Interface
*[HA]: High Availability
*[S3]: Amazon Simple Storage Service
*[GCS]: Google Cloud Storage
*[MinIO]: S3-compatible object storage
*[JSON]: JavaScript Object Notation
*[Avro]: Apache Avro serialization format
*[Protobuf]: Protocol Buffers
```

**5. assets/favicon.ico** - Add favicon (if not exists)
- Create or download a favicon for the site

---

### Key Improvements Summary

#### Visual & UX Enhancements
- ✅ **Navigation tabs** - Top-level sections for major areas
- ✅ **Instant loading** - SPA-like navigation without page reloads
- ✅ **Code copy buttons** - One-click code copying
- ✅ **Breadcrumbs** - Clear navigation path
- ✅ **Back to top** - Quick return to top of page
- ✅ **Footer navigation** - Previous/Next page links
- ✅ **Search suggestions** - Auto-suggest as you type
- ✅ **Last updated dates** - Shows content freshness

#### Content Features
- ✅ **Admonitions** - Note, warning, tip, danger boxes
- ✅ **Mermaid diagrams** - Architecture diagrams in markdown
- ✅ **Emojis & icons** - Visual enhancement :rocket:
- ✅ **Task lists** - Interactive checkboxes
- ✅ **Collapsible sections** - Better content organization
- ✅ **Code annotations** - Inline code explanations
- ✅ **Math equations** - LaTeX support for formulas

#### Professional Polish
- ✅ **Social cards** - Beautiful link previews on social media
- ✅ **Minified assets** - Faster page loads
- ✅ **Custom branding** - Colors, fonts, favicon
- ✅ **Edit page links** - Encourage contributions
- ✅ **Abbreviation tooltips** - Hover to see definitions
- ✅ **Smart symbols** - Auto-convert arrows, fractions, etc.

#### Performance
- ✅ **Instant prefetching** - Preload pages on hover
- ✅ **Progress indicator** - Visual feedback during navigation
- ✅ **Minified HTML/CSS/JS** - Reduced payload size
- ✅ **Optimized search** - Better search performance

---

## Implementation Steps

### For Objective 1 (Navigation & Structure):

1. **Create new directories:**
   ```bash
   cd /home/rdan/my_projects/danube_docs
   mkdir -p docs/concepts
   mkdir -p docs/contributing
   mkdir -p docs/assets/img/architecture
   mkdir -p docs/assets/img/concepts
   mkdir -p docs/assets/img/getting_started
   ```

2. **Move markdown files to concepts/:**
   ```bash
   mv docs/architecture/topics.md docs/concepts/
   mv docs/architecture/subscriptions.md docs/concepts/
   mv docs/architecture/dispatch_strategy.md docs/concepts/
   mv docs/architecture/messages.md docs/concepts/
   mv docs/architecture/messaging_patterns_queuing_vs_pubsub.md docs/concepts/
   mv docs/architecture/messaging_modes_pubsub_vs_streaming.md docs/concepts/
   ```

3. **Move markdown files to contributing/:**
   ```bash
   mv docs/development/dev_environment.md docs/contributing/
   mv docs/development/internal_resources.md docs/contributing/
   rmdir docs/development
   ```

4. **Move images to centralized assets folder:**
   ```bash
   # Architecture images
   mv docs/architecture/img/Danube_architecture.png docs/assets/img/architecture/
   mv docs/architecture/img/Wal_Cloud.png docs/assets/img/architecture/
   
   # Concepts images
   mv docs/architecture/img/partitioned_topics.png docs/assets/img/concepts/
   mv docs/architecture/img/producers_consumers.png docs/assets/img/concepts/
   mv docs/architecture/img/exclusive_subscription_non_partitioned.png docs/assets/img/concepts/
   mv docs/architecture/img/exclusive_subscription_partitioned.png docs/assets/img/concepts/
   mv docs/architecture/img/shared_subscription_non_partitioned.png docs/assets/img/concepts/
   mv docs/architecture/img/shared_subscription_partitioned.png docs/assets/img/concepts/
   mv docs/architecture/img/failover_subscription_non_partitioned.png docs/assets/img/concepts/
   mv docs/architecture/img/failover_subscription_partitioned.png docs/assets/img/concepts/
   
   # Getting started images (if exist)
   if [ -d "docs/getting_started/img" ]; then
       mv docs/getting_started/img/* docs/assets/img/getting_started/
       rmdir docs/getting_started/img
   fi
   
   # Remove empty architecture/img directory
   rmdir docs/architecture/img
   ```

5. **Update image paths in moved files:**
   
   In `docs/concepts/topics.md`:
   ```bash
   sed -i 's|img/partitioned_topics.png|../assets/img/concepts/partitioned_topics.png|g' docs/concepts/topics.md
   ```
   
   In `docs/concepts/subscriptions.md`:
   ```bash
   sed -i 's|img/producers_consumers.png|../assets/img/concepts/producers_consumers.png|g' docs/concepts/subscriptions.md
   sed -i 's|img/exclusive_subscription_|../assets/img/concepts/exclusive_subscription_|g' docs/concepts/subscriptions.md
   sed -i 's|img/shared_subscription_|../assets/img/concepts/shared_subscription_|g' docs/concepts/subscriptions.md
   sed -i 's|img/failover_subscription_|../assets/img/concepts/failover_subscription_|g' docs/concepts/subscriptions.md
   ```

6. **Update image paths in architecture files:**
   
   In `docs/architecture/architecture.md`:
   ```bash
   sed -i 's|img/Danube_architecture.png|../assets/img/architecture/Danube_architecture.png|g' docs/architecture/architecture.md
   ```
   
   In `docs/architecture/persistence.md`:
   ```bash
   sed -i 's|img/Wal_Cloud.png|../assets/img/architecture/Wal_Cloud.png|g' docs/architecture/persistence.md
   ```
   
   In `docs/getting_started/*.md` (if images exist):
   ```bash
   sed -i 's|img/|../assets/img/getting_started/|g' docs/getting_started/*.md
   ```

7. **Update mkdocs.yml** with new navigation structure

8. **Verify all image links work:**
   ```bash
   # Test build to check for broken links
   mkdocs build --strict
   ```

### For Objective 2 (MkDocs Configuration):

1. **Update requirements.txt** with new dependencies

2. **Install new packages:**
   ```bash
   pip install -r requirements.txt
   ```

3. **Update mkdocs.yml** with enhanced configuration

4. **Create supporting files:**
   ```bash
   mkdir -p docs/stylesheets docs/javascripts docs/includes
   touch docs/stylesheets/extra.css
   touch docs/javascripts/extra.js
   touch docs/javascripts/mathjax.js
   touch docs/includes/abbreviations.md
   ```

5. **Add content to new files** (CSS, JS, abbreviations)

6. **Test locally:**
   ```bash
   mkdocs serve
   ```

7. **Build and deploy:**
   ```bash
   mkdocs build
   mkdocs gh-deploy
   ```

---

## Expected Results

### Before vs After Comparison

**Navigation:**
- Before: Getting Started at level 1, Architecture at level 2
- After: Getting Started → Concepts → Clients → CLI → Architecture (progressive)

**User Journey:**
- Before: See architecture before trying product
- After: Run it → Learn concepts → Use clients → Understand internals

**Professionalism:**
- Before: Basic Material theme, minimal features
- After: Modern UI with tabs, instant loading, social cards, enhanced search

**Content Discovery:**
- Before: CLI tools buried 3 levels deep
- After: CLI tools at top level with clear organization

**Visual Appeal:**
- Before: Plain documentation
- After: Icons, emojis, diagrams, admonitions, code copy buttons

---

## Success Metrics

- ⬆️ Faster time to "hello world" (better Getting Started placement)
- ⬆️ Reduced bounce rate (better first impression)
- ⬆️ Increased page views per session (easier navigation)
- ⬆️ Better SEO rankings (social cards, meta descriptions)
- ⬆️ More GitHub stars (professional appearance)
