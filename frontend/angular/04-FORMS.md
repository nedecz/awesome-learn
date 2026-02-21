# Angular Forms

## Table of Contents

1. [Overview](#overview)
2. [Template-Driven vs Reactive Forms](#template-driven-vs-reactive-forms)
3. [FormControl, FormGroup, FormArray](#formcontrol-formgroup-formarray)
4. [Typed Reactive Forms](#typed-reactive-forms)
5. [Form Validation](#form-validation)
6. [Dynamic Forms](#dynamic-forms)
7. [Form Patterns](#form-patterns)
8. [Forms with Signals](#forms-with-signals)
9. [Error Message Handling](#error-message-handling)
10. [Form Testing Strategies](#form-testing-strategies)
11. [When to Use Template-Driven vs Reactive](#when-to-use-template-driven-vs-reactive)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Angular provides two approaches to building forms: template-driven forms and reactive forms. Both support validation, error handling, and data binding, but differ in how form logic is structured. This document covers both approaches, typed reactive forms (Angular 14+), validation patterns, dynamic forms, and testing strategies.

### Scope

- Template-driven forms with `ngModel`
- Reactive forms with `FormControl`, `FormGroup`, `FormArray`
- Typed reactive forms for compile-time safety
- Built-in validators, custom validators, async validators
- Dynamic form generation
- Cross-field and conditional validation
- Signal integration with forms
- Error message display patterns
- Form testing strategies

---

## Template-Driven vs Reactive Forms

| Aspect | Template-Driven | Reactive |
|--------|----------------|----------|
| **Form model** | Created by directives in the template | Created explicitly in the component class |
| **Data flow** | Two-way binding with `ngModel` | Observable-based with `valueChanges` |
| **Validation** | Template attributes + directive validators | Programmatic validators on controls |
| **Typing** | Loosely typed | Strongly typed (Angular 14+) |
| **Testability** | Requires DOM rendering | Testable without DOM |
| **Dynamic forms** | Difficult | Straightforward |
| **Complexity** | Simpler for basic forms | Better for complex forms |
| **Required import** | `FormsModule` | `ReactiveFormsModule` |

---

## FormControl, FormGroup, FormArray

### FormControl

A single form input:

```typescript
import { Component } from '@angular/core';
import { FormControl, ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-search',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <input [formControl]="searchControl" placeholder="Search..." />
    <p>Current value: {{ searchControl.value }}</p>
  `,
})
export class SearchComponent {
  searchControl = new FormControl('');
}
```

### FormGroup

A group of related controls:

```typescript
import { Component } from '@angular/core';
import { FormGroup, FormControl, Validators, ReactiveFormsModule } from '@angular/forms';

@Component({
  selector: 'app-registration',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="registrationForm" (ngSubmit)="onSubmit()">
      <label>
        Name
        <input formControlName="name" />
      </label>
      <label>
        Email
        <input formControlName="email" type="email" />
      </label>

      <fieldset formGroupName="address">
        <legend>Address</legend>
        <input formControlName="street" placeholder="Street" />
        <input formControlName="city" placeholder="City" />
        <input formControlName="zipCode" placeholder="ZIP" />
      </fieldset>

      <button type="submit" [disabled]="registrationForm.invalid">Register</button>
    </form>
  `,
})
export class RegistrationComponent {
  registrationForm = new FormGroup({
    name: new FormControl('', [Validators.required, Validators.minLength(2)]),
    email: new FormControl('', [Validators.required, Validators.email]),
    address: new FormGroup({
      street: new FormControl('', Validators.required),
      city: new FormControl('', Validators.required),
      zipCode: new FormControl('', [Validators.required, Validators.pattern(/^\d{5}$/)]),
    }),
  });

  onSubmit(): void {
    if (this.registrationForm.valid) {
      console.log(this.registrationForm.value);
    }
  }
}
```

### FormArray

A dynamic list of controls:

```typescript
@Component({
  selector: 'app-skills-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="profileForm">
      <div formArrayName="skills">
        @for (skill of skills.controls; track $index; let i = $index) {
          <div>
            <input [formControlName]="i" [placeholder]="'Skill ' + (i + 1)" />
            <button type="button" (click)="removeSkill(i)">Remove</button>
          </div>
        }
      </div>
      <button type="button" (click)="addSkill()">Add Skill</button>
    </form>
  `,
})
export class SkillsFormComponent {
  profileForm = new FormGroup({
    skills: new FormArray([new FormControl('', Validators.required)]),
  });

  get skills(): FormArray {
    return this.profileForm.get('skills') as FormArray;
  }

  addSkill(): void {
    this.skills.push(new FormControl('', Validators.required));
  }

  removeSkill(index: number): void {
    this.skills.removeAt(index);
  }
}
```

---

## Typed Reactive Forms

Angular 14 introduced **typed reactive forms** that provide compile-time type safety.

### Typed FormGroup

```typescript
interface UserForm {
  name: FormControl<string>;
  email: FormControl<string>;
  age: FormControl<number | null>;
  newsletter: FormControl<boolean>;
}

@Component({ /* ... */ })
export class UserFormComponent {
  userForm = new FormGroup<UserForm>({
    name: new FormControl('', { nonNullable: true, validators: [Validators.required] }),
    email: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] }),
    age: new FormControl<number | null>(null),
    newsletter: new FormControl(false, { nonNullable: true }),
  });

  onSubmit(): void {
    // form.value is fully typed
    const value = this.userForm.value;
    // value.name is string | undefined (undefined if control is disabled)

    // getRawValue() includes disabled controls
    const raw = this.userForm.getRawValue();
    // raw.name is string (always defined)
  }
}
```

### NonNullableFormBuilder

```typescript
@Component({ /* ... */ })
export class UserFormComponent {
  private fb = inject(NonNullableFormBuilder);

  userForm = this.fb.group({
    name: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]],
    age: this.fb.control<number | null>(null),
    newsletter: [false],
  });

  reset(): void {
    this.userForm.reset();  // Resets to initial values, not null
  }
}
```

---

## Form Validation

### Built-in Validators

| Validator | Usage | Description |
|-----------|-------|-------------|
| `Validators.required` | `[Validators.required]` | Field must not be empty |
| `Validators.email` | `[Validators.email]` | Must be a valid email format |
| `Validators.minLength(n)` | `[Validators.minLength(3)]` | Minimum character count |
| `Validators.maxLength(n)` | `[Validators.maxLength(100)]` | Maximum character count |
| `Validators.min(n)` | `[Validators.min(0)]` | Minimum numeric value |
| `Validators.max(n)` | `[Validators.max(150)]` | Maximum numeric value |
| `Validators.pattern(regex)` | `[Validators.pattern(/^\d+$/)]` | Must match regex pattern |

### Custom Validator

```typescript
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export function forbiddenNameValidator(forbiddenName: RegExp): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const forbidden = forbiddenName.test(control.value);
    return forbidden ? { forbiddenName: { value: control.value } } : null;
  };
}

// Usage
name: new FormControl('', [
  Validators.required,
  forbiddenNameValidator(/admin/i),
]),
```

### Cross-Field Validator

```typescript
export const passwordMatchValidator: ValidatorFn = (group: AbstractControl): ValidationErrors | null => {
  const password = group.get('password')?.value;
  const confirm = group.get('confirmPassword')?.value;
  return password === confirm ? null : { passwordMismatch: true };
};

// Apply at the group level
this.form = new FormGroup({
  password: new FormControl('', [Validators.required, Validators.minLength(8)]),
  confirmPassword: new FormControl('', Validators.required),
}, { validators: passwordMatchValidator });
```

### Async Validator

```typescript
import { AsyncValidatorFn } from '@angular/forms';

export function uniqueEmailValidator(userService: UserService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return userService.checkEmailExists(control.value).pipe(
      map(exists => exists ? { emailTaken: true } : null),
      catchError(() => of(null)),
    );
  };
}

// Usage
email: new FormControl('', {
  validators: [Validators.required, Validators.email],
  asyncValidators: [uniqueEmailValidator(inject(UserService))],
  updateOn: 'blur',  // Only validate on blur to avoid excessive API calls
}),
```

---

## Dynamic Forms

Generate form controls dynamically from a configuration:

```typescript
interface FormFieldConfig {
  key: string;
  label: string;
  type: 'text' | 'number' | 'email' | 'select';
  required: boolean;
  options?: { value: string; label: string }[];
}

@Component({
  selector: 'app-dynamic-form',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      @for (field of fields; track field.key) {
        <div class="form-field">
          <label [for]="field.key">{{ field.label }}</label>
          @switch (field.type) {
            @case ('select') {
              <select [id]="field.key" [formControlName]="field.key">
                @for (opt of field.options; track opt.value) {
                  <option [value]="opt.value">{{ opt.label }}</option>
                }
              </select>
            }
            @default {
              <input [id]="field.key" [type]="field.type" [formControlName]="field.key" />
            }
          }
        </div>
      }
      <button type="submit" [disabled]="form.invalid">Submit</button>
    </form>
  `,
})
export class DynamicFormComponent {
  fields = input.required<FormFieldConfig[]>();
  formSubmit = output<Record<string, any>>();

  form!: FormGroup;
  private fb = inject(NonNullableFormBuilder);

  ngOnInit(): void {
    const controls: Record<string, FormControl> = {};
    for (const field of this.fields()) {
      const validators = field.required ? [Validators.required] : [];
      if (field.type === 'email') validators.push(Validators.email);
      controls[field.key] = this.fb.control('', validators);
    }
    this.form = this.fb.group(controls);
  }

  onSubmit(): void {
    if (this.form.valid) {
      this.formSubmit.emit(this.form.getRawValue());
    }
  }
}
```

---

## Form Patterns

### Conditional Validation

Enable or disable validation based on other form values:

```typescript
export class ShippingFormComponent {
  private fb = inject(NonNullableFormBuilder);

  form = this.fb.group({
    deliveryMethod: ['standard'],
    expressDate: [''],
  });

  constructor() {
    this.form.get('deliveryMethod')!.valueChanges.subscribe(method => {
      const expressDate = this.form.get('expressDate')!;
      if (method === 'express') {
        expressDate.addValidators(Validators.required);
      } else {
        expressDate.clearValidators();
      }
      expressDate.updateValueAndValidity();
    });
  }
}
```

### Form with Sub-Components

Split large forms across components while sharing the same FormGroup:

```typescript
// Parent
@Component({
  template: `
    <form [formGroup]="orderForm" (ngSubmit)="submit()">
      <app-customer-fields />
      <app-shipping-fields />
      <button type="submit">Place Order</button>
    </form>
  `,
})
export class OrderFormComponent {
  orderForm = inject(NonNullableFormBuilder).group({
    customer: this.buildCustomerGroup(),
    shipping: this.buildShippingGroup(),
  });
}

// Child — inject parent FormGroupDirective
@Component({
  selector: 'app-customer-fields',
  template: `
    <fieldset formGroupName="customer">
      <input formControlName="name" placeholder="Name" />
      <input formControlName="email" placeholder="Email" />
    </fieldset>
  `,
})
export class CustomerFieldsComponent {}
```

---

## Forms with Signals

### Reactive Form + Signal Derived State

```typescript
@Component({
  selector: 'app-signup',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="email" placeholder="Email" />
      <input formControlName="password" type="password" placeholder="Password" />

      @if (passwordStrength() === 'weak') {
        <p class="warning">Password is too weak</p>
      }

      <button type="submit" [disabled]="!isValid()">Sign Up</button>
    </form>
  `,
})
export class SignupComponent {
  form = new FormGroup({
    email: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.email] }),
    password: new FormControl('', { nonNullable: true, validators: [Validators.required, Validators.minLength(8)] }),
  });

  // Bridge form state to signals using toSignal
  private formStatus = toSignal(this.form.statusChanges, { initialValue: this.form.status });
  private passwordValue = toSignal(
    this.form.controls.password.valueChanges,
    { initialValue: '' },
  );

  isValid = computed(() => this.formStatus() === 'VALID');

  passwordStrength = computed(() => {
    const pwd = this.passwordValue();
    if (pwd.length < 8) return 'weak';
    if (/[A-Z]/.test(pwd) && /\d/.test(pwd) && /[^A-Za-z0-9]/.test(pwd)) return 'strong';
    return 'medium';
  });

  onSubmit(): void {
    if (this.form.valid) {
      console.log(this.form.getRawValue());
    }
  }
}
```

---

## Error Message Handling

### Centralized Error Messages

```typescript
const VALIDATION_MESSAGES: Record<string, (params?: any) => string> = {
  required: () => 'This field is required.',
  email: () => 'Please enter a valid email address.',
  minlength: (params) => `Minimum length is ${params.requiredLength} characters.`,
  maxlength: (params) => `Maximum length is ${params.requiredLength} characters.`,
  min: (params) => `Minimum value is ${params.min}.`,
  max: (params) => `Maximum value is ${params.max}.`,
  pattern: () => 'Invalid format.',
  passwordMismatch: () => 'Passwords do not match.',
  emailTaken: () => 'This email is already registered.',
};

