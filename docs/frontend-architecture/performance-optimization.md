[Back to Index](./index.md)

# Performance Optimization

## Bundle Size Optimization

### 1. Lazy Loading

Load feature modules on demand to reduce initial bundle size:

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'jobs',
    loadChildren: () => import('./features/jobs/jobs.routes').then(m => m.JOBS_ROUTES),
    data: { preload: true } // Preload after initial load
  },
  {
    path: 'costs',
    loadChildren: () => import('./features/costs/costs.routes').then(m => m.COSTS_ROUTES)
    // Not preloaded - only loaded when user navigates
  }
];
```

### 2. Code Splitting

Split large components and libraries:

```typescript
// Lazy load heavy dependencies
export class ReportComponent {
  async generatePDF() {
    const { jsPDF } = await import('jspdf');
    // Use jsPDF
  }

  async exportExcel() {
    const XLSX = await import('xlsx');
    // Use XLSX
  }
}
```

### 3. Tree Shaking

Ensure proper imports for tree shaking:

```typescript
// ✅ Good - Allows tree shaking
import { map, filter } from 'rxjs/operators';

// ❌ Bad - Imports entire library
import * as operators from 'rxjs/operators';

// ✅ Good - Named imports
import { Component, OnInit } from '@angular/core';

// ❌ Bad - Namespace import
import * as ng from '@angular/core';
```

### 4. Bundle Analysis

Monitor bundle size with webpack-bundle-analyzer:

```bash
# Analyze production bundle
nx build mobile-app --configuration=production --stats-json
npx webpack-bundle-analyzer dist/apps/mobile-app/stats.json
```

### 5. Image Optimization

Use optimized image formats:

```typescript
// Prefer WebP with fallback
<picture>
  <source srcset="image.webp" type="image/webp">
  <source srcset="image.jpg" type="image/jpeg">
  <img src="image.jpg" alt="Description">
</picture>

