[Back to Index](./index.md)

# Routing

## Complete Routing Implementation

This section covers the complete routing configuration for the FinDogAI application, including guards, resolvers, and lazy loading strategies.

## Main Application Routes

```typescript
// app/app.routes.ts - Main application routes
import { Routes } from '@angular/router';
import { AuthGuard } from '@core/guards/auth.guard';
import { TenantGuard } from '@core/guards/tenant.guard';
import { OnboardingGuard } from '@core/guards/onboarding.guard';
import { RoleGuard } from '@core/guards/role.guard';

export const routes: Routes = [
  {
    path: '',
    redirectTo: '/tabs/jobs',
    pathMatch: 'full'
  },
  {
    path: 'auth',
    loadChildren: () => import('./features/auth/auth.routes').then(m => m.AUTH_ROUTES)
  },
  {
    path: 'onboarding',
    loadChildren: () => import('./features/onboarding/onboarding.routes').then(m => m.ONBOARDING_ROUTES),
    canActivate: [AuthGuard]
  },
  {
    path: 'tabs',
    loadComponent: () => import('./layouts/tabs/tabs.page').then(m => m.TabsPage),
    canActivate: [AuthGuard, TenantGuard],
    children: [
      {
        path: 'jobs',
        loadChildren: () => import('./features/jobs/jobs.routes').then(m => m.JOBS_ROUTES),
        data: { preload: true }
      },
      {
        path: 'voice',
        loadChildren: () => import('./features/voice/voice.routes').then(m => m.VOICE_ROUTES),
        data: { preload: true }
      },
      {
        path: 'costs',
        loadChildren: () => import('./features/costs/costs.routes').then(m => m.COSTS_ROUTES)
      },
      {
        path: 'team',
        loadChildren: () => import('./features/team/team.routes').then(m => m.TEAM_ROUTES),
        canActivate: [RoleGuard],
        data: { roles: ['owner', 'representative'] }
      },
      {
        path: 'settings',
        loadChildren: () => import('./features/settings/settings.routes').then(m => m.SETTINGS_ROUTES)
      },
      {
        path: '',
        redirectTo: '/tabs/jobs',
        pathMatch: 'full'
      }
    ]
  },
  {
    path: '**',
    loadComponent: () => import('./pages/not-found/not-found.page').then(m => m.NotFoundPage)
  }
];
```

## Feature Routes

```typescript
// features/jobs/jobs.routes.ts - Feature-specific routes
import { Routes } from '@angular/router';
import { JobsListPage } from './pages/jobs-list/jobs-list.page';
import { JobDetailPage } from './pages/job-detail/job-detail.page';
import { JobEditPage } from './pages/job-edit/job-edit.page';
import { JobCreatePage } from './pages/job-create/job-create.page';
import { PendingChangesGuard } from '@core/guards/pending-changes.guard';
import { JobResolver } from './resolvers/job.resolver';

export const JOBS_ROUTES: Routes = [
  {
    path: '',
    component: JobsListPage,
    title: 'Jobs'
  },
  {
    path: 'create',
    component: JobCreatePage,
    canDeactivate: [PendingChangesGuard],
    title: 'Create Job'
  },
  {
    path: ':id',
    component: JobDetailPage,
    resolve: {
      job: JobResolver
    },
    title: 'Job Details'
  },
  {
    path: ':id/edit',
    component: JobEditPage,
    canDeactivate: [PendingChangesGuard],
    resolve: {
      job: JobResolver
    },
    title: 'Edit Job'
  },
  {
    path: ':id/costs',
    loadChildren: () => import('./features/job-costs/job-costs.routes').then(m => m.JOB_COSTS_ROUTES)
  }
];
```

## Route Guards

### Auth Guard

```typescript
// core/guards/auth.guard.ts - Authentication guard
import { Injectable, inject } from '@angular/core';
import { CanActivate, Router, UrlTree } from '@angular/router';
import { Auth, authState } from '@angular/fire/auth';
import { Observable } from 'rxjs';
import { map, take } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  private readonly auth = inject(Auth);
  private readonly router = inject(Router);

  canActivate(): Observable<boolean | UrlTree> {
    return authState(this.auth).pipe(
      take(1),
      map(user => {
        if (user) {
          return true;
        }
        // Store intended URL for redirecting after login
        const returnUrl = window.location.pathname;
        return this.router.createUrlTree(['/auth/login'], {
          queryParams: { returnUrl }
        });
      })
    );
  }
}
```

