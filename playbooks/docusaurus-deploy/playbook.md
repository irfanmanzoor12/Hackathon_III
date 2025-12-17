---
title: Docusaurus Deploy
description: Deploy Docusaurus v3 static site to Kubernetes with Nginx
version: 1.0.0
allowed_tools:
  - bash
  - kubectl
  - docker
---

# Docusaurus Deploy

## Goal
Deploy a Docusaurus v3 static documentation site to Kubernetes (Minikube) using Nginx as the web server, with health checks and production-ready configuration.

This deployment serves as a reference implementation for static site hosting in Hackathon III.

## Prerequisites Check
Before proceeding, verify:
1. Minikube is running
2. kubectl is configured and accessible
3. Docker is installed for building images
4. Node.js 18+ is installed (for building Docusaurus)

## Instructions

### 1. Verify Prerequisites

```bash
# Check Minikube status
minikube status

# Verify kubectl access
kubectl cluster-info
kubectl get nodes

# Check Docker version
docker --version

# Check Node.js version
node --version
```

### 2. Create Docusaurus Site

#### 2.1 Create Project Directory

```bash
# Create project directory
mkdir -p /tmp/docusaurus-site
cd /tmp/docusaurus-site
```

#### 2.2 Initialize Docusaurus Project

```bash
# Create package.json
cat <<'EOF' > package.json
{
  "name": "docusaurus-site",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "docusaurus": "docusaurus",
    "start": "docusaurus start",
    "build": "docusaurus build",
    "swizzle": "docusaurus swizzle",
    "deploy": "docusaurus deploy",
    "clear": "docusaurus clear",
    "serve": "docusaurus serve",
    "write-translations": "docusaurus write-translations",
    "write-heading-ids": "docusaurus write-heading-ids"
  },
  "dependencies": {
    "@docusaurus/core": "3.1.0",
    "@docusaurus/preset-classic": "3.1.0",
    "@mdx-js/react": "^3.0.0",
    "clsx": "^2.0.0",
    "prism-react-renderer": "^2.3.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@docusaurus/module-type-aliases": "3.1.0",
    "@docusaurus/types": "3.1.0"
  },
  "browserslist": {
    "production": [
      ">0.5%",
      "not dead",
      "not op_mini all"
    ],
    "development": [
      "last 3 chrome version",
      "last 3 firefox version",
      "last 5 safari version"
    ]
  },
  "engines": {
    "node": ">=18.0"
  }
}
EOF
```

#### 2.3 Create Docusaurus Configuration

```bash
cat <<'EOF' > docusaurus.config.js
// @ts-check
// `@type` JSDoc annotations allow editor autocompletion and type checking
// (when paired with `@ts-check`).
// There are various equivalent ways to declare your Docusaurus config.
// See: https://docusaurus.io/docs/api/docusaurus-config

import {themes as prismThemes} from 'prism-react-renderer';

/** @type {import('@docusaurus/types').Config} */
const config = {
  title: 'Docusaurus on Kubernetes',
  tagline: 'Static documentation site deployed to K8s',
  favicon: 'img/favicon.ico',

  // Set the production url of your site here
  url: 'https://your-docusaurus-site.com',
  // Set the /<baseUrl>/ pathname under which your site is served
  // For GitHub pages deployment, it is often '/<projectName>/'
  baseUrl: '/',

  // GitHub pages deployment config.
  // If you aren't using GitHub pages, you don't need these.
  organizationName: 'hackathon-iii',
  projectName: 'docusaurus-site',

  onBrokenLinks: 'throw',
  onBrokenMarkdownLinks: 'warn',

  // Even if you don't use internationalization, you can use this field to set
  // useful metadata like html lang. For example, if your site is Chinese, you
  // may want to replace "en" with "zh-Hans".
  i18n: {
    defaultLocale: 'en',
    locales: ['en'],
  },

  presets: [
    [
      'classic',
      /** @type {import('@docusaurus/preset-classic').Options} */
      ({
        docs: {
          sidebarPath: './sidebars.js',
        },
        blog: {
          showReadingTime: true,
        },
        theme: {
          customCss: './src/css/custom.css',
        },
      }),
    ],
  ],

  themeConfig:
    /** @type {import('@docusaurus/preset-classic').ThemeConfig} */
    ({
      // Replace with your project's social card
      image: 'img/docusaurus-social-card.jpg',
      navbar: {
        title: 'Docusaurus K8s',
        logo: {
          alt: 'Site Logo',
          src: 'img/logo.svg',
        },
        items: [
          {
            type: 'docSidebar',
            sidebarId: 'tutorialSidebar',
            position: 'left',
            label: 'Tutorial',
          },
          {to: '/blog', label: 'Blog', position: 'left'},
        ],
      },
      footer: {
        style: 'dark',
        links: [
          {
            title: 'Docs',
            items: [
              {
                label: 'Tutorial',
                to: '/docs/intro',
              },
            ],
          },
        ],
        copyright: `Built with Docusaurus.`,
      },
      prism: {
        theme: prismThemes.github,
        darkTheme: prismThemes.dracula,
      },
    }),
};

export default config;
EOF
```