// Lazy load images
<img loading="lazy" src="image.jpg" alt="Description">
```

## Runtime Performance

### 1. Change Detection Strategy

Use OnPush for presentational components:

```typescript
@Component({
  selector: 'app-job-card',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `...`
})
export class JobCardComponent {
  @Input() job!: Job; // Immutable input
}
```

### 2. Virtual Scrolling

Use CDK Virtual Scroll for long lists:

```typescript
import { CdkVirtualScrollViewport, ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  selector: 'app-jobs-list',
  standalone: true,
  imports: [ScrollingModule],
  template: `
    <cdk-virtual-scroll-viewport itemSize="80" class="jobs-viewport">
      <div *cdkVirtualFor="let job of jobs" class="job-item">
        <app-job-card [job]="job"></app-job-card>
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`
    .jobs-viewport {
      height: 600px;
    }
  `]
})
export class JobsListComponent {
  @Input() jobs: Job[] = [];
}
```

### 3. TrackBy Functions

Always use trackBy with ngFor:

```typescript
@Component({
  template: `
    @for (job of jobs(); track trackByJobId($index, job)) {
      <app-job-card [job]="job"></app-job-card>
    }
  `
})
export class JobsListComponent {
  trackByJobId(index: number, job: Job): string {
    return job.id;
  }
}
```

### 4. Debounce User Input

Debounce search and filter operations:

```typescript
import { debounceTime, distinctUntilChanged } from 'rxjs/operators';

export class JobsSearchComponent {
  searchControl = new FormControl('');

  constructor() {
    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      takeUntilDestroyed()
    ).subscribe(term => {
      this.performSearch(term);
    });
  }
}
```

### 5. Memoize Expensive Computations

Use computed signals for derived state:

```typescript
export class JobDetailComponent {
  job = signal<Job | null>(null);

  // Memoized computation
  totalCosts = computed(() => {
    const job = this.job();
    if (!job) return 0;
    return job.costs.reduce((sum, cost) => sum + cost.amount, 0);
  });
}
```

## PWA Optimization

### 1. Service Worker Configuration

Configure caching strategies:

```json
// ngsw-config.json
{
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "resources": {
        "files": [
          "/favicon.ico",
          "/index.html",
          "/manifest.json",
          "/*.css",
          "/*.js"
        ]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": [
          "/assets/**",
          "/*.(svg|cur|jpg|jpeg|png|apng|webp|gif)"
        ]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-calls",
      "urls": [
        "https://api.findogai.com/**"
      ],
      "cacheConfig": {
        "strategy": "freshness",
        "maxSize": 100,
        "maxAge": "3d",
        "timeout": "10s"
      }
    },
    {
      "name": "firestore-cache",
      "urls": [
        "https://firestore.googleapis.com/**"
      ],
      "cacheConfig": {
        "strategy": "performance",
        "maxSize": 500,
        "maxAge": "7d"
      }
    }
  ]
}
```

### 2. Precaching Critical Resources

Precache essential assets:

```typescript
// app.config.ts
import { provideServiceWorker } from '@angular/service-worker';

export const appConfig: ApplicationConfig = {
  providers: [
    provideServiceWorker('ngsw-worker.js', {
      enabled: environment.production,
      registrationStrategy: 'registerWhenStable:30000'
    })
  ]
};
```

### 3. Resource Hints

Use preload and prefetch:

```html
<!-- index.html -->
<head>
  <!-- Preload critical resources -->
  <link rel="preload" href="styles.css" as="style">
  <link rel="preload" href="main.js" as="script">

  <!-- Prefetch next page resources -->
  <link rel="prefetch" href="jobs-module.js">

  <!-- DNS prefetch for external domains -->
  <link rel="dns-prefetch" href="https://firestore.googleapis.com">
</head>
```

## Loading Performance

### 1. Skeleton Screens

Show skeleton UI while loading:

```typescript
@Component({
  template: `
    @if (loading()) {
      <div class="skeleton-card">
        <div class="skeleton-header"></div>
        <div class="skeleton-content"></div>
      </div>
    } @else {
      <app-job-card [job]="job()"></app-job-card>
    }
  `,
  styles: [`
    .skeleton-card {
      animation: pulse 1.5s infinite;
    }
    .skeleton-header,
    .skeleton-content {
      background: linear-gradient(90deg, #f0f0f0 25%, #e0e0e0 50%, #f0f0f0 75%);
      background-size: 200% 100%;
      animation: loading 1.5s infinite;
    }
    @keyframes loading {
      0% { background-position: 200% 0; }
      100% { background-position: -200% 0; }
    }
  `]
})
export class JobCardSkeletonComponent { }
```

### 2. Progressive Enhancement

Load content progressively:

```typescript
export class JobDetailComponent implements OnInit {
  async ngOnInit() {
    // Load critical data first
    await this.loadJobBasicInfo();

    // Then load additional data
    this.loadJobCosts();
    this.loadJobAdvances();
    this.loadJobJourneys();
  }
}
```

### 3. Intersection Observer

Lazy load images when visible:

```typescript
@Directive({
  selector: '[appLazyLoad]',
  standalone: true
})
export class LazyLoadDirective implements OnInit, OnDestroy {
  @Input() appLazyLoad!: string;

  private observer?: IntersectionObserver;

  constructor(private el: ElementRef) {}

  ngOnInit() {
    this.observer = new IntersectionObserver(entries => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          this.loadImage();
          this.observer?.unobserve(this.el.nativeElement);
        }
      });
    });

    this.observer.observe(this.el.nativeElement);
  }

  private loadImage() {
    const img = this.el.nativeElement as HTMLImageElement;
    img.src = this.appLazyLoad;
  }

  ngOnDestroy() {
    this.observer?.disconnect();
  }
}
```

## Memory Management

### 1. Unsubscribe from Observables

Use takeUntilDestroyed:

```typescript
export class JobsListComponent {
  constructor() {
    this.store.select(selectAllJobs).pipe(
      takeUntilDestroyed() // Automatic cleanup
    ).subscribe(jobs => {
      this.jobs.set(jobs);
    });
  }
}
```

### 2. Avoid Memory Leaks

Clean up listeners and timers:

```typescript
export class TimerComponent implements OnDestroy {
  private timerId?: number;

  startTimer() {
    this.timerId = window.setInterval(() => {
      // Timer logic
    }, 1000);
  }

  ngOnDestroy() {
    if (this.timerId) {
      clearInterval(this.timerId);
    }
  }
}
```

### 3. Limit Cached Data

Set reasonable cache limits:

```typescript
// Firestore configuration
const firestoreConfig = {
  cacheSizeBytes: 50 * 1024 * 1024, // 50MB max cache
  experimentalForceLongPolling: false
};
```

## Network Performance

### 1. HTTP Compression

Enable gzip/brotli compression:

```typescript
// angular.json
{
  "projects": {
    "mobile-app": {
      "architect": {
        "build": {
          "configurations": {
            "production": {
              "optimization": {
                "scripts": true,
                "styles": {
                  "minify": true,
                  "inlineCritical": true
                },
                "fonts": true
              }
            }
          }
        }
      }
    }
  }
}
```

### 2. Request Batching

Batch multiple requests:

```typescript
export class BatchService {
  private pendingRequests: Array<() => Promise<any>> = [];
  private batchTimer?: number;

  addRequest(request: () => Promise<any>) {
    this.pendingRequests.push(request);

    if (!this.batchTimer) {
      this.batchTimer = window.setTimeout(() => {
        this.executeBatch();
      }, 100);
    }
  }

  private async executeBatch() {
    const requests = [...this.pendingRequests];
    this.pendingRequests = [];
    this.batchTimer = undefined;

    await Promise.all(requests.map(req => req()));
  }
}
```

### 3. Request Caching

Cache API responses:

```typescript
@Injectable({ providedIn: 'root' })
export class CachedApiService {
  private cache = new Map<string, { data: any; timestamp: number }>();
  private readonly CACHE_TTL = 5 * 60 * 1000; // 5 minutes

  async get<T>(key: string, fetcher: () => Promise<T>): Promise<T> {
    const cached = this.cache.get(key);

    if (cached && Date.now() - cached.timestamp < this.CACHE_TTL) {
      return cached.data;
    }

    const data = await fetcher();
    this.cache.set(key, { data, timestamp: Date.now() });
    return data;
  }
}
```

## Performance Monitoring

### 1. Web Vitals

Track Core Web Vitals:

```typescript
import { onCLS, onFID, onLCP } from 'web-vitals';

export class PerformanceMonitor {
  static init() {
    onCLS(console.log); // Cumulative Layout Shift
    onFID(console.log); // First Input Delay
    onLCP(console.log); // Largest Contentful Paint
  }
}
```

### 2. Performance Marks

Use Performance API:

```typescript
export class JobsService {
  async loadJobs() {
    performance.mark('jobs-load-start');

    const jobs = await this.fetchJobs();

    performance.mark('jobs-load-end');
    performance.measure('jobs-load', 'jobs-load-start', 'jobs-load-end');

    const measure = performance.getEntriesByName('jobs-load')[0];
    console.log(`Jobs loaded in ${measure.duration}ms`);

    return jobs;
  }
}
```

## Performance Checklist

- [ ] Lazy load all feature modules
- [ ] Use OnPush change detection for presentational components
- [ ] Implement virtual scrolling for long lists
- [ ] Add trackBy functions to all loops
- [ ] Debounce user input operations
- [ ] Optimize images (WebP format, lazy loading)
- [ ] Configure service worker caching
- [ ] Minimize bundle size (< 500KB initial)
- [ ] Achieve LCP < 2.5s
- [ ] Achieve FID < 100ms
- [ ] Monitor and fix memory leaks

---

[← Previous: Testing Strategy](./testing-strategy.md) | [Back to Index](./index.md) | [Next: Performance Monitoring →](./performance-monitoring.md)