### Tenant Guard

```typescript
// core/guards/tenant.guard.ts - Ensure user has tenant association
import { Injectable, inject } from '@angular/core';
import { CanActivate, Router, UrlTree } from '@angular/router';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { map, take } from 'rxjs/operators';
import { selectCurrentTenant } from '@store/auth/auth.selectors';

@Injectable({ providedIn: 'root' })
export class TenantGuard implements CanActivate {
  private readonly store = inject(Store);
  private readonly router = inject(Router);

  canActivate(): Observable<boolean | UrlTree> {
    return this.store.select(selectCurrentTenant).pipe(
      take(1),
      map(tenant => {
        if (tenant) {
          return true;
        }
        // Redirect to onboarding if no tenant
        return this.router.createUrlTree(['/onboarding']);
      })
    );
  }
}
```

### Pending Changes Guard

```typescript
// core/guards/pending-changes.guard.ts - Prevent navigation with unsaved changes
import { Injectable } from '@angular/core';
import { CanDeactivate } from '@angular/router';
import { Observable } from 'rxjs';
import { AlertController } from '@ionic/angular';

export interface ComponentCanDeactivate {
  canDeactivate: () => Observable<boolean> | Promise<boolean> | boolean;
}

@Injectable({ providedIn: 'root' })
export class PendingChangesGuard implements CanDeactivate<ComponentCanDeactivate> {
  constructor(private alertController: AlertController) {}

  async canDeactivate(component: ComponentCanDeactivate): Promise<boolean> {
    if (!component.canDeactivate || component.canDeactivate()) {
      return true;
    }

    const alert = await this.alertController.create({
      header: 'Unsaved Changes',
      message: 'You have unsaved changes. Do you want to leave without saving?',
      buttons: [
        {
          text: 'Stay',
          role: 'cancel',
          cssClass: 'secondary'
        },
        {
          text: 'Leave',
          role: 'confirm',
          cssClass: 'danger'
        }
      ]
    });

    await alert.present();
    const { role } = await alert.onDidDismiss();

    return role === 'confirm';
  }
}
```

### Role Guard

```typescript
// core/guards/role.guard.ts - Role-based access control
import { Injectable, inject } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, Router, UrlTree } from '@angular/router';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { map, take } from 'rxjs/operators';
import { selectUserRole } from '@store/auth/auth.selectors';

@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {
  private readonly store = inject(Store);
  private readonly router = inject(Router);

  canActivate(route: ActivatedRouteSnapshot): Observable<boolean | UrlTree> {
    const requiredRoles = route.data['roles'] as string[];

    return this.store.select(selectUserRole).pipe(
      take(1),
      map(userRole => {
        if (requiredRoles.includes(userRole)) {
          return true;
        }
        // Redirect to unauthorized page or home
        return this.router.createUrlTree(['/tabs/jobs'], {
          queryParams: { unauthorized: true }
        });
      })
    );
  }
}
```

## Resolvers

### Job Resolver

```typescript
// core/resolvers/job.resolver.ts - Preload job data
import { Injectable, inject } from '@angular/core';
import { Resolve, ActivatedRouteSnapshot } from '@angular/router';
import { Store } from '@ngrx/store';
import { Observable, of } from 'rxjs';
import { take, tap, filter, switchMap } from 'rxjs/operators';
import { JobsActions } from '@features/jobs/data-access/jobs.actions';
import { selectJobById } from '@features/jobs/data-access/jobs.selectors';
import { Job } from '@shared/types';

@Injectable({ providedIn: 'root' })
export class JobResolver implements Resolve<Job> {
  private readonly store = inject(Store);

  resolve(route: ActivatedRouteSnapshot): Observable<Job> {
    const jobId = route.params['id'];

    // Check if job is already in store
    return this.store.select(state => selectJobById(jobId)(state)).pipe(
      take(1),
      switchMap(job => {
        if (job) {
          return of(job);
        }

        // Load job if not in store
        this.store.dispatch(JobsActions.loadJob({ id: jobId }));

        // Wait for job to be loaded
        return this.store.select(state => selectJobById(jobId)(state)).pipe(
          filter(job => !!job),
          take(1)
        );
      })
    );
  }
}
```

## Router Configuration