export function getErrorMessage(control: AbstractControl): string {
  if (!control.errors) return '';
  const firstErrorKey = Object.keys(control.errors)[0];
  const messageFn = VALIDATION_MESSAGES[firstErrorKey];
  return messageFn ? messageFn(control.errors[firstErrorKey]) : 'Invalid value.';
}
```

### Reusable Error Component

```typescript
@Component({
  selector: 'app-field-error',
  standalone: true,
  template: `
    @if (control() && control()!.invalid && (control()!.dirty || control()!.touched)) {
      <span class="error">{{ errorMessage() }}</span>
    }
  `,
  styles: [`.error { color: #dc2626; font-size: 0.875rem; }`],
})
export class FieldErrorComponent {
  control = input.required<AbstractControl | null>();

  errorMessage = computed(() => {
    const ctrl = this.control();
    return ctrl ? getErrorMessage(ctrl) : '';
  });
}
```

```html
<!-- Usage -->
<input formControlName="email" />
<app-field-error [control]="form.controls.email" />
```

---

## Form Testing Strategies

### Testing Reactive Forms

```typescript
describe('RegistrationComponent', () => {
  let component: RegistrationComponent;
  let fixture: ComponentFixture<RegistrationComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [RegistrationComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(RegistrationComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should be invalid when empty', () => {
    expect(component.registrationForm.valid).toBeFalse();
  });

  it('should be valid when all required fields are filled', () => {
    component.registrationForm.patchValue({
      name: 'Alice',
      email: 'alice@example.com',
      address: {
        street: '123 Main St',
        city: 'Springfield',
        zipCode: '12345',
      },
    });
    expect(component.registrationForm.valid).toBeTrue();
  });

  it('should show email validation error for invalid email', () => {
    const emailControl = component.registrationForm.get('email')!;
    emailControl.setValue('not-an-email');
    expect(emailControl.errors?.['email']).toBeTruthy();
  });
});
```

### Testing Template-Driven Forms

```typescript
it('should update model when input changes', async () => {
  const input: HTMLInputElement = fixture.nativeElement.querySelector('#name');
  input.value = 'Bob';
  input.dispatchEvent(new Event('input'));
  fixture.detectChanges();
  await fixture.whenStable();
  expect(component.model.name).toBe('Bob');
});
```

---

## When to Use Template-Driven vs Reactive

| Scenario | Recommended Approach |
|----------|---------------------|
| Simple login/contact form | Template-driven |
| Complex multi-step wizard | Reactive |
| Forms with dynamic fields | Reactive |
| Forms needing cross-field validation | Reactive |
| Strong typing required | Reactive (typed forms) |
| Quick prototype | Template-driven |
| Forms tested without DOM | Reactive |
| Forms with real-time computed state | Reactive + Signals |

**General recommendation:** Use **reactive forms** for most Angular applications. They offer better type safety, testability, and scalability.

---

## Next Steps

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-RXJS-AND-OBSERVABLES](05-RXJS-AND-OBSERVABLES.md) | RxJS operators for form data streams |
| 2 | [07-HTTP-AND-API-INTEGRATION](07-HTTP-AND-API-INTEGRATION.md) | Submitting form data to APIs |
| 3 | [08-TESTING](08-TESTING.md) | Comprehensive testing strategies |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Angular forms documentation |