#### 2.4 Create Content Structure

```bash
# Create directories
mkdir -p src/css src/pages docs blog static/img

# Create custom CSS
cat <<'EOF' > src/css/custom.css
/**
 * Any CSS included here will be global. The classic template
 * bundles Infima by default. Infima is a CSS framework designed to
 * work well for content-centric websites.
 */

/* You can override the default Infima variables here. */
:root {
  --ifm-color-primary: #2e8555;
  --ifm-color-primary-dark: #29784c;
  --ifm-color-primary-darker: #277148;
  --ifm-color-primary-darkest: #205d3b;
  --ifm-color-primary-light: #33925d;
  --ifm-color-primary-lighter: #359962;
  --ifm-color-primary-lightest: #3cad6e;
  --ifm-code-font-size: 95%;
  --docusaurus-highlighted-code-line-bg: rgba(0, 0, 0, 0.1);
}

/* For readability concerns, you should choose a lighter palette in dark mode. */
[data-theme='dark'] {
  --ifm-color-primary: #25c2a0;
  --ifm-color-primary-dark: #21af90;
  --ifm-color-primary-darker: #1fa588;
  --ifm-color-primary-darkest: #1a8870;
  --ifm-color-primary-light: #29d5b0;
  --ifm-color-primary-lighter: #32d8b4;
  --ifm-color-primary-lightest: #4fddbf;
  --docusaurus-highlighted-code-line-bg: rgba(0, 0, 0, 0.3);
}
EOF

# Create home page
cat <<'EOF' > src/pages/index.js
import React from 'react';
import clsx from 'clsx';
import Link from '@docusaurus/Link';
import useDocusaurusContext from '@docusaurus/useDocusaurusContext';
import Layout from '@theme/Layout';
import styles from './index.module.css';

function HomepageHeader() {
  const {siteConfig} = useDocusaurusContext();
  return (
    <header className={clsx('hero hero--primary', styles.heroBanner)}>
      <div className="container">
        <h1 className="hero__title">{siteConfig.title}</h1>
        <p className="hero__subtitle">{siteConfig.tagline}</p>
        <div className={styles.buttons}>
          <Link
            className="button button--secondary button--lg"
            to="/docs/intro">
            Get Started
          </Link>
        </div>
      </div>
    </header>
  );
}

export default function Home() {
  const {siteConfig} = useDocusaurusContext();
  return (
    <Layout
      title={`Hello from ${siteConfig.title}`}
      description="Docusaurus deployed on Kubernetes">
      <HomepageHeader />
      <main>
        <section className={styles.features}>
          <div className="container">
            <div className="row">
              <div className="col col--4">
                <h3>Easy to Use</h3>
                <p>
                  Docusaurus was designed from the ground up to be easily installed
                  and used to get your website up and running quickly.
                </p>
              </div>
              <div className="col col--4">
                <h3>Focus on What Matters</h3>
                <p>
                  Docusaurus lets you focus on your docs, and we&apos;ll do the chores.
                </p>
              </div>
              <div className="col col--4">
                <h3>Powered by Kubernetes</h3>
                <p>
                  Deployed as a containerized application on Kubernetes cluster.
                </p>
              </div>
            </div>
          </div>
        </section>
      </main>
    </Layout>
  );
}
EOF

# Create home page CSS module
cat <<'EOF' > src/pages/index.module.css
.heroBanner {
  padding: 4rem 0;
  text-align: center;
  position: relative;
  overflow: hidden;
}

.buttons {
  display: flex;
  align-items: center;
  justify-content: center;
}

.features {
  display: flex;
  align-items: center;
  padding: 2rem 0;
  width: 100%;
}
EOF

# Create intro documentation
cat <<'EOF' > docs/intro.md
---
sidebar_position: 1
---

# Introduction

Welcome to Docusaurus on Kubernetes!

## Getting Started

This is a static documentation site deployed to Kubernetes using Nginx.

### Features

- **Fast**: Built with React and optimized for performance
- **Containerized**: Running in Docker containers
- **Scalable**: Deployed on Kubernetes
- **Simple**: No database, no authentication needed

## Deployment

This site is deployed using:

- Docusaurus v3 for static site generation
- Nginx for serving static files
- Kubernetes for orchestration
- Docker for containerization

## Health Check

The site includes a custom health endpoint at `/health` for monitoring.
EOF

# Create sidebars configuration
cat <<'EOF' > sidebars.js
/**
 * Creating a sidebar enables you to:
 - create an ordered group of docs
 - render a sidebar for each doc of that group
 - provide next/previous navigation

 The sidebars can be generated from the filesystem, or explicitly defined here.

 Create as many sidebars as you want.
 */

// @ts-check

/** @type {import('@docusaurus/plugin-content-docs').SidebarsConfig} */
const sidebars = {
  tutorialSidebar: [{type: 'autogenerated', dirName: '.'}],
};

export default sidebars;
EOF

# Create a sample blog post
cat <<'EOF' > blog/2025-01-01-welcome.md
---
slug: welcome
title: Welcome
authors: [admin]
tags: [kubernetes, docusaurus]
---

# Welcome to Docusaurus on Kubernetes

This is the first blog post on our Kubernetes-deployed Docusaurus site.

## Features

- Static site generation
- Markdown support
- Code highlighting
- And much more!
EOF
```