```typescript
// app.config.ts - Configure routing with preloading strategy
import { ApplicationConfig } from '@angular/core';
import {
  provideRouter,
  withPreloading,
  PreloadAllModules,
  withComponentInputBinding,
  withInMemoryScrolling,
  withRouterConfig
} from '@angular/router';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      routes,
      withPreloading(PreloadAllModules), // Preload all lazy modules for PWA
      withComponentInputBinding(), // Enable router input binding
      withInMemoryScrolling({
        scrollPositionRestoration: 'enabled', // Restore scroll position
        anchorScrolling: 'enabled'
      }),
      withRouterConfig({
        paramsInheritanceStrategy: 'always' // Inherit params from parent routes
      })
    ),
    // ... other providers
  ]
};
```

## Programmatic Navigation

### Using Router Service

```typescript
import { Component, inject } from '@angular/core';
import { Router } from '@angular/router';

export class JobsListComponent {
  private readonly router = inject(Router);

  navigateToJobDetail(jobId: string): void {
    this.router.navigate(['/tabs/jobs', jobId]);
  }

  navigateToJobEdit(jobId: string): void {
    this.router.navigate(['/tabs/jobs', jobId, 'edit']);
  }

  navigateWithQueryParams(): void {
    this.router.navigate(['/tabs/jobs'], {
      queryParams: { status: 'active', sort: 'date' }
    });
  }
}
```

### Using Ionic NavController

```typescript
import { Component, inject } from '@angular/core';
import { NavController } from '@ionic/angular';

export class JobDetailComponent {
  private readonly navCtrl = inject(NavController);

  goBack(): void {
    this.navCtrl.back();
  }

  navigateRoot(): void {
    this.navCtrl.navigateRoot('/tabs/jobs');
  }

  navigateForward(jobId: string): void {
    this.navCtrl.navigateForward(['/tabs/jobs', jobId]);
  }
}
```

## Deep Linking

### Configure Deep Links

```typescript
// capacitor.config.ts
import { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.findogai.app',
  appName: 'FinDogAI',
  plugins: {
    App: {
      iosScheme: 'findogai',
      androidScheme: 'findogai'
    }
  }
};

export default config;
```

### Handle Deep Links

```typescript
// app.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { Router } from '@angular/router';
import { App, URLOpenListenerEvent } from '@capacitor/app';

@Component({
  selector: 'app-root',
  template: '<ion-app><ion-router-outlet></ion-router-outlet></ion-app>'
})
export class AppComponent implements OnInit {
  private readonly router = inject(Router);

  ngOnInit(): void {
    // Handle deep links
    App.addListener('appUrlOpen', (event: URLOpenListenerEvent) => {
      const slug = event.url.split('findogai://').pop();
      if (slug) {
        this.router.navigateByUrl(`/${slug}`);
      }
    });
  }
}
```

## Route Data and Params

### Accessing Route Parameters

```typescript
import { Component, inject, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

export class JobDetailComponent implements OnInit {
  private readonly route = inject(ActivatedRoute);

  ngOnInit(): void {
    // Get route params
    const jobId = this.route.snapshot.params['id'];

    // Subscribe to param changes
    this.route.params.subscribe(params => {
      const id = params['id'];
      this.loadJob(id);
    });

    // Get query params
    this.route.queryParams.subscribe(queryParams => {
      const view = queryParams['view'];
    });
  }
}
```

### Using Component Input Binding (Angular 20+)

```typescript
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-job-detail',
  template: `<h1>Job {{ id }}</h1>`
})
export class JobDetailComponent {
  // Automatically bound from route params
  @Input() id?: string;
}
```

## Route Animations

```typescript
// app.routes.ts - Add animation data
export const routes: Routes = [
  {
    path: 'tabs/jobs',
    component: JobsListPage,
    data: { animation: 'JobsListPage' }
  },
  {
    path: 'tabs/jobs/:id',
    component: JobDetailPage,
    data: { animation: 'JobDetailPage' }
  }
];
```

## Routing Best Practices

1. **Lazy Loading**: Use lazy loading for all feature modules
2. **Guards**: Protect routes with appropriate guards
3. **Resolvers**: Preload data before route activation
4. **Title Strategy**: Set page titles for better UX and SEO
5. **Deep Linking**: Support deep links for mobile apps
6. **Type Safety**: Use typed route params and query params
7. **Navigation State**: Pass data between routes when needed
8. **Error Handling**: Handle navigation errors gracefully

---

[← Previous: API Integration](./api-integration.md) | [Back to Index](./index.md) | [Next: Styling Guidelines →](./styling-guidelines.md)
