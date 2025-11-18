# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Maker Maker** is a WordPress plugin for scaffolding CRUD applications and managing advanced custom resources. It's built on top of TypeRocket (a WordPress development framework) and manages a comprehensive service catalog system with pricing, equipment, coverage areas, and deliverables.

## Architecture

### Plugin Structure

This is a TypeRocket-based WordPress plugin with the following architecture:

- **Entry Point**: `makermaker.php` - Registers plugin hooks and initializes the main plugin class
- **Main Plugin Class**: `app/MakermakerTypeRocketPlugin.php` extends `TypeRocket\Pro\Register\BasePlugin`
  - Handles initialization, routes, policies, migrations, and asset registration
  - Auto-discovers policies from `app/Auth/*Policy.php`
  - Auto-loads resources from `inc/resources/*.php`

### Directory Structure

```
app/
  Auth/              - Policy classes for authorization
  Controllers/       - CRUD controllers for admin resources
  Helpers/           - Helper classes (e.g., ServiceCatalogHelper)
  Http/Fields/       - Form field definitions
  Models/            - Eloquent-style models extending TypeRocket\Models\Model
  View.php           - Custom view class

inc/
  resources/         - Resource registration files (e.g., service.php)
  routes/
    api.php         - API route definitions
    public.php      - Public web route definitions

database/
  migrations/        - Database migration files
  docs/              - Database documentation

resources/
  js/                - JavaScript assets (front.js, admin.js)
  sass/              - SASS stylesheets (front.scss, admin.scss)
  views/             - Blade-style view templates

tests/               - Pest/PHPUnit test suites
  Unit/
  Integration/
  Feature/
  Acceptance/
```

### Core Domain Models

The plugin manages a complex service catalog with these main entities:

**Service Management**:
- `Service` - Core service entity with SKU, pricing, complexity
- `ServiceCategory` - Hierarchical categorization
- `ServiceType` - Service type classification
- `ComplexityLevel` - Complexity tiers with price multipliers

**Pricing**:
- `ServicePrice` - Current pricing for services
- `PricingModel` - Pricing model definitions (hourly, fixed, tiered)
- `PricingTier` - Customer tiers (e.g., small_business, enterprise)
- `PriceRecord` - Historical price tracking
- `CurrencyRate` - Multi-currency support

**Service Relationships**:
- `ServiceAddon` - Required/optional add-on services
- `ServiceRelationship` - Prerequisites, dependencies, incompatibilities
- `ServiceBundle` / `BundleItem` - Bundled service packages

**Equipment & Deliverables**:
- `Equipment` - Required equipment for services
- `ServiceEquipment` - Many-to-many junction with quantity tracking
- `Deliverable` - Service deliverables
- `ServiceDeliverable` - Many-to-many junction
- `DeliveryMethod` - Delivery method options
- `ServiceDelivery` - Service delivery associations

**Coverage**:
- `CoverageArea` - Geographic coverage areas
- `ServiceCoverage` - Service availability by area

### Key Patterns

**Resource Registration**: Resources are registered in `inc/resources/` files using the `mm_create_custom_resource()` helper function. This creates admin pages with CRUD interfaces.

**Policy-Based Authorization**: All models have corresponding policy classes in `app/Auth/` that control access. Policies are auto-discovered by matching `*Policy.php` files to model names.

**Soft Deletes**: Models use `deleted_at` timestamp for soft deletion rather than hard deletes.

**Optimistic Locking**: Models have a `version` field for conflict detection during updates.

**Audit Trail**: Models track `created_by`, `updated_by`, `created_at`, `updated_at` using WordPress user IDs.

**Relationship Loading**: Models define extensive relationships and use `$with` property for eager loading to prevent N+1 queries.

## Development Commands

### Testing

```bash
# Run all tests with Pest
composer test

# Run specific test suites
composer test:unit          # Unit tests only
composer test:quick         # Unit + smoke tests with documentation
composer test:ci            # With coverage (minimum 85%)
composer test:all           # All test suites
composer test:affected      # Exclude slow/quarantined tests
```

### Dependencies

```bash
# Install PHP dependencies
composer install

# Install frontend dependencies
npm install
```

### Asset Compilation

```bash
# Development - watch for changes
npm run watch

# Production build
npm run prod
```

Assets are compiled from `resources/js/` and `resources/sass/` to `public/` directory with versioning.

## Database

### Migrations

- Migrations are in `database/migrations/`
- Plugin handles migrations automatically via `MakermakerTypeRocketPlugin::activate()`
- Migration key: `makermaker_migrations`

### Table Prefix

All custom tables use the `srvc_` prefix (defined in models via `$resource` property).
Global WordPress table prefix is available as `GLOBAL_WPDB_PREFIX` constant.

## Testing

The test suite uses **Pest** with **Brain Monkey** for mocking WordPress functions:

- Tests are organized into Unit, Integration, Feature, and Acceptance suites
- Tests automatically inherit group tags based on directory
- Brain Monkey setup/teardown happens automatically via `tests/Pest.php`
- Custom expectations available: `toCallWordPressFunction`, `toHaveWordPressAction`, `toHaveWordPressFilter`

## Important Helper Functions

**Auto-generate codes**: `autoGenerateCode($fields, $targetField, $sourceField, $separator, ...)` - Used in controllers to generate SKUs and slugs from names.

**Resource creation**: `mm_create_custom_resource($modelName, $controllerName, $displayName, $showInMenu = true)` - Creates TypeRocket resource registrations.

## TypeRocket Integration

This plugin extends TypeRocket Pro functionality:

- Uses TypeRocket's model system (extends `TypeRocket\Models\Model`)
- Controllers extend `TypeRocket\Controllers\Controller`
- Forms use `tr_form()` helper
- Routes defined in TypeRocket's route collection
- Views use TypeRocket's view system (accessed via `MakerMaker\View`)

## Constants

```php
MAKERMAKER_PLUGIN_DIR       // Plugin directory path
MAKERMAKER_PLUGIN_URL       // Plugin URL
GLOBAL_WPDB_PREFIX          // WordPress table prefix
TYPEROCKET_PLUGIN_MAKERMAKER_VIEWS_PATH  // Views directory path
```

## Service Model Business Logic

The `Service` model contains extensive business logic:

- **Price Calculation**: `getCalculatedPrice()` - Applies complexity multipliers to base prices
- **Availability Checks**: `isAvailableInArea()` - Geographic availability validation
- **Prerequisites**: `getPrerequisites()` - Returns required predecessor services
- **Conflicts**: `conflictsWith()` - Checks service incompatibilities
- **Equipment Costing**: `getRequiredEquipmentCost()` - Calculates total equipment costs
- **Effort Estimation**: `getTotalEstimatedEffort()` - Sums service + deliverable hours
- **Quantity Validation**: `validateQuantity()` - Enforces min/max quantity constraints

## WordPress Hooks

The plugin hooks into:
- `typerocket_loaded` (priority 9) - Main initialization
- `register_activation_hook` - Runs migrations, flushes rewrite rules
- `delete_plugin` - Cleanup on deletion