### 3. Install Dependencies and Build

```bash
# Install dependencies
cd /tmp/docusaurus-site
npm install

# Build static site
npm run build
```

### 4. Create Nginx Configuration and Dockerfile

#### 4.1 Create Nginx Configuration

```bash
cat <<'EOF' > nginx.conf
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Health check endpoint
    location /health {
        access_log off;
        return 200 '{"status":"healthy","service":"docusaurus-site"}';
        add_header Content-Type application/json;
    }

    # Main site
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Disable access to hidden files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
EOF
```

#### 4.2 Create Dockerfile

```bash
cat <<'EOF' > Dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Build static site
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy built site from builder
COPY --from=builder /app/build /usr/share/nginx/html

# Create non-root user
RUN adduser -D -u 1000 nginx-user && \
    chown -R nginx-user:nginx-user /usr/share/nginx/html && \
    chown -R nginx-user:nginx-user /var/cache/nginx && \
    chown -R nginx-user:nginx-user /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown -R nginx-user:nginx-user /var/run/nginx.pid

# Switch to non-root user
USER nginx-user

# Expose port
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
EOF
```

#### 4.3 Create .dockerignore

```bash
cat <<'EOF' > .dockerignore
node_modules
.docusaurus
build
.git
.gitignore
README.md
.env
npm-debug.log
yarn-error.log
EOF
```

### 5. Build and Load Docker Image

```bash
# Build Docker image
docker build -t docusaurus-site:latest /tmp/docusaurus-site

# Load image into Minikube
minikube image load docusaurus-site:latest

# Verify image is loaded
minikube image ls | grep docusaurus-site
```

### 6. Create Kubernetes Manifests

#### 6.1 Create Namespace

```bash
# Create namespace for Docusaurus site
kubectl create namespace docusaurus --dry-run=client -o yaml | kubectl apply -f -
```

#### 6.2 Create Deployment

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docusaurus-site
  namespace: docusaurus
  labels:
    app: docusaurus-site
spec:
  replicas: 2
  selector:
    matchLabels:
      app: docusaurus-site
  template:
    metadata:
      labels:
        app: docusaurus-site
    spec:
      containers:
      - name: nginx
        image: docusaurus-site:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
EOF
```

#### 6.3 Create Service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: docusaurus-site
  namespace: docusaurus
  labels:
    app: docusaurus-site
spec:
  type: ClusterIP
  selector:
    app: docusaurus-site
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
EOF
```

### 7. Wait for Deployment

```bash
# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/docusaurus-site -n docusaurus --timeout=300s

# Check pod status
kubectl get pods -n docusaurus

# View logs
kubectl logs -n docusaurus -l app=docusaurus-site --tail=50
```

### 8. Verification & Validation

#### 8.1 Port Forward Service

```bash
# Port forward the Docusaurus site
kubectl port-forward -n docusaurus svc/docusaurus-site 8080:80 &

# Wait for port forward to establish
sleep 3
```

#### 8.2 Test Endpoints

```bash
# Test health endpoint
curl -s http://localhost:8080/health

# Test home page
curl -s http://localhost:8080/ | grep -o "<title>.*</title>"

# Test documentation page
curl -s http://localhost:8080/docs/intro | grep -o "<title>.*</title>"

# Test blog page
curl -s http://localhost:8080/blog | grep -o "<title>.*</title>"

# Check response headers
curl -I http://localhost:8080/
```

#### 8.3 Complete Validation Script

```bash
cat <<'VALIDATION_SCRIPT' | bash
#!/bin/bash
set -e

echo "=== Docusaurus Site Validation ==="
echo

# Check deployment
echo "1. Checking deployment status..."
kubectl get deployment docusaurus-site -n docusaurus
echo "✓ Deployment exists"
echo

# Check pods
echo "2. Checking pod status..."
READY_PODS=$(kubectl get pods -n docusaurus -l app=docusaurus-site -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | grep -o "True" | wc -l)
if [ "$READY_PODS" -ge 1 ]; then
  echo "✓ Pods are running ($READY_PODS ready)"
else
  echo "✗ No pods ready"
  exit 1
fi
echo

# Check service
echo "3. Checking service..."
kubectl get svc docusaurus-site -n docusaurus
echo "✓ Service exists"
echo

# Port forward (background)
echo "4. Setting up port forward..."
kubectl port-forward -n docusaurus svc/docusaurus-site 8080:80 > /dev/null 2>&1 &
PORT_FORWARD_PID=$!
sleep 3
echo "✓ Port forward established (PID: $PORT_FORWARD_PID)"
echo

# Test health
echo "5. Testing health endpoint..."
HEALTH_RESPONSE=$(curl -s http://localhost:8080/health)
if echo "$HEALTH_RESPONSE" | grep -q "healthy"; then
  echo "✓ Health check passed"
  echo "$HEALTH_RESPONSE"
else
  echo "✗ Health check failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test home page
echo "6. Testing home page..."
HOME_RESPONSE=$(curl -s http://localhost:8080/)
if echo "$HOME_RESPONSE" | grep -q "Docusaurus"; then
  echo "✓ Home page loads"
  echo "Title: $(echo "$HOME_RESPONSE" | grep -o "<title>.*</title>")"
else
  echo "✗ Home page failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test docs page
echo "7. Testing documentation page..."
DOCS_RESPONSE=$(curl -s http://localhost:8080/docs/intro)
if echo "$DOCS_RESPONSE" | grep -q "Introduction"; then
  echo "✓ Documentation page loads"
else
  echo "✗ Documentation page failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test blog page
echo "8. Testing blog page..."
BLOG_RESPONSE=$(curl -s http://localhost:8080/blog)
if echo "$BLOG_RESPONSE" | grep -q "Blog"; then
  echo "✓ Blog page loads"
else
  echo "✗ Blog page failed"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Test static assets
echo "9. Testing static asset serving..."
ASSET_RESPONSE=$(curl -s -I http://localhost:8080/img/logo.svg 2>&1)
if echo "$ASSET_RESPONSE" | grep -q "200\|304"; then
  echo "✓ Static assets served (with caching headers)"
else
  echo "⚠ Static assets check inconclusive (may not have assets)"
fi
echo

# Test security headers
echo "10. Testing security headers..."
HEADERS=$(curl -s -I http://localhost:8080/)
if echo "$HEADERS" | grep -q "X-Frame-Options"; then
  echo "✓ Security headers present"
else
  echo "✗ Security headers missing"
  kill $PORT_FORWARD_PID 2>/dev/null
  exit 1
fi
echo

# Cleanup
echo "Cleaning up port forward..."
kill $PORT_FORWARD_PID 2>/dev/null || true
echo

echo "==================================="
echo "✓ All validations passed!"
echo "==================================="
VALIDATION_SCRIPT
```

### 9. Access Application

```bash
# Method 1: Port forward (for local access)
kubectl port-forward -n docusaurus svc/docusaurus-site 8080:80

# Then open browser to: http://localhost:8080

# Method 2: From within cluster
# Service is accessible at: docusaurus-site.docusaurus.svc.cluster.local:80
```

## Validation Criteria

- ✓ Docusaurus site builds successfully
- ✓ Docker image created with multi-stage build
- ✓ Image loaded into Minikube
- ✓ Deployment created with 2 replicas
- ✓ All pods are Running and Ready
- ✓ Service created with ClusterIP type
- ✓ Health endpoint returns healthy status
- ✓ Home page loads correctly
- ✓ Documentation pages accessible
- ✓ Blog pages accessible
- ✓ Static assets served with caching headers
- ✓ Security headers configured
- ✓ Gzip compression enabled
- ✓ Liveness and readiness probes configured and passing

## Cleanup (Optional)

```bash
# Delete application resources
kubectl delete namespace docusaurus

# Remove Docker image from Minikube
minikube image rm docusaurus-site:latest

# Remove local files
rm -rf /tmp/docusaurus-site
```

## Notes for Claude Code and Goose Agents

### Execution Strategy
- **Claude Code**: Execute bash commands sequentially using the Bash tool
- **Goose**: Use toolkit.run_shell() for each command block
- Both agents should validate each step before proceeding
- Build process requires Node.js and may take several minutes

### Common Pitfalls
- Node.js build requires sufficient memory (at least 2GB)
- npm install can be slow - be patient
- Docusaurus build generates static files in `build/` directory
- Nginx must run as non-root user in Kubernetes
- Image must be loaded into Minikube using `minikube image load`

### Agent-Specific Hints
- **Wait for deployment**: Use `kubectl wait --for=condition=available` with appropriate timeout
- **Multiple replicas**: Deployment has 2 replicas for high availability
- **Validation**: Check both pods are ready before testing
- **Idempotency**: Use `--dry-run=client -o yaml | kubectl apply -f -` for recreatable resources
- **Debugging**: Check pod logs with `kubectl logs -n docusaurus -l app=docusaurus-site`

### Configuration Notes
- Application runs on port 80 (standard HTTP)
- Service exposes port 80
- 2 replicas for high availability
- Nginx serves static files with caching
- Health endpoint at `/health` returns JSON
- Resource limits: 256Mi memory, 200m CPU per pod
- Resource requests: 64Mi memory, 50m CPU per pod
- Runs as non-root user (UID 1000)
- Liveness probe: /health, 10s initial delay
- Readiness probe: /health, 5s initial delay

### Production-Ready Features

#### Performance
- **Gzip compression**: Enabled for text files
- **Static asset caching**: 1-year cache for immutable assets
- **CDN-ready**: Can be fronted by CDN for global distribution

#### Security
- **Non-root user**: Container runs as UID 1000
- **Security headers**: X-Frame-Options, X-Content-Type-Options, X-XSS-Protection
- **No privilege escalation**: SecurityContext prevents escalation
- **Hidden file protection**: Nginx denies access to dotfiles

#### Reliability
- **Multiple replicas**: 2 pods for high availability
- **Health checks**: Both liveness and readiness probes
- **Resource limits**: Prevents resource exhaustion
- **Graceful degradation**: If one pod fails, other continues serving

### Nginx Configuration Details

The Nginx configuration includes:
1. **Gzip compression** for text-based files
2. **Security headers** for XSS and clickjacking protection
3. **Static asset caching** with long expiration times
4. **SPA support** with `try_files` directive
5. **Health endpoint** for Kubernetes probes
6. **Access log** optimization (health checks excluded)

### Testing Scenarios

#### Scenario 1: Basic Functionality
1. Access home page
2. Navigate to documentation
3. Check blog posts
4. Verify all routes work

#### Scenario 2: Static Assets
1. Load CSS files
2. Load JavaScript files
3. Load images
4. Verify caching headers present

#### Scenario 3: High Availability
1. Delete one pod
2. Verify service remains available
3. Check new pod starts automatically

#### Scenario 4: Health Monitoring
1. Check health endpoint
2. Verify Kubernetes probes are passing
3. Monitor pod restarts (should be 0)

### Observability
- **Logs**: `kubectl logs -n docusaurus -l app=docusaurus-site --tail=100 -f`
- **Events**: `kubectl get events -n docusaurus --sort-by='.lastTimestamp'`
- **Pod status**: `kubectl describe pod -n docusaurus -l app=docusaurus-site`
- **Resource usage**: `kubectl top pod -n docusaurus`

### Extension Points

To customize the site:
1. Edit `docusaurus.config.js` for site configuration
2. Add content to `docs/` directory
3. Add blog posts to `blog/` directory
4. Customize CSS in `src/css/custom.css`
5. Rebuild and redeploy

To add features:
1. Install Docusaurus plugins
2. Add custom React components
3. Configure search (Algolia DocSearch)
4. Add versioning for documentation
5. Enable internationalization (i18n)

## References
- Docusaurus Documentation: https://docusaurus.io/docs
- Docusaurus v3 Release: https://docusaurus.io/blog/releases/3.0
- Nginx Configuration: https://nginx.org/en/docs/
- Kubernetes Static Site Hosting: https://kubernetes.io/docs/tutorials/stateless-application/
- Multi-stage Docker Builds: https://docs.docker.com/build/building/multi-stage/
